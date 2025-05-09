---
title: Android AMS 之 Activity 启动流程
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 Android AMS 之 Activity 启动流程
date: 2022-11-23 10:00:00
---


本文简单介绍一下 Activity 的启动流程，尽量关注流程，不会介绍太多的代码细节。     
代码基于 Android U    

## 分析启动时多个Activity之间生命周期

### 通过 event 日志分析

我们先通过 event 日志来大致看一下：    

```
 2719  4421 I wm_task_created: 80
 2719  4421 I wm_task_moved: [80,80,0,1,2]
 2719  4421 I wm_task_to_front: [0,80,0]
 2719  4421 I wm_create_task: [0,80,80,0]
 2719  4421 I wm_create_activity: [0,76210624,80,com.hq.android.androiddemo/.MainActivity,android.intent.action.MAIN,NULL,NULL,270532608]
 2719  4421 I wm_task_moved: [80,80,0,1,2]
 2719  4421 I wm_pause_activity: [0,30674795,com.**.launcher/com.android.launcher3.uioverrides.QuickstepLauncher,userLeaving=true,pauseBackTasks]
20677 20677 I wm_on_top_resumed_lost_called: [30674795,com.android.launcher3.uioverrides.QuickstepLauncher,topStateChangedWhenResumed]
20677 20677 I wm_on_paused_called: [30674795,com.android.launcher3.uioverrides.QuickstepLauncher,performPause,0]
 2719 12699 I wm_add_to_stopping: [0,30674795,com.**.launcher/com.android.launcher3.uioverrides.QuickstepLauncher,makeInvisible]
 2719  4750 I wm_restart_activity: [0,76210624,80,com.hq.android.androiddemo/.MainActivity]
 2719  4750 I wm_set_resumed_activity: [0,com.hq.android.androiddemo/.MainActivity,minimalResumeActivityLocked - onActivityStateChanged]
21354 21354 I wm_on_create_called: [76210624,com.hq.android.androiddemo.MainActivity,performCreate,56]
21354 21354 I wm_on_start_called: [76210624,com.hq.android.androiddemo.MainActivity,handleStartActivity,1]
21354 21354 I wm_on_resume_called: [76210624,com.hq.android.androiddemo.MainActivity,RESUME_ACTIVITY,1]
21354 21354 I wm_on_top_resumed_gained_called: [76210624,com.hq.android.androiddemo.MainActivity,topStateChangedWhenResumed]
 2719  3020 I wm_activity_launch_time: [0,76210624,com.hq.android.androiddemo/.MainActivity,498]
 2719  3024 I wm_wallpaper_surface: [0,0,null]
 2719  3023 I wm_stop_activity: [0,30674795,com.**.launcher/com.android.launcher3.uioverrides.QuickstepLauncher]
20677 20677 I wm_on_stop_called: [30674795,com.android.launcher3.uioverrides.QuickstepLauncher,STOP_ACTIVITY_ITEM,3]
```

从桌面启动应用时，一般会走下面的流程：    

 1. 先创建承载Activity的容器，比如Task、ActivityRecord之类。     
 2. 暂停 Launcher Activity，执行 onPause 生命周期    
 3. 创建 App 新进程
 4. 2和3都完成后真正启动进程，App经历 onCreate->onStart->onResume 生命周期
 5. 执行 Launcher 的onStop 生命周期

新 Activity 启动前，首先系统会通知旧 Activity 调用 onTopResumedActivityChanged，然后旧 Activity 的 onPause 先调用，新Activity再执行 onCreate -> onStart -> onResume，然后旧 Activity 再 onStop，这是为了防止跳转 Activity 时，屏幕上是黑屏，用户体验极差。      
onPause中不能做重量级操作、耗时操作，使新的 Activity 尽快显示出来并切换到前台。          

这个流程设计到了四个进程：system_server 进程、Launcher进程、zygote 进程和新的App进程。     
system_server 和 App 进程通过 binder 进行通信，这里涉及到两个 binder 接口：    

 - IApplicationThread: 作为系统进程请求应用进程的接口;
 - IActivityManager: 作为应用进程请求系统进程的接口。

system_server 和 zygote 进程通过Socket进行通信。    

### 流程图

<img src="/images/android-framework-activity-starting-process/1.png" width="813" height="1229"/>

## 桌面启动应用

这里省略调桌面点击应用的启动流程。     

