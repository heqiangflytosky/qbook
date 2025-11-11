---
title: Android 窗口更新 Input 信息
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: Android 窗口更新 Input 信息
date: 2022-11-23 10:00:00
---


我们在前面介绍 Input 系统时提到过，SurfaceFlinger 会同步一个窗口列表信息同步至 InputDispatcher ，那么 WMS 又会更新窗口的信息到 SurfaceFlinger，本文就主要介绍一下窗口是如何更新 Input 的相关信息到 SurfaceFlinger。      


## InputMonitor

```
final class InputMonitor {
    ......
    void updateInputWindowsLw(boolean force) {
        if (!force && !mUpdateInputWindowsNeeded) {
            return;
        }
        scheduleUpdateInputWindows();
    }
    
    private void scheduleUpdateInputWindows() {
        if (mDisplayRemoved) {
            return;
        }

        if (!mUpdateInputWindowsPending) {
            mUpdateInputWindowsPending = true;
            mHandler.post(mUpdateInputWindows);
        }
    }
    
    private class UpdateInputWindows implements Runnable {
        @Override
        public void run() {
            synchronized (mService.mGlobalLock) {
                mUpdateInputWindowsPending = false;
                mUpdateInputWindowsNeeded = false;

                if (mDisplayRemoved) {
                    return;
                }

                // Populate the input window list with information about all of the windows that
                // could potentially receive input.
                // As an optimization, we could try to prune the list of windows but this turns
                // out to be difficult because only the native code knows for sure which window
                // currently has touch focus.

                // If there's a drag in flight, provide a pseudo-window to catch drag input
                final boolean inDrag = mService.mDragDropController.dragDropActiveLocked();

                // Add all windows on the default display.
                mUpdateInputForAllWindowsConsumer.updateInputWindows(inDrag);
            }
        }
    }
    
    private final UpdateInputWindows mUpdateInputWindows = new UpdateInputWindows();
    
    

}
```

InputMonitor.updateInputWindowsLw 的地方非常多，下面就列举几个常见的时机：     

```
RootWindowContainer.performSurfacePlacementNoTrace:945
  RootWindowContainer.forAllDisplays:1269
    InputMonitor.updateInputWindowsLw:348
```

```
TaskFragment.startPausing
  ActivityRecord.pauseKeyDispatchingLocked
    InputMonitor.pauseDispatchingLw
      InputMonitor.updateInputWindowsLw
```

```
WindowManagerService.addWindow
  InputMonitor.updateInputWindowsLw
```

```
DisplayContent.updateFocusedWindowLocked
  InputMonitor.setInputFocusLw
    InputMonitor.updateInputWindowsLw
```

```
ActivityRecord.commitVisibility
  InputMonitor.updateInputWindowsLw
```

```
WindowState.removeImmediately
  WindowManagerService.postWindowRemoveCleanupLocked
    InputMonitor.updateInputWindowsLw
```


## updateInputWindows 介绍

```
InputMonitor.updateInputWindowsLw
  InputMonitor.scheduleUpdateInputWindows()
    UpdateInputWindows.run()
      UpdateInputForAllWindowsConsumer.updateInputWindows
        DisplayContent.forAllWindows()
          UpdateInputForAllWindowsConsumer.accept()
            // 设置  Input Consumer recents_animation_input_consumer
            mRecentsAnimationInputConsumer.reparent
            mRecentsAnimationInputConsumer.show
            // 设置 Input Consumer pip_window_input_consumer
            mPipInputConsumer.layout
            mPipInputConsumer.reparent
            mPipInputConsumer.show
            // 设置 Input Consumer wallpaper_input_consumer
            mWallpaperInputConsumer.show
            // 设置 input 信息，把WindowState转换成inputWindowHandle
            InputMonitor.populateInputWindowHandle
            // 向 SF 设置 InputWindowInfo
            InputMonitor.setInputWindowInfoIfNeeded
              InputWindowHandleWrapper.applyChangesToSurface
                SurfaceControl.Transaction.setInputWindowInfo
```

Input Consumer pip_window_input_consumer 和  Input Consumer recents_animation_input_consumer 为系统预置的图层：      

<img src="/images/android-window-system-update-input-info/input_consumers.png" width="509" height="133"/>


## WindowState 转换为 InputWindowHandle

InputMonitor.populateInputWindowHandle 用来把WindowState转换成inputWindowHandle信息。    


