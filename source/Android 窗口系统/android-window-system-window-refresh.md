---
title: Android 窗口系统-窗口刷新
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口刷新
date: 2022-11-23 10:00:00
---


## 窗口的五个绘制状态

WindowStateAnimator的 mDrawState 属性用于表示窗口的绘制状态。mDrawState有以下五个状态：     

 - NO_SURFACE：窗口刚刚被创建，还没有Surface。
 - DRAW_PENDING：窗口创建了Surface，但还没有绘制内容。
 - COMMIT_DRAW_PENDING：窗口完成了第一次绘制，但还没有显示。
 - READY_TO_SHOW：窗口已经准备好显示，但可能还需要等待同一WindowToken的的所有窗口都准备好显示。
 - HAS_DRAWN：窗口已经完成绘制并显示在屏幕上。

## DRAW_PENDING

创建 SurfaceControl 时会把 mDrawState 的初始状态设置为 DRAW_PENDING。     

```
        // 创建 SurfaceControl
        WindowManagerService.createSurfaceControl()
            WindowStateAnimator.createSurfaceLocked()
                WindowStateAnimator.resetDrawState()
                    mDrawState = DRAW_PENDING;
```

## COMMIT_DRAW_PENDING

当应用端执行 measure-layout-draw 流程之后，便会调用 WindowManagerService.finishDrawingWindow，处理 Surface 的状态变更并将 Surface show 出来。     

```
//app
ViewRootImpl.performTraversals()
    SurfaceSyncGroup.markSyncReady()
        SurfaceSyncGroup.checkIfSyncIsComplete()
            // 执行创建 SurfaceSyncGroup 时的回调
            mTransactionReadyConsumer.accept(mTransaction)
                ViewRootImpl.reportDrawFinished()
                    WindowSession.finishDrawing()

// system_server
Session.finishDrawing()
    WindowManagerService.finishDrawingWindow()
        WindowState.finishDrawing
            WindowStateAnimator.finishDrawingLocked()
                // 修改 mDrawState 状态为 COMMIT_DRAW_PENDING
                mDrawState = COMMIT_DRAW_PENDING;
        WindowSurfacePlacer.requestTraversal()
            WindowSurfacePlacer.performSurfacePlacement()
```



```
//WindowStateAnimator.java
    boolean finishDrawingLocked(SurfaceControl.Transaction postDrawTransaction) {
        final boolean startingWindow =
                mWin.mAttrs.type == WindowManager.LayoutParams.TYPE_APPLICATION_STARTING;
        if (startingWindow) {
            ProtoLog.v(WM_DEBUG_STARTING_WINDOW, "Finishing drawing window %s: mDrawState=%s",
                    mWin, drawStateToString());
        }

        boolean layoutNeeded = false;
        // 状态变更
        if (mDrawState == DRAW_PENDING) {
            mDrawState = COMMIT_DRAW_PENDING;
            layoutNeeded = true;
        }

        if (postDrawTransaction != null) {
            mWin.getSyncTransaction().merge(postDrawTransaction);
            layoutNeeded = true;
        }

        return layoutNeeded;
    }
```

## READY_TO_SHOW

前面把 mDrawState 状态变更为 COMMIT_DRAW_PENDING 后，在 WindowManagerService.finishDrawingWindow 中会请求一次布局刷新。     

```
// WindowSurfacePlacer.java
    void requestTraversal() {
        if (mTraversalScheduled) {
            return;
        }

        // Set as scheduled even the request will be deferred because mDeferredRequests is also
        // increased, then the end of deferring will perform the request.
        mTraversalScheduled = true;
        if (mDeferDepth > 0) {
            mDeferredRequests++;
            if (DEBUG) Slog.i(TAG, "Defer requestTraversal " + Debug.getCallers(3));
            return;
        }
        mService.mAnimationHandler.post(mPerformSurfacePlacement);
    }
```

requestTraversal() 方法首先将遍历标志为mTraversalSchedule置为true，然后发送handle消息mPerformSurfacePlacement。    
mPerformSurfacePlacement会新建一个线程调用 WindowSurfacePlacer.performSurfacePlacement，这个逻辑在前面窗口布局文章中已经介绍过了。具体流程可以看看前面流程图。       

