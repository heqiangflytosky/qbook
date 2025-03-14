---
title: Android WindowOrganizerController
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 WindowOrganizerController
date: 2022-11-23 10:00:00
---

## WindowOrganizerController

服务端用来组织管理窗口的实现。主要是对应用侧传递过来的 WindowContainerTransaction 的变更应用的对应的窗口上面。          

## applyTransaction

applyTransaction 方法主要是实施在应用侧对窗口的修改，主要分为 Change 相关、HierarchyOp 相关、setBoundsChangeTransaction 相关以及更新 Activity 可见性和 Configuration 等部分，核心是对应用侧传来的 WindowContainerTransaction 对象进行解析和应用到系统侧窗口数据。    

```
    // 
    private int applyTransaction(@NonNull WindowContainerTransaction t, int syncId,
            @Nullable Transition transition, @NonNull CallerInfo caller,
            @Nullable Transition finishTransition) {
```

applyHierarchyOp 主要处理系统侧传来的 HierarchyOp 相关变更      

```
    private int applyHierarchyOp(WindowContainerTransaction.HierarchyOp hop, int effects,
            int syncId, @Nullable Transition transition, boolean isInLockTaskMode,
            @NonNull CallerInfo caller, @Nullable IBinder errorCallbackToken,
            @Nullable ITaskFragmentOrganizer organizer, @Nullable Transition finishTransition) {
```

applyWindowContainerChange 主要处理应用侧传来的 WindowContainerTransaction.Change 变更      

```
    private int applyWindowContainerChange(WindowContainer wc,
            WindowContainerTransaction.Change c, @Nullable IBinder errorCallbackToken) {
```

WindowOrganizerController.applyTransaction() 主要做了下面几件事：      
 - 遍历 WindowContainerTransaction.Change，并应用到窗口
 - 应用 HierarchyOp相关
 - 获取 mBoundsChangeSurfaceBounds，并应用到对应窗口上
 - 更新 Activity 的可见性和 Configuration

