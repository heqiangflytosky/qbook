---
title: Android 分屏
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 分屏
date: 2022-11-23 10:00:00
---

## 相关类

### 窗口层级

分屏前，Task 12 和 Task 13 分别对应两个即将分屏的 Task，mode=fullscreen。Task 9 对应分屏容器所在的 Task，下面有两个 Task 10 和 Task 11，mode=MULTI-WINDOW，用来状态需要分屏的 Task。     
分屏后，Task 12 和 Task 13 移动到了 Task 9 下面的 Task 10 和 Task 11。mode 变成了 MULTI-WINDOW。      

<img src="/images/android-window-system-split-screen/0.png" width="757" height="142"/>

### SplitScreenController

分屏管理器，负责构造 StageCoordinator，负责和 Launcher 进行通信。     

### StageCoordinator

协调分屏 MainStage 和 SideStage 可见性、大小调整等，MainStage 和 SideStage作为分屏 RootTask 的载体，分别代表主分屏区和辅助分屏区。      
创建分屏的 RootTask(36) 和上下分屏的 Root Task(37和38)。      

```
                  │  ├─ Task=36 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
                  │  │  ├─ Task=37 type=undefined mode=MULTI-WINDOW override-mode=MULTI-WINDOW requested-bounds=[0,1184][1080,2340] bounds=[0,1184][1080,2340]
                  │  │  └─ Task=38 type=undefined mode=MULTI-WINDOW override-mode=MULTI-WINDOW requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]

```

StageCoordinator 实现了 ShellTaskOrganizer.TaskListener 接口，来监听 Task 的变化情况。       

```
public class StageCoordinator implements SplitLayout.SplitLayoutHandler,
        DisplayController.OnDisplaysChangedListener, Transitions.TransitionHandler,
        ShellTaskOrganizer.TaskListener {
    ......
    // 表示 SideStage 代表的是上分屏还是下分屏
    private int mSideStagePosition = SPLIT_POSITION_BOTTOM_OR_RIGHT;

    protected StageCoordinator(Context context, int displayId, SyncTransactionQueue syncQueue,
            ShellTaskOrganizer taskOrganizer, DisplayController displayController,
            DisplayImeController displayImeController,
            DisplayInsetsController displayInsetsController, Transitions transitions,
            TransactionPool transactionPool,
            IconProvider iconProvider, ShellExecutor mainExecutor,
            Optional<RecentTasksController> recentTasks,
            LaunchAdjacentController launchAdjacentController,
            Optional<WindowDecorViewModel> windowDecorViewModel) {
        ......
        // 创建 Root Task，并设置自己为 TaskListener    
        taskOrganizer.createRootTask(displayId, WINDOWING_MODE_FULLSCREEN, this /* listener */);

        ProtoLog.d(WM_SHELL_SPLIT_SCREEN, "Creating main/side root task");
        // 创建 MainStage 和 SideStage
        mMainStage = new MainStage(
                mContext,
                mTaskOrganizer,
                mDisplayId,
                mMainStageListener,
                mSyncQueue,
                mSurfaceSession,
                iconProvider,
                mWindowDecorViewModel);
        mSideStage = new SideStage(
                mContext,
                mTaskOrganizer,
                mDisplayId,
                mSideStageListener,
                mSyncQueue,
                mSurfaceSession,
                iconProvider,
                mWindowDecorViewModel);
        ....
    }
```

当分屏 RootTask 创建完成后，再挂载上下分屏的 RootTask。       

