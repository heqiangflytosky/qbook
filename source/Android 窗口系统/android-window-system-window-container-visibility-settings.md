---
title: 窗口动画和Activity生命周期与窗口可见性设置
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍窗口动画和Activity生命周期与窗口可见性设置
date: 2022-11-23 10:00:00
---


## 概述

我们讨论 WindowContainer 的可见性离不开 setVisibleRequested、isVisibleRequested()、setVisible()、isVisible() 这几个方法。        
isVisble，我们理解为“窗口可见性”，isVIsibleRequested，我们理解为“期望可见性”或者“请求可见性”。     

### isVisible

判断一个窗口是否可见，遍历当前 WindowContainer 的所有子容器，只要一个子容器的isVisible返回了true，或者说只要有一个子容器是可见的，那么当前WindowContainer就会被认为是可见的。     

```
//WindowContainer.java
    boolean isVisible() {
        for (int i = mChildren.size() - 1; i >= 0; --i) {
            final WindowContainer wc = mChildren.get(i);
            if (wc.isVisible()) {
                return true;
            }
        }
        return false;
    }
```

WindowContainer 的很多子类都重写了这个方法。    
ActivityRecord 通过 mVisible 属性来判断是否可见，mVisible 通过 setVisible 方法设置。    

```
//ActivityRecord.java
    boolean isVisible() {
        return mVisible;
    }
```

```
//WindowState.java
    boolean isVisible() {
        return wouldBeVisibleIfPolicyIgnored() && isVisibleByPolicyOrInsets();
    }
```

```
// DisplayContent.java
    boolean isVisible() {
        return true;
    }
```

```
//WallpaperWindowToken.java
    boolean isVisible() {
        return isClientVisible();
    }
```

而其他的子类，比如Task或者TaskDisplayArea，它们本身是没有一个成员变量来表示自身是否可见的（区别于下面的WindowContainer.isVisibleRequested是有一个WindowContainer.mVisibleRequested来表示isVisibleRequested的状态的），它们是否可见完全取决于它们的子容器是否可见。    
比如TaskDisplayArea是否可见依赖于它内部是否能寻到一个Task是可见的，而Task是否可见同样依赖于它内部是否能寻到一个ActivityRecord是可见的，所以在这个路径下，最终有决定权的是ActivityRecord。   

### setVisible



```
//ActivityRecord.java
    void setVisible(boolean visible) {
        if (visible != mVisible) {
            Slog.e("TestAA","setVisible "+visible+", "+this,new Throwable());
            mVisible = visible;
            if (app != null) {
                mTaskSupervisor.onProcessActivityStateChanged(app, false /* forceBatch */);
            }
            scheduleAnimation();
        }
    }
```

### isVisibleRequested

不同于刚刚的WindowContainer.isVisible，这里的WindowContainer.isVisibleRequested是有一个请求的意思，即这个WindowContainer将要变成可见/不可见，虽然它现在仍然是不可见/可见，因此我称为期望可见性。它和isVisible的主要区别在Transition流程中会非常明显的表现出来。    

```
// WindowContainer.java
    boolean isVisibleRequested() {
        return mVisibleRequested;
    }
```

```
// WindowState.java
    boolean isVisibleRequested() {
        final boolean localVisibleRequested =
                wouldBeVisibleRequestedIfPolicyIgnored() && isVisibleByPolicyOrInsets();
        if (localVisibleRequested && shouldCheckTokenVisibleRequested()) {
            return mToken.isVisibleRequested();
        }
        return localVisibleRequested;
    }
```

```
//DisplayContent.java
    boolean isVisibleRequested() {
        return isVisible() && !mRemoved && !mRemoving;
    }
```

### setVisibleRequested

WindowContainer提供了setVisibleRequested方法来让所有WindowContainer可以主动调用此方法来设置自身的期望可见性mVisibleRequested的值，但是实际上用到这个方法只有ActivityRecord和WallpaperWindowToken。    

```
// WindowContainer.java

    boolean setVisibleRequested(boolean visible) {
        if (mVisibleRequested == visible) return false;
           mVisibleRequested = visible;
        final WindowContainer parent = getParent();
        if (parent != null) {
            parent.onChildVisibleRequestedChanged(this);
        }

        // Notify listeners about visibility change.
        for (int i = mListeners.size() - 1; i >= 0; --i) {
            mListeners.get(i).onVisibleRequestedChanged(mVisibleRequested);
        }
        return true;
    }
```

