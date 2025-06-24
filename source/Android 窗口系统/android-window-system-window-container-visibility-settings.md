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
IActivityClientController$Stub.onTransact
  ActivityClientController.activityPaused
    ActivityRecord.activityPaused
      TaskFragment.completePause
        RootWindowContainer.resumeFocusedTasksTopActivities
          Task.resumeTopActivityUncheckedLocked
            Task.resumeTopActivityInnerLocked
              TaskFragment.resumeTopActivity
                ActivityTaskSupervisor.startSpecificActivity
                  ActivityTaskSupervisor.realStartActivityLocked
                    RootWindowContainer.ensureVisibilityAndConfig
                      RootWindowContainer.ensureActivitiesVisible
                        DisplayContent.ensureActivitiesVisible
                          WindowContainer.forAllRootTasks
                            Task.forAllRootTasks
                              DisplayContent$$ExternalSyntheticLambda29.accept
                                DisplayContent.lambda$ensureActivitiesVisible
                                  Task.ensureActivitiesVisible
                                    Task.forAllLeafTasks
                                      Task.lambda$ensureActivitiesVisible
                                        TaskFragment.updateActivityVisibilities
                                          EnsureActivitiesVisibleHelper.process
                                            EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                              ActivityRecord.makeVisibleIfNeeded
                                                ActivityRecord.setVisibility
                                                  ActivityRecord.setVisibleRequested
                                                    WindowContainer.setVisibleRequested(true)
```

在准备开始做动画时会通过 commitVisibility 再次设置，此时 VisibleRequested 已经为true，直接返回。 

那么这里为什么不直接把 ActivityB 的 mVisible 也设置为 true 呢？因为 ActivityB 可能还没有绘制好。如果绘制完成开始做动画时会设置 mVisible。    

### ActivityA 设置期望可见性 false

和ActivityRecordB的mVisibleRequested设置为true都是在 ActivityA pause 流程中。

```
IActivityClientController$Stub.onTransact
  ActivityClientController.activityPaused
    ActivityRecord.activityPaused
      TaskFragment.completePause
        RootWindowContainer.resumeFocusedTasksTopActivities
          Task.resumeTopActivityUncheckedLocked
            Task.resumeTopActivityInnerLocked
              TaskFragment.resumeTopActivity
                ActivityTaskSupervisor.startSpecificActivity
                  ActivityTaskSupervisor.realStartActivityLocked
                    RootWindowContainer.ensureVisibilityAndConfig
                      RootWindowContainer.ensureActivitiesVisible
                        DisplayContent.ensureActivitiesVisible
                          WindowContainer.forAllRootTasks
                            Task.forAllRootTasks
                              DisplayContent$$ExternalSyntheticLambda29.accept
                                DisplayContent.lambda$ensureActivitiesVisible
                                  Task.ensureActivitiesVisible
                                    Task.forAllLeafTasks
                                      Task.lambda$ensureActivitiesVisible
                                        TaskFragment.updateActivityVisibilities
                                          EnsureActivitiesVisibleHelper.process
                                            EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                              ActivityRecord.makeInvisible
                                                ActivityRecord.setVisibility
                                                  ActivityRecord.setVisibleRequested
                                                    WindowContainer.setVisibleRequested(false)
```

另外在转场动画结束时也会在 Transition.finishTransition 中通过 ActivityRecord.commitVisibility 再次设置，此时 VisibleRequested 以及为false，会直接 return。    

### ActivityB 设置可见性 true

当窗口绘制完成，可以做动画了，那么就可以设置容器可见了。    

```
WindowSurfacePlacer.performSurfacePlacementLoop
  RootWindowContainer.performSurfacePlacement
    RootWindowContainer.performSurfacePlacementNoTrace
      BLASTSyncEngine.onSurfacePlacement
        BLASTSyncEngine$SyncGroup.-$$Nest$mtryFinish
          BLASTSyncEngine$SyncGroup.tryFinish
            BLASTSyncEngine$SyncGroup.finishNow
              Transition.onTransactionReady
                Transition.commitVisibleActivities
                  ActivityRecord.commitVisibility
                    ActivityRecord.setVisible(true)
