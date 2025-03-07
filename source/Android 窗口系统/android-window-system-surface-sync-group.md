---
title: Android Surface 同步机制 SurfaceSyncGroup
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android Surface 同步机制 SurfaceSyncGroup
date: 2022-11-23 10:00:00
---


## 概述

SurfaceSyncGroup 提供了用于同步多个 Surface 操作的一种机制，它允许多个 Surface 之间的渲染和显示操作能够同步执行，以确保它们在屏幕上的呈现是协调的。一般用于 AttachedSurfaceControl、SurfaceView、SurfaceControlViewHost 和任何其他希望参与同步的 Surface。这让 Android 中的不同模块（同进程）、不同进程之间可以自行同步不同 Window 和 Layer。      
只有当添加到 SurfaceSyncGroup 的所有 Surface 都完成绘制，SurfaceSyncGroup 的 syncComplete 方法才会被调用。    
在 Android 13 中增加了 Surface 同步机制，主要作用是提供 ViewRootImpl、SurfaceView 和 SurfaceControlViewHost 的渲染同步服务。具体的，在 ViewRootImpl、SurfaceView 和 SurfaceControlViewHost 均完成绘制时再去上报至 WMS 进行窗口状态的切换。避免 ViewRootImpl 绘制完成上报至 WMS 后、同时 SurfaceView/SurfaceControlViewHost 迟迟没有绘制完成使得 startingWindow 过早移除从而产生黑屏/闪屏的现象。    
Android13 中 Surface 同步相关的代码主要在 SurfaceSyncer.java，Android14 重构为 SurfaceSyncGroup.java。    

它提供了下面的方法：

 - add(@Nullable AttachedSurfaceControl attachedSurfaceControl,@Nullable Runnable runnable) ：添加 ViewRootImpl 到同步 Group 的方法。
 - add(@NonNull SurfaceControlViewHost.SurfacePackage surfacePackage,@Nullable Runnable runnable)：添加 SurfaceControlViewHost.SurfacePackage 到同步 Group 的方法，常用于跨进程 Surface 同步。
 - add(@NonNull SurfaceSyncGroup surfaceSyncGroup,@Nullable Runnable runnable)
 - add(ISurfaceSyncGroup surfaceSyncGroup, boolean parentSyncGroupMerge,@Nullable Runnable runnable) 
 - add(SurfaceView surfaceView,Consumer<SurfaceViewFrameCallback> frameCallbackConsumer)：添加 SurfaceView 到同步 Group 的方法
 - addTransaction(@NonNull Transaction transaction)：向此 SurfaceSyncGroup 添加事务。这允许调用方提供应与此 SurfaceSyncGroup 中的其他事务同步的其他信息。
 - addSyncCompleteCallback(Executor executor, Runnable runnable)：添加监听同步结束的方法。
 - markSyncReady()：将 SurfaceSyncGroup 标记为 ready to complete。无法向此 SurfaceSyncGroup 添加更多数据。    
 - addSyncToWm()：
 - addLocalSync()：添加本地的 SurfaceSyncGroup，也就是同一个进程的。

SurfaceSyncGroup 的一些特点：

 - 每一个 SurfaceSyncGroup 对象都有一个 mTransactionReadyConsumer 回调，当对应的 ViewRootImpl/SurfaceView/SurfaceControlViewHost 绘制完成后，在 checkIfSyncIsComplete() 方法中会调用到该回调。
 - 多个 SurfaceSyncGroup 可以通过树结构组织起来，通过成员 `ISurfaceSyncGroup mParentSyncGroup;` 可以找到其父节点。ViewRootImpl 中的 mWmsRequestSyncGroup 成员是这颗树的根节点。
 - 当 SurfaceSyncGroup 节点 A 添加一个子节点 B 时
  - 会将子节点 B 的 ISurfaceSyncGroup mParentSyncGroup; 成员指向父节点 A 的 ISurfaceSyncGroup mISurfaceSyncGroup 成员。mISurfaceSyncGroup 有一个隐式的 this 引用指向 SurfaceSyncGroup 节点 A。可以通过 getSurfaceSyncGroup() 方法获取。    
  - 会将节点 A 的 Consumer<Transaction> mTransactionReadyConsumer; 包装成一个 ITransactionReadyCallback 对象，添加到节点 A 的ArraySet<ITransactionReadyCallback> mPendingSyncs 成员中。
  - 同时将该 ITransactionReadyCallback 对象发送给节点 B，节点 B 重新构造自己的 Consumer<Transaction> mTransactionReadyConsumer;
  - 当节点 B 对应的 Surface 绘制完后后，就会调用重新构造的 Consumer<Transaction> mTransactionReadyConsumer; 回调，在该回调中，先调用节点 B 自己原来的 mTransactionReadyConsumer 回调，再调用父节点 A 传过来的 ITransactionReadyCallback，同时将 ITransactionReadyCallback 从父节点 A 的 mPendingSyncs 成员中删除
 - 只有 A 节点中的 mPendingSyncs 为空时，A 节点的子节点们才算都绘制完成了。

