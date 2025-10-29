---
title: Android StartingWindow
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android StartingWindow
date: 2022-11-23 10:00:00
---


## SplashScreen 简介

[Android Development SplashScreen](https://developer.android.google.cn/develop/ui/views/launch/splash-screen?hl=zh-cn)     
Splash Screen 属于 StartingWindow 的一种类型，是 Android 12 版本带来的新功能，它可以让应用启动时显示启动动画。优化从应用启动到真实UI界面显示之前这段时间的显示效果，解决了真正UI显示之前的白屏问题，给用户带来更好的体验。    
SplashScreen 的原理是在启动 Activity 的ActivityRecord 中新加入一个 Window，盖在 Activity 层级之上。Activity 启动完毕后删除该窗口。    
在 ActivityRecord 中有 StartingWindow 相关的成员变量：    

```
public final class ActivityRecord extends WindowToken implements WindowManagerService.AppFreezeListener {

    // starting window 相关的类
    StartingData mStartingData;
    // starting window 窗口
    WindowState mStartingWindow;
    StartingSurfaceController.StartingSurface mStartingSurface;
    boolean startingMoved;
    
    // 获取 StartingWindow 窗口类型
    getStartingWindowType()
```

Android 为 SplashScreen 提供了默认的启动动画，也提供了 API 进行自定义。    

国内厂商一般会把 Android 原生的 SplashScreen 效果去掉了，但是自定义的 SplashScreen 效果一般不会去掉。     
本文基于 Android 14 分析    

### 开启相关日志

下面是打开 WM_SHELL_STARTING_WINDOW 日志方法，因为 WmShell 是在 systemui 进程执行的，所以和 systme_server 中打开 ProtoLog 日志方式不太一样。    

```
adb shell dumpsys activity service SystemUIService WMShell protolog  enable-text WM_SHELL_STARTING_WINDOW
```

### StartingWindow 类型

```
    // 不启动startingwindow，常见于应用内的activity切换
    static final int STARTING_WINDOW_TYPE_NONE = 0;
    // 快照启动窗口，显示的内容为最近一次的可见内容的快照。使用场景如task从后台到前台的切换，屏幕解锁。
    static final int STARTING_WINDOW_TYPE_SNAPSHOT = 1;
    // 闪屏启动窗口，显示的内容时空白的窗口，背景和应用的主题有关。使用场景如应用冷启动
    static final int STARTING_WINDOW_TYPE_SPLASH_SCREEN = 2;
```

### SplashScreen 相关类

WmCore侧:

 - StartingSurfaceController:
 - TaskOrganizerController: 
 - StartingData：startingwindow的数据模型的抽象。抽象了关键的方法：`StartingSurface createStartingSurface(ActivityRecord activity)` 以及 mTypeParams。     
  - SplashScreenStartingData：闪屏类型启动窗口的startingData实现，封装了闪屏类型启动窗口所需要的数据，如theme，icon，windowflags等等，这些数据来源于ActivityRecord。     
  - SnapshotStartingData：快照类型启动窗口的startingData实现，持有了TaskSnapshot。     


WmShell侧:
StartingWindowController:startingwindow的添加和移除最终的调用都在这里     
StartingSurfaceDrawer:startingwindow的添加和移除的实现     
SplashscreenWindowCreator:     
SplashScreenView：     

## StartingWindow Demo

Android sample 里面自带了一个 StartingWindow 的例子。在 `development/samples/StartingWindow/` 目录。    

SplashScreen 相关的配置：        

```
    <style name="BaseAppTheme" parent="Theme.AppCompat.DayNight">
        <!-- Splashscreen Attributes -->
        <item name="android:windowSplashScreenBackground">#ffffff</item>
        <item name="android:windowSplashScreenAnimatedIcon">@drawable/avd_anim</item>

        <!-- Here we want to match the duration of our AVD -->
        <item name="android:windowSplashScreenAnimationDuration">900</item>
```

设置 StartingWindow 动画监听：    

```
    // On Android S, this new method has been added to Activity
    SplashScreen splashScreen = getSplashScreen();

    // Setting an OnExitAnimationListener on the SplashScreen indicates
    // to the system that the application will handle the exit animation.
    // This means that the SplashScreen will be inflated in the application
    // process once the process has started.
    // Otherwise, the splashscreen stays in the SystemUI process and will be
    // dismissed once the first frame of the app is drawn
    splashScreen.setOnExitAnimationListener(this::onSplashScreenExit);
```

根据 StartingWindow 动画完成后传回的 SplashScreenView 来做应用内的动画：    

```
  private void onSplashScreenExit(SplashScreenView view) {
    // At this point the first frame of the application is drawn and
    // the SplashScreen is ready to be removed.

    // It is now up to the application to animate the provided view
    // since the listener is registered
    AnimatorSet animatorSet = new AnimatorSet();
    animatorSet.setDuration(500);

    ObjectAnimator translationY = ObjectAnimator.ofFloat(view, "translationY", 0, view.getHeight());
    translationY.setInterpolator(ACCELERATE);

    ObjectAnimator alpha = ObjectAnimator.ofFloat(view, "alpha", 1, 0);
    alpha.setInterpolator(ACCELERATE);

    // To get fancy, we'll also animate our content
    ValueAnimator marginAnimator = createContentAnimation();

    animatorSet.playTogether(translationY, alpha, marginAnimator);
    animatorSet.addListener(new AnimatorListenerAdapter() {
      @Override
      public void onAnimationEnd(Animator animation) {
        view.remove();
      }
    });

    // If we want to wait for our Animated Vector Drawable to finish animating, we can compute
    // the remaining time to delay the start of the exit animation
    long delayMillis = Instant.now()
          .until(view.getIconAnimationStart().plus(view.getIconAnimationDuration()),
              ChronoUnit.MILLIS);
    view.postDelayed(animatorSet::start, (long) (delayMillis * animationScale));
  }
```

现在来编译安装体验一下 StartingWindow 动画效果。    

```
make StartingWindow
adb install -r out/target/product/qssi_64/system/app/StartingWindow/StartingWindow.apk
```

在 StartingWindow 过程中，使用 `adb shell dumpsys activity containers` 查看对应的窗口层级树，这时候多了一层 `Splash Screen *****` 的层级：    

```
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
        #2 Task=63 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
         #0 ActivityRecord{d1507c7 u0 com.example.android.startingwindow/.CustomizeExitActivity t63} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
          #1 e6528bf Splash Screen com.example.android.startingwindow type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
          #0 bf305fa com.example.android.startingwindow/com.example.android.startingwindow.CustomizeExitActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
        #1 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
```

使用 `adb shell dumpsys window windows` 来看 splash 窗口信息:    

```
  Window #11 Window{2053b1a u0 Splash Screen com.example.android.startingwindow}:
    mDisplayId=0 rootTaskId=81 mSession=Session{cfafdb7 3981:u0a10071} mClient=android.os.BinderProxy@792d5c5
    mOwnerUid=10071 showForAllUsers=true package=com.example.android.startingwindow appop=NONE
    mAttrs={(0,0)(fillxfill) sim={adjust=pan} ty=APPLICATION_STARTING fmt=TRANSLUCENT wanim=0x10302fe
      fl=NOT_FOCUSABLE NOT_TOUCHABLE LAYOUT_IN_SCREEN LAYOUT_INSET_DECOR ALT_FOCUSABLE_IM HARDWARE_ACCELERATED DRAWS_SYSTEM_BAR_BACKGROUNDS
      pfl=SHOW_FOR_ALL_USERS USE_BLAST APPEARANCE_CONTROLLED FIT_INSETS_CONTROLLED
      apr=LIGHT_STATUS_BARS LIGHT_NAVIGATION_BARS
      bhv=DEFAULT
      fitSides=}
    Requested w=1080 h=2340 mLayoutSeq=1823
    mBaseLayer=21000 mSubLayer=0    mToken=ActivityRecord{854720d u0 com.example.android.startingwindow/.CustomizeExitActivity t81}
    mActivityRecord=ActivityRecord{854720d u0 com.example.android.startingwindow/.CustomizeExitActivity t81}
    drawnStateEvaluated=true    mightAffectAllDrawn=true
    mViewVisibility=0x0 mHaveFrame=true mObscured=false
    mGivenContentInsets=[0,0][0,0] mGivenVisibleInsets=[0,0][0,0]
    mFullConfiguration={1.0 ?mcc0mnc [zh_CN] ldltr sw393dp w393dp h792dp 440dpi nrml long hdr widecg port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2340) mAppBounds=Rect(0, 92 - 1080, 2271) mMaxBounds=Rect(0, 0 - 1080, 2340) mDisplayRotation=ROTATION_0 mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} as.5 s.1 fontWeightAdjustment=0}
    mLastReportedConfiguration={1.0 ?mcc0mnc [zh_CN] ldltr sw393dp w393dp h792dp 440dpi nrml long hdr widecg port finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2340) mAppBounds=Rect(0, 92 - 1080, 2271) mMaxBounds=Rect(0, 0 - 1080, 2340) mDisplayRotation=ROTATION_0 mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} as.5 s.1 fontWeightAdjustment=0}
    mHasSurface=true isReadyForDisplay()=true mWindowRemovalAllowed=false
    Frames: parent=[0,0][1080,2340] display=[0,0][1080,2340] frame=[0,0][1080,2340] last=[0,0][1080,2340] insetsChanged=false
     surface=[0,0][0,0]
    WindowStateAnimator{ad15f01 Splash Screen com.example.android.startingwindow}:
      mSurface=Surface(name=Splash Screen com.example.android.startingwindow)/@0x31b9f79
      Surface: shown=true layer=0 alpha=1.0 rect=(0.0,0.0)  transform=(1.0, 0.0, 0.0, 1.0)
      mDrawState=HAS_DRAWN       mLastHidden=false
      mEnterAnimationPending=false      mSystemDecorRect=[0,0][0,0]
    mForceSeamlesslyRotate=false seamlesslyRotate: pending=null    isOnScreen=true
    isVisible=true
    keepClearAreas: restricted=[], unrestricted=[]
    mPrepareSyncSeqId=0
```

Window 类型是 `APPLICATION_STARTING`。


## 添加 StartingWindow

```
ActivityTaskManagerService.startActivity
    ActivityTaskManagerService.startActivityAsUser
        ActivityStarter.execute
            ActivityStarter.executeRequest
                ActivityStarter.startActivityUnchecked
                    ActivityStarter.startActivityInner
                        Task.startActivityLocked
                            StartingSurfaceController.showStartingWindow
                                ActivityRecord.showStartingWindow
                                    ActivityRecord.showStartingWindow
                                        ActivityRecord.addStartingWindow
                                            // 获取 StartingWindow 类型
                                            ActivityRecord.getStartingWindowType()
                                            // 
                                            StartingSurfaceController.makeStartingWindowTypeParameter()
                                            // 如果是 SNAPSHOT 类型就走这里
                                            ActivityRecord.createSnapshot()
                                                // 创建SnapshotStartingData类型的 StartingData
                                                mStartingData = new SnapshotStartingData
                                                // 走 scheduleAddStartingWindow 流程，和下面一样
                                                ActivityRecord.scheduleAddStartingWindow()
                                            // SPLASH_SCREEN 类型 
                                            // 看看是否可以 transfer StartingWindow
                                            ActivityRecord.transferStartingWindow
                                              mStartingData = fromActivity.mStartingData
                                              mStartingSurface = fromActivity.mStartingSurface
                                              mStartingWindow = tStartingWindow
                                              WindowState.reparent()
                                            // 如果不可以就添加 StartingWindow
                                            ActivityRecord.scheduleAddStartingWindow
                                                AddStartingWindow.run()
                                                    SplashScreenStartingData.createStartingSurface
                                                        StartingSurfaceController.createSplashScreenStartingSurface()
                                                            TaskOrganizerController.addStartingWindow()
                                                                ITaskOrganizer.addStartingWindow ----> systemui
```

SystemUI:

```
ITaskOrganizer.Stub().addStartingWindow()
    TaskOrganizer.addStartingWindow()
        StartingWindowController.addStartingWindow()
            // 获取 StartingWindow 的类型
            PhoneStartingWindowTypeAlgorithm.getSuggestedWindowType()
            StartingSurfaceDrawer.addSplashScreenStartingWindow()
                SplashscreenWindowCreator.addSplashScreenStartingWindow()
                    // 生成splashscreen window的配置信息
                    SplashscreenContentDrawer.createLayoutParameters()
                    // 生成 splashscreen 内容
                    SplashscreenContentDrawer.createContentView()
                        // 构造 SplashScreenView 
                        SplashscreenContentDrawer.makeSplashScreenContentView()
                            SplashscreenContentDrawer.getWindowAttrs()
                                //配置 mSplashScreenIcon mIconBgColor 等
                                attrs.mSplashScreenIcon = 
                            // 如果是 STARTING_WINDOW_TYPE_LEGACY_SPLASH_SCREEN
                            // 就获取 Legacy Drawable
                            SplashscreenContentDrawer.peekLegacySplashscreenContent()
                            SplashscreenWindowCreator.SplashScreenViewSupplier.setView()
                    // 添加窗口
                    SplashscreenContentDrawer.addWindow()
                        WindowManagerGlobal.addView()
                            ViewRootImpl.setView()
                                // system_server 开始执行 addWindow 流程
                                WindowSession.addToDisplayAsUser()
                    // 添加 SplashScreenView
                    SplashscreenWindowCreator.SplashScreenViewSupplier.get()
                        rootLayout.addView(contentView)
                        SplashWindowRecord.setSplashScreenView()
```

添加完 StartingWindow 窗口后，StartingWindow 执行 relayout 时，由于 window type 是 TYPE_APPLICATION_STARTING，因此在 WindowState.performShowLocked 方法中会执行 ActivityRecord.onStartingWindowDrawn() 来设置 AppTransition 状态。后面就可以开始做窗口动画了。    

```
WindowSurfacePlacer$Traverser.run
    WindowSurfacePlacer.performSurfacePlacement
        WindowSurfacePlacer.performSurfacePlacementLoop
            RootWindowContainer.performSurfacePlacement
                RootWindowContainer.performSurfacePlacementNoTrace
                    RootWindowContainer.applySurfaceChangesTransaction
                        DisplayContent.applySurfaceChangesTransaction
                            WindowContainer.forAllWindows
                                WindowState.forAllWindows
                                    WindowState.applyInOrderWithImeWindows
                                        DisplayContent.mApplySurfaceChangesTransaction
                                            WindowStateAnimator.commitFinishDrawingLocked
                                                WindowState.performShowLocked
                                                    ActivityRecord.onStartingWindowDrawn()
                                                        DisplayContent.executeAppTransition()
                                                            TransitionController.setReady()
                                                                AppTransition.setReady()
                                                                    AppTransition.setAppTransitionState(APP_STATE_READY)
```

### 获取 StartingWindow 类型

根据条件返回  StartingWindow 类型。    
这个方法是在 `system_server` 端调用，获取的类型通过 `makeStartingWindowTypeParameter` 方法转化为 `TYPE_PARAMETER_**` 保存在 StartingWindowInfo 中传到 SystemUI。     

```
// ActivityRecord.java
    private int getStartingWindowType(boolean newTask, boolean taskSwitch, boolean processRunning,
            boolean allowTaskSnapshot, boolean activityCreated, boolean activityAllDrawn,
            TaskSnapshot snapshot) {
        // A special case that a new activity is launching to an existing task which is moving to
        // front. If the launching activity is the one that started the task, it could be a
        // trampoline that will be always created and finished immediately. Then give a chance to
        // see if the snapshot is usable for the current running activity so the transition will
        // look smoother, instead of showing a splash screen on the second launch.

        if (!newTask && taskSwitch && processRunning && !activityCreated && task.intent != null
                && mActivityComponent.equals(task.intent.getComponent())) {
            final ActivityRecord topAttached = task.getActivity(ActivityRecord::attachedToProcess);
            if (topAttached != null) {
                if (topAttached.isSnapshotCompatible(snapshot)
                        // This trampoline must be the same rotation.
                        && mDisplayContent.getDisplayRotation().rotationForOrientation(
                                getOverrideOrientation(),
                                mDisplayContent.getRotation()) == snapshot.getRotation()) {
                    return STARTING_WINDOW_TYPE_SNAPSHOT;
                }
                // No usable snapshot. And a splash screen may also be weird because an existing
                // activity may be shown right after the trampoline is finished.
                return STARTING_WINDOW_TYPE_NONE;
            }
        }
        final boolean isActivityHome = isActivityTypeHome();
        // 如果是新启动应用/进程为启动/线程切换但是Activity没有创建 等条件，返回 SPLASH_SCREEN 类型
        if ((newTask || !processRunning || (taskSwitch && !activityCreated))
                && !isActivityHome) {
            return STARTING_WINDOW_TYPE_SPLASH_SCREEN;
        }
        // 如果是 Task 切换
        if (taskSwitch) {
            if (allowTaskSnapshot) {
                // 判断是否有合适的快照
                if (isSnapshotCompatible(snapshot)) {
                    // 返回快照类型
                    return STARTING_WINDOW_TYPE_SNAPSHOT;
                }
                if (!isActivityHome) {
                    return STARTING_WINDOW_TYPE_SPLASH_SCREEN;
                }
            }
            if (!activityAllDrawn && !isActivityHome) {
                return STARTING_WINDOW_TYPE_SPLASH_SCREEN;
            }
        }
        return STARTING_WINDOW_TYPE_NONE;
    }
```

在 SystemUI 也会根据 PhoneStartingWindowTypeAlgorithm.getSuggestedWindowType() 再次获取 StartingWindow 类型，为什么要获取两次呢？     
我们来看一下 makeStartingWindowTypeParameter 方法：     

```
    static int makeStartingWindowTypeParameter(boolean newTask, boolean taskSwitch,
            boolean processRunning, boolean allowTaskSnapshot, boolean activityCreated,
            boolean isSolidColor, boolean useLegacy, boolean activityDrawn, int startingWindowType,
            String packageName, int userId) {
        int parameter = 0;
        ......
        if (activityCreated || startingWindowType == STARTING_WINDOW_TYPE_SNAPSHOT) {
            parameter |= TYPE_PARAMETER_ACTIVITY_CREATED;
        }
        if (isSolidColor) {
            parameter |= TYPE_PARAMETER_USE_SOLID_COLOR_SPLASH_SCREEN;
        }
        if (useLegacy) {
            parameter |= TYPE_PARAMETER_LEGACY_SPLASH_SCREEN;
        }
        if (activityDrawn) {
            parameter |= TYPE_PARAMETER_ACTIVITY_DRAWN;
        }
        if (startingWindowType == STARTING_WINDOW_TYPE_SPLASH_SCREEN
                && CompatChanges.isChangeEnabled(ALLOW_COPY_SOLID_COLOR_VIEW, packageName,
                UserHandle.of(userId))) {
            parameter |= TYPE_PARAMETER_ALLOW_HANDLE_SOLID_COLOR_SCREEN;
        }
        return parameter;
    }
```

它把 `STARTING_WINDOW_TYPE_*` 转换成 `TYPE_PARAMETER_ACTIVITY_CREATED` 或者 `TYPE_PARAMETER_ALLOW_HANDLE_SOLID_COLOR_SCREEN`，而在 SystemUI 中调用 `PhoneStartingWindowTypeAlgorithm.getSuggestedWindowType()` 中又会根据 server 传来的参数转换成 `STARTING_WINDOW_TYPE_*` 参数。     

```
    public int getSuggestedWindowType(StartingWindowInfo windowInfo) {
        ......
        final boolean allowTaskSnapshot = (parameter & TYPE_PARAMETER_ALLOW_TASK_SNAPSHOT) != 0;
        final boolean activityCreated = (parameter & TYPE_PARAMETER_ACTIVITY_CREATED) != 0;
        final boolean isSolidColorSplashScreen =
                (parameter & TYPE_PARAMETER_USE_SOLID_COLOR_SPLASH_SCREEN) != 0;
        final boolean legacySplashScreen =
                ((parameter & TYPE_PARAMETER_LEGACY_SPLASH_SCREEN) != 0);
        final boolean activityDrawn = (parameter & TYPE_PARAMETER_ACTIVITY_DRAWN) != 0;
        final boolean windowlessSurface = (parameter & TYPE_PARAMETER_WINDOWLESS) != 0;
        final boolean topIsHome = windowInfo.taskInfo.topActivityType == ACTIVITY_TYPE_HOME;

        ......

        if (windowlessSurface) {
            return STARTING_WINDOW_TYPE_WINDOWLESS;
        }
        if (!topIsHome) {
            if (!processRunning || newTask || (taskSwitch && !activityCreated)) {
                return getSplashscreenType(isSolidColorSplashScreen, legacySplashScreen);
            }
        }

        if (taskSwitch) {
            if (allowTaskSnapshot) {
                if (windowInfo.taskSnapshot != null) {
                    return STARTING_WINDOW_TYPE_SNAPSHOT;
                }
                if (!topIsHome) {
                    return STARTING_WINDOW_TYPE_SOLID_COLOR_SPLASH_SCREEN;
                }
            }
            if (!activityDrawn && !topIsHome) {
                return getSplashscreenType(isSolidColorSplashScreen, legacySplashScreen);
            }
        }
        return STARTING_WINDOW_TYPE_NONE;
    }
```


### 从资源文件获取动画属性

SplashscreenContentDrawer.getWindowAttrs()

```
    private static void getWindowAttrs(Context context, SplashScreenWindowAttrs attrs) {
        final TypedArray typedArray = context.obtainStyledAttributes(
                com.android.internal.R.styleable.Window);
        attrs.mWindowBgResId = typedArray.getResourceId(R.styleable.Window_windowBackground, 0);
        attrs.mWindowBgColor = safeReturnAttrDefault((def) -> typedArray.getColor(
                R.styleable.Window_windowSplashScreenBackground, def),
                Color.TRANSPARENT);
        attrs.mSplashScreenIcon = safeReturnAttrDefault((def) -> typedArray.getDrawable(
                R.styleable.Window_windowSplashScreenAnimatedIcon), null);
        attrs.mBrandingImage = safeReturnAttrDefault((def) -> typedArray.getDrawable(
                R.styleable.Window_windowSplashScreenBrandingImage), null);
        attrs.mIconBgColor = safeReturnAttrDefault((def) -> typedArray.getColor(
                R.styleable.Window_windowSplashScreenIconBackgroundColor, def),
                Color.TRANSPARENT);
        typedArray.recycle();
        ProtoLog.v(ShellProtoLogGroup.WM_SHELL_STARTING_WINDOW,
                "getWindowAttrs: window attributes color: %s, replace icon: %b",
                Integer.toHexString(attrs.mWindowBgColor), attrs.mSplashScreenIcon != null);
    }
```

SplashscreenContentDrawer.createLayoutParameters

```
    static WindowManager.LayoutParams createLayoutParameters(Context context,
            StartingWindowInfo windowInfo,
            @StartingWindowInfo.StartingWindowType int suggestType,
            CharSequence title, int pixelFormat, IBinder appToken) {
        // 
        final WindowManager.LayoutParams params = new WindowManager.LayoutParams(
                WindowManager.LayoutParams.TYPE_APPLICATION_STARTING);
        params.setFitInsetsSides(0);
        params.setFitInsetsTypes(0);
        params.format = pixelFormat;
        int windowFlags = WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED
                | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                | WindowManager.LayoutParams.FLAG_LAYOUT_INSET_DECOR;
        final TypedArray a = context.obtainStyledAttributes(R.styleable.Window);
        if (a.getBoolean(R.styleable.Window_windowShowWallpaper, false)) {
            windowFlags |= WindowManager.LayoutParams.FLAG_SHOW_WALLPAPER;
        }
        if (suggestType == STARTING_WINDOW_TYPE_LEGACY_SPLASH_SCREEN) {
            if (a.getBoolean(R.styleable.Window_windowDrawsSystemBarBackgrounds, false)) {
                windowFlags |= WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS;
            }
        } else {
            windowFlags |= WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS;
        }
        params.layoutInDisplayCutoutMode = a.getInt(
                R.styleable.Window_windowLayoutInDisplayCutoutMode,
                params.layoutInDisplayCutoutMode);
        params.windowAnimations = a.getResourceId(R.styleable.Window_windowAnimationStyle, 0);
        a.recycle();

        final ActivityManager.RunningTaskInfo taskInfo = windowInfo.taskInfo;
        final ActivityInfo activityInfo = windowInfo.targetActivityInfo != null
                ? windowInfo.targetActivityInfo
                : taskInfo.topActivityInfo;
        final int displayId = taskInfo.displayId;
        // Assumes it's safe to show starting windows of launched apps while
        // the keyguard is being hidden. This is okay because starting windows never show
        // secret information.
        // TODO(b/113840485): Occluded may not only happen on default display
        if (displayId == DEFAULT_DISPLAY && windowInfo.isKeyguardOccluded) {
            windowFlags |= WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
        }

        // Force the window flags: this is a fake window, so it is not really
        // touchable or focusable by the user.  We also add in the ALT_FOCUSABLE_IM
        // flag because we do know that the next window will take input
        // focus, so we want to get the IME window up on top of us right away.
        // Touches will only pass through to the host activity window and will be blocked from
        // passing to any other windows.
        windowFlags |= WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM;
        params.flags = windowFlags;
        params.token = appToken;
        params.packageName = activityInfo.packageName;
        params.privateFlags |= WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS;

        if (!context.getResources().getCompatibilityInfo().supportsScreen()) {
            params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
        }

        params.setTitle("Splash Screen " + title);
        return params;
    }
```

## 删除 StartingWindow

当 Activity 的 WindowState 第一帧绘制完成后，它的widow type 为 TYPE_BASE_APPLICATION，因此在 WindowState.performShowLocked() 方法中会执行ActivityRecord.onFirstWindowDrawn() 执行删除逻辑。      

```
WindowSurfacePlacer$Traverser.run
    WindowSurfacePlacer.performSurfacePlacement
        WindowSurfacePlacer.performSurfacePlacementLoop
            RootWindowContainer.performSurfacePlacement
                RootWindowContainer.performSurfacePlacementNoTrace
                    RootWindowContainer.applySurfaceChangesTransaction
                        DisplayContent.applySurfaceChangesTransaction
                            WindowContainer.forAllWindows
                                WindowState.forAllWindows
                                    WindowState.applyInOrderWithImeWindows
                                        DisplayContent.mApplySurfaceChangesTransaction
                                            WindowStateAnimator.commitFinishDrawingLocked
                                                WindowState.performShowLocked
                                                    ActivityRecord.onFirstWindowDrawn
                                                        ActivityRecord.removeStartingWindow
                                                            ActivityRecord.transferSplashScreenIfNeeded()
                                                                ActivityRecord.requestCopySplashScreen()
                                                                    TaskOrganizerController.copySplashScreenView()
                                                                        // copy SplashScreenView 给应用
                                                                        ITaskOrganizer.copySplashScreenView()  ----> systemui
                                                            ActivityRecord.removeStartingWindowAnimation
                                                                StartingSurfaceController$StartingSurface.remove
                                                                    TaskOrganizerController.removeStartingWindow
                                                                        // 删除 StartingWindow
                                                                        ITaskOrganizer.removeStartingWindow()  ----> systemui
```

SystemUI:

```
ITaskOrganizer.Stub().removeStartingWindow
    TaskOrganizer.removeStartingWindow()
        ShellTaskOrganizer.removeStartingWindow()
            StartingWindowController.removeStartingWindow()
                StartingSurfaceDrawer.removeStartingWindow()
                    StartingSurfaceDrawer.StartingWindowRecordManager.removeWindow()
                        SplashscreenWindowCreator.SplashWindowRecord.removeIfPossible()
                            SplashscreenWindowCreator.removeWindowInner()
                                WindowManagerGlobal.removeView()
                                    WindowManagerGlobal.removeViewLocked()
                                        ViewRootImpl.die()
                                            // 异步执行
                                            ViewRootImpl.doDie()
                                                // system_server 执行删除window流程
                                                WindowSession.remove()  ----> system_server
```

copySplashScreenView

Systemui:

```
ITaskOrganizer.Stub().copySplashScreenView()
    ShellTaskOrganizer.copySplashScreenView()
        StartingWindowController.copySplashScreenView()
            StartingSurfaceDrawer.copySplashScreenView()
                SplashscreenWindowCreator.copySplashScreenView(taskId)
                    ActivityTaskManager.getInstance().onSplashScreenViewCopyFinished()
                        IActivityTaskManager.onSplashScreenViewCopyFinished()
```

system_server:

```
ActivityTaskManagerService.onSplashScreenViewCopyFinished()
    ActivityRecord.onCopySplashScreenFinish()
        TransferSplashScreenViewStateItem.obtain()
        ClientLifecycleManager.scheduleTransaction()
```

App:

```
TransferSplashScreenViewStateItem.execute()
    ActivityThread.handleAttachSplashScreenView()
        ActivityThread.createSplashScreen()
            SplashScreenView view = builder.createFromParcel(parcelable).build()
            OnPreDrawListener.onPreDraw()
                ActivityThread.syncTransferSplashscreenViewTransaction()
                    ActivityThread.reportSplashscreenViewShown()
                        SplashScreen.SplashScreenManagerGlobal.handOverSplashScreenView()
                            SplashScreen.SplashScreenManagerGlobal.dispatchOnExitAnimation()
                                ExitAnimationListener.onSplashScreenExit(SplashScreenView view)
```

## 关于 Transfer StartingWindow


```
// Task.java
    void startActivityLocked(ActivityRecord r, @Nullable Task topTask, boolean newTask,
            boolean isTaskSwitch, ActivityOptions options, @Nullable ActivityRecord sourceRecord) {
        .......
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            // 在同一个Task内找到前面还在做Starting动画的Activity prev
            Task baseTask = r.getTask();
            final ActivityRecord prev = baseTask.getActivity(
                    a -> a.mStartingData != null && a.showToCurrentUser());
            // mStartingData 不为 null 表示正在做 Starting 动画
            mWmService.mStartingSurfaceController.showStartingWindow(r, prev, newTask,
                    isTaskSwitch, sourceRecord);
        }
    }
```

ActivityRecord.transferStartingWindow() 方法会 fromActivity 的 StartingWindow 添加到当前的 ActivityRecord。      

```
// ActivityRecord.java
    private boolean transferStartingWindow(@NonNull ActivityRecord fromActivity) {
        final WindowState tStartingWindow = fromActivity.mStartingWindow;
        if (tStartingWindow != null && fromActivity.mStartingSurface != null) {
            if (tStartingWindow.getParent() == null) {
                // The window has been detached from the parent, so the window cannot be transfer
                // to another activity because it may be in the remove process.
                // Don't need to remove the starting window at this point because that will happen
                // at #postWindowRemoveCleanupLocked
                return false;
            }
            // Do not transfer if the orientation doesn't match, redraw starting window while it is
            // on top will cause flicker.
            if (fromActivity.getRequestedConfigurationOrientation()
                    != getRequestedConfigurationOrientation()) {
            // In this case, the starting icon has already been displayed, so start
            // letting windows get shown immediately without any more transitions.
            if (fromActivity.mVisible) {
                mDisplayContent.mSkipAppTransitionAnimation = true;
            }

            ProtoLog.v(WM_DEBUG_STARTING_WINDOW, "Moving existing starting %s"
                    + " from %s to %s", tStartingWindow, fromActivity, this);

            final long origId = Binder.clearCallingIdentity();
            try {
                // Link the fixed rotation transform to this activity since we are transferring the
                // starting window.
                if (fromActivity.hasFixedRotationTransform()) {
                    mDisplayContent.handleTopActivityLaunchingInDifferentOrientation(this,
                            false /* checkOpening */);
                }

                // Transfer the starting window over to the new token.
                mStartingData = fromActivity.mStartingData;
                mStartingSurface = fromActivity.mStartingSurface;
                mStartingWindow = tStartingWindow;
                reportedVisible = fromActivity.reportedVisible;
                fromActivity.mStartingData = null;
                fromActivity.mStartingSurface = null;
                fromActivity.mStartingWindow = null;
                fromActivity.startingMoved = true;
                tStartingWindow.mToken = this;
                tStartingWindow.mActivityRecord = this;

                if (mStartingData.mRemoveAfterTransaction == AFTER_TRANSITION_FINISH) {
                    mStartingData.mRemoveAfterTransaction = AFTER_TRANSACTION_IDLE;
                }

                ProtoLog.v(WM_DEBUG_ADD_REMOVE,
                        "Removing starting %s from %s", tStartingWindow, fromActivity);
                mTransitionController.collect(tStartingWindow);
                tStartingWindow.reparent(this, POSITION_TOP);

                // Clear the frozen insets state when transferring the existing starting window to
                // the next target activity.  In case the frozen state from a trampoline activity
                // affecting the starting window frame computation to see the window being
                // clipped if the rotation change during the transition animation.
                tStartingWindow.clearFrozenInsetsState();

                // Propagate other interesting state between the tokens. If the old token is
                // displayed, we should immediately force the new one to be displayed. If it is
                // animating, we need to move that animation to the new one.
                if (fromActivity.allDrawn) {
                    allDrawn = true;
                }
                if (fromActivity.firstWindowDrawn) {
                    firstWindowDrawn = true;
                }
                if (fromActivity.isVisible()) {
                    setVisible(true);
                    setVisibleRequested(true);
                    mVisibleSetFromTransferredStartingWindow = true;
                }
                setClientVisible(fromActivity.isClientVisible());

                if (fromActivity.isAnimating()) {
                    transferAnimation(fromActivity);

                    // When transferring an animation, we no longer need to apply an animation to
                    // the token we transfer the animation over. Thus, set this flag to indicate
                    // we've transferred the animation.
                    mTransitionChangeFlags |= FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT;
                } else if (mTransitionController.getTransitionPlayer() != null) {
                    // In the new transit system, just set this every time we transfer the window
                    mTransitionChangeFlags |= FLAG_STARTING_WINDOW_TRANSFER_RECIPIENT;
                }
                // Post cleanup after the visibility and animation are transferred.
                fromActivity.postWindowRemoveStartingWindowCleanup(tStartingWindow);
                fromActivity.mVisibleSetFromTransferredStartingWindow = false;

                mWmService.updateFocusedWindowLocked(
                        UPDATE_FOCUS_WILL_PLACE_SURFACES, true /*updateInputWindows*/);
                getDisplayContent().setLayoutNeeded();
                mWmService.mWindowPlacerLocked.performSurfacePlacement();
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

## 去除系统默认 SPLASH_SCREEN

可以在 `SplashscreenWindowCreator.addSplashScreenStartingWindow()` 和 `SplashscreenContentDrawer.makeSplashScreenContentView()` 方法的流程中实现。        
在 SplashscreenWindowCreator.addSplashScreenStartingWindow() 方法中根据是否设置了自定义的 Splash Icon 来决定是不是 STARTING_WINDOW_TYPE_LEGACY_SPLASH_SCREEN 类型，而在 SplashscreenContentDrawer.makeSplashScreenContentView() 根据是否是 STARTING_WINDOW_TYPE_LEGACY_SPLASH_SCREEN 决定是否使用 Legacy Drawable。      

## 关于热启动

热启动的 StartingWindow 和冷启动流程类似，不同的有以下几点：   

 - 构造 SnapshotStartingData
 - 调用 StartingSurfaceController.createTaskSnapshotSurface() 构造 StartingSurface

## 相关文章

[Android T 启动窗口(startingwindow)流程梳理 ](https://juejin.cn/post/7365351534601994303)