```
    private int applyTransaction(@NonNull WindowContainerTransaction t, int syncId,
            @Nullable Transition transition, @NonNull CallerInfo caller,
            @Nullable Transition finishTransition) {
        int effects = TRANSACT_EFFECTS_NONE;
        ProtoLog.v(WM_DEBUG_WINDOW_ORGANIZER, "Apply window transaction, syncId=%d", syncId);
        mService.deferWindowLayout();
        mService.mTaskSupervisor.setDeferRootVisibilityUpdate(true /* deferUpdate */);
        boolean deferTransitionReady = false;
        if (transition != null && !t.isEmpty()) {
            if (transition.isCollecting()) {
                deferTransitionReady = true;
                transition.deferTransitionReady();
            } else if (Flags.alwaysDeferTransitionWhenApplyWct()) {
                Slog.w(TAG, "Transition is not collecting when applyTransaction."
                        + " transition=" + transition + " state=" + transition.getState());
                transition = null;
            }
        }
        try {
            final ArraySet<WindowContainer<?>> haveConfigChanges = new ArraySet<>();
            if (transition != null) {
                transition.applyDisplayChangeIfNeeded(haveConfigChanges);
                if (!haveConfigChanges.isEmpty()) {
                    effects |= TRANSACT_EFFECTS_CLIENT_CONFIG;
                }
            }
            // 对 WindowContainerTransaction.Change处理
            final List<WindowContainerTransaction.HierarchyOp> hops = t.getHierarchyOps();
            final int hopSize = hops.size();
            Iterator<Map.Entry<IBinder, WindowContainerTransaction.Change>> entries;
            if (transition != null) {
                // Mark any config-at-end containers before applying config changes so that
                // the config changes don't dispatch to client.
                entries = t.getChanges().entrySet().iterator();
                while (entries.hasNext()) {
                    final Map.Entry<IBinder, WindowContainerTransaction.Change> entry =
                            entries.next();
                    if (!entry.getValue().getConfigAtTransitionEnd()) continue;
                    final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                    if (wc == null || !wc.isAttached()) continue;
                    transition.setConfigAtEnd(wc);
                }
            }
            entries = t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                // Make sure we add to the syncSet before performing
                // operations so we don't end up splitting effects between the WM
                // pending transaction and the BLASTSync transaction.
                if (syncId >= 0) {
                    addToSyncSet(syncId, wc);
                }
                if (transition != null) transition.collect(wc);

                if ((entry.getValue().getChangeMask()
                        & WindowContainerTransaction.Change.CHANGE_FORCE_NO_PIP) != 0) {
                    // Disable entering pip (eg. when recents pretends to finish itself)
                    if (finishTransition != null) {
                        finishTransition.setCanPipOnFinish(false /* canPipOnFinish */);
                    } else if (transition != null) {
                        transition.setCanPipOnFinish(false /* canPipOnFinish */);
                    }
                }
                // A bit hacky, but we need to detect "remove PiP" so that we can "wrap" the
                // setWindowingMode call in force-hidden.
                boolean forceHiddenForPip = false;
                if (wc.asTask() != null && wc.inPinnedWindowingMode()
                        && entry.getValue().getWindowingMode() != WINDOWING_MODE_PINNED) {
                    // We are going out of pip. Now search hierarchy ops to determine whether we
                    // are removing pip or expanding pip.
                    for (int i = 0; i < hopSize; ++i) {
                        final WindowContainerTransaction.HierarchyOp hop = hops.get(i);
                        if (hop.getType() != HIERARCHY_OP_TYPE_REORDER) continue;
                        final WindowContainer hopWc = WindowContainer.fromBinder(
                                hop.getContainer());
                        if (!wc.equals(hopWc)) continue;
                        forceHiddenForPip = !hop.getToTop();
                    }
                }
                if (forceHiddenForPip) {
                    wc.asTask().setForceHidden(FLAG_FORCE_HIDDEN_FOR_PINNED_TASK, true /* set */);
                    // When removing pip, make sure that onStop is sent to the app ahead of
                    // onPictureInPictureModeChanged.
                    // See also PinnedStackTests#testStopBeforeMultiWindowCallbacksOnDismiss
                    wc.asTask().ensureActivitiesVisible(null /* starting */);
                    wc.asTask().mTaskSupervisor.processStoppingAndFinishingActivities(
                            null /* launchedActivity */, false /* processPausingActivities */,
                            "force-stop-on-removing-pip");
                }
                // 获取一个的change，取出change中包裹的具体WindowContainer，即上屏和下屏的对应Task，
                // 然后调用applyWindowContainerChange进行对应处理
                int containerEffect = applyWindowContainerChange(wc, entry.getValue(),
                        t.getErrorCallbackToken());
                effects |= containerEffect;

                if (forceHiddenForPip) {
                    wc.asTask().setForceHidden(FLAG_FORCE_HIDDEN_FOR_PINNED_TASK, false /* set */);
                }

                // Lifecycle changes will trigger ensureConfig for everything.
                if ((effects & TRANSACT_EFFECTS_LIFECYCLE) == 0
                        && (containerEffect & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                    haveConfigChanges.add(wc);
                }
            }
            // Hierarchy changes
            // 对 WindowContainerTransaction.HierarchyOp 处理
            if (hopSize > 0) {
                final boolean isInLockTaskMode = mService.isInLockTaskMode();
                for (int i = 0; i < hopSize; ++i) {
                    effects |= applyHierarchyOp(hops.get(i), effects, syncId, transition,
                            isInLockTaskMode, caller, t.getErrorCallbackToken(),
                            t.getTaskFragmentOrganizer(), finishTransition);
                }
            }
            // Queue-up bounds-change transactions for tasks which are now organized. Do
            // this after hierarchy ops so we have the final organized state.
            // Change.mBoundsChangeSurfaceBounds处理
            entries = t.getChanges().entrySet().iterator();
            while (entries.hasNext()) {
                final Map.Entry<IBinder, WindowContainerTransaction.Change> entry = entries.next();
                final WindowContainer wc = WindowContainer.fromBinder(entry.getKey());
                if (wc == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to operate on detached container: " + wc);
                    continue;
                }
                final Task task = wc.asTask();
                final Rect surfaceBounds = entry.getValue().getBoundsChangeSurfaceBounds();
                if (task == null || !task.isAttached() || surfaceBounds == null) {
                    continue;
                }
                if (!task.isOrganized()) {
                    final Task parent = task.getParent() != null ? task.getParent().asTask() : null;
                    // Also allow direct children of created-by-organizer tasks to be
                    // controlled. In the future, these will become organized anyways.
                    if (parent == null || !parent.mCreatedByOrganizer) {
                        throw new IllegalArgumentException(
                                "Can't manipulate non-organized task surface " + task);
                    }
                }
                final SurfaceControl.Transaction sft = new SurfaceControl.Transaction();
                final SurfaceControl sc = task.getSurfaceControl();
                sft.setPosition(sc, surfaceBounds.left, surfaceBounds.top);
                if (surfaceBounds.isEmpty()) {
                    sft.setWindowCrop(sc, null);
                } else {
                    sft.setWindowCrop(sc, surfaceBounds.width(), surfaceBounds.height());
                }
                task.setMainWindowSizeChangeTransaction(sft);
            }
            //5.更新Activity的可见性和Configuration
            if ((effects & TRANSACT_EFFECTS_LIFECYCLE) != 0) {
                mService.mTaskSupervisor.setDeferRootVisibilityUpdate(false /* deferUpdate */);
                // Already calls ensureActivityConfig
                mService.mRootWindowContainer.ensureActivitiesVisible();
                mService.mRootWindowContainer.resumeFocusedTasksTopActivities();
            } else if ((effects & TRANSACT_EFFECTS_CLIENT_CONFIG) != 0) {
                for (int i = haveConfigChanges.size() - 1; i >= 0; --i) {
                    haveConfigChanges.valueAt(i).forAllActivities(r -> {
                        if (r.isVisibleRequested()) {
                            r.ensureActivityConfiguration(true /* ignoreVisibility */);
                        }
                    });
                }
            }

            if (effects != 0) {
                mService.mWindowManager.mWindowPlacerLocked.requestTraversal();
            }
        } finally {
            if (deferTransitionReady) {
                transition.continueTransitionReady();
            }
            mService.mTaskSupervisor.setDeferRootVisibilityUpdate(false /* deferUpdate */);
            // 应用属性的变更
            mService.continueWindowLayout();
        }
        return effects;
    }

```


