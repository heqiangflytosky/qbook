---
title: Android 屏幕旋转之 AsyncRotation
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 屏幕旋转之 AsyncRotation
date: 2022-11-23 10:00:00
---

## 概述


AsyncRotationController 也是 Android 系统中用于管理屏幕旋转过程中窗口内容过渡的核心组件之一，旨在通过异步处理提升用户体验，它主要处理一些系统窗口的旋转。      该控制器的主要目的是在设备旋转（如横屏与竖屏切换）时，非Activity窗口能够平滑更新，避免界面闪烁或内容突变。    
比如一些系统窗口：状态栏和导航栏，可能在转屏动画还没有开始时，他们已经完成了横竖屏切换，那么这个时候可能就会在屏幕上闪现一些切换的界面。AsyncRotationController 可以使得这些窗口先隐藏起来，等旋转动画做完的时候再开始显示他们。      

场景：
 - 带旋转的打开应用：目标窗口在开始过渡动画时渐隐，然后在切换动画完成后以新旋转方向绘制窗口后淡入。    
 - 正常旋转：在显示截图图层后，目标窗口被父窗口以零alpha隐藏，在新旋转方向绘制完成时会淡入。    
 - 无缝旋转： 在这种情况下，只有 shell transition 才使用这个控制器。目标窗口会被单独请求使用同步事务。他们的 WindowToken 保持旧的方向。在切换动画的start transaction被apply后并以新旋转绘制窗口后，旧的旋转变换将通过应用同步事务被移除。     


## 流程

```
//DisplayContent.java
    private boolean startAsyncRotation(boolean shouldDebounce) {
        if (shouldDebounce) {
            // 需要延时启动的场景,横屏的应用返回桌面时
            mWmService.mH.postDelayed(() -> {
                synchronized (mWmService.mGlobalLock) {
                    if (mFixedRotationLaunchingApp != null
                            && startAsyncRotation(false /* shouldDebounce */)) {
                        // Apply the transaction so the animation leash can take effect immediately.
                        getPendingTransaction().apply();
                    }
                }
            }, FIXED_ROTATION_HIDE_ANIMATION_DEBOUNCE_DELAY_MS);
            return false;
        }
        if (mAsyncRotationController == null) {
            mAsyncRotationController = new AsyncRotationController(this);
            mAsyncRotationController.start();
            return true;
        }
        return false;
    }
```

下面类型的窗口才可以 AsyncRotation:

```
    static boolean canBeAsync(WindowToken token) {
        final int type = token.windowType;
        return type > WindowManager.LayoutParams.LAST_APPLICATION_WINDOW
                && type != WindowManager.LayoutParams.TYPE_INPUT_METHOD
                && type != WindowManager.LayoutParams.TYPE_WALLPAPER
                && type != WindowManager.LayoutParams.TYPE_NOTIFICATION_SHADE;
    }

```

所以说，普通应用的窗口、输入法窗口、壁纸窗口和通知面板窗口是不能 AsyncRotation 的。     

```
DisplayContent.startAsyncRotation
  AsyncRotationController.<init>
    if (mHasScreenRotationAnimation)
      // 如果有屏幕旋转动画，那么就立即隐藏窗口
      // 但是在无缝旋转动画需要 Fixed Rotation，不需要立即旋转屏幕，mHideImmediately为false
      // 正常旋转屏幕时为true
      mHideImmediately = true
    WindowContainer.forAllWindows
      WindowState.forAllWindows
        WindowState.applyInOrderWithImeWindows
          WindowContainer$ForAllWindowsConsumerWrapper.apply
            AsyncRotationController.accept
              AsyncRotationController.canBeAsync
              mTargetWindowTokens.put(w.mToken, new Operation(action))
  AsyncRotationController.start()
    FadeAnimationController.fadeWindowToken()
      // 如果mHideImmediately为true，alpha 由 1 -> 1, duration 为 0
      // mHideImmediately 为false时，alpha 由 1 -> 0, duration 为200
      AsyncRotationController.getFadeOutAnimation()
      FadeAnimationController.createAnimationSpec
      FadeAnimationController.createAdapter
      WindowContainer.startAnimation
        SurfaceAnimator.startAnimation
```

```
Session.finishDrawing
  WindowManagerService.finishDrawingWindow
    WindowState.finishDrawing
      AsyncRotationController.handleFinishDrawing
        DisplayContent.finishAsyncRotation
          AsyncRotationController.completeRotation
            AsyncRotationController.finishOp
              FadeAnimationController.fadeWindowToken
                // alpha 由 0 -> 1, duration 为 200
                AsyncRotationController.getFadeInAnimation()
                WindowContainer.startAnimation
                  SurfaceAnimator.startAnimation
```



