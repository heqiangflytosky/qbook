---
title: Android SurfaceControlViewHost 实现 WindowlessWindow
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android SurfaceControlViewHost 实现 WindowlessWindow
date: 2022-11-23 10:00:00
---




## 概述

SurfaceControlViewHost 可以实现一个进程的 View 在另外一个进程显示出来（网上说法），这样描述其实并不准确，其实也就是把一个进程 View 的图层挂载到了另外一个进程的图层上面。     
通过 SurfaceControlViewHost 创建的图层在窗口层级树中没有对应的 WindowState，所以也叫 Windowless 模式。      

具体使用 Windowless 的场景在 wmshell 的业务部分使用比较多的，比如分屏DivideView，PIP 的 CaptionView。     

SurfaceControlViewHost 的成员变量：    
 - WindowlessWindowManager mWm：构造时传入，或者构造时创建。为IWindowSession 子类， 该类并不将一个view加入到wms中作为窗口管理， 而是将该view作为一个子surface加入到另一个父surface中。构造时创建时， 使用本类的mSurfaceControl作为参数， 作为WindowlessWindowManager的mRootSurface。 WindowlessWindowManager类的addToDisplay是按照 WindowManager.LayoutParams 创建一个surfacecontrol， 该surfacecontrol 对应SurfaceFlinger的buffer Layer, 分配具体的绘制buffer, 绘制进程的view 即绘制在该surface上。 该surface 存入WindowlessWindowManager.State.mSurfaceControl， mRootSurface为其parent。 WindowlessWindowManager类的relayout()中按照输入高宽及LayoutParams调整WindowlessWindowManager.State.mSurfaceControl的参数。 
 - ViewRootImpl mViewRoot：在SurfaceControlViewHost类构造时创建， 传入的参数为WindowlessWindowManager， 构造时会调用ViewRootImpl.forceDisableBLAST(),即绘制buffer在surfaceFlinger侧分配管理， 而不是在app侧。
 - SurfaceControl mSurfaceControl：构造时创建， 名字为“SurfaceControlViewHost”， 对应SurfaceFlinger中的ContainerLayer， 作为整个绘制surface的根。 其子layer 为在WindowlessWindowManager.addToDisplay中创建的buffer layer。 mSurfaceControl也作为根layer通过SurfacePackage传递给远端显示进程。 

方法：
 - getSurfacePackage() ：创建SurfacePackage：  new SurfacePackage(mSurfaceControl, mAccessibilityEmbeddedConnection); 其中SurfaceControlViewHost.mSurfaceControl 也作为SurfacePackage的mSurfaceControl， 会加入到显示进程中的SurfaceView中。、
 - setView(View, ......): 最终调用的是mViewRoot.setView(view, attrs, null)，进而调用WindowlessWindowManager.addToDisplay() 和relayout(), 将该view内容与WindowlessWindowManager.State.mSurfaceControl关联。 该mSurfaceControl即为buffer  layer。

## 分屏中的使用

```
// SplitWindowManager.java
    void init(SplitLayout splitLayout, InsetsState insetsState, boolean isRestoring) {
    ......
        mViewHost = new SurfaceControlViewHost(mContext, mContext.getDisplay(), this,
                "SplitWindowManager");
        mDividerView = (DividerView) LayoutInflater.from(mContext)
                .inflate(R.layout.split_divider, null /* root */);

        final Rect dividerBounds = splitLayout.getDividerBounds();
        WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
                dividerBounds.width(), dividerBounds.height(), TYPE_DOCK_DIVIDER,
                FLAG_NOT_FOCUSABLE | FLAG_NOT_TOUCH_MODAL | FLAG_WATCH_OUTSIDE_TOUCH
                        | FLAG_SLIPPERY,
                PixelFormat.TRANSLUCENT);
        lp.token = new Binder();
        lp.setTitle(mWindowName);
        lp.privateFlags |= PRIVATE_FLAG_NO_MOVE_ANIMATION | PRIVATE_FLAG_TRUSTED_OVERLAY;
        lp.accessibilityTitle = mContext.getResources().getString(R.string.accessibility_divider);
        mViewHost.setView(mDividerView, lp);
        mDividerView.setup(splitLayout, this, mViewHost, insetsState);
        if (isRestoring) {
            mDividerView.setInteractive(mLastDividerInteractive, mLastDividerHandleHidden,
                    "restore_setup");
        }
```

