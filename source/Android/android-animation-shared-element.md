---
title: Android 共享元素动画实现以及原理
categories: Android
comments: true
tags: [Attribute和Style]
description: 动画实践
date: 2016-7-2 10:00:00
---




## 概述



## 实现

ActivityA 启动 ActivityB，共享元素为一个按钮。    

ActivityA：    

```
    fun test1(view: View) {
        var intent = Intent(this,MainActivity2::class.java)
        var  options = ActivityOptionsCompat.makeSceneTransitionAnimation(this,view,"sharedButton")
        Log.d("TestAA","startActivity")
        startActivity(intent,options.toBundle())
    }
```


ActivityB：    

为元素设置 `android:transitionName`。     

```
    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:transitionName="sharedButton"
        android:text="测试"
        android:onClick="test1"/>
```

回调监听：
Activity A只触发setExitSharedElementCallback 的 SharedElementCallback，activity B只触发setEnterSharedElementCallback 的 SharedElementCallback。    

```
SharedElement A         com.meizu.flyme.myapplication        D  onMapSharedElements
SharedElement A         com.meizu.flyme.myapplication        D  onCaptureSharedElementSnapshot
SharedElementB          com.meizu.flyme.myapplication        D  onMapSharedElements
SharedElement A         com.meizu.flyme.myapplication        D  onSharedElementsArrived
SharedElementB          com.meizu.flyme.myapplication        D  onSharedElementsArrived
SharedElementB          com.meizu.flyme.myapplication        D  onRejectSharedElements
SharedElementB          com.meizu.flyme.myapplication        D  onCreateSnapshotView
SharedElementB          com.meizu.flyme.myapplication        D  onSharedElementStart
SharedElementB          com.meizu.flyme.myapplication        D  onSharedElementEnd
```

## 原理

在执行共享元素动画时，是取消了窗口切换的动画的，取而代之的是 View 动画。    

```
// public class DefaultTransitionHandler implements Transitions.TransitionHandler {
    private Animation loadAnimation(@WindowManager.TransitionType int type,
            @NonNull TransitionInfo info, @NonNull TransitionInfo.Change change,
            int wallpaperTransit, boolean isDreamTransition) {
            ......
        } else if (overrideType == ANIM_SCENE_TRANSITION) {
            // 如果有共享元素动画，不设置跳转动画
            // If there's a scene-transition, then jump-cut.
            return null;
        } 
```


### 关键类和方法

 - EnterTransitionCoordinator 和 ExitTransitionCoordinator
 - GhostView
 - Transition
 - SharedElementCallback：



### 动画流程


```
MainActivity.onCreate
  AppCompatActivity.setContentView
    AppCompatActivity.initViewTreeOwners
      PhoneWindow.getDecorView
        PhoneWindow.installDecor
          mSharedElementEnterTransition = PhoneWindow.getTransition(R.styleable.windowSharedElementEnterTransition)
            TransitionInflater.inflateTransition
              TransitionInflater.createTransitionFromXml
                TransitionSet.addTransition
                  TransitionSet.addTransitionInternal
                    mTransitions.add()
          mSharedElementExitTransition = PhoneWindow.getTransition(R.styleable.Window_windowSharedElementExitTransition)
```

frameworks/base/core/res/res/transition/move.xml    

```
        <item name="windowSharedElementEnterTransition">@transition/move</item>
        <item name="windowSharedElementExitTransition">@transition/move</item>
```

定义了一个 transitionSet 包含四个 Transition。    

```
<?xml version="1.0" encoding="utf-8"?>
-->
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
    <changeBounds/>
    <changeTransform/>
    <changeClipBounds/>
    <changeImageTransform/>
</transitionSet>
```

```
ActivityOptionsCompat.makeSceneTransitionAnimation
  ActivityOptions.makeSceneTransitionAnimation
    ActivityOptions.makeSceneTransitionAnimation
      new ExitTransitionCoordinator() // 创建 ExitTransitionCoordinator
        ActivityTransitionCoordinator.mapSharedElements
        ActivityTransitionCoordinator.viewsReady
Activity.startActivity       
  ComponentActivity.startActivityForResult
    Activity.startActivityForResult
      Activity.cancelInputsAndStartExitTransition
        ActivityTransitionState.startExitOutTransition
          ExitTransitionCoordinator.startExit
            // 将共享元素提升到 Overlay
            ActivityTransitionCoordinator.moveSharedElementsToOverlay
              GhostView.addGhost(view, decor, tempMatrix)
            ActivityTransitionCoordinator.startTransition
              ExitTransitionCoordinator.beginTransitions
                ExitTransitionCoordinator.mergeTransitions
                TransitionManager.beginDelayedTransition
                  TransitionSet.clone
                    TransitionSet.addTransitionInternal
```

```
ActivityThread.handleStartActivity
  Activity.performStart
    ActivityTransitionState.enterReady
      new EnterTransitionCoordinator
        EnterTransitionCoordinator.prepareEnter()
        send(MSG_SET_REMOTE_RECEIVER)
```

```
ViewRootImpl.performTraversals
  ViewTreeObserver.dispatchOnPreDraw
    TransitionManager$MultiListener.onPreDraw
      Transition.playTransition
        TransitionSet.createAnimators
          Transition.createAnimators
            ChangeBounds.createAnimator
              // 创建POSITION动画
              anim = ObjectAnimator.ofObject(view, POSITION_PROPERTY
            Visibility.createAnimator
              Visibility.onAppear
                Fade.onAppear
                  Fade.createAnimation
                    // 创建 alpha 动画
                    anim = ObjectAnimator.ofFloat(view, "transitionAlpha", endAlpha)
            mAnimators.add(animator)
        TransitionSet.runAnimators
          Transition.runAnimators
            Transition.runAnimator
              Animator.start()
```

## 相关文章

[Android Developer Shared Element Animation](https://developer.android.com/develop/ui/compose/animation/shared-elements?hl=zh-cn)        
[从源码了解共享元素的具体实现逻辑 ](https://juejin.cn/post/6950520259098968100)        