### system_server 流程


```
ActivityTaskManagerService.startActivity()
    ActivityTaskManagerService.startActivityAsUser()
        ActivityStartController.obtainStarter()
        ActivityStarter.execute()
            ActivityStarter$Request.resolveActivity() //解析启动请求参数
            ActivityStarter.executeRequest()
                ActivityRecord.Builder.build() // 创建 ActivityRecord
                    new ActivityRecord()
                ActivityStarter.startActivityUnchecked()
                    ActivityStarter.startActivityInner()
                        // 判断是否有可以重复使用的 Task，比如挂在已经存在的 Task
                        ActivityStarter.getReusableTask()
                        // 判断是否有可以回收使用的 Task
                        ActivityStarter.recycleTask()
                            ActivityStarter.setTargetRootTaskIfNeeded()
                                Task.moveTaskToFront()
                                    ActivityRecord.moveFocusableActivityToTop()
                                        Task.moveToFront()
                                            TaskDisplayArea.positionChildAt()
                                                TaskDisplayArea.positionChildTaskAt()
                                                    ActivityTaskSupervisor.updateTopResumedActivityIfNeeded()
                                                        ActivityRecord.scheduleTopResumedActivityChanged()
                                                            TopResumedActivityChangeItem.obtain()
                                                            // 通知 Launcher 客户端执行 TopResumedActivityChanged
                                                            ClientLifecycleManager.scheduleTransaction()
                        // 如果有可以复用的 Task，这里直接返回
                        if (startResult != START_SUCCESS) {
                            return startResult;
                        }
                        ActivityStarter.getOrCreateRootTask() //创建或者获取Task
                            RootWindowContainer.getOrCreateRootTask()
                                TaskDisplayArea.getOrCreateRootTask()
                                    Task.Builder.build()
                                        Task.Builder.buildInner()
                                            new Task()  // 创建 Task
                                        TaskDisplayArea.addChild
                                            TaskDisplayArea.addChildTask
                                                TaskDisplayArea.findPositionForRootTask // 为task寻找层级位置
                                                    TaskDisplayArea.findMaxPositionForRootTask
                                                        TaskDisplayArea.getPriority
                                                    TaskDisplayArea.findMinPositionForRootTask
                        ActivityStarter.setNewTask() //将Task与activityRecord 绑定
                            ActivityStarter.addOrReparentStartingActivity()
                        Task.moveToFront()  //移动Task到栈顶
                            TaskDisplayArea.positionChildAt()
                                TaskDisplayArea.positionChildTaskAt()
                                    ActivityTaskSupervisor.updateTopResumedActivityIfNeeded()
                        Task.startActivityLocked()
                            // 执行 prepareAppTransition
                            DisplayContent.prepareAppTransition(TRANSIT_OPEN)
                            // 添加 StartingWindow 流程
                            StartingSurfaceController.showStartingWindow()
                        RootWindowContainer.resumeFocusedTasksTopActivities()
                            Task.resumeTopActivityUncheckedLocked
                                Task.resumeTopActivityInnerLocked
                                    TaskFragment.resumeTopActivity
                                        // 暂停 LauncherActivity
                                        TaskDisplayArea.pauseBackTasks()
                                            WindowContainer.forAllLeafTasks()
                                                Task.forAllLeafTasks()
                                                    TaskDisplayArea.pauseBackTasks.callback
                                                        TaskFragment.forAllLeafTaskFragments()
                                                            TaskDisplayArea.pauseBackTasks.callback
                                                                TaskFragment.startPausing()
                                                                    TaskFragment.schedulePauseActivity()
                                                                        //构建 PauseActivityItem
                                                                        PauseActivityItem.obtain()
                                                                            new PauseActivityItem()
                                                                        // 触发Launcher 暂停
                                                                        ClientLifecycleManager.scheduleTransaction()
                                                                            ClientTransaction.schedule()
                                                                                // 和应用进程通知，执行pause
                                                                                IApplicationThread.scheduleTransaction()
                                        // 创建待启动的进程
                                        ActivityTaskManagerService.startProcessAsync
                                            ActivityManagerService.LocalService.startProcess()
                                                ActivityManagerService.startProcessLocked()
                                                    ProcessList.startProcessLocked()
        
```

这个流程有三个主要的地方和App进程交互，一个是通知 Launcher onTopResumedActivityChanged，一个是通知Launcher 执行pause流程，还有一个就是创建新 App 的进程。    

