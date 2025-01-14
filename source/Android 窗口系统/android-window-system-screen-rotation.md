---
title: Android 屏幕旋转流程
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 屏幕旋转流程
date: 2022-11-23 10:00:00
---


## 概述

打开相关 Proto 日志：    

```
adb shell wm logging enable-text WM_DEBUG_ORIENTATION WM_DEBUG_STATES WM_DEBUG_CONFIGURATION
```

相关日志：    
打印冻屏时长：     

```
I WindowManager: Screen frozen for +293ms due to Window{4130042 u0 com.hq.android.androiddemo/com.hq.android.androiddemo.wms.WMSTestActivity}
```
Android U 上屏幕旋转动画是本地动画方式实现的。    
ScreenRotationAnimation：旋转动画管理类。    

本文基于 Android U 分析。

## 流程

首先简单介绍一下旋转屏幕的流程，首先各个界面要进行重绘，在重绘过程中要进行冻屏，只有所有Window都进行绘制完成了才进行转屏。     
初始化 ScreenRotationAnimation 阶段，创建了BackColorSurface、RotationLayer 和 EnterBlackFrameLayer三个图层。
```
SystemSensorManager$SensorEventQueue.dispatchSensorEvent
    WindowOrientationListener$OrientationSensorJudge.onSensorChanged
        WindowOrientationListener$OrientationSensorJudge.finalizeRotation
            DisplayRotation$OrientationListener.onProposedRotationChanged
                WindowManagerService.updateRotation
                    WindowManagerService.updateRotationUnchecked
                        DisplayContent.updateRotationUnchecked
                            DisplayRotation.updateRotationUnchecked
                                DisplayRotation.prepareNormalRotationAnimation
                                    WindowManagerService.startFreezingDisplay
                                        WindowManagerService.startFreezingDisplay
                                            WindowManagerService.doStartFreezingDisplay
                                                // 设置标记位
                                                mDisplayFrozen = true;
                                                new ScreenRotationAnimation
                                                    // 获取当前截图
                                                    screenshotBuffer = ScreenCapture.captureLayers(captureArgs)
                                                    // 创建 BackColorSurface 图层的 SurfaceControl
                                                    mBackColorSurface = displayContent.makeChildSurface(null).setName("BackColorSurface").setColorLayer()
                                                    // 创建 RotationLayer 图层的 SurfaceControl
                                                    mScreenshotLayer = displayContent.makeOverlay().setBLASTLayer().build();
                                                    // 创建 EnterBlackFrameLayer 图层的 SurfaceControl
                                                    mEnterBlackFrameLayer = displayContent.makeOverlay().setName("EnterBlackFrameLayer")..setContainerLayer()
                                                    // 获取 HardwareBuffer
                                                    HardwareBuffer hardwareBuffer = screenshotBuffer.getHardwareBuffer();
                                                    // 绑定
                                                    t.setBuffer(mScreenshotLayer, hardwareBuffer);
                                                    // 显示截图
                                                    t.show(mScreenshotLayer);
                                                    ScreenRotationAnimation.setRotation
                                                        // 计算矩阵
                                                        ScreenRotationAnimation.computeRotationMatrix
                                                        ScreenRotationAnimation.setRotationTransform
                                                            SurfaceControl$Transaction.setMatrix
                                                DisplayContent.setRotationAnimation
                                                    DisplayContent.startAsyncRotationIfNeeded
                                                        DisplayContent.startAsyncRotation
                                                            AsyncRotationController.start

```


