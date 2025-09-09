---
title: Android ShellTransition 多动画执行场景
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 ShellTransition 多动画执行场景
date: 2022-11-23 10:00:00
---

## 合并动画

在这个场景中，新的动画的创建时间是在上一个动画的 Collecting 阶段。    
类似于前面的 Task 切换动画，打开应用时会创建一个  Transition(TRANSIT_OPEN) 动画，后面隐藏第一个 Activity 时需要建立 TRANSIT_CLOSE，由于 TRANSIT_OPEN 动画正在收集中，那么就会把第二个 ActivityRecord 添加到 TRANSIT_OPEN 组中去，后面会为这个容器创建 TRANSIT_CLOSE 的 Change，在第一个 Transition 动画中合并执行。     


## 并行动画

在这个场景中，新的动画的创建时间是在上一个动画的 Playing 阶段。    
比如在这样的场景中，我们在一个 Activity 中延时启动另外一个 Activity，然后打开这个 Activity，进入多任务后不松手，直到延时启动的那个 Activity 启动完成。      
由于在多任务不松手的情况下，多任务的动画就不会结束，而且由于多任务的动画收集阶段已经结束，那么后来的 Activity 切换动画不会合并到这个动画，那么就会并行执行。     
下面来看看并行动画执行场景和其他动画的不同之处。     

### 支持并行动画的场景

TransitionController.getIsIndependent 方法用来判断当前动画和正在运行的动画是否支持并行运行。    
可以看到，目前支持并行动画的场景仅仅只有多任务动画和 Activity 级别的动画。也就是说只支持多任务动画和Activity切换动画同时进行。     

```
// TransitionController.java
    static boolean getIsIndependent(Transition running, Transition incoming) {
        // For tests
        if (running.mParallelCollectType == Transition.PARALLEL_TYPE_MUTUAL
                && incoming.mParallelCollectType == Transition.PARALLEL_TYPE_MUTUAL) {
            return true;
        }
        // For now there's only one mutually-independent pair: an all activity-level transition and
        // a transient-launch where none of the activities are part of the transient-launch task,
        // so the following logic is hard-coded specifically for this.
        // Also, we currently restrict valid transient-launches to just recents.
        final Transition recents;
        final Transition other;
        if (running.mParallelCollectType == Transition.PARALLEL_TYPE_RECENTS
                && running.hasTransientLaunch()) {
            if (incoming.mParallelCollectType == Transition.PARALLEL_TYPE_RECENTS) {
                // Recents can't be independent from itself.
                return false;
            }
            recents = running;
            other = incoming;
        } else if (incoming.mParallelCollectType == Transition.PARALLEL_TYPE_RECENTS
                && incoming.hasTransientLaunch()) {
            recents = incoming;
            other = running;
        } else {
            return false;
        }
        // Check against *targets* because that is the post-promotion set of containers that are
        // actually animating.
        for (int i = 0; i < other.mTargets.size(); ++i) {
            final WindowContainer wc = other.mTargets.get(i).mContainer;
            final ActivityRecord ar = wc.asActivityRecord();
            if (ar == null && wc.asWindowState() == null && wc.asWindowToken() == null) {
                // Is task or above, so for now don't let them be independent.
                return false;
            }
            if (ar != null && recents.isTransientLaunch(ar)) {
                // Change overlaps with recents, so serialize.
                return false;
            }
        }
        return true;
    }
```

### 创建 Track

并行动画的执行就需要新建 Track 来执行，这样和正在执行的动画在不同的 Track 中。     