```
// ActivityRecord.java
    boolean setVisibleRequested(boolean visible) {
        if (!super.setVisibleRequested(visible)) return false;
        setInsetsFrozen(!visible);
        updateVisibleForServiceConnection();
        if (app != null) {
            mTaskSupervisor.onProcessActivityStateChanged(app, false /* forceBatch */);
        }
        logAppCompatState();
        if (!visible) {
            final InputTarget imeInputTarget = mDisplayContent.getImeInputTarget();
            mLastImeShown = imeInputTarget != null && imeInputTarget.getWindowState() != null
                    && imeInputTarget.getWindowState().mActivityRecord == this
                    && mDisplayContent.mInputMethodWindow != null
                    && mDisplayContent.mInputMethodWindow.isVisible();
            finishOrAbortReplacingWindow();
        }
        return true;
    }
```

```
//WallpaperWindowToken.java
    protected boolean setVisibleRequested(boolean visible) {
        if (!super.setVisibleRequested(visible)) return false;
        setInsetsFrozen(!visible);
        return true;
    }
```

### setClientVisible

### onChildVisibleRequestedChanged

既然只有 ActivityRecord 和 WallpaperWindowToken 会调用 setVisibleRequested 来设置自身的期望可见性 mVisibleRequested 的值，那么其它类型的 WindowContainer，比如 Task 和 TaskDisplayArea，它们是如何修改自身的期望可见性 mVisibleRequested 的值呢？    
这就涉及到了 WindowContainer.onChildVisibleRequestedChanged 方法。      
WindowContainer.onChildVisibleRequestedChanged 方法就是在 WindowContainer.setVisibleRequested 中被调用的，很简单，当 ActivityRecord 调用 setVisibleRequested 的时候，它会主动调用 parent 的 WindowContainer.onChildVisibleRequestedChanged 方法，来看它的 parent 是否需要设置自己的期望可见性 mVisibleRequested 的值。    
再看 WindowContainer.onChildVisibleRequestedChanged 方法的内容：    
1）、如果子容器的期望可见性为true，自身的期望可见性为false，那么直接设置自身期望可加性为true。    
2）、如果子容器的期望可见性为false，自身的期望可见性为true，那么继续寻找其它子容器，只要能找到一个子容器的期望可见性仍然为true，那么当前 WindowContainer 的期望可见性仍然保持为true。    
这里的逻辑和 WindowContainer.isVisible 其实很像。    


## ActivityRecord 可见性

这里我们重点介绍一下 ActivityRecord可见性，因为它和 Activity 关系比较密切。    
ActivityRecord 有两个表示可见性的成员变量了：

 - mVisibleRequsted，期望可见性。
 - mVisible，可见性。

以 ActivityA 启动 ActivityB 为例来介绍一下 ActivityRecord 可见性的变化。

### ActivityB 设置期望可见性 true

这个过程是在 ActivityA pause之后发起的