## 对 Change 的处理

```
    private int applyWindowContainerChange(WindowContainer wc,
            WindowContainerTransaction.Change c, @Nullable IBinder errorCallbackToken) {
        // 确认容器类型
        sanitizeWindowContainer(wc);
        //根据不同的容器类型进行不同的操作
        if (wc.asDisplayArea() != null) {
            return applyDisplayAreaChanges(wc.asDisplayArea(), c);
        } else if (wc.asTask() != null) {
            return applyTaskChanges(wc.asTask(), c);
        } else if (wc.asTaskFragment() != null && wc.asTaskFragment().isEmbedded()) {
            return applyTaskFragmentChanges(wc.asTaskFragment(), c, errorCallbackToken);
        } else {
            return applyChanges(wc, c);
        }
    }
```

applyDisplayAreaChanges()方法是对DisplayArea进行修改，对设置了 CHANGE_IGNORE_ORIENTATION_REQUEST 和 CHANGE_HIDDEN 变更的容器进行相关配置的修改。    

```
    private int applyDisplayAreaChanges(DisplayArea displayArea,
            WindowContainerTransaction.Change c) {
        final int[] effects = new int[1];
        // 应用 Configuration相关Changes。
        effects[0] = applyChanges(displayArea, c);

        //调用displayArea的setIgnoreOrientationRequest方法
        if ((c.getChangeMask()
                & WindowContainerTransaction.Change.CHANGE_IGNORE_ORIENTATION_REQUEST) != 0) {
            if (displayArea.setIgnoreOrientationRequest(c.getIgnoreOrientationRequest())) {
                effects[0] |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }
        //遍历当前displayArea下的所有task，执行回调方法
        displayArea.forAllTasks(task -> {
            Task tr = (Task) task;
            if ((c.getChangeMask() & WindowContainerTransaction.Change.CHANGE_HIDDEN) != 0) {
                //通过task调用setForceHidden方法强制隐藏task
                if (tr.setForceHidden(FLAG_FORCE_HIDDEN_FOR_TASK_ORG, c.getHidden())) {
                    effects[0] |= TRANSACT_EFFECTS_LIFECYCLE;
                }
            }
        });

        return effects[0];
    }
```