```
    public void onTaskAppeared(ActivityManager.RunningTaskInfo taskInfo, SurfaceControl leash) {
        if (mRootTaskInfo != null || taskInfo.hasParentTask()) {
            throw new IllegalArgumentException(this + "\n Unknown task appeared: " + taskInfo);
        }

        ProtoLog.d(WM_SHELL_SPLIT_SCREEN, "onTaskAppeared: task=%s", taskInfo);
        mRootTaskInfo = taskInfo;
        mRootTaskLeash = leash;
        // 创建 SplitLayout
        if (mSplitLayout == null) {
            mSplitLayout = new SplitLayout(TAG + "SplitDivider", mContext,
                    mRootTaskInfo.configuration, this, mParentContainerCallbacks,
                    mDisplayController, mDisplayImeController, mTaskOrganizer,
                    PARALLAX_ALIGN_CENTER /* parallaxType */);
            mDisplayInsetsController.addInsetsChangedListener(mDisplayId, mSplitLayout);
        }
        // 挂载两个分屏的 Root Task。    
        onRootTaskAppeared();
    }
    
    void onRootTaskAppeared() {
        ProtoLog.d(WM_SHELL_SPLIT_SCREEN, "onRootTaskAppeared: rootTask=%s mainRoot=%b sideRoot=%b",
                mRootTaskInfo, mMainStageListener.mHasRootTask, mSideStageListener.mHasRootTask);
        // Wait unit all root tasks appeared.
        if (mRootTaskInfo == null
                || !mMainStageListener.mHasRootTask
                || !mSideStageListener.mHasRootTask) {
            return;
        }

        final WindowContainerTransaction wct = new WindowContainerTransaction();
        // 挂载上下分屏的 Root Task。
        wct.reparent(mMainStage.mRootTaskInfo.token, mRootTaskInfo.token, true);
        wct.reparent(mSideStage.mRootTaskInfo.token, mRootTaskInfo.token, true);
        // Make the stages adjacent to each other so they occlude what's behind them.
        wct.setAdjacentRoots(mMainStage.mRootTaskInfo.token, mSideStage.mRootTaskInfo.token);
        setRootForceTranslucent(true, wct);
        mSplitLayout.getInvisibleBounds(mTempRect1);
        wct.setBounds(mSideStage.mRootTaskInfo.token, mTempRect1);
        mSyncQueue.queue(wct);
        mSyncQueue.runInSync(t -> {
            t.setPosition(mSideStage.mRootLeash, mTempRect1.left, mTempRect1.top);
        });
        mLaunchAdjacentController.setLaunchAdjacentRoot(mSideStage.mRootTaskInfo.token);
    }
```

### SplitDecorManager

通常我们在缩放分屏时，在被拉伸的 Task 上面会有个遮罩层，用来遮挡应用程序在拉伸过程中出现的奇怪的布局。    
遮罩层的构成有 mIconLeash（显示图标），mBackgroundLeash 和 mGapBackgroundLeash。    
SplitDecorManager 继承自 WindowlessWindowManager，表示这个界面不需要 WMS 的窗口管理，直接操作 Layer 图层进行界面显示。      

```
public class SplitDecorManager extends WindowlessWindowManager {
```

对应图层:      
对于每个分屏分别构造了下面图层：
 - SplitDecorManager(mIconLeash)，mBackgroundLeash 和 mGapBackgroundLeash，它们和分屏的两个 Task( Task 1456 和 Task 1407)分别挂载到了同一个父层级。     

<img src="/images/android-window-system-split-screen/1.png" width="436" height="593"/>

### SplitWindowManager

持有分屏操作窗口的根布局以及帮助分屏分隔条进行分屏。       
主要作用是协调两个应用窗口的布局和交互。        

```
public final class SplitWindowManager extends WindowlessWindowManager {
    private SurfaceControlViewHost mViewHost;
    private SurfaceControl mLeash;
    private DividerView mDividerView;
```

对应图层见上图:      
和两个分屏的父 Task(Task 10 和 Task 11) 挂载到了同一个父层级。     

分屏的时候我们看到了上下屏都绘制了圆角，但是我们看 sf 图层却没有圆角效果，这时为什么呢？其实是分隔条所在的图层绘制的效果，盖再来分屏的图层上面，具体在 DividerRoundedCorner 里面绘制。      

## MainStage 和 SideStage

代表分屏的上下屏幕，继承 StageTaskListener。      
初始化时为上下分屏创建 Root Task。     

```
    StageTaskListener(Context context, ShellTaskOrganizer taskOrganizer, int displayId,
            StageListenerCallbacks callbacks, SyncTransactionQueue syncQueue,
            SurfaceSession surfaceSession, IconProvider iconProvider,
            Optional<WindowDecorViewModel> windowDecorViewModel) {
        mContext = context;
        mCallbacks = callbacks;
        mSyncQueue = syncQueue;
        mSurfaceSession = surfaceSession;
        mIconProvider = iconProvider;
        mWindowDecorViewModel = windowDecorViewModel;
        taskOrganizer.createRootTask(displayId, WINDOWING_MODE_MULTI_WINDOW, this);
    }
```

