---
title: Android PIP模式
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android PIP模式
date: 2022-11-23 10:00:00
---


## 概述

在 WMShell 侧通过 WindowContainerTransaction 来修改 Task 的大小和显示区域，然后传递到 WMCore 侧变更窗口容器的属性来实现。    
通过下图可以看到，PIP的实现是通过修改窗口容器的显示范围来实现的。       

<img src="/images/android-window-system-pip/1.png" width="578" height="722"/>
<img src="/images/android-window-system-pip/2.png" width="604" height="735"/> 

另外PIP所在的 Task 所在的图层是可见状态，是被绘制的，而普通 Activity 所在的 Task 容器在 sf 是不会被绘制的。原因是PIP模式时，为Task设置了阴影。       

本文基于 Android 15。     

执行 `adb shell settings put global enable_freeform_support 1` 可以使支持 PIP 的应用在多任务的菜单里面显示自由窗口的选项，如果没有这个选项，可以执行 `adb shell settings put global force_resizable_activities 1` 让所有的 Activity 强制支持自由窗口模式。      

## 相关类

PipSurfaceTransactionHelper：处理图层变换的类    
PipAnimationController:     
PipTransitionController，PipTransition：执行 PIP 的 Transition      

## App 配置支持 PIP

```
        <activity
            android:name=".wms.WMSTestActivity"
            android:exported="false"
            android:label="WMSTestActivity"
            android:supportsPictureInPicture="true"
```

```
    fun enterPipMode(view:View){
        if(isCanPipModel()){
            enterPipModel()
        }
    }
    private fun isCanPipModel(): Boolean {
        return packageManager.hasSystemFeature(PackageManager.FEATURE_PICTURE_IN_PICTURE)
    }

    private fun enterPipModel() {
        val builder = PictureInPictureParams.Builder()

        //设置宽高比例，第一个是分子，第二个是分母，执行宽高比，必须在2.39：1或1：2.39之间
        builder.setAspectRatio(Rational(3, 7))
        enterPictureInPictureMode(builder.build())
    }
```

## 进入PIP

system_server       

```
ActivityClientController.enterPictureInPictureMode
  ActivityTaskManagerService.enterPictureInPictureMode
    new Transition(TRANSIT_PIP) // 创建一个 TRANSIT_PIP 类型的 Transition
    TransitionController.startCollectOrQueue()
      TransitionController.moveToCollecting()
        onStartCollect.onCollectStarted()
          ActivityTaskManagerService.enterPictureInPictureMode.enterPipRunnable.run()
            RootWindowContainer.moveActivityToPinnedRootTask(ActivityRecord r)
              task = ActivityRecord.getTask() // 获取 Activity 所在的 Task
              TransitionController.collect(task) // 搜集 Task
              // 这里要设置一下 WINDOWING_MODE_FULLSCREEN，要不它会跟随 父容器
              // 等动画做完会设置为 WINDOWING_MODE_PINNED
              ActivityRecord.setWindowingMode(WINDOWING_MODE_FULLSCREEN)
              //为 Task 设置 WINDOWING_MODE_PINNED  
              Task.setWindowingMode(WINDOWING_MODE_PINNED)
              task.getNonFinishingActivityCount()
              // 判断是否需要新建 Task
              rootTask = task 或者 new Task()
              RootWindowContainer.ensureActivitiesVisible
                DisplayContent.ensureActivitiesVisible
                  WindowContainer.forAllRootTasks
                    Task.forAllRootTasks
                      Task.ensureActivitiesVisible
                        Task.forAllLeafTasks
                          TaskFragment.updateActivityVisibilities
                            EnsureActivitiesVisibleHelper.process
                              EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                ActivityRecord.makeActiveIfNeeded
                                  ActivityRecord.shouldResumeActivity()
                                    ActivityRecord.shouldResumeActivity
                                      // 重要的方法，判断 Activity 可见性，是pause还是resume
                                      TaskFragment.getVisibility()
                                  // resume pip activity 下面的Activity
                                  Task.resumeTopActivityUncheckedLocked()
                                    Task.resumeTopActivityInnerLocked
                                      TaskFragment.resumeTopActivity
                                        ActivityRecord.setState
                                          TaskFragment.onActivityStateChanged
                                            TaskFragment.setResumedActivity
                                              ActivityTaskSupervisor.updateTopResumedActivityIfNeeded
                                                ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                        ResumeActivityItem.obtain()
                                        getLifecycleManager().scheduleTransactionItem(ResumeActivityItem)
                                  // else
                                  ActivityRecord.shouldPauseActivity
                                  // PIP Activity pause
                                  PauseActivityItem.obtain
                                  getLifecycleManager().scheduleTransactionItem(PauseActivityItem)
              // 开始请求动画
              TransitionController.requestStartTransition(Transition)
```