applyTaskChanges() 方法针对 Task 容器的属性进行修改。主要针对 Change.CHANGE_HIDDEN、CHANGE_FORCE_TRANSLUCENT、CHANGE_DRAG_RESIZING、处理画中画模式。    

```
    private int applyTaskChanges(Task tr, WindowContainerTransaction.Change c) {
        final SurfaceControl.Transaction t = c.getBoundsChangeTransaction();
        ......
        if (t != null) {
            tr.setMainWindowSizeChangeTransaction(t);
        }
        // 应用 Configuration相关Changes。
        int effects = applyChanges(tr, c);
        if ((c.getChangeMask() & WindowContainerTransaction.Change.CHANGE_HIDDEN) != 0) {
            if (tr.setForceHidden(FLAG_FORCE_HIDDEN_FOR_TASK_ORG, c.getHidden())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }

        if ((c.getChangeMask() & CHANGE_FORCE_TRANSLUCENT) != 0) {
            if (tr.setForceTranslucent(c.getForceTranslucent())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }

        if ((c.getChangeMask() & WindowContainerTransaction.Change.CHANGE_DRAG_RESIZING) != 0) {
            tr.setDragResizing(c.getDragResizing());
        }

        final int childWindowingMode = c.getActivityWindowingMode();
        if (!ActivityTaskManagerService.isPip2ExperimentEnabled()
                && tr.getWindowingMode() == WINDOWING_MODE_PINNED
                && (childWindowingMode == WINDOWING_MODE_PINNED
                || childWindowingMode == WINDOWING_MODE_UNDEFINED)) {
            ......
            mService.mRootWindowContainer.removeAllMaybeAbortPipEnterRunnable();
        }
        if (childWindowingMode > -1) {
            tr.forAllActivities(a -> { a.setWindowingMode(childWindowingMode); });
        }
        //设置PIP相关bounds
        Rect enterPipBounds = c.getEnterPipBounds();
        if (enterPipBounds != null) {
            tr.mDisplayContent.mPinnedTaskController.setEnterPipBounds(enterPipBounds);
        }
        // 小窗模式
        if (c.getWindowingMode() == WINDOWING_MODE_PINNED
                && !tr.inPinnedWindowingMode()) {
            final ActivityRecord activity = tr.getTopNonFinishingActivity();
            if (activity != null) {
                final boolean lastSupportsEnterPipOnTaskSwitch =
                        activity.supportsEnterPipOnTaskSwitch;
                // Temporarily force enable enter PIP on task switch so that PIP is requested
                // regardless of whether the activity is resumed or paused.
                activity.supportsEnterPipOnTaskSwitch = true;
                boolean canEnterPip = activity.checkEnterPictureInPictureState(
                        "applyTaskChanges", true /* beforeStopping */);
                if (canEnterPip) {
                    canEnterPip = mService.mActivityClientController
                            .requestPictureInPictureMode(activity);
                }
                if (!canEnterPip) {
                    // Restore the flag to its previous state when the activity cannot enter PIP.
                    activity.supportsEnterPipOnTaskSwitch = lastSupportsEnterPipOnTaskSwitch;
                }
            }
        }

        return effects;
    }
```

