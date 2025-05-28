---
title: Android Task 切换动画
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android Task 切换动画
date: 2022-11-23 10:00:00
---


## 概述

我们以启动 Activity 时需要创建新的 Task 这样的场景来介绍一下 Task 切换动画的执行流程。     

## 图层分析

Task 切换动画同样会创建一个 Transition Root 图层，作为动画图层的根图层，把两个 Task 挂载到这个 Leash 图层下面来做动画。     
动画的本质是对两个 Task 分别修改它们的位置和缩放。     
动画过程中还创建了一个 "animation-background" 的背景图层。      
<img src="/images/android-window-system-task-switch-animation/1.png" width="593" height="338"/>

## 代码流程分析 

这里重点介绍 Task 切换动画和其他动画不同的地方，其他流程可以参考前面博客。     

### 动画准备

```
ActivityStarter.startActivityUnchecked
    // 构建 TRANSIT_OPEN 类型的动画
    TransitionController.createAndStartCollecting(TRANSIT_OPEN)
        // 由于当前没有 Transition 正在执行，因此这里新建一个
        transit = new Transition(TRANSIT_OPEN)
        TransitionController.moveToCollecting()
            // 赋值 mCollectingTransition，表示当前有动画正在 Collecting
            mCollectingTransition = transition;
            // mCollectingTransition 执行收集流程
            Transition.startCollecting()
    TransitionController.collect(ActivityRecord)
        // 把当前的 ActivityRecord 添加到同步组中去
        Transition.collect()
            BLASTSyncEngine.addToSyncSet()
            // 创建 ChangeInfo，并加入到 mChanges 列表
            info =new ChangeInfo
            mChanges.put(wc, info);
            // 添加到动画参与者列表
            mParticipants.add(wc)
    ActivityStarter.startActivityInner    
        ActivityStarter.getOrCreateRootTask()
            RootWindowContainer.getOrCreateRootTask()
                TaskDisplayArea.getOrCreateRootTask()
                    //创建一个新的 Task
                    new Task()
        ActivityStarter.setNewTask
            // 把新创建的 Task 也加入到当前 Transition 的收集列表中
            // 添加完成后 target 容器有两个，一个是 ActivityRecord，一个是 Task
            TransitionController.collectExistenceChange
                Transition.collectExistenceChange
                    Transition.collect()
    ActivityStarter.handleStartResult()
        TransitionController.collectExistenceChange(started);
            Transition.collectExistenceChange
                ChangeInfo.mExistenceChanged = true;
        mTransitionPlayers.getLast().mPlayer.requestStartTransition()
            // 调到 WMShell 端执行
```

WMShell 收到 requestStartTransition    

```
Transitions.TransitionPlayerImpl.requestStartTransition()
    Transitions.requestStartTransition()
        // 循环遍历所有的 TransitionHandler
        // 找出可以执行此动画的 TransitionHandler，并保存在 ActiveTransition
        // 这其实没有找到任何 TransitionHandler 来执行，因为 DefaultTransitionHandler handleRequest 返回的是 null。
        for (int i = mHandlers.size() - 1; i >= 0; --i)
            TransitionHandler.handleRequest()
```

当第一个 Activity pause 时，设置第二个 ActivityRecord 可见。这个过程会再次收集一下第一个 ActivityRecord，当然这个是属于重复收集，也会收集第二个 ActivityRecord到当前同步组内。       


