---
title: Android 图形系统 --  Window，WindowManager，DecorView 和 ViewRootImpl
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统 --  DecorView 和 ViewRootImpl
date: 2016-7-28 10:00:00
---


## 概述

本文是了解View绘制流程前的一些预备知识，本文会介绍 DecorView 是如何添加到PhoneWindow中，以及 ViewRootImpl 是如何连接 WindowManager 和 DecorView 的。     
Activity 控制生命周期和处理事件，Window 是视图的承载器，内部有个 DecorView 作为视图的根布局。在 Activity 中持有的是 Window 的子类 PhoneWindow。     
WindowManagerImpl 是 WindowManager 的实现类。
DecorView 是一个顶层 View，是应用的整个界面。     
ViewRootImpl 是 View 的大管家，实现 View 的绘制的类，里面有三个关键的方法来完成 View 的绘制流程。它是 DecorView 对应的 mParent 。普通子 View (非根 View) 对应的 mParent 是子 View 的父 View (即 ViewGroup)。     
在 ActivityThread 中，当 Activity 对象被创建完毕后，会将 DecorView 添加到 Window 中，同时会创建ViewRootImpl 对象，并将 ViewRootImpl 对象和 DecorView 建立关联，然后从 ViewRootImpl 的performTraversals 方法开始 View 的绘制流程。     

## PhoneWindow

PhoneWindow 是 Window 的子类。APP 中每个页面都对应一个 Window，例如每启动一个 Activity ，都会创建一个 PhoneWindow，当前页面所有的 View 都会在这个容器中展示。    
PhoneWindow的初始化：     

```
ActivityThread.handleLaunchActivity
    ActivityThread.performLaunchActivity
        Activity.attach
            PhoneWindow.<init>
```

```
//Activity.java
    @UnsupportedAppUsage
    final void attach(Context context, .....) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ......
    }
```

Activity 中创建了 Window，并且每个 Activity 都有一个与之对应的 Window ，具体的实现类，其实是一个 PhoneWindow。     

## WindowManager

WindowManagerImpl 是 WindowManager 的实现类，WindowManager是用于对Window的管理，例如前面我们提到的每次启动一个Activity都会创建一个Window，除了创建之外，还包括更新以及删除。    
过调用 `getSystemService(Context.WINDOW_SERVICE)` 返回的就是 `WindowManagerImpl` 实例。       

## WindowManagerGlobal

WindowManagerGlobal 在一个进程中存在一个实例，它的 addView 方法中会把的 View（View一般是DecorView类型），LayoutParams，ViewRootImpl 这些值保存起来，提供给其他类使用。并且最终会调用ViewRootImpl的setView方法开启View的绘制等流程。它的 removeView 方法中会把保存的View，LayoutParams，ViewRootImpl这些值移除。    


```
    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
```

另外 WindowManagerGlobal 创建了 IWindowSession 和 WMS 进行通信的对象。      

```
    private static IWindowManager sWindowManagerService;
    private static IWindowSession sWindowSession;
```

```
    public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                try {
                    if (sWindowManagerService != null) {
                        ValueAnimator.setDurationScale(
                                sWindowManagerService.getCurrentAnimatorScale());
                        sUseBLASTAdapter = sWindowManagerService.useBLAST();
                    }
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowManagerService;
        }
    }
    
    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    // Emulate the legacy behavior.  The global instance of InputMethodManager
                    // was instantiated here.
                    // TODO(b/116157766): Remove this hack after cleaning up @UnsupportedAppUsage
                    InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                    IWindowManager windowManager = getWindowManagerService();
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            });
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }    
```

IWindowSession：定义应用窗口和应用之间的会话，其实现类是Session；该会话是由IWindowManager负责open并维护的，他负责与APP直接打交道，通过binder机制实现IPC调用，APP通过它实现对IWindowManager的间接调用。    
IWindowManager 和  IWindowSession 都是和 WMS 进行通信的，那么为什么有些调用 APP不直接通过 IWindowManager 来和 WMS 打交道呢，还要通过 IWindowSession 来进行一次中转，这样不是反而让系统架构变得更复杂了吗？     

## DecorView

DecorView继承自FrameLayout，其作为顶级View被加载到window中，主要包含titlebar和contentParent两个子元素，其中titlebar可以通过系统属性指定其状态。     
我们来看一个应用的布局：     

```
DecorView
    ActionBarOverlayLayout(R.id.decor_content_parent)
        (R.id.content)
        ActionBarContainer(R.id.action_bar_container)
            Toolbar(R.id.action_bar)
        View(R.id.statusBarBackground)
        View(R.id.navigationBarBackground)
```

```
Activity.setContentView
    PhoneWindow.setContentView
        PhoneWindow.installDecor()
            PhoneWindow.generateDecor()
                new DecorView()
            PhoneWindow.generateLayout()
                requestFeature(FEATURE_ACTION_BAR)//设置actionbar
                contentParent=findViewById(ID_ANDROID_CONTENT)//获取R.id.content
                DecorView.setWindowBackground()//设置背景
    LayoutInflater.inflate(layoutResID, mContentParent)
```

DecorView 是在 PhoneWindow 中生成的，然后把用户设置的布局添加到DecorView对应的子View上去。     

## ViewRootImpl

ViewRootImpl 可以理解为一个 Window 中所有 View 的根 View 的管理者（但 ViewRootImpl 不是 View，只是实现了 ViewParent 接口），是 View 与 WindowManagerService 之间联系的桥梁，作用总结如下：     

 - 是 APP 视图和 WMS 沟通的桥梁，通过 mWindowSession 调用 WMS 方法以及 回调。     
 - 完成 View 的绘制过程，包括 measure、layout、draw 过程    
 - 向 DecorView 分发收到的用户发起的 event 事件，如按键，触屏等事件。    

在创建ViewRootImple对象时，默认调用WindowManagerGlobal的静态方法getWindowSession，获取WMS本地代理对象，调用WMS的openSession方法创建Session对象。      

```
ActivityThread.handleResumeActivity()
　　Activity.getWindow() //获取activity的PhoneWindow
    PhoneWindow.getDecorView()//获取PhoneWindow的DecorView
    Activity.getWindowManager()//获取Activity的WindowManagerImpl
    WindowManagerImpl.addView(decor, l)//通过WindowManagerImpl将DecorView添加到Activity
        WindowManagerGlobal.addView(DecorView)
            new ViewRootImpl()//创建ViewRootImpl对象
            ViewRootImpl.setView()
                ViewRootImpl.requestLayout()
                    ViewRootImpl.scheduleTraversals()
                        Choreographer.postCallback(mTraversalRunnable)
                            ViewRootImpl.doTraversal()
                                ViewRootImpl.performTraversals()
                                    ViewRootImpl.relayoutWindow()
                                    ViewRootImpl.performMeasure()
                                        DecorView.measure()
                                    ViewRootImpl.performLayout()
                                        DecorView.layout()
                                    ViewRootImpl.performDraw()
                                        DecorView.draw()
                WindowSession.addToDisplay()
                new WindowInputEventReceiver()
```

然后，DecorView的绘制流程就从ViewRoot的 performTraversals 方法开始了，后面就是遍历子view完成整个界面的测量，布局和绘制。     
performTraversals()可能并不会执行完整的measure、layout和draw流程，内部有很复杂的判断条件，比如执行 `requestLayout()` 可能只会执行measure和layout，执行invalidate()只会执行draw方法。这方面的知识点后面详细介绍。     