## SplitScreenTransitions

管理分屏的过渡动画，以及和 server 端的通信。       

## 桌面启动分屏

桌面部分的主要工作是选择上下分屏的 Task，将 Task 传给 SystemUI 执行分屏操作，完成真正分屏前的一些动画。     

### 选择上分屏

```
TaskShortcutFactory.SplitSelectSystemShortcut.onClick()
    TaskView.initiateSplitSelect
        LauncherRecentsView.initiateSplitSelect
            StateManager.goToState
                StateManager.goToStateAnimated
                    // 创建动画
                    StateManager.createAnimationToNewWorkspaceInternal
                        BaseRecentsViewStateController.setStateWithAnimation
                            RecentsViewStateController.setStateWithAnimationInternal
                                RecentsViewStateController.handleSplitSelectionState
                                    RecentsView.createSplitSelectInitAnimation
                                        RecentsView.createTaskDismissAnimation
                                            RecentsView.createInitialSplitSelectAnimation
                                                // 构造 FloatingTaskThumbnailView
                                                FloatingTaskView.getFloatingTaskView()
                    // 开启动画
                    new StartAnimRunnable(animation)
```

### 选择下分屏

```
TaskView.confirmSecondSplitSelectApp()
    RecentsView.confirmSplitSelect()
        SplitSelectStateController.setSecondTask()
        new PendingAnimation
        pendingAnimation.addEndListener
            //动画完成时，启动分屏
            SplitSelectStateController.launchSplitTasks
                SplitSelectStateController.launchTasks
                    SplitSelectStateController.getShellRemoteTransition
                        // 创建动画执行器
                        new RemoteSplitLaunchTransitionRunner
                        new RemoteTransition
                    SystemUiProxy.startTasks()
                        ISplitScreen.startTasks()  -----> SystemUI 执行
        // 开启动画
        pendingAnimation.buildAnim().start()
```

## SystemUI 启动分屏流程

从 Launcher 到 SystemUI 传递了上下分屏的 taskID，SplitPosition类型等参数。     

```
        public void startTasks(int taskId1, @Nullable Bundle options1, int taskId2,
                @Nullable Bundle options2, @SplitPosition int splitPosition,
                @PersistentSnapPosition int snapPosition,
                @Nullable RemoteTransition remoteTransition, InstanceId instanceId) {
```

```
SplitScreenController.ISplitScreenImpl.startTasks()
    //线程切换 binder -> wmshell.main
    executeRemoteCallWithTaskPermission()
        StageCoordinator.startTasks()
            // 创建一个 
            new WindowContainerTransaction()
            StageCoordinator.setSideStagePosition()
                //设置 mSideStagePosition
                mSideStagePosition = sideStagePosition;
            // 关联options1与taskId1
            WindowContainerTransaction.startTask()
                // 创建 HierarchyOp
                HierarchyOp.createForTaskLaunch()
                // 添加到 mHierarchyOps
                mHierarchyOps.add()
            StageCoordinator.startWithTask()
                // 设置分屏比例
                SplitLayout.setDivideRatio(snapPosition)
                    // 设置分割线
                    SplitLayout.setDividerPosition()
                        SplitLayout.updateBounds()
                // 设置上下屏的bound，传递到 server 端执行
                StageCoordinator.updateWindowBounds()
                    SplitLayout.applyTaskChanges()
                        SplitLayout.setTaskBounds()
                            //设置 Task 显示区域
                            WindowContainerTransaction.setBounds()
                                WindowContainerTransaction.getOrCreateChange()
                                    // 报存到 mChanges
                                    mChanges.put()
                            // 分屏后，屏幕最小宽度可能发生变化，需要重新设置一下
                            WindowContainerTransaction.setSmallestScreenWidthDp()
                //让装载分屏的rootTask进行reorder，主要就是为了把分屏移到最前面
                WindowContainerTransaction.reorder()
                // 添加启动的下分屏task
                WindowContainerTransaction.startTask()
                SplitScreenTransitions.startEnterTransition(TRANSIT_TO_FRONT)
                    //启动动画和提交WindowContainerTransaction相关事务
                    Transitions.startTransition()
                        // 通过mOrganizer.startNewTransition启动一个新的事务，调到 system_servr 端
                        new ActiveTransition(mOrganizer.startNewTransition(type, wct))
                            WindowOrganizer.startNewTransition
                                // system_server开始执行事务，参数 type 为 TRANSIT_TO_FRONT
                                getWindowOrganizerController().startNewTransition() ----> system_servr
                        // 放到mPendingTransitions 中，等待 同步 准备好后执行
                        mPendingTransitions.add(active)
                    //设置mPendingEnter动画
                    SplitScreenTransitions.setEnterTransition()
                        mPendingEnter = new EnterSession()
                    
```

