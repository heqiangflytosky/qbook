---
title: Android 基于的Choreographer渲染机制
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍基于的Choreographer渲染机制
date: 2015-1-16 10:00:00
---


## 简介

Choreographer 也可以成为“编舞者”，主要是配合 Vsync ，给上层 App 的输入、动画、渲染提供一个稳定的处理时机，也就是 Vsync 到来的时候 ，系统通过对 Vsync 信号周期的调整，来控制每一帧绘制操作的时机。    
了解 Choreographer 还可以帮助 App 开发者知道程序每一帧运行的基本原理。      

## input 与 Choreographer

Choreographer.CALLBACK_INPUT 是在 doFrame() 方法中第一个执行的 callback。     
主要用来消费那些被“批处理”机制暂存起来的输入事件。对于事件的批处理，请参考input相关的博客。     
比如 Down 事件不用批处理，就直接进行分发。直接走 InputEventReceiver.dispatchInputEvent 分发，最终会直接走到 ViewRootImpl 的事件分发流程中。               

 - 事件到达，暂不处理：当应用端的 InputEventReceiver 从 InputDispatcher 收到一个可用的“批处理”事件（如 ACTION_MOVE）时，它不会立刻分发。而是会通过 Choreographer 请求一个 CALLBACK_INPUT 回调，安排在下一个VSYSC到来时执行。      
 - VSYNC触发，开始消费：下一个VSYNC信号到来，Choreographer 执行 doCallbacks(Choreographer.CALLBACK_INPUT, ...)。     
 - 一次性取出所有事件：在这个回调中，系统会一次性从Socket中取出所有在上一段时间内积累的输入事件（包括最新的和历史点）。       
 - 分发给应用：这些事件被整合成一个或多个 MotionEvent 对象，然后才真正分发给 View 树进行处理。        

```
MessageQueue.nativePollOnce
  ViewRootImpl$WindowInputEventReceiver.onBatchedInputEventPending
    ViewRootImpl.scheduleConsumeBatchedInput
      Choreographer.postCallback(Choreographer.CALLBACK_INPUT,mConsumedBatchedInputRunnable)
        Choreographer.postCallbackDelayed
          Choreographer.postCallbackDelayedInternal
            mCallbackQueues[callbackType].addCallbackLocked
            Choreographer.scheduleFrameLocked
              Choreographer.scheduleVsyncLocked
                DisplayEventReceiver.scheduleVsync()
```

收到 Vsync 事件：     

```
Choreographer$FrameDisplayEventReceiver.run
  Choreographer.doFrame
    Choreographer.doCallbacks(Choreographer.CALLBACK_INPUT)
      Choreographer$CallbackRecord.run
        ViewRootImpl$ConsumeBatchedInputRunnable.run
          ViewRootImpl.doConsumeBatchedInput
            InputEventReceiver.consumeBatchedInputEvents
              InputEventReceiver.nativeConsumeBatchedInputEvents
              //----JNI
                NativeInputEventReceiver::consumeEvents()
                  env->CallVoidMethod(gInputEventReceiverClassInfo.dispatchInputEvent)
                  //---->Java
                    InputEventReceiver.dispatchInputEvent
                      ViewRootImpl$WindowInputEventReceiver.onInputEvent
                        ....
                        // View 事件分发
                        View.dispatchPointerEvent
```


## 动画与 Choreographer

从一个alpha动画来看Choreographer的执行路程。      


普通的alpha动画。    

```
        ValueAnimator animator = ValueAnimator.ofFloat(1.0f, 0.0f);
        animator.setDuration(5000);
        animator.setInterpolator(new LinearInterpolator());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                float animatedValue = (float) valueAnimator.getAnimatedValue();
                binding.blueView.setAlpha(animatedValue);
            }
        });
        animator.start();
```

start 方法中调用了 addAnimationCallback 方法。    
post callback:    

```
ValueAnimator.start
  ValueAnimator.addAnimationCallback
    AnimationHandler.addAnimationFrameCallback
      AnimationHandler$MyFrameCallbackProvider.postFrameCallback
        Choreographer.postFrameCallback
          Choreographer.postFrameCallbackDelayed
            // post 了一个 animation callback
            Choreographer.postCallbackDelayedInternal(CALLBACK_ANIMATION)
              Choreographer.scheduleVsyncLocked()
                // 请求 Vsync
                DisplayEventReceiver.scheduleVsync()
      AnimationHandler.mAnimationCallbacks.add(AnimationFrameCallback)
```


Vsync 到来时执行 callback，一直到动画执行完毕，remove AnimationFrameCallback，如果没有其他更新，就停止申请 Vsync。       