在 server 端，SurfaceSyncGroupController

## 同步流程

ViewRootImpl 有下面几个 SurfaceSyncGroup 对象。    

```
// ViewRootImpl.java
    // WMS 要求 sync buffer是创建，一般是 SurfaceSyncGroup 树的根节点
    private SurfaceSyncGroup mWmsRequestSyncGroup;
    /**
     * The SurfaceSyncGroup that represents the active VRI SurfaceSyncGroup. This is non null if
     * anyone requested the SurfaceSyncGroup for this VRI to ensure that anyone trying to sync with
     * this VRI are collected together. The SurfaceSyncGroup is cleared when the VRI draws since
     * that is the stop point where all changes are have been applied. A new SurfaceSyncGroup is
     * created after that point when something wants to sync VRI again.
     */
    private SurfaceSyncGroup mActiveSurfaceSyncGroup;
    /**
     * Wraps the TransactionCommitted callback for the previous SSG so it can be added to the next
     * SSG if started before previous has completed.
     */
    @GuardedBy("mPreviousSyncSafeguardLock")
    private SurfaceSyncGroup mPreviousSyncSafeguard;
```

```
    private void performTraversals() {
    ......
        boolean cancelDueToPreDrawListener = mAttachInfo.mTreeObserver.dispatchOnPreDraw();
        boolean cancelAndRedraw = cancelDueToPreDrawListener
                 || (cancelDraw && mDrewOnceForSync);

        if (!cancelAndRedraw) {
            // A sync was already requested before the WMS requested sync. This means we need to
            // sync the buffer, regardless if WMS wants to sync the buffer.
            if (mActiveSurfaceSyncGroup != null) {
                mSyncBuffer = true;
            }

            createSyncIfNeeded();
    }
```

### 整体流程

```
ViewRootImpl.performTraversals
    ViewRootImpl.createSyncIfNeeded
        // 当此次刷新需要重新绘制时创建
        // 
        mWmsRequestSyncGroup = new SurfaceSyncGroup("wmsSync-"
        // 把 ViewRootImpl 本身添加到这个同步组
        mWmsRequestSyncGroup.add(ViewRootImpl)
            ViewRootImpl.getOrCreateSurfaceSyncGroup()
                // 创建mActiveSurfaceSyncGroup
                mActiveSurfaceSyncGroup = new SurfaceSyncGroup(mTag)
            // 把 mActiveSurfaceSyncGroup 添加到 mWmsRequestSyncGroup
            SurfaceSyncGroup.add()
                SurfaceSyncGroup.add()
                    // 判断是否本地 SurfaceSyncGroup
                    SurfaceSyncGroup.isLocalBinder()
                    SurfaceSyncGroup.addLocalSync()
                        SurfaceSyncGroup.createTransactionReadyCallback()
                            // 添加到 mPendingSyncs 列表
                            mPendingSyncs.add(transactionReadyCallback);
                            // 超时处理
                            addTimeout()
                        SurfaceSyncGroup.setTransactionCallbackFromParent()
                    // 如果是远程的，其他进程的 SurfaceSyncGroup
                    SurfaceSyncGroup.addSyncToWm()
    //开始绘制，注入传入参数为 mActiveSurfaceSyncGroup
    ViewRootImpl.performDraw(mActiveSurfaceSyncGroup)
        ViewRootImpl.draw
            ViewRootImpl.registerCallbacksForSync()
                // 注册下一帧绘制完成的回调，该回调发生在RT线程，且在swap交换前执行
                mAttachInfo.mThreadedRenderer.registerRtFrameCallback()
                    // 当一帧绘制完成时，回调
                    // 将传入的事务与 BLASTBufferQueue 中的下一个事务合并。
                    ViewRootImpl.mergeWithNextTransaction()
                        BlastBufferQueue.mergeWithNextTransaction()
                    // 绘制完成，通知 SurfaceSyncGroup
                    // 标记 mActiveSurfaceSyncGroup markSyncReady
                    // 通常情况下此时 mWmsRequestSyncGroup 还没有ready
                    SurfaceSyncGroup.markSyncReady()
    // mWmsRequestSyncGroup markSyncReady
    SurfaceSyncGroup.markSyncReady()
        SurfaceSyncGroup.checkIfSyncIsComplete()
            // 执行创建 SurfaceSyncGroup 时的回调
            mTransactionReadyConsumer.accept(mTransaction)
                ViewRootImpl.reportDrawFinished()
                    // 通知 WMS 绘制完成，可以显示了
                    WindowSession.finishDrawing()
```

