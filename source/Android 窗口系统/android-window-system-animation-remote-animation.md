---
title: Android 远程窗口动画流程
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 远程窗口动画流程
date: 2022-11-23 10:00:00
---

## 概述

前面我们介绍了本地动画，那么远程动画和本地动画的区别是什么呢？    
本地动画的创建和执行都是同一个进程，那么远程动画就不在同一个进程。    
比如桌面点击应用的启动动画，动画的Surface创建在 system_server 进程，而动画的执行是在 Launcher 进程。    
至于这样做的原因应该是更方便 Launcher 来定制启动动画。    

## 整体流程

### 整体流程图

<img src="/images/android-window-system-animation-remote-animation/00.png" width="918" height="1509"/>

### 窗口层级分析

1.创建 Task 和 ActivityRecord 容器。    

<img src="/images/android-window-system-animation-remote-animation/01.png" width="443" height="114"/>

2.创建 SplashScreen 窗口。    

<img src="/images/android-window-system-animation-remote-animation/02.png" width="518" height="143"/>

3.创建 WindowState 。    

<img src="/images/android-window-system-animation-remote-animation/03.png" width="506" height="157"/>

4.此时的窗口显示情况是这样的。显示的是 StartingWindow。    

<img src="/images/android-window-system-animation-remote-animation/04.png" width="188" height="413"/>

5.Activity 窗口完全显示，SplashScreen 删除。    

<img src="/images/android-window-system-animation-remote-animation/05.png" width="429" height="164"/>

### 图层分析

1.创建 Task 和 ActivityRecord 对应的图层。    

<img src="/images/android-window-system-animation-remote-animation/11.png" width="719" height="108"/>

2.创建 Splash Screen 对应图层。    

<img src="/images/android-window-system-animation-remote-animation/12.png" width="706" height="122"/>

3.创建 Splash Screen 绘制图层。    

<img src="/images/android-window-system-animation-remote-animation/13.png" width="720" height="167"/>

4.创建窗口动画Leash图层    

<img src="/images/android-window-system-animation-remote-animation/14.png" width="717" height="229"/>

5.创建 WindowState 对应图层    

<img src="/images/android-window-system-animation-remote-animation/15.png" width="706" height="278"/>

6.创建 starting_reveal 图层，为WindowState显示做动画准备    

<img src="/images/android-window-system-animation-remote-animation/16.png" width="704" height="421"/>

7.把1879图层移动到1887的子节点，开始做动画。创建1889图层作为 SplashScreen 的父节点，开始做消失动画。    

<img src="/images/android-window-system-animation-remote-animation/17.png" width="716" height="583"/>

8.WindowState动画完毕，删除 Leash 图层。

<img src="/images/android-window-system-animation-remote-animation/18.png" width="712" height="329"/>

9.SplashScreen 使命完成，删除 SplashScreen 图层。    

<img src="/images/android-window-system-animation-remote-animation/19.png" width="718" height="258"/>

10.Activity 启动完成，删除 Leash 图层。    

<img src="/images/android-window-system-animation-remote-animation/20.png" width="715" height="225"/>

## 过程分析

### 桌面启动应用

```
QuickstepLauncher.startActivitySafely()
    Launcher.startActivitySafely()
        QuickstepLauncher.getActivityLaunchOptions()
            QuickstepTransitionManager.getActivityLaunchOptions()
                new LauncherAnimationRunner()
                new RemoteAnimationAdapter()
                ActivityOptions.makeRemoteAnimation()
                Activity.startActivity()
```

### 系统初始化参数

```
ActivityTaskManagerService.startActivity()
    ActivityTaskManagerService.startActivityAsUser()
        ActivityStartController.obtainStarter()
        ActivityStarter.execute()
            ActivityStarter$Request.resolveActivity() //解析启动请求参数
            ActivityStarter.executeRequest()
                ActivityRecord.Builder.build() // 创建 ActivityRecord
                    new ActivityRecord()
                        ActivityRecord.setOptions()
                            // 初始化
                            mPendingRemoteAnimation = options.getRemoteAnimationAdapter()
                            mPendingRemoteTransition = options.getRemoteTransition()
```

### 系统创建 RemoteAnimationController

Launcher 完成 Pause 流程后，系统创建 RemoteAnimationController 准备开始动画。    

```
ActivityClientController.activityPaused()
    ActivityRecord.activityPaused()
        TaskFragment.completePause()
            RootWindowContainer.resumeFocusedTasksTopActivities()
                Task.resumeTopActivityUncheckedLocked()
                    Task.resumeTopActivityInnerLocked
                        TaskFragment.resumeTopActivity
                            ActivityRecord.applyOptionsAnimation()
                                AppTransition.overridePendingAppTransitionRemote()
                                    // 创建 RemoteAnimationController
                                    mRemoteAnimationController = new RemoteAnimationController()
                            // activity 生命周期流程
                            RootWindowContainer.ensureVisibilityAndConfig()
```

### 系统创建动画

App 请求 relayout 后，系统创建动画 leash，通知 Launcher 开始动画。    