```
06-23 17:19:01.132  7771  8968 E TestAA  : setVisibleRequested true, ActivityRecord{848f8d3 u0 com.hq.android.androiddemo/.performance.PerfActivity t859}
06-23 17:19:01.132  7771  8968 E TestAA  : java.lang.Throwable
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.setVisibleRequested(WindowContainer.java:1363)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibleRequested(ActivityRecord.java:5728)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibility(ActivityRecord.java:5840)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibility(ActivityRecord.java:5776)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.makeVisibleIfNeeded(ActivityRecord.java:6618)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.EnsureActivitiesVisibleHelper.setActivityVisibilityState(EnsureActivitiesVisibleHelper.java:217)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.EnsureActivitiesVisibleHelper.process(EnsureActivitiesVisibleHelper.java:136)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.updateActivityVisibilities(TaskFragment.java:1366)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.lambda$ensureActivitiesVisible$19(Task.java:5165)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task$$ExternalSyntheticLambda3.accept(D8$$SyntheticClass:0)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.forAllLeafTasks(Task.java:3237)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.ensureActivitiesVisible(Task.java:5164)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent.lambda$ensureActivitiesVisible$46(DisplayContent.java:6734)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent$$ExternalSyntheticLambda29.accept(D8$$SyntheticClass:0)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.forAllRootTasks(Task.java:3249)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2236)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent.ensureActivitiesVisible(DisplayContent.java:6733)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.ensureActivitiesVisible(RootWindowContainer.java:1964)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.ensureVisibilityAndConfig(RootWindowContainer.java:1816)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityTaskSupervisor.realStartActivityLocked(ActivityTaskSupervisor.java:893)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityTaskSupervisor.startSpecificActivity(ActivityTaskSupervisor.java:1146)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.resumeTopActivity(TaskFragment.java:1769)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.resumeTopActivityInnerLocked(Task.java:5364)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.Task.resumeTopActivityUncheckedLocked(Task.java:5296)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2663)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2649)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.completePause(TaskFragment.java:2061)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.activityPaused(ActivityRecord.java:6931)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityClientController.activityPaused(ActivityClientController.java:237)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at android.app.IActivityClientController$Stub.onTransact(IActivityClientController.java:698)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityClientController.onTransact(ActivityClientController.java:173)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at android.os.Binder.execTransactInternal(Binder.java:1507)
06-23 17:19:01.132  7771  8968 E TestAA  : 	at android.os.Binder.execTransact(Binder.java:1451)
```

在准备开始做动画时会通过 commitVisibility 再次设置，此时 VisibleRequested 已经为true，直接返回。 

那么这里为什么不直接把 ActivityB 的 mVisible 也设置为 true 呢？因为 ActivityB 可能还没有绘制好。如果绘制完成开始做动画时会设置 mVisible。    

### ActivityA 设置期望可见性 false

和ActivityRecordB的mVisibleRequested设置为true都是在 ActivityA pause 流程中。

```
06-23 17:19:01.133  7771  8968 E TestAA  : setVisibleRequested false, ActivityRecord{1119a51 u0 com.hq.android.androiddemo/.MainActivity t859}
06-23 17:19:01.133  7771  8968 E TestAA  : java.lang.Throwable
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.setVisibleRequested(WindowContainer.java:1363)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibleRequested(ActivityRecord.java:5728)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibility(ActivityRecord.java:5840)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisibility(ActivityRecord.java:5776)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.makeInvisible(ActivityRecord.java:6653)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.EnsureActivitiesVisibleHelper.setActivityVisibilityState(EnsureActivitiesVisibleHelper.java:227)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.EnsureActivitiesVisibleHelper.process(EnsureActivitiesVisibleHelper.java:136)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.updateActivityVisibilities(TaskFragment.java:1366)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.lambda$ensureActivitiesVisible$19(Task.java:5165)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task$$ExternalSyntheticLambda3.accept(D8$$SyntheticClass:0)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.forAllLeafTasks(Task.java:3237)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.ensureActivitiesVisible(Task.java:5164)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent.lambda$ensureActivitiesVisible$46(DisplayContent.java:6734)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent$$ExternalSyntheticLambda29.accept(D8$$SyntheticClass:0)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.forAllRootTasks(Task.java:3249)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2243)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2236)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.DisplayContent.ensureActivitiesVisible(DisplayContent.java:6733)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.ensureActivitiesVisible(RootWindowContainer.java:1964)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.ensureVisibilityAndConfig(RootWindowContainer.java:1816)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityTaskSupervisor.realStartActivityLocked(ActivityTaskSupervisor.java:893)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityTaskSupervisor.startSpecificActivity(ActivityTaskSupervisor.java:1146)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.resumeTopActivity(TaskFragment.java:1769)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.resumeTopActivityInnerLocked(Task.java:5364)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.Task.resumeTopActivityUncheckedLocked(Task.java:5296)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2663)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2649)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.TaskFragment.completePause(TaskFragment.java:2061)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityRecord.activityPaused(ActivityRecord.java:6931)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityClientController.activityPaused(ActivityClientController.java:237)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at android.app.IActivityClientController$Stub.onTransact(IActivityClientController.java:698)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at com.android.server.wm.ActivityClientController.onTransact(ActivityClientController.java:173)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at android.os.Binder.execTransactInternal(Binder.java:1507)
06-23 17:19:01.133  7771  8968 E TestAA  : 	at android.os.Binder.execTransact(Binder.java:1451)
```