```
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
                                        DisplayContent.ensureActivitiesVisible
                                            WindowContainer.forAllRootTasks
                                                DisplayContent$$accept
                                                    Task.ensureActivitiesVisible
                                                        // 这循环调用所有 Task 
                                                        Task.forAllLeafTasks
                                                            Task$$.accept
                                                                TaskFragment.updateActivityVisibilities
                                                                    EnsureActivitiesVisibleHelper.process
                                                                        // 这循环调用 Task 下所有 ActivityRecord
                                                                        for (int i = mTaskFragment.mChildren.size() - 1; i >= 0; --i) {
                                                                        EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                                                            ActivityRecord.makeVisibleIfNeeded
                                                                                // 设置第二个 Activity 可见
                                                                                ActivityRecord.setVisibility(true)
                                                                                    // 这里也会执行一次收集，属于重复收集
                                                                                    TransitionController.collect()
                                                                                    ActivityRecord.setVisibleRequested()
                                                                                        WindowContainer.setVisibleRequested()
                                                                                            mVisibleRequested = visible;
                                                                                // 设置第一个 ActivityRecord 不可见
                                                                                ActivityRecord.makeInvisible()
                                                                                    ActivityRecord.setVisibility(false)
                                                                                        TransitionController.collect()
                                                                                            Transition.collect()
                                                                                                // 这里把第一个 ActivityRecord 放进同步组
                                                                                                // 这时同步组内有三个容器
                                                                                                info = mChanges.get(wc)
                                                                                                mChanges.put(wc, info)
                                                                                                mParticipants.add(wc)
                                                                                        // 第一个 ActivityRecord mVisibleRequested false
                                                                                        ActivityRecord.setVisibleRequested(false)
```

### onTransactionReady

等待同步组准备好，这里为同步组内的容器设置了动画类型，并且手动显示了 TRANSIT_OPEN 的图层。      
只有设置了 setVisibleRequested(true) ， onTransactionReady 时才会执行 transaction.show(ar.getSurfaceControl());      

```
RootWindowContainer.performSurfacePlacementNoTrace
    BLASTSyncEngine.onSurfacePlacement
        BLASTSyncEngine$SyncGroup.tryFinish
            BLASTSyncEngine$SyncGroup.finishNow
                Transition.onTransactionReady
                    mTargets = Transition.calculateTargets(mParticipants, mChanges, this)
                        Targets targets = new Targets()
                        // 动画层级提升，结果保存在 mTargets
                        Transition.tryPromote()
                    Transition.calculateTransitionInfo(mTargets)
                        TransitionInfo out = new TransitionInfo(type, flags)
                        // 创建 Transition Root 图层，用于动画
                        Transition.calculateTransitionRoots
                        // 这里会循环遍历当前 mTargets 里面的 Change。
                        // 因为经过了tryPromote动画层级提升，
                        // 这里一共有两个，第一个和第二个 ActivityRecord 所在的Task。
                        for (int i = 0; i < count; ++i)
                        ChangeInfo.getTransitMode
                        // 设置 TransitionMode，具体类型参见 TransitionInfo 类
                        // 主要有 TRANSIT_OPEN，TRANSIT_CLOSE，TRANSIT_TO_FRONT，TRANSIT_TO_BACK，TRANSIT_CHANGE
                        // 这为 Task 设置了 TRANSIT_OPEN 和 TRANSIT_TO_BACK 动画
                        TransitionInfo.Change.setMode()
                    // 这里遍历所有的动画参与者，一共三个
                    // 第一个 ActivityRecord，第二个 ActivityRecord 及其所在的 Task
                    for (int i = mParticipants.size() - 1; i >= 0; --i) 
                        // 如果该容器是 ActivityRecord，而且已经设置了 VisibleRequested
                        // 那么这里就告诉 sf 显示该图层
                        final ActivityRecord ar = mParticipants.valueAt(i).asActivityRecord()
                        if (ar == null || !ar.isVisibleRequested()) continue;
                        SurfaceControl.Transaction.show(ar.getSurfaceControl())
```

getTransitMode 方法会根据当前的 mVisibleRequested 和 mExistenceChanged 情况类决定是执行什么类型的动画。    

```
        int getTransitMode(@NonNull WindowContainer wc) {
            if ((mFlags & ChangeInfo.FLAG_ABOVE_TRANSIENT_LAUNCH) != 0) {
                return mExistenceChanged ? TRANSIT_CLOSE : TRANSIT_TO_BACK;
            }
            final boolean nowVisible = wc.isVisibleRequested();
            if (nowVisible == mVisible) {
                return TRANSIT_CHANGE;
            }
            if (mExistenceChanged) {
                return nowVisible ? TRANSIT_OPEN : TRANSIT_CLOSE;
            } else {
                return nowVisible ? TRANSIT_TO_FRONT : TRANSIT_TO_BACK;
            }
        }
```

