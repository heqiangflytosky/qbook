---
title: Android 屏幕旋转之 Fixed Rotation 流程
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 屏幕旋转之 Fixed Rotation 流程
date: 2022-11-23 10:00:00
---

## 基本概念

### 为什么要Fixed Rotation

FixedRotation 解决在 Activity 启动时发生方向变化时需要屏幕旋转动画的问题。     
当将要启动的 Activity 和当前的 Activity 处于不同方向时，FixedRotation 能够在一定时间段内保存当前屏幕的方向，直到启动动画完成。避免以错误的方向显示当前的 Activity，从而实现 Activity 的无缝切换。

### 原理

当启动Acitivity时判断到方向发生改变的时候，系统会模拟转屏后的相关信息（DisplayAdjustment），然后将这个相关信息传给应用。当应用拿到新的信息之后发起绘制，系统会基于旧方向对新方向进行方向补偿，这样就不需要进行转屏，Activity的切换就可以使用原过渡动画，过渡动画完成之后再进行转屏，从而实现无缝旋转。    
进行Fixedrotation必须在应用绘制好之前就进行判断，也就是onResume之前要确定。否则应用已经绘制完成那么在走Fixedrotation还是要重新绘制会导致屏幕跳闪。    

## 基本流程

流程的起点就是应用启动时是否需要 Fixed Rotation，这个判断主要是在 `DisplayContent.handleTopActivityLaunchingInDifferentOrientation` 方法中进行。      
当判断需要 Fixed Roation 后就会去设置 FixedRotationLaunchingApp，

### Fixed Roation 发起流程

Fixed Roation 的判断时机：

一个是在创建 StartingWindow 的时候。    

```
ActivityStarter.startActivityInner
  ActivityStarter.recycleTask
    ActivityRecord.showStartingWindow
      ActivityRecord.addStartingWindow
        ActivityRecord.createSnapshot
          ActivityRecord.scheduleAddStartingWindow
            SnapshotStartingData.createStartingSurface
              StartingSurfaceController.createTaskSnapshotSurface
                DisplayContent.handleTopActivityLaunchingInDifferentOrientation
```

第二个是在 Activity resume 的时候。这个时候会调用  DisplayContent.updateOrientation 来更新屏幕方向，这个时候会去判断是否需要 Fixed Roation。    

```
ActivityClientController.activityPaused
  ActivityRecord.activityPaused
    TaskFragment.completePause
      RootWindowContainer.resumeFocusedTasksTopActivities
        Task.resumeTopActivityUncheckedLocked
          Task.resumeTopActivityInnerLocked
            TaskFragment.resumeTopActivity
              RootWindowContainer.ensureVisibilityAndConfig
                DisplayContent.updateOrientationAndComputeConfig
                  DisplayContent.updateOrientation
                    DisplayContent.handleTopActivityLaunchingInDifferentOrientation
```


### Fixed Roation 判断逻辑

handleTopActivityLaunchingInDifferentOrientation 方法用来决定启动的 Activity 是否需要 Fixed Roation

@param r 启动Activity，可能会改变显示方向
@param orientationSrc 如果发射活动采用“后方”方向，可能与{@param r}不同。
@param checkOpening 是否要检查Activity在进行动画。如果调用者不确定 Activity 是否正在启动，则设置为 {@code true}。
@return {@code true} 返回true表示需要 Fixed Roation。

