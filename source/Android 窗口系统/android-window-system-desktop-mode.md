---
title: Android 桌面模式
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 桌面模式
date: 2022-11-23 10:00:00
---

## 概述

开启桌面模式(DesktopMode)的方法：设置 -> 开发者选项 -> 桌面    

这一个操作打开了两个开关：GLOBAL SETTINGS：force_desktop_mode_on_external_displays 和 GLOBAL SETTINGS：enable_freeform_support。     
force_desktop_mode_on_external_displays 是支持创建虚拟屏幕时可以启动桌面的配置项。     
enable_freeform_support 是支持自由窗口的配置项，桌面模式也是基于自由窗口模式的。     
Android 的桌面模式是需要自由窗口模式来支持的。       
本文基于 Android 16。     

<img src="/images/android-window-system-desktop-mode/1.png" width="589" height="945"/>

## 窗口层级树的差异

在桌面模式下，DefaultTaskDisplayArea 被设置成了 FREEFORM 类型。     

```
   ├─ Display 2 name="TestDisplay" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1896,1080] bounds=[0,0][1896,1080]
   │  ├─ Leaf:32:38 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │  └─ WindowedMagnification:0:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ Leaf:15:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  ├─ WindowToken{d68ea1d type=2024 android.os.BinderProxy@ab9a2f4} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  │  └─ 92f0b19 SecondaryHomeHandle2 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  └─ WindowToken{ad831ca type=2019 android.os.BinderProxy@ba65f6c} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │     └─ 62fa096 NavigationBar2 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ ImePlaceholder:13:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  └─ ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │     └─ WindowToken{cc53cbf type=2011 android.os.Binder@d12f2de} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │        └─ c26a10a InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ Leaf:3:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ DefaultTaskDisplayArea type=undefined mode=FREEFORM override-mode=FREEFORM requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  ├─ Task=146 type=standard mode=FREEFORM override-mode=undefined requested-bounds=[417,103][1031,979] bounds=[417,103][1031,979]
   │     │  │  └─ ActivityRecord{222434144 u0 com.hq.android.androiddemo/.MainActivity t146} type=standard mode=FREEFORM override-mode=undefined requested-bounds=[0,0][0,0] bounds=[417,103][1031,979]
   │     │  │     └─ 10de366 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=FREEFORM override-mode=undefined requested-bounds=[0,0][0,0] bounds=[417,103][1031,979]
   │     │  └─ Task=144 type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │     └─ Task=145 type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │        └─ ActivityRecord{260889411 u0 com.meizu.flyme.launcher/com.android.launcher3.secondarydisplay.SecondaryDisplayLauncher t145} type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │           └─ ed89184 com.meizu.flyme.launcher/com.android.launcher3.secondarydisplay.SecondaryDisplayLauncher type=HOME mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     └─ Leaf:0:1 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │        └─ WallpaperWindowToken{73c97b2 showWhenLocked=false} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │           └─ fd36c55 com.android.systemui.wallpapers.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]

```

```
RootWindowContainer.onDisplayAdded
  RootWindowContainer.getDisplayContentOrCreate
    new DisplayContent
      DisplayWindowSettings.applySettingsToDisplayLocked
        //根据配置获取DefaultTaskDisplayArea()的WindowingMode
        DisplayWindowSettings.getWindowingModeLocked
        TaskDisplayArea.setWindowingMode(WINDOWING_MODE_FREEFORM)
          WindowConfiguration.setWindowingMode
```

```
//DisplayWindowSettings.java
    private int getWindowingModeLocked(@NonNull SettingsProvider.SettingsEntry settings,
            @NonNull DisplayContent dc) {
        final int windowingModeFromDisplaySettings = settings.mWindowingMode;
        // This display used to be in freeform, but we don't support freeform anymore, so fall
        // back to fullscreen.
        if (windowingModeFromDisplaySettings == WindowConfiguration.WINDOWING_MODE_FREEFORM
                && !mService.mAtmService.mSupportsFreeformWindowManagement) {
            return WindowConfiguration.WINDOWING_MODE_FULLSCREEN;
        }
        if (windowingModeFromDisplaySettings != WindowConfiguration.WINDOWING_MODE_UNDEFINED) {
            return windowingModeFromDisplaySettings;
        }
        // No record is present so use default windowing mode policy.
        final boolean forceFreeForm = mService.mAtmService.mSupportsFreeformWindowManagement
                && (mService.mIsPc || dc.isPublicSecondaryDisplayWithDesktopModeForceEnabled());
        // 设置为 WINDOWING_MODE_FREEFORM
        if (forceFreeForm) {
            return WindowConfiguration.WINDOWING_MODE_FREEFORM;
        }
        final int currentWindowingMode = dc.getDefaultTaskDisplayArea().getWindowingMode();
        if (currentWindowingMode == WindowConfiguration.WINDOWING_MODE_UNDEFINED) {
            // No record preset in settings + no mode set via the display area policy.
            // Move to fullscreen as a fallback.
            return WindowConfiguration.WINDOWING_MODE_FULLSCREEN;
        }
        if (currentWindowingMode == WindowConfiguration.WINDOWING_MODE_FREEFORM) {
            // Freeform was enabled before but disabled now, the TDA should now move to fullscreen.
            return WindowConfiguration.WINDOWING_MODE_FULLSCREEN;
        }
        return currentWindowingMode;
    }
```