```
    void populateInputWindowHandle(final InputWindowHandleWrapper inputWindowHandle,
            final WindowState w) {
        // Add a window to our list of input windows.
        inputWindowHandle.setInputApplicationHandle(w.mActivityRecord != null
                ? w.mActivityRecord.getInputApplicationHandle(false /* update */) : null);
        inputWindowHandle.setToken(w.mInputChannelToken);
        inputWindowHandle.setDispatchingTimeoutMillis(w.getInputDispatchingTimeoutMillis());
        inputWindowHandle.setTouchOcclusionMode(w.getTouchOcclusionMode());
        inputWindowHandle.setPaused(w.mActivityRecord != null && w.mActivityRecord.paused);
        inputWindowHandle.setWindowToken(w.mClient.asBinder());

        inputWindowHandle.setName(w.getName());

        // Update layout params flags to force the window to be not touch modal. We do this to
        // restrict the window's touchable region to the task even if it requests touches outside
        // its window bounds. An example is a dialog in primary split should get touches outside its
        // window within the primary task but should not get any touches going to the secondary
        // task.
        int flags = w.mAttrs.flags;
        if (w.mAttrs.isModal()) {
            flags = flags | FLAG_NOT_TOUCH_MODAL;
        }
        inputWindowHandle.setLayoutParamsFlags(flags);
        inputWindowHandle.setInputConfigMasked(
                InputConfigAdapter.getInputConfigFromWindowParams(
                        w.mAttrs.type, flags, w.mAttrs.inputFeatures),
                InputConfigAdapter.getMask());

        final boolean focusable = w.canReceiveKeys()
                && (mDisplayContent.hasOwnFocus() || mDisplayContent.isOnTop());
        inputWindowHandle.setFocusable(focusable);

        final boolean hasWallpaper = mDisplayContent.mWallpaperController.isWallpaperTarget(w)
                && !mService.mPolicy.isKeyguardShowing()
                && w.mAttrs.areWallpaperTouchEventsEnabled();
        inputWindowHandle.setHasWallpaper(hasWallpaper);

        // Surface insets are hardcoded to be the same in all directions
        // and we could probably deprecate the "left/right/top/bottom" concept.
        // we avoid reintroducing this concept by just choosing one of them here.
        inputWindowHandle.setSurfaceInset(w.mAttrs.surfaceInsets.left);

        // If we are scaling the window, input coordinates need to be inversely scaled to map from
        // what is on screen to what is actually being touched in the UI.
        inputWindowHandle.setScaleFactor(w.mGlobalScale != 1f ? (1f / w.mGlobalScale) : 1f);

        boolean useSurfaceBoundsAsTouchRegion = false;
        SurfaceControl touchableRegionCrop = null;
        final Task task = w.getTask();
        if (task != null) {
            if (task.isOrganized() && task.getWindowingMode() != WINDOWING_MODE_FULLSCREEN
                    //Flyme|Core-Framework|wangchende@meizu.com|windowmode {@
                    && !task.isMzMiniWindow() && !task.isMzMagicWindow()
                    //@}
            ) {
                // If the window is in a TaskManaged by a TaskOrganizer then most cropping will
                // be applied using the SurfaceControl hierarchy from the Organizer. This means
                // we need to make sure that these changes in crop are reflected in the input
                // windows, and so ensure this flag is set so that the input crop always reflects
                // the surface hierarchy. However, we only want to set this when the client did
                // not already provide a touchable region, so that we don't ignore the one provided.
                if (w.mTouchableInsets != TOUCHABLE_INSETS_REGION) {
                    useSurfaceBoundsAsTouchRegion = true;
                }

                if (w.mAttrs.isModal()) {
                    TaskFragment parent = w.getTaskFragment();
                    touchableRegionCrop = parent != null ? parent.getSurfaceControl() : null;
                }
            } else if (task.cropWindowsToRootTaskBounds() && !w.inFreeformWindowingMode()) {
                touchableRegionCrop = task.getRootTask().getSurfaceControl();
            }
        }
        inputWindowHandle.setReplaceTouchableRegionWithCrop(useSurfaceBoundsAsTouchRegion);
        inputWindowHandle.setTouchableRegionCrop(touchableRegionCrop);

        if (!useSurfaceBoundsAsTouchRegion) {
            w.getSurfaceTouchableRegion(mTmpRegion, w.mAttrs);
            inputWindowHandle.setTouchableRegion(mTmpRegion);
        }
    }
```

## native 部分流程

```
SurfaceControl.Transaction.setInputWindowInfo
  nativeSetInputWindowInfo
    //拿java层的InputWindowHandle构造NativeInputWindowHandle
    sp<NativeInputWindowHandle> handle = android_view_InputWindowHandle_getHandle
    // 把java层对象数据进行拷贝到我们的native对象NativeInputWindowHandle中的mInfo中
    NativeInputWindowHandle::updateInfo()
    // 对应的WindowInfo数据传递给对应的layer_state_t,其实对应就是Layer
    SurfaceComposerClient::Transaction::setInputWindowInfo()
    
```

```
SurfaceComposerClient::Transaction::apply
  // 跨进程调用到 SF
  SurfaceFlinger::setTransactionState
```

## SF 部分流程


```
SurfaceFlinger::setTransactionState
  // 把TransactionState放入队列，且会启动对应的scheduleCommit
  SurfaceFlinger::queueTransaction
```

```
SurfaceFlinger::updateInputFlinger()
    SurfaceFlinger::buildWindowInfos()
    WindowInfosListenerInvoker::windowInfosChanged()
```

后面的流程就是传递给 InputDispatch 了，这部分在 Input 相关文章中有介绍。    

