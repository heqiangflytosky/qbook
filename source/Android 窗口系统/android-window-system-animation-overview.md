---
title: Android 窗口动画概述
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口动画
date: 2022-11-23 10:00:00
---

## 概述

本节开始介绍一下Android的窗口动画，先来列举一下窗口动画的一些特点：     

 - 和App动画一样，窗口动画添加的目的是使得窗口间的切换有更好的视觉效果，从而带来更好的体验。因此他本身不应该干预业务逻辑，也就是说就算把动画这段代码移掉，业务逻辑也应该正在执行。    
 - 虽然窗口动画是 Framework 层的动画，但是它本质上仍然和 App 内写动画一样，要么是加载动画的 xml 文件，要么是通过 Animator 对象。    
 - 因此我们分析窗口动画的重点不是动画本身，而是动画的播放流程和时机。      

我们把 Framework 层的窗口动画从动画执行的进程角度来分为两类来介绍：

 - 本地动画：动画的播放在 system_server 进程播放。比如，应用内添加一个窗口的动画就是在 system_server 进程内播放的。    
 - 远程动画：动画的播放在非 system_server 进程播放，比如，从桌面点击图标后有个图标开始放大铺满屏幕的动效，就是在 launcher 进程播放的，所以也是远程动画。    

下面我们首先以本地动画来简单认识一下窗口动画。     

## 准备工作

所谓本地窗口动画我们指的是在应用内创建一个新窗口时的动画。我们先来通过一个Demo来实现这样一个流程。    
添加和删除窗口：    
```
    fun addLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        var lp = WindowManager.LayoutParams()
        lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        lp.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
        lp.format = PixelFormat.RGBA_8888
        lp.gravity = Gravity.CENTER
        lp.width = WindowManager.LayoutParams.WRAP_CONTENT
        lp.height = WindowManager.LayoutParams.WRAP_CONTENT
        lp.windowAnimations = R.style.MyWindowAnimation
        lp.title = "WMSTestActivity 测试窗口"

        windowView = LayoutInflater.from(this).inflate(R.layout.window, null);
        windowManager.addView(windowView, lp)
        hasAddedWindow = true
    }

    fun removeLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        windowManager.removeView(windowView)
        hasAddedWindow = false
    }
```

动画：     
在res/values/styles.xml目录下添加styles     

```
    <style name="MyWindowAnimation">
        <item name="android:windowEnterAnimation">@anim/enter</item>
        <item name="android:windowExitAnimation">@anim/exit</item>
    </style>
```
创建res/anim/enter.xml和res/anim/exit.xml     
R.anim.enter    

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha android:fromAlpha="0"
        android:toAlpha="1.0"
        android:duration="1000"/>
</set>
```

R.anim.exit    

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha android:fromAlpha="1.0"
        android:toAlpha="0"
        android:duration="1000"/>
</set>
```

实现添加窗口是有一个透明度的变化效果。    

## 动画流程分析

分析动画的流程，我们从现象到本质来逐步分析，首先来看看动画过程中图层的变化。我们使用 Winscope 工具来观察一下。    

在没有添加窗口时节点情况：    
<img src="/images/android-window-system-animation-overview/1.png" width="596" height="152"/>
点击添加窗口后，在叶子节点下面创建WindowToken和WindowState对应的图层：    
<img src="/images/android-window-system-animation-overview/2.png" width="707" height="208"/>
调整窗口图层的层级到最前：    
<img src="/images/android-window-system-animation-overview/3.png" width="707" height="211"/>
WindowState对应的图层创建可以绘制内容的Layer：    
<img src="/images/android-window-system-animation-overview/4.png" width="716" height="231"/>
创建动画 leash 图层     
<img src="/images/android-window-system-animation-overview/5.png" width="700" height="333"/>