我们mSplitTransitions.startEnterTransition这个方法中传递了WindowContainerTransaction对象wct，用于后续system_server流程中提交该事务，并且启动分屏。      

```
// Transitions.java
    public IBinder startTransition(@WindowManager.TransitionType int type,
            @NonNull WindowContainerTransaction wct, @Nullable TransitionHandler handler) {
        //启动新的过渡动画,开启同步过程，并提交WindowContainerTransaction相关事务
        final ActiveTransition active =
                new ActiveTransition(mOrganizer.startNewTransition(type, wct));
        active.mHandler = handler;
        mKnownTransitions.put(active.mToken, active);
        mPendingTransitions.add(active);
        return active.mToken;
    }
```

当一切同步工作都准备好后,就开始执行刚才的 ActiveTransition，这部分具体介绍在 ShellTransition 一节中。      

```
Transitions.TransitionPlayerImpl.onTransitionReady()
    Transitions.onTransitionReady()
        Transitions.dispatchReady()
            // 中间其他部分省略，和前面 ShellTransition 中介绍的了流程相似
            Transitions.processReadyQueue()
                active.mHandler.startAnimation() 实际执行 StageCoordinator.startAnimation()
                    StageCoordinator.startPendingAnimation()
                        StageCoordinator.startPendingEnterAnimation()
                            StageCoordinator.finishEnterSplitScreen
                                SplitLayout.update()
                                    SplitLayout.init()
                                        SplitWindowManager.init()
                                            new SurfaceControlViewHost()
                                                SurfaceControlViewHost.setView()
                                                    ViewRootImpl.setView
                                                        WindowlessWindowManager.addToDisplayAsUser
                                                            WindowlessWindowManager.addToDisplay
                                                                SplitWindowManager.getParentSurface
                                                                    // 构建 Leash
                                                                    new SurfaceControl.Builder.build()
                                // 构建上下分屏遮罩的视图
                                SplitDecorManager.inflate()
                                    new SurfaceControlViewHost()
                                        SurfaceControlViewHost.setView()
                                finishT.show(mRootTaskLeash)
                                // 开始两个分屏出现的动画
                                SplitScreenTransitions.playAnimation()
                                    // 获取到前面创建的 EnterSession
                                    getPendingTransition()
                                    // 执行到 桌面侧 ----> launcher
                                    SplitScreenTransitions.EnterSession.mRemoteHandler.startAnimation
```


## 桌面执行动画

SplitSelectStateController.RemoteSplitLaunchTransitionRunner.startAnimation()


## system_server 部分

前面说到 SystemUI 在处理分屏逻辑时，创建了 WindowContainerTransaction，经过一系列的属性设置后，通过 `mOrganizer.startNewTransition(type, wct)` 将相关事务提交到 system_server，那么接下来就讲解 WindowOrganizerController是如何处理WindowContainerTransaction的。    

systemui 通过 `getWindowOrganizerController().startNewTransition()` 方法跨进程通信到 system_server。    

