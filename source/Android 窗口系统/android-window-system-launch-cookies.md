---
title: Android LaunchCookie
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 LaunchCookie
date: 2022-11-23 10:00:00
---



## 概述

设置 LaunchCookie 可用于追踪因该选项而启动的 Activity 和 Task。如果启动的 Activity 立即启动另一个 Activity，Cookie 将被转移到下一个Activity。      
LaunchCookie 可以通过 startActivity 时通过 ActivityOptions 来设置，但是该接口是 `@hide` 的。       

```
// ActivityOptions.java

    /**
     * Sets a launch cookie that can be used to track the {@link Activity} and task that are
     * launched as a result of this option. If the launched activity is a trampoline that starts
     * another activity immediately, the cookie will be transferred to the next activity.
     *
     * @param launchCookie a developer specified identifier for a specific task.
     *
     * @hide
     */
    @SuppressLint("UnflaggedApi")
    @TestApi
    public void setLaunchCookie(@NonNull LaunchCookie launchCookie) {
        setLaunchCookie(launchCookie.binder);
    }
    
    /**
     * Sets a launch cookie that can be used to track the activity and task that are launch as a
     * result of this option. If the launched activity is a trampoline that starts another activity
     * immediately, the cookie will be transferred to the next activity.
     *
     * @hide
     */
    public void setLaunchCookie(IBinder launchCookie) {
        mLaunchCookie = launchCookie;
    }
```

应用场景，桌面启动 Activity 时设置包含快捷图标信息的 LaunchCookie，方便返回桌面时寻找动画的终点位置。      

## 设置流程


以桌面设置 launcheCookies 设置流程为例来介绍。     

### 设置 LaunchCookie

打开动画时通过 QuickstepInteractionHandler.onInteraction 设置 activityOptions.options.setLaunchCookie(launchCookie) 和 WindowManagerExt.getInstance(MzCommonUtils.getApplicationContext()).setLaunchCookie。        

```
QuickstepInteractionHandler.onInteraction
    activityOptions.options.setLaunchCookie(launchCookie)
    WindowManagerExt.getInstance(MzCommonUtils.getApplicationContext()).setLaunchCookie
```

```
ActivityTaskManagerService.startActivity
  ActivityTaskManagerService.startActivityAsUser
    ActivityStarter.execute
      // 获取 LaunchingState
      launchingState = ActivityMetricsLogger.notifyActivityLaunching 
        // 遍历 mTransitionInfoList 获取 existingInfo
        for (int i = mTransitionInfoList.size() - 1; i >= 0; i--) {
        // 如果 existingInfo == null，重新创建 LaunchingState，
        // 那么 launchingState.mAssociatedTransitionInfo 就为空
        launchingState = new LaunchingState()
        // 如果 existingInfo 不为空，就返回 existingInfo.mLaunchingState
        // 那么 launchingState.mAssociatedTransitionInfo 就不为空
        return existingInfo.mLaunchingState;
      ActivityStarter.executeRequest
        new ActivityRecord()
          ActivityRecord.mLaunchCookie = options.getLaunchCookie()
        ActivityStarter.startActivityUnchecked
          ActivityStarter.postStartActivityProcessing
              Task.getTaskInfo
                Task.fillTaskInfo
                  ActivityTaskSupervisor$TaskInfoHelper.fillAndReturnTop
                    ActivityTaskSupervisor$TaskInfoHelper.accept
                      TaskInfo.addLaunchCookie(ActivityRecord.mLaunchCookie)
      ActivityMetricsLogger.notifyActivityLaunched(LaunchingState launchingState)
        // 如果 launchingState.mAssociatedTransitionInfo 不为空而且 待启动的 Activity 属于当前的 transition
        // 其实也就是表示连续启动
        if (info != null && info.canCoalesce(launchedActivity)) {
        ActivityMetricsLogger$TransitionInfo.setLatestLaunchedActivity
          mLastLaunchedActivity.mLaunchCookie = null;// 把上一个ActivityRecord的cookies置空
            FlymeWindowManagerInjectImpl.handleActivityLaunchCookie(r)
              FlymeWindowManagerService.getLaunchCookie
        else // 也就是不连续启动
        ActivityMetricsLogger$TransitionInfo.create
          new ActivityMetricsLogger$TransitionInfo()
```