```
// WindowStateAnimator.java
    boolean commitFinishDrawingLocked() {
        if (mDrawState != COMMIT_DRAW_PENDING && mDrawState != READY_TO_SHOW) {
            return false;
        }
        ProtoLog.i(WM_DEBUG_ANIM, "commitFinishDrawingLocked: mDrawState=READY_TO_SHOW %s",
                mSurfaceController);
        mDrawState = READY_TO_SHOW;
        boolean result = false;
        final ActivityRecord activity = mWin.mActivityRecord;
        //直接进入到WindowState.performShowLocked()流程的三种情况
        //1.如果ActivityRecord为空，这种情况可以理解为不依赖Activity的窗口，比如常见的悬浮窗
        //2.或者canShowWindows()为true，这个方法大概是说：只有当所有窗口都已绘制完成，并且没有正在进行父级窗口的应用过渡动画，并且没有非默认颜色的窗口存在时，返回true
        //3.或者窗口类型为启动窗口，启动窗口就是StartingWindow，应用启动时出现的窗口，常见的就是Splash screen ，许多应用都会定义自己的SplashActivity
        //进入performShowLocked()流程后mDrawState更新HAS_DRAWN
        //由于非这三种情况最终也会调用到performShowLocked()，因此下面这种情况我们暂不讨论
        if (activity == null || activity.canShowWindows()
                || mWin.mAttrs.type == TYPE_APPLICATION_STARTING) {
            result = mWin.performShowLocked();
        }
        return result;
    }
```

commitFinishDrawingLocked() 方法这里的大概流程：

 - 对mDrawState的状态进行过滤，非COMMIT_DRAW_PENDING和READY_TO_SHOW则直接返回。
 - 将状态更新为READY_TO_SHOW。
 - 根据情况决定是否执行 performShowLocked 变更状态，并做过渡动画,具体见代码注释

performSurfacePlacement 方法和前面执行窗口布局流程的不同地方有下面几点：    

 - 窗口状态刷新流程在DisplayContent.applySurfaceChangesTransaction中调用mApplySurfaceChangesTransaction，处理mDrawState状态。
 - 窗口状态刷新流程在RootWindowContainer.performSurfacePlacementNoTrace中调用checkAppTransitionReady，处理mDrawState状态变更为HAS_DRAWN，触发Activity过渡动画。
 - 窗口状态刷新流程在WindowSurfacePlacementLoop.performSurfacePlacementLoop中会调用requestTraversal，请求再次布局。
 - 窗口状态刷新流程在mDrawState状态变更为HAS_DRAWN后，会通过WindowStateAnimator.prepareSurfaceLocked处理surface的位置、大小以及显示等。

## HAS_DRAWN

在前面窗口布局流程中，对与两种情况下执行 HAS_DRAWN 状态变更都有介绍，这里我们就只介绍一下启动 Activity 窗口时的情况，悬浮窗口的情况可关注上面 commitFinishDrawingLocked() 方法中的注释。    
这里我们主要介绍一下 `checkAppTransitionReady()` 方法中的状态变更逻辑。     


```
// 窗口尺寸的计算以及Surface的状态变更
WindowSurfacePlacer.performSurfacePlacement
    // 窗口布局循环的逻辑
    WindowSurfacePlacer.performSurfacePlacementLoop()
        RootWindowContainer.performSurfacePlacementNoTrace()
            // 更新焦点
            WindowManagerService.updateFocusedWindowLocked
            // 执行窗口尺寸计算，surface状态变更等操作
            RootWindowContainer.applySurfaceChangesTransaction()
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
                                        WindowState.performShowLocked()
                                            // 进入窗口动画流程
                                            WindowStateAnimator.applyEnterAnimationLocked()
                                            //更新 mDrawState = HAS_DRAWN
                                            mDrawState = HAS_DRAWN
                                            // 
                                            WindowManagerService.scheduleAnimationLocked()
```

### checkAppTransitionReady

checkAppTransitionReady() 会遍历所有DisplayContent，处理activity的过滤动画

```
// RootWindowContainer.java
    private void checkAppTransitionReady(WindowSurfacePlacer surfacePlacer) {
        // Trace all displays app transition by Z-order for pending layout change.
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            final DisplayContent curDisplay = mChildren.get(i);

            //检查所有要显示的app token，是否已经准备就绪
            if (curDisplay.mAppTransition.isReady()) {
                // handleAppTransitionReady may modify curDisplay.pendingLayoutChanges.
                curDisplay.mAppTransitionController.handleAppTransitionReady();
            }

            if (curDisplay.mAppTransition.isRunning() && !curDisplay.isAppTransitioning()) {
                curDisplay.handleAnimatingStoppedAndTransition();
            }
        }
    }
```