```
Transitions.onTransitionReady
  Transitions.dispatchReady
    Transitions.processReadyQueue
      Transitions.playTransitionWithTracing
        Transitions.playTransition
          StageCoordinator.startAnimation
            StageCoordinator.startPendingAnimation
              StageCoordinator.startPendingEnterAnimation
                StageCoordinator.finishEnterSplitScreen
                  SplitLayout.update
                    SplitWindowManager.init
                      SurfaceControlViewHost.setView
                        ViewRootImpl.setView
                          WindowlessWindowManager.addToDisplayAsUser
                            WindowlessWindowManager.addToDisplay
                              SplitWindowManager.getParentSurface
                                StageCoordinator$1.attachToParentSurface
                                  SurfaceControl$Builder.setParent
```

```
// SplitWindowManager.java
    void init(SplitLayout splitLayout, InsetsState insetsState, boolean isRestoring) {
    ......
        // 创建 SurfaceControlViewHost hostToken 设置为 SplitWindowManager
        // SplitWindowManager 继承自 WindowlessWindowManager
        mViewHost = new SurfaceControlViewHost(mContext, mContext.getDisplay(), this,
                "SplitWindowManager");
```

初始化 SurfaceControlViewHost 时需要配置 WindowlessWindowManager 参数。     

```
//SurfaceControlViewHost.java

    public SurfaceControlViewHost(@NonNull Context c, @NonNull Display d,
            @NonNull WindowlessWindowManager wwm, @NonNull String callsite) {
        mSurfaceControl = wwm.mRootSurface;
        mWm = wwm;
        // 创建 ViewRootImpl ，IWindowSession 设置为 WindowlessWindowManager
        mViewRoot = new ViewRootImpl(c, d, mWm, new WindowlessWindowLayout());
        mCloseGuard.openWithCallSite("release", callsite);
        setConfigCallback(c, d);

        WindowManagerGlobal.getInstance().addWindowlessRoot(mViewRoot);

        mAccessibilityEmbeddedConnection = mViewRoot.getAccessibilityEmbeddedConnection();
    }
```

这里 创建 ViewRootImpl 时 IWindowSession 设置为 WindowlessWindowManager，那么后面的 `mWindowSession.addToDisplayAsUser` 就调用到了 `WindowlessWindowManager.addToDisplayAsUser`。     


## 示例