### Launcher 执行 onTopResumedActivityChanged

这个过程仅仅是通知 Activity 执行 onTopResumedActivityChanged，并不涉及到生命周期状态变更。    

```
ActivityThread.ApplicationThread.scheduleTransaction
    ClientTransactionHandler.scheduleTransaction()
        ClientTransaction.preExecute()
            TopResumedActivityChangeItem.preExecute()
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)
            ActivityThread.H.handleMessage()
                TransactionExecutor.execute()
                    TransactionExecutor.executeCallbacks()
                        ActivityTransactionItem.execute()
                            TopResumedActivityChangeItem.execute()
                                ActivityThread.handleTopResumedActivityChanged()
                                    ActivityThread.reportTopResumedActivityChanged()
                                        Activity.performTopResumedActivityChanged()
                                            Activity.onTopResumedActivityChanged()
```

### Launcher 执行 pause

Launcher 执行 pause 生命周期，设置 ON_PAUSE 状态，然后通知 system_server pause 执行完毕。    

```
ActivityThread.ApplicationThread.scheduleTransaction
    ClientTransactionHandler.scheduleTransaction()
        ClientTransaction.preExecute()
            PauseActivityItem.preExecute()
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)
            ActivityThread.H.handleMessage()
                TransactionExecutor.execute()
                    TransactionExecutor.executeCallbacks()
                    TransactionExecutor.executeLifecycleState()
                        TransactionExecutor.cycleToPath()
                        ActivityTransactionItem.execute
                            PauseActivityItem.execute
                                ActivityThread.handlePauseActivity
                                    ActivityThread.performPauseActivity
                                        ActivityThread.performPauseActivityIfNeeded
                                            Instrumentation.callActivityOnPause
                                                Activity.performPause
                                                    Activity.onPause()
                                            //变更生命周期状态
                                            ActivityThread$ActivityClientRecord.(ON_PAUSE)
                        PauseActivityItem.postExecute()
                            ActivityClient.activityPaused()
                                // 通知 system_server pause 执行完毕
                                IActivityClientController.activityPaused()
```

###  system_server 接收到 Launcher 的 activityPaused

再回到 system_server，接收 Launcher 的activityPaused通知：    

```
ActivityClientController.activityPaused()
    ActivityRecord.activityPaused()
        TaskFragment.completePause()
            RootWindowContainer.resumeFocusedTasksTopActivities()
                Task.resumeTopActivityUncheckedLocked()
                    Task.resumeTopActivityInnerLocked
                        TaskFragment.resumeTopActivity
                            RootWindowContainer.ensureVisibilityAndConfig()
                                RootWindowContainer.ensureActivitiesVisible()
                                    DisplayContent.ensureActivitiesVisible()
                                        WindowContainer.forAllRootTasks()
                                            Task.forAllRootTasks()
                                                DisplayContent.ensureActivitiesVisible().callback
                                                    Task.ensureActivitiesVisible()
                                                        Task.forAllLeafTasks()
                                                            Task.ensureActivitiesVisible().callback
                                                                EnsureActivitiesVisibleHelper.process()
                                                                    EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                                                        ActivityRecord.makeInvisible()
                                                                            ActivityRecord.addToStopping()
                                                                                // 先添加到 mStoppingActivities 列表中，后面合适时机执行
                                                                                ActivityTaskSupervisor.mStoppingActivities::add
                                                                                // 发送IDLE_NOW_MSG
                                                                                ActivityTaskSupervisor::scheduleIdle
                            ActivityTaskSupervisor.startSpecificActivity()
            RootWindowContainer.ensureActivitiesVisible()
                DisplayContent.ensureActivitiesVisible()
                    WindowContainer.forAllRootTasks()
                        Task.forAllRootTasks()
                            DisplayContent.ensureActivitiesVisible.callback
                                Task.ensureActivitiesVisible()
                                    Task.forAllLeafTasks()
                                        Task.ensureActivitiesVisible().callback
                                            TaskFragment.updateActivityVisibilities()
                                                EnsureActivitiesVisibleHelper.process()
                                                    EnsureActivitiesVisibleHelper.setActivityVisibilityState()
                                                        EnsureActivitiesVisibleHelper.makeVisibleAndRestartIfNeeded()
                                                            ActivityTaskSupervisor.startSpecificActivity()
                                                                // 判断是否要执行startActivity流程。  
                                                                ActivityTaskSupervisor.realStartActivityLocked
```