另外在转场动画结束时也会在 Transition.finishTransition 中通过 ActivityRecord.commitVisibility 再次设置，此时 VisibleRequested 以及为false，会直接 return。    

### ActivityB 设置可见性 true

当窗口绘制完成，可以做动画了，那么就可以设置容器可见了。    

```
06-23 17:23:58.377  7771  7825 E TestAA  : setVisible true, ActivityRecord{bf59f44 u0 com.hq.android.androiddemo/.performance.PerfActivity t859}
06-23 17:23:58.377  7771  7825 E TestAA  : java.lang.Throwable
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisible(ActivityRecord.java:5713)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.ActivityRecord.commitVisibility(ActivityRecord.java:6021)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.Transition.commitVisibleActivities(Transition.java:2282)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.Transition.onTransactionReady(Transition.java:1785)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.finishNow(BLASTSyncEngine.java:290)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.tryFinish(BLASTSyncEngine.java:222)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.-$$Nest$mtryFinish(Unknown Source:0)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.BLASTSyncEngine.onSurfacePlacement(BLASTSyncEngine.java:605)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.RootWindowContainer.performSurfacePlacementNoTrace(RootWindowContainer.java:831)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.RootWindowContainer.performSurfacePlacement(RootWindowContainer.java:776)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop(WindowSurfacePlacer.java:177)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:126)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:115)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.wm.WindowSurfacePlacer$Traverser.run(WindowSurfacePlacer.java:57)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at android.os.Looper.loop(Looper.java:317)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 17:23:58.377  7771  7825 E TestAA  : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

### ActivityA 设置可见性 false

当动画结束时，去设置 ActivityA 为不可见。    

```
06-23 17:19:01.859  7771  9433 E TestAA  : setVisible false, ActivityRecord{1119a51 u0 com.hq.android.androiddemo/.MainActivity t859}
06-23 17:19:01.859  7771  9433 E TestAA  : java.lang.Throwable
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.ActivityRecord.setVisible(ActivityRecord.java:5713)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.ActivityRecord.commitVisibility(ActivityRecord.java:6021)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.Transition.finishTransition(Transition.java:1368)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.TransitionController.finishTransition(TransitionController.java:975)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.WindowOrganizerController.finishTransition(WindowOrganizerController.java:498)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at android.window.IWindowOrganizerController$Stub.onTransact(IWindowOrganizerController.java:289)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at com.android.server.wm.WindowOrganizerController.onTransact(WindowOrganizerController.java:208)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at android.os.Binder.execTransactInternal(Binder.java:1512)
06-23 17:19:01.859  7771  9433 E TestAA  : 	at android.os.Binder.execTransact(Binder.java:1451)
```

### mVisibleAtTransitionEndTokens

Transition 中有个 mVisibleAtTransitionEndTokens 集合，用来保存在动画结束时需要可见的容器。    

## 图层可见性

前面也只是在容器层面设置了可见性，那么真正的要显示出来，还是需要SF正在把这个图层合成以及提交显示才行。而请求SF的过程我们需要跟踪一下 SurfaceControl.Transaction 的相关方法。        
以 ActivityA 启动 ActivityB 为例来介绍一下。        

ActivityA 设置 hide 的时机。    
在动画 onTransitionReady 时会设置 Transaction mFinishT，在 mFinishT hide ActivityA。     
```
06-23 15:43:51.018 17224 17390 D TestHQ8 : Transaction showorhide hide Surface(name=ActivityRecord{fdbbc8e u0 com.hq.android.androiddemo/.MainActivity t842})/@0x219568d, TransactionID = 68032281971426
06-23 15:43:51.018 17224 17390 D TestHQ8 : java.lang.Throwable
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.hide(SurfaceControl.java:3080)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.setupStartState(Transitions.java:617)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.dispatchReady(Transitions.java:910)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onTransitionReady(Transitions.java:808)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 15:43:51.018 17224 17390 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```

Transaction apply hide 的时机：    
在动画完成后执行 mFinishT.apply()，此时去 hide ActivityA。    

```
06-23 15:43:51.573 17224 17390 D TestHQ8 : Transaction apply TransactionID = 68032281971426, this = android.view.SurfaceControl$Transaction@5336d2b
06-23 15:43:51.573 17224 17390 D TestHQ8 : java.lang.Throwable
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onFinish(Transitions.java:1209)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.lambda$playTransition$3(Transitions.java:1060)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.$r8$lambda$96XCaSAhJXxB3nlVJy1TNPeH46k(Unknown Source:0)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$$ExternalSyntheticLambda6.onTransitionFinished(D8$$SyntheticClass:0)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.lambda$startAnimation$1(DefaultTransitionHandler.java:329)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.$r8$lambda$2ee7viqDfcH2oT9s1JvE2VA6hu4(Unknown Source:0)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler$$ExternalSyntheticLambda6.run(D8$$SyntheticClass:0)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.lambda$buildSurfaceAnimation$7(DefaultTransitionHandler.java:864)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler$$ExternalSyntheticLambda10.run(D8$$SyntheticClass:0)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 15:43:51.573 17224 17390 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```

另外在 prepareSurfaces() 方法中，如果检测到了 mVisible false，也会结合其他条件判断来通过 getSyncTransaction()设置发起 hide 调用。     

```
06-23 16:03:37.312 23932 24007 D TestHQ8 : Transaction showorhide hide Surface(name=ActivityRecord{197651 u0 com.hq.android.androiddemo/.MainActivity t846})/@0x7d9973e, TransactionID = 102787157328515
06-23 16:03:37.312 23932 24007 D TestHQ8 : java.lang.Throwable
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.hide(SurfaceControl.java:3080)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.ActivityRecord.prepareSurfaces(ActivityRecord.java:8270)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.TaskFragment.prepareSurfaces(TaskFragment.java:3275)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.Task.prepareSurfaces(Task.java:3384)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayContent.prepareSurfaces(DisplayContent.java:5731)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.animate(WindowAnimator.java:143)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.lambda$new$1(WindowAnimator.java:99)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1558)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1569)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doCallbacks(Choreographer.java:1162)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doFrame(Choreographer.java:1087)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