### SurfaceSyncGroup 初始化

```
    private void createSyncIfNeeded() {
        // 如果 WMS 是正在请求同步或者没有绘制的需要，就返回
        if (isInWMSRequestedSync() || !mReportNextDraw) {
            return;
        }

        final int seqId = mSyncSeqId;
        mWmsRequestSyncGroupState = WMS_SYNC_PENDING;
        // 创建 SurfaceSyncGroup
        mWmsRequestSyncGroup = new SurfaceSyncGroup("wmsSync-" + mTag, t -> {
            mWmsRequestSyncGroupState = WMS_SYNC_MERGED;
            if (mWindowSession instanceof Binder) {
                ....
                Transaction transactionCopy = new Transaction();
                transactionCopy.merge(t);
                mHandler.postAtFrontOfQueue(() -> reportDrawFinished(transactionCopy, seqId));
            } else {
                reportDrawFinished(t, seqId);
            }
        });

        ......
        // 把 ViewRootImpl 添加到 mWmsRequestSyncGroup
        mWmsRequestSyncGroup.add(this, null /* runnable */);
    }
```

createSyncIfNeeded 方法创建了名称为 `wmsSync-VRI[WMSTestActivity]#2`的 SurfaceSyncGroup，并把 ViewRootImpl 本身添加到 mWmsRequestSyncGroup 来构建 SurfaceSyncGroup 树。     
先来看一下 SurfaceSyncGroup 的构造方法。    

```
//SurfaceSyncGroup.java
    // 传入的 transactionReadyConsumer 也是 ready时才会执行
    public SurfaceSyncGroup(String name, Consumer<Transaction> transactionReadyConsumer) {
        if (sCounter.get() >= MAX_COUNT) {
            sCounter.set(0);
        }
        // 设置名称
        mName = name + "#" + sCounter.getAndIncrement();
        mTrackName = "SurfaceSyncGroup " + name;
        // 构建 mTransactionReadyConsumer，当 SurfaceSyncGroup ready时执行的回调
        mTransactionReadyConsumer = (transaction) -> {
            ...
            // 执行传入的 transactionReadyConsumer 回调
            transactionReadyConsumer.accept(transaction);
            synchronized (mLock) {
                // 执行 mSyncCompleteCallbacks 回调
                if (mSurfaceSyncGroupCompletedListener == null) {
                    invokeSyncCompleteCallbacks();
                }
            }
        };

        ....
    }
```

再来看看构建 SurfaceSyncGroup 树的流程

```
//SurfaceSyncGroup.java
    public boolean add(@Nullable AttachedSurfaceControl attachedSurfaceControl,
            @Nullable Runnable runnable) {
        if (attachedSurfaceControl == null) {
            return false;
        }
        // 调用 ViewRootImpl 的 getOrCreateSurfaceSyncGroup 创建一个 SurfaceSyncGroup
        // 名称为 VRI[WMSTestActivity]#3
        SurfaceSyncGroup surfaceSyncGroup = attachedSurfaceControl.getOrCreateSurfaceSyncGroup();
        if (surfaceSyncGroup == null) {
            return false;
        }
        // 把 ViewRootImpl 添加到 mWmsRequestSyncGroup 树
        return add(surfaceSyncGroup, runnable);
    }
```