### WMShell 播放动画

```
Transitions$TransitionPlayerImpl.onTransitionReady
    Transitions.onTransitionReady
        Transitions.dispatchReady
            Transitions.processReadyQueue
                Transitions.playTransition
                    // 由于 active.mHandler 为空
                    // 这里就调用 dispatchTransition 直接分发 Transition
                    active.mHandler = Transitions.dispatchTransition()
                        for (int i = mHandlers.size() - 1; i >= 0; --i) {
                            // 调用所有的 TransitionHandler 来直接执行动画
                            // 看被哪个消费
                            consumed = mHandlers.get(i).startAnimation()
                            // 如果被消费了，就返回当前 TransitionHandler
                            // 最终返回的是  DefaultTransitionHandler
                            if (consumed) {return mHandlers.get(i);
```


这里重点介绍一下 DefaultTransitionHandler.startAnimation 方法    
代码流程：    

```
DefaultTransitionHandler.startAnimation
    // 根据 TransitionInfo 获取动画类型
    DefaultTransitionHandler.getTransitionTypeFromInfo()
    DefaultTransitionHandler.loadAnimation
        DefaultTransitionHandler.loadAttributeAnimation
            // 加载动画资源，两个动画
            animAttr = R.styleable.WindowAnimation_taskOpenExitAnimation;
            animAttr = R.styleable.WindowAnimation_taskOpenEnterAnimation
            TransitionAnimation.loadDefaultAnimationAttr
                TransitionAnimation.loadAnimationAttr
    // 构架 Surface 动画
    DefaultTransitionHandler.buildSurfaceAnimation
        ValueAnimator.setDuration
        ValueAnimator.addUpdateListener
            // 根据动画进度更新位移
            DefaultTransitionHandler.applyTransformation
                SurfaceControl.Transaction.apply()
        va.addListener()
            onAnimationEnd
                DefaultTransitionHandler.onFinish()
    // 动画是添加一个背景图层
    DefaultTransitionHandler.addBackgroundColor()
    // 执行 startTransaction
    startTransaction.apply()
    for (int i = 0; i < animations.size(); ++i) {
        // 开始所有动画
        animations.get(i).start();
```

代码详解：     

