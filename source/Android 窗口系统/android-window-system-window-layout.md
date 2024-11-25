---
title: Android 窗口系统-窗口布局
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口布局
date: 2022-11-23 10:00:00
---

在窗口布局这里主要做的事情有：创建 SurfaceControl、配置窗口属性、窗口尺寸的计算以及Surface的状态变更等。    

## App 端

App 侧在创建ViewRootImpl对象后，会向 WMS 申请窗口布局的     

```
// app 侧
ActivityThread.handleResumeActivity()
　　Activity.getWindow() //获取activity的PhoneWindow
    PhoneWindow.getDecorView()//获取PhoneWindow的DecorView
    Activity.getWindowManager()//获取Activity的WindowManagerImpl
    WindowManagerImpl.addView(decor, l)//通过WindowManagerImpl将DecorView添加到Activity
        WindowManagerGlobal.addView(DecorView)
            new ViewRootImpl()//创建ViewRootImpl对象
            ViewRootImpl.setView()
                ViewRootImpl.requestLayout()
                    ViewRootImpl.scheduleTraversals()
                        Choreographer.postCallback(mTraversalRunnable)
                            ViewRootImpl.doTraversal()
                                ViewRootImpl.performTraversals()
                                    ViewRootImpl.relayoutWindow()
                                        ViewRootImpl$TraversalRunnable.run
                                            ViewRootImpl.performTraversals
                                                ViewRootImpl.relayoutWindow
                                                    WindowSession.relayout
```

## Server 端

```
Session.relayout
    WindowManagerService.relayoutWindow
        WindowManagerService.windowForClientLocked
            // 从mWindowMap中获取对应的 WindowState
            WindowState win = mWindowMap.get(client)
        // 设置窗口大小
        WindowState.setRequestedSize
            mLayoutNeeded = true;
            mRequestedWidth = requestedWidth;
            mRequestedHeight = requestedHeight;
        // 调整特殊类型的Window#attrs属性
        DisplayPolicy.adjustWindowParamsLw
        // 调整 mLayoutNeeded 属性
        WindowState.mLayoutNeeded = true;
        // 设置窗口缩放
        WindowState.setWindowScale
        // 设置窗口可见性
        WindowState.setViewVisibility
        WindowState.setDisplayLayoutNeeded()
            DisplayContent.setLayoutNeeded()
                mLayoutNeeded = true
        // 调整壁纸窗口属性
        WallpaperController.adjustWallpaperWindows()
        // 创建 SurfaceControl
        WindowManagerService.createSurfaceControl()
            WindowStateAnimator.createSurfaceLocked()
                WindowStateAnimator.resetDrawState()
                    mDrawState = DRAW_PENDING;
                new WindowSurfaceController
                    new SurfaceControl
        // 窗口尺寸的计算以及Surface的状态变更
        WindowSurfacePlacer.performSurfacePlacement
            // 窗口布局循环的逻辑
            WindowSurfacePlacer.performSurfacePlacementLoop()
                RootWindowContainer.performSurfacePlacementNoTrace()
                    // 更新焦点
                    WindowManagerService.updateFocusedWindowLocked
                    // 执行窗口尺寸计算，surface状态变更等操作
                    RootWindowContainer.applySurfaceChangesTransaction()
                        // 水印、StrictMode警告框以及模拟器显示的布局
                        Watermark.positionSurface
                        StrictModeFlash.positionSurface
                        EmulatorDisplayOverlay.positionSurface
                        //历所有窗口，计算窗口的布局大小
                        DisplayContent.applySurfaceChangesTransaction()
                            DisplayContent.performLayout
                                DisplayContent.performLayoutNoTrace()
                                    WindowContainer.forAllWindows
                                        WindowState.forAllWindows
                                            WindowState.applyInOrderWithImeWindows
                                                DisplayContent.mPerformLayout.run
                                                    //计算窗口位置大小
                                                    DisplayPolicy.layoutWindowLw()
                                                        //计算窗口布局大小
                                                        WindowLayout.computeFrames()
                                                        //将计算的布局参数赋值给windowFrames
                                                        WindowState.setFrames
                            // 遍历所有窗口，改变surface的状态。
                            WindowContainer.forAllWindows
                                DisplayContent.mApplySurfaceChangesTransaction
                                    WindowStateAnimator.commitFinishDrawingLocked()
                                        mDrawState = READY_TO_SHOW;
                                        // 某些情况下更新 mDrawState = HAS_DRAWN，具体见下文
                                        WindowState.performShowLocked()
                                            mWinAnimator.mDrawState = HAS_DRAWN;
                            // 处理各个surface的位置、大小以及是否要在屏幕上显示等。
                            DisplayContent.prepareSurfaces()
                                DisplayArea.Dimmable.prepareSurfaces()
                                    WindowContainer.prepareSurfaces()
                                        WindowState.prepareSurfaces()
                                            WindowStateAnimator.prepareSurfaceLocked
                                                判断 w.isOnScreen()
                                                // 上屏显示
                                                WindowSurfaceController.showRobustly()
                                                    // 显示
                                                    SurfaceControl.Transaction.show()
                    // 将Surface状态变更为HAS_DRAWN，触发App触发动画。
                    RootWindowContainer.checkAppTransitionReady()
                        AppTransitionController.handleAppTransitionReady()
                            AppTransitionController.applyAnimations()
                            AppTransitionController.handleClosingApps()
                            AppTransitionController.handleOpeningApps()
                                ActivityRecord.showAllWindowsLocked()
                                    WindowContainer.forAllWindows()
                                        WindowState.forAllWindows()
                                            WindowState.applyInOrderWithImeWindows
                                                WindowState..performShowLocked()
                                                    // 进入窗口动画流程
                                                    WindowStateAnimator.applyEnterAnimationLocked()
                                                    //更新 mDrawState = HAS_DRAWN
                                                    mDrawState = HAS_DRAWN
                                                    WindowManagerService.scheduleAnimationLocked()
                    // 如果过程中size或者位置变化，则通知客户端重新relayout
                    RootWindowContainer.handleResizingWindows()
                        WindowState.reportResized()
                            mClient.resized()
        // 填充计算好的frame返回给客户端，更新mergedConfiguration对象
        WindowState.fillClientWindowFramesAndConfiguration
```

