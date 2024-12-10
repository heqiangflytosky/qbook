---
title: Android 窗口系统相关类
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口系统相关类
date: 2022-11-23 10:00:00
---

## 窗口相关类

### WindowManagerService

mWindowMap:mWindowMap保存了每个WindowState和客户端窗口的映射关系，客户端应用请求窗口操作时，通过mWindowMap查询到对应的WindowState。    

### ActivityTaskManagerService

```
    RootWindowContainer mRootWindowContainer;
    WindowManagerService mWindowManager;
```

### WindowContainer

窗口容器类，它有众多的子类来构建窗口层级系统，具体见[Android 窗口层级树](http://www.heqiangfly.com/qbook/source/Android%20%E7%AA%97%E5%8F%A3%E7%B3%BB%E7%BB%9F/android-window-system-window-tree-basic.html)。    

### DisplayAreaPolicy

用来管理 DisplayArea

```
    // 对应的 DefaultTaskDisplayArea
    protected final RootDisplayArea mRoot;
```

### Surface

窗口的本质是进行绘制所使用的画布：Surface。    
在Android中，Window与Surface一一对应，当一块Surface显示在屏幕上时，就是用户所看到的窗口了。客户端向WMS添加一个窗口的过程，就是WMS为其分配一块Surface的过程，一块块Surface在WMS的管理之下有序地排布在屏幕上，Android才得以呈现出各种各样的界面出来。    

Surface的初始化时机在 ViewRootImpl 里，mSurface在初始化时是空的，在relayoutWindow窗口布局阶段时被填充。    

```
// ViewRootImpl.java
    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();
    private final Transaction mTransaction = new Transaction();
```

### SurfaceControl

用于管理和控制Surface的创建、显示和销毁。    

### SurfaceControl.Transaction 
An atomic set of changes to a set of SurfaceControl，控制 `SurfaceControl` 的原子操作    
`Transaction` 是应用与 `SurfaceFlinger` 交流的方式之一，应用通过打开一个Transaction，然后设置各种 `setXXX` 操作，最后通过 `apply` 把所有的设定操作提交给 `SurfaceFlinger` 进行处理。    

### ViewRootImpl、 WindowManagerImpl、 WindowManagerGlobal

ViewRootImpl, WindowManagerImpl, WindowManagerGlobal 三者都存在于应用（有Activity)的进程空间里，一个Activity对应一个WindowManagerImpl、 一个DecorView(ViewRoot)、以及一个ViewRootImpl，而WindowManagerGlobals是一个全局对象，一个应用永远只有一个。    

#### WindowManagerImpl


### WindowStateAnimator

WindowStateAnimator 用来帮助WindowState管理 animator 和 surface 基本操作的    

```
    private final WallpaperController mWallpaperControllerLocked;

    // 如果在显示窗口时，它应执行 enter 动画，则设置为 true。
    boolean mEnterAnimationPending;
    // 用于指示此窗口正在进行 enter 动画。
    boolean mEnteringAnimation;
    
    // 定义窗口从最初创建到显示在屏幕上的5个状态
    // 窗口刚刚被创建，还没有Surface。
    static final int NO_SURFACE = 0;
    //在创建 Surface 之后但在绘制窗口之前设置的。在此期间，Surface 是隐藏的。
    此时窗口创建了Surface，但还没有绘制内容。
    static final int DRAW_PENDING = 1;
    //这是在窗口首次完成绘制之后，但在显示其 Surface 之前设置的。运行下一个布局时，将显示图面。
    //此时窗口完成了第一次绘制，但还没有显示。
    static final int COMMIT_DRAW_PENDING = 2;
    // 这是在提交窗口的绘制之后和实际显示其 Surface 之前的时间内设置的。它用于延迟显示 Surface，直到同一WindowToken的的所有窗口都准备好显示。
    // 此时窗口已经准备好显示，但可能还需要等待其他窗口准备就绪。
    static final int READY_TO_SHOW = 3;
    // 当窗口在在屏幕中首次显示之后设置。
    // 此时窗口已经完成绘制并显示在屏幕上
    static final int HAS_DRAWN = 4;
    
    // 显示 surface
    void prepareSurfaceLocked(SurfaceControl.Transaction t)
```

### WindowSurfaceController

为WindowStateAnimator对象管理SurfaceControl对象。    

### WindowSurfacePlacer

定位窗口和Surface的位置。    

### WindowLayout

计算窗口和Surface大小，主要有两个方法：

 - computeFrames：用于计算窗口的位置和大小
 - computeSurfaceSize

### WindowFrames

这个类中提供了众多Rect来描述不同的区域边界，这些Rect的值有的来自DisplayFrames中，有的来自computeFrameLw的计算。    
通过 `WindowState.setFrames` 更新 Frames。    

```
    /**
     * The frame to be referenced while applying gravity and MATCH_PARENT.
     */
    public final Rect mParentFrame = new Rect();

    /**
     * The bounds that the window should fit.
     */
    public final Rect mDisplayFrame = new Rect();

    /**
     * "Real" frame that the application sees, in display coordinate space.
     */
    final Rect mFrame = new Rect();

    // 报送给客户端的最后一帧
    final Rect mLastFrame = new Rect();

    /**
     * mFrame but relative to the parent container.
     */
    final Rect mRelFrame = new Rect();

    // 报送给客户端的最后一帧，但是是和父容器的相对位置
    final Rect mLastRelFrame = new Rect();


    // Frame that is scaled to the application's coordinate space when in
    // screen size compatibility mode.
    final Rect mCompatFrame = new Rect();
```

### WallpaperController

控制墙纸窗口的可见性、排序等。    

### AppTransitionController

检查应用程序过渡准备情况，解析动画属性，并为在应用程序过渡过程中添加动画效果的应用程序执行可见性更改。    

```
    // 处理给定 Display 的应用程序过渡。
    void handleAppTransitionReady()
    // 执行应用过渡动画
    private void applyAnimations(ArraySet<ActivityRecord> openingApps,
            ArraySet<ActivityRecord> closingApps, @TransitionOldType int transit,
            LayoutParams animLp, boolean voiceInteraction)
```

### TaskLaunchParamsModifier

定义 Task 的默认启动参数的类。      


### SurfaceAnimator

Android系统中用于处理窗口动画的核心组件，主要负责管理和执行窗口的动画效果。通过 WindowAnimationRunner 和 WindowAnimationSpec 等组件来具体实现动画效果。      
可以在具有一组子表面的对象上运行动画的类。我们通过将对象的所有子表面重新设置到一个名为 “Leash” 的新表面上来实现这一点。Leash 将附加到子对象所附加到的 Surface 层次结构中。然后，我们将 Leash 移交给处理动画的组件，该动画由 {@link AnimationAdapter} 指定。当动画完成动画制作后，将调用我们的回调以完成动画，此时我们将子项重新设置为原始父项。      

### SurfaceAnimationRunner

用于管理窗口动画的一个关键组件。它主要负责在窗口动画过程中调度和管理动画任务，确保动画的平滑和高效执行。       

### WindowAnimationSpec

用于定义窗口动画的规格。它允许开发者指定动画的运行时长、当前时间比例等参数，从而控制动画的播放效果。       


