---
title: Android 可预测返回手势动画
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 可预测返回手势动画
date: 2022-11-23 10:00:00
---

Predictive back gesture（预测性返回手势）是Android 13开始引入的新功能，允许用户在返回操作触发后预览上层界面，并决定是否继续返回。     
同时废弃了返回键相关的 API。     
本文基于 Android 16。    
如果应用想应用预测性返回手势，需要在 Manifest 中配置 `enableOnBackInvokedCallback` 属性。     

```
<application
    ...
    android:enableOnBackInvokedCallback="true"
    ... >
...

</application>
```

在 Activity中注册回调：     

```
        getOnBackInvokedDispatcher().registerOnBackInvokedCallback(0, () -> {
            Log.d("MainActivity2", "onBackInvoked");
        });
```

 

## 相关类

WmShell 代码在 `wm/shell/back` 目录。     

BackNavigationController：用于处理与 server 的后退手势相关操作。    

BackNavigationInfo：要发送到 SysUI 的有关返回事件的信息。    

返回事件类型：    
```
ublic final class BackNavigationInfo implements Parcelable {

    // 后退导航的目标未定义。
    public static final int TYPE_UNDEFINED = -1;

    // 向后导航将关闭当前可见的对话框
    public static final int TYPE_DIALOG_CLOSE = 0;

    // 向后导航会回到桌面
    public static final int TYPE_RETURN_TO_HOME = 1;

    // 向后导航会将用户带到同一 Task 中的上一个 Activity
    public static final int TYPE_CROSS_ACTIVITY = 2;

    // 向后导航会将用户带到上一个 Task 中的上一个Activity
    public static final int TYPE_CROSS_TASK = 3;

    // OnBackInvokedCallback 可用，需要被调用
    public static final int TYPE_CALLBACK = 4;
```
分别对应的动画实现：    
  - ShellBackAnimationRegistry
  - DefaultCrossActivityBackAnimation
  - CrossTaskBackAnimation
  - CustomCrossActivityBackAnimation
 

BackAnimationController:WMShell端控制用户启动后退手势时运行的窗口动画。    
ShellBackAnimationRegistry：WMShell端所有类型的默认后退动画的注册表     
注册各种返回类型注册动画：     

```
    public ShellBackAnimationRegistry(
            @ShellBackAnimation.CrossActivity @Nullable ShellBackAnimation crossActivityAnimation,
            @ShellBackAnimation.CrossTask @Nullable ShellBackAnimation crossTaskAnimation,
            @ShellBackAnimation.DialogClose @Nullable ShellBackAnimation dialogCloseAnimation,
            @ShellBackAnimation.CustomizeActivity @Nullable
                    ShellBackAnimation customizeActivityAnimation,
            @ShellBackAnimation.ReturnToHome @Nullable
                    ShellBackAnimation defaultBackToHomeAnimation) {
        if (crossActivityAnimation != null) {
            mAnimationDefinition.set(
                    BackNavigationInfo.TYPE_CROSS_ACTIVITY, crossActivityAnimation.getRunner());
        }
        if (crossTaskAnimation != null) {
            mAnimationDefinition.set(
                    BackNavigationInfo.TYPE_CROSS_TASK, crossTaskAnimation.getRunner());
        }
        if (dialogCloseAnimation != null) {
            mAnimationDefinition.set(
                    BackNavigationInfo.TYPE_DIALOG_CLOSE, dialogCloseAnimation.getRunner());
        }
        if (defaultBackToHomeAnimation != null) {
            mAnimationDefinition.set(
                    BackNavigationInfo.TYPE_RETURN_TO_HOME, defaultBackToHomeAnimation.getRunner());
        }
```

## Framework 禁用该功能

禁用 OnBackInvokedCallback 分发      

```
//WindowOnBackInvokedDispatcher.java
    private static final boolean ALWAYS_ENFORCE_PREDICTIVE_BACK = SystemProperties
            .getInt("persist.wm.debug.predictive_back_always_enforce", 0) != 0;
```