先来解释一下通过 Session.relayout 方法从客户端传递过来的参数：     

```
client：是WMS与客户端通信的Binder。
attrs：窗口的布局属性，根据attrs提供的属性来布局窗口。
requestWidth、requestHeight：客户端请求的窗口尺寸。
viewFlags：窗口的可见性。包括VISIBLE（0，view可见），INVISIBLE（4，view不可见，但是仍然占用布局空间）GONE（8，view不可见，不占用布局空间）
flags：定义一些布局行为。
outFrames：返回给客户端的，保存了重新布局之后的位置与大小。
mergedConfiguration:相关配置信息。
outSurfaceControl:返回给客户端的surfaceControl。
outInsetsState：用来保存系统中所有Insets的状态。
outActiveControls：InSetsSourceControl数组。
outSyncSeqIdBundle：与布局同步有关。

```

Session调用 WMS.relayoutWindow 将客户端传入的参数传递给WMS。    

### relayoutWindow

这里主要做了一下的工作：

 - 根据客户端传过来的Iwindow从mWindowMap中获取对应的WindowState（在添加窗口阶段创建）
 - 设置 WindowState.mLayoutNeeded 以及shouldRelayout标志位
 - 创建 Surface的流程。
 - 窗口尺寸的计算以及Surface的状态变更。

```
// server 侧
// WindowManagerService.java

    public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
            int lastSyncSeqId, ClientWindowFrames outFrames,
            MergedConfiguration outMergedConfiguration, SurfaceControl outSurfaceControl,
            InsetsState outInsetsState, InsetsSourceControl.Array outActiveControls,
            Bundle outSyncIdBundle) {
            ......
        synchronized (mGlobalLock) {
            //根据客户端传过来的Iwindow从mWindowMap中获取对应的WindowState
            final WindowState win = windowForClientLocked(session, client, false);
            if (win == null) {
                return 0;
            }
            ......
            final DisplayContent displayContent = win.getDisplayContent();
            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();

            WindowStateAnimator winAnimator = win.mWinAnimator;
            if (viewVisibility != View.GONE) {
                //根据客户端请求的窗口大小设置WindowState的requestedWidth, requestedHeight
                //并设置 mLayoutNeeded = true;
                win.setRequestedSize(requestedWidth, requestedHeight);
            }
            ......
            // 设置窗口缩放
            win.setWindowScale(win.mRequestedWidth, win.mRequestedHeight);
            ......
            win.mRelayoutCalled = true;
            win.mInRelayout = true;
            // 设置窗口可见性
            win.setViewVisibility(viewVisibility);
            ......
            // 判断是否允许relayout，此时为true
            final boolean shouldRelayout = viewVisibility == View.VISIBLE &&
                    (win.mActivityRecord == null || win.mAttrs.type == TYPE_APPLICATION_STARTING
                            || win.mActivityRecord.isClientVisible());
            .......
            // 创建 surface
            if (shouldRelayout && outSurfaceControl != null) {
                try {
                    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
                } catch (Exception e) {
                    ......
                    return 0;
                }
            }
            //窗口尺寸的计算以及Surface的状态变更
            //WindowSurfacePlacer在WMS初始化的时候创建
            mWindowPlacerLocked.performSurfacePlacement(true /* force */);
            
            if (outFrames != null && outMergedConfiguration != null) {
                //填充计算好的frame返回给客户端，更新mergedConfiguration对象
                win.fillClientWindowFramesAndConfiguration(outFrames, outMergedConfiguration,
                        false /* useLatestConfig */, shouldRelayout);

                win.onResizeHandled();
            }
```