```
//ViewRootImpl.java
    @Override
    public SurfaceSyncGroup getOrCreateSurfaceSyncGroup() {
        boolean newSyncGroup = false;
        if (mActiveSurfaceSyncGroup == null) {
            mActiveSurfaceSyncGroup = new SurfaceSyncGroup(mTag);
            mActiveSurfaceSyncGroup.setAddedToSyncListener(() -> {
                Runnable runnable = () -> {
                    // Check if it's already 0 because the timeout could have reset the count to
                    // 0 and we don't want to go negative.
                    if (mNumPausedForSync > 0) {
                        mNumPausedForSync--;
                    }
                    if (mNumPausedForSync == 0) {
                        mHandler.removeMessages(MSG_PAUSED_FOR_SYNC_TIMEOUT);
                        if (!mIsInTraversal) {
                            scheduleTraversals();
                        }
                    }
                };

                if (Thread.currentThread() == mThread) {
                    runnable.run();
                } else {
                    mHandler.post(runnable);
                }
            });
            newSyncGroup = true;
        }

        ......

        mNumPausedForSync++;
        mHandler.removeMessages(MSG_PAUSED_FOR_SYNC_TIMEOUT);
        mHandler.sendEmptyMessageDelayed(MSG_PAUSED_FOR_SYNC_TIMEOUT,
                1000 * Build.HW_TIMEOUT_MULTIPLIER);
        return mActiveSurfaceSyncGroup;
    };
```

在 getOrCreateSurfaceSyncGroup 方法中构造的 ViewRootImpl 的 mActiveSurfaceSyncGroup SurfaceSyncGroup 时的构造方法为：    
这里配置了一个默认的 ready 回调，就是简单的 apply transaction。    

```
    public SurfaceSyncGroup(@NonNull String name) {
        this(name, transaction -> {
            if (transaction != null) {
                ......
                transaction.apply();
            }
        });
    }
```

接着上面的 add 流程介绍。        
上面是把 ViewRootImpl 做为参数调用 add 方法，然后创建了 SurfaceSyncGroup，添加到 mWmsRequestSyncGroup 树。

```
    public boolean add(@NonNull SurfaceSyncGroup surfaceSyncGroup,
            @Nullable Runnable runnable) {
        // 获取 ViewRootImpl 的 mISurfaceSyncGroup 执行 add 方法
        return add(surfaceSyncGroup.mISurfaceSyncGroup, false /* parentSyncGroupMerge */,
                runnable);
    }
```

ViewRootImpl 的 mISurfaceSyncGroup 是一个 ISurfaceSyncGroupImpl，方便执行跨检查的同步。    

```
    private class ISurfaceSyncGroupImpl extends ISurfaceSyncGroup.Stub {
        @Override
        public boolean onAddedToSyncGroup(IBinder parentSyncGroupToken,
                boolean parentSyncGroupMerge) {
            ...
            boolean didAdd = addSyncToWm(parentSyncGroupToken, parentSyncGroupMerge, null);
            if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                Trace.asyncTraceForTrackEnd(Trace.TRACE_TAG_VIEW, mTrackName, hashCode());
            }
            return didAdd;
        }

        @Override
        public boolean addToSync(ISurfaceSyncGroup surfaceSyncGroup, boolean parentSyncGroupMerge) {
            return SurfaceSyncGroup.this.add(surfaceSyncGroup, parentSyncGroupMerge,
                    null /* runnable */);
        }
        // ISurfaceSyncGroupImpl是一个内部类，这里可以获取到外部类对象
        SurfaceSyncGroup getSurfaceSyncGroup() {
            return SurfaceSyncGroup.this;
        }
    }
```

接着上面的 add 流程介绍。    

