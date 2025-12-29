---
title: Android WindowContainer 动画
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android WindowContainer 动画
date: 2022-11-23 10:00:00
---


## 概述

本文主要围绕直接作用在窗口容器的动画来介绍，虽然使用 ShellTransition 后，窗口的动画一般是用这个架构了，但是部分动画仍旧依赖 SurfaceAnimator 来实现，比如窗口显示时的一个渐显动画，已经封装的针对 WindowToken 封装的 FadeAnimationController 等。      
它主要是用过 WindowContainer.startAnimation() 来实现窗口动画。     
前面的文章针对本地窗口动画的流程和细节做了一些介绍，而且版本比较老，本文主要是简单介绍如何使用 SurfaceAnimator 来实现一个 WindowContainer 动画。     
本文基于Android 16。     

## 核心类和方法

WindowContainer.startAnimation()    
SurfaceAnimator：在 Surface 上面运行动画。将对象窗口重新 reparent 到一个名为 “Leash” 的新 Surface 上来实现这一点。Leash 将附加到 Surface 层次结构中。然后，我们将在 Leash 传递给处理动画的组件来实现动画，该动画由 {@link AnimationAdapter} 指定。当动画完成后，将调用我们的回调以完成动画，此时我们将原来的 Surface 设置为原始 parent。    
SurfaceAnimationRunner：用来运行SurfaceAnimation，它是 WindowManagerService 的成员变量，通过 WindowContainer.getSurfaceAnimationRunner() 来获取。    

实现流程：    
 - 创建一个 Animation，设置执行容器的动画。
 - 创建 AnimationSpec，它封装了动画内容，内部包含一个Animation，用来具体实现一个动画，apply() 方法用来在动画过程中来具体更改Leash的属性来实现动画效果。    
 - 创建 AnimationAdapter，它是负责运行动画的组件，它包含 AnimationSpec 和 SurfaceAnimationRunner，LocalAnimationAdapter 是它的实现类之一。    
 - 调用 WindowContainer.startAnimation() 来开启动画。    

## 实现

```
    public void fadeWindowToken(boolean show, WindowToken windowToken, int animationType,
            SurfaceAnimator.OnAnimationFinishedCallback finishedCallback) {
        if (windowToken == null || windowToken.getParent() == null) {
            return;
        }

        final Animation animation = show ? getFadeInAnimation() : getFadeOutAnimation();
        final FadeAnimationAdapter animationAdapter = animation != null
                ? createAdapter(createAnimationSpec(animation), show, windowToken) : null;
        if (animationAdapter == null) {
            return;
        }
        windowToken.startAnimation(windowToken.getPendingTransaction(), animationAdapter,
                show /* hidden */, animationType, finishedCallback);
    }
```

## 流程

```
FadeAnimationController.fadeWindowToken
  WindowContainer.startAnimation
    SurfaceAnimator.startAnimation
      LocalAnimationAdapter.startAnimation
        SurfaceAnimationRunner.startAnimation
          // 把动画放到 mPendingAnimations
          mPendingAnimations.put(animationLeash, RunningAnimation)
          // post 执行 startAnimations
          mChoreographer.postFrameCallback(this::startAnimations)
          // 执行一次Transformation
          applyTransformation()
```

真正的开始执行动画：

```
mChoreographer.postFrameCallback(this::startAnimations)
  SurfaceAnimationRunner.startAnimations
    SurfaceAnimationRunner.startPendingAnimationsLocked
      SurfaceAnimationRunner.startAnimationLocked
        ValueAnimator.setDuration
        ValueAnimator.addUpdateListener
          onAnimationUpdate
            SurfaceAnimationRunner.applyTransformation
              RunningAnimation.mAnimSpec.apply()
                FadeAnimationController.LocalAnimationAdapter.AnimationSpec().apply()
                  SurfaceControl.Transaction.setAlpha
        ValueAnimator.addListener
        mRunningAnimations.put(a.mLeash, a)
        ValueAnimator.start()
```