applyChanges 方法Change中的保存的configurations修改和焦点的修改，以及windowingMode的修改。    

```
    private int applyChanges(@NonNull WindowContainer<?> container,
            @NonNull WindowContainerTransaction.Change change) {
        //判断Change中包含哪些configurations相关修改
        final int configMask = change.getConfigSetMask() & CONTROLLABLE_CONFIGS;
        //判断Change中包含哪些Window相关修改
        final int windowMask = change.getWindowSetMask() & CONTROLLABLE_WINDOW_CONFIGS;
        int effects = TRANSACT_EFFECTS_NONE;
        //获取窗口模式
        final int windowingMode = change.getWindowingMode();
        if (configMask != 0) {
            if (windowingMode > -1 && windowingMode != container.getWindowingMode()) {
                //请求一个Configuration，用于设置新的Configuration
                final Configuration c = container.getRequestedOverrideConfiguration();
                // 设置相关属性
                c.setTo(change.getConfiguration(), configMask, windowMask);
                // 这里不调用 onRequestedOverrideConfigurationChanged 是因为在后面设置窗口模式时
                // container.setWindowingMode(windowingMode) 会调用 onRequestedOverrideConfigurationChanged
                // 方法去更新configuration相关操作。无需重复调用
            } else {
                //windowingMode无效或未发生改变的场景
                final Configuration c =
                        new Configuration(container.getRequestedOverrideConfiguration());
                c.setTo(change.getConfiguration(), configMask, windowMask);
                //更新Configuration
                container.onRequestedOverrideConfigurationChanged(c);
            }
            effects |= TRANSACT_EFFECTS_CLIENT_CONFIG;
            if (windowMask != 0 && container.isEmbedded()) {
                // Changing bounds of the embedded TaskFragments may result in lifecycle changes.
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }
        if ((change.getChangeMask() & WindowContainerTransaction.Change.CHANGE_FOCUSABLE) != 0) {
            if (container.setFocusable(change.getFocusable())) {
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }

        if (windowingMode > -1) {
            if (mService.isInLockTaskMode()
                    && WindowConfiguration.inMultiWindowMode(windowingMode)) {
                Slog.w(TAG, "Dropping unsupported request to set multi-window windowing mode"
                        + " during locked task mode.");
                return effects;
            }

            if (windowingMode == WINDOWING_MODE_PINNED) {
                // Do not directly put the container into PINNED mode as it may not support it or
                // the app may not want to enter it. Instead, send a signal to request PIP
                // mode to the app if they wish to support it below in #applyTaskChanges.
                return effects;
            }

            final int prevMode = container.getRequestedOverrideWindowingMode();
            //设置新窗口模式
            container.setWindowingMode(windowingMode);
            if (prevMode != container.getWindowingMode()) {
                // The activity in the container may become focusable or non-focusable due to
                // windowing modes changes (such as entering or leaving pinned windowing mode),
                // so also apply the lifecycle effects to this transaction.
                effects |= TRANSACT_EFFECTS_LIFECYCLE;
            }
        }
        return effects;
    }

```

## 对 HierarchyOp 的处理

applyHierarchyOp() 方法主要是针对 HierarchyOp 变更的处理，这个方法比较长，里面主要是针对 HierarchyOp 各个属性变更分别调用对应的方法去处理，这里我们只挑选加个进行介绍。    

