---
title: Android 虚拟屏
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 虚拟屏
date: 2022-11-23 10:00:00
---



# 概述

介绍虚拟屏的一些相关知识，其实非虚拟屏的知识也可以参考这里。       
本文基于 Android 16。     

# 虚拟屏创建流程

## 创建方法

```
        Surface surface = new Surface();
        DisplayManager displayManager = (DisplayManager) this.getSystemService(DISPLAY_SERVICE);
        mVirtualDisplay = displayManager.createVirtualDisplay(DisplayName, width, height, dpi,surface,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY |
                DisplayManager.VIRTUAL_DISPLAY_FLAG_PRESENTATION |
                DisplayManager.VIRTUAL_DISPLAY_FLAG_TRUSTED |
                DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC ,
                null, null);
        mDisplayId = mVirtualDisplay.getDisplay().getDisplayId();
```

## App流程


```
VirtualDisplay.createVirtualDisplay
  new VirtualDisplayConfig.Builder()
  builder.setFlags()
  builder.setSurface()
  VirtualDisplay.createVirtualDisplay
    DisplayManagerGlobal.createVirtualDisplay()
      new VirtualDisplayCallback()
      // DMS创建虚拟屏，并返回DisplayID
      displayId = DisplayManagerService.createVirtualDisplay()
        //--------> system_server
      DisplayManagerGlobal.createVirtualDisplayWrapper(displayId)
        // 通过DisplayID，取得Display对象信息
        Display display = DisplayManagerGlobal.getRealDisplay(displayId)
          DisplayManagerGlobal.getCompatibleDisplay(displayId)
            DisplayManagerGlobal.getDisplayInfo(displayId)
              DisplayManagerGlobal.getDisplayInfoLocked(displayId)
                mDisplayCache.recompute()
                  DisplayManagerService.getDisplayInfo
                    //--------> system_server
            new Display(this, displayId, displayInfo, daj)
        // 创建VirtualDisplay
        new VirtualDisplay(display)
```

## DMS 创建虚拟屏