## 虚拟屏启动桌面

开启桌面模式后，在创建虚拟屏后，会自动启动桌面应用。      


```
RootWindowContainer.onDisplayAdded
  RootWindowContainer.startSystemDecorations
    RootWindowContainer.startHomeOnDisplay
      WindowContainer.reduceOnAllTaskDisplayAreas
        DisplayArea.reduceOnAllTaskDisplayAreas
          TaskDisplayArea.reduceOnAllTaskDisplayAreas
            RootWindowContainer.startHomeOnTaskDisplayArea
              RootWindowContainer.shouldPlaceSecondaryHomeOnDisplayArea
                TaskDisplayArea.canHostHomeTask
                  DisplayContent.isHomeSupported
                    DisplayContent.isSystemDecorationsSupported
                      DisplayContent.isPublicSecondaryDisplayWithDesktopModeForceEnabled
              ActivityStartController.startHomeActivity()
```

下面方法返回true，所以会启动桌面。    

```
//DisplayContent.java
    boolean isPublicSecondaryDisplayWithDesktopModeForceEnabled() {

        if (!mWmService.mForceDesktopModeOnExternalDisplays || isDefaultDisplay || isPrivate()) {
            return false;
        }
        // Desktop mode is not supported on virtual devices.
        int deviceId = mRootWindowContainer.mTaskSupervisor.getDeviceIdForDisplayId(mDisplayId);
        return deviceId == Context.DEVICE_ID_DEFAULT;
    }
```

## 创建工具栏

<img src="/images/android-window-system-desktop-mode/2.png" width="522" height="250"/>

桌面模式的工具栏和自由窗口模式的工具栏是一样的。      
在 Open 动画过程中创建顶部工具栏。      

```
Transitions.onTransitionReady
  Transitions.dispatchReady
    FreeformTaskTransitionObserver.onTransitionReady
      FreeformTaskTransitionObserver.onOpenTransitionReady
        CaptionWindowDecorViewModel.onTaskOpening
          CaptionWindowDecorViewModel.shouldShowWindowDecor
          CaptionWindowDecorViewModel.createWindowDecoration
            CaptionWindowDecoration.relayout
              WindowDecoration.relayout
                WindowDecoration.updateDecorationContainerSurface
                  // 创建 Decor container of Task=，并挂载到 Task 的SurfaceControl上面
                WindowDecoration.updateViewHierarchy
```

## 移除工具栏

```
ShellTaskOrganizer.onTaskInfoChanged
  FullscreenTaskListener.onTaskInfoChanged
    CaptionWindowDecorViewModel.onTaskInfoChanged
      CaptionWindowDecoration.relayout
        WindowDecoration.relayout
          WindowDecoration.releaseViews
```

## 窗口的操作

窗口最大化操作：

```
View.performClick
  CaptionWindowDecorViewModel$CaptionTouchEventListener.onClick
    TaskOperations.maximizeTask
      FreeformTaskTransitionHandler.startWindowingModeTransition
        Transitions.startTransition
          WindowOrganizer.startNewTransition
```

窗口的最大化、最小化以及自由拖动操作其实就是针对自由窗口的操作，这里就不再详细介绍。    

## 在普通设备上开启桌面模式

其实只要应用支持 enable_freeform_support，然后在应用启动时创建 Task 的工具栏就可以了，具体的修改也很简单。    
首先要设置 enable_freeform_support 为true。    
然后修改：


```
// CaptionWindowDecorViewModel.java
    private boolean shouldShowWindowDecor(RunningTaskInfo taskInfo) {
        ......
        final DisplayAreaInfo rootDisplayAreaInfo =
                mRootTaskDisplayAreaOrganizer.getDisplayAreaInfo(taskInfo.displayId);
        if (rootDisplayAreaInfo != null) {
            return true;
//            return rootDisplayAreaInfo.configuration.windowConfiguration.getWindowingMode()
//                    == WINDOWING_MODE_FREEFORM;
        }
```

这样，在主屏幕上打开应用，也支持多窗口模式了。     

## 保存 LaunchParams

每次移动或者修改 Task 窗口的大小，都会把这些信息保存在设备上面，这样下次启动该应用时就会以前面的位置和大小来启动应用。     