当 Launcher pause 流程执行完毕后，`ActivityTaskSupervisor.startSpecificActivity()` 先把 Launcher Activity 放到 mStoppingActivities 列表中，然后去判断新 App 进程是否启动完毕，如果启动完毕就调 startActivity 流程，如果没有准备完毕就继续等App的回调。    

```
    void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        // 判断进程以及主线程是否创建好，貌似一般条件不会完全满足
        if (wpc != null && wpc.hasThread()) {
            try {
                if (mPerfBoost != null) {
                    Slog.i(TAG, "The Process " + r.processName + " Already Exists in BG. So sending its PID: " + wpc.getPid());
                    mPerfBoost.perfHint(BoostFramework.VENDOR_HINT_FIRST_LAUNCH_BOOST, r.processName, wpc.getPid(), BoostFramework.Launch.TYPE_START_APP_FROM_BG);
                }
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
            knownToBeDead = true;
            // Remove the process record so it won't be considered as alive.
            mService.mProcessNames.remove(wpc.mName, wpc.mUid);
            mService.mProcessMap.remove(wpc.getPid());
        } else if (r.intent.isSandboxActivity(mService.mContext)) {
            Slog.e(TAG, "Abort sandbox activity launching as no sandbox process to host it.");
            r.finishIfPossible("No sandbox process for the activity", false /* oomAdj */);
            r.launchFailed = true;
            r.detachFromProcess();
            return;
        }

        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

        final boolean isTop = andResume && r.isTopRunningActivity();
        mService.startProcessAsync(r, knownToBeDead, isTop,
                isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
                        : HostingRecord.HOSTING_TYPE_ACTIVITY);
    }
```

### system_server 触发创建新进程

接下来我们介绍创建 App 进程这一环节。这部分代码流程在上面已经有，这里不做过多介绍。这部分流程是和上面的 Launcher 的 pause 流程是同时进行的。     
ProcessList.startProcessLocked() 中system_server 通过Socket进行跨进程通信通知 zygote 进程 fork 出一个新的 App 进程，来开启我们的目标App。    
在 App 进程中进程启动流程如下：    

```
ZygoteInit.main
    RuntimeInit$MethodAndArgsCaller.run
        Method.invoke
            ActivityThread.main
                ActivityThread.attach
                    IActivityManager.attachApplication()
```

app 进程启动后，首先是实例化 ActivityThread，并执行其main()函数：创建 ApplicationThread，Looper，Handler 对象，并开启主线程消息循环Looper.loop()。    

接下来 App 进程会和 AMS 进程跨进程通信来绑定 Application。

### system_server 接收到新进程创建完毕的通知

App 进程通知 system_server 进程执行 `ActivityManagerService.attachApplication(mAppThread)` 方法，用于初始化 Application 和 Activity。 在system_server进程中，`ActivityManagerService.attachApplication(mAppThread)`里依次初始化了Application和Activity，分别有2个关键函数：    
 - IApplicationThread.bindApplication()方法跨进程通知App主线程Handler 创建 Application 对象、绑定 Context 、执行 Application#onCreate() 生命周期。    
 - `mStackSupervisor.attachApplicationLocked()`方法中调用 `ActivityThread.ApplicationThread.scheduleLaunchActivity()` 方法，进而通过主线程 Handler消息通知创建 Activity 对象，然后再调用 `mInstrumentation.callActivityOnCreate()`执行 `Activity.onCreate()` 生命周期。


```
ActivityManagerService.attachApplication()
    ActivityManagerService.attachApplicationLocked()
        // 新的进程与应用绑定，App创建 Application 和 Activity
        IApplicationThread.bindApplication()
        ActivityManagerService.finishAttachApplicationInner()
            ActivityTaskManagerService$LocalService.attachApplication()
                RootWindowContainer.attachApplication()
                    RootWindowContainer$AttachApplicationHelper.process()
                        WindowContainer.forAllRootTasks()
                            Task.forAllRootTasks()
                                RootWindowContainer$AttachApplicationHelper.accept()
                                    WindowContainer.forAllActivities()
                                        ActivityRecord.forAllActivities()
                                            RootWindowContainer$AttachApplicationHelper.test()
                                                //真正的执行 StartActivity
                                                ActivityTaskSupervisor.realStartActivityLocked()
                                                    // 通知 App 进程执行 start 和 resume 流程
                                                    LaunchActivityItem.obtain()
                                                    ResumeActivityItem.obtain()
                                                    ClientTransaction.setLifecycleStateRequest()
                                                    ClientLifecycleManager.scheduleTransaction()
```

