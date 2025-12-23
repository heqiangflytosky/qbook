---
title: Android WMS UnknownAppVisibilityController
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android WMS UnknownAppVisibilityController
date: 2022-11-23 10:00:00
---



## 概述

先看一下这个类的文档解释：UnknownAppVisibilityController 是用来管理我们还不知道是否可见的 ActivityRecord 的 集合。比如，在锁屏显示时启动 Activity 时的场景。在这种情况下，应用程序可能设置的密码锁会影响它的可见性，因此我们等到应用可见后再开始过渡，以避免闪烁。     

mUnknownAppVisibilityController 是 DisplayContent 的成员变量，用来管理锁屏上启动应用的生命周期逻辑。     

## 流程简介

UnknownAppVisibilityController 管理的 ActivityRecord 有下面几个状态：

 - private static final int UNKNOWN_STATE_WAITING_RESUME = 1：等待Activity Resume的状态，这个也是初始状态
 - private static final int UNKNOWN_STATE_WAITING_RELAYOUT = 2：Activity已经完成 Resume，正在等待 relayout
 - private static final int UNKNOWN_STATE_WAITING_VISIBILITY_UPDATE = 3：Activity已经完成relayout，并设置了相应的Keyguard标志，正在等待AMS更新所有应用的可见性。

流程：    

App launched activity -> mUnknownApps.put -> UNKNOWN_STATE_WAITING_RESUME -> App resume finished activity -> UNKNOWN_STATE_WAITING_RELAYOUT -> App relayouted appWindow -> UNKNOWN_STATE_WAITING_VISIBILITY_UPDATE -> mUnknownApps.remove

### 添加 

```
ActivityStarter.startActivityUnchecked
  ActivityStarter.startActivityInner
    RootWindowContainer.resumeFocusedTasksTopActivities
      Task.resumeTopActivityUncheckedLocked
        Task.resumeTopActivityInnerLocked
          TaskFragment.resumeTopActivity
            ActivityTaskSupervisor.startSpecificActivity
              ActivityTaskSupervisor.realStartActivityLocked
                ActivityRecord.notifyUnknownVisibilityLaunchedForKeyguardTransition
                  UnknownAppVisibilityController.notifyLaunched
                    mUnknownApps.put(activity, UNKNOWN_STATE_WAITING_RESUME)
```

```
// ActivityRecord.java
    void notifyUnknownVisibilityLaunchedForKeyguardTransition() {
        // 只有在锁屏时才会执行下面逻辑
        if (mNoDisplay || !isKeyguardLocked()) {
            return;
        }

        mDisplayContent.mUnknownAppVisibilityController.notifyLaunched(this);
    }
```

```
// ActivityRecord
    void notifyUnknownVisibilityLaunchedForKeyguardTransition() {
        // No display activities never add a window, so there is no point in waiting them for
        // relayout.
        if (mNoDisplay || !isKeyguardLocked()) {
            return;
        }

        mDisplayContent.mUnknownAppVisibilityController.notifyLaunched(this);
    }
```

```
// UnknownAppVisibilityController.java

    void notifyLaunched(@NonNull ActivityRecord activity) {
        ....
        // 添加到 mUnknownApps 并设置状态
        if (!activity.mLaunchTaskBehind) {
            mUnknownApps.put(activity, UNKNOWN_STATE_WAITING_RESUME);
        } else {
            mUnknownApps.put(activity, UNKNOWN_STATE_WAITING_RELAYOUT);
        }
    }
```


### Resumed

```
ActivityClientController.activityResumed
  ActivityRecord.activityResumedLocked
    UnknownAppVisibilityController.notifyAppResumedFinished
      mUnknownApps.put(activity, UNKNOWN_STATE_WAITING_RELAYOUT)
```

### relayout 和 remove

```
IWindowSession$Stub.onTransact
  Session.relayout
    WindowManagerService.relayoutWindow
      UnknownAppVisibilityController.notifyRelayouted
        mUnknownApps.put(activity, UNKNOWN_STATE_WAITING_VISIBILITY_UPDATE)
        UnknownAppVisibilityController.notifyVisibilitiesUpdated
          mUnknownApps.removeAt
```

## 需要进行判断 isVisibilityUnknown() 的场景

当 mUnknownApps 不为空时，表示 Activity 还没有做好显示的准备。     

```
// UnknownAppVisibilityController.java
    boolean isVisibilityUnknown(ActivityRecord r) {
        if (mUnknownApps.isEmpty()) {
            return false;
        }
        return mUnknownApps.containsKey(r);
    }
```

在判断 ActivityRecord 是否同步完成时，有对 UnknownAppVisibilityController.isVisibilityUnknown 的判断。    

```
    boolean isSyncFinished(BLASTSyncEngine.SyncGroup group) {
    .....
        if (mDisplayContent != null && mDisplayContent.mUnknownAppVisibilityController
                .isVisibilityUnknown(this)) {
            return false;
        }
    ....
    }
```

所以，如果遇到锁屏界面启动应用有延迟5s的可以排查一下这里，曾遇到过 UnknownAppVisibilityController中 notifyRelayouted 和 notifyAppResumedFinished 时序异常导致 UnknownAppVisibilityController 无法移除观测 ActivityRecord，导致该容器所在同步组一直无法完成同步，没有及时进行动画引发的启动延迟。    