mTransitionInfoList 删除 TransitionInfo 的时机：       
首帧绘制完成，把 TransitionInfo 从 mTransitionInfoList 中删除，那么后面如果这个Activity再启动其他Activity，就不属于连续启动。        

```
ActivityRecord.onFirstWindowDrawn
  ActivityRecord.updateReportedVisibilityLocked
    ActivityRecord.onWindowsDrawn
      ActivityMetricsLogger.notifyWindowsDrawn
        ActivityMetricsLogger.done
          mTransitionInfoList.remove(info)
```



桌面插件动画流程：

手势上划：

server:

```
BLASTSyncEngine$SyncGroup.tryFinish
    Transition.onTransactionReady
        Transition.calculateTransitionInfo
            new TransitionInfo.Change
            tinfo = new ActivityManager.RunningTaskInfo()
            Task.fillTaskInfo(tinfo)
                ActivityTaskSupervisor.TaskInfoHelper.fillAndReturnTop
                    Task.forAllActivities
                        ActivityTaskSupervisor.TaskInfoHelper.accept
                            // 把 ActivityRecord mLaunchCookie 赋值给 TaskInfo
                            TaskInfo.addLaunchCookie(ActivityRecord.mLaunchCookie);
            TransitionInfo$Change.setTaskInfo
```



SystemUI： TransitionInfo.Change 转换为 RemoteAnimationTarget。      

```
Transitions.onTransitionReady
    Transitions.dispatchReady
        Transitions.processReadyQueue
            Transitions.playTransition
                RecentsTransitionHandler.startAnimation
                    RecentsTransitionHandler$RecentsController.start
                        TransitionUtil.newTarget
                            TransitionUtil.newTarget
                                TransitionUtil.newTarget
                                    taskInfo = change.getTaskInfo()
                                    new RemoteAnimationTarget(taskInfo)
                        IRecentsAnimationRunner.onAnimationStart()
```



Launcher：传递 Cookies，通过 RemoteAnimationTarget.taskInfo。         


```
IRecentsAnimationRunner$Stub.onTransact
    SystemUiProxy$1.onAnimationStart
        RecentsAnimationCallbacks.onAnimationStart
            LauncherSwipeHandlerV2.onRecentsAnimationStart
                AbsSwipeUpHandler.onRecentsAnimationStart
                    mRecentsAnimationTargets = targets
```


多任务返回，通过 findWorkspaceViewForF** 读取前面设置的图标的 id，然后匹配桌面图标 id 进行做动画。       

```
LauncherSwipeHandlerV2.onRecentsAnimationStart
  AbsSwipeUpHandler.onRecentsAnimationStart
    AbsSwipeUpHandler.flushOnRecentsAnimationAndLauncherBound
      AbsSwipeUpHandler.lambda$handleNormalGestureEnd$18
        AbsSwipeUpHandler.animateGestureEnd
          AbsSwipeUpHandler.animateToProgressInternal
            LauncherSwipeHandlerV2.createHomeAnimationFactory
              LauncherSwipeHandlerV2.findWorkspaceViewForF**
```


back:

通过 QuickstepTransitionManager.findLauncherView 获取图标 id 进行动画。     

```
QuickstepTransitionManager$WallpaperOpenLauncherAnimationRunner.onAnimationStart
  F**QuickstepTransitionManager.onWallpaperOpenAnimationStart
    F**QuickstepTransitionManager.createWallpaperOpenAnimations
      F**QuickstepTransitionManager.findLauncherView
        QuickstepTransitionManager.findLauncherView
```