### createSurfaceControl

在客户端ViewRootImpl中，有一个 Surface 对象，它是这个ViewRootImpl中View显示内容的最终呈现者。     
在应用进程 ViewRootImpl 在创建 Surface 和 SurfaceControl 对象时，仅仅是一个空壳，什么都没做，然后在relayout()时传递 SurfaceControl 到WMS进程，WMS中会创建一个 SurfaceControl 对象，并将它复制给传入的 SurfaceControl，最终 ViewRootImpl 从 SurfaceControl 中获取内容。    

```
// ViewRootImpl.java
    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();
```

关于SurfaceControl的创建在WMS中主要做两件事：    

 - 调用 WindwoStateAnimator 执行具体的 SurfaceControl 的创建。
 - 将创建的 SurfaceControl 赋值给客户端的 outSurfaceControl。

```
    private int createSurfaceControl(SurfaceControl outSurfaceControl, int result,
            WindowState win, WindowStateAnimator winAnimator) {
        if (!win.mHasSurface) {
            result |= RELAYOUT_RES_SURFACE_CHANGED;
        }

        WindowSurfaceController surfaceController;
        try {
            //WMS将创建的surfaceContorl的操作交给windowAnimator来处理
            surfaceController = winAnimator.createSurfaceLocked();
        ......
        if (surfaceController != null) {
            //将WMS的SurfaceControl赋值给客户端的outSurfaceControl
            surfaceController.getSurfaceControl(outSurfaceControl);
            ......

        return result;
    }
```

在WindowStateAnimator中创建SurfaceControl主要经过以下三个步骤：    

 1.重置Surface标志位，变更mDrawState状态为DRAW_PENDING。
 2.通过实例化WindowSurfaceController来创建SurfaceControl。
 3.处理Surface标志位，将其置为true，标志着当前WindowState已经有surface了。


```
// WindowStateAnimator.java
    WindowSurfaceController createSurfaceLocked() {
        final WindowState w = mWin;
        //首先判断是否存在mSurfaceController
        if (mSurfaceController != null) {
            return mSurfaceController;
        }
        //设置WindowState的mHasSurface设置为false
        w.setHasSurface(false);
        //将WindowStateAnimator中的DrawState设置为DRAW_PENDING
        resetDrawState();

        mService.makeWindowFreezingScreenIfNeededLocked(w);
        //将surface创建flag设置为hidden
        int flags = SurfaceControl.HIDDEN;
        //获取windowState的布局参数
        final WindowManager.LayoutParams attrs = w.mAttrs;

        // Device Integration: This is to make screenshot not include our black screen
        if ((mWin.mAttrs.privateFlags & PRIVATE_FLAG_IS_ROUNDED_CORNERS_OVERLAY) != 0
                || (!DeviceIntegrationUtils.DISABLE_DEVICE_INTEGRATION
                && attrs.type == TYPE_SYSTEM_BLACKSCREEN_OVERLAY)) {
            flags |= SurfaceControl.SKIP_SCREENSHOT;
        }

        // Set up surface control with initial size.
        try {

            // This can be removed once we move all Buffer Layers to use BLAST.
            final boolean isHwAccelerated = (attrs.flags & FLAG_HARDWARE_ACCELERATED) != 0;
            final int format = isHwAccelerated ? PixelFormat.TRANSLUCENT : attrs.format;
            //创建WindowSurfaceController
            mSurfaceController = new WindowSurfaceController(attrs.getTitle().toString(), format,
                    flags, this, attrs.type);
            mSurfaceController.setColorSpaceAgnostic(w.getPendingTransaction(),
                    (attrs.privateFlags & LayoutParams.PRIVATE_FLAG_COLOR_SPACE_AGNOSTIC) != 0);
            //将WindowState的hasSurface标志设置为true，标志着道歉WindowState已经有surface了
            w.setHasSurface(true);
            // The surface instance is changed. Make sure the input info can be applied to the
            // new surface, e.g. relaunch activity.
            w.mInputWindowHandle.forceChange();

        } catch (OutOfResourcesException e) {
            .....
        }

        ......

        mLastHidden = true;

        return mSurfaceController;
    }
```

### performSurfacePlacement