```
WindowOrganizerController.startNewTransition()
    TransitionController.startCollectOrQueue()
        mCollectingTransition = transition;
        Transition.startCollecting()
            // 修改状态
            mState = STATE_COLLECTING;
            // BLASTSyncEngine 流程，此处略去
            BLASTSyncEngine.startSyncSet()
        // 也就是传入的回调，在 startCollectOrQueue 方法中定义
        onStartCollect.onCollectStarted
            Transition.start()
                mState = STATE_STARTED;
            // 开始处理事务
            WindowOrganizerController.applyTransaction()
                Transition.deferTransitionReady()
                // 获取 HierarchyOp，也就是前面对 WindowContainerTransaction的一系列操作
                // 包括 setBounds、startTask、reorder等
                WindowContainerTransaction.getHierarchyOps()
                // 遍历 WindowContainerTransaction.Change，并应用到窗口
                WindowOrganizerController.applyWindowContainerChange()
                // 应用 HierarchyOp相关，主要是 
                WindowOrganizerController.applyHierarchyOp()
                // 获取 mBoundsChangeSurfaceBounds，并应用到对应窗口上
                WindowContainerTransaction.Change.getBoundsChangeSurfaceBounds()
                SurfaceControl.Transaction.setPosition
                SurfaceControl.Transaction.setWindowCrop
                // 更新Activity的可见性和Configuration
                RootWindowContainer.ensureActivitiesVisible()
                RootWindowContainer.resumeFocusedTasksTopActivities()
                ActivityRecord.ensureActivityConfiguration
                ActivityTaskManagerService.continueWindowLayout()
                
```


### 事务处理

WindowOrganizerController.applyTransaction() 主要做了下面几件事：
 - 遍历 WindowContainerTransaction.Change，并应用到窗口
 - 应用 HierarchyOp相关
 - 获取 mBoundsChangeSurfaceBounds，并应用到对应窗口上
 - 更新Activity的可见性和Configuration

## 分屏拉伸

当触摸分割线进行移动时会动态改变上下两个分屏的大小。      

```
DividerView.onTouch()：MotionEvent.ACTION_MOVE
    SplitLayout.updateDividerBounds()
        SplitLayout.updateBounds()
        StageCoordinator.onLayoutSizeChanging()
            StageCoordinator.updateSurfaceBounds()
                //调整上下分屏的位置
                SplitLayout.applySurfaceChanges()
                    SurfaceControl.Transaction.setPosition(leash1, mTempRect.left, mTempRect.top)
                    SurfaceControl.Transaction.setPosition(leash2, mTempRect.left, mTempRect.top)
            // 获取上下分屏的区域
            StageCoordinator.getMainStageBounds(mTempRect1)
            StageCoordinator.getSideStageBounds(mTempRect2)
            MainStage.onResizing()
            SideStage.onResizing()
                SplitDecorManager.onResizing()
                    // 如果需要就去构造这两个图层
                    mBackgroundLeash = SurfaceUtils.makeColorLayer
                    mGapBackgroundLeash = SurfaceUtils.makeColorLayer
                    // 设置 mIconLeash 的位置
                    SurfaceControl.Transaction.setPosition(mIconLeash
                    SplitDecorManager.startFadeAnimation()
                        mFadeAnimator.addUpdateListener
                            //对 mBackgroundLeash 和 mIconLeash 图层做alpha动画
                            SurfaceControl.Transaction.setAlpha(mBackgroundLeash)
                            SurfaceControl.Transaction.setAlpha(mIconLeash
                            SurfaceControl.Transaction.apply()
                        // 启动显示遮罩层的动画
                        mFadeAnimator.start()
            SurfaceControl.Transaction.apply()
```

移动结束，ACTION_UP 时，开始 Transition 动画。     

```
DividerView.onTouch()：MotionEvent.ACTION_UP
    SplitLayout.snapToTarget()
        SplitLayout.flingDividerPosition()
            AnimatorUpdateListener.onAnimationUpdate
                SplitLayout.updateDividerBounds
            AnimatorListenerAdapter.onAnimationEnd
                SplitLayout.setDividerPosition
                    StageCoordinator.onLayoutSizeChanged
                        new WindowContainerTransaction()
                        StageCoordinator.updateWindowBounds
                            SplitScreenTransitions.startResizeTransition
                                SplitLayout.applyTaskChanges()
                                    SplitLayout.setTaskBounds()
                                        WindowContainerTransaction.setBounds()
                        SplitScreenTransitions.startResizeTransition()
                            Transitions.startTransition(TRANSIT_CHANGE)
            ValueAnimator.start()
```

动画准备好后，    

