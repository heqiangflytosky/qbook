---
title: Android VSYNC 机制
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统基础知识和相关类
date: 2022-11-23 10:00:00
---

## VSYNC 基础

在 Android 中，VSYNC（Vertical Synchronization）是一个垂直同步信号，用于协调显示刷新和绘图操作。VSYNC 信号的主要作用是控制屏幕刷新频率与图形渲染的同步，Vsync + TripleBuffer + Choreographer 一起工作，以确保画面显示平滑且没有掉帧和撕裂现象。    
撕裂现象指的是正在渲染时传入新的图像，这时会导致屏幕上面部分绘制的上一帧图像，下面部分绘制的是下一帧图像导致画面撕裂的问题。也就是说用两帧的部分数据合成一帧。    
如下图：    

<img src="/images/android-graphic-system-vsync/0.png" width="1054" height="392" />

### VSync信号分类

VSync信号可以分为分类：

*   软件VSync信号（SW-VSync）：SW-VSync模型产生。
*   硬件VSync信号（HW-VSync）：负责对SW-VSync模型进行校准。

系统中分发的VSync信号是SW-VSync信号。SW-VSync信号的分发采用单次申请回调制，一次申请对应一次回调，不申请则没有回调。    
我们通过 Perfetto 中观察到的 `HW_VSYNC_displayid` 就是 HW_VSYNC。`HW_VSYNC_ON_displayid` 表示 HW_VSYNC 是否打开。displayId在sf中标识这个显示屏的唯一字符串。    

<img src="/images/android-performance-optimization-tools-perfetto/hw-vsync.png" width="483" height="267" />

硬件VSYNC大部分时间是关闭的，只有在特殊场景下才会打开（比如更新SW VSYNC模型的时候）。    

三种场景下会开启硬件VSync信号HW-VSync会对软件VSync信号SW-VSync进行校准。      

*   SurfaceFlinger初始化。
*   连续两次请求VSync-app信号的时间间隔超过750ms。
*   SurfaceFlinger合成后，添加FenceTime到VSyncTracker中导致模型计算误差过大。

软件vsync是基于硬件vsync产生的计算模型产生，为什么不直接使用硬件vsync呢？      
如果这样的话会让每个使用的App和Surfacefliner去直接监听硬件vsync会导致上层直接连接到硬件，这样的耦合性太高了，而且App和Surfacefliner去直接监听硬件vsync的话，会导致功耗增大。      
vsync信号是固定周期的，用软件容易进行模拟。      

### SW-VSync 分类

App的绘制以及SF的合成分别由对应的软件VSYNC来驱动的。      
SW-VSync信号的分类：      

*   app-vsync：控制 app上帧的vsync，触发app进行doframe，驱动 App 绘制。
*   appsf-vsync: 一般控制 system_server 和某些 app（比如systemui）上帧的 vsync，触发app进行doframe， 和 app-vsync相位上有一定的偏差，使用较少。下面详细介绍。    
*   sf-vsync：控制 SurfaceFliner进行 Layer 合成的vsync。

这些 SW-VSync信号都是按需发射的，如果 App 需要更新界面，它就需要申请 VSYNC-app，如果没有App申请VSYNC-app，那么VSYNC-app将不再发射。同样，当App更新了界面，它会把对应的Graphic Buffer放到Buffer Queue中。Buffer Queue通知SF进行合成，此时SF会申请VSYNC-sf。如果SF不再申请VSYNC-sf，VSYNC-sf将不再发射。    
意味着，如果App要持续不断的更新，它就得不断去申请VSYNC-app；而对SF来说，只要有合成任务，它就得再去申请VSYNC-sf。    
VSYNC-app与VSYNC-sf是相互独立的。VSYNC-app触发App的绘制，Vsync-sf触发SF合成。App绘制与SF合成都会加大CPU的负载，为了避免绘制与合成打架造成的性能问题，VSYNC-app可以与VSYNC-sf稍微错开一下。     

<img src="/images/android-graphic-system-vsync/app-sf-phase.png" width="972" height="114" />

### appsf-vsync

这里着重介绍一下 appsf-vsync，因为在早期的Android版本中，是没有这个类型的VSync信号的。我们先看看Google在添加这个类型的VSync信号时代码中添加的注释：      

```
    SurfaceFlinger: decouple EventThread from SF wakeup
        
        Today we have two instances of EventThread:
        1. 'app' - used to wake up Choreographer clients
        2. 'sf' - used to wake up SF mian thead *and*
                  Choreographer clients that uses sf instance
        
        Now this creates an ambiguity when trying to reason about the expected
        vsync time and deadline of 'sf' EventThread:
         - SF wakes up sfWorkDuration before a vsync and targets that vsync
         - Choreographer users wakes up with SF main thread but targets the
           vsync that happens after the next SF wakeup.
        
        To resolve this ambiguity we are decoupling SF wakeup from 'sf'
        EventThread. This means that Choreographer clients that uses 'sf'
        instance will keep using the EventThread but SF will be waking up
        directly by a callback with VSyncDispatch. This allows us to correct
        the expected vsync and deadline for both.
        
        Test: Interacting with the device and observe systraces
        Test: new unit test added to SF suite
        Bug: 166302754
        
        Change-Id: I76d154029b4bc1902198074c33d38ff030c4601b
```
翻译：    