<img src="/images/android-window-system-animation-overview/6.png" width="694" height="271"/>
开始动画，可以看到窗口透明度的变化：    
<img src="/images/android-window-system-animation-overview/7.png" width="445" height="148"/>
动画完成，开始移除 leash 图层：   
<img src="/images/android-window-system-animation-overview/8.png" width="706" height="337"/>
leash 移除完成，添加动画结束。
<img src="/images/android-window-system-animation-overview/9.png" width="706" height="227"/>

窗口移除动画的流程类似，不同的就是透明度的变化是从1到0。    
整个过程可以简化如下图表示：   

<img src="/images/android-window-system-animation-overview/10.png" width="888" height="522"/>

动画的显示过程就是为其添加或移除的窗口和这个该窗口的父窗口之前添加一个层级（leash）用于显示动画；动画播放完成后，再移除这个层级（leash）。    
`Surface(name=89bedf7 WMSTestActivity 测试窗口)/@0x19fcb82 - animation-leash of window_animation#806 ` 中的 `window_animation` 表示动画的类型是窗口动画。还有一种动画类型是 `insets_animation`（一般是输入法、状态栏、导航栏等涉及），流程上大体也相同。     
比如：`Surface(name=9ab42ce InputMethod)/@0xbfab85 - animation-leash of insets_animation#808`。     
上面是本地动画，其实对于远端动画，基本原理也是一样的。      
不同的是本地动画是在 WindowToken 上面创建 leash，远端动画会创建在 Task 上面。    


<img src="/images/android-window-system-animation-overview/11.png" width="923" height="660"/>

## 认识 leash 图层

既然是动画，肯定是对某个“对象”不停的修改其相关属性来达到动画效果的，比如写 APP 的时候写的动画“对象”会是一个 View 。而当前分析的 framework 层动画的“对象”则是一个 Surface （图层）。      
前面我们提到，动画只是为了带来更好的体验，他本身不应该干预业务逻辑。    
因此，在Framework 做动画时，会创建一个 leash 图层，然后把需要做动画的图层挂载到其下面，再对 “leash” 图层做动画。从而不会对业务逻辑有影响。      
Leash 解释为：（牵狗的）皮带。其实这个就是 AOSP 单独为动画创建了一个图层，然后把需要做动画的容器图层挂载到这个图层下，那么就也有了动画效果。如果看到 leash 图层这个名词，指的就是做动画的那个图层了。     

## Leash 图层的创建

从上面流程可以看出，不论是添加还是移除窗口，都会创建类似 `Surface(name=89bedf7 WMSTestActivity 测试窗口)/@0x19fcb82 - animation-leash of window_animation#806` 这样一个 Surface，那么我们在代码中搜索 `animation-leash of` 看看具体创建的位置在哪里？    
在 SurfaceAnimator.createAnimationLeash 方法中找到这样的代码，在这里创建了 SurfaceControl 并命名为上面的类型。    
```
//SurfaceAnimator.java
    static SurfaceControl createAnimationLeash(Animatable animatable, SurfaceControl surface,
            Transaction t, @AnimationType int type, int width, int height, int x, int y,
            boolean hidden, Supplier<Transaction> transactionFactory) {
        ProtoLog.i(WM_DEBUG_ANIM, "Reparenting to leash for %s", animatable);
        final SurfaceControl.Builder builder = animatable.makeAnimationLeash()
                .setParent(animatable.getAnimationLeashParent())
                .setName(surface + " - animation-leash of " + animationTypeToString(type))
                // TODO(b/151665759) Defer reparent calls
                // We want the leash to be visible immediately because the transaction which shows
                // the leash may be deferred but the reparent will not. This will cause the leashed
                // surface to be invisible until the deferred transaction is applied. If this
                // doesn't work, you will can see the 2/3 button nav bar flicker during seamless
                // rotation.
                .setHidden(hidden)
                .setEffectLayer()
                .setCallsite("SurfaceAnimator.createAnimationLeash");
        final SurfaceControl leash = builder.build();
        t.setWindowCrop(leash, width, height);
        t.setPosition(leash, x, y);
        t.show(leash);
        t.setAlpha(leash, hidden ? 0 : 1);

        t.reparent(surface, leash);
        return leash;
    }
```