```
RemoteDisplayChangeController.createCallback.continueDisplayChange
    RemoteDisplayChangeController.continueDisplayChange
        DisplayRotation.continueRotation
            DisplayContent.sendNewConfiguration
                DisplayContent.updateDisplayOverrideConfigurationLocked
                    ActivityTaskManagerService.updateGlobalConfigurationLocked
                        WindowContainer.onConfigurationChanged
                            ConfigurationContainer.onConfigurationChanged
                                RootWindowContainer.dispatchConfigurationToChild
                                    DisplayContent.performDisplayOverrideConfigUpdate
                                        DisplayContent.onRequestedOverrideConfigurationChanged
                                            DisplayContent.applyRotationAndFinishFixedRotation
                                                DisplayContent.applyRotation
                                                    ScreenRotationAnimation.setRotation
                                                        //计算变换矩阵
                                                        ScreenRotationAnimation.computeRotationMatrix
                                                        ScreenRotationAnimation.setRotationTransform
                                                            SurfaceControl$Transaction.setMatrix
                                                WindowContainer.forAllWindows
                                                    WindowState.seamlesslyRotateIfAllowed
                                                        SeamlessRotator.unrotate
                                                            SeamlessRotator.applyTransform
                                                                SurfaceControl$Transaction.setMatrix
                                                    // 设置正在进行转屏动画
                                                    WindowState.setOrientationChanging(true)
                                                        mOrientationChanging = changing;
                                                        mWmService.mRoot.mOrientationChangeComplete = false;
                                            WindowContainer.onRequestedOverrideConfigurationChanged
                                                ConfigurationContainer.onRequestedOverrideConfigurationChanged
                                                    DisplayContent.onConfigurationChanged
                                                        DisplayArea.onConfigurationChanged
                                                            WindowContainer.onConfigurationChanged
                                                                ConfigurationContainer.onConfigurationChanged
                                                                    ConfigurationContainer.dispatchConfigurationToChild
                                                DisplayContent.onResize()
                                                    WindowContainer.onParentResize()
                                                        WindowState.onResize()
                                                            mWmService.mResizingWindows.add // 后面在 reportResized() 中跨进程更新客户端
                    ActivityTaskManagerService.ensureConfigAndVisibilityAfterUpdate()
                        ActivityRecord.ensureActivityConfiguration()
                            ActivityRecord.scheduleConfigurationChanged()
                                ActivityConfigurationChangeItem.obtain()
                                ClientLifecycleManager.scheduleTransaction()  --> App 端onConfigurationChanged和requestLayout
            ActivityTaskManagerService.continueWindowLayout()
                WindowSurfacePlacer.continueLayout()
                    WindowSurfacePlacer.performSurfacePlacement()
                        WindowSurfacePlacer.performSurfacePlacementLoop()
                            RootWindowContainer.performSurfacePlacement()
                                RootWindowContainer.performSurfacePlacementNoTrace()
                                    RootWindowContainer.handleResizingWindows()
                                        WindowState.reportResized()
                                            IWindow.resized()   -->  App 端 handleResized 和 requestLayout
```

当所有的窗口都已经根据最新的 configration 准备好了，开始进行解冻，触发旋转动画开启。    
创建两个 screen_rotation leash，是在 WindowedMagnification:0:31 这个节点创建的 leash，另外一个是在 Display Overlay 节点为 RotationLayer 做动画的 leash。        
动画也分为两个部分，截图动画也就是 RotationLayer 执行的是退出的动画，Display Overlay 图层执行的是进入的动画。    

```
WindowSurfacePlacer.performSurfacePlacement
    WindowSurfacePlacer.performSurfacePlacementLoop
        RootWindowContainer.performSurfacePlacement
            RootWindowContainer.performSurfacePlacementNoTrace
                // 这里会对 mOrientationChangeComplete 条件进行判断 if (mOrientationChangeComplete) {
                WindowManagerService.stopFreezingDisplayLocked
                    WindowManagerService.doStopFreezingDisplayLocked
                        // 设置标记位
                        mDisplayFrozen = false;
                        ScreenRotationAnimation.dismiss
                            ScreenRotationAnimation.startAnimation
                                // 加载动画资源
                                AnimationUtils.loadAnimation()
                                ScreenRotationAnimation$SurfaceRotationAnimationController.startScreenRotationAnimation
                                    // DisplayContent 动画
                                    ScreenRotationAnimation$SurfaceRotationAnimationController.startDisplayRotation
                                        ScreenRotationAnimation$SurfaceRotationAnimationController.startAnimation
                                            // 本地动画
                                            new LocalAnimationAdapter()
                                            SurfaceAnimator.startAnimation
                                                SurfaceAnimator.startAnimation
                                                    SurfaceAnimator.createAnimationLeash
                                                    // 开始动画
                                                    LocalAnimationAdapter.startAnimation()
                                    // 截图的动画
                                    ScreenRotationAnimation$SurfaceRotationAnimationController.startScreenshotRotationAnimation()
                                        ScreenRotationAnimation$SurfaceRotationAnimationController.startAnimation
                                            // 本地动画
                                            new LocalAnimationAdapter()
                                            SurfaceAnimator.startAnimation
                                                SurfaceAnimator.startAnimation
                                                    SurfaceAnimator.createAnimationLeash
                                                    LocalAnimationAdapter.startAnimation()
```