```
Transitions$TransitionPlayerImpl.onTransitionReady
    Transitions.onTransitionReady
        Transitions.dispatchReady
            Transitions.processReadyQueue
                Transitions.playTransition
                    StageCoordinator.startAnimation
                        StageCoordinator.startPendingAnimation
                            SplitScreenTransitions.playResizeAnimation
                                // 遍历收集到的动画参与者
                                for (int i = info.getChanges().size() - 1; i >= 0; --i) {
                                SplitDecorManager.onResized()
                                    SplitDecorManager.fadeOutDecor()
                                        // 执行因此遮罩图层的动画
                                        SplitDecorManager.startFadeAnimation()
                                            onAnimationUpdate
                                                // 修改遮罩图层的透明度
                                                SurfaceControl.Transaction.setAlpha
                                            onAnimationEnd
                                                // 删除遮罩图层
                                                SplitDecorManager.releaseDecor()
                                        mShown = false
                                    SplitDecorManager.releaseDecor()
                                        // 删除遮罩图层
                                        SurfaceControl.Transaction.remove(mBackgroundLeash)
                                        SurfaceControl.Transaction.remove(mGapBackgroundLeash)
```


## 分屏切换

双击分屏中间的分隔条可以实现上下分屏的切换。    
对比窗口层级树我们会发现，切换分屏前后层级树结构没有发生任何变化，只是把上下分屏的位置做了改变。然后修改了 mSideStagePosition 模式。    

<img src="/images/android-window-system-split-screen/3.png" width="936" height="92"/>

```
DividerView$DoubleTapListener.onDoubleTap
    SplitLayout.onDoubleTappedDivider
        StageCoordinator.onDoubleTappedDivider
            StageCoordinator.switchSplitPosition
                // 对上下两个分屏进行截图，
                // 将这两个截图图层挂载到两个分屏的RootTask对应节点上
                ScreenshotUtils.takeScreenshot()
                ScreenshotUtils.takeScreenshot()
                SplitLayout.splitSwitching
                    // 分屏构造上下分屏和分割线动画
                    SplitLayout.moveSurface
                    SplitLayout.moveSurface
                    SplitLayout.moveSurface
                    onAnimationEnd()
                        StageCoordinator.reverseSplitPosition()
                        StageCoordinator.setSideStagePosition()
                            StageCoordinator.updateWindowBounds()
                                SplitLayout.applyTaskChanges()
                                    SplitLayout.setTaskBounds()
                                        WindowContainerTransaction.setBounds()
                                        WindowContainerTransaction.setSmallestScreenWidthDp()
                    AnimatorSet.start()
```

ScreenshotUtils screenshot 图层：    

<img src="/images/android-window-system-split-screen/2.png" width="380" height="461"/>

```
    void switchSplitPosition(String reason) {
        ProtoLog.d(WM_SHELL_SPLIT_SCREEN, "switchSplitPosition");
        final SurfaceControl.Transaction t = mTransactionPool.acquire();
        mTempRect1.setEmpty();
        final StageTaskListener topLeftStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mSideStage : mMainStage;
        final SurfaceControl topLeftScreenshot = ScreenshotUtils.takeScreenshot(t,
                topLeftStage.mRootLeash, mTempRect1, Integer.MAX_VALUE - 1);
        final StageTaskListener bottomRightStage =
                mSideStagePosition == SPLIT_POSITION_TOP_OR_LEFT ? mMainStage : mSideStage;
        final SurfaceControl bottomRightScreenshot = ScreenshotUtils.takeScreenshot(t,
                bottomRightStage.mRootLeash, mTempRect1, Integer.MAX_VALUE - 1);
        mSplitLayout.splitSwitching(t, topLeftStage.mRootLeash, bottomRightStage.mRootLeash,
                insets -> {
                    // onAnimationEnd 回调执行
                    // 构造 WindowContainerTransaction，因为需要最终由 system_server 来修改 Task 位置
                    WindowContainerTransaction wct = new WindowContainerTransaction();
                    // 设置最终的 Task 位置，以及修改 mSideStagePosition
                    setSideStagePosition(reverseSplitPosition(mSideStagePosition), wct);
                    // 把 WindowContainerTransaction 放到 SyncTransactionQueue 中执行
                    mSyncQueue.queue(wct);
                    mSyncQueue.runInSync(st -> {
                        updateSurfaceBounds(mSplitLayout, st, false /* applyResizingOffset */);
                        st.setPosition(topLeftScreenshot, -insets.left, -insets.top);
                        st.setPosition(bottomRightScreenshot, insets.left, insets.top);

                        final ValueAnimator va = ValueAnimator.ofFloat(1, 0);
                        va.addUpdateListener(valueAnimator-> {
                            final float progress = (float) valueAnimator.getAnimatedValue();
                            t.setAlpha(topLeftScreenshot, progress);
                            t.setAlpha(bottomRightScreenshot, progress);
                            t.apply();
                        });
                        va.addListener(new AnimatorListenerAdapter() {
                            @Override
                            public void onAnimationEnd(
                                    @androidx.annotation.NonNull Animator animation) {
                                t.remove(topLeftScreenshot);
                                t.remove(bottomRightScreenshot);
                                t.apply();
                                mTransactionPool.release(t);
                            }
                        });
                        va.start();
                    });
                });
    }
```