```
    private boolean handleTopActivityLaunchingInDifferentOrientation(@NonNull ActivityRecord r,
            @NonNull ActivityRecord orientationSrc, boolean checkOpening) {
        // 默认是开启的
        if (!WindowManagerService.ENABLE_FIXED_ROTATION_TRANSFORM) {
            return false;
        }
        // 如果当前FixedRotationTransform正在结束，返回false
        if (r.isFinishingFixedRotationTransform()) {
            return false;
        }
        // 如果当前正在进行 FixedRotationTransform 返回true
        if (r.hasFixedRotationTransform()) {
            if (mWmService.mFlags.mRespectNonTopVisibleFixedOrientation
                    && mFixedRotationLaunchingApp == null) {
                // It could be finishing the previous top translucent activity, and the next fixed
                // orientation activity becomes the current top.
                setFixedRotationLaunchingAppUnchecked(r,
                        r.getWindowConfiguration().getDisplayRotation());
            }
            // It has been set and not yet finished.
            return true;
        }
        // 在 Android 16版本中，mRespectNonTopVisibleFixedOrientation 为 true
        if (mWmService.mFlags.mRespectNonTopVisibleFixedOrientation) {
            if (r.isReportedDrawn()) {
                // 如果应用已经绘制完成，那么就不需要进行 FixedRotation 了，可以进行
                return false;
            }
        } else if (!r.occludesParent() || r.isReportedDrawn()) {
            // 进入或离开半透明或浮动Activity（例如对话框样式）时，背景中有一个可见的Activity。
            // 这种情况下需要旋转动画。
            return false;
        }
        if (checkOpening) {
            if (!mTransitionController.isCollecting(r)) {
                // Apply normal rotation animation in case the activity changes requested
                // orientation without activity switch.
                return false;
            }
            if (r.isState(RESUMED) && !r.getTask().mInResumeTopActivity) {
                // 如果活动正在执行或已完成生命周期回调，请使用正常旋转动画，
                // 以便显示信息能立即更新（参见 updateDisplayAndOrientation）。
                // 这样可以防止兼容性问题，比如在 ActivityonCreate 中调用 setRequestedOrientation然后获取显示信息。
                // 如果应用了固定旋转，显示旋转仍是旧的，除非客户端在调整到来后重新获得旋转。
                return false;
            }
        } else if (r != topRunningActivity()) {
            // 如果Activity没有启动，而且不是最顶端的Activity，也不用 FixedRotation
            return false;
        }
        if (mLastWallpaperVisible && r.windowsCanBeWallpaperTarget()
                && !mTransitionController.isTransientLaunch(r)) {
            // 如果没有在执行 transient-launch 动画，r 可以作为WallpaperTarget，那么就不需要要 FixedRotation
            // 这时就可以用选择动画来改变壁纸方向
            return false;
        }
        final int rotation = rotationForActivityInDifferentOrientation(orientationSrc);
        if (rotation == ROTATION_UNDEFINED) {
            // The display rotation won't be changed by current top activity. The client side
            // adjustments of previous rotated activity should be cleared earlier. Otherwise if
            // the current top is in the same process, it may get the rotated state. The transform
            // will be cleared later with transition callback to ensure smooth animation.
            return false;
        }
        if (!r.getDisplayArea().matchParentBounds()) {
            // Because the fixed rotated configuration applies to activity directly, if its parent
            // has it own policy for bounds, the activity bounds based on parent is unknown.
            return false;
        }
        // 开始进行 Fixed Rotation
        setFixedRotationLaunchingApp(r, rotation);
        return true;
    }

```

### setFixedRotationLaunchingApp 流程

提前计算得到转屏后的config，activity或者wallpaper使用这个模拟的config去绘制。    
由于还在旧的方向上，所以需要在旧方向加上rotation transform。    

