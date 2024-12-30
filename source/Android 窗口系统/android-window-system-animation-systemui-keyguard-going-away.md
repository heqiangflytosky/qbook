---
title: Android SystemUI 解锁动画
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android SystemUI 远程动画
date: 2022-11-23 10:00:00
---

## 概述

Android 系统解锁后，当前显示的界面会有个缩放的动画，这个缩放动画也是远程动画，和前面的桌面远程动画相比，它在 SystemUI 进程播放动画。    
本文基于 Android 14 分析。    

## 动画流程


### SystemUI 初始化 RemoteAnimationAdapter    

```
KeyguardService.onCreate()
    new RemoteAnimationDefinition()
    KeyguardViewMediator.getExitAnimationRunner()
        KeyguardViewMediator.validatingRemoteAnimationRunner
            new IRemoteAnimationRunner.Stub()
    new RemoteAnimationAdapter()
    RemoteAnimationDefinition.addRemoteAnimation(TRANSIT_OLD_KEYGUARD_GOING_AWAY,RemoteAnimationAdapter)
    ActivityTaskManager.getInstance().registerRemoteAnimationsForDisplay()
```

SystemUI 创建了 TRANSIT_OLD_KEYGUARD_GOING_AWAY 类型的动画，将其保存在 RemoteAnimationDefinition 对象，并通过 ATMS 的 registerRemoteAnimationsForDisplay() 方法来通知 Server 进行动画的创建和执行。    

### system_server 注册动画    

将 SystemUI 设置的 RemoteAnimationDefinition 保存在 AppTransitionController。    

```
ActivityTaskManagerService.registerRemoteAnimationsForDisplay
    DisplayContent.registerRemoteAnimations()
        AppTransitionController.registerRemoteAnimations()
```

### system_server 准备动画

开始解锁时，prepareAppTransition()

```
ActivityTaskManagerService.onTransact
    ActivityTaskManagerService.keyguardGoingAway()
        RootWindowContainer.forAllDisplays()  --> callback
            KeyguardController.keyguardGoingAway()
                DisplayContent.prepareAppTransition()
                    AppTransition.prepareAppTransition()
                        mNextAppTransitionRequests.add(transit)
                        
```

AppTransition 准备好时，创建 leash 和 RemoteAnimationTarget，传递给 SystemUI。    

```
RootWindowContainer.performSurfacePlacementNoTrace
    RootWindowContainer.checkAppTransitionReady
        AppTransitionController.handleAppTransitionReady
            AppTransitionController.overrideWithRemoteAnimationIfSet
                // 判断是否是解锁
                AppTransition.isKeyguardGoingAwayTransitOld
                // 根据transit类型从前面设置的 RemoteAnimationDefinitio 取出
                // 对应 RemoteAnimationAdapter
                mRemoteAnimationDefinition.getAdapter
                AppTransition.overridePendingAppTransitionRemote
                    //创建 RemoteAnimationController
                    mRemoteAnimationController = new RemoteAnimationController
            AppTransition.goodToGo()
                //这里的流程基本和桌面远程动画一致
                RemoteAnimationController.goodToGo()
                    RemoteAnimationController.createAppAnimations
                        RemoteAnimationController$RemoteAnimationRecord.createRemoteAnimationTarget
                            TaskFragment.createRemoteAnimationTarget
                                ActivityRecord.createRemoteAnimationTarget
                                    new RemoteAnimationTarget
                    WindowAnimator.addAfterPrepareSurfacesRunnable()
                        // 重要方法：跨进程通信到systemui进程调用onAnimationStart方法
                        RemoteAnimationController.goodToGo.callback
                            IRemoteAnimationRunner.onAnimationStart() --> systemui
```

### SystemUI 执行动画

设置 RemoteAnimationTarget

```
KeyguardViewMediator.validatingRemoteAnimationRunner.IRemoteAnimationRunner.Stub().onAnimationStart()
    KeyguardViewMediator.mExitAnimationRunner.onAnimationStart
        KeyguardViewMediator.startKeyguardExitAnimation
            mHandler.sendMessage(START_KEYGUARD_EXIT_ANIM)
                KeyguardViewMediator.handleMessage
                    KeyguardViewMediator.handleStartKeyguardExitAnimation
                        KeyguardUnlockAnimationController.notifyStartSurfaceBehindRemoteAnimation
                            // 设置需要做动画的Array<RemoteAnimationTarget>，
                            // 为后面做动画做准备
                            surfaceBehindRemoteAnimationTargets = targets
```

上划面板解锁时开始做动画：

```
CentralSurfacesImpl.onPanelExpansionChanged
    CentralSurfacesImpl.dispatchPanelExpansionForKeyguardDismiss
        KeyguardStateControllerImpl.notifyKeyguardDismissAmountChanged
            KeyguardUnlockAnimationController.onKeyguardDismissAmountChanged
                KeyguardUnlockAnimationController.showOrHideSurfaceIfDismissAmountThresholdsReached
                    KeyguardUnlockAnimationController.finishKeyguardExitRemoteAnimationIfReachThreshold
                        KeyguardUnlockAnimationController.setSurfaceBehindAppearAmount$default
                            KeyguardUnlockAnimationController.setSurfaceBehindAppearAmount
                                // 获取 leash ，设置各种属性，完成动画效果
                                surfaceBehindRemoteAnimationTarget.leash
                                SurfaceControl.Transaction().setMatrix
```