```
//WindowOnBackInvokedDispatcher.java
    public static boolean isOnBackInvokedCallbackEnabled(@Nullable ActivityInfo activityInfo,
            @NonNull ApplicationInfo applicationInfo,
            @NonNull Supplier<Context> contextSupplier) {
        // new back is enabled if the feature flag is enabled AND the app does not explicitly
        // request legacy back.
        if (!ENABLE_PREDICTIVE_BACK) {
            return false;
        }

        if (ALWAYS_ENFORCE_PREDICTIVE_BACK) {
            return true;
        }

        boolean requestsPredictiveBack;
        // Activity 是否使能该功能
        if (activityInfo != null && activityInfo.hasOnBackInvokedCallbackEnabled()) {
            requestsPredictiveBack = activityInfo.isOnBackInvokedCallbackEnabled();
            ......
            return requestsPredictiveBack;
        }

        // Application是否使能该功能
        requestsPredictiveBack = applicationInfo.isOnBackInvokedCallbackEnabled();
        ......
        if (requestsPredictiveBack) {
            return true;
        }

        ......
        return requestsPredictiveBack;
    }
}
```


```
//BackNavigationController.java
    static final boolean sPredictBackEnable =
            SystemProperties.getBoolean("persist.wm.debug.predictive_back", true);
```


禁用动画：    

Android 15上可以通过设置 predictive_back_system_anims 配置来修改，但是这个配置在 Android 16上已经去掉了。        

```
Flags.predictiveBackSystemAnims()
```

```
//BackAnimationController.java
    private void updateEnableAnimationFromFlags() {
        boolean isEnabled = predictiveBackSystemAnims() || isDeveloperSettingEnabled();
        mEnableAnimations.set(isEnabled);
    }
```


## 流程介绍

SystemUI 发起调用：      

```
BackAnimationController.onMotionEvent:524
  BackAnimationController.onGestureStarted:568
    BackAnimationController.startBackNavigation:588
      //调用 startBackNavigation，传入 mBackAnimationAdapter
      // 返回 BackNavigationInfo
      BackNavigationInfo = ActivityTaskManager.startBackNavigation(mNavigationObserver, mBackAnimationAdapter)
        ---> system_server 
      BackAnimationController.onBackNavigationInfoReceived()
        // 判断是否需要动画
        // 需要动画
        ShellBackAnimationRegistry.startGesture()
          BackAnimationRunner.startGesture()
        // 不需要动画
        BackNavigationInfo.getOnBackInvokedCallback()//获取 OnBackInvoked 回调
        BackTouchTracker.createStartEvent()
          new BackMotionEvent()
        BackAnimationController.tryDispatchOnBackStarted()
          BackAnimationController.dispatchOnBackStarted()
            IOnBackInvokedCallback.onBackStarted()
              // ---> 分发到 app 进程
              WindowOnBackInvokedDispatcher.onBackStarted()
                // 看有没有配置 OnBackInvokedCallback
                WindowOnBackInvokedDispatcher.OnBackInvokedCallbackWrapper.getBackAnimationCallback()
                
```



system_server:    

```
ActivityTaskManagerService.startBackNavigation(BackAnimationAdapter adapter):1911
  BackNavigationController.startBackNavigation:448
    // 设置back类型
    backType = BackNavigationInfo.TYPE_CROSS_ACTIVITY
    // 准备动画，设置回调等
    BackNavigationController$AnimationHandler.prepareAnimation()
    BackNavigationController.scheduleAnimation:670
      BackNavigationController$AnimationHandler$ScheduleAnimationBuilder.build:1931
        // 获取需要打开的 Activity
        openingActivities = getTopOpenActivities(mOpenTargets)
        BackNavigationController$AnimationHandler.composeAnimations:1239
          BackNavigationController$AnimationHandler.initiate:1156
            BackNavigationController$AnimationHandler$ScheduleAnimationBuilder.prepareTransitionIfNeeded:1875
              TransitionController.createTransition(TRANSIT_PREPARE_BACK_NAVIGATION)
              // 请求 PREDICTIVE_BACK 类型动画
              TransitionController.requestStartTransition:838
            // 创建关闭动画 AnimationAdapter
            mCloseAdaptor = BackNavigationController$AnimationHandler.createAdaptor:1476
              WindowContainer.startAnimation:2958
                SurfaceAnimator.startAnimation:201
                  // 创建 animation-leash of predict_back Leash 图层
                  SurfaceAnimator.createAnimationLeash:425
                  BackNavigationController$AnimationHandler$BackWindowAnimationAdaptor.startAnimation()
                    BackWindowAnimationAdaptor.createRemoteAnimationTarget()
            mOpenAnimAdaptor = new BackWindowAnimationAdaptorWrapper()
        BackNavigationController$AnimationHandler.applyPreviewStrategy(openingActivities)
          // 设置 Activity 图层可见
          WindowContainer.enforceSurfaceVisible()
      // 开始动画
      BackNavigationController.startAnimation()
        mPendingAnimation.run()
          BackNavigationController$AnimationHandler$ScheduleAnimationBuilder.lambda$build$0:1943
            IBackAnimationRunner.onAnimationStart() 
              --> 进程间通信到 SystemUI
              BackAnimationController.createAdapter().IBackAnimationRunner.onAnimationStart()
```