```
    public static void addWindowlessSurface(Context context,SurfaceControl rootSurface) {
        WindowlessWindowManager windowlessWindowManager = new MyWindowlessWindowManager(context.getResources().getConfiguration(), rootSurface, null);
        SurfaceControlViewHost host = new SurfaceControlViewHost(context, context.getDisplay(), windowlessWindowManager,"Test");

        WindowManager.LayoutParams lp = new WindowManager.LayoutParams();
        lp.token = new Binder();

        lp.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        lp.format = PixelFormat.RGBA_8888;
        lp.gravity = Gravity.CENTER;
        lp.width = 600;
        lp.height = 600;
        lp.x = 200;
        lp.y = 200;
        //lp.windowAnimations = R.style.MyWindowAnimation;
        //lp.preferredRefreshRate = 30f
        lp.privateFlags |= PRIVATE_FLAG_NO_MOVE_ANIMATION | PRIVATE_FLAG_TRUSTED_OVERLAY;
        lp.inputFeatures |= INPUT_FEATURE_NO_INPUT_CHANNEL;
        lp.setTitle("WMSTestActivity 测试窗口");

        View view = LayoutInflater.from(context).inflate(R.layout.window, null);
        view.findViewById(R.id.button).setOnClickListener(v -> {
            Log.d("TAG", "click");
        });

        host.setView(view, lp);
    }

    static class MyWindowlessWindowManager extends WindowlessWindowManager {
        // 主要是 WindowlessWindowManager 的 mRootSurface 的设置
        public MyWindowlessWindowManager(Configuration c, SurfaceControl rootSurface, InputTransferToken hostInputTransferToken) {
            super(c, rootSurface, hostInputTransferToken);
        }

        @Override
        protected SurfaceControl getParentSurface(IWindow window, WindowManager.LayoutParams attrs) {
            SurfaceControl leash = new SurfaceControl.Builder()
                    .setBLASTLayer()
                    .setName("TestLeash")
                    .setHidden(false)
                    .setParent(mRootSurface)
                    .setCallsite("Test").build();

            SurfaceControl.Transaction transaction = new SurfaceControl.Transaction();
            transaction.setLayer(leash, 10);
            transaction.apply();

            return leash;
        }
    }
```


需要及时移除 WindowlessWindow      

## 为什么要使用 WindowlessWindow

 - 可以实现把一个View附加到没有Activity，没有Window等窗口容器，直接附加到SurfaceControl的显示容器，提供显示View一种新方法。
 - 轻量级别使用的 ViewRootImpl，不需要维护 wms 窗口，只需要考虑绘制和触摸。
 - 理论上所有Windowless显示内容都可以使用正常WidnowState显示，但是因为wmshell中有很多额外窗口需求，这样可能会需要额外增加比较多的窗口类型type，这样针对一些少见业务场景，然后去改动整个wms层级结构树的业务成本太大，即实现了一些业务窗口与整个系统窗口结构管理的解耦。解决了为每个场景新增 WindowState 子类这种类膨胀问题。


## SurfaceView + SurfaceControlViewHost

结合 SurfaceView  独立渲染的特性，我们可以通过 SurfaceControlViewHost 把一个 View 结构嵌入到 SurfaceView 中，适合于一个经常需要刷新 View，但又不想影响父布局刷新的场景。      
比如在某个View 中需要显示一个时钟的场景。       

```
    private void initView() {
        mSurfaceView = findViewById(R.id.sv);
        mAttachView = View.inflate(this, R.layout.layout_clock, null);
        ClockTextView clockTextView = mAttachView.findViewById(R.id.clockView);
        clockTextView.start();

        mSurfaceView.getHolder().addCallback(new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(@NonNull SurfaceHolder surfaceHolder) {
                mViewHost = new SurfaceControlViewHost(
                        SurfaceViewActivity.this,
                        getDisplay(),
                        getWindow().getDecorView().getWindowToken());
                mViewHost.setView(mAttachView, 400,200);

                // 将SurfaceView的子Surface设置为SurfaceControlViewHost的Surface
                // 这样就把 mAttachView 的视图结构绘制到 SurfaceView 上
                mSurfaceView.setChildSurfacePackage(mViewHost.getSurfacePackage());
            }

            @Override
            public void surfaceChanged(@NonNull SurfaceHolder surfaceHolder, int i, int i1, int i2) {

            }

            @Override
            public void surfaceDestroyed(@NonNull SurfaceHolder surfaceHolder) {
                mViewHost.release();
                mViewHost = null;
            }
        });
    }
```

<img src="/images/android-window-system-surface-control-view-host/layer.png" width="401" height="456"/>

可以看到，被嵌入到 SurfaceView 的视图也是在一个单独的 Layer 上面绘制的。         

## 相关文章

[安卓为什么要引入无窗口显示方式Windowless/SurfaceControlViewHost](https://www.bilibili.com/opus/1025828023586258944)       