```
//TransitionController.java
    void assignTrack(Transition transition, TransitionInfo info) {
        int track = -1;
        boolean sync = false;
        for (int i = 0; i < mPlayingTransitions.size(); ++i) {
            // ignore ourself obviously
            if (mPlayingTransitions.get(i) == transition) continue;
            // 这里判断是否可以并行执行
            if (getIsIndependent(mPlayingTransitions.get(i), transition)) continue;
            if (track >= 0) {
                // 表示在前面的循环中，已经判定动画不能并行执行，那么这里就退出循环
                sync = true;
                break;
            }
            // 如果不能并行执行就获取正在执行的动画的 TrackID
            track = mPlayingTransitions.get(i).mAnimationTrack;
        }
        if (sync) {
            track = 0;
        }
        if (track < 0) {
            // 由于当前有正在执行的动画，因此 mTrackCount 不为 0，那么分配的 track 就会大于0
            track = mTrackCount;
            if (track > 0) {
                ProtoLog.v(ProtoLogGroup.WM_DEBUG_WINDOW_TRANSITIONS, "Playing #%d in parallel on "
                        + "track #%d", transition.getSyncId(), track);
            }
        }
        // 为 Transition 设置 TrackID
        transition.mAnimationTrack = track;
        // 为 TransitionInfo 设置 TrackID
        info.setTrack(track);
        mTrackCount = Math.max(mTrackCount, track + 1);
        ....
    }
```

```
// Transitions.java
    boolean dispatchReady(ActiveTransition active) {
        final TransitionInfo info = active.mInfo;

        ......

        // 并行动画场景中，由于 trackId >= mTracks.size()，因此会新创建一个 Track
        final Track track = getOrCreateTrack(info.getTrack());
        track.mReadyTransitions.add(active);

        ......
        // 播放 Track 中的动画
        processReadyQueue(track);
        return true;
    }
```


```
    private Track getOrCreateTrack(int trackId) {
        while (trackId >= mTracks.size()) {
            mTracks.add(new Track());
        }
        return mTracks.get(trackId);
    }
```

## 串行动画

### 第一个动画的 Handler 决定 merge 策略

