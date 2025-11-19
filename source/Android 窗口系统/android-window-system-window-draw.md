---
title: Android 窗口系统-窗口绘制
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口绘制
date: 2022-11-23 10:00:00
---



## 概述

窗口的绘制主要在 App 端，App 绘制完成后，将状态同步给 WMS，然后就将绘制指令发给 SurfaceFlinger，进行合成。     
前面的文章中我们介绍到，App 在 relayout 时，会把 ViewRootImpl 创建的 SurfaceControl 传到 WMS，然后 WMS 会对 SurfaceControl 重新赋值回传给 App，App拿到 SurfaceControl 对象后就可以构建 BLASTBufferQueue 和 Surface，建立和 SurfaceFlinger 的联系，后面 App 就可以进行绘制流程了。    


## Surface 的创建

```
ViewRootImpl.relayoutWindow
  IWindowSession.relayout
    // ----> wms
      WindowManagerService.relayoutWindow
        WindowManagerService.createSurfaceControl
          WindowStateAnimator.createSurfaceLocked
            new SurfaceControl
          WindowSurfaceController.getSurfaceControl
            SurfaceControl.copyFrom
  ViewRootImpl.updateBlastSurfaceIfNeeded()
    new BLASTBufferQueue()
    BlastBufferQueue.update
    mSurface=BlastBufferQueue.createSurface()
```

## 绘制流程

详细介绍参考 [Android Surface 同步机制 SurfaceSyncGroup](https://www.heqiangfly.com/qbook/source/Android%20%E7%AA%97%E5%8F%A3%E7%B3%BB%E7%BB%9F/android-window-system-surface-sync-group.html)

```
ViewRootImpl.performTraversals
  // 创建 getOrCreateSurfaceSyncGroup
  ViewRootImpl.createSyncIfNeeded
  ViewRootImpl.relayoutWindow()
    ......
  // 开始绘制
  ViewRootImpl.performDraw(mActiveSurfaceSyncGroup)
    ViewRootImpl.draw
      if (isHardwareEnabled())
        ThreadedRenderer.draw()
      else
        ViewRootImpl.drawSoftware()
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
  SurfaceSyncGroup.markSyncReady
    SurfaceSyncGroup.checkIfSyncIsComplete
      mTransactionReadyConsumer.accept()
        transactionReadyConsumer.accept()
          ViewRootImpl.reportDrawFinished()
            WindowSession.finishDrawing()
              // ----> WMS
              WindowManagerService.finishDrawingWindow()
                WindowState.finishDrawing
                  WindowStateAnimator.finishDrawingLocked()
                    // 修改 mDrawState 状态为 COMMIT_DRAW_PENDING
                    mDrawState = COMMIT_DRAW_PENDING;
```