```
DisplayManagerService.createVirtualDisplayInternal(virtualDisplayConfig)
  Surface surface = virtualDisplayConfig.getSurface()
  int flags = virtualDisplayConfig.getFlags()
  // 配置Flag
  flags |=  ... flags &=
  String displayUniqueId = VirtualDisplayAdapter.generateDisplayUniqueId()
  displayId = DisplayManagerService.createVirtualDisplayLocked()
    DisplayDevice device = VirtualDisplayAdapter.createVirtualDisplayLocked()
      displayToken = VirtualDisplayAdapter.SurfaceControlDisplayFactory.createDisplay()
        displayToken = DisplayControl.createVirtualDisplay()
          displayToken = nativeCreateVirtualDisplay()
            // -------> jni
      VirtualDisplayDevice device = new VirtualDisplayDevice(displayToken)
    // 发送 DISPLAY_DEVICE_EVENT_ADDED 事件
    DisplayDeviceRepository.onDisplayDeviceEvent(device, DisplayAdapter.DISPLAY_DEVICE_EVENT_ADDED)
      DisplayDeviceRepository.handleDisplayDeviceAdded
        DisplayDeviceRepositorysendEventLocked(device, DISPLAY_DEVICE_EVENT_ADDED)
          LogicalDisplayMapper.sendDisplayEventIfEnabledLocked(display, DisplayManagerGlobal.EVENT_DISPLAY_ADDED)
            LogicalDisplayMapper.handleDisplayDeviceAddedLocked
              LogicalDisplayMapper.updateLogicalDisplaysLocked
                LogicalDisplayMapper.sendUpdatesForDisplaysLocked(LOGICAL_DISPLAY_EVENT_ADDED)
                  DisplayManagerService$LogicalDisplayListener.onLogicalDisplayEventLocked
                    DisplayManagerService.handleLogicalDisplayAddedLocked
                      DisplayManagerService.sendDisplayEventIfEnabledLocked( DisplayManagerGlobal.EVENT_DISPLAY_ADDED)
                        DisplayManagerService.sendDisplayEventLocked
                          Handler.obtainMessage(MSG_DELIVER_DISPLAY_EVENT)
                            DisplayManagerService$DisplayManagerHandler.handleMessage
                              DisplayManagerService.deliverDisplayEvent
                                DisplayManagerService.deliverEventFlagged
                                  DisplayManagerService$CallbackRecord.notifyDisplayEventAsync
                                    DisplayManagerService$CallbackRecord.transmitDisplayEvent
                                      DisplayManagerGlobal$DisplayManagerCallback.onDisplayEvent
                                        DisplayManagerGlobal.handleDisplayEvent
                                          DisplayManagerGlobal$DisplayListenerDelegate.sendDisplayEvent
                                            case EVENT_DISPLAY_ADDED
                                            DisplayManagerGlobal$DisplayListenerDelegate.handleDisplayEventInner
                                              RootWindowContainer.onDisplayAdded
                                                // 创建 DisplayContent
                                                RootWindowContainer.getDisplayContentOrCreate
                                                  DisplayContent.init
                                                    DisplayAreaPolicy$DefaultProvider.instantiate
                                                      // 创建 DefaultTaskDisplayArea
                                                      new TaskDisplayArea()
                                                      DisplayAreaPolicyBuilder.build
                                                        // 构建窗口层级树
                                                        DisplayAreaPolicyBuilder$HierarchyBuilder.build
                                                    DisplayWindowSettings.applySettingsToDisplayLocked
                                                      //根据配置获取DefaultTaskDisplayArea()的WindowingMode
                                                      DisplayWindowSettings.getWindowingModeLocked
                                                      // 设置 DefaultTaskDisplayArea 的WindowingMode
                                                      TaskDisplayArea.setWindowingMode()
                                                        WindowConfiguration.setWindowingMode
                                                RootWindowContainer.startSystemDecorations
                                                  // 启动桌面
                                                  RootWindowContainer.startHomeOnDisplay
                                                    WindowContainer.reduceOnAllTaskDisplayAreas
                                                      DisplayArea.reduceOnAllTaskDisplayAreas
                                                        TaskDisplayArea.reduceOnAllTaskDisplayAreas
                                                          RootWindowContainer.startHomeOnTaskDisplayArea
                                                            // 判断是否需要启动 SecondaryHome
                                                            RootWindowContainer.shouldPlaceSecondaryHomeOnDisplayArea
                                                            // 获取home信息
                                                            RootWindowContainer.resolveSecondaryHomeActivity
                                                            ActivityStartController.startHomeActivity()
```


## 启动桌面

每创建并启动一个屏幕时，原则上都会去启动桌面，但是会有一系列的判断条件来决定当前屏幕是否需要启动桌面。    

```
RootWindowContainer.startHomeOnTaskDisplayArea
  // 判断是否是DefaultTaskDisplayArea
  taskDisplayArea == getDefaultTaskDisplayArea()
  // 判断是否需要启动PrimaryHome
  WindowManagerService.shouldPlacePrimaryHomeOnDisplay()
  // 否则判断是否需要启动 SecondaryHome
  RootWindowContainer.shouldPlaceSecondaryHomeOnDisplayArea
  // 构建启动桌面的 Intent 
  Intent homeIntent =
  RootWindowContainer.canStartHomeOnDisplayArea
    RootWindowContainer.shouldPlacePrimaryHomeOnDisplay
    RootWindowContainer.shouldPlaceSecondaryHomeOnDisplayArea
```