```
    public boolean add(ISurfaceSyncGroup surfaceSyncGroup, boolean parentSyncGroupMerge,
            @Nullable Runnable runnable) {
        ......
        synchronized (mLock) {
            if (mSyncReady) {
                ......
                return false;
            }
        }

        if (runnable != null) {
            runnable.run();
        }
        // 如果是本地的 SurfaceSyncGroup，也就是同一个进程的
        if (isLocalBinder(surfaceSyncGroup.asBinder())) {
            // 调用 addLocalSync，重点关注这个方法
            boolean didAddLocalSync = addLocalSync(surfaceSyncGroup, parentSyncGroupMerge);
            ......
            return didAddLocalSync;
        }
        // 设计跨进程的 SurfaceSyncGroup，这里不重点关注
        synchronized (mLock) {
            if (!mHasWMSync) {
                // 向 WMS 去注册，因为其他进程的 SurfaceSyncGroup 的信息只有 WMS 才有
                mSurfaceSyncGroupCompletedListener = new ISurfaceSyncGroupCompletedListener.Stub() {
                    @Override
                    public void onSurfaceSyncGroupComplete() {
                        synchronized (mLock) {
                            invokeSyncCompleteCallbacks();
                        }
                    }
                };
                if (!addSyncToWm(mToken, false /* parentSyncGroupMerge */,
                        mSurfaceSyncGroupCompletedListener)) {
                    mSurfaceSyncGroupCompletedListener = null;
                    if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                        Trace.asyncTraceForTrackEnd(Trace.TRACE_TAG_VIEW, mTrackName, hashCode());
                    }
                    return false;
                }
                mHasWMSync = true;
            }
        }

        ......
    }
```

addLocalSync 这个方法其实就是把 ViewRootImpl 的 mActiveSurfaceSyncGroup 添加到 mWmsRequestSyncGroup，组成一个 SurfaceSyncGroup 树。    

```
    private boolean addLocalSync(ISurfaceSyncGroup childSyncToken, boolean parentSyncGroupMerge) {
        ...
        
        // 通过 this 引用，拿到 ViewRootImpl.mActiveSurfaceSyncGroup
        SurfaceSyncGroup childSurfaceSyncGroup = getSurfaceSyncGroup(childSyncToken);
        if (childSurfaceSyncGroup == null) {
            ......
            return false;
        }

        ......
        // 构建当前节点也就是父节点 mWmsRequestSyncGroup 的回调包装
        ITransactionReadyCallback callback =
                createTransactionReadyCallback(parentSyncGroupMerge);

        if (callback == null) {
            return false;
        }
        // 将子节点插入树中，并传入了父节点 mWmsRequestSyncGroup 的回调包装
        childSurfaceSyncGroup.setTransactionCallbackFromParent(mISurfaceSyncGroup, callback);
        ......
        return true;
    }
```

```
    public ITransactionReadyCallback createTransactionReadyCallback(boolean parentSyncGroupMerge) {
        ...
        // 构建当前回调 mTransactionReadyConsumer 的包装，传递给子节点
        ITransactionReadyCallback transactionReadyCallback =
                new ITransactionReadyCallback.Stub() {
                    @Override
                    public void onTransactionReady(Transaction t) {
                        synchronized (mLock) {
                            if (t != null) {
                                t.sanitize(Binder.getCallingPid(), Binder.getCallingUid());
                                ...
                                //合并 Transaction
                                if (parentSyncGroupMerge) {
                                    t.merge(mTransaction);
                                }
                                mTransaction.merge(t);
                            }
                            mPendingSyncs.remove(this);
                            ....
                            checkIfSyncIsComplete();
                        }
                    }
                };

        synchronized (mLock) {
            if (mSyncReady) {
                ...
                return null;
            }
            // 新构建的 ITransactionReadyCallback 添加到 mPendingSyncs 中
            mPendingSyncs.add(transactionReadyCallback);
            ....
        }

        // 添加超时处理
        addTimeout();

        return transactionReadyCallback;
    }
```

构建了一个 ITransactionReadyCallback 回调，回调中合并了参数 Transaction，将 ITransactionReadyCallback 从 mPendingSyncs 移除，然后调用 checkIfSyncIsComplete。     
ITransactionReadyCallback 构建完成后会添加到成员 mPendingSyncs 中去。也就是说，每添加一个子节点 mPendingSyncs 中就会多一个 ITransactionReadyCallback 对象。   

