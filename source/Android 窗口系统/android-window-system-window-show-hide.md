---
title: Android 窗口系统-窗口的显示和隐藏
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口的显示和隐藏
date: 2022-11-23 10:00:00
---

## 概述

窗口容器的 prepareSurface() 方法是窗口管理系统中的协调中枢，确保从根容器到每个子窗口的 Surface 状态在每一帧更新时保持同步。它在窗口渲染流程中扮演了“最后一公里”配置的角色。      
prepareSurfaces()方法将会对每个 WindowContainer 的 Surface 做最后的准备工作，各个 Surface 位置、大小、是否显示在屏幕上等，都将通过这个方法来进行设置，并在关闭事务后，提交给 SurfaceFlinger 进行显示。    
需要注意到，虽然 WindowContainer 的 SurfaceControl 对应的图层没有 UI 绘制，但是在 Layer 层级树中，必须父层级的 SurfaceControl 显示，WindowState 管理的实际绘制的 SurfaceControl 才会显示。     
本文基于 Android 15 进行介绍。      

## prepareSurface() 方法的调用

WindowContainer.scheduleAnimation 从动画线程触发对 prepareSurfaces 的调用，以便应用待处理的事务。     
以 ActivityRecord 设置可见性时的更新为例来介绍。    

```
ActivityRecord.setVisible
    WindowContainer.scheduleAnimation
        WindowManagerService.scheduleAnimationLocked()
            WindowAnimator.scheduleAnimation()
                Choreographer.postFrameCallback(mAnimationFrameCallback)
```

一般是 WindowAnimator.animate 方法中发起对 DisplayContent.prepareSurfaces 的调用。    

```
WindowAnimator.mAnimationFrameCallback
    WindowAnimator.animate
        DisplayContent.prepareSurfaces
            DisplayArea$Dimmable.prepareSurfaces
                WindowContainer.prepareSurfaces
                    DisplayArea$Dimmable.prepareSurfaces
                        WindowContainer.prepareSurfaces
                            Task.prepareSurfaces
                                TaskFragment.prepareSurfaces
                                    WindowContainer.prepareSurfaces
                                        ActivityRecord.prepareSurfaces
                                            WindowContainer.prepareSurfaces
                                                WindowState.prepareSurfaces
                                                    WindowStateAnimator.prepareSurfaceLocked
                                                        WindowSurfaceController.showRobustly
```

## DisplayContent.prepareSurfaces

DisplayContent 这里除了额外的抓了 Trace 外，就执行了父类 DisplayArea.Dimmable 的方法。    

```
// DisplayContent.java
    void prepareSurfaces() {
        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "prepareSurfaces");
        try {
            super.prepareSurfaces();
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
    }
```

## DisplayArea.Dimmable.prepareSurfaces()

```
        void prepareSurfaces() {
            mDimmer.resetDimStates();
            super.prepareSurfaces();
            final Rect dimBounds = mDimmer.getDimBounds();
            if (dimBounds != null) {
                // Bounds need to be relative, as the dim layer is a child.
                getBounds(dimBounds);
                dimBounds.offsetTo(0 /* newLeft */, 0 /* newTop */);
            }

            // If SystemUI is dragging for recents, we want to reset the dim state so any dim layer
            // on the display level fades out.
            if (!mTransitionController.isShellTransitionsEnabled()
                    && forAllTasks(task -> !task.canAffectSystemUiFlags())) {
                mDimmer.resetDimStates();
            }

            if (dimBounds != null) {
                if (mDimmer.updateDims(getSyncTransaction())) {
                    scheduleAnimation();
                }
            }
        }
    }
```

## WindowContainer.prepareSurfaces()

WindowContainer 的 prepareSurfaces() 方法主要是遍历执行子容器的 prepareSurfaces() 方法，从 DisplayContent 到叶子节点，确保层级树中的所有容器的 prepareSurfaces() 方法都会执行。    