WindowSurfacePlacer.performSurfacePlacement是一个确定所有窗口的Surface的如何摆放，如何显示、显示在什么位置、显示区域多大的一个入口方法。      
这部分有一下四个流程：     

 - 处理有关窗口布局循环的逻辑。performSurfacePlacementLoop
 - 处理Surface的状态变更，以及调用 layoutWindowLw 的流程。
 - 计算窗口位置大小。主要在 WindowLayout.computeFrames
 - 处理surface的位置、大小以及显示等。



#### 处理窗口布局循环

```
    final void performSurfacePlacement(boolean force) {
        if (mDeferDepth > 0 && !force) {
            mDeferredRequests++;
            return;
        }
        //将循环的最大次数设置为6次
        int loopCount = 6;
        do {
            // 将 mTraversalScheduled 设置为false，在执行 performSurfacePlacementLoop 过程中
            // 只有 mTraversalScheduled 不重新设置true，那么这个循环就只会执行一次
            mTraversalScheduled = false;
            performSurfacePlacementLoop();
            mService.mAnimationHandler.removeCallbacks(mPerformSurfacePlacement);
            loopCount--;
            //只有当mTraversalScheduled为true且循环次数大于0时，才会再次循环执行布局
        } while (mTraversalScheduled && loopCount > 0);
        mService.mRoot.mWallpaperActionPending = false;
    }
```

performSurfacePlacementLoop方法主要做两件事：

 - 调用RootWindowContainer对所有窗口执行布局操作
 - 处理是否再次进行布局的逻辑。如果DisplayContent.mLayoutNeeded标志位为true且布局循环次数小于6次，则会将mTraversalScheduled标志位置为true，在performSurfacePlacement中会再次调用performSurfacePlacementLoop。

```
    private void performSurfacePlacementLoop() {
        if (mInLayout) {
            // 若当前已经进行布局，则返回
            return;
        }

        // 设置正在布局中的标志
        mInLayout = true;

        ......

        try {
            //调用RootWindowContainer的performSurfacePlacement()方法对所有窗口执行布局操作
            mService.mRoot.performSurfacePlacement();

            mInLayout = false;

            if (mService.mRoot.isLayoutNeeded()) {
                //若需要布局，且布局次数小于6次，则需要再次请求布局
                if (++mLayoutRepeatCount < 6) {
                    //该方法中会将mTraversalScheduled标志位设置位true
                    requestTraversal();
                } else {
                    mLayoutRepeatCount = 0;
                }
            } else {
                mLayoutRepeatCount = 0;
            }

            if (mService.mWindowsChanged && !mService.mWindowChangeListeners.isEmpty()) {
                mService.mH.removeMessages(REPORT_WINDOWS_CHANGE);
                mService.mH.sendEmptyMessage(REPORT_WINDOWS_CHANGE);
            }
        } catch (RuntimeException e) {
            mInLayout = false;
        }
    }
```


#### 处理Surface的状态变更

处理所有Surface的状态变更主要在 performSurfacePlacementNoTrace() 方法：    

1.如果有焦点变化，更新焦点。
2.执行窗口尺寸计算，surface状态变更等操作。
3.将Surface状态变更为HAS_DRAWN，触发App触发动画。该过程在finishdrawing()中再详细分析。
4.如果壁纸有变化，更新壁纸。
5.再次处理焦点变化。
6.如果过程中由size或者位置变化，则通知客户端重新relayout。
7.销毁不可见的窗口。