```
RootWindowContainer.performSurfacePlacementNoTrace()
    // 在这里改变窗口状态，以及处理本地窗口动画相关，前面文章中已经介绍过
    //RootWindowContainer.applySurfaceChangesTransaction()
    // 执行 mAfterPrepareSurfacesRunnables 中的动画
    WindowAnimator.executeAfterPrepareSurfacesRunnables()
    RootWindowContainer.checkAppTransitionReady()
        AppTransitionController.handleAppTransitionReady()
            // 推迟动画执行，直到调用 continueStartingAnimations 为止
            SurfaceAnimationRunner.deferStartingAnimations()
            // 应用app transition动画（远程动画）
            AppTransitionController.applyAnimations()
                // 打开和关闭的窗口应用动画。这是通过调重载的applyAnimations方法完成的，
                // 传递相应的参数，如动画的目标、过渡类型等。
                AppTransitionController.applyAnimations(openingWcs, openingApps, transit)
                    WindowContainer.applyAnimation()
                        // 传递相关参数，创建AnimationAdapter和AnimationRunnerBuilder，准备启动动画
                        Task.applyAnimationUnchecked()
                            // 获取RecentsAnimationController,只有在最近任务中，切换到另一个应用时才会创建
                            WindowManagerService.getRecentsAnimationController()
                            WindowContainer.applyAnimationUnchecked()
                                // 重要方法：创建RemoteAnimationRecord
                                WindowContainer.getAnimationAdapter()
                                new AnimationRunnerBuilder()
                                WindowContainer.AnimationRunnerBuilder.build().startAnimation()
                                    WindowContainer.startAnimation()
                                        SurfaceAnimator.startAnimation
                                            // 创建mLeash
                                            SurfaceAnimator.createAnimationLeash()
                                            // 将leash传递给 RemoteAnimationAdapterWrapper
                                            RemoteAnimationController.RemoteAnimationAdapterWrapper.startAnimation()
                AppTransitionController.applyAnimations(closingWcs, closingApps, transit)
            // 处理closing activity可见性
            AppTransitionController.handleClosingApps()
            // 处理opening actvity可见性
            AppTransitionController.handleOpeningApps()
            // 处理用于处理正在切换的应用
            AppTransitionController.handleChangingApps()
            //处理正在关闭或更改的容器
            AppTransitionController.handleClosingChangingContainers()
            // 播放远程动画
            AppTransition.goodToGo()
                setAppTransitionState(APP_STATE_RUNNING)
                RemoteAnimationController.goodToGo()
                    // 创建动画完成时的回调函数
                    new FinishedCallback(this)
                    // 创建类型为App的RemoteAnimationTarget
                    RemoteAnimationController.createAppAnimations()
                        RemoteAnimationController.createRemoteAnimationTarget()
                            TaskFragment.createRemoteAnimationTarget()
                                WindowContainer.getTopMostActivity()
                                    ActivityRecord.getActivity()
                                ActivityRecord.createRemoteAnimationTarget()
                                    new RemoteAnimationTarget()
                    // 创建壁纸类型的RemoteAnimationTarget
                    RemoteAnimationController.createWallpaperAnimations()
                    //创建非app类型的RemoteAnimationTarget，例如状态栏，导航栏等
                    RemoteAnimationController.createNonAppWindowAnimations()
                    // 添加动画启动后的回调,把动画启动流程放到 WindowAnimator 的 mAfterPrepareSurfacesRunnables 中
                    // 等待执行
                    WindowAnimator.addAfterPrepareSurfacesRunnable()
                        // 重要方法：跨进程通信到桌面进程调用onAnimationStart方法
                        // 在这里执行 RemoteAnimationController.goodToGo 里面的 callback
                        RemoteAnimationController.goodToGo.callback
                            // 传递 RemoteAnimationController.FinishedCallback 到 Launcher
                            // 动画结束时跨进程调用到 system_server
                            IRemoteAnimationRunner.onAnimationStart()
            // SurfaceAnimationRunner.continueStartingAnimations()
```


RemoteAnimationTarget 主要用来存放动画图层。    


### 桌面执行动画

```
RemoteAnimationRunnerCompat.onAnimationStart()
    LauncherAnimationRunner.onAnimationStart()
        new AnimationResult // 保存 server 端的回调函数
        QuickstepTransitionManager.AppLaunchAnimationRunner.onAnimationStart()
            QuickstepTransitionManager.composeIconLaunchAnimator()
                QuickstepTransitionManager.getOpeningWindowAnimators()
            LauncherAnimationRunner$AnimationResult.setAnimation()
                AnimatorSet.addListener()
                    onAnimationEnd()
                        LauncherAnimationRunner$AnimationResult.finish()
                            mASyncFinishRunnable.run()
                                RemoteAnimationRunnerCompat.onAnimationStart.callback.run()
                                    // 重要方法：执行从 system_server 端传的 FinishedCallback
                                    finishedCallback.onAnimationFinished()
                AnimatorSet.start()
    
```

### 系统执行结束流程

```
RemoteAnimationController$FinishedCallback.onAnimationFinished()
    RemoteAnimationController.onAnimationFinished()
        SurfaceAnimator.reset()
```