这里会 merge Transaction。    

```
06-23 16:03:37.312 23932 24007 D TestHQ8 : Transaction merge other TransactionID = 102787157328515, this = 102787157328524
06-23 16:03:37.312 23932 24007 D TestHQ8 : java.lang.Throwable
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.merge(SurfaceControl.java:4585)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.animate(WindowAnimator.java:168)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.lambda$new$1(WindowAnimator.java:99)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1558)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1569)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doCallbacks(Choreographer.java:1162)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doFrame(Choreographer.java:1087)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 16:03:37.312 23932 24007 D TestHQ8 : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

apply 时机    

```
06-23 16:03:37.313 23932 24007 D TestHQ8 : Transaction apply TransactionID = 102787157328524, this = android.view.SurfaceControl$Transaction@dc9c8a2
06-23 16:03:37.313 23932 24007 D TestHQ8 : java.lang.Throwable
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.animate(WindowAnimator.java:209)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.lambda$new$1(WindowAnimator.java:99)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1558)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1569)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doCallbacks(Choreographer.java:1162)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doFrame(Choreographer.java:1087)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 16:03:37.313 23932 24007 D TestHQ8 : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

ActivityB 设置 show 的时机

```
06-23 16:03:36.727 23932 24007 D TestHQ8 : Transaction showorhide show Surface(name=ActivityRecord{d058e3b u0 com.hq.android.androiddemo/.performance.PerfActivity t846})/@0xfa989b3, TransactionID = 102787157328518
06-23 16:03:36.727 23932 24007 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.show(SurfaceControl.java:3060)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.Transition.onTransactionReady(Transition.java:1904)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.finishNow(BLASTSyncEngine.java:290)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.tryFinish(BLASTSyncEngine.java:222)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.-$$Nest$mtryFinish(Unknown Source:0)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine.onSurfacePlacement(BLASTSyncEngine.java:605)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.RootWindowContainer.performSurfacePlacementNoTrace(RootWindowContainer.java:831)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.RootWindowContainer.performSurfacePlacement(RootWindowContainer.java:776)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop(WindowSurfacePlacer.java:177)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:126)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:115)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowSurfacePlacer$Traverser.run(WindowSurfacePlacer.java:57)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 16:03:36.727 23932 24007 D TestHQ8 : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

在 mStartT 又设置了一次：    

```
06-23 16:03:36.736 25233 25405 D TestHQ8 : Transaction showorhide show Surface(name=ActivityRecord{d058e3b u0 com.hq.android.androiddemo/.performance.PerfActivity t846})/@0xa530485, TransactionID = 102787157328518
06-23 16:03:36.736 25233 25405 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.show(SurfaceControl.java:3060)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.setupStartState(Transitions.java:604)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.dispatchReady(Transitions.java:910)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onTransitionReady(Transitions.java:808)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:36.736 25233 25405 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```

在 startAnimation 时 apply startT。    

```
06-23 16:03:36.750 25233 25405 D TestHQ8 : Transaction apply TransactionID = 102787157328518, this = android.view.SurfaceControl$Transaction@b412965
06-23 16:03:36.750 25233 25405 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.startAnimation(DefaultTransitionHandler.java:606)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.dispatchTransition(Transitions.java:1074)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.playTransition(Transitions.java:1059)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.processReadyQueue(Transitions.java:976)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.dispatchReady(Transitions.java:916)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onTransitionReady(Transitions.java:808)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:36.750 25233 25405 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```

另外在 mFinishT 中也会请求 show：    

```
06-23 16:03:36.738 25233 25405 D TestHQ8 : Transaction showorhide show Surface(name=ActivityRecord{d058e3b u0 com.hq.android.androiddemo/.performance.PerfActivity t846})/@0xa530485, TransactionID = 102787157328519
06-23 16:03:36.738 25233 25405 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.show(SurfaceControl.java:3060)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.setupStartState(Transitions.java:615)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.dispatchReady(Transitions.java:910)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onTransitionReady(Transitions.java:808)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:36.738 25233 25405 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```

在Shell端动画 onFinish 时 apply。    

```
06-23 16:03:37.300 25233 25405 D TestHQ8 : Transaction apply TransactionID = 102787157328519, this = android.view.SurfaceControl$Transaction@c218b5b
06-23 16:03:37.300 25233 25405 D TestHQ8 : java.lang.Throwable
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.onFinish(Transitions.java:1209)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.lambda$playTransition$3(Transitions.java:1060)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions.$r8$lambda$96XCaSAhJXxB3nlVJy1TNPeH46k(Unknown Source:0)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.Transitions$$ExternalSyntheticLambda6.onTransitionFinished(D8$$SyntheticClass:0)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.lambda$startAnimation$1(DefaultTransitionHandler.java:329)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.$r8$lambda$2ee7viqDfcH2oT9s1JvE2VA6hu4(Unknown Source:0)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler$$ExternalSyntheticLambda6.run(D8$$SyntheticClass:0)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler.lambda$buildSurfaceAnimation$7(DefaultTransitionHandler.java:864)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at com.android.wm.shell.transition.DefaultTransitionHandler$$ExternalSyntheticLambda10.run(D8$$SyntheticClass:0)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:37.300 25233 25405 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
```


另外还有就是在 prepareSurfaces 时如果容器是显示的，那么也会请求 show ：

```
06-23 16:03:36.737 23932 24007 D TestHQ8 : Transaction showorhide show Surface(name=ActivityRecord{d058e3b u0 com.hq.android.androiddemo/.performance.PerfActivity t846})/@0xfa989b3, TransactionID = 102787157328506
06-23 16:03:36.737 23932 24007 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.show(SurfaceControl.java:3060)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.ActivityRecord.prepareSurfaces(ActivityRecord.java:8268)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.TaskFragment.prepareSurfaces(TaskFragment.java:3275)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.Task.prepareSurfaces(Task.java:3384)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowContainer.prepareSurfaces(WindowContainer.java:2906)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.DisplayContent.prepareSurfaces(DisplayContent.java:5731)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.animate(WindowAnimator.java:143)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.lambda$new$1(WindowAnimator.java:99)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.wm.WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1558)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1569)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doCallbacks(Choreographer.java:1162)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.Choreographer.doFrame(Choreographer.java:1087)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.os.Handler.handleCallback(Handler.java:959)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.os.Handler.dispatchMessage(Handler.java:100)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.os.Looper.loopOnce(Looper.java:232)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.os.Looper.loop(Looper.java:317)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at android.os.HandlerThread.run(HandlerThread.java:85)
06-23 16:03:36.737 23932 24007 D TestHQ8 : 	at com.android.server.ServiceThread.run(ServiceThread.java:46)
```

```
06-23 16:03:36.763 23932 29987 D TestHQ8 : Transaction merge other TransactionID = 102787157328506, this = 102787157328522
06-23 16:03:36.763 23932 29987 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.merge(SurfaceControl.java:4585)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.wm.WindowContainer.onSyncTransactionCommitted(WindowContainer.java:4377)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.wm.ActivityRecord.onSyncTransactionCommitted(ActivityRecord.java:3107)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup$1CommitCallback.onCommitted(BLASTSyncEngine.java:259)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.lambda$finishNow$1(BLASTSyncEngine.java:286)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup$$ExternalSyntheticLambda0.onTransactionCommitted(D8$$SyntheticClass:0)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction$$ExternalSyntheticLambda4.run(D8$$SyntheticClass:0)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at com.android.server.SystemServerInitThreadPool$$ExternalSyntheticLambda0.execute(D8$$SyntheticClass:0)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.lambda$addTransactionCommittedListener$0(SurfaceControl.java:4672)
06-23 16:03:36.763 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction$$ExternalSyntheticLambda2.onTransactionCommitted(D8$$SyntheticClass:0)
```

apply

```
06-23 16:03:36.764 23932 29987 D TestHQ8 : Transaction apply TransactionID = 102787157328522, this = android.view.SurfaceControl$Transaction@5e6e526
06-23 16:03:36.764 23932 29987 D TestHQ8 : java.lang.Throwable
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup$1CommitCallback.onCommitted(BLASTSyncEngine.java:261)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup.lambda$finishNow$1(BLASTSyncEngine.java:286)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at com.android.server.wm.BLASTSyncEngine$SyncGroup$$ExternalSyntheticLambda0.onTransactionCommitted(D8$$SyntheticClass:0)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction$$ExternalSyntheticLambda4.run(D8$$SyntheticClass:0)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at com.android.server.SystemServerInitThreadPool$$ExternalSyntheticLambda0.execute(D8$$SyntheticClass:0)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction.lambda$addTransactionCommittedListener$0(SurfaceControl.java:4672)
06-23 16:03:36.764 23932 29987 D TestHQ8 : 	at android.view.SurfaceControl$Transaction$$ExternalSyntheticLambda2.onTransactionCommitted(D8$$SyntheticClass:0)
```


## 可见性总结

结合 ActivityA 启动 ActivityB 的动画已经生命周期流程，来总结一下 Activity 可见性的变化流程。    

1. 动画开始，ActivityRecordA 完成 pause，开始resume ActivityRecordB 的时候，这个时候可以认为是动画开始不久，这里将ActivityRecordA 和ActivityRecordB 收集到Transition的SyncGroup的同时，分别设置这两个ActivityRecord的期可见性 mVisibleRequested。
2. 所有参与动画的窗口已经绘制完成（主要是ActivityRecordB 的窗口），Transition走到 onTransactionReady，设置ActivityRecordB 的 isVisble 为true。
3. 切换到 WMShell，开始播放动画，在播放动画前，onAnimationStart 阶段，调用 startT 的 apply 方法将 ActivityRecordB 的窗口真正显示出来。
4. 动画播放结束，在 Transitions.onFinish 中， 调用 finishT 的 apply 方法将 ActivityRecordA 的窗口真正隐藏掉。
5. 切换回 WMCore，在Transition.finishTransition中，调用 ActivityRecord.commitVisibility 设置 ActivityRecordA 的 isVisble 为false。

## 相关文章

[Android15 ShellTransitions】（九）结束动画+Android原生ANR问题分析 ](https://juejin.cn/post/7486664338889326646)      