```
    // 构造切换分屏的动画
    // leash1和leash2分别代表 两个分屏所在 Task的SurfaceControl
    // finishCallback在 StageCoordinator.switchSplitPosition 方法中构造，
    // 动画执行完毕后执行，主要是隐藏 Task 截图，调整 mSideStagePosition 的值。
    //以及修改上面分屏 Task 的最终位置
    public void splitSwitching(SurfaceControl.Transaction t, SurfaceControl leash1,
            SurfaceControl leash2, Consumer<Rect> finishCallback) {
        final Rect insets = getDisplayStableInsets(mContext);
        insets.set(mIsLeftRightSplit ? insets.left : 0, mIsLeftRightSplit ? 0 : insets.top,
                mIsLeftRightSplit ? insets.right : 0, mIsLeftRightSplit ? 0 : insets.bottom);

        final int dividerPos = mDividerSnapAlgorithm.calculateNonDismissingSnapTarget(
                mIsLeftRightSplit ? mBounds2.width() : mBounds2.height()).position;
        final Rect distBounds1 = new Rect();
        final Rect distBounds2 = new Rect();
        final Rect distDividerBounds = new Rect();
        // Compute dist bounds.
        updateBounds(dividerPos, distBounds2, distBounds1, distDividerBounds,
                false /* setEffectBounds */);
        // Offset to real position under root container.
        distBounds1.offset(-mRootBounds.left, -mRootBounds.top);
        distBounds2.offset(-mRootBounds.left, -mRootBounds.top);
        distDividerBounds.offset(-mRootBounds.left, -mRootBounds.top);
        // 构造动画
        ValueAnimator animator1 = moveSurface(t, leash1, getRefBounds1(), distBounds1,
                -insets.left, -insets.top);
        ValueAnimator animator2 = moveSurface(t, leash2, getRefBounds2(), distBounds2,
                insets.left, insets.top);
        ValueAnimator animator3 = moveSurface(t, getDividerLeash(), getRefDividerBounds(),
                distDividerBounds, 0 /* offsetX */, 0 /* offsetY */);

        AnimatorSet set = new AnimatorSet();
        set.playTogether(animator1, animator2, animator3);
        set.setDuration(FLING_SWITCH_DURATION);
        set.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animation) {
                InteractionJankMonitorUtils.beginTracing(CUJ_SPLIT_SCREEN_DOUBLE_TAP_DIVIDER,
                        mContext, getDividerLeash(), null /*tag*/);
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                mDividerPosition = dividerPos;
                updateBounds(mDividerPosition);
                finishCallback.accept(insets);
                InteractionJankMonitorUtils.endTracing(CUJ_SPLIT_SCREEN_DOUBLE_TAP_DIVIDER);
            }

            @Override
            public void onAnimationCancel(Animator animation) {
                InteractionJankMonitorUtils.cancelTracing(CUJ_SPLIT_SCREEN_DOUBLE_TAP_DIVIDER);
            }
        });
        set.start();
    }
    
    private ValueAnimator moveSurface(SurfaceControl.Transaction t, SurfaceControl leash,
            Rect start, Rect end, float offsetX, float offsetY) {
        Rect tempStart = new Rect(start);
        Rect tempEnd = new Rect(end);
        final float diffX = tempEnd.left - tempStart.left;
        final float diffY = tempEnd.top - tempStart.top;
        final float diffWidth = tempEnd.width() - tempStart.width();
        final float diffHeight = tempEnd.height() - tempStart.height();
        ValueAnimator animator = ValueAnimator.ofFloat(0, 1);
        animator.addUpdateListener(animation -> {
            if (leash == null) return;

            final float scale = (float) animation.getAnimatedValue();
            final float distX = tempStart.left + scale * diffX;
            final float distY = tempStart.top + scale * diffY;
            final int width = (int) (tempStart.width() + scale * diffWidth);
            final int height = (int) (tempStart.height() + scale * diffHeight);
            // 更新分屏的位置
            if (offsetX == 0 && offsetY == 0) {
                t.setPosition(leash, distX, distY);
                t.setWindowCrop(leash, width, height);
            } else {
                final int diffOffsetX = (int) (scale * offsetX);
                final int diffOffsetY = (int) (scale * offsetY);
                t.setPosition(leash, distX + diffOffsetX, distY + diffOffsetY);
                mTempRect.set(0, 0, width, height);
                mTempRect.offsetTo(-diffOffsetX, -diffOffsetY);
                t.setCrop(leash, mTempRect);
            }
            // apply 事务
            t.apply();
        });
        return animator;
    }
```