使用 Winscop 抓取上面创建的三个图层已经两个leash图层在sf图层中的位置如下图：    

<img src="/images/android-window-system-screen-rotation/overlay-leash-1.png" width="882" height="237"/>

<img src="/images/android-window-system-screen-rotation/screen-rotation-leash-1.png" width="904" height="241"/>


Activity 不重建时布局更新流程：      



App:    

```
ActivityConfigurationChangeItem.execute
    ActivityThread.handleActivityConfigurationChanged
        ActivityThread.performConfigurationChangedForActivity
            ActivityThread.performActivityConfigurationChanged
                Activity.onConfigurationChanged()
        ViewRootImpl.updateConfiguration()
            ViewRootImpl.requestLayout
```



App:    

```
ViewRootImpl.W.resized
    ViewRootImpl.dispatchResized
        Handler.sendMessage(MSG_RESIZED_REPORT)
            ViewRootImpl.ViewRootHandler.handleMessageImpl
                ViewRootImpl.handleResized
                    ViewRootImpl.requestLayout()
```


## 流程详解

```
    ScreenRotationAnimation(DisplayContent displayContent, @Surface.Rotation int originalRotation) {
        mService = displayContent.mWmService;
        mContext = mService.mContext;
        mDisplayContent = displayContent;
        final Rect currentBounds = displayContent.getBounds();
        final int width = currentBounds.width();
        final int height = currentBounds.height();

        mPerf = new BoostFramework();

        // Screenshot does NOT include rotation!
        final DisplayInfo displayInfo = displayContent.getDisplayInfo();
        final int realOriginalRotation = displayInfo.rotation;

        mOriginalRotation = originalRotation;
        // If the delta is not zero, the rotation of display may not change, but we still want to
        // apply rotation animation because there should be a top app shown as rotated. So the
        // specified original rotation customizes the direction of animation to have better look
        // when restoring the rotated app to the same rotation as current display.
        final int delta = deltaRotation(originalRotation, realOriginalRotation);
        final boolean flipped = delta == Surface.ROTATION_90 || delta == Surface.ROTATION_270;
        mOriginalWidth = flipped ? height : width;
        mOriginalHeight = flipped ? width : height;
        final int logicalWidth = displayInfo.logicalWidth;
        final int logicalHeight = displayInfo.logicalHeight;
        final boolean isSizeChanged =
                logicalWidth > mOriginalWidth == logicalHeight > mOriginalHeight
                && (logicalWidth != mOriginalWidth || logicalHeight != mOriginalHeight);
        mSurfaceRotationAnimationController = new SurfaceRotationAnimationController();

        // Check whether the current screen contains any secure content.
        boolean isSecure = displayContent.hasSecureWindowOnScreen();
        final int displayId = displayContent.getDisplayId();
        final SurfaceControl.Transaction t = mService.mTransactionFactory.get();

        try {
            final ScreenCapture.ScreenshotHardwareBuffer screenshotBuffer;
            if (isSizeChanged) {
                final DisplayAddress address = displayInfo.address;
                if (!(address instanceof DisplayAddress.Physical)) {
                    Slog.e(TAG, "Display does not have a physical address: " + displayId);
                    return;
                }
                final DisplayAddress.Physical physicalAddress =
                        (DisplayAddress.Physical) address;
                final IBinder displayToken = DisplayControl.getPhysicalDisplayToken(
                        physicalAddress.getPhysicalDisplayId());
                if (displayToken == null) {
                    Slog.e(TAG, "Display token is null.");
                    return;
                }
                // Temporarily not skip screenshot for the rounded corner overlays and screenshot
                // the whole display to include the rounded corner overlays.
                setSkipScreenshotForRoundedCornerOverlays(false, t);
                mRoundedCornerOverlay = displayContent.findRoundedCornerOverlays();
                final ScreenCapture.DisplayCaptureArgs captureArgs =
                        new ScreenCapture.DisplayCaptureArgs.Builder(displayToken)
                                .setSourceCrop(new Rect(0, 0, width, height))
                                .setAllowProtected(true)
                                .setCaptureSecureLayers(true)
                                .setHintForSeamlessTransition(true)
                                .build();
                screenshotBuffer = ScreenCapture.captureDisplay(captureArgs);
            } else {
                ScreenCapture.LayerCaptureArgs captureArgs =
                        new ScreenCapture.LayerCaptureArgs.Builder(
                                displayContent.getSurfaceControl())
                                .setCaptureSecureLayers(true)
                                .setAllowProtected(true)
                                .setSourceCrop(new Rect(0, 0, width, height))
                                .setHintForSeamlessTransition(true)
                                .build();
                screenshotBuffer = ScreenCapture.captureLayers(captureArgs);
            }

            if (screenshotBuffer == null) {
                Slog.w(TAG, "Unable to take screenshot of display " + displayId);
                return;
            }

            // If the screenshot contains secure layers, we have to make sure the
            // screenshot surface we display it in also has FLAG_SECURE so that
            // the user can not screenshot secure layers via the screenshot surface.
            if (screenshotBuffer.containsSecureLayers()) {
                isSecure = true;
            }
            //创建了BackColorSurface的surfaceflinger图层，做背景色变化动画
            mBackColorSurface = displayContent.makeChildSurface(null)
                    .setName("BackColorSurface")
                    .setColorLayer()
                    .setCallsite("ScreenRotationAnimation")
                    .build();
            //创建了RotationLayer的surfaceflinger图层
            String name = "RotationLayer";
            mScreenshotLayer = displayContent.makeOverlay()
                    .setName(name)
                    .setOpaque(true)
                    .setSecure(isSecure)
                    .setCallsite("ScreenRotationAnimation")
                    .setBLASTLayer()
                    .build();
            // This is the way to tell the input system to exclude this surface from occlusion
            // detection since we don't have a window for it. We do this because this window is
            // generated by the system as well as its content.
            InputMonitor.setTrustedOverlayInputInfo(mScreenshotLayer, t, displayId, name);

            mEnterBlackFrameLayer = displayContent.makeOverlay()
                    .setName("EnterBlackFrameLayer")
                    .setContainerLayer()
                    .setCallsite("ScreenRotationAnimation")
                    .build();

            HardwareBuffer hardwareBuffer = screenshotBuffer.getHardwareBuffer();
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER,
                    "ScreenRotationAnimation#getMedianBorderLuma");
            mStartLuma = TransitionAnimation.getBorderLuma(hardwareBuffer,
                    screenshotBuffer.getColorSpace());
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);

            t.setLayer(mScreenshotLayer, SCREEN_FREEZE_LAYER_BASE);
            t.reparent(mBackColorSurface, displayContent.getSurfaceControl());
            // If hdr layers are on-screen, e.g. picture-in-picture mode, the screenshot of
            // rotation animation is an sdr image containing tone-mapping hdr content, then
            // disable dimming effect to get avoid of hdr content being dimmed during animation.
            t.setDimmingEnabled(mScreenshotLayer, !screenshotBuffer.containsHdrLayers());
            t.setLayer(mBackColorSurface, -1);
            t.setColor(mBackColorSurface, new float[]{mStartLuma, mStartLuma, mStartLuma});
            t.setAlpha(mBackColorSurface, 1);
            t.setBuffer(mScreenshotLayer, hardwareBuffer);
            t.setColorSpace(mScreenshotLayer, screenshotBuffer.getColorSpace());
            t.show(mScreenshotLayer);
            t.show(mBackColorSurface);
            hardwareBuffer.close();

            if (mRoundedCornerOverlay != null) {
                for (SurfaceControl sc : mRoundedCornerOverlay) {
                    if (sc.isValid()) {
                        t.hide(sc);
                    }
                }
            }

        } catch (OutOfResourcesException e) {
            Slog.w(TAG, "Unable to allocate freeze surface", e);
        }

        // If display size is changed with the same orientation, map the bounds of screenshot to
        // the new logical display size. Currently pending transaction and RWC#mDisplayTransaction
        // are merged to global transaction, so it can be synced with display change when calling
        // DisplayManagerInternal#performTraversal(transaction).
        if (mScreenshotLayer != null && isSizeChanged) {
            displayContent.getPendingTransaction().setGeometry(mScreenshotLayer,
                    new Rect(0, 0, mOriginalWidth, mOriginalHeight),
                    new Rect(0, 0, logicalWidth, logicalHeight), Surface.ROTATION_0);
        }

        ProtoLog.i(WM_SHOW_SURFACE_ALLOC,
                "  FREEZE %s: CREATE", mScreenshotLayer);
        if (originalRotation == realOriginalRotation) {
            setRotation(t, realOriginalRotation);
        } else {
            // If the given original rotation is different from real original display rotation,
            // this is playing non-zero degree rotation animation without display rotation change,
            // so the snapshot doesn't need to be transformed.
            mCurRotation = realOriginalRotation;
            mSnapshotInitialMatrix.reset();
            setRotationTransform(t, mSnapshotInitialMatrix);
        }
        t.apply();
    }
```