```
DisplayContent.setFixedRotationLaunchingApp
  DisplayContent.startFixedRotationTransform
    // 根据给定的rotation 计算 Configuration，而不改变当前显示。
    DisplayContent.computeScreenConfiguration()
    DisplayContent.calculateDisplayCutoutForRotation
    DisplayContent.calculateRoundedCornersForRotation
    DisplayContent.calculatePrivacyIndicatorBoundsForRotation
    DisplayContent.calculateDisplayShapeForRotation
    // 根据前面的计算信息创建 DisplayFrames
    new DisplayFrames(new InsetsState(), info,cutout, roundedCorners, indicatorBounds, displayShape)
    ActivityRecord.applyFixedRotationTransform
      WindowToken.applyFixedRotationTransform
        new WindowToken$FixedRotationTransformState
        // 根据前面的计算信息模拟 DisplayFrames，这个不会更改现有显示
        DisplayPolicy.simulateLayoutDisplay
        WindowToken.onFixedRotationStatePrepared()
          ActivityRecord.onConfigurationChanged
            WindowContainer.onConfigurationChanged
              WindowContainer.updateSurfacePositionNonOrganized
                WindowToken.updateSurfacePosition
                  WindowContainer.updateSurfaceRotation
                    RotationUtils.rotatePoint
                      // 旋转窗口方向，使其按照现有的屏幕方向显示
                      Transaction.setMatrix
                    // 修改一下位置偏移
                    Transaction.setPosition
  DisplayContent.setFixedRotationLaunchingAppUnchecked
    DisplayContent.startAsyncRotation
      AsyncRotationController.start
```

那么 ActivityRecord 在这里变换后的矩阵为：    

```
Requested  Transform:

0   1   1200

-1  0   0

0   0   1
```

对 ActivityRecord 或者 WallpaperWindowToken做rotation的变换

```
// WindowContainer.java
    protected void updateSurfaceRotation(Transaction t, @Surface.Rotation int deltaRotation,
            @Nullable SurfaceControl positionLeash) {
        // parent must be non-null otherwise deltaRotation would be 0.
        RotationUtils.rotateSurface(t, mSurfaceControl, deltaRotation);
        mTmpPos.set(mLastSurfacePosition.x, mLastSurfacePosition.y);
        final Rect parentBounds = getParent().getBounds();
        final boolean flipped = (deltaRotation % 2) != 0;
        RotationUtils.rotatePoint(mTmpPos, deltaRotation,
                flipped ? parentBounds.height() : parentBounds.width(),
                flipped ? parentBounds.width() : parentBounds.height());
        t.setPosition(positionLeash != null ? positionLeash : mSurfaceControl,
                mTmpPos.x, mTmpPos.y);
    }
```



### 开始继续进行屏幕的 Rotation 动画

当 Top Activity 的启动动画结束之后，就可以继续进行屏幕 Rotation 的动画了。这个时候，其他窗口的方向才会真正的改变。      
通过动画结束时执行 continueUpdateOrientationForDiffOrienLaunchingApp 来进行。    

```
TransitionController.finishTransition
  Transition.finishTransition
    DisplayContent.onTransitionFinished
      DisplayContent.continueUpdateOrientationForDiffOrienLaunchingApp
        DisplayRotation.updateOrientation
          // 这里也会进行Fixed Rotation 的判断，返回false才会进行 updateRotationUnchecked
          DisplayContent.handleTopActivityLaunchingInDifferentOrientation()
          DisplayRotation.updateRotationUnchecked
            DisplayContent.applyFixedRotationForNonTopVisibleActivityIfNeeded
            // 开始请求Transition 动画 
            DisplayContent.requestChangeTransition()
        // 如果屏幕方向没有进行改变的必要，清除 FixedRotationLaunchingApp
        DisplayContent.clearFixedRotationLaunchingApp()
```

### 结束 Fixed Roation

转屏动画开始时结束 Fixed Roation

```
WindowOrganizerController.applyTransaction
  Transition.applyDisplayChangeIfNeeded
    DisplayContent.sendNewConfiguration
      DisplayContent.updateDisplayOverrideConfigurationLocked
        ActivityTaskManagerService.updateGlobalConfigurationLocked
          RootWindowContainer.onConfigurationChanged
            WindowContainer.onConfigurationChanged
              ConfigurationContainer.onConfigurationChanged
                RootWindowContainer.dispatchConfigurationToChild
                  DisplayContent.performDisplayOverrideConfigUpdate
                    DisplayContent.onRequestedOverrideConfigurationChanged
                      DisplayContent.applyRotationAndFinishFixedRotation
                        WindowToken.finishFixedRotationTransform
```



