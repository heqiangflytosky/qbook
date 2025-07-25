---
title: Android 添加窗口的 token 和 type 与窗口层级
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 添加窗口的 token 和 type 与窗口层级的关系
date: 2022-11-23 10:00:00
---


## 概述

Android 应用开发可以创建不同类型的窗口，窗口的类型不同，在窗口层级的位置也就不同。    
本文基于 Android 15分析。    

## 窗口类型和层级

APPLICATION 类型窗口（1 - 99）    
APPLICATION 类型窗口会显示在它的token的 ActivityRecord 的下面，和WindowState同级。    
```
        public static final int FIRST_APPLICATION_WINDOW = 1;

        // 其他的应用窗口都会在TYPE_BASE_APPLICATION的上面
        public static final int TYPE_BASE_APPLICATION   = 1;

        // 普通应用程序窗口 token 必须是 Activity token
        public static final int TYPE_APPLICATION        = 2;

        // Starting Window
        public static final int TYPE_APPLICATION_STARTING = 3;

        // 可确保窗口管理器在显示应用之前等待绘制此窗口
        public static final int TYPE_DRAWN_APPLICATION = 4;

        public static final int LAST_APPLICATION_WINDOW = 99;
```

子窗口类型（1000 - 1999）    
显示在 WindowState 节点的下面。      

```
        public static final int FIRST_SUB_WINDOW = 1000;

        // 显示在应用的窗口之上
        public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;

        // 显示在它 Attach Window 之后
        public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;

        // 显示在应用的窗口和TYPE_APPLICATION_PANEL之上
        public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;

        public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;

        @UnsupportedAppUsage
        public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;

        public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;

        public static final int LAST_SUB_WINDOW = 1999;
```

系统窗口类型（2000 - 2999）     

```
        public static final int FIRST_SYSTEM_WINDOW     = 2000;

        public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;

        public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;

        @Deprecated
        public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;

        @Deprecated
        public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;

        public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;

        @Deprecated
        public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;

        @Deprecated
        public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;

        @Deprecated
        public static final int TYPE_PRIORITY_PHONE     = FIRST_SYSTEM_WINDOW+7;

        public static final int TYPE_SYSTEM_DIALOG      = FIRST_SYSTEM_WINDOW+8;

        public static final int TYPE_KEYGUARD_DIALOG    = FIRST_SYSTEM_WINDOW+9;

        @Deprecated
        public static final int TYPE_SYSTEM_ERROR       = FIRST_SYSTEM_WINDOW+10;

        public static final int TYPE_INPUT_METHOD       = FIRST_SYSTEM_WINDOW+11;

        public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;

        public static final int TYPE_WALLPAPER          = FIRST_SYSTEM_WINDOW+13;

        public static final int TYPE_STATUS_BAR_PANEL   = FIRST_SYSTEM_WINDOW+14;

        @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        public static final int TYPE_SECURE_SYSTEM_OVERLAY = FIRST_SYSTEM_WINDOW+15;

        public static final int TYPE_DRAG               = FIRST_SYSTEM_WINDOW+16;

        public static final int TYPE_STATUS_BAR_SUB_PANEL = FIRST_SYSTEM_WINDOW+17;

        public static final int TYPE_POINTER = FIRST_SYSTEM_WINDOW+18;

        public static final int TYPE_NAVIGATION_BAR = FIRST_SYSTEM_WINDOW+19;

        public static final int TYPE_VOLUME_OVERLAY = FIRST_SYSTEM_WINDOW+20;

        public static final int TYPE_BOOT_PROGRESS = FIRST_SYSTEM_WINDOW+21;

        public static final int TYPE_INPUT_CONSUMER = FIRST_SYSTEM_WINDOW+22;

        public static final int TYPE_NAVIGATION_BAR_PANEL = FIRST_SYSTEM_WINDOW+24;

        @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        public static final int TYPE_DISPLAY_OVERLAY = FIRST_SYSTEM_WINDOW+26;

        public static final int TYPE_MAGNIFICATION_OVERLAY = FIRST_SYSTEM_WINDOW+27;

        public static final int TYPE_PRIVATE_PRESENTATION = FIRST_SYSTEM_WINDOW+30;

        public static final int TYPE_VOICE_INTERACTION = FIRST_SYSTEM_WINDOW+31;

        public static final int TYPE_ACCESSIBILITY_OVERLAY = FIRST_SYSTEM_WINDOW+32;

        public static final int TYPE_VOICE_INTERACTION_STARTING = FIRST_SYSTEM_WINDOW+33;

        public static final int TYPE_DOCK_DIVIDER = FIRST_SYSTEM_WINDOW+34;

        public static final int TYPE_QS_DIALOG = FIRST_SYSTEM_WINDOW+35;

        public static final int TYPE_SCREENSHOT = FIRST_SYSTEM_WINDOW + 36;

        public static final int TYPE_PRESENTATION = FIRST_SYSTEM_WINDOW + 37;

        public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;

        public static final int TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 39;

        public static final int TYPE_NOTIFICATION_SHADE = FIRST_SYSTEM_WINDOW + 40;

        public static final int TYPE_STATUS_BAR_ADDITIONAL = FIRST_SYSTEM_WINDOW + 41;

        public static final int LAST_SYSTEM_WINDOW      = 2999;
```