```
// ScreenRotationAnimation.java
    private boolean startAnimation(SurfaceControl.Transaction t, long maxAnimationDuration,
            float animationScale, int finalWidth, int finalHeight, int exitAnim, int enterAnim) {
        if (mScreenshotLayer == null) {
            // Can't do animation.
            return false;
        }
        if (mStarted) {
            return true;
        }

        mStarted = true;

        // Figure out how the screen has moved from the original rotation.
        int delta = deltaRotation(mCurRotation, mOriginalRotation);

        final boolean customAnim;
        if (exitAnim != 0 && enterAnim != 0) {
            customAnim = true;
            mRotateExitAnimation = AnimationUtils.loadAnimation(mContext, exitAnim);
            mRotateEnterAnimation = AnimationUtils.loadAnimation(mContext, enterAnim);
            mRotateAlphaAnimation = AnimationUtils.loadAnimation(mContext,
                    R.anim.screen_rotate_alpha);
        } else {
            customAnim = false;
            switch (delta) { /* Counter-Clockwise Rotations */
                case Surface.ROTATION_0:
                    mRotateExitAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_0_exit);
                    mRotateEnterAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.rotation_animation_enter);
                    break;
                case Surface.ROTATION_90:
                    mRotateExitAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_plus_90_exit);
                    mRotateEnterAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_plus_90_enter);
                    break;
                case Surface.ROTATION_180:
                    mRotateExitAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_180_exit);
                    mRotateEnterAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_180_enter);
                    break;
                case Surface.ROTATION_270:
                    mRotateExitAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_minus_90_exit);
                    mRotateEnterAnimation = AnimationUtils.loadAnimation(mContext,
                            R.anim.screen_rotate_minus_90_enter);
                    break;
            }
        }

        ProtoLog.d(WM_DEBUG_ORIENTATION, "Start rotation animation. customAnim=%s, "
                        + "mCurRotation=%s, mOriginalRotation=%s",
                customAnim, Surface.rotationToString(mCurRotation),
                Surface.rotationToString(mOriginalRotation));

        mRotateExitAnimation.initialize(finalWidth, finalHeight, mOriginalWidth, mOriginalHeight);
        mRotateExitAnimation.restrictDuration(maxAnimationDuration);
        mRotateExitAnimation.scaleCurrentDuration(animationScale);
        mRotateEnterAnimation.initialize(finalWidth, finalHeight, mOriginalWidth, mOriginalHeight);
        mRotateEnterAnimation.restrictDuration(maxAnimationDuration);
        mRotateEnterAnimation.scaleCurrentDuration(animationScale);

        mAnimRunning = false;
        mFinishAnimReady = false;
        mFinishAnimStartTime = -1;

        if (customAnim) {
            mRotateAlphaAnimation.restrictDuration(maxAnimationDuration);
            mRotateAlphaAnimation.scaleCurrentDuration(animationScale);
        }

        if (customAnim && mEnteringBlackFrame == null) {
            try {
                Rect outer = new Rect(-finalWidth, -finalHeight,
                        finalWidth * 2, finalHeight * 2);
                Rect inner = new Rect(0, 0, finalWidth, finalHeight);
                mEnteringBlackFrame = new BlackFrame(mService.mTransactionFactory, t, outer, inner,
                        SCREEN_FREEZE_LAYER_BASE, mDisplayContent, false, mEnterBlackFrameLayer);
            } catch (OutOfResourcesException e) {
                Slog.w(TAG, "Unable to allocate black surface", e);
            }
        }

        if (customAnim) {
            mSurfaceRotationAnimationController.startCustomAnimation();
        } else {
            mSurfaceRotationAnimationController.startScreenRotationAnimation();
        }

        return true;
    }
```