AppTransitionController的handleAppTransitionReady()方法主要执行了下面的逻辑：

 - 处理activity的过渡动画（远程动画）
 - 分别调用 handleClosingApps以及handleOpeningApps对要关闭的和要打开的Activity进行可见性更新。
 - 调用AppTransition.goodToGo方法走播放远程动画流程。
 - 调用onTransitionStarting方法开始快照相关流程。
 - 由于activity的可见性变更，将DisplayContent.mLayoutNeeded设置为true，该标志位在DisplayContent.performLayoutNoTrace中用来判断是否对当前DisplayContent下的所有窗口进行刷新。

```
// AppTransitionController.java
    void handleAppTransitionReady() {
        
        ......
        //通过getTransitCompatType方法获取transit的值
        @TransitionOldType final int transit = getTransitCompatType(
                mDisplayContent.mAppTransition, tmpOpenApps,
                tmpCloseApps, mDisplayContent.mChangingContainers,
                mWallpaperControllerLocked.getWallpaperTarget(), getOldWallpaper(),
                mDisplayContent.mSkipAppTransitionAnimation);
        mDisplayContent.mSkipAppTransitionAnimation = false;

        ......

    	//获取正在打开 (mOpeningApps)、关闭 (mClosingApps) 和切换 (mChangingContainers) 的应用的activity类型
    	//并将它们存储在 activityTypes 集合中。
        final ArraySet<Integer> activityTypes = collectActivityTypes(tmpOpenApps,
                tmpCloseApps, mDisplayContent.mChangingContainers);
        //被用于查找与给定transit和activityTypes相关的 ActivityRecord
        //也就是我们当前打开的应用的ActivityRecord
        final ActivityRecord animLpActivity = findAnimLayoutParamsToken(transit, activityTypes,
                tmpOpenApps, tmpCloseApps, mDisplayContent.mChangingContainers);
        //获取正在打开的应用列表 (mOpeningApps) 中的顶层应用。
        //ignoreHidden 参数设置为 false，意味着即使应用是隐藏的，也会被考虑在内
        final ActivityRecord topOpeningApp =
                getTopApp(tmpOpenApps, false /* ignoreHidden */);
        //获取正在关闭的应用列表 (mClosingApps) 中的顶层应用
        final ActivityRecord topClosingApp =
                getTopApp(tmpCloseApps, false /* ignoreHidden */);
        //获取正在切换的应用列表 (mChangingContainers) 中的顶层应用
        //其取决于参数DisplayContent.mChangingContainers中是否有值
        /** 
        有三种情况会给DisplayContent.mChangingContainers中添加值
        1.｛@link Task｝在全屏和浮窗之间发生切换
        2.｛@link TaskFragment｝已组织好并且正在更改窗口边界
        3.｛@link ActivityRecord｝被重新分配到一个有组织的｛@link TaskFragment｝中
        **/
        final ActivityRecord topChangingApp =
                getTopApp(mDisplayContent.mChangingContainers, false /* ignoreHidden */);
        //从之前找到的animLpActivity（正在打开的应用的ActivityRecord）的窗口中获取布局参数
        final WindowManager.LayoutParams animLp = getAnimLp(animLpActivity);

        ......
        try {
            //应用app transition动画（远程动画）
            applyAnimations(tmpOpenApps, tmpCloseApps, transit, animLp, voiceInteraction);
            //处理closing activity可见性
            handleClosingApps();
            // 处理opening actvity可见性
            handleOpeningApps();
            // 处理用于处理正在切换的应用
            handleChangingApps(transit);
            // 处理正在关闭或更改的容器
            handleClosingChangingContainers();
            // 设置与最后一次应用过渡动画相关的信息
            appTransition.setLastAppTransition(transit, topOpeningApp,
                    topClosingApp, topChangingApp);

            final int flags = appTransition.getTransitFlags();
            //播放远程动画
            layoutRedo = appTransition.goodToGo(transit, topOpeningApp);
            appTransition.postAnimationCallback();
        } finally {
            appTransition.clear();
            mService.mSurfaceAnimationRunner.continueStartingAnimations();
        }
        //开始快照相关流程
        mService.mSnapshotController.onTransitionStarting(mDisplayContent);

        mDisplayContent.mOpeningApps.clear();
        mDisplayContent.mClosingApps.clear();
        mDisplayContent.mChangingContainers.clear();
        mDisplayContent.mUnknownAppVisibilityController.clear();
        mDisplayContent.mClosingChangingContainers.clear();

        // 由于activity 窗口的可见性变更，将DisplayContent.mLayoutNeeded标志位置为true
        mDisplayContent.setLayoutNeeded();

        mDisplayContent.computeImeTarget(true /* updateImeTarget */);

        mService.mAtmService.mTaskSupervisor.getActivityMetricsLogger().notifyTransitionStarting(
                mTempTransitionReasons);

        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);

        mDisplayContent.pendingLayoutChanges |=
                layoutRedo | FINISH_LAYOUT_REDO_LAYOUT | FINISH_LAYOUT_REDO_CONFIG;
    }
```

