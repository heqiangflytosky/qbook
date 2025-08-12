---
title: Android SyncTransactionQueue
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 SyncTransactionQueue
date: 2022-11-23 10:00:00
---


## 概述

前面文章介绍了应用层向系统发送 WindowContainerTransaction 的两种方法，其中一种是可以通过 SyncTransactionQueue 来发送，本文着重介绍一下 SyncTransactionQueue 的机制。    

## 流程分析

### 构建 WindowContainerTransaction

当应用侧需要修改 WindowContainer 属性的时候，就需要构建WindowContainerTransaction对象并设置其参数，然后发送给系统侧，然后在系统侧完成对WindowContainer的修改。    

### 使用 SyncTransactionQueue 发送

```
        final WindowContainerTransaction wct = new WindowContainerTransaction();
        // wct do something
        // 把 WindowContainerTransaction 放到队列中
        mSyncQueue.queue(wct);
        // 放入一个回调，如果 wct 中的变更绘制完成，就会调用 st 回调。
        mSyncQueue.runInSync(st -> {})
```

构建完 WindowContainerTransaction 后，如果使用 SyncTransactionQueue 来发送，那么就需要把将wct放入SyncTransactionQueue的对象mSyncQueue之中。       

```
    public void queue(WindowContainerTransaction wct) {
        if (wct.isEmpty()) {
            if (DEBUG) Slog.d(TAG, "Skip queue due to transaction change is empty");
            return;
        }
        // 将 WCT 包装进了 SyncCallback 中
        SyncCallback cb = new SyncCallback(wct);
        synchronized (mQueue) {
            if (DEBUG) Slog.d(TAG, "Queueing up " + wct);
            mQueue.add(cb);
            if (mQueue.size() == 1) {
                // 发送 WCT
                cb.send();
            }
        }
    }
```

其实 SyncCallback.send() 的本质还是调用 WindowOrganizer 的 applySyncTransaction 方法来实现 WindowContainerTransaction 的传输。    

```
        void send() {
            .....
            mInFlight = this;
            ......
            if (mLegacyTransition != null) {
                mId = new WindowOrganizer().startLegacyTransition(mLegacyTransition.getType(),
                        mLegacyTransition.getAdapter(), this, mWCT);
            } else {
                mId = new WindowOrganizer().applySyncTransaction(mWCT, this);
            }
            if (DEBUG) Slog.d(TAG, " Sent sync transaction. Got id=" + mId);
            mMainExecutor.executeDelayed(mOnReplyTimeout, REPLY_TIMEOUT);
        }
```

### WMCore 执行 applySyncTransaction

关于 WindowOrganizerController.applySyncTransaction 前面已经介绍过，这里就不再详细介绍。     
WindowOrganizerController 实现了 BLASTSyncEngine.TransactionReadyListener。     
```
class WindowOrganizerController extends IWindowOrganizerController.Stub
        implements BLASTSyncEngine.TransactionReadyListener {
```

```
WindowOrganizerController.applySyncTransaction
    WindowOrganizerController.prepareSyncWithOrganizer
        BLASTSyncEngine.prepareSyncSet
            new BLASTSyncEngine$SyncGroup
```


当完成对 WindowContainer 相关属性变更的绘制后，就会调用 WindowOrganizerController.onTransactionReady 方法。    

```
RootWindowContainer.performSurfacePlacement
    RootWindowContainer.performSurfacePlacementNoTrace
        BLASTSyncEngine.onSurfacePlacement
            BLASTSyncEngine$SyncGroup.tryFinish
                BLASTSyncEngine$SyncGroup.finishNow
                    WindowOrganizerController.onTransactionReady
                        
```


### onTransactionReady


```
SyncTransactionQueue$SyncCallbac.onTransactionReady
    SyncTransactionQueue.onTransactionReceived
```

在执行 mSyncQueue.runInSync 时 传入了一个 TransactionRunnable。会保存在 mRunnables 中。      

```
    public void runInSync(TransactionRunnable runnable) {
        synchronized (mQueue) {
            if (DEBUG) Slog.d(TAG, "Run in sync. mInFlight=" + mInFlight);
            if (mInFlight != null) {
                mRunnables.add(runnable);
                return;
            }
        }
        SurfaceControl.Transaction t = mTransactionPool.acquire();
        runnable.runWithTransaction(t);
        t.apply();
        mTransactionPool.release(t);
    }
```

当 onTransactionReady 时，就会调用保存在 mRunnables 中的 TransactionRunnable。     

```
    private void onTransactionReceived(@NonNull SurfaceControl.Transaction t) {
        if (DEBUG) Slog.d(TAG, "  Running " + mRunnables.size() + " sync runnables");
        final int n = mRunnables.size();
        for (int i = 0; i < n; ++i) {
            mRunnables.get(i).runWithTransaction(t);
        }
        // More runnables may have been added, so only remove the ones that ran.
        mRunnables.subList(0, n).clear();
    }
```

## 相关文章

[WCT系列（二）：SyncTransactionQueue 类详解](https://blog.csdn.net/weixin_39702448/article/details/141305422)      
[2【Android 12】【WCT的发送】SyncTransactionQueue](https://blog.csdn.net/ukynho/article/details/126747799)      
