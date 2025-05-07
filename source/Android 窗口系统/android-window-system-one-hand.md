---
title: Android 单手模式
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 单手模式
date: 2022-11-23 10:00:00
---



## 概述

启用单手模式，需要在设置里面打开单手模式的开关。
通过图层我们可以看到，单手模式下，把窗口层级树下所有 OneHanded 节点的容器对应的图层都向下做了位移，做位移的容器是 OneHanded 节点对应的图层。    

OneHandedState:STATE_NONE -> STATE_ENTERING -> STATE_ACTIVE -> STATE_EXITING -> STATE_NONE    

OneHandedDisplayAreaOrganizer 中的变量 mDisplayAreaTokenMap 存储了所有的 FEATURE_ONE_HANDED 节点，OneHandedDisplayAreaOrganizer 继承了 DisplayAreaOrganizer，在其 onDisplayAreaAppeared() 方法中会把所有 OneHand 类型的节点放入到 mDisplayAreaTokenMap 中存储。         
在 SystemUI 初始化的时候，会调用     

```
            mDisplayAreaOrganizer.registerOrganizer(
                    OneHandedDisplayAreaOrganizer.FEATURE_ONE_HANDED);
```

来注册 FEATURE_ONE_HANDED 类型节点的监听，     

```
OneHandedController.onInit
    OneHandedController.updateSettings
        OneHandedController.setOneHandedEnabled
            OneHandedController.updateOneHandedEnabled
                OneHandedDisplayAreaOrganizer.registerOrganizer
                    DisplayAreaOrganizer.registerOrganizer
                    OneHandedDisplayAreaOrganizer.onDisplayAreaAppeared
```

后面单手模式的位移和动画都是针对 mDisplayAreaTokenMap 中的 FEATURE_ONE_HANDED 节点来操作。      

## 代码流程

桌面进程：    

```
TouchInteractionService.onInputEvent
    OneHandedModeInputConsumer.onMotionEvent
        case ACTION_MOVE:
            mPassedSlop = isInSystemGestureRegion(mLastPos)
        case ACTION_UP:
            if (mLastPos.y >= mDownPos.y && mPassedSlop)
            OneHandedModeInputConsumer.onStartGestureDetected()
                SystemUiProxy.startOneHandedMode()
                    IOneHanded.startOneHanded()
            
```




SystemUI 进程    

```
OneHandedController.IOneHandedImpl.startOneHanded()
    OneHandedController.startOneHanded()
        OneHandedState.setState(STATE_ENTERING)
            OneHandedTutorialHandler.onStateChanged()
                OneHandedTutorialHandler.createViewAndAttachToWindow
                    OneHandedTutorialHandler.attachTargetToWindow()
                        BackgroundWindowManager.showBackgroundLayer()
                            BackgroundWindowManager.initView()
                                new SurfaceControlViewHost()
        OneHandedDisplayAreaOrganizer.scheduleOffset()
            //  针对中的所有 FEATURE_ONE_HANDED 容器做动画
            mDisplayAreaTokenMap.forEach()
            OneHandedDisplayAreaOrganizer.animateWindows()
                // 开始下拉动画
                OneHandedAnimationController.OneHandedTransitionAnimator.setTransitionDirection
                OneHandedTransitionAnimator.addOneHandedAnimationCallback()
                    onOneHandedAnimationStart()
                    onOneHandedAnimationEnd()
                        OneHandedDisplayAreaOrganizer.finishOffset()
                            OneHandedController.OneHandedTransitionCallback.onStartFinished()
                                OneHandedState.setState(STATE_ENTERING)
                OneHandedTransitionAnimator.start()
                    onAnimationStart()
                    onAnimationEnd()
                    onAnimationUpdate()
                        OneHandedTransitionAnimator.applySurfaceControlTransaction
                            OneHandedSurfaceTransactionHelper.getSurfaceTransactionHelper()
                            OneHandedSurfaceTransactionHelper.crop().round().translate()
                            SurfaceControl.Transaction.apply()
```



超时处理：      


```
OneHandedTouchHandler.EventReceiver.onInputEvent()
    OneHandedTouchHandler.onInputEvent()
        OneHandedTouchHandler.onMotionEvent()
            OneHandedTimeoutHandler.resetTimer()
                MainExecutor.executeDelayed(OneHandedTimeoutHandler.onStop, mTimeoutMs)
```

```
OneHandedTimeoutHandler.onStop
    OneHandedController.setupTimeoutListener().lambda
        OneHandedController.stopOneHanded
            OneHandedState.setState(STATE_EXITING)
                // 开始位移动画
                OneHandedDisplayAreaOrganizer.scheduleOffset()
```