而对于 `animation-leash of` 后面的就是动画的类型，如下定义：     

```
// SurfaceAnimator.java
    static String animationTypeToString(@AnimationType int type) {
        switch (type) {
            case ANIMATION_TYPE_NONE: return "none";
            case ANIMATION_TYPE_APP_TRANSITION: return "app_transition";
            case ANIMATION_TYPE_SCREEN_ROTATION: return "screen_rotation";
            case ANIMATION_TYPE_DIMMER: return "dimmer";
            case ANIMATION_TYPE_RECENTS: return "recents_animation";
            case ANIMATION_TYPE_WINDOW_ANIMATION: return "window_animation";
            case ANIMATION_TYPE_INSETS_CONTROL: return "insets_animation";
            case ANIMATION_TYPE_TOKEN_TRANSFORM: return "token_transform";
            case ANIMATION_TYPE_STARTING_REVEAL: return "starting_reveal";
            case ANIMATION_TYPE_PREDICT_BACK: return "predict_back";
            default: return "unknown type:" + type;
        }
    }
```

然后我们就可以通过调试手段来先看看它的一个大致调用流程，具体代码分析我们后面再进行。    
```
Session.finishDrawing
    WindowManagerService.finishDrawingWindow
        WindowSurfacePlacer.requestTraversal
            mPerformSurfacePlacement.run()
```

```
WindowSurfacePlacer.performSurfacePlacement()
    WindowSurfacePlacer.performSurfacePlacementLoop()
        RootWindowContainer.performSurfacePlacement()
            RootWindowContainer.performSurfacePlacementNoTrace()
                RootWindowContainer.applySurfaceChangesTransaction()
                    DisplayContent.applySurfaceChangesTransaction()
                        WindowContainer.forAllWindows
                            WindowState.forAllWindows
                                WindowState.applyInOrderWithImeWindows()
                                    //执行callback
                                    DisplayContent.mApplySurfaceChangesTransaction
                                        WindowStateAnimator.commitFinishDrawingLocked()
                                            WindowState.performShowLocked()
                                                WindowStateAnimator.applyEnterAnimationLocked()
                                                    WindowStateAnimator.applyAnimationLocked()
                                                        WindowState.startAnimation()
                                                            WindowContainer.startAnimation()
                                                                SurfaceAnimator.startAnimation()
                                                                    SurfaceAnimator.createAnimationLeash()
```


```
WindowManagerService.removeWindow
    WindowState.removeIfPossible()
        WindowStateAnimator.applyAnimationLocked()
            WindowState.startAnimation()
                WindowContainer.startAnimation()
                    SurfaceAnimator.startAnimation()
                        SurfaceAnimator.createAnimationLeash()
```


上面介绍了两种动画的 createAnimationLeash 流程，下面来看看 removeLeash 的流程，这个流程对应两种动画都是一样的，在动画完成后删除 leash。    
首先在开始动画时添加了完成动画的回调。    

```
// LocalAnimationAdapter.java
    public void startAnimation(SurfaceControl animationLeash, Transaction t,
            @AnimationType int type, @NonNull OnAnimationFinishedCallback finishCallback) {
        mAnimator.startAnimation(mSpec, animationLeash, t,
                () -> finishCallback.onAnimationFinished(type, this));
    }
```

当动画完成后调用这个回调。    

```
LocalAnimationAdapter.startAnimation.callback
    SurfaceAnimator.OnAnimationFinishedCallback.onAnimationFinished
        SurfaceAnimator.getFinishedCallback()
            SurfaceAnimator.getFinishedCallback().resetAndInvokeFinish
                SurfaceAnimator.reset()
                    SurfaceAnimator.removeLeash()
```