## PREDICTIVE_BACK 动画


动画播放      
以 TYPE_CROSS_ACTIVITY 来介绍。     

```
Transitions.onTransitionReady:849
  Transitions.dispatchReady:976
    Transitions.processReadyQueue:1036
      Transitions.playTransitionWithTracing:1127
        Transitions.playTransition:1147
          BackTransitionHandler.startAnimation:1308
            BackAnimationController.kickStartAnimation:1102
              BackAnimationController.startSystemAnimation:1062
                // 获取 BackAnimationRunner
                ShellBackAnimationRegistry.getAnimationRunnerAndInit(mBackNavigationInfo)
                // 开始动画并设置 finishedCallback = BackAnimationController.onBackAnimationFinished()
                BackAnimationRunner.startAnimation(finishedCallback):154
                  CrossActivityBackAnimation.onAnimationStart()
                BackAnimationRunner.dispatchOnBackStarted():702
                  CrossActivityBackAnimation$Callback.onBackStarted:537
                    BackProgressAnimator.onBackStarted()
                      // 设置动画过程中的callback
                      mCallback = callback = CrossActivityBackAnimation.onGestureProgress()
                      // 设置 Spring动画
                      SpringAnimation.setSpring(mGestureSpringForce)
                      BackProgressAnimator.onBackProgressed
                        // 开始 Spring动画
                        SpringAnimation.animateToFinalPosition()
```

动画过程中更新图层：   

```
CrossActivityBackAnimation.onGestureProgress
  CrossActivityBackAnimation.applyTransform(closingTarget?.leash)
  CrossActivityBackAnimation.applyTransform(enteringTarget?.leash)
```

进行缩放位移以及透明度变化。     

```
    protected fun applyTransform(
        leash: SurfaceControl?,
        rect: RectF,
        alpha: Float,
        baseTransformation: Transformation? = null,
        flingMode: FlingMode = FlingMode.NO_FLING
    ) {
        if (leash == null || !leash.isValid) return
        tempRectF.set(rect)
        if (flingMode != FlingMode.NO_FLING) {
            lastPostCommitFlingScale = min(
                postCommitFlingScale.value / SPRING_SCALE,
                if (flingMode == FlingMode.FLING_BOUNCE) 1f else lastPostCommitFlingScale
            )
            // apply an additional scale to the closing target to account for fling velocity
            tempRectF.scaleCentered(lastPostCommitFlingScale)
        }
        val scale = tempRectF.width() / backAnimRect.width()
        val matrix = baseTransformation?.matrix ?: transformMatrix.apply { reset() }
        val scalePivotX =
            if (isLetterboxed && enteringHasSameLetterbox) {
                closingTarget!!.localBounds.left.toFloat()
            } else {
                0f
            }
        matrix.postScale(scale, scale, scalePivotX, 0f)
        matrix.postTranslate(tempRectF.left, tempRectF.top)
        transaction
            .setAlpha(leash, alpha)
            .setMatrix(leash, matrix, tmpFloat9)
            .setCrop(leash, cropRect)
            .setCornerRadius(leash, cornerRadius)
    }
```

动画完成：    

```

```

动画完成，执行back：      

SystemUI    

```
BatchedInputEventReceiver.doConsumeBatchedInput:91
  InputEventReceiver.consumeBatchedInputEvents:259
    InputEventReceiver.nativeConsumeBatchedInputEvents
      InputEventReceiver.dispatchInputEvent:300
        InputChannelCompat$InputEventReceiver$1.onInputEvent:78
          EdgeBackGestureHandler$InputMonitorResource.onInputEvent:0
            EdgeBackGestureHandler.onInputEvent:870
              EdgeBackGestureHandler.onMotionEvent:1240
                EdgeBackGestureHandler.dispatchToBackAnimation:1268
                  BackAnimationController$BackAnimationImpl.onBackMotion:330
                    BackAnimationController.onMotionEvent:534
                      BackAnimationController.onGestureFinished:870
                        BackAnimationController.startPostCommitAnimation:895
                          BackAnimationController.dispatchOnBackInvoked:719
                            BackAnimationController.BackAnimationImpl.onBackMotion()



```

App    

```
WindowOnBackInvokedDispatcher.OnBackInvokedCallbackWrapper.onBackInvoked()
  Activity.onBackInvoked()
    ActivityClient.getInstance().onBackPressed()
```