reverseSplitPosition 方法将分屏模式进行翻转。比如以前的 SideStage 代表下分屏，那么反转后就代表上分屏。     

```
    public static int reverseSplitPosition(@SplitScreenConstants.SplitPosition int position) {
        switch (position) {
            case SPLIT_POSITION_TOP_OR_LEFT:
                return SPLIT_POSITION_BOTTOM_OR_RIGHT;
            case SPLIT_POSITION_BOTTOM_OR_RIGHT:
                return SPLIT_POSITION_TOP_OR_LEFT;
            case SPLIT_POSITION_UNDEFINED:
            default:
                return SPLIT_POSITION_UNDEFINED;
        }
    }
```


## 分屏退出

```
DividerView.onTouch
    MotionEvent.ACTION_UP
        SplitLayout.snapToTarget()
            SplitLayout.flingDividerPosition()
                onAnimationUpdate
                onAnimationEnd
                    StageCoordinator.onSnappedToDismiss
                        // 创建 WindowContainerTransaction
                        WindowContainerTransaction wct = new WindowContainerTransaction()
                        StageCoordinator.prepareExitSplitScreen
                            // 分屏的Task重新挂载
                            SideStage.removeAllTasks()
                                WindowContainerTransaction.reparentTasks()
                            MainStage.deactivate
                                WindowContainerTransaction.reparentTasks()
                        SplitScreenTransitions.startDismissTransition()
                            Transitions.startTransition(TRANSIT_SPLIT_DISMISS_SNAP)
                            mPendingDismiss = new DismissSession
                ValueAnimator.start()
```



```
Transitions$TransitionPlayerImpl.onTransitionReady
    Transitions.onTransitionReady
        Transitions.dispatchReady
            Transitions.processReadyQueue
                Transitions.playTransition
                    StageCoordinator.startAnimation
                        StageCoordinator.startPendingAnimation
                            SplitScreenTransitions.playDragDismissAnimation
                                SplitDecorManager.onResized
                                    SplitDecorManager.fadeOutDecor
                                        //隐藏遮罩层动画
                                        SplitDecorManager.startFadeAnimation
                                
```


## 相关文章

[Android U 多任务启动分屏——Launcher流程（上分屏）](https://blog.csdn.net/yimelancholy/article/details/141851599)      
[aosp13/14命令行进入分屏相关实战](https://zhuanlan.zhihu.com/p/692879274)      
[Android U system_server侧WindowContainerTransaction处理](https://blog.csdn.net/yimelancholy/article/details/144694952)      
[安卓分屏下Activity启动其他Activity为啥也在分屏下？-framework深入剖析](https://zhuanlan.zhihu.com/p/706097799)      
[android T 分屏流程之systemui部分/android framework车载车机手机实战开发 ](https://juejin.cn/post/7265891728163749903)      