```
    private void setTransactionCallbackFromParent(ISurfaceSyncGroup parentSyncGroup,
            ITransactionReadyCallback transactionReadyCallback) {
        ...
        addTimeout();

        boolean finished = false;
        Runnable addedToSyncListener = null;
        synchronized (mLock) {
            if (mFinished) {
                finished = true;
            } else {
                // 已有父节点，不进入这里
                if (mParentSyncGroup != null && mParentSyncGroup != parentSyncGroup) {
                    ......
                    try {
                        parentSyncGroup.addToSync(mParentSyncGroup,
                                true /* parentSyncGroupMerge */);
                    } catch (RemoteException e) {
                    }
                }

                ......

                Consumer<Transaction> lastCallback = mTransactionReadyConsumer;
                mParentSyncGroup = parentSyncGroup;
                // 重新定义 子节点的 mTransactionReadyConsumer 回调
                mTransactionReadyConsumer = (transaction) -> {
                    ......
                    // 这里会执行原来的 旧的 mTransactionReadyConsumer
                    lastCallback.accept(null);
                    //  调用父节点的 mTransactionReadyConsumer,通过参数传入
                    try {
                        transactionReadyCallback.onTransactionReady(transaction);
                    } catch (RemoteException e) {
                        transaction.apply();
                    }
                    ....
                };
                addedToSyncListener = mAddedToSyncListener;
            }
        }

        // Invoke the callback outside of the lock when the SurfaceSyncGroup being added was already
        // complete.
        if (finished) {
            try {
                transactionReadyCallback.onTransactionReady(null);
            } catch (RemoteException e) {
            }
        } else if (addedToSyncListener != null) {
            addedToSyncListener.run();
        }
        ......
    }
```

这里使用父节点中重新构建的 ITransactionReadyCallback 对象，重新定义了子节点的 mTransactionReadyConsumer 回调。该回调中会先调用了原来的 mTransactionReadyConsumer 回调，然后调用父节点提供的 ITransactionReadyCallback 回调。         


### 绘制流程

在 View 开始绘制时，为 mActiveSurfaceSyncGroup 注册了 registerRtFrameCallback 回调，该回调会在当前帧绘制完成后调用。    

这里介绍一下 mSyncBuffer 概念，它是 ViewRootImpl 的一个变量，表示

```
//ViewRootImpl.java
    private void registerCallbacksForSync(boolean syncBuffer,
            final SurfaceSyncGroup surfaceSyncGroup) {
        if (!isHardwareEnabled()) {
            return;
        }

        ......

        final Transaction t;
        if (mHasPendingTransactions) {
            t = new Transaction();
            t.merge(mPendingTransaction);
        } else {
            t = null;
        }

        mAttachInfo.mThreadedRenderer.registerRtFrameCallback(new FrameDrawingCallback() {
            @Override
            public void onFrameDraw(long frame) {
            }

            @Override
            public HardwareRenderer.FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                ...
                if (t != null) {
                    mergeWithNextTransaction(t, frame);
                }

                ...
                // syncBuffer一般为false
                if (syncBuffer) {
                    boolean result = mBlastBufferQueue.syncNextTransaction(transaction -> {
                        Runnable timeoutRunnable = () -> Log.e(mTag,
                                "Failed to submit the sync transaction after 4s. Likely to ANR "
                                        + "soon");
                        mHandler.postDelayed(timeoutRunnable, 4000L * Build.HW_TIMEOUT_MULTIPLIER);
                        transaction.addTransactionCommittedListener(mSimpleExecutor,
                                () -> mHandler.removeCallbacks(timeoutRunnable));
                        surfaceSyncGroup.addTransaction(transaction);
                        surfaceSyncGroup.markSyncReady();
                    });
                    if (!result) {
                        // 
                        surfaceSyncGroup.markSyncReady();
                    }
                }

                return didProduceBuffer -> {
                    ...

                    // 处理没有绘制的情况
                    if (!didProduceBuffer) {
                        // 清除事务，以免影响下一次绘制尝试。
                        mBlastBufferQueue.clearSyncTransaction();

                        // 收集发送到 mergeWithNextTransaction 的事务，因为该帧未在此 vsync 上绘制。
                        surfaceSyncGroup.addTransaction(
                                mBlastBufferQueue.gatherPendingTransactions(frame));
                        surfaceSyncGroup.markSyncReady();
                        return;
                    }

                    // 如果我们没有请求同步缓冲区，那么我们将不会收到 syncNextTransaction 回调。相反，只需向 Syncer 报告，以便它知道此同步请求已完成。
                    if (!syncBuffer) {
                        surfaceSyncGroup.markSyncReady();
                    }
                };
            }
        });
    }
```

