---
title: Android BLASTSyncEngine
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android BLASTSyncEngine
date: 2022-11-23 10:00:00
---

## 概述

BLASTSyncEngine 设计的目的就是收集所有参与同步操作容器的 Transition 最后一起同步处理。用来代替之前的 APPTransaction。     
最开始是用在分屏中，将2个 Task 及其子容器对 Surface 的操作（通过Transition） 一起提交给 SF ，这样避免了止两个窗口在大小变化时由于刷新不同步导致显示异常。     
在新的版本中，ShellTransitions 的核心类，在窗口动画过程中起到很关键的作用。   

开启相关日志：     

```
adb shell wm logging enable-text WM_DEBUG_SYNC_ENGINE
```

BLASTSyncEngine 是 WMS 的成员变量，基本每次窗口的事务切换都会围绕这个对象进行。     

```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    final BLASTSyncEngine mSyncEngine;
    ......
    private WindowManagerService(Context context, InputManagerService inputManager,
        ......
        mSyncEngine = new BLASTSyncEngine(this);
        ......
    }
    
}
```

BLASTSyncEngine 下维护了一个集合 mActiveSyncs ，这个集合里的元素是 SyncGroup。      

```
class BLASTSyncEngine {
    /** Currently active syncs. Intentionally ordered by start time. */
    private final ArrayList<SyncGroup> mActiveSyncs = new ArrayList<>();
```

SyncGroup 代表一次同步任务组，也就是系统中某个逻辑，需要使用同步引擎完成时，就会为这次操作创建一个 SyncGroup。    
SyncGroup 下维护了一个集合 mRootMembers ，内部保存的是容器，但是一般都是 Task , ActivityRecord 级别。因为一般需要同步操作时，改变操作的也是这个级别的容器，命名为 “root”是因为虽然添加进集合的是它们，但是它们内部还有子容器，一般指的是 WindowState。    
SyncGroup 创建的时候接收一个接口回调: TransactionReadyListener ，这个回调也将被保存在成员变量 mListener 中。作用是这次同步任务结束后，通过接口方法回调给使用者。      
当 SyncGroup 下 mRootMembers  集合所有容器极其子容器都完成同步后，这个同步组的同步任务也就完成了，就会回调  TransactionReadyListener::onTransactionReady 方法回调给调用者。      

```
    class SyncGroup {
        final TransactionReadyListener mListener;
        // 变量保存了参与动画的WindowContainer的集合。
        final ArraySet<WindowContainer> mRootMembers = new ArraySet<>();
```

```
    interface TransactionReadyListener {
        void onTransactionReady(int mSyncId, SurfaceControl.Transaction transaction);
        default void onTransactionCommitTimeout() {}
        default void onReadyTimeout() {}
    }
```

TransactionReadyListener 的实现类有 Transition 和 WindowOrganizerController。    
onTransactionReady() 方法中的参数 SurfaceControl.Transaction 代表了这次操作各个容器对应的 Surface 操作。     
当同步任务完成时，会收集参与容器的 SurfaceControl.Transaction 合并成一个 SurfaceControl.Transaction 作为 TransactionReadyListener::onTransactionReady 的参数回调出去。    
发起同步任务者，收到回调的同时，也拿到了 SurfaceControl.Transaction 的合集，就可以做对应的行为了。    
因此一次同步任务可以分为下面几步：    

 - 启动一个同步组
 - 往同步组添加容器
 - 触发容器调整
 - 设置同步组准备就绪
 - 检查是否同步完成

## 同步流程

### 搜集阶段

这个阶段主要包含:    
 - 为对应操作创建 SyncGroup，后续才能把需要参与同步的容器添加到这个 SyncGroup。     
 - 然后把这个 SyncGroup 添加到 BLASTSyncEngine 中的 mActiveSyncs 列表中。    
 - 设置超时处理。       

```
TransitionController.moveToCollecting
    Transition.startCollecting
        BLASTSyncEngine.startSyncSet
            BLASTSyncEngine.prepareSyncSet
                //创建 SyncGroup
                new SyncGroup
                    mSyncId = id
                    mListener = listener
                    mOnTimeout = {}
        BLASTSyncEngine.startSyncSet
            // SyncGroup 添加到 mActiveSyncs
            mActiveSyncs.add(s)
            // 设置超时处理
            BLASTSyncEngine.scheduleTimeout()
                
```

创建 SyncGroup 会设置 TransactionReadyListener，一般也是同步任务的发起者，当同步完成后会通知监听者。    
也会设置一个唯一的 ID mSyncId。    