```
ActivityStarter.startActivityUnchecked
  ActivityStarter.startActivityInner
    ActivityStarter.setInitialState
      LaunchParamsController.calculate
        LaunchParamsPersister.getLaunchParams
          LaunchParams.mWindowingMode = persistableParams.mWindowingMode
          // 设置 mTmpParams.mBounds
          LaunchParams.mBounds.set()
    ActivityStarter.recycleTask
      ActivityStarter.setTargetRootTaskIfNeeded
        LaunchParamsController.layoutTask
          Task.setBounds
            Task.setBoundsUnchecked
              ConfigurationContainer.setBounds
                WindowContainer.onRequestedOverrideConfigurationChanged
```

在 LaunchParamsController.layoutTask 中 mTmpParams.mBounds 不为空，重新设置了 Task 的区域。    

LaunchParamsPersister 保存在 mLaunchParamsMap 中的 LaunchParams    

```
    LaunchParamsPersister(PersisterQueue persisterQueue, ActivityTaskSupervisor supervisor) {
        this(persisterQueue, supervisor, Environment::getDataSystemCeDirectory);
    }
```
启动参数保存在 `/data/system_ce/0/launch_params`。     
以包名和组件的名称保存：    

```
com.android.settings-.homepage.SettingsHomepageActivity.xml                                   
com.meizu.flyme.myapplication-.MainActivity.xml
com.meizu.flyme.launcher-com.android.launcher3.secondarydisplay.SecondaryDisplayLauncher.xml
```

里面保存了 windowing_mode、bounds 和 display_unique_id等信息。     
从存储中读出来后保存在 PersistableLaunchParams 中：    

```
    private static class PersistableLaunchParams {
        private static final String ATTR_WINDOWING_MODE = "windowing_mode";
        private static final String ATTR_DISPLAY_UNIQUE_ID = "display_unique_id";
        private static final String ATTR_BOUNDS = "bounds";
        private static final String ATTR_WINDOW_LAYOUT_AFFINITY = "window_layout_affinity";
```

保存 LaunchingState：    


```
Task.onConfigurationChanged
  Task.onConfigurationChangedInner
    Task.saveLaunchingStateIfNeeded
      LaunchParamsPersister.saveTask
        PersisterQueue.updateLastOrAddItem
          PersisterQueue.addItem
```

```
PersisterQueue$LazyTaskWriterThread.run
  PersisterQueue.processNextItem
    LaunchParamsPersister$LaunchParamsWriteQueueItem.process
      LaunchParamsPersister$LaunchParamsWriteQueueItem.saveParamsToXml
        LaunchParamsPersister$PersistableLaunchParams.saveToXml
```


在 DesktopMode 下，DefaultTaskDisplayArea 的类型是 FREEFORM，因此，需要保存 LaunchingState。    

```
   ├─ Display 2 name="TestDisplay" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1896,1080] bounds=[0,0][1896,1080]
   │  ├─ Leaf:32:38 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │  └─ WindowedMagnification:0:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ Leaf:15:31 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  ├─ WindowToken{f830a3a type=2024 android.os.BinderProxy@577ed65} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  │  └─ 9879006 SecondaryHomeHandle2 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  └─ WindowToken{3509d0a type=2019 android.os.BinderProxy@f2588ac} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │     └─ 6e37fd6 NavigationBar2 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ ImePlaceholder:13:14 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │  └─ ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │     └─ WindowToken{5a0c099 type=2011 android.os.Binder@24115e0} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     │        └─ e902343 InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ Leaf:3:12 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
   │     ├─ DefaultTaskDisplayArea type=undefined mode=FREEFORM override-mode=FREEFORM requested-bounds=[0,0][0,0] bounds=[0,0][1896,1080]
```

```
    private void saveLaunchingStateIfNeeded(DisplayContent display) {
        if (!isLeafTask()) {
            return;
        }

        if (!getHasBeenVisible()) {
            // Not ever visible to user.
            return;
        }

        final int windowingMode = getWindowingMode();
        if (windowingMode != WINDOWING_MODE_FULLSCREEN
                && windowingMode != WINDOWING_MODE_FREEFORM) {
            return;
        }

        // Don't persist state if Task Display Area isn't in freeform mode. Then the task will be
        // launched back to its last state in a freeform Task Display Area when it's launched in a
        // freeform Task Display Area next time.
        // 普通模式下，这里会返回
        if (getTaskDisplayArea() == null
                || getTaskDisplayArea().getWindowingMode() != WINDOWING_MODE_FREEFORM) {
            return;
        }

        // Saves the new state so that we can launch the activity at the same location.
        mTaskSupervisor.mLaunchParamsPersister.saveTask(this, display);
    }
```