### App 执行 Application 和 Activity 创建，以及对应生命周期流程

Application.onCreate

```
ActivityThread.handleBindApplication()
    Instrumentation.callApplicationOnCreate()
        Application.onCreate()
```

Activity 生命周期：onCreate，onStart，onResume

```
ActivityThread.ApplicationThread.scheduleTransaction
    ClientTransactionHandler.scheduleTransaction()
        ClientTransaction.preExecute()
            LaunchActivityItem.preExecute()
            ResumeActivityItem.preExecute()
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)
            ActivityThread.H.handleMessage()
                TransactionExecutor.execute()
                    TransactionExecutor.executeCallbacks()
                        LaunchActivityItem.execute()
                            ActivityThread.handleLaunchActivity()
                                ActivityThread.performLaunchActivity()
                                    Instrumentation.callActivityOnCreate()
                                        Activity.performCreate()
                                            Activity.onCreate()
                                    setState(ON_CREATE)
                        LaunchActivityItem.postExecute()
                    TransactionExecutor.executeLifecycleState()
                        TransactionExecutor.cycleToPath()
                            TransactionExecutorHelper.getLifecyclePath()
                            TransactionExecutor.performLifecycleSequence()
                                    ActivityThread.handleStartActivity()
                                        Activity.performStart()
                                            Instrumentation.callActivityOnStart()
                                                Activity.onStart()
                                        setState(ON_START)
                        ResumeActivityItem.execute()
                            ActivityThread.handleResumeActivity()
                                ActivityThread.performResumeActivity()
                                    Activity.performResume()
                                        Instrumentation.callActivityOnResume()
                                            Activity.onResume()
                                    setState(ON_RESUME)
                        ResumeActivityItem.postExecute()
```



### system_server 触发 Launcher stop 流程

前面在 Launcher 执行完 pause 流程通知 system_server 执行ActivityRecord.activityPaused() 流程中，会把 Launcher Activity 放入到 mStoppingActivities 列表中，然后发送 IDLE_NOW_MSG ，等待合适时机执行 stop 流程。    
而且 stop 也确实是在 IDLE_NOW_MSG 消息中处理的。    

```
ActivityTaskSupervisor$ActivityTaskSupervisorHandler.handleMessage
    ActivityTaskSupervisor$ActivityTaskSupervisorHandler.handleMessageInner
        ActivityTaskSupervisor$ActivityTaskSupervisorHandler.activityIdleFromMessage
            ActivityTaskSupervisor.activityIdleInternal
                ActivityTaskSupervisor.processStoppingAndFinishingActivities
                    ActivityRecord.stopIfPossible
                        // 通知 App 执行 stop 流程
                        StopActivityItem.obtain
                        ClientLifecycleManager.scheduleTransaction
```

那么真正触发执行 stop 的时机，我们就要看看哪里调用 ActivityTaskSupervisor.scheduleIdle() 去触发 IDLE_NOW_MSG 的发送。    

```
    final void scheduleIdle() {
        if (!mHandler.hasMessages(IDLE_NOW_MSG)) {
            if (DEBUG_IDLE) Slog.d(TAG_IDLE, "scheduleIdle: Callers=" + Debug.getCallers(4));
            mHandler.sendEmptyMessage(IDLE_NOW_MSG);
        }
    }
```

```
RemoteAnimationController$FinishedCallback.onAnimationFinished
    RemoteAnimationController.onAnimationFinished
        WindowContainer.doAnimationFinished()
            ActivityRecord.onAnimationFinished
                ActivityTaskSupervisor.scheduleProcessStoppingAndFinishingActivitiesIfNeeded
                    ActivityTaskSupervisor.scheduleIdle
                        
```

Activity 切换动画做完后调用 ActivityTaskSupervisor.scheduleIdle 发送 IDLE_NOW_MSG，然后 system_server 通知 Launcher 执行 stop 流程。    

### Launcher 执行 stop

这部分和App执行其他生命周期流程类似，不做过多介绍。    