### relaunch 相关

如果我们在 AndroidManifest 里面配置Activity 的下面属性：     

```
android:configChanges="orientation|screenSize"
```

那么在旋转屏幕时就不会进行 Activity 的销毁和重建，那么流程上的差异主要在：      


```
// ActivityRecord.java
    boolean ensureActivityConfiguration(int globalChanges, boolean preserveWindow,
            boolean ignoreVisibility, boolean isRequestedOrientationChanged) {
            ......
        boolean shouldRelaunchLocked = shouldRelaunchLocked(changes, mTmpConfig);
        if (!DeviceIntegrationUtils.DISABLE_DEVICE_INTEGRATION) {
            shouldRelaunchLocked &= !mAtmService.mRemoteTaskManager.shouldIgnoreRelaunch(task, displayChanged,
                    lastReportDisplayID, newDisplayId, changes);
        }
        if (shouldRelaunchLocked || forceNewConfig) {
        ......
                relaunchActivityLocked(preserveWindow);
            return false;
        }
        
        if (displayChanged) {
            scheduleActivityMovedToDisplay(newDisplayId, newMergedOverrideConfig);
        } else {
            scheduleConfigurationChanged(newMergedOverrideConfig);
        }

// 是否需要 reLaunche Activity
    private boolean shouldRelaunchLocked(int changes, Configuration changesConfig) {
        int configChanged = info.getRealConfigChanged();
        boolean onlyVrUiModeChanged = onlyVrUiModeChanged(changes, changesConfig);

        // Override for apps targeting pre-O sdks
        // If a device is in VR mode, and we're transitioning into VR ui mode, add ignore ui mode
        // to the config change.
        // For O and later, apps will be required to add configChanges="uimode" to their manifest.
        if (info.applicationInfo.targetSdkVersion < O
                && requestedVrComponent != null
                && onlyVrUiModeChanged) {
            configChanged |= CONFIG_UI_MODE;
        }
        // TODO(b/274944389): remove workaround after long-term solution is implemented
        // Don't restart due to desk mode change if the app does not have desk resources.
        if (mWmService.mSkipActivityRelaunchWhenDocking && onlyDeskInUiModeChanged(changesConfig)
                && !hasDeskResources()) {
            configChanged |= CONFIG_UI_MODE;
        }

        return (changes&(~configChanged)) != 0;
    }

// 创建 ActivityConfigurationChangeItem
    private void scheduleConfigurationChanged(Configuration config) {
        if (!attachedToProcess()) {
            ProtoLog.w(WM_DEBUG_CONFIGURATION, "Can't report activity configuration "
                    + "update - client not running, activityRecord=%s", this);
            return;
        }
        try {
            ProtoLog.v(WM_DEBUG_CONFIGURATION, "Sending new config to %s, "
                    + "config: %s", this, config);

            mAtmService.getLifecycleManager().scheduleTransaction(app.getThread(), token,
                    ActivityConfigurationChangeItem.obtain(config));
        } catch (RemoteException e) {
            // If process died, whatever.
        }
    }
```