```
    SurfaceFlinger：将 EventThread 与 SF 唤醒解耦

    如今我们有两个 EventThread 实例：

        “app” - 用于唤醒 Choreographer 客户端
        “sf” - 用于唤醒 SF 主线程以及使用 sf 实例的 Choreographer 客户端


    现在，在试图推断 “sf” EventThread 的预期垂直同步时间和截止日期时，这会产生歧义：

        SF 在垂直同步前 sfWorkDuration 唤醒并以该垂直同步为目标
        Choreographer 用户与 SF 主线程一起唤醒，但以 SF 下次唤醒后发生的垂直同步为目标。


    为了解决这种歧义，我们正在将 SF 唤醒与 “sf” EventThread 解耦。这意味着使用 “sf” 实例的 Choreographer 客户端将继续使用 EventThread，但 SF 将通过 VSyncDispatch 的回调直接唤醒。这使我们能够为两者更正预期的垂直同步和截止日期。
```

也就是说，以前有两个 EventThread 实例，一个应用唤醒 Choreographer 客户端，一个用于唤醒 sf 自己主线程以及使用 sf vsync 的Choreographer客户端，现在为了解耦，要把 sf 这个 EventThread 实例进行拆分，唤醒 sf 自己主线程的 vsync 的部分移到 VSyncDispatch直接进行，另外使用 sf vsync 的 Choreographer 客户端的仍然留在 EventThread 中处理，把这部分修改成 appsf，这就是 appsf-vsync 的由来。     
那么大家可能会有个疑问，app 不是给客户端提供 vsync 信号的吗？为什么 sf 也会给客户端提供 vsync 信号呢？      
我们来看Choreographer的构造方法：      

```
        private Choreographer(Looper looper, int vsyncSource, long layerHandle) {
            mLooper = looper;
            mHandler = new FrameHandler(looper);
            mDisplayEventReceiver = USE_VSYNC
                    ? new FrameDisplayEventReceiver(looper, vsyncSource, layerHandle)
                    : null;
```

有个参数 `vsyncSource`，这个参数又用来构造 FrameDisplayEventReceiver，在其父类中有对这个参数做解释：    

```
        /**
         * Creates a display event receiver.
         *
         * @param looper The looper to use when invoking callbacks.
         * @param vsyncSource The source of the vsync tick. Must be on of the VSYNC_SOURCE_* values.
         * @param eventRegistration Which events to dispatch. Must be a bitfield consist of the
         * EVENT_REGISTRATION_*_FLAG values.
         * @param layerHandle Layer to which the current instance is attached to
         */
        public DisplayEventReceiver(Looper looper, int vsyncSource, int eventRegistration,
                long layerHandle) {
            if (looper == null) {
                throw new IllegalArgumentException("looper must not be null");
            }
```

垂直同步时钟周期的来源。必须是 `VSYNC_SOURCE_*` 值之一。    
它对应的值是：    

```
    // DisplayEventReceiver
        /**
         * When retrieving vsync events, this specifies that the vsync event should happen at the normal
         * vsync-app tick.
         * <p>
         * Keep in sync with frameworks/native/libs/gui/aidl/android/gui/ISurfaceComposer.aidl
         */
        public static final int VSYNC_SOURCE_APP = 0;

        /**
         * When retrieving vsync events, this specifies that the vsync event should happen whenever
         * Surface Flinger is processing a frame.
         * <p>
         * Keep in sync with frameworks/native/libs/gui/aidl/android/gui/ISurfaceComposer.aidl
         */
        public static final int VSYNC_SOURCE_SURFACE_FLINGER = 1;
```
一个是常见的 app source，另外一个就是 sf source，也就是说，app客户端是可以指定跟随使用 sf 的 vsync 信号的，具体有谁使用呢，我们可以在代码里面查看一下，    

```
    /**
     * @hide
     */
    @UnsupportedAppUsage
    public static Choreographer getSfInstance() {
        return sSfThreadInstance.get();
    }

    // Thread local storage for the SF choreographer.
    private static final ThreadLocal<Choreographer> sSfThreadInstance =
            new ThreadLocal<Choreographer>() {
                @Override
                protected Choreographer initialValue() {
                    Looper looper = Looper.myLooper();
                    if (looper == null) {
                        throw new IllegalStateException("The current thread must have a looper!");
                    }
                    return new Choreographer(looper, VSYNC_SOURCE_SURFACE_FLINGER);
                }
            };

```

使用 getSfInstance() 的地方还是蛮多的，比如 systemui 中，SurfaceAnimationRunner 中，WindowAnimator中等。

```
        SurfaceAnimationRunner(@Nullable AnimationFrameCallbackProvider callbackProvider,
                AnimatorFactory animatorFactory, Transaction frameTransaction,
                PowerManagerInternal powerManagerInternal) {
            mSurfaceAnimationHandler.runWithScissors(() -> mChoreographer = getSfInstance(),
                    0 /* timeout */);
```

```
        WindowAnimator(final WindowManagerService service) {
            mService = service;
            mContext = service.mContext;
            mPolicy = service.mPolicy;
            mTransaction = service.mTransactionFactory.get();
            service.mAnimationHandler.runWithScissors(
                    () -> mChoreographer = Choreographer.getSfInstance(), 0 /* timeout */);
```

## Vsync框架