```
    void prepareSurfaces() {
        // If a leash has been set when the transaction was committed, then the leash reparent has
        // been committed.
        mCommittedReparentToAnimationLeash = mSurfaceAnimator.hasLeash();
        for (int i = 0; i < mChildren.size(); i++) {
            mChildren.get(i).prepareSurfaces();
        }
    }
```

## Task.prepareSurfaces()

```
    void prepareSurfaces() {
        mDimmer.resetDimStates();
        super.prepareSurfaces();

        final Rect dimBounds = mDimmer.getDimBounds();
        if (dimBounds != null) {
            getDimBounds(dimBounds);

            // Bounds need to be relative, as the dim layer is a child.
            if (inFreeformWindowingMode()) {
                getBounds(mTmpRect);
                dimBounds.offsetTo(dimBounds.left - mTmpRect.left, dimBounds.top - mTmpRect.top);
            } else {
                dimBounds.offsetTo(0, 0);
            }
        }

        final SurfaceControl.Transaction t = getSyncTransaction();
        if (dimBounds != null && mDimmer.updateDims(t)) {
            scheduleAnimation();
        }

        // Let organizer manage task visibility for shell transition. So don't change it's
        // visibility during collecting.
        if (mTransitionController.isCollecting() && mCreatedByOrganizer) {
            return;
        }

        // We intend to let organizer manage task visibility but it doesn't
        // have enough information until we finish shell transitions.
        // In the mean time we do an easy fix here.
        final boolean visible = isVisible();
        final boolean show = visible || isAnimating(TRANSITION | PARENTS | CHILDREN);
        if (mSurfaceControl != null) {
            if (show != mLastSurfaceShowing) {
                t.setVisibility(mSurfaceControl, show);
            }
        }
        // Only show the overlay if the task has other visible children
        if (mOverlayHost != null) {
            mOverlayHost.setVisibility(t, visible);
        }
        mLastSurfaceShowing = show;
    }
```

## TaskFragment.prepareSurfaces()

```
    void prepareSurfaces() {
        if (asTask() != null) {
            super.prepareSurfaces();
            return;
        }

        mDimmer.resetDimStates();
        super.prepareSurfaces();

        final Rect dimBounds = mDimmer.getDimBounds();
        if (dimBounds != null) {
            // Bounds need to be relative, as the dim layer is a child.
            dimBounds.offsetTo(0 /* newLeft */, 0 /* newTop */);
            if (mDimmer.updateDims(getSyncTransaction())) {
                scheduleAnimation();
            }
        }
    }
```

## ActivityRecord.prepareSurfaces()

在ActivityRecord中，将会根据其mVisible属性，对它的 mSurfaceControl 进行 show 或 hide 操作，同时遍历子容器 prepareSurfaces 操作。     

```
    void prepareSurfaces() {
        final boolean isDecorSurfaceBoosted =
                getTask() != null && getTask().isDecorSurfaceBoosted();
        // 判断窗口是否可见
        final boolean show = (isVisible()
                // Ensure that the activity content is hidden when the decor surface is boosted to
                // prevent UI redressing attack.
                && !isDecorSurfaceBoosted)
                || isAnimating(PARENTS, ANIMATION_TYPE_APP_TRANSITION | ANIMATION_TYPE_RECENTS
                        | ANIMATION_TYPE_PREDICT_BACK);

        if (mSurfaceControl != null) {
            // 对该 SurfaceControl 进行show 或者hide
            if (show && !mLastSurfaceShowing) {
                getSyncTransaction().show(mSurfaceControl);
            } else if (!show && mLastSurfaceShowing) {
                getSyncTransaction().hide(mSurfaceControl);
            }
            // Input sink surface 不是 animation 的一部分，因此只需在具有待处理事务的稳定状态 （非同步） 下应用即可。
            if (show && mSyncState == SYNC_STATE_NONE) {
                mActivityRecordInputSink.applyChangesToSurfaceIfChanged(getPendingTransaction());
            }
        }
        if (mThumbnail != null) {
            mThumbnail.setShowing(getPendingTransaction(), show);
        }
        // 表示最近一次prepareSurfaces操作
        mLastSurfaceShowing = show;
        super.prepareSurfaces();
    }
```