### 绘制完成

当应用端执行 measure-layout-draw 流程之后，RenderThread 绘制完成就会调用在 registerCallbacksForSync 方法中注册 mActiveSurfaceSyncGroup 的回调函数。    
当绘制完成后，在 onFrameDraw 方法中调用 mActiveSurfaceSyncGroup 的 markSyncReady() 方法。    

```
    public void markSyncReady() {
        .....
        synchronized (mLock) {
            // 由于没有涉及到远程同步的情况，mHasWMSync 为 false
            if (mHasWMSync) {
                try {
                    WindowManagerGlobal.getWindowManagerService().markSurfaceSyncGroupReady(mToken);
                } catch (RemoteException e) {
                }
            }
            // 设置 mSyncReady
            mSyncReady = true;
            checkIfSyncIsComplete();
        }
    }
    
    private void checkIfSyncIsComplete() {
        .....
        // 前面已经设置当前同步组 mSyncReady 为false，
        // 由于mActiveSurfaceSyncGroup是子节点 mPendingSyncs 为空。所以不会走这里。 
        // 前面说过，子节点会调用父节点的 mTransactionReadyConsumer，
        // 在子节点的 Ready 流程中走的父节点的 mTransactionReadyConsumer
        // 如果父节点没有 Ready，那么这里就会返回
        if (!mSyncReady || !mPendingSyncs.isEmpty()) {
            ...
            return;
        }

        ...
        // 执行 mTransactionReadyConsumer
        mTransactionReadyConsumer.accept(mTransaction);
        mFinished = true;
        // 清除超时处理
        if (mTimeoutAdded) {
            mHandler.removeCallbacksAndMessages(this);
        }
    }
```

可以看到，registerCallbacksForSync 注册的 onFrameDraw 回调，设置了 mActiveSurfaceSyncGroup 的 Ready 状态。     
而只有 mWmsRequestSyncGroup 也 Ready，那么整个 SurfaceSyncGroup 树才是 Ready 状态。    
通过前面的流程图，可以看到 mWmsRequestSyncGroup Ready状态的配置在 ViewRootImpl.performTraversals 流程中。    
在 mWmsRequestSyncGroup 的 mTransactionReadyConsumer 回调中：    

```
// ViewRootImpl.java
    private void createSyncIfNeeded() {
        ...
        mWmsRequestSyncGroup = new SurfaceSyncGroup("wmsSync-" + mTag, t -> {
            mWmsRequestSyncGroupState = WMS_SYNC_MERGED;
            ....
            if (mWindowSession instanceof Binder) {
                // ...
                Transaction transactionCopy = new Transaction();
                transactionCopy.merge(t);
                mHandler.postAtFrontOfQueue(() -> reportDrawFinished(transactionCopy, seqId));
            } else {
                // 远程调用 wms ，通知 wms 可以显示图层了
                reportDrawFinished(t, seqId);
            }
        });
    private void reportDrawFinished(@Nullable Transaction t, int seqId) {
        .....
        try {
            mWindowSession.finishDrawing(mWindow, t, seqId);
        } catch (RemoteException e) {
            Log.e(mTag, "Unable to report draw finished", e);
            if (t != null) {
                t.apply();
            }
        } finally {
            if (t != null) {
                t.clear();
            }
        }
    }
```

## 总结

SurfaceSyncGroup 以树的形式组织在一起。    
SurfaceSyncGroup 以此往上回调，只有每个节点都调用了markSyncReady 后，才能调用到根节点的 mTransactionReadyConsumer 回调，进一步调用到 reportDrawFinished 方法，通知 wms 显示图层。    

## 相关文章

[SurfaceSyncGroup 简介](https://juejin.cn/post/7436423794711805978)      
[Android14 Surface 同步机制 SurfaceSyncGroup 实现分析](https://juejin.cn/post/7434179604774322216)     
[探索Surface同步机制](https://blog.csdn.net/weixin_44088874/article/details/134769749)      
[[097]SurfaceSyncGroup的细节](https://cloud.tencent.com/developer/article/2416557)    
[[093]SurfaceSyncer的致命缺陷](https://cloud.tencent.com/developer/article/2368868)