## 添加和删除窗口

```
    fun addLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        var lp = WindowManager.LayoutParams()
        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG
        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL
        //lp.token = window.decorView.windowToken

        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION

        lp.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        lp.format = PixelFormat.RGBA_8888
        lp.gravity = Gravity.CENTER
        //lp.x = 200
        //lp.y = 200
        lp.width = WindowManager.LayoutParams.WRAP_CONTENT
        lp.height = WindowManager.LayoutParams.WRAP_CONTENT
        lp.windowAnimations = R.style.MyWindowAnimation
        //lp.preferredRefreshRate = 30f
        lp.title = "WMSTestActivity 测试窗口"

        windowView = LayoutInflater.from(this).inflate(R.layout.window, null);
        windowManager.addView(windowView, lp)
    }
```

```
    fun removeLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        windowManager.removeView(windowView)
    }
```

## 窗口 token

创建窗口时不设置 token 和 tpye ，系统会有个默认的设置：     
默认的 type：    

```
WindowManager.lava
        public LayoutParams() {
            super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
            // 创建 LayoutParams 时默认设置 type 为 TYPE_APPLICATION。
            type = TYPE_APPLICATION;
            format = PixelFormat.OPAQUE;
        }
```
 
根据 type 来配置默认的 token：     

```
Window.java

    void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
        CharSequence curTitle = wp.getTitle();
        if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            // 如果是子窗口类型
            if (wp.token == null) {
                View decor = peekDecorView();
                if (decor != null) {
                    // 获取 DecorView 的 token
                    wp.token = decor.getWindowToken();
                }
            }
            if (curTitle == null || curTitle.length() == 0) {
                final StringBuilder title = new StringBuilder(32);
                .....
                //设置title
                wp.setTitle(title);
            }
        } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
            // 如果是系统窗口
            // 设置title
                wp.setTitle(title);
            }
        } else {
            // 应用类型的窗口会走到这里
            if (wp.token == null) {
                // 没有配置 token的haul会获取 mAppToken
                wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
            }
            if ((curTitle == null || curTitle.length() == 0)
                    && mAppName != null) {
                wp.setTitle(mAppName);
            }
        }
        if (wp.packageName == null) {
            wp.packageName = mContext.getPackageName();
        }
        if (mHardwareAccelerated ||
                (mWindowAttributes.flags & FLAG_HARDWARE_ACCELERATED) != 0) {
            wp.flags |= FLAG_HARDWARE_ACCELERATED;
        }
    }
```

### 创建子窗口

DecorView 的 mWindowToken 和 WMS 的 WindowState 的 mClient.asBinder() 对应：    

```
View.java
    public IBinder getWindowToken() {
        return mAttachInfo != null ? mAttachInfo.mWindowToken : null;
    }
    
        AttachInfo(IWindowSession session, IWindow window, Display display,
                ViewRootImpl viewRootImpl, Handler handler, Callbacks effectPlayer,
                Context context) {
            mSession = session;
            mWindow = window;
            mWindowToken = window.asBinder();
```

应用侧：    

```
ResumeActivityItem.execute
    ActivityThread.handleResumeActivity
        WindowManagerImpl.addView
            WindowManagerGlobal.addView
                new ViewRootImpl()
                    mWindow = new W(this);
                    new View$AttachInfo(ViewRootImpl.W)
                        mWindowToken = window.asBinder()
                // 把 mWindow 传给服务端
                IWindowSession.addToDisplayAsUser(mWindow)
```

```
static class W extends IWindow.Stub    
```

系统侧：    

```
Session.addToDisplayAsUser(IWindow window)
    WindowManagerService.addWindow(IWindow window)
        new WindowState(IWindow c)
            WindowState.mClient = c
```

寻找父窗口，在 `HashMap<IBinder, WindowState> mWindowMap` 中寻找到对应的 WindowState。    

```
            if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                parentWindow = windowForClientLocked(null, attrs.token, false);
```

type TYPE_APPLICATION_PANEL token 默认 如图：    
<img src="/images/android-window-system-add-window-type-token/sub-window.png" width="411" height="348"/>

### 应用窗口


Window.mAppToken 和 ActivityRecord.token 对应，传递路径：

SystemServer，传递ActivityRecord的token：

```
ActivityTaskSupervisor.realStartActivityLocked(ActivityRecord)
    LaunchActivityItem.obtain(ActivityRecord.token)
        LaunchActivityItem.setValues()
            instance.mActivityToken = activityToken;
```
App 端：