(#1408194)    
如果一个动画的启动在正在运行的动画的 Playing 阶段，而且又不支持并行动画，那么就会执行串行动画，会在第一个动画执行完后执行后面的动画。     
merge的策略由当前的 Handler 来定，一般的 merge 策略是立即结束当前动画，从而可以立即开始等待的动画。        
这里我们构造一个场景，在退出分屏动画时，启动一个Activity，并且，这里延长一下分屏动画的执行时长，确保 TRANSIT_OPEN 动画来的时候分屏动画完成了收集而且正在执行动画。      
这样第二个动画来时就会创建新的 Transition，由于在这种场景中，不符合并行动画场景，那么就会合并动画，第二个动画会等待第一个动画完成后再去执行。     
实现：由于退出分屏会执行 Activity 的 onConfigurationChanged ，可以在 onConfigurationChanged 中启动另外一个 Activity。    

```
Transitions.TransitionPlayerImpl.onTransitionReady()
    Transitions.onTransitionReady
        Transitions.dispatchReady()
            // 分配一个已有 Track
            track = getOrCreateTrack(info.getTrack())
            // 先加入到 mReadyTransitions 队列
            track.mReadyTransitions.add(active)
            Transitions.processReadyQueue(track)
                // 表示当前没有执行的动画，这时会直接执行，这个场景不会走这里
                if (track.mActiveTransition == null) {
                    Transitions.playTransition()
                        // 执行动画
                        TransitionHandler.startAnimation
                // 当前有正在执行的动画，就会执行 merge，执行串行动画
                else
                TransitionHandler(StageCoordinator).mergeAnimation()
                    SplitScreenTransitions.mergeAnimation
```

退出分屏动画结束时执行Activity切换动画

```
SplitDecorManager$3.onAnimationEnd
    SplitScreenTransitions.onFinish
        Transitions.TransitionFinishCallback.onTransitionFinished
            // 在Transitions.playTransition startAnimation 时构造的回调
            Transitions.onFinish
                // 由于 mReadyTransitions 队列不为空，不会返回
                // 再次执行一次 processReadyQueue
                Transitions.processReadyQueue
                    Transitions.playTransition
                        Transitions.dispatchTransition
                            DefaultTransitionHandler.startAnimation
```

### WMShell 取消第二个动画

在 `Transitions.dispatchReady()` 方法中在某些条件下会调用 `onAbort(active)` 方法。       
在这个方法中，会设置 mAborted 为true，然后在 processReadyQueue 方法中如果当前有正在执行的动画，那么就执行 `onMerged()` 方法进行合并。      

```
    private void onAbort(ActiveTransition transition) {
        final Track track = mTracks.get(transition.getTrack());
        transition.mAborted = true;

        ......
        processReadyQueue(track);
    }
```
```
    void processReadyQueue(Track track) {
        ......
        // An existing animation is playing, so see if we can merge.
        final ActiveTransition playing = track.mActiveTransition;
        if (ready.mAborted) {
            // record as merged since it is no-op. Calls back into processReadyQueue
            onMerged(playing, ready);
            return;
        }
        ......
    }
```

onMerged 方法会把当前的 ActiveTransition 保存在正在执行的动画的 mMerged 数组中。

```
    private void onMerged(@NonNull ActiveTransition playing, @NonNull ActiveTransition merged) {
        ......
        track.mReadyTransitions.remove(readyIdx);
        if (playing.mMerged == null) {
            playing.mMerged = new ArrayList<>();
        }
        playing.mMerged.add(merged);
        // if it was aborted, then onConsumed has already been reported.
        if (merged.mHandler != null && !merged.mAborted) {
            merged.mHandler.onTransitionConsumed(merged.mToken, false /* abort */, merged.mFinishT);
        }
        for (int i = 0; i < mObservers.size(); ++i) {
            mObservers.get(i).onTransitionMerged(merged.mToken, playing.mToken);
        }
        mTransitionTracer.logMerged(merged.mInfo.getDebugId(), playing.mInfo.getDebugId());
        // See if we should merge another transition.
        processReadyQueue(track);
    }
```

在第一个动画执行 onFinish 时，会把第二个动画的 mStartT 和 mFinishT Transaction merge 到第一个动画的 mFinishT 中一起执行。    
然后还会通知被取消的第二个动画执行 finishTransition。      
因此这两个动画的 finishTransition 基本是同步执行的。    

```
    private void onFinish(IBinder token,
            @Nullable WindowContainerTransaction wct) {
        ......

        // Merge all associated transactions together
        SurfaceControl.Transaction fullFinish = active.mFinishT;
        if (active.mMerged != null) {
            for (int iM = 0; iM < active.mMerged.size(); ++iM) {
                final ActiveTransition toMerge = active.mMerged.get(iM);
                // Include start. It will be a no-op if it was already applied. Otherwise, we need
                // it to maintain consistent state.
                if (toMerge.mStartT != null) {
                    if (fullFinish == null) {
                        fullFinish = toMerge.mStartT;
                    } else {
                        fullFinish.merge(toMerge.mStartT);
                    }
                }
                if (toMerge.mFinishT != null) {
                    if (fullFinish == null) {
                        fullFinish = toMerge.mFinishT;
                    } else {
                        fullFinish.merge(toMerge.mFinishT);
                    }
                }
            }
        }
        if (fullFinish != null) {
            fullFinish.apply();
        }
        // Now perform all the finish callbacks (starting with the playing one and then all the
        // transitions merged into it).
        releaseSurfaces(active.mInfo);
        mOrganizer.finishTransition(active.mToken, wct);
        if (active.mMerged != null) {
            for (int iM = 0; iM < active.mMerged.size(); ++iM) {
                ActiveTransition merged = active.mMerged.get(iM);
                mOrganizer.finishTransition(merged.mToken, null /* wct */);
                releaseSurfaces(merged.mInfo);
                mKnownTransitions.remove(merged.mToken);
            }
            active.mMerged.clear();
        }
        ......
    }
```

什么场景下会执行这样的merge操作呢？      
第一个场景是没有 Transition Root。      
第二个场景是动画中没有 Task 相关动画，而且这动画有 TransitionInfo.Change 标记了 FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT 时。      


```
    boolean dispatchReady(ActiveTransition active) {
    
        if (info.getRootCount() == 0 && !KeyguardTransitionHandler.handles(info)) {
            ...
            onAbort(active);
            return true;
        }
        ....
        final int changeSize = info.getChanges().size();
        boolean taskChange = false;
        boolean transferStartingWindow = false;
        int animBehindStartingWindow = 0;
        boolean allOccluded = changeSize > 0;
        for (int i = changeSize - 1; i >= 0; --i) {
            final TransitionInfo.Change change = info.getChanges().get(i);
            // 是否有 Task 切换
            taskChange |= change.getTaskInfo() != null;
            // 是否有 FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT
            transferStartingWindow |= change.hasFlags(FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT);
            if (change.hasAllFlags(FLAG_IS_BEHIND_STARTING_WINDOW | FLAG_NO_ANIMATION)
                    || change.hasAllFlags(
                            FLAG_IS_BEHIND_STARTING_WINDOW | FLAG_IN_TASK_WITH_EMBEDDED_ACTIVITY)) {
                animBehindStartingWindow++;
            }
            if (!change.hasFlags(FLAG_IS_OCCLUDED)) {
                allOccluded = false;
            } else if (change.hasAllFlags(TransitionInfo.FLAGS_IS_OCCLUDED_NO_ANIMATION)) {
                // Remove the change because it should be invisible in the animation.
                info.getChanges().remove(i);
                continue;
            }
            // The change has already animated by back gesture, don't need to play transition
            // animation on it.
            if (change.hasFlags(FLAG_BACK_GESTURE_ANIMATED)) {
                info.getChanges().remove(i);
            }
        }
        ....
        if (!taskChange && (transferStartingWindow || animBehindStartingWindow == changeSize)
                && changeSize >= 1
                // B. It's visibility change if the TRANSIT_TO_BACK/TO_FRONT happened when all
                // changes are underneath another change.
                || ((info.getType() == TRANSIT_TO_BACK || info.getType() == TRANSIT_TO_FRONT)
                && allOccluded)) {
            ....
            onAbort(active);
            return true;
        }
```

第一个要求是动画中没有 Task 相关动画，这个比较好理解，因为如果是 Task 内的Activity切换动画就满足，那么第二个 FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT 这个 Flag 是什么时候设置的呢？      
在第二个Activity添加 StartingWindow 会进行 transferStartingWindow() 判断，fromActivity 参数是启动它的那个 Activity。      

```
// ActivityRecord.java
    private boolean transferStartingWindow(@NonNull ActivityRecord fromActivity) {
        final WindowState tStartingWindow = fromActivity.mStartingWindow;
        if (tStartingWindow != null && fromActivity.mStartingSurface != null) {
            if (tStartingWindow.getParent() == null) {
                // 如果第一个 Activity 的 StartingWindow 已经 deattach 那么这里返回false
                return false;
            }
            ......

                if (fromActivity.isAnimating()) {
                    transferAnimation(fromActivity);

                    // When transferring an animation, we no longer need to apply an animation to
                    // the token we transfer the animation over. Thus, set this flag to indicate
                    // we've transferred the animation.
                    mTransitionChangeFlags |= FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT;
                } else if (mTransitionController.getTransitionPlayer() != null) {
                    // 使用 Shell Trasition 动画 ，添加 FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT
                    mTransitionChangeFlags |= FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT;
                }
                //////
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return true;
        } else if (fromActivity.mStartingData != null) {
            // The previous app was getting ready to show a
            // starting window, but hasn't yet done so.  Steal it!
            ProtoLog.v(WM_DEBUG_STARTING_WINDOW,
                    "Moving pending starting from %s to %s", fromActivity, this);
            mStartingData = fromActivity.mStartingData;
            fromActivity.mStartingData = null;
            fromActivity.startingMoved = true;
            scheduleAddStartingWindow();
            return true;
        }

        // TODO: Transfer thumbnail

        return false;
    }
```
## 其他文章

WM Shell多动画场景处理 https://blog.csdn.net/xiaoyantan/article/details/138583890