```

### ActivityA 设置可见性 false

当动画结束时，去设置 ActivityA 为不可见。    

```
WindowOrganizerController.onTransact
  IWindowOrganizerController$Stub.onTransact
    WindowOrganizerController.finishTransition
      TransitionController.finishTransition
        Transition.finishTransition
          ActivityRecord.commitVisibility
            ActivityRecord.setVisible(false)
```

### mVisibleAtTransitionEndTokens

Transition 中有个 mVisibleAtTransitionEndTokens 集合，用来保存在动画结束时需要可见的容器。    

## 图层可见性

前面也只是在容器层面设置了可见性，那么真正的要显示出来，还是需要SF正在把这个图层合成以及提交显示才行。而请求SF的过程我们需要跟踪一下 SurfaceControl.Transaction 的相关方法。        
以 ActivityA 启动 ActivityB 为例来介绍一下。        

ActivityA 设置 hide 的时机。    
在动画 onTransitionReady 时会设置 Transaction mFinishT，在 mFinishT hide ActivityA。     

```
Transitions.onTransitionReady(Transitions.java:808)
    Transitions.dispatchReady(Transitions.java:910)
        Transitions.setupStartState(Transitions.java:617)
            SurfaceControl$Transaction.hide(SurfaceControl.java:3080) TransactionID = 68032281971426
```

Transaction apply hide 的时机：    
在动画完成后执行 mFinishT.apply()，此时去 hide ActivityA。    

```
DefaultTransitionHandler$$ExternalSyntheticLambda10.run(D8$$SyntheticClass:0)
  DefaultTransitionHandler.lambda$buildSurfaceAnimation$7(DefaultTransitionHandler.java:864)
    DefaultTransitionHandler$$ExternalSyntheticLambda6.run(D8$$SyntheticClass:0)
      DefaultTransitionHandler.$r8$lambda$2ee7viqDfcH2oT9s1JvE2VA6hu4(Unknown Source:0)
        DefaultTransitionHandler.lambda$startAnimation$1(DefaultTransitionHandler.java:329)
          Transitions$$ExternalSyntheticLambda6.onTransitionFinished(D8$$SyntheticClass:0)
            Transitions.$r8$lambda$96XCaSAhJXxB3nlVJy1TNPeH46k(Unknown Source:0)
              Transitions.lambda$playTransition$3(Transitions.java:1060)
                Transitions.onFinish(Transitions.java:1209)
                  SurfaceControl$Transaction.apply(SurfaceControl.java:2929) TransactionID = 68032281971426
```

另外在 prepareSurfaces() 方法中，如果检测到了 mVisible false，也会结合其他条件判断来通过 getSyncTransaction()设置发起 hide 调用。     

```
Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
  Choreographer.doFrame(Choreographer.java:1087)
    Choreographer.doCallbacks(Choreographer.java:1162)
      Choreographer$CallbackRecord.run(Choreographer.java:1569)
        WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
          WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
            WindowAnimator.lambda$new$1(WindowAnimator.java:99)
              WindowAnimator.animate(WindowAnimator.java:143)
                DisplayContent.prepareSurfaces(DisplayContent.java:5731)
                  DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
                    WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                      DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
                        WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                          Task.prepareSurfaces(Task.java:3384)
                            TaskFragment.prepareSurfaces(TaskFragment.java:3275)
                              WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                                ActivityRecord.prepareSurfaces(ActivityRecord.java:8270)
                                  SurfaceControl$Transaction.hide(SurfaceControl.java:3080) TransactionID = 102787157328515
```

这里会 merge Transaction。    

```
Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
  Choreographer.doFrame(Choreographer.java:1087)
    Choreographer.doCallbacks(Choreographer.java:1162)
      Choreographer$CallbackRecord.run(Choreographer.java:1569)
        Choreographer$CallbackRecord.run(Choreographer.java:1558)
          WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
            WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
              WindowAnimator.lambda$new$1(WindowAnimator.java:99)
                WindowAnimator.animate(WindowAnimator.java:168)
                  SurfaceControl$Transaction.merge(SurfaceControl.java:4585) merge other TransactionID = 102787157328515, this = 102787157328524