```
        private SyncGroup(TransactionReadyListener listener, int id, String name) {
            mSyncId = id;
            mSyncName = name;
            mListener = listener;
            mOnTimeout = () -> {
                Slog.w(TAG, "Sync group " + mSyncId + " timeout");
                synchronized (mWm.mGlobalLock) {
                    onTimeout();
                }
            };
            ......
        }
```

### 添加容器到同步组

使用 `TransitionController.collect` 方法可以将参与同步的 WindowContainer 添加进 SyncGroup。     
主要包含下面几个步骤:      

 - 将 WindowContainer 添加到 mRootMembers 集合。
 - 为 WindowContainer 设置 SyncGroup
 - 设置 WindowContainer 的同步状态。其中 WindowContainer 设置为 SYNC_STATE_READY，因为它没有内容，不需要绘制，而 WindowState 需要绘制，设置状态为 SYNC_STATE_WAITING_FOR_DRAW。    
 - 触发一次 requestTraversal()。

```
TransitionController.collect(WindowContainer wc)
    Transition.collect
        BLASTSyncEngine.addToSyncSet
            SyncGroup.addToSync
                mRootMembers.add(wc)
                WindowContainer.setSyncGroup()
                WindowContainer.prepareSync()
                    mSyncState = SYNC_STATE_READY 或 SYNC_STATE_WAITING_FOR_DRAW;
                WindowSurfacePlacer.requestTraversal()
```

WindowContainer 类有个 SyncGroup 变量。    

```
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable,
        InsetsControlTarget {
    ......
    BLASTSyncEngine.SyncGroup mSyncGroup = null;
```


### 容器属性变更

开发者使用同步引擎就是业务逻辑需要修改多个容器的属性，配置时会触发容器重绘，而业务逻辑需要等所有容器重绘后，拿到各个容器等事务一起处理。      
那么接下来就是触发触发相关容器的属性改变的逻辑了。    


### 设置同步组准备完毕

当应用端完成所有绘制，那么就会通知 WMCore 来设置同步组响应的状态，此时表示可以提交到 SF 进行合成了。    
这里为什么要执行一次刷新逻辑呢？因为后面是在每一次 layout 的时候都检查一下，参与同步的容器是不是都绘制好了。    

```
WindowOrganizerController.onTransact
    Transition.start
        Transition.applyReady
            BLASTSyncEngine.setReady
                SyncGroup.setReady
                    mReady = true
                    // 触发刷新
                    WindowSurfacePlacer.requestTraversal()
```

### 同步状态检查

系统在每一次 layout 的时候会同时检查同步组是否完成了绘制。     
如果同步组内容器都完成了绘制，构造了一个局部 Transaction 变量 merged，它将所有参与动画的 WindowContainer 在动画期间发生的 mSyncTransaction 操作都合并到这个局部变量 merged 中。    
然后引擎回通过 `TransactionReadyListener.onTransactionReady` 方法将参与同步操作的容器的所有同步事务，回调给调用者。      

```
RootWindowContainer.performSurfacePlacementNoTrace()
    BLASTSyncEngine.onSurfacePlacement
        SyncGroup.tryFinish
            SyncGroup.finishNow
                // 构造 Transaction
                merged = new SurfaceControl.Transaction
                WindowContainer.finishSync
                    // 把该 WindowContainer 的 Transaction 合并到 merged
                    Transaction.merge(mSyncTransaction)
                    mSyncState = SYNC_STATE_NONE;
                    mSyncGroup = null
                // 将搜集到的 Transaction 发送给监听者
                TransactionReadyListener.onTransactionReady(mSyncId, merged)
```

```
BLASTSyncEngine.SyncGroup
        private boolean tryFinish() {
            if (!mReady) return false;
            ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d: onSurfacePlacement checking %s",
                    mSyncId, mRootMembers);
            if (!mDependencies.isEmpty()) {
                ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d:  Unfinished dependencies: %s",
                        mSyncId, mDependencies);
                return false;
            }
            // 遍历同步组内的所有容器，必须所有容器都同步完成才继续走下面逻辑
            for (int i = mRootMembers.size() - 1; i >= 0; --i) {
                final WindowContainer wc = mRootMembers.valueAt(i);
                if (!wc.isSyncFinished(this)) {
                    ProtoLog.v(WM_DEBUG_SYNC_ENGINE, "SyncGroup %d:  Unfinished container: %s",
                            mSyncId, wc);
                    return false;
                }
            }
            // 全部完成同步后逻辑
            finishNow();
            return true;
        }
```

容器完成同步的条件：    