```
    void performSurfacePlacementNoTrace() {
        ......
        // 处理焦点变化
        if (mWmService.mFocusMayChange) {
            mWmService.mFocusMayChange = false;
            mWmService.updateFocusedWindowLocked(
                    UPDATE_FOCUS_WILL_PLACE_SURFACES, false /*updateInputWindows*/);
        }

        ......
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "applySurfaceChanges");
        //开启事务，获取GlobalTransactionWrapper对象
        mWmService.openSurfaceTransaction();
        try {
            //执行窗口尺寸计算，surface状态变更等操作
            applySurfaceChangesTransaction();
        } catch (RuntimeException e) {
            Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
        } finally {
            //关闭事务
            mWmService.closeSurfaceTransaction("performLayoutAndPlaceSurfaces");
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }

        // Send any pending task-info changes that were queued-up during a layout deferment
        mWmService.mAtmService.mTaskOrganizerController.dispatchPendingEvents();
        mWmService.mAtmService.mTaskFragmentOrganizerController.dispatchPendingEvents();
        //wmshell的流程的动画
        mWmService.mSyncEngine.onSurfacePlacement();
        mWmService.mAnimator.executeAfterPrepareSurfacesRunnables();
        //将Surface状态变更为HAS_DRAWN，触发App触发动画。该过程在“2.3.3mDrawState变更为HAS_DRAW”流程中再详细分析
        checkAppTransitionReady(surfacePlacer);

        // Defer starting the recents animation until the wallpaper has drawn
        final RecentsAnimationController recentsAnimationController =
                mWmService.getRecentsAnimationController();
        if (recentsAnimationController != null) {
            recentsAnimationController.checkAnimationReady(defaultDisplay.mWallpaperController);
        }
        mWmService.mAtmService.mBackNavigationController
                .checkAnimationReady(defaultDisplay.mWallpaperController);
        //遍历所有DisplayContent，如果壁纸有变化，更新壁纸
        for (int displayNdx = 0; displayNdx < mChildren.size(); ++displayNdx) {
            final DisplayContent displayContent = mChildren.get(displayNdx);
            if (displayContent.mWallpaperMayChange) {
                ProtoLog.v(WM_DEBUG_WALLPAPER, "Wallpaper may change!  Adjusting");
                displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
            }
        }
        //在此处理焦点变化
        if (mWmService.mFocusMayChange) {
            mWmService.mFocusMayChange = false;
            mWmService.updateFocusedWindowLocked(UPDATE_FOCUS_PLACING_SURFACES,
                    false /*updateInputWindows*/);
        }

        if (isLayoutNeeded()) {
            defaultDisplay.pendingLayoutChanges |= FINISH_LAYOUT_REDO_LAYOUT;
        }
        //如果过程中size或者位置变化，则通知客户端重新relayout
        handleResizingWindows();

        if (mOrientationChangeComplete) {
            if (mWmService.mWindowsFreezingScreen != WINDOWS_FREEZING_SCREENS_NONE) {
                mWmService.mWindowsFreezingScreen = WINDOWS_FREEZING_SCREENS_NONE;
                mWmService.mLastFinishedFreezeSource = mLastWindowFreezeSource;
                mWmService.mH.removeMessages(WINDOW_FREEZE_TIMEOUT);
            }
            mWmService.stopFreezingDisplayLocked();
        }

        // 销毁不可见的窗口
        i = mWmService.mDestroySurface.size();
        if (i > 0) {
            do {
                i--;
                WindowState win = mWmService.mDestroySurface.get(i);
                win.mDestroying = false;
                final DisplayContent displayContent = win.getDisplayContent();
                if (displayContent.mInputMethodWindow == win) {
                    displayContent.setInputMethodWindowLocked(null);
                }
                if (displayContent.mWallpaperController.isWallpaperTarget(win)) {
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                }
                win.destroySurfaceUnchecked();
            } while (i > 0);
            mWmService.mDestroySurface.clear();
        }

        for (int displayNdx = 0; displayNdx < mChildren.size(); ++displayNdx) {
            final DisplayContent displayContent = mChildren.get(displayNdx);
            if (displayContent.pendingLayoutChanges != 0) {
                displayContent.setLayoutNeeded();
            }
        }

        ......
    }
```

RootWindowContainer.applySurfaceChangesTransaction() 方法中其主要执行：     

 - 水印、StrictMode警告框以及模拟器显示的布局。
 - 遍历所有DisplayContent执行其 applySurfaceChangesTransaction

DisplayContent.applySurfaceChangesTransaction() 方法主要执行：     

 - 遍历所有窗口，计算窗口的布局大小，具体流程查看 DisplayContent.performLayoutNoTrace()
 - 遍历所有窗口过程中进行surface的状态更改。mDrawState变更为HAS_DRAW”
 - 处理surface的位置、大小以及显示等。

```
    void applySurfaceChangesTransaction() {
        //获取WindowSurfacePlacer 
        final WindowSurfacePlacer surfacePlacer = mWmService.mWindowPlacerLocked;

        .....

        // 执行布局，该方法最终会调用DisplayContent.performLayoutNoTrace，计算窗口的布局参数
        performLayout(true /* initial */, false /* updateInputWindows */);
        pendingLayoutChanges = 0;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "applyPostLayoutPolicy");
        try {
            mDisplayPolicy.beginPostLayoutPolicyLw();
            forAllWindows(mApplyPostLayoutPolicy, true /* traverseTopToBottom */);
            mDisplayPolicy.finishPostLayoutPolicyLw();
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        mInsetsStateController.onPostLayout();

        mTmpApplySurfaceChangesTransactionState.reset();

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "applyWindowSurfaceChanges");
        try {
            //遍历所有窗口，主要是改变surface的状态。
            forAllWindows(mApplySurfaceChangesTransaction, true /* traverseTopToBottom */);
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        //处理各个surface的位置、大小以及是否要在屏幕上显示等。后面finishDrawing()流程中再跟踪
        prepareSurfaces();

        ......
    }
```