```
Handler.onVsync
Handler.handleCallback
  Choreographer$FrameDisplayEventReceiver.run
    Choreographer.doFrame
      Choreographer.doCallbacks
        Choreographer$CallbackRecord.run
          AnimationHandler$mFrameCallback.doFrame
            // 执行 onAnimationUpdate
            AnimationHandler.doAnimationFrame
              ValueAnimator.doAnimationFrame
                boolean finished = ValueAnimator.animateBasedOnTime
                  ValueAnimator.animateValue
                    Animator.callOnList
                      AnimationActivity$1.onAnimationUpdate
                if (finished) 
                  // 结束动画，removeCallback，后面就不继续postFrameCallback
                  ValueAnimator.endAnimation
                    ValueAnimator.removeAnimationCallback
                      AnimationHandler.removeCallback
                        // 设置元素为空
                        mAnimationCallbacks.set(id, null)
              AnimationHandler.cleanUpList()
                if (mAnimationCallbacks.get(i) == null)
                  // remove callback
                  mAnimationCallbacks.remove(i)
            // 继续 postFrameCallback
            if (mAnimationCallbacks.size() > 0)
            AnimationHandler$MyFrameCallbackProvider.postFrameCallback
              Choreographer.postFrameCallback
                Choreographer.postFrameCallbackDelayed
                  Choreographer.postCallbackDelayedInternal
```

## View 的渲染流程与 Choreographer

Activity 开始显示时：        

```
ActivityThread.handleResumeActivity
  WindowManagerImpl.addView
    WindowManagerGlobal.addView
      ViewRootImpl.setView
        ViewRootImpl.requestLayout
          ViewRootImpl.scheduleTraversals
            Choreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL)
              Choreographer.postCallbackDelayed
                Choreographer.postCallbackDelayedInternal
                  Choreographer.scheduleFrameLocked
                    Choreographer.scheduleVsyncLocked
```

收到 onVsync 后， View开始绘制后，持续申请 Vsync。     


```
Handler.onVsync
Handler.handleCallback
  Choreographer$FrameDisplayEventReceiver.run
    Choreographer.doFrame
      Choreographer.doCallbacks(Choreographer.CALLBACK_TRAVERSAL)
        Choreographer$CallbackRecord.run
          ViewRootImpl$TraversalRunnable.run
            // 执行 doTraversal：measure，layout，draw
            ViewRootImpl.doTraversal
              ViewRootImpl.performTraversals
                ViewRootImpl.performDraw
                  ViewRootImpl.draw
                    ThreadedRenderer.draw
                      ......
                      View.draw
                        View.invalidate
                          ......
                          ViewRootImpl.invalidate
                            ViewRootImpl.scheduleTraversals
                              Choreographer.postCallback
                                Choreographer.postCallbackDelayed
                                  Choreographer.postCallbackDelayedInternal
                                    Choreographer.scheduleFrameLocked
                                      Choreographer.scheduleVsyncLocked
```

## scheduleFrameLocked

```
// Choreographer.java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            // USE_VSYNC 表示使用 VSYNC，为false时表示直接执行doFrame，
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }

```




## 关于插帧

```
// Choreographer.java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            // 可以在这里通过动态控制执行插帧方案
            // USE_VSYNC 表示使用 VSYNC，为false时表示直接 doFrame，
            // 高通在这里做了判断，进行列表的滑动插帧
            // if (ScrollOptimizer.shouldUseVsync(USE_VSYNC)) {
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }
```

```
Choreographer.doFrame
  ....
  RecyclerView$ViewFlinger.internalPostOnAnimation
    View.postOnAnimation
      Choreographer.postCallback
        Choreographer.postCallbackDelayed
          Choreographer.postCallbackDelayedInternal
            Choreographer.scheduleFrameLocked
              // 这里不使用 Vsync，直接发起doFrame
              mHandler.obtainMessage(MSG_DO_FRAME)
```

```
Choreographer$FrameHandler.handleMessage
  case MSG_DO_FRAME:
    Choreographer.doFrame
      Choreographer.doCallbacks(Choreographer.CALLBACK_ANIMATION)
        Choreographer$CallbackRecord.run
          AnimationHandler$1.doFrame
            AnimationHandler$MyFrameCallbackProvider.postFrameCallback
              Choreographer.postFrameCallback:761
                Choreographer.postFrameCallbackDelayed
                  Choreographer.postCallbackDelayedInternal
                    // 继续申请 Frame
                    Choreographer.scheduleFrameLocked
                      // 这里再次决定是走Vsync还是插帧(直接doFrame)
                      if (ScrollOptimizer.shouldUseVsync)
                      Choreographer.scheduleVsyncLocked
                        nativeScheduleVsync
                      else
                      // 插帧
                      mHandler.sendMessageAtTime(MSG_DO_FRAME)
```

## 相关文章

[Android 基于 Choreographer 的渲染机制详解](https://www.androidperformance.com/2019/10/22/Android-Choreographer)        
[Choreographer详解](https://github.com/zhpanvip/AndroidNote/wiki/Choreographer%E8%AF%A6%E8%A7%A3)        
[深入分析UI 上层事件处理核心机制 Choreographer](https://blog.csdn.net/farmer_cc/article/details/18619429)        
[Android图形渲染之Choreographer原理](https://hningoba.github.io/2019/11/28/Android%20Choreographer%E5%8E%9F%E7%90%86/)        
[Android 图形渲染【2】ViewRootImpl 与 Choreographer](https://juejin.cn/post/7475184869792284723)       

