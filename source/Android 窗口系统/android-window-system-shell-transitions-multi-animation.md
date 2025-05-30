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
可以看到，目前支持并行动画的场景仅仅只有多任务动画和 Activity 级别的动画。    

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

(#1408194)    
如果一个动画的启动在正在运行的动画的 Playing 阶段，而且又不支持并行动画，那么就会执行串行动画，会在第一个动画执行完后执行后面的动画。     
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



## 其他文章

WM Shell多动画场景处理 https://blog.csdn.net/xiaoyantan/article/details/138583890