变更surface的状态：     

```
// WindowState.java
    boolean commitFinishDrawing(SurfaceControl.Transaction t) {
    ......
        mWinAnimator.mLastAlpha = -1;
        ProtoLog.v(WM_DEBUG_ANIM, "performShowLocked: mDrawState=HAS_DRAWN in %s", this);
        mWinAnimator.mDrawState = HAS_DRAWN;
        mWmService.scheduleAnimationLocked();
```

#### 计算窗口大小

```
// WindowLayout.java

    public void computeFrames(WindowManager.LayoutParams attrs, InsetsState state,
            Rect displayCutoutSafe, Rect windowBounds, @WindowingMode int windowingMode,
            int requestedWidth, int requestedHeight, @InsetsType int requestedVisibleTypes,
            float compatScale, ClientWindowFrames frames) {
        // 传入的参数attrs中提取出窗口的类型（type）、标志（fl）、私有标志（pfl）
        // 和布局是否在屏幕内（layoutInScreen）
        final int type = attrs.type;
        final int fl = attrs.flags;
        final int pfl = attrs.privateFlags;
        //FLAG_LAYOUT_IN_SCREEN：窗口占满整个屏幕，忽略周围的装饰边框，比如状态栏
        final boolean layoutInScreen = (fl & FLAG_LAYOUT_IN_SCREEN) == FLAG_LAYOUT_IN_SCREEN;
        //定义了用于存储结果的矩形变量，包含：attachedWindow边界（attachedWindowFrame）显示边界（outDisplayFrame）、父边界（outParentFrame）和实际边界（outFrame）
        final Rect attachedWindowFrame = frames.attachedFrame;
        final Rect outDisplayFrame = frames.displayFrame;
        final Rect outParentFrame = frames.parentFrame;
        final Rect outFrame = frames.frame;

        //计算窗口被Insets限制的边界。Insets是屏幕边缘的空间，用于放置状态栏、导航栏等。
        //这一步通过调用state.calculateInsets()方法完成，该方法需要窗口边界和窗口布局参数作为输入。
        final Insets insets = state.calculateInsets(windowBounds, attrs.getFitInsetsTypes(),
                attrs.isFitInsetsIgnoringVisibility());
        //代码根据Insets的边类型（LEFT、TOP、RIGHT、BOTTOM），从计算出的Insets中提取出相应的边距，
        //并将它们添加到窗口的原始边界上，得到显示边界。
        final @WindowInsets.Side.InsetsSide int sides = attrs.getFitInsetsSides();
        final int left = (sides & WindowInsets.Side.LEFT) != 0 ? insets.left : 0;
        final int top = (sides & WindowInsets.Side.TOP) != 0 ? insets.top : 0;
        final int right = (sides & WindowInsets.Side.RIGHT) != 0 ? insets.right : 0;
        final int bottom = (sides & WindowInsets.Side.BOTTOM) != 0 ? insets.bottom : 0;
        //代码将计算出的显示边界赋值给outDisplayFrame
        outDisplayFrame.set(windowBounds.left + left, windowBounds.top + Math.max(top, minTop),
                windowBounds.right - right, windowBounds.bottom - Math.max(bottom, minBottom));
                
        //计算 outParentFrame
        if (attachedWindowFrame == null) {
            //将outParentFrame设置为与outDisplayFrame相同，这意味着父边界与显示边界相同
            outParentFrame.set(outDisplayFrame);
            //检查私有标志PRIVATE_FLAG_INSET_PARENT_FRAME_BY_IME是否被设置。
            //这个标志可能表示是否需要根据输入法窗口（IME）的位置来调整父边界。
            if ((pfl & PRIVATE_FLAG_INSET_PARENT_FRAME_BY_IME) != 0) {
                //从状态中获取输入法窗口
                final InsetsSource source = state.peekSource(ID_IME);
                if (source != null) {
                    //如果输入法窗口的source存在，则使用该source来计算父边界的内边距（Insets）。
                    outParentFrame.inset(source.calculateInsets(
                            outParentFrame, false /* ignoreVisibility */));
                }
            }
        } else {
            //如果layoutInScreen为true，则将outParentFrame设置为与attachedWindowFrame相同。
	        //这表示父边界是由附加窗口的边界决定的。
	        //如果layoutInScreen为false，则将outParentFrame设置为与outDisplayFrame相同。
	        //这表示父边界与显示边界相同。
            outParentFrame.set(!layoutInScreen ? attachedWindowFrame : outDisplayFrame);
        }

        // 开始计算被cutout限制的边界
        //根据屏幕的显示切边和窗口的布局属性来计算窗口在屏幕上受到限制的位置和大小，确保窗口不会覆盖到显示切边区域
        final int cutoutMode = attrs.layoutInDisplayCutoutMode; // 切边模式
        final DisplayCutout cutout = state.getDisplayCutout();//屏幕上的显示切边区域
        //将displayCutoutSafeExceptMaybeBars设置为与displayCutoutSafe相同，
        //这是一个临时矩形，用于稍后计算不受某些系统界面元素（如状态栏）影响的显示切边安全区域。
        final Rect displayCutoutSafeExceptMaybeBars = mTempDisplayCutoutSafeExceptMaybeBarsRect;
        displayCutoutSafeExceptMaybeBars.set(displayCutoutSafe);
        //设置为false，表示父边界目前没有被显示切边裁剪
        frames.isParentFrameClippedByDisplayCutout = false;
        // 窗口没有声明使用刘海区域，并且显示切边不为空
        if (cutoutMode != LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS && !cutout.isEmpty()) {
            //获取屏幕的显示边界（displayFrame）
            final Rect displayFrame = state.getDisplayFrame();
            // 这种模式下如果屏幕短边有cutout,会延伸过去;若cutout在长边,一定不会延伸过去。
            if (cutoutMode == LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES) {
                if (displayFrame.width() < displayFrame.height()) {
                    //如果屏幕的宽度小于高度，则将displayCutoutSafeExceptMaybeBars的顶部和底部设置为
                    //最大和最小整数值，这意味着不考虑这些方向上的显示切边。
                    displayCutoutSafeExceptMaybeBars.top = MIN_Y;
                    displayCutoutSafeExceptMaybeBars.bottom = MAX_Y;
                } else {
                    //将左侧和右侧设置为最大和最小整数值
                    //这意味着不考虑这些方向上的显示切边。
                    displayCutoutSafeExceptMaybeBars.left = MIN_X;
                    displayCutoutSafeExceptMaybeBars.right = MAX_X;
                }
            }
            // FLAG_LAYOUT_INSET_DECOR:表示窗口的内容布局在DecorView之内
            final boolean layoutInsetDecor = (attrs.flags & FLAG_LAYOUT_INSET_DECOR) != 0;
            if (layoutInScreen && layoutInsetDecor
                    && (cutoutMode == LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT
                    || cutoutMode == LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES)) {
                //显示模式为默认或者SHORT_EDGES模式
                final Insets systemBarsInsets = state.calculateInsets(
                        displayFrame, systemBars(), requestedVisibleTypes);
                if (systemBarsInsets.left > 0) {
                    displayCutoutSafeExceptMaybeBars.left = MIN_X;
                }
                if (systemBarsInsets.top > 0) {
                    displayCutoutSafeExceptMaybeBars.top = MIN_Y;
                }
                if (systemBarsInsets.right > 0) {
                    displayCutoutSafeExceptMaybeBars.right = MAX_X;
                }
                if (systemBarsInsets.bottom > 0) {
                    displayCutoutSafeExceptMaybeBars.bottom = MAX_Y;
                }
            }
            //如果窗口类型是输入法（IME）
            if (type == TYPE_INPUT_METHOD
                    && displayCutoutSafeExceptMaybeBars.bottom != MAX_Y
                    && state.calculateInsets(displayFrame, navigationBars(), true).bottom > 0) {
                // The IME can always extend under the bottom cutout if the navbar is there.
                displayCutoutSafeExceptMaybeBars.bottom = MAX_Y;
            }
            final boolean attachedInParent = attachedWindowFrame != null && !layoutInScreen;

            // 判断窗口是否为浮窗
            final boolean floatingInScreenWindow = !attrs.isFullscreen() && layoutInScreen
                    && type != TYPE_BASE_APPLICATION;

            //对于非附加到父窗口和非浮动在屏幕上的窗口，需要处理其与显示切边的交集。这是因为这些窗口需要避免与显示切边重叠。
            if (!attachedInParent && !floatingInScreenWindow) {
                mTempRect.set(outParentFrame);
                outParentFrame.intersectUnchecked(displayCutoutSafeExceptMaybeBars);
                frames.isParentFrameClippedByDisplayCutout = !mTempRect.equals(outParentFrame);
            }
            outDisplayFrame.intersectUnchecked(displayCutoutSafeExceptMaybeBars);
        }
        //FLAG_LAYOUT_NO_LIMITS表示允许窗口布局到屏幕外侧。
        final boolean noLimits = (attrs.flags & FLAG_LAYOUT_NO_LIMITS) != 0;
        //检查当前窗口是否处于多窗口模式
        final boolean inMultiWindowMode = WindowConfiguration.inMultiWindowMode(windowingMode);

        if (noLimits && type != TYPE_SYSTEM_ERROR && !inMultiWindowMode) {
            outDisplayFrame.left = MIN_X;
            outDisplayFrame.top = MIN_Y;
            outDisplayFrame.right = MAX_X;
            outDisplayFrame.bottom = MAX_Y;
        }

        final boolean hasCompatScale = compatScale != 1f;
        final int pw = outParentFrame.width();
        final int ph = outParentFrame.height();
        //判断窗口的布局尺寸是否因为显示切边而扩展
        //某些设备可能具有物理上的切边（如刘海屏、水滴屏等），这些切边区域不能用于显示内容。
        //PRIVATE_FLAG_LAYOUT_SIZE_EXTENDED_BY_CUTOUT作用就是为了确保应用程序的布局在具有切边的设备上仍然正确显示
        //设置这个标志时，窗口的实际尺寸将大于其请求的尺寸，以便在切边区域周围填充空间。
        final boolean extendedByCutout =
                (attrs.privateFlags & PRIVATE_FLAG_LAYOUT_SIZE_EXTENDED_BY_CUTOUT) != 0;
        int rw = requestedWidth;
        int rh = requestedHeight;
        float x, y;
        int w, h;

        // If the view hierarchy hasn't been measured, the requested width and height would be
        // UNSPECIFIED_LENGTH. This can happen in the first layout of a window or in the simulated
        // layout. If extendedByCutout is true, we cannot use the requested lengths. Otherwise,
        // the window frame might be extended again because the requested lengths may come from the
        // window frame.
        if (rw == UNSPECIFIED_LENGTH || extendedByCutout) {
            rw = attrs.width >= 0 ? attrs.width : pw;
        }
        if (rh == UNSPECIFIED_LENGTH || extendedByCutout) {
            rh = attrs.height >= 0 ? attrs.height : ph;
        }
        //如果设置了FLAG_SCALED标志，代码会根据是否应用兼容性缩放来调整窗口的宽度和高度。
        if ((attrs.flags & FLAG_SCALED) != 0) {
            if (attrs.width < 0) {
                w = pw;
            } else if (hasCompatScale) {
                w = (int) (attrs.width * compatScale + .5f);
            } else {
                w = attrs.width;
            }
            if (attrs.height < 0) {
                h = ph;
            } else if (hasCompatScale) {
                h = (int) (attrs.height * compatScale + .5f);
            } else {
                h = attrs.height;
            }
        } else {
            if (attrs.width == MATCH_PARENT) {
                w = pw;
            } else if (hasCompatScale) {
                w = (int) (rw * compatScale + .5f);
            } else {
                w = rw;
            }
            if (attrs.height == MATCH_PARENT) {
                h = ph;
            } else if (hasCompatScale) {
                h = (int) (rh * compatScale + .5f);
            } else {
                h = rh;
            }
        }

        if (hasCompatScale) {
            x = attrs.x * compatScale;
            y = attrs.y * compatScale;
        } else {
            x = attrs.x;
            y = attrs.y;
        }

        if (inMultiWindowMode
                && (attrs.privateFlags & PRIVATE_FLAG_LAYOUT_CHILD_WINDOW_IN_PARENT_FRAME) == 0) {
            // Make sure window fits in parent frame since it is in a non-fullscreen task as
            // required by {@link Gravity#apply} call.
            w = Math.min(w, pw);
            h = Math.min(h, ph);
        }

        // We need to fit it to the display if either
        // a) The window is in a fullscreen container, or we don't have a task (we assume fullscreen
        // for the taskless windows)
        // b) If it's a secondary app window, we also need to fit it to the display unless
        // FLAG_LAYOUT_NO_LIMITS is set. This is so we place Popups, dialogs, and similar windows on
        // screen, but SurfaceViews want to be always at a specific location so we don't fit it to
        // the display.
        final boolean fitToDisplay = !inMultiWindowMode
                || ((attrs.type != TYPE_BASE_APPLICATION) && !noLimits);

        // 根据给定的重力属性、宽度、高度、父边界等，计算并设置outFrame。
        // 这里主要是确定窗口的位置。
        Gravity.apply(attrs.gravity, w, h, outParentFrame,
                (int) (x + attrs.horizontalMargin * pw),
                (int) (y + attrs.verticalMargin * ph), outFrame);

        // Now make sure the window fits in the overall display frame.
        if (fitToDisplay) {
            Gravity.applyDisplay(attrs.gravity, outDisplayFrame, outFrame);
        }

        if (extendedByCutout) {
            extendFrameByCutout(displayCutoutSafe, outDisplayFrame, outFrame,
                    mTempRect);
        }
    }

```