```
    private int applyHierarchyOp(WindowContainerTransaction.HierarchyOp hop, int effects,
            int syncId, @Nullable Transition transition, boolean isInLockTaskMode,
            @NonNull CallerInfo caller, @Nullable IBinder errorCallbackToken,
            @Nullable ITaskFragmentOrganizer organizer, @Nullable Transition finishTransition) {
        final int type = hop.getType();
        switch (type) {
            case HIERARCHY_OP_TYPE_REMOVE_TASK: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                if (wc == null || wc.asTask() == null || !wc.isAttached()) {
                    Slog.e(TAG, "Attempt to remove invalid task: " + wc);
                    break;
                }
                final Task task = wc.asTask();
                if (task.isVisibleRequested() || task.isVisible()) {
                    effects |= TRANSACT_EFFECTS_LIFECYCLE;
                }
                if (task.isLeafTask()) {
                    mService.mTaskSupervisor
                            .removeTask(task, true, REMOVE_FROM_RECENTS, "remove-task"
                                    + "-through-hierarchyOp");
                } else {
                    mService.mTaskSupervisor.removeRootTask(task);
                }
                break;
            }
        ......
            case HIERARCHY_OP_TYPE_LAUNCH_TASK: {
                mService.mAmInternal.enforceCallingPermission(START_TASKS_FROM_RECENTS,
                        "launchTask HierarchyOp");
                final Bundle launchOpts = hop.getLaunchOptions();
                final int taskId = launchOpts.getInt(
                        WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                launchOpts.remove(WindowContainerTransaction.HierarchyOp.LAUNCH_KEY_TASK_ID);
                final SafeActivityOptions safeOptions =
                        SafeActivityOptions.fromBundle(launchOpts, caller.mPid, caller.mUid);
                waitAsyncStart(() -> mService.mTaskSupervisor.startActivityFromRecents(
                        caller.mPid, caller.mUid, taskId, safeOptions));
                break;
            }
            case HIERARCHY_OP_TYPE_REORDER:
            case HIERARCHY_OP_TYPE_REPARENT: {
                final WindowContainer wc = WindowContainer.fromBinder(hop.getContainer());
                ......
                effects |= sanitizeAndApplyHierarchyOp(wc, hop);
                break;
            }
```

sanitizeAndApplyHierarchyOp 分别对 reorder 和 reparent 进行操作。     

```
    private int sanitizeAndApplyHierarchyOp(WindowContainer container,
            WindowContainerTransaction.HierarchyOp hop) {
        final Task task = container.asTask();
        ......
        final Task as = task;

        if (hop.isReparent()) {
            final boolean isNonOrganizedRootableTask =
                    task.isRootTask() || task.getParent().asTask().mCreatedByOrganizer;
            if (isNonOrganizedRootableTask) {
                ......
                if (task.getParent() != newParent) {
                    if (newParent.asTaskDisplayArea() != null) {
                        // For now, reparenting to displayarea is different from other reparents...
                        as.reparent(newParent.asTaskDisplayArea(), hop.getToTop());
                    } else if (newParent.asTask() != null) {
                        ......
                        task.reparent((Task) newParent,
                                hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                                false /*moveParents*/, "sanitizeAndApplyHierarchyOp");
                    } else {
                        throw new RuntimeException("Can only reparent task to another task or"
                                + " taskDisplayArea, but not " + newParent);
                    }
                } else {
                    ......
                    as.getDisplayArea().positionChildAt(
                            hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM, rootTask,
                            false /* includingParents */);
                }
            } else {
                throw new RuntimeException("Reparenting leaf Tasks is not supported now. " + task);
            }
        } else {
            if (hop.getToTop() && task.isRootTask()) {
                final ActivityRecord pipCandidate = task.findEnterPipOnTaskSwitchCandidate(
                        task.getDisplayArea().getTopRootTask());
                task.enableEnterPipOnTaskSwitch(pipCandidate, task, null /* toFrontActivity */,
                        null /* options */);
            }

            task.getParent().positionChildAt(
                    hop.getToTop() ? POSITION_TOP : POSITION_BOTTOM,
                    task, hop.includingParents());
        }
        return TRANSACT_EFFECTS_LIFECYCLE;
    }
```

## 相关文章

[Android U system_server侧WindowContainerTransaction处理](https://blog.csdn.net/yimelancholy/article/details/144694952)      
[WCT系列（一）：WindowContainerTransaction类详解 ](https://juejin.cn/post/7408855714700361766)      