## WindowState.prepareSurfaces

WindowState.prepareSurfaces 是所有 prepareSurfaces 方法中最重要的，因为 WindowState 间接管理着 WindowSurfaceController 的 mSurfaceControl，它是真正绘制 UI 的 SurfaceControl。    
在这个方法中，将会设置其本身的 SurfaceControl 的位置、Alpha值、Matrix、Crop等各种属性，以及设置 WindowSurfaceController 中 mSurfaceControl 的可见性。    
可以看到，WindowState 并没有对其本身的 mSurfaceControl 的可见性有任何操作。      

```
BLASTSyncEngine$SyncGroup.tryFinish
    BLASTSyncEngine$SyncGroup.finishNow
        Transition.onTransactionReady
            Transition.commitVisibleActivities
                ActivityRecord.commitFinishDrawing
                    WindowState.commitFinishDrawing
                        WindowStateAnimator.prepareSurfaceLocked
                            WindowSurfaceController.showRobustly
```



```
    void prepareSurfaces() {
        mIsDimming = false;
        if (mHasSurface) {
            // 应用Dim Layer
            if (!Dimmer.DIMMER_REFACTOR) {
                applyDims();
            }
            // 更新Surface的位置
            updateSurfacePositionNonOrganized();
            // Send information to SurfaceFlinger about the priority of the current window.
            updateFrameRateSelectionPriorityIfNeeded();
            // 设置 scale
            updateScaleIfNeeded();
            //入WindowStateAnimator中进行实际Surface的准备工作
            mWinAnimator.prepareSurfaceLocked(getSyncTransaction());
            if (Dimmer.DIMMER_REFACTOR) {
                applyDims();
            }
        }
        super.prepareSurfaces();
    }
```

### updateSurfacePosition

WindowContainer.updateSurfacePositionNonOrganized() --> WindowState.updateSurfacePosition

```
// WindowState.java
    void updateSurfacePosition(Transaction t) {
        ......

        mSurfacePlacementNeeded = false;
        // 将WindowState#mWindowFrames转换成Position
        // 使用WindowState的mWindowFrames.mFrame获得SurfaceControl的位置坐标点
        // 每个窗口的WindowFrame对象的mFrame属性，就决定了该窗口要具体显示的位置。
        transformFrameToSurfacePosition(mWindowFrames.mFrame.left, mWindowFrames.mFrame.top,
                mSurfacePosition);

        if (mWallpaperScale != 1f) {
            final Rect bounds = getParentFrame();
            Matrix matrix = mTmpMatrix;
            matrix.setTranslate(mXOffset, mYOffset);
            matrix.postScale(mWallpaperScale, mWallpaperScale, bounds.exactCenterX(),
                    bounds.exactCenterY());
            matrix.getValues(mTmpMatrixArray);
            mSurfacePosition.offset(Math.round(mTmpMatrixArray[Matrix.MTRANS_X]),
                Math.round(mTmpMatrixArray[Matrix.MTRANS_Y]));
        } else {
            mSurfacePosition.offset(mXOffset, mYOffset);
        }

        final AsyncRotationController asyncRotationController =
                mDisplayContent.getAsyncRotationController();
        if ((asyncRotationController != null
                && asyncRotationController.hasSeamlessOperation(mToken))
                || mPendingSeamlessRotate != null) {
            // Freeze position while un-rotating the window, so its surface remains at the position
            // corresponding to the original rotation.
            return;
        }

        if (!mSurfaceAnimator.hasLeash() && !mLastSurfacePosition.equals(mSurfacePosition)) {
            final boolean frameSizeChanged = mWindowFrames.isFrameSizeChangeReported();
            final boolean surfaceInsetsChanged = surfaceInsetsChanging();
            final boolean surfaceSizeChanged = frameSizeChanged || surfaceInsetsChanged;
            mLastSurfacePosition.set(mSurfacePosition.x, mSurfacePosition.y);
            if (surfaceInsetsChanged) {
                mLastSurfaceInsets.set(mAttrs.surfaceInsets);
            }
            final boolean surfaceResizedWithoutMoveAnimation = surfaceSizeChanged
                    && mWinAnimator.getShown() && !canPlayMoveAnimation() && okToDisplay()
                    && mSyncState == SYNC_STATE_NONE;
            final ActivityRecord activityRecord = getActivityRecord();
            // 如果此窗口属于由于方向更改而重新启动的 Activity，则延迟位置更新，直到它重新绘制，以避免任何闪烁。
            final boolean isLetterboxedAndRelaunching = activityRecord != null
                    && activityRecord.areBoundsLetterboxed()
                    && activityRecord.mLetterboxUiController
                        .getIsRelaunchingAfterRequestedOrientationChanged();
            if (surfaceResizedWithoutMoveAnimation || isLetterboxedAndRelaunching) {
                applyWithNextDraw(mSetSurfacePositionConsumer);
            } else {
                // 设置position
                mSetSurfacePositionConsumer.accept(t);
            }
        }
    }

    private final Consumer<SurfaceControl.Transaction> mSetSurfacePositionConsumer = t -> {
        // Only apply the position to the surface when there's no leash created.
        if (mSurfaceControl != null && mSurfaceControl.isValid() && !mSurfaceAnimator.hasLeash()) {
            t.setPosition(mSurfaceControl, mSurfacePosition.x, mSurfacePosition.y);
        }
    };
```