```
// DefaultTransitionHandler.java
    @Override
    public boolean startAnimation(@NonNull IBinder transition, @NonNull TransitionInfo info,
            @NonNull SurfaceControl.Transaction startTransaction,
            @NonNull SurfaceControl.Transaction finishTransaction,
            @NonNull Transitions.TransitionFinishCallback finishCallback) {
        ......
        // 构建 animations
        final ArrayList<Animator> animations = new ArrayList<>();
        mAnimations.put(transition, animations);
        // 构造 onAnimFinish 回调
        final Runnable onAnimFinish = () -> {
            if (!animations.isEmpty()) return;
            mAnimations.remove(transition);
            finishCallback.onTransitionFinished(null /* wct */);
        };

        final List<Consumer<SurfaceControl.Transaction>> postStartTransactionCallbacks =
                new ArrayList<>();

        @ColorInt int backgroundColorForTransition = 0;
        final int wallpaperTransit = getWallpaperTransitType(info);
        boolean isDisplayRotationAnimationStarted = false;
        final boolean isDreamTransition = isDreamTransition(info);
        final boolean isOnlyTranslucent = isOnlyTranslucent(info);
        final boolean isActivityLevel = isActivityLevelOnly(info);
        // 遍历 Changes 这里一共有两个，一个是 OPEN 动画，一个是 TO_BACK 动画
        for (int i = info.getChanges().size() - 1; i >= 0; --i) {
            final TransitionInfo.Change change = info.getChanges().get(i);
            if (change.hasAllFlags(FLAG_IN_TASK_WITH_EMBEDDED_ACTIVITY
                    | FLAG_IS_BEHIND_STARTING_WINDOW)) {
                //如果嵌入的 Activity 被启动窗口覆盖，则不要对嵌入的 Activity进行动画处理。
                //非嵌入的情况仍然需要动画，因为容器仍然可以一起为起始窗口添加动画效果，例如 CLOSE 或 CHANGE 类型。
                continue;
            }
            if (change.hasFlags(TransitionInfo.FLAGS_IS_NON_APP_WINDOW)) {
                // Wallpaper, IME, and system windows don't need any default animations.
                continue;
            }
            final boolean isTask = change.getTaskInfo() != null;
            final int mode = change.getMode();
            boolean isSeamlessDisplayChange = false;

            // 省略的这部分是执行 TRANSIT_CHANGE 类型的动画，这里暂时不看

            // 如果正在播放显示旋转动画，则直接隐藏不可见的surface，而不对其进行动画处理。
            if (isDisplayRotationAnimationStarted && TransitionUtil.isClosingType(mode)) {
                startTransaction.hide(change.getLeash());
                continue;
            }

            // Don't animate anything that isn't independent.
            if (!TransitionInfo.isIndependent(change, info)) continue;
            // 根据 TransitionInfo 获取动画类型，其实也就是根据 TransitionInfo.Change 的 Mode 来转成 TransitionType
            final int type = getTransitionTypeFromInfo(info);
            // 加载动画
            Animation a = loadAnimation(type, info, change, wallpaperTransit, isDreamTransition);
            if (a != null) {
                if (isTask) {
                    final boolean isTranslucent = (change.getFlags() & FLAG_TRANSLUCENT) != 0;
                    if (!isTranslucent && TransitionUtil.isOpenOrCloseMode(mode)
                            && TransitionUtil.isOpenOrCloseMode(info.getType())
                            && wallpaperTransit == WALLPAPER_TRANSITION_NONE) {
                        // Use the overview background as the background for the animation
                        final Context uiContext = ActivityThread.currentActivityThread()
                                .getSystemUiContext();
                        backgroundColorForTransition =
                                uiContext.getColor(R.color.overview_background);
                    }
                    if (wallpaperTransit == WALLPAPER_TRANSITION_OPEN
                            && TransitionUtil.isOpeningType(info.getType())) {
                        // Need to flip the z-order of opening/closing because the WALLPAPER_OPEN
                        // always animates the closing task over the opening one while
                        // traditionally, an OPEN transition animates the opening over the closing.

                        // See Transitions#setupAnimHierarchy for details about these variables.
                        final int numChanges = info.getChanges().size();
                        final int zSplitLine = numChanges + 1;
                        if (TransitionUtil.isOpeningType(mode)) {
                            final int layer = zSplitLine - i;
                            startTransaction.setLayer(change.getLeash(), layer);
                        } else if (TransitionUtil.isClosingType(mode)) {
                            final int layer = zSplitLine + numChanges - i;
                            startTransaction.setLayer(change.getLeash(), layer);
                        }
                    } else if (isOnlyTranslucent && TransitionUtil.isOpeningType(info.getType())
                                && TransitionUtil.isClosingType(mode)) {
                        // If there is a closing translucent task in an OPENING transition, we will
                        // actually select a CLOSING animation, so move the closing task into
                        // the animating part of the z-order.

                        // See Transitions#setupAnimHierarchy for details about these variables.
                        final int numChanges = info.getChanges().size();
                        final int zSplitLine = numChanges + 1;
                        final int layer = zSplitLine + numChanges - i;
                        startTransaction.setLayer(change.getLeash(), layer);
                    }
                }

                final float cornerRadius;
                if (a.hasRoundedCorners()) {
                    final int displayId = isTask ? change.getTaskInfo().displayId
                            : info.getRoot(TransitionUtil.rootIndexFor(change, info))
                                    .getDisplayId();
                    final Context displayContext =
                            mDisplayController.getDisplayContext(displayId);
                    cornerRadius = displayContext == null ? 0
                            : ScreenDecorationsUtils.getWindowCornerRadius(displayContext);
                } else {
                    cornerRadius = 0;
                }

                backgroundColorForTransition = getTransitionBackgroundColorIfSet(info, change, a,
                        backgroundColorForTransition);

                if (!isTask && a.hasExtension()) {
                    if (!TransitionUtil.isOpeningType(mode)) {
                        // Can screenshot now (before startTransaction is applied)
                        edgeExtendWindow(change, a, startTransaction, finishTransaction);
                    } else {
                        // Need to screenshot after startTransaction is applied otherwise activity
                        // may not be visible or ready yet.
                        postStartTransactionCallbacks
                                .add(t -> edgeExtendWindow(change, a, t, finishTransaction));
                    }
                }

                final Rect clipRect = TransitionUtil.isClosingType(mode)
                        ? new Rect(mRotator.getEndBoundsInStartRotation(change))
                        : new Rect(change.getEndAbsBounds());
                clipRect.offsetTo(0, 0);

                final TransitionInfo.Root animRoot = TransitionUtil.getRootFor(change, info);
                final Point animRelOffset = new Point(
                        change.getEndAbsBounds().left - animRoot.getOffset().x,
                        change.getEndAbsBounds().top - animRoot.getOffset().y);

                if (change.getActivityComponent() != null) {
                    // For appcompat letterbox: we intentionally report the task-bounds so that we
                    // can animate as-if letterboxes are "part of" the activity. This means we can't
                    // always rely solely on endAbsBounds and need to also max with endRelOffset.
                    animRelOffset.x = Math.max(animRelOffset.x, change.getEndRelOffset().x);
                    animRelOffset.y = Math.max(animRelOffset.y, change.getEndRelOffset().y);
                }

                if (change.getActivityComponent() != null && !isActivityLevel) {
                    // At this point, this is an independent activity change in a non-activity
                    // transition. This means that an activity transition got erroneously combined
                    // with another ongoing transition. This then means that the animation root may
                    // not tightly fit the activities, so we have to put them in a separate crop.
                    final int layer = Transitions.calculateAnimLayer(change, i,
                            info.getChanges().size(), info.getType());
                    final SurfaceControl leash = new SurfaceControl.Builder()
                            .setName("Transition ActivityWrap: "
                                    + change.getActivityComponent().toShortString())
                            .setParent(animRoot.getLeash())
                            .setContainerLayer().build();
                    startTransaction.setCrop(leash, clipRect);
                    startTransaction.setPosition(leash, animRelOffset.x, animRelOffset.y);
                    startTransaction.setLayer(leash, layer);
                    startTransaction.show(leash);
                    startTransaction.reparent(change.getLeash(), leash);
                    startTransaction.setPosition(change.getLeash(), 0, 0);
                    animRelOffset.set(0, 0);
                    finishTransaction.reparent(leash, null);
                    leash.release();
                }

                buildSurfaceAnimation(animations, a, change.getLeash(), onAnimFinish,
                        mTransactionPool, mMainExecutor, animRelOffset, cornerRadius,
                        clipRect);

                final TransitionInfo.AnimationOptions options;
                if (Flags.moveAnimationOptionsToChange()) {
                    options = info.getAnimationOptions();
                } else {
                    options = change.getAnimationOptions();
                }
                if (options != null) {
                    attachThumbnail(animations, onAnimFinish, change, info.getAnimationOptions(),
                            cornerRadius);
                }
            }
        }
        // 添加背景图层 
        if (backgroundColorForTransition != 0) {
            addBackgroundColor(info, backgroundColorForTransition, startTransaction,
                    finishTransaction);
        }

        if (postStartTransactionCallbacks.size() > 0) {
            // postStartTransactionCallbacks require that the start transaction is already
            // applied to run otherwise they may result in flickers and UI inconsistencies.
            startTransaction.apply(true /* sync */);
            // startTransaction is empty now, so fill it with the edge-extension setup
            for (Consumer<SurfaceControl.Transaction> postStartTransactionCallback :
                    postStartTransactionCallbacks) {
                postStartTransactionCallback.accept(startTransaction);
            }
        }
        startTransaction.apply();

        // now start animations. they are started on another thread, so we have to post them
        // *after* applying the startTransaction
        mAnimExecutor.execute(() -> {
            for (int i = 0; i < animations.size(); ++i) {
                animations.get(i).start();
            }
        });

        mRotator.cleanUp(finishTransaction);
        TransitionMetrics.getInstance().reportAnimationStart(transition);
        // run finish now in-case there are no animations
        onAnimFinish.run();
        return true;
    }
```