```
    boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
            boolean allowInstrumenting, boolean fromHomeKey) {
        // Fallback to top focused display area if the provided one is invalid.
        if (taskDisplayArea == null) {
            final Task rootTask = getTopDisplayFocusedRootTask();
            taskDisplayArea = rootTask != null ? rootTask.getDisplayArea()
                    : getDefaultTaskDisplayArea();
        }

        // When display content mode management flag is enabled, the task display area is marked as
        // removed when switching from extended display to mirroring display. We need to restart the
        // task display area before starting the home.
        if (ENABLE_DISPLAY_CONTENT_MODE_MANAGEMENT.isTrue()
                && taskDisplayArea.shouldKeepNoTask()) {
            taskDisplayArea.setShouldKeepNoTask(false);
        }

        Intent homeIntent = null;
        ActivityInfo aInfo = null;
        if (taskDisplayArea == getDefaultTaskDisplayArea()
                || mWmService.shouldPlacePrimaryHomeOnDisplay(
                        taskDisplayArea.getDisplayId(), userId)) {
            homeIntent = mService.getHomeIntent();
            aInfo = resolveHomeActivity(userId, homeIntent);
        } else if (shouldPlaceSecondaryHomeOnDisplayArea(taskDisplayArea)) {
            Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, taskDisplayArea);
            aInfo = info.first;
            homeIntent = info.second;
        }

        if (aInfo == null || homeIntent == null) {
            return false;
        }

        if (!canStartHomeOnDisplayArea(aInfo, taskDisplayArea, allowInstrumenting)) {
            return false;
        }

        if (mService.mAmInternal.shouldDelayHomeLaunch(userId)) {
            Slog.d(TAG, "ThemeHomeDelay: Home launch was deferred with user " + userId);
            return false;
        }

        // Updates the home component of the intent.
        homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
        // Updates the extra information of the intent.
        if (fromHomeKey) {
            homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
        }
        homeIntent.putExtra(WindowManagerPolicy.EXTRA_START_REASON, reason);

        // Update the reason for ANR debugging to verify if the user activity is the one that
        // actually launched.
        final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(
                aInfo.applicationInfo.uid) + ":" + taskDisplayArea.getDisplayId();
        mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
                taskDisplayArea);
        return true;
    }
```

# 虚拟屏 VIRTUAL_DISPLAY_FLAG 介绍

安卓投屏之createVirtualDisplay相关flags参数的实战介绍      
https://juejin.cn/post/7420000413321723955     

## Flag 介绍


### VIRTUAL_DISPLAY_FLAG_PUBLIC

公共的 Display，它有如下特性：

 - 允许其他应用程序在 Display 上打开窗口。    
 - 自带了 VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR 标志，在没有内容显示时会显示其他 Display 的镜像。    
 - 如果没有这个标志，那么默认创建的就是 private Display。    

下面来介绍一下 Private Display 的特性：

 - 只有创建该虚拟屏的应用和该 Display 上已有的具有相同 UID 的应用才能在 Private Display 上显示。
 - Private Display 不显示镜像内容，也不许与将自己的内容镜像到其他地方。     

### VIRTUAL_DISPLAY_FLAG_PRESENTATION

创建一个用于演示的Display，应用程序可以自动将其内容投影到演示显示，以提供更丰富的第二屏幕体验。    
可以将 Presentation 组件方便地投屏到虚拟屏。     

```
        Presentation presentation = new Presentation(this, mVirtualDisplay.getDisplay());
        presentation.setContentView(R.layout.window);
        presentation.show();
```

### VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR

 - PUBLIC Display 自带该属性。    
 - 允许 Private Display 在没有内容显示的时候镜像其他 Display 内容。    
 - 它和 VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY 互斥，如果两个同时设置，那么 VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR 不生效，VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY 生效。    
 
### VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY

 - 仅仅显示自己 Display 的内容，不镜像其他屏幕内容。   
 - 一般用在 Public Display 上，因为 Public Display 自带了 VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR Flag，VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY 会帮助去掉 AUTO_MIRROR Flag。    

### VIRTUAL_DISPLAY_FLAG_TRUSTED

加入这个权限后其他第三方应用可以把自己的 Activity 启动显示在这个当前 Display 上面。     


# 相关文章

[安卓投屏之createVirtualDisplay相关flags参数的实战介绍](https://juejin.cn/post/7420000413321723955)    