### WindowStateAnimator#prepareSurfaceLocked()


```
// WindowStateAnimator.java

    void prepareSurfaceLocked(SurfaceControl.Transaction t) {
        final WindowState w = mWin;
        ....
        // 计算 mShownAlpha
        computeShownFrameLocked();

        if (!w.isOnScreen()) {
            // 隐藏 SurfaceControl
            hide(t, "prepareSurfaceLocked");
            if (!w.mIsWallpaper || !Flags.ensureWallpaperInTransitions()) {
                // 也需要隐藏以它为 target 的壁纸
                mWallpaperControllerLocked.hideWallpapers(w);
            }

            .....
        } else if (mLastAlpha != mShownAlpha
                || mLastHidden) {
            mLastAlpha = mShownAlpha;
            ......
            // 通过 WindowSurfaceController 设置 alpha
            boolean prepared =
                mSurfaceController.prepareToShowInTransaction(t, mShownAlpha);

            if (prepared && mDrawState == HAS_DRAWN) {
                if (mLastHidden) {
                    // 显示 SurfaceController
                    mSurfaceController.showRobustly(t);
                    mLastHidden = false;
                    final DisplayContent displayContent = w.getDisplayContent();
                    if (!displayContent.getLastHasContent()) {
                        // This draw means the difference between unique content and mirroring.
                        // Run another pass through performLayout to set mHasContent in the
                        // LogicalDisplay.
                        displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_ANIM;
                        ......
                    }
                }
            }
        }

        ......
    }
```


## WallpaperWindowToken.prepareSurfaces()

```
    public void prepareSurfaces() {
        super.prepareSurfaces();

        if (Flags.ensureWallpaperInTransitions()) {
            // Similar to Task.prepareSurfaces, outside of transitions we need to apply visibility
            // changes directly. In transitions the transition player will take care of applying the
            // visibility change.
            if (!mTransitionController.inTransition(this)) {
                getSyncTransaction().setVisibility(mSurfaceControl, isVisible());
            }
        }
    }
```

## 相关文章

[Android R WindowManagerService模块(4) Window的定位过程 ](https://juejin.cn/post/6955427217895587853)