```


apply 时机    

```
Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1537)
  Choreographer.doFrame(Choreographer.java:1087)
    Choreographer.doCallbacks(Choreographer.java:1162)
      Choreographer$CallbackRecord.run(Choreographer.java:1569)
        Choreographer$CallbackRecord.run(Choreographer.java:1558)
          WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
            WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
              WindowAnimator.lambda$new$1(WindowAnimator.java:99)
                WindowAnimator.animate(WindowAnimator.java:209)
                  SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
                    SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
                      SurfaceControl$Transaction.apply(SurfaceControl.java:2977) TransactionID = 102787157328524
```


ActivityB 设置 show 的时机    

```
WindowSurfacePlacer$Traverser.run(WindowSurfacePlacer.java:57)
  WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:115)
    WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:126)
      WindowSurfacePlacer.performSurfacePlacementLoop(WindowSurfacePlacer.java:177)
        RootWindowContainer.performSurfacePlacement(RootWindowContainer.java:776)
          RootWindowContainer.performSurfacePlacementNoTrace(RootWindowContainer.java:831)
            BLASTSyncEngine.onSurfacePlacement(BLASTSyncEngine.java:605)
              BLASTSyncEngine$SyncGroup.-$$Nest$mtryFinish(Unknown Source:0)
                BLASTSyncEngine$SyncGroup.tryFinish(BLASTSyncEngine.java:222)
                  BLASTSyncEngine$SyncGroup.finishNow(BLASTSyncEngine.java:290)
                    Transition.onTransactionReady(Transition.java:1904)
                      SurfaceControl$Transaction.show(SurfaceControl.java:3060) TransactionID = 102787157328518
                        
```

在 mStartT 又设置了一次：    

```
Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
  Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
    Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
      Transitions.onTransitionReady(Transitions.java:808)
        Transitions.dispatchReady(Transitions.java:910)
          Transitions.setupStartState(Transitions.java:604)
            SurfaceControl$Transaction.show(SurfaceControl.java:3060)  TransactionID = 102787157328518
```

在 startAnimation 时 apply startT。    

```
Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
  Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
    Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
      Transitions.onTransitionReady(Transitions.java:808)
        Transitions.dispatchReady(Transitions.java:916)
          Transitions.processReadyQueue(Transitions.java:976)
            Transitions.playTransition(Transitions.java:1059)
              Transitions.dispatchTransition(Transitions.java:1074)
                DefaultTransitionHandler.startAnimation(DefaultTransitionHandler.java:606)
                  SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
                    SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
                      SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
                        TransactionID = 102787157328518
```

另外在 mFinishT 中也会请求 show：    

```
Transitions$TransitionPlayerImpl$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
  Transitions$TransitionPlayerImpl.$r8$lambda$E1Ms2CLuO14YRHZa6uUNycRrU-A(Unknown Source:0)
    Transitions$TransitionPlayerImpl.lambda$onTransitionReady$0(Transitions.java:1608)
      Transitions.onTransitionReady(Transitions.java:808)
        Transitions.dispatchReady(Transitions.java:910)
          Transitions.setupStartState(Transitions.java:615)
            SurfaceControl$Transaction.show(SurfaceControl.java:3060)  TransactionID = 102787157328519
```

在Shell端动画 onFinish 时 apply。    

```
DefaultTransitionHandler$$ExternalSyntheticLambda10.run(D8$$SyntheticClass:0)
  DefaultTransitionHandler.lambda$buildSurfaceAnimation$7(DefaultTransitionHandler.java:864)
    DefaultTransitionHandler$$ExternalSyntheticLambda6.run(D8$$SyntheticClass:0)
      DefaultTransitionHandler.$r8$lambda$2ee7viqDfcH2oT9s1JvE2VA6hu4(Unknown Source:0)
        DefaultTransitionHandler.lambda$startAnimation$1(DefaultTransitionHandler.java:329)
          Transitions$$ExternalSyntheticLambda6.onTransitionFinished(D8$$SyntheticClass:0)
            Transitions.$r8$lambda$96XCaSAhJXxB3nlVJy1TNPeH46k(Unknown Source:0)
              Transitions.lambda$playTransition$3(Transitions.java:1060)
                Transitions.onFinish(Transitions.java:1209)
                  SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
                    SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
                      SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
                        TransactionID = 102787157328519