当 pip 的 Activity和其他 Activity 在同一个 task 中时，进入pip会新建一个 task 来承载这个 Activity，如果 pip Activity 在单独的一个 task 里面，则不会新建 task。      

SystemUI      

```
Transitions.requestStartTransition
  PipTransition.handleRequest()
    new WindowContainerTransaction()
```

```
Transitions.onTransitionReady
  Transitions.dispatchReady
    Transitions.processReadyQueue
      Transitions.playTransition
        PipTransition.startAnimation
          PipTransition.startEnterAnimation
            // 获取 pip 窗口的位置
            destinationBounds = PipBoundsAlgorithm.getEntryDestinationBounds()
            // 获取当前窗口位置
            currentBounds = TransitionInfo.Change.getStartAbsBounds()
            PipTransitionAnimator animator = PipAnimationController.getAnimator(currentBounds, destinationBounds)
              PipTransitionAnimator.ofBounds(currentBounds, destinationBounds)
                new PipTransitionAnimator<Rect>(startValue,endValue)
                  mDestinationBounds = destinationBounds
                  PipTransitionAnimator.addUpdateListener(this)
                    // 动画结束
                    PipTransitionAnimator.onAnimationEnd
                      PipAnimationController.onEndTransaction
                        // 对 task 裁剪
                        getSurfaceTransactionHelper().crop()
                          SurfaceControl.Transaction.setWindowCrop().setPosition()
                      mPipAnimationCallback.onPipAnimationEnd()
                        PipTransition.onFinishResize()
                          new WindowContainerTransaction()
                          WindowContainerTransaction.setActivityWindowingMode()
                            WindowContainerTransaction.getOrCreateChange()
                              new Change()
                            Change.mActivityWindowingMode = windowingMode // 设置 WindowingMode 变更
                          WindowContainerTransaction.setBounds(destinationBounds) // 设置task Bounds 
                            WindowContainerTransaction.getOrCreateChange
                            chg.mConfiguration.windowConfiguration.setBounds(bounds);
                            chg.mConfigSetMask |= ActivityInfo.CONFIG_WINDOW_CONFIGURATION;
                            chg.mWindowSetMask |= WindowConfiguration.WINDOW_CONFIG_BOUNDS;
                          SurfaceTransactionHelper.crop().resetScale().round() // 对 task 裁剪
                          WindowContainerTransaction.setBoundsChangeTransaction
                            WindowContainerTransaction.getOrCreateChange
                            chg.mBoundsChangeTransaction = t;
                            chg.mChangeMask |= Change.CHANGE_BOUNDS_TRANSACTION; // 设置 Bounds 变更
                          PipTransition.callFinishCallback(wct)
                            // 传递的 wct 不为空，system_server apply change
                            mFinishCallback.onTransitionFinished(wct)
                              Transitions.onFinish
                                WindowOrganizer.finishTransition()
                                  getWindowOrganizerController().finishTransition()
                                    // ------> WMCore
              PipAnimationController.setupPipTransitionAnimator // 配置动画属性
              PipTransitionAnimator.setPipAnimationCallback(mPipAnimationCallback)
                PipTransitionAnimator.applySurfaceControlTransaction()
                  PipSurfaceTransactionHelper.scaleAndCrop()
                  PipSurfaceTransactionHelper.shadow()
                    Transaction.setShadowRadius() // 设置阴影，因此 Task 也是需要实际绘制的
              PipTransitionAnimator.start() // 开始动画
```

system_server:

设置Task区域：       


