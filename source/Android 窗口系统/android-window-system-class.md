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

### WindowConfiguration

WindowConfiguration类中包含窗口的基本信息，如bounds信息，窗口的模式，activity的类型等。      
作为 Configuration 类的成员变量。     

```
    private final Rect mBounds = new Rect();
    private Rect mAppBounds;
    private final Rect mMaxBounds = new Rect()
    private int mDisplayRotation = ROTATION_UNDEFINED;
    private int mRotation = ROTATION_UNDEFINED;
    // 当前container处于哪一种多窗口模式里。
    private @WindowingMode int mWindowingMode;
    //用来声明当前container是否总是处于顶层。
    private @AlwaysOnTop int mAlwaysOnTop;
    
    // 对于PIP模式和DREAM类型的container，是需要用于位于顶层的，其他的container，
    // 即使设置了mAlwaysOnTop为ALWAYS_ON_TOP_ON，也需要是WINDOWING_MODE_FREEFORM或WINDOWING_MODE_MULTI_WINDOW类型才行。
    public boolean isAlwaysOnTop() {
        if (mWindowingMode == WINDOWING_MODE_PINNED) return true;
        if (mActivityType == ACTIVITY_TYPE_DREAM) return true;
        if (mAlwaysOnTop != ALWAYS_ON_TOP_ON) return false;
        return mWindowingMode == WINDOWING_MODE_FREEFORM
                    || mWindowingMode == WINDOWING_MODE_MULTI_WINDOW;
    }
```

### DisplayAreaPolicy

用来管理 DisplayArea

```
    // 对应的 DefaultTaskDisplayArea
    protected final RootDisplayArea mRoot;
```

### Surface

窗口的本质是进行绘制所使用的画布：Surface。    
在Android中，Window与Surface一一对应，每个Window都对应一个Surface。当一块Surface显示在屏幕上时，就是用户所看到的窗口了。      
Window是Android框架中用于定义和管理一个可视区域的抽象概念，代表了应用程序的一个显示窗口。每个应用程序至少有一个Window，通常由Activity、Dialog或Popup创建。Window提供了高级别的API来管理其内容，比如设置布局参数、背景、透明度等，并且可以包含多个View层次结构。      
Surface则是 Android 底层图形系统中的一个概念，用于存储和展示像素数据，每个Window都有一个与之关联的Surface，由SurfaceFlinger服务管理。      
客户端向WMS添加一个窗口的过程，就是WMS为其分配一块Surface的过程，一块块Surface在WMS的管理之下有序地排布在屏幕上，Android才得以呈现出各种各样的界面出来。    

Surface的初始化时机在 ViewRootImpl 里，mSurface在初始化时是空的，在relayoutWindow窗口布局阶段时被填充。    

```
// ViewRootImpl.java
    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();
    private final Transaction mTransaction = new Transaction();
```

### SurfaceControl

用于管理和控制Surface的创建、显示和销毁。    
Surface和SurfaceControl的关系是，Surface是图形生产者和图像消费之间传递数据的缓冲区，而SurfaceControl是Surface的控制类，可以实现复杂的Surface操作，如改变位置、缩放、剪切、透明度以及Z序等。所以当我们需要对某个Surface操作时，只需要修改其SurfaceControl即可。      

### SurfaceControl.Transaction 
An atomic set of changes to a set of SurfaceControl，控制 `SurfaceControl` 的原子操作    
`Transaction` 是应用与 `SurfaceFlinger` 交流的方式之一，应用通过打开一个Transaction，然后设置各种 `setXXX` 操作，最后通过 `apply` 把所有的设定操作提交给 `SurfaceFlinger` 进行处理。    

### ViewRootImpl、 WindowManagerImpl、 WindowManagerGlobal

ViewRootImpl, WindowManagerImpl, WindowManagerGlobal 三者都存在于应用（有Activity)的进程空间里，一个Activity对应一个WindowManagerImpl、 一个DecorView(ViewRoot)、以及一个ViewRootImpl，而WindowManagerGlobals是一个全局对象，一个应用永远只有一个。    

#### WindowManagerImpl


### WindowStateAnimator

WindowStateAnimator 用来帮助 WindowState 管理 animator 和 surface 基本操作的，是窗口动画的总管。     

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
    
    // 管理SurfaceControl 的 WindowSurfaceController 对象
    WindowSurfaceController mSurfaceController;
    
    // 显示 surface
    void prepareSurfaceLocked(SurfaceControl.Transaction t)
```

### WindowSurfaceController

为 WindowStateAnimator 对象管理SurfaceControl对象。    

```
   // 真正用来绘制的带 buffer 的 SurfaceControl，是WindowState对应的 SurfaceControl
   SurfaceControl mSurfaceControl;
```

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

### AppTransition

应用程序状态切换时的状态管理。mNextAppTransition 是将要执行的过渡动画的类型。     
mAppTransitionState 保存的是当前过渡动画的状态。分别有 APP_STATE_IDLE，APP_STATE_READY，APP_STATE_RUNNING 和 APP_STATE_TIMEOUT 四种状态。     

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

### SurfaceFreezer

处理 Surface 冻结操作，比如在动画开始时冻结窗口的更新，以防止在动画过程中窗口的内容闪烁。    
此类处理 Animatable 的 “冻结”。有问题的 Animatable 应实现 Freezable。这样做的目的是使 WindowContainer 能够各自冻结自身。冻结意味着拍摄快照并将其放置在子层次结构中的所有内容之上。“放置在上方” 要求将父曲面插入到目标曲面的上方，以便目标曲面和快照是同级曲面。    
使用此功能进行过渡的总体流程为：      
1. 在 mChangingApps 中设置过渡并录制可动画化的动画     
2. 调用 {@link freeze} 设置皮带并用快照覆盖。     
3. 当过渡参与者准备就绪时，以此作为参数启动 SurfaceAnimator      
4. 然后，SurfaceAnimator 将 {@link takeLeashForAnimation} 而不是创建另一个皮带。     
5. 动画系统最终应该通过 {@link unfreeze} 来清理它。

### LocalAnimationAdapter

执行本地窗口动画，无需持有窗口管理器锁即可执行的动画。    

```
    private final AnimationSpec mSpec;

    private final SurfaceAnimationRunner mAnimator;
```

### RemoteAnimationAdapter

描述如何执行一个远程动画，远程动画可以让一个应用控制另一个应用的过渡。     

```
    private final IRemoteAnimationRunner mRunner;
    private IApplicationThread mCallingApplication;
```

### RemoteAnimationController

Helper 类，用于在远程进程中运行应用动画。      

```
    private final RemoteAnimationAdapter mRemoteAnimationAdapter;
```

### RemoteAnimationTarget

RemoteAnimationTarget 主要用来存放远程动画图层。     

### StartingSurfaceController

用来创建和删除 StartingWindow。     

### WindowOrganizerController

详情可参考后面的 WindowOrganizerController 相关博客。     

### ShellTransitions 相关类

参考 ShellTransitions 博客

#### TransitionController

#### Transition

#### Transitions

#### Track

#### TransitionPlayerImpl

#### ActiveTransition

#### TransitionInfo

#### TransitionHandler

#### TransitionPlayerRecord

#### TransitionRequestInfo

### TaskOrganizer

参考 WM shell 相关博客。    

### BLASTSyncEngine

详情可阅读后面的 BLASTSyncEngine 相关博客。      

### WindowContainerTransaction

详情可阅读后面的 WindowContainerTransaction 相关博客。      