```


另外还有就是在 prepareSurfaces 时如果容器是显示的，那么也会请求 show ：

```
Choreographer$CallbackRecord.run(Choreographer.java:1558)
  WindowAnimator$$ExternalSyntheticLambda1.doFrame(D8$$SyntheticClass:0)
    WindowAnimator.$r8$lambda$AS_wbK9i-bc6ocCFop7s9PnXP80(Unknown Source:0)
      WindowAnimator.lambda$new$1(WindowAnimator.java:99)
        WindowAnimator.animate(WindowAnimator.java:143)
          DisplayContent.prepareSurfaces(DisplayContent.java:5731)
            DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
              WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                DisplayArea$Dimmable.prepareSurfaces(DisplayArea.java:834)
                  WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                    Task.prepareSurfaces(Task.java:3384)
                      TaskFragment.prepareSurfaces(TaskFragment.java:3275)
                        WindowContainer.prepareSurfaces(WindowContainer.java:2906)
                          ActivityRecord.prepareSurfaces(ActivityRecord.java:8268)
                            SurfaceControl$Transaction.show(SurfaceControl.java:3060)
                              TransactionID = 102787157328506
```

```
SurfaceControl$Transaction$$ExternalSyntheticLambda2.onTransactionCommitted(D8$$SyntheticClass:0)
  SurfaceControl$Transaction.lambda$addTransactionCommittedListener$0(SurfaceControl.java:4672)
    SystemServerInitThreadPool$$ExternalSyntheticLambda0.execute(D8$$SyntheticClass:0)
      SurfaceControl$Transaction$$ExternalSyntheticLambda4.run(D8$$SyntheticClass:0)
        BLASTSyncEngine$SyncGroup$$ExternalSyntheticLambda0.onTransactionCommitted(D8$$SyntheticClass:0)
          BLASTSyncEngine$SyncGroup.lambda$finishNow$1(BLASTSyncEngine.java:286)
            BLASTSyncEngine$SyncGroup$1CommitCallback.onCommitted(BLASTSyncEngine.java:259)
              ActivityRecord.onSyncTransactionCommitted(ActivityRecord.java:3107)
                WindowContainer.onSyncTransactionCommitted(WindowContainer.java:4377)
                  SurfaceControl$Transaction.merge(SurfaceControl.java:4585)  
                    merge other TransactionID = 102787157328506, this = 102787157328522
```


apply

```
SurfaceControl$Transaction$$ExternalSyntheticLambda2.onTransactionCommitted(D8$$SyntheticClass:0)
  SurfaceControl$Transaction.lambda$addTransactionCommittedListener$0(SurfaceControl.java:4672)
    SystemServerInitThreadPool$$ExternalSyntheticLambda0.execute(D8$$SyntheticClass:0)
      SurfaceControl$Transaction$$ExternalSyntheticLambda4.run(D8$$SyntheticClass:0)
        BLASTSyncEngine$SyncGroup$$ExternalSyntheticLambda0.onTransactionCommitted(D8$$SyntheticClass:0)
          BLASTSyncEngine$SyncGroup.lambda$finishNow$1(BLASTSyncEngine.java:286)
            BLASTSyncEngine$SyncGroup$1CommitCallback.onCommitted(BLASTSyncEngine.java:261)
              SurfaceControl$Transaction.apply(SurfaceControl.java:2929)
                SurfaceControl$Transaction.apply(SurfaceControl.java:2973)
                  SurfaceControl$Transaction.apply(SurfaceControl.java:2977)
                    apply TransactionID = 102787157328522
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