```
    boolean isSyncFinished(BLASTSyncEngine.SyncGroup group) {
        //1. 不可见则不需要等待绘制，直接视为同步完成
        if (!isVisibleRequested()) {
            return true;
        }
        // 2. 设置容步状态
        if (mSyncState == SYNC_STATE_NONE && getSyncGroup() != null) {
            Slog.i(TAG, "prepareSync in isSyncFinished: " + this);
            prepareSync();
        }
        // 3. 等待绘制的状态肯定返回 false 
        if (mSyncState == SYNC_STATE_WAITING_FOR_DRAW) {
            return false;
        }
        // READY
        // Loop from top-down.
        // 遍历子节点
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            final WindowContainer child = mChildren.get(i);
            final boolean childFinished = group.isIgnoring(child) || child.isSyncFinished(group);
            // 有任意1个孩子满足同时满足下面3个条件，则视为当前容器已经完成同步
            // 同步完成 
            // 请求可见
            // 不透明且填满父容器
            if (childFinished && child.isVisibleRequested() && child.fillsParent()) {
                // Any lower children will be covered-up, so we can consider this finished.
                return true;
            }
            // 有任意一个孩子没同步完成，则直接false
            if (!childFinished) {
                return false;
            }
        }
        return true;
    }
```

```
        private void finishNow() {
            ....
            // 构造一个事务
            SurfaceControl.Transaction merged = mWm.mTransactionFactory.get();
            if (mOrphanTransaction != null) {
                // 把 mOrphanTransaction 合并到 merged 事务
                merged.merge(mOrphanTransaction);
            }
            // 2. 遍历参与同步的容器，拿到它们的 Transaction
            for (WindowContainer wc : mRootMembers) {
                wc.finishSync(merged, this, false /* cancel */);
            }
            //3. 遍历参与同步的容器，将其添加到wcAwaitingCommit集合
            // 因为距离 merged 被 apply 还有一段距离，
            // 在这段时间内参与到动画的WindowContainer是有可能继续发生变化的，
            // 而 syncTransaction 合并到 merged 的操作已经结束了，
            // 为了让这个时间段的变化也能够被应用，所以要确保 WindowContainer.getSyncTransaction()
            // 返回的是 WindowContainer.mSyncTransaction
            // 具体的解释见 ShellTransition 一节中的介绍
            final ArraySet<WindowContainer> wcAwaitingCommit = new ArraySet<>();
            for (WindowContainer wc : mRootMembers) {
                wc.waitForSyncTransactionCommit(wcAwaitingCommit);
            }

            final int syncId = mSyncId;
            final long mergedTxId = merged.getId();
            final String syncName = mSyncName;
            // 4. 定义回调。当merged这个Transaction对象被apply后，执行其 onCommitted 方法
            class CommitCallback implements Runnable {
                // Can run a second time if the action completes after the timeout.
                boolean ran = false;
                public void onCommitted(SurfaceControl.Transaction t) {
                    // 移除超时
                    mHandler.removeCallbacks(this);
                    synchronized (mWm.mGlobalLock) {
                        if (ran) {
                            return;
                        }
                        ran = true;
                        // 8. 收集 wcAwaitingCommit 下的事务
                        for (WindowContainer wc : wcAwaitingCommit) {
                            wc.onSyncTransactionCommitted(t);
                        }
                        // 提交事务
                        t.apply();
                        // 清空集合
                        wcAwaitingCommit.clear();
                    }
                }

                // 超时才执行
                @Override
                public void run() {
                    ......
                    // 超时处理
                    synchronized (mWm.mGlobalLock) {
                        mListener.onTransactionCommitTimeout();
                        onCommitted(merged.mNativeObject != 0
                                ? merged : mWm.mTransactionFactory.get());
                    }
                }
            };
            CommitCallback callback = new CommitCallback();
            //5. 添加 apply 监听
            merged.addTransactionCommittedListener(Runnable::run,
                    () -> callback.onCommitted(new SurfaceControl.Transaction()));
            mHandler.postDelayed(callback, BLAST_TIMEOUT_DURATION);

            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "onTransactionReady");
            // 6.同步引擎最后一步：回调总的 Transition 给调用者
            mListener.onTransactionReady(mSyncId, merged);
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            // 7.移除工作
            mActiveSyncs.remove(this);
            mHandler.removeCallbacks(mOnTimeout);

            ......
        }
```


## 相关文章

[【Android 14源码分析】BLASTSyncEngine 设计剖析 ](https://juejin.cn/post/7410672790020440100)      
[BLAST深入源码剖析](https://blog.csdn.net/learnframework/article/details/135337779)      
[【Android 12】【WCT的同步】BLASTSyncEngine](https://blog.csdn.net/ukynho/article/details/126747856)      
[SHELL TRANSITIONS](https://www.jianshu.com/p/0efc1898f241)      