```
WindowOrganizerController.finishTransition()
  WindowOrganizerController.applyTransaction
    WindowOrganizerController.applyWindowContainerChange
      WindowOrganizerController.applyTaskChanges
        WindowOrganizerController.applyChanges
          c=new Configuration(container.getRequestedOverrideConfiguration())
          Configuration.setTo(change.getConfiguration())
            WindowConfiguration.setTo()
              WindowConfiguration.setBounds
          WindowContainer(Task).onRequestedOverrideConfigurationChanged(WindowConfiguration c)
            ConfigurationContainer.onRequestedOverrideConfigurationChanged
              ConfigurationContainer.updateRequestedOverrideConfiguration
                //把 shell 传递来的配置给 mRequestedOverrideConfiguration
                mRequestedOverrideConfiguration.setTo(overrideConfiguration)
              Task.onConfigurationChanged(getParent().getConfiguration())
                Task.onConfigurationChangedInner
                  TaskFragment.onConfigurationChanged
                    WindowContainer.onConfigurationChanged
                      ConfigurationContainer.onConfigurationChanged
                        TaskFragment.resolveOverrideConfiguration
                          ConfigurationContainer.resolveOverrideConfiguration
                            // 把 mRequestedOverrideConfiguration 设置给 mResolvedOverrideConfiguration
                            mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration)
                          Task.resolveLeafTaskOnlyOverrideConfigs
                        // 把 mResolvedOverrideConfiguration 设置给 mFullConfiguration
                        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
                        ConfigurationContainer.onMergedOverrideConfigurationChanged()
                          // 把 mResolvedOverrideConfiguration 设置给 mMergedOverrideConfiguration
                          mMergedOverrideConfiguration.updateFrom(mResolvedOverrideConfiguration)
                        // 分发给子容器
                        for (int i = getChildCount() - 1; i >= 0; --i) {
                        dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
                        }
```


## 工具栏

工具栏是在 Task 图层下面的一个图层。基于 SurfaceControlViewHost 将工具栏图层直接挂载在 Task 图层上。      

<img src="/images/android-window-system-pip/pip-menu.png" width="505" height="283"/>

### 创建工具栏

```
Transitions.onTransitionReady
  Transitions.dispatchReady
    FreeformTaskTransitionObserver.onTransitionReady
      FreeformTaskTransitionObserver.onToFrontTransitionReady
        CaptionWindowDecorViewModel.onTaskChanging
          CaptionWindowDecorViewModel.createWindowDecoration
            CaptionWindowDecoration.relayout
              WindowDecoration.relayout
                WindowDecoration.updateDecorationContainerSurface
                  // 创建 Decor container of Task=，并挂载到 Task 的SurfaceControl上面
                  SurfaceControl.Builder.setParent(mTaskSurface).build()
```

### 最大化


```
View.performClick
  CaptionWindowDecorViewModel$CaptionTouchEventListener.onClick
    TaskOperations.maximizeTask
      FreeformTaskTransitionHandler.startWindowingModeTransition
        Transitions.startTransition
          WindowOrganizer.startNewTransition
```

### 移除工具栏

```
ShellTaskOrganizer.onTaskInfoChanged
  ShellTaskOrganizer.updateTaskListenerIfNeeded
    FreeformTaskListener.onTaskVanished
      CaptionWindowDecorViewModel.destroyWindowDecoration
        CaptionWindowDecoration.close
          CaptionWindowDecoration.closeDragResizeListener
            DragResizeInputListener.close
              SurfaceControl$Transaction.remove
```

## 移动


### 注册事件监听：

https://juejin.cn/post/7265939798795026466

```
// PipInputConsumer.java

    public void registerInputConsumer() {
        if (mInputEventReceiver != null) {
            return;
        }
        final InputChannel inputChannel = new InputChannel();
        try {
            // TODO(b/113087003): Support Picture-in-picture in multi-display.
            mWindowManager.destroyInputConsumer(mToken, DEFAULT_DISPLAY);
            mWindowManager.createInputConsumer(mToken, mName, DEFAULT_DISPLAY, inputChannel);
        } catch (RemoteException e) {
            ProtoLog.e(ShellProtoLogGroup.WM_SHELL_PICTURE_IN_PICTURE,
                    "%s: Failed to create input consumer, %s", TAG, e);
        }
        mMainExecutor.execute(() -> {
            mInputEventReceiver = new InputEventReceiver(inputChannel,
                Looper.myLooper(), Choreographer.getInstance());
            if (mRegistrationListener != null) {
                mRegistrationListener.onRegistrationChanged(true /* isRegistered */);
            }
        });
    }
```