applyAnimations() 基于一组ActivityRecord来应用动画，这些ActivityRecord表示正在进行切换的应用。这里先不着重介绍。     

### handleOpeningApps

handleOpeningApps() 主要逻辑如下：
 - 将所有即将open的activity的mVisible标志位设置为true
 - 调用ActivityRecord.showAllWindowsLocked()，最终会调用到WindowState.performShowLocked() ，处理mDrawState的状态变更

```
//WindowState.java
    boolean performShowLocked() {
        ......

        final int drawState = mWinAnimator.mDrawState;
        if ((drawState == HAS_DRAWN || drawState == READY_TO_SHOW) && mActivityRecord != null) {
            if (mAttrs.type != TYPE_APPLICATION_STARTING) {
                mActivityRecord.onFirstWindowDrawn(this);
            } else {
                mActivityRecord.onStartingWindowDrawn();
            }
        }

        if (mWinAnimator.mDrawState != READY_TO_SHOW || !isReadyForDisplay()) {
            return false;
        }

        logPerformShow("Showing ");

        mWmService.enableScreenIfNeededLocked();
        //进入窗口动画流程
        mWinAnimator.applyEnterAnimationLocked();

        // Force the show in the next prepareSurfaceLocked() call.
        mWinAnimator.mLastAlpha = -1;
        // 变更mDrawState 状态为 HAS_DRAWN
        mWinAnimator.mDrawState = HAS_DRAWN;
        mWmService.scheduleAnimationLocked();

        ......

        return true;
    }
```

### 再次请求布局

再次进行布局循环，主要目的是为了 show surface。     

## show surface

```
// 窗口尺寸的计算以及Surface的状态变更
WindowSurfacePlacer.performSurfacePlacement
    // 窗口布局循环的逻辑
    WindowSurfacePlacer.performSurfacePlacementLoop()
            RootWindowContainer.performSurfacePlacement()
            RootWindowContainer.performSurfacePlacementNoTrace()
                // 执行窗口尺寸计算，surface状态变更等操作
                RootWindowContainer.applySurfaceChangesTransaction()
                    //历所有窗口，计算窗口的布局大小
                    DisplayContent.applySurfaceChangesTransaction()
                        DisplayContent.performLayout()
                        // 处理各个surface的位置、大小以及是否要在屏幕上显示等。
                        DisplayContent.prepareSurfaces()
                            DisplayArea.Dimmable.prepareSurfaces()
                                WindowContainer.prepareSurfaces()
                                    WindowState.prepareSurfaces()
                                        WindowState.applyDims()
                                        //最终调用自身的updateSurfacePosition()（自身有重写该方法）计算surface的位置
                                        WindowState.updateSurfacePositionNonOrganized()
                                        WindowState.updateFrameRateSelectionPriorityIfNeeded()
                                        //更新窗口比例
                                        WindowState.updateScaleIfNeeded()
                                        WindowStateAnimator.prepareSurfaceLocked
                                            //设置mShowAlpha
                                            WindowStateAnimator.computeShownFrameLocked()
                                            判断 w.isOnScreen()
                                            // 上屏显示
                                            WindowSurfaceController.showRobustly()
                                                // 显示
                                                SurfaceControl.Transaction.show()
```