```
LaunchActivityItem()
    LaunchActivityItem.setValues()
        instance.mActivityToken = activityToken;

LaunchActivityItem.execute
    new ActivityClientRecord(mActivityToken)
        this.token = token;
    ActivityThread.handleLaunchActivity(ActivityClientRecord)
        ActivityThread.performLaunchActivity(ActivityClientRecord)
            Activity.attach(ActivityClientRecord.token)
                Activity.mToken = token
                Window.setWindowManager(mToken)
                    mAppToken = appToken;
```

```
    void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
        CharSequence curTitle = wp.getTitle();
        if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            if (wp.token == null) {
                View decor = peekDecorView();
                if (decor != null) {
                    wp.token = decor.getWindowToken();
                }
            }
 
```

```
    fun addLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        var lp = WindowManager.LayoutParams()
        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        //lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG
        lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL
        //lp.token = window.decorView.windowToken
```


mWm.mWindowMap.put(win.mClient.asBinder(), win);     

寻找父窗口，在 DisplayContent 的 `HashMap<IBinder, WindowToken> mTokenMap = new HashMap()` 中寻找到对应的 ActivityRecord。       

```
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
```

type TYPE_APPLICATION token 默认，如图：     
<img src="/images/android-window-system-add-window-type-token/app-window.png" width="428" height="475"/>

### 系统窗口

创建系统窗口是不需要传递 token 就可以的，比如创建一个 type 为 TYPE_APPLICATION_OVERLAY的窗口，它的窗口层级如图：          

通过 WindowManagerPolicy.getWindowLayerFromTypeLw() 找可挂载的对应的层级。    

```
WindowManagerService.addWindow
    WindowToken$Builder.build
        WindowToken()
            DisplayContent.addWindowToken
                DisplayContent.findAreaForToken
                    DisplayContent.findAreaForWindowType
                        DisplayAreaPolicyBuilder$Result.findAreaForWindowType
                            RootDisplayArea.findAreaForWindowTypeInLayer
                DisplayArea.Tokens.addChild(WindowToken) //WindowToken添加到对应层级
    win = new WindowState
    WindowState.attach()
    mWindowMap.put() // 保存WindowState和客户端窗口的映射关系
    win.mToken.addWindow(win)
```

type TYPE_APPLICATION_OVERLAY token 无，如图：     

<img src="/images/android-window-system-add-window-type-token/system-window.png" width="468" height="360"/>

### 定制token


如果想放在非当前的 Activity 下，可以拿到 app 侧用对应的 Activity.getWindow().mAppToken，但是三方应用一般拿不到。     
system 侧使用对应的 activityRecord.token。       
type 就要使用 TYPE_APPLICATION 或者 TYPE_DRAWN_APPLICATION        

或者使用 decorView.windowToken，type使用子窗口，那么新建窗口就会挂载到对应 Activity 层级的 WindowState 下面。       

```
class CommonTestActivity : AppCompatActivity() {
    private var extra = 0

    companion object {
        var windowToken : IBinder? = null
    }
    
    
    windowToken = window.decorView.windowToken
```

```
class CommonTestActivity2 : AppCompatActivity() {

        lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_SUB_PANEL
        lp.token = CommonTestActivity.windowToken
```

type 为TYPE_APPLICATION_SUB_PANEL，token 为其他Activity的 window.decorView.windowToken，如图：       

<img src="/images/android-window-system-add-window-type-token/custom-window.png" width="704" height="384"/>

其实只要能拿到对应的 token，跨进程添加窗口也是可以的。        
添加的进程杀掉后，被添加窗口的进程也会把窗口 remove 掉。     

添加窗口时，该应用添加的所有窗口都会在 server 端保存在 Session 的 mAddedWindows 数组中，当 Session binder 断开时，会remove所有窗口。       

```
onWindowAdded:784, Session (com.android.server.wm)
addWindow:1875, WindowManagerService (com.android.server.wm)
addToDisplayAsUser:260, Session (com.android.server.wm)
onTransact:640, IWindowSession$Stub (android.view)
onTransact:206, Session (com.android.server.wm)
execTransactInternal:1507, Binder (android.os)
execTransact:1451, Binder (android.os)
```

```
// Session.java
    @Override
    public void binderDied() {
        synchronized (mService.mGlobalLock) {
            mCallback.asBinder().unlinkToDeath(this, 0);
            mClientDead = true;
            try {
                for (int i = mAddedWindows.size() - 1; i >= 0; i--) {
                    final WindowState w = mAddedWindows.get(i);
                    Slog.i(TAG_WM, "WIN DEATH: " + w);
                    if (w.mActivityRecord != null && w.mActivityRecord.findMainWindow() == w) {
                        mService.mSnapshotController.onAppDied(w.mActivityRecord);
                    }
                    w.removeIfPossible();
                }
            } finally {
                killSessionLocked();
            }
        }
    }
```