WMS 创建 InputConsumer      

```
    public void createInputConsumer(IBinder token, String name, int displayId,
            InputChannel inputChannel) {
        if (!mAtmService.isCallerRecents(Binder.getCallingUid())
                && mContext.checkCallingOrSelfPermission(INPUT_CONSUMER) != PERMISSION_GRANTED) {
            throw new SecurityException("createInputConsumer requires INPUT_CONSUMER permission");
        }

        synchronized (mGlobalLock) {
            DisplayContent display = mRoot.getDisplayContent(displayId);
            if (display != null) {
                display.getInputMonitor().createInputConsumer(token, name, inputChannel,
                        Binder.getCallingPid(), Binder.getCallingUserHandle());
            }
        }
    }
```

创建 Surface(name=Input Consumer pip_input_consumer#325) 图层：     

```
    InputConsumerImpl(WindowManagerService service, IBinder token, String name,
            InputChannel inputChannel, int clientPid, UserHandle clientUser, int displayId,
            SurfaceControl.Transaction t) {
        ......

        mInputSurface = mService.makeSurfaceBuilder()
                .setContainerLayer()
                .setName("Input Consumer " + name)
                .setCallsite("InputConsumerImpl")
                .build();
        mWindowHandle.setTrustedOverlay(t, mInputSurface, true);
    }
```

初始设置 mInputSurface 的大小和位置：      

```
    void layout(SurfaceControl.Transaction t, Rect r) {
        mTmpClipRect.set(0, 0, r.width(), r.height());

        if (mOldPosition.equals(r.left, r.top) && mOldWindowCrop.equals(mTmpClipRect)) {
            return;
        }

        t.setPosition(mInputSurface, r.left, r.top);
        t.setWindowCrop(mInputSurface, mTmpClipRect);

        mOldPosition.set(r.left, r.top);
        mOldWindowCrop.set(mTmpClipRect);
    }
```

更新 Surface(name=Input Consumer pip_input_consumer#325) 图层的位置以及将其挂载到 DefaultTaskDisplayArea 上。      

```
InputMonitor$UpdateInputForAllWindowsConsumer.updateInputWindows:615
  WindowContainer.forAllWindows
    WindowState.forAllWindows:4471
      WindowState.applyInOrderWithImeWindows:4597
        WindowContainer$ForAllWindowsConsumerWrapper.apply
          InputMonitor$UpdateInputForAllWindowsConsumer.accept
            InputConsumerImpl.layout
              Transaction.setPosition(mInputSurface, r.left, r.top)
              Transaction.setWindowCrop(mInputSurface, mTmpClipRect)
            PipInputConsumer.reparent(mInputTransaction, targetDA)
              Transaction.reparent
            PipInputConsumer.show()
              Transaction.show
              Transaction.setInputWindowInfo // 设置窗口的触摸区域
              Transaction.setLayer
              
```

可以看到 PIP 的触摸事件其实是自己使用独立的图层来进行的事件接受，和 pip 这个activity的窗口inputchannel没有啥关系。       


<img src="/images/android-window-system-pip/pip-surface.png" width="411" height="342"/>


### 拖动流程

在拖动过程中实时直接修改图层的位置，此时窗口的位置并不变化，ACTION_UP 事件时才最终修改窗口的位置。       

SystemUI

```
PipTouchHandler.handleTouchEvent()
    MotionEvent.ACTION_MOVE
    PipTouchHandler.DefaultPipTouchGesture.onMove()
        PipTouchHandler.movePip()
            PipTaskOrganizer.scheduleUserResizePip()
                PipSurfaceTransactionHelper.scale().round()
                    // 根据移动位置直接修改图层
                    SurfaceControl.Transaction.setMatrix()
                        SurfaceControl.Transaction.setMatrix()
                        SurfaceControl.Transaction.setPosition()
    MotionEvent.ACTION_UP
    PipTouchHandler.DefaultPipTouchGesture.onUp() // 松手后PIP窗口贴边动画
        PipMotionHelper.flingToSnapTarget()
            PipMotionHelper.movetoTarget()
                PipMotionHelper.startBoundsAnimator()
                    PhysicsAnimator.addUpdateListener(mResizePipUpdateListener)
                        PipTaskOrganizer.scheduleUserResizePip() //mResizePipUpdateListener回调中更新位置
                    PhysicsAnimator.withEndActions(this::onBoundsPhysicsAnimationEnd) // 完成动画回调
                        PipTaskOrganizer.scheduleFinishResizePip()
                            PipTaskOrganizer.createFinishResizeSurfaceTransaction()
                            PipTaskOrganizer.finishResize()
                                PipTaskOrganizer.applyFinishBoundsResize()
                                    PipTaskOrganizer.applyTransaction() --> system_server
                    PhysicsAnimator.start()
                        
                        
```

system_server
```
WindowOrganizerController.applyTransaction()
```

## 缩放


缩放是通过实时修改窗口的大小来实现的。       

SystemUI：

```
DragResizeInputListener$TaskResizeInputEventReceiver.handleMotionEvent
  case MotionEvent.ACTION_MOVE:
    FluidResizeTaskPositioner.onDragPositioningMove
      // 修改窗口 Bounds 大小
      WindowOrganizer.applyTransaction
  case MotionEvent.ACTION_UP:
    FluidResizeTaskPositioner.onDragPositioningEnd
      // 直接发起动画
      Transitions.startTransition
        WindowOrganizer.startNewTransition(TRANSIT_CHANGE)
        new ActiveTransition
        active.mHandler = FluidResizeTaskPositioner
```

动画的 Handler 是 FluidResizeTaskPositioner。     

system_server    

```
WindowOrganizerController.applyTransaction
  WindowOrganizerController.applyWindowContainerChange
    WindowOrganizerController.applyTaskChanges
      WindowOrganizerController.applyChanges
        WindowContainer.onRequestedOverrideConfigurationChanged
          ConfigurationContainer.onRequestedOverrideConfigurationChanged
            Task.onConfigurationChanged
              Task.onConfigurationChangedInner
                TaskFragment.onConfigurationChanged
                  WindowContainer.onConfigurationChanged
```


## 退出

退出动画由 Transitions.startTransition() 直接发起。     
这里有一个动画实现的细节：在PIP退出动画时，是一个全屏后的界面从小放大的过程，这就要求在动画前，全屏界面是已经绘制好一帧的，然后在开始进行放大动画，这个实现方式有两种。      
第一个就是通过 `WindowOrganizer.startNewTransition` 发起 ShellTransition 动画时，把设置全屏的 WindowContainerTransaction 作为参数传入，那么就会在动画前执行 wct，提前绘制全屏。这种是在开启ShellTransition 动画时使用。      
另外一种就是通过 SyncTransactionQueue，这个在前面文章中已经介绍了，SyncTransactionQueue.queue() 发送设置全屏的 wct，然后WMCore绘制好后，会执行通过 SyncTransactionQueue.runInSync 设置的回调函数，在回调函数中再开始动画。这种是在以前的动画时使用。      

SystemUI
```
PipMenuView.expandPip()
    PipMenuView.hideMenu()
        MenuContainerAnimator.start()
            onAnimationEnd()
                PhonePipMenuController.onPipExpand()
                    PipTouchHandler.onPipExpand()
                        PipMotionHelper.expandLeavePip()
                            PhonePipMenuController.hideMenu()
                                PipMenuView.hideMenu()
                            PipTaskOrganizer.exitPip()
                                wct = new WindowContainerTransaction()
                                wct.setActivityWindowingMode()
                                wct.setBounds()
                                if (Transitions.ENABLE_SHELL_TRANSITIONS)
                                PipTransition.startExitTransition()
                                    // WMShell 直接发起 ShellTransition 动画
                                    Transitions.startTransition()
                                        WindowOrganizer.startNewTransition(wct)
                                        active = new ActiveTransition()
                                        // 直接设置动画 Handler，
                                        // WMShell 发起的动画省去了requestStartTransition
                                        active.mHandler = handler
                                else
                                SyncTransactionQueue.queue(wct)
                                SyncTransactionQueue.runInSync()
                                    PipTaskOrganizer.animateResizePip()
```

## PIP 模式对Activity生命周期和可见性的修改

一般我们启动一个全屏的 Activity 时，它下面的 Activity 就要进入 pause 状态，而且会被设置为不可见。但是当Activity 进入 PIP 模式后，它后面的 Activity 却是可见的，处于 Resume 状态的。     
在这种模式下，系统是如何处理Activity的暂停逻辑的呢？接下来主要介绍这个知识点。     

在前面流程介绍时也提到过，`TaskFragment.getVisibility()` 是决定 Activity 可见性的重要的逻辑，根据它返回的状态，比如：TASK_FRAGMENT_VISIBILITY_INVISIBLE、TASK_FRAGMENT_VISIBILITY_VISIBLE、TASK_FRAGMENT_VISIBILITY_VISIBLE_BEHIND_TRANSLUCENT 来决定 Activity 应该进入的生命周期。        



```
    @TaskFragmentVisibility
    int getVisibility(ActivityRecord starting) {

        if (!isAttached() || isForceHidden()) {
            return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
        }

        if (isTopActivityLaunchedBehind()) {
            return TASK_FRAGMENT_VISIBILITY_VISIBLE;
        }
        final WindowContainer<?> parent = getParent();
        final Task thisTask = asTask();
        if (thisTask != null && parent.asTask() == null
                && mTransitionController.isTransientVisible(thisTask)) {
            // Keep transient-hide root tasks visible. Non-root tasks still follow standard rule.
            return TASK_FRAGMENT_VISIBILITY_VISIBLE;
        }

        boolean gotTranslucentFullscreen = false;
        boolean gotTranslucentAdjacent = false;
        boolean shouldBeVisible = true;

        // This TaskFragment is only considered visible if all its parent TaskFragments are
        // considered visible, so check the visibility of all ancestor TaskFragment first.
        if (parent.asTaskFragment() != null) {
            final int parentVisibility = parent.asTaskFragment().getVisibility(starting);
            if (parentVisibility == TASK_FRAGMENT_VISIBILITY_INVISIBLE) {
                // Can't be visible if parent isn't visible
                return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
            } else if (parentVisibility == TASK_FRAGMENT_VISIBILITY_VISIBLE_BEHIND_TRANSLUCENT) {
                // Parent is behind a translucent container so the highest visibility this container
                // can get is that.
                gotTranslucentFullscreen = true;
            }
        }

        final List<TaskFragment> adjacentTaskFragments = new ArrayList<>();
        // 遍历父容器的所有子容器，由于 mChildren 是按 z-order 顺序排列，最顶部的容器在数组的尾部
        // 因此这里是从容器的顶端到尾端顺序进行遍历
        for (int i = parent.getChildCount() - 1; i >= 0; --i) {
            final WindowContainer other = parent.getChildAt(i);
            if (other == null) continue;

            final boolean hasRunningActivities = hasRunningActivity(other);
            if (other == this) {
                // 如果遍历到自己了，遍历任务就算完成了，因为可见性的设置只需要关系自己上面的容器是否把该容器遮挡
                if (!adjacentTaskFragments.isEmpty() && !gotTranslucentAdjacent) {
                    // The z-order of this TaskFragment is in middle of two adjacent TaskFragments
                    // and it cannot be visible if the TaskFragment on top is not translucent and
                    // is occluding this one.
                    mTmpRect.set(getBounds());
                    for (int j = adjacentTaskFragments.size() - 1; j >= 0; --j) {
                        final TaskFragment taskFragment = adjacentTaskFragments.get(j);
                        final TaskFragment adjacentTaskFragment =
                                taskFragment.mAdjacentTaskFragment;
                        if (adjacentTaskFragment == this) {
                            continue;
                        }
                        if (mTmpRect.intersect(taskFragment.getBounds())
                                || mTmpRect.intersect(adjacentTaskFragment.getBounds())) {
                            return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
                        }
                    }
                }
                // Should be visible if there is no other fragment occluding it, unless it doesn't
                // have any running activities, not starting one and not home stack.
                shouldBeVisible = hasRunningActivities
                        || (starting != null && starting.isDescendantOf(this))
                        || (isActivityTypeHome() && !isEmbedded());
                break;
            }

            if (!hasRunningActivities) {
                continue;
            }

            final int otherWindowingMode = other.getWindowingMode();
            // 如果遍历到有全屏的容器，或者容器的window mode不是PINNED，且没有override父布局边界（和父布局边界一样）
            // 那么就返回INVISIBLE，因为它会把当前的容器遮挡
            if (otherWindowingMode == WINDOWING_MODE_FULLSCREEN
                    || (otherWindowingMode != WINDOWING_MODE_PINNED && other.matchParentBounds())) {
                if (isTranslucent(other, starting)) {
                    // Can be visible behind a translucent fullscreen TaskFragment.
                    gotTranslucentFullscreen = true;
                    continue;
                }
                return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
            }

            final TaskFragment otherTaskFrag = other.asTaskFragment();
            if (otherTaskFrag != null && otherTaskFrag.mAdjacentTaskFragment != null) {
                if (adjacentTaskFragments.contains(otherTaskFrag.mAdjacentTaskFragment)) {
                    if (otherTaskFrag.isTranslucent(starting)
                            || otherTaskFrag.mAdjacentTaskFragment.isTranslucent(starting)) {
                        // Can be visible behind a translucent adjacent TaskFragments.
                        gotTranslucentFullscreen = true;
                        gotTranslucentAdjacent = true;
                        continue;
                    }
                    // Can not be visible behind adjacent TaskFragments.
                    return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
                } else {
                    adjacentTaskFragments.add(otherTaskFrag);
                }
            }

        }

        if (!shouldBeVisible) {
            return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
        }

        // Lastly - check if there is a translucent fullscreen TaskFragment on top.
        return gotTranslucentFullscreen
                ? TASK_FRAGMENT_VISIBILITY_VISIBLE_BEHIND_TRANSLUCENT
                : TASK_FRAGMENT_VISIBILITY_VISIBLE;
    }
```

## 焦点处理

设置 WINDOWING_MODE_PINNED 类型的窗口不能获取焦点。        

```
// WindowConfiguration.java
    public boolean canReceiveKeys() {
        return mWindowingMode != WINDOWING_MODE_PINNED;
    }
```

## 实战作业

启动某个 Activity 时，以自定义的某个尺寸打开。        
在 DefaultTransitionHandler.startAnimation 的 onAnimFinish 回调中添加下面的代码，在 OPEN 动画完成后，修改 Task 的 Bounds 属性，设置自动以的大小。      

```
    @Override
    public boolean startAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
            @NonNull SurfaceControl.Transaction startTransaction,
            @NonNull SurfaceControl.Transaction finishTransaction,
            @NonNull Transitions.TransitionFinishCallback finishCallback) {
        ...
        final Runnable onAnimFinish = () -> {
            if (!animations.isEmpty()) return;
            mAnimations.remove(transition);
            // ----->
            WindowContainerTransaction wct = new WindowContainerTransaction();
            for (int i = info.getChanges().size() - 1; i >= 0; --i) {
                TransitionInfo.Change change = info.getChanges().get(i);
                if (change.getMode() == TRANSIT_OPEN && change.getTaskInfo() != null
                        && change.getTaskInfo().topActivity.getClassName().contains("com.hq.android.androiddemo.common")) {
                    wct.setBounds(change.getTaskInfo().token, new Rect(100, 200, 600, 1600));
                }
            }
            finishCallback.onTransitionFinished(wct );
            // <-------
            //finishCallback.onTransitionFinished(null /* wct */);
        };
```

发现打开 Activity 还是以全屏的方式，通过 Debug 发现，Task 的 mResolvedOverrideConfiguration 的 Bounds 在设置为自定义大小后，又被重置为 (0,0,0,0)，具体在下面的逻辑中：      

```
// Task.java
    void resolveLeafTaskOnlyOverrideConfigs(Configuration newParentConfig, Rect previousBounds) {
        ...
        Rect outOverrideBounds = getResolvedOverrideConfiguration().windowConfiguration.getBounds();

        if (windowingMode == WINDOWING_MODE_FULLSCREEN) {
            if (!mCreatedByOrganizer)
                // 这里条件满足，因此又被设置为空
                // Use empty bounds to indicate "fill parent".
                outOverrideBounds.setEmpty();
            }
            // The bounds for fullscreen mode shouldn't be adjusted by minimal size. Otherwise if
            // the parent or display is smaller than the size, the content may be cropped.
            return;
        }

        ...
    }
```

因此，需要添加一个标记来让 Task 知道，在这个需求下面不对 mResolvedOverrideConfiguration 的 Bounds 置空。     
在 WindowConfiguration 中添加下面方法：     

```
public class WindowConfiguration implements Parcelable, Comparable<WindowConfiguration> {

    private boolean isTestMiniWindow;
    /**
     * @hide
     */
    public void setIsTestMiniWindow(boolean isTestMiniWindow) {
        this.isTestMiniWindow = isTestMiniWindow;
    }
    /**
     * @hide
     */
    public boolean isTestMiniWindow() {
        return isTestMiniWindow;
    }
```

然后在 在 DefaultTransitionHandler.startAnimation 方法中设置 setIsTestMiniWindow。     

```
    @Override
    public boolean startAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
            @NonNull SurfaceControl.Transaction startTransaction,
            @NonNull SurfaceControl.Transaction finishTransaction,
            @NonNull Transitions.TransitionFinishCallback finishCallback) {
        ...
        final Runnable onAnimFinish = () -> {
            if (!animations.isEmpty()) return;
            mAnimations.remove(transition);
            // ----->
            WindowContainerTransaction wct = new WindowContainerTransaction();
            for (int i = info.getChanges().size() - 1; i >= 0; --i) {
                TransitionInfo.Change change = info.getChanges().get(i);
                if (change.getMode() == TRANSIT_OPEN && change.getTaskInfo() != null
                        && change.getTaskInfo().topActivity.getClassName().contains("com.hq.android.androiddemo.common")) {
                    wct.setTestMiniWindow(change.getTaskInfo().token, true);
                    wct.setBounds(change.getTaskInfo().token, new Rect(100, 200, 600, 1600));
                }
            }
            finishCallback.onTransitionFinished(wct );
            // <-------
            //finishCallback.onTransitionFinished(null /* wct */);
```

在 WindowContainerTransaction 中添加方法：     

```
    /**
     * @hide
     */
    @NonNull
    public WindowContainerTransaction setTestMiniWindow(
            @NonNull WindowContainerToken container, boolean isMiniWindow) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.windowConfiguration.setIsTestMiniWindow(isMiniWindow);
        return this;
    }
```

还需要在 Task.resolveLeafTaskOnlyOverrideConfigs 方法中添加下面判断：      

```
// Task.java
    void resolveLeafTaskOnlyOverrideConfigs(Configuration newParentConfig, Rect previousBounds) {
        ...
        Rect outOverrideBounds = getResolvedOverrideConfiguration().windowConfiguration.getBounds();

        if (windowingMode == WINDOWING_MODE_FULLSCREEN) {
            if (!mCreatedByOrganizer && !getResolvedOverrideConfiguration().windowConfiguration.isTestMiniWindow()) { 
                // Use empty bounds to indicate "fill parent".
                outOverrideBounds.setEmpty();
            }
            // The bounds for fullscreen mode shouldn't be adjusted by minimal size. Otherwise if
            // the parent or display is smaller than the size, the content may be cropped.
            return;
        }

        ...
    }
```

再次打开 Activity，就会以小窗形式打开了。     
但是，小窗周围是黑色的区域，通过 Winscope 发现，是 Letterbox，可以通过下面方式去掉：    

```
//LetterboxUiController.java
    @VisibleForTesting
    boolean shouldShowLetterboxUi(WindowState mainWindow) {
        // ------>
        if (mActivityRecord.getTask().getResolvedOverrideConfiguration().windowConfiguration.isTestMiniWindow()) {
            return false;
        }
        // <------
```

发现去掉后，后面的内容还是没有绘制。    

做如下修改：

```
//TaskFragment.java

    int getVisibility(ActivityRecord starting) {
            final int otherWindowingMode = other.getWindowingMode();
            //if (otherWindowingMode == WINDOWING_MODE_FULLSCREEN
            // ----->
            if (otherWindowingMode == WINDOWING_MODE_FULLSCREEN && !other.getResolvedOverrideConfiguration().windowConfiguration.isTestMiniWindow()
            // <------
                    || (otherWindowingMode != WINDOWING_MODE_PINNED && other.matchParentBounds())) {
                    // Can be visible behind a translucent fullscreen TaskFragment.
                    gotTranslucentFullscreen = true;
                    continue;
                }
                return TASK_FRAGMENT_VISIBILITY_INVISIBLE;
            }
    }
```

就会在前面设置的区域内以小窗形式打开。     

<img src="/images/android-window-system-pip/1.png" width="578" height="722"/>

通过这个作业，主要是学习了窗口可见性设置的关键方法，以及 `WindowContainerTransaction() ` 的运用。      


