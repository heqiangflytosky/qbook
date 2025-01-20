---
title: Android 窗口系统-移栈
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口层级移栈的流程
date: 2022-11-23 10:00:00
---


## 概述

前面我们在介绍构建窗口层级的意义时讲过，窗口层级的构建使得系统方便对窗口进行管理，这使得移栈这种操作也简单和便利。     
移动应用进程，其实移动就是就是对应Activity所在的Stack。     
本文就介绍一下窗口在不同的显示设备之间的移动流程。       


## 实现

在开发者选项中打开模拟辅助显示设备，就会开启另一个虚拟屏幕。    
用 `adb shell dumpsys activity containers` 此时的窗口层级情况是：    

```
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  #1 Display 0 name="内置屏幕" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1368,3192] bounds=[0,0][1368,3192]
  ......
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
        #3 Task=117 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
         #0 ActivityRecord{b2468e4 u0 com.hq.android.androiddemo/.MainActivity t117} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
          #0 161f95b com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  ......
  #0 Display 5 name="叠加视图 #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1920,1080] bounds=[0,0][1920,1080]
  ......
     #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
     ......
```

执行移栈命令： `adb shell am display move-stack 117 5`       
窗口层级树为：     

```
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  #1 Display 5 name="叠加视图 #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1920,1080] bounds=[0,0][1920,1080]
  ......
     #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
      #0 Task=117 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
       #0 ActivityRecord{b2468e4 u0 com.hq.android.androiddemo/.MainActivity t117} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
        #0 3bdaa62 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]

  ......
  #0 Display 0 name="内置屏幕" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1368,3192] bounds=[0,0][1368,3192]
  ......
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
```

可见，Task=117 从 Display 0 移动到了 Display 5 。    

## 代码流程

原理其实就是把 Task 容器重新挂载到新的 display 的 TaskDisplayArea 上。    

### 绑定Display和TextureView

前面说过，一个 DisplayContent 代表一个屏幕，那么我们创建虚拟屏幕就会创建一个新的 DisplayContent。    


因为我们手机就一个屏幕，无法用硬件设备来显示新建的 DisplayContent，那么就在手机上创建一个新的 Window 用来显示虚拟屏幕内容。    
承载虚拟屏幕显示的是 OverlayDisplayWindow 类。   
它会在主屏幕上新建一个窗口，布局中的 TextureView 来提供对应的 Surface 来显示对应虚拟屏幕数据。    

```
OverlayDisplayAdapter.registerLocked().ContentObserver
    OverlayDisplayAdapter.updateOverlayDisplayDevices()
        OverlayDisplayAdapter.updateOverlayDisplayDevicesLocked()
            new OverlayDisplayHandle
                OverlayDisplayHandle.showLocked()
                    mShowRunnable.run
                        new OverlayDisplayWindow
                            OverlayDisplayWindow.createWindow()
                                TextureView.setSurfaceTextureListener
                                    SurfaceTextureListener.onSurfaceTextureAvailable
                                        OverlayDisplayHandle.onWindowCreated()
                                            // 通知 SF 创建 Display
                                            displayToken = DisplayControl.createDisplay()
                                                DisplayControl.nativeCreateDisplay
                                            // 创建 DisplayDevice，传入 displayToken 和 surfaceTexture
                                            new OverlayDisplayDevice(displayToken,...,surfaceTexture)
                                            sendDisplayDeviceEventLocked(mDevice, DISPLAY_DEVICE_EVENT_ADDED)
                                                DisplayDeviceRepository.handleDisplayDeviceAdded
                                                    DisplayDeviceRepository.sendEventLocked(device, DISPLAY_DEVICE_EVENT_ADDED)
                                                        LogicalDisplayMapper.handleDisplayDeviceAddedLocked()
                                                            LogicalDisplayMapper.updateLogicalDisplaysLocked()
                                                                LogicalDisplayMapper.sendUpdatesForDisplaysLocked(LOGICAL_DISPLAY_EVENT_ADDED)
                                                                    DisplayManagerService.onLogicalDisplayEventLocked
                                                                        DisplayManagerService.handleLogicalDisplayAddedLocked
                                                                            DisplayManagerService.sendDisplayEventLocked(display, DisplayManagerGlobal.EVENT_DISPLAY_ADDED) // 转下面接收 Message 流程
                        OverlayDisplayWindow.show()
                            DisplayManager.registerDisplayListener
                                
```

createWindow() 时为 TextureView 注册监听，当 onSurfaceTextureAvailable 回调时调用 OverlayDisplayHandle.onWindowCreated() 创建一个 OverlayDisplayDevice，把 SurfaceTexture 作为参数传入。来显示对应 DisplayDevice 的内容。        

```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:background="#000000">
    <TextureView android:id="@+id/overlay_display_window_texture"
               android:layout_width="0px"
               android:layout_height="0px" />
    <TextView android:id="@+id/overlay_display_window_title"
               android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:layout_gravity="top|center_horizontal" />
</FrameLayout>
```


```
//OverlayDisplayAdapter.java

        public void performTraversalLocked(SurfaceControl.Transaction t) {
            if (mSurfaceTexture != null) {
                if (mSurface == null) {
                    // 
                    mSurface = new Surface(mSurfaceTexture);
                }
                setSurfaceLocked(t, mSurface);
            }
        }
    // 绑定 Surface 到 SF 创建的 Display，这样前面创建的 TextureView 就会显示虚拟Display的内容    
    public final void setSurfaceLocked(SurfaceControl.Transaction t, Surface surface) {
        if (mCurrentSurface != surface) {
            mCurrentSurface = surface;
            t.setDisplaySurface(mDisplayToken, surface);
        }
    }
```

### 创建新的 DisplayContent

收到创建虚拟设备消息后，RootWindowContainer 创建新的 DisplayContent

```
DisplayManagerGlobal.DisplayListenerDelegate.handleMessage
    EVENT_DISPLAY_ADDED
    RootWindowContainer.onDisplayAdded
        RootWindowContainer.getDisplayContentOrCreate
            new DisplayContent()
                // 构建层级树，前面已经讲过
                DisplayContent.configureSurfaces
```


### 移栈

```
ActivityManagerShellCommand.runDisplay
    ActivityManagerShellCommand.runDisplayMoveStack
        ActivityTaskManagerService.moveRootTaskToDisplay
            RootWindowContainer.moveRootTaskToDisplay
                // 获取或者创建 DisplayContent
                RootWindowContainer.getDisplayContentOrCreate()
                // 获取 DefaultTaskDisplayArea
                DisplayContent.getDefaultTaskDisplayArea()
                RootWindowContainer.moveRootTaskToTaskDisplayArea
                    Task.reparent(TaskDisplayArea)
                        WindowContainer.reparent(TaskDisplayArea)
                    
```

```
// RootWindowContainer.java
// 参数：rootTaskId：对应被移动的任务栈
// taskDisplayArea：新的承载 stack 的 Display
// onTop：判断stack在new Display的显示层级，默认是显示在最上层

    void moveRootTaskToTaskDisplayArea(int rootTaskId, TaskDisplayArea taskDisplayArea,
            boolean onTop) {
        // 根据 TaskId 获取对应的 Task
        final Task rootTask = getRootTask(rootTaskId);
        if (rootTask == null) {
            throw new IllegalArgumentException("moveRootTaskToTaskDisplayArea: Unknown rootTaskId="
                    + rootTaskId);
        }
        // 获取 Task 当前所在的 TaskDisplayArea
        final TaskDisplayArea currentTaskDisplayArea = rootTask.getDisplayArea();
        if (currentTaskDisplayArea == null) {
            throw new IllegalStateException("moveRootTaskToTaskDisplayArea: rootTask=" + rootTask
                    + " is not attached to any task display area.");
        }

        if (taskDisplayArea == null) {
            throw new IllegalArgumentException(
                    "moveRootTaskToTaskDisplayArea: Unknown taskDisplayArea=" + taskDisplayArea);
        }

        if (currentTaskDisplayArea == taskDisplayArea) {
            throw new IllegalArgumentException("Trying to move rootTask=" + rootTask
                    + " to its current taskDisplayArea=" + taskDisplayArea);
        }
        // 把 Task 挂载到新的 TaskDisplayArea
        rootTask.reparent(taskDisplayArea, onTop);

        // Resume focusable root task after reparenting to another display area.
        rootTask.resumeNextFocusAfterReparent();

        // TODO(multi-display): resize rootTasks properly if moved from split-screen.
    }
```

## 虚拟屏显示内容

我们看到，当虚拟屏里面没有内容显示的时候，它和手机主屏显示的内容是一样的，当把一个task移过去后，它就显示了这个task内容，这部分流程是怎么执行的呢？     
通过分析可以看到，当 mDisplayContent.getLastHasContent() 判断不成立时，就会开启虚拟屏对主屏的录制，此时虚拟屏显示的是主屏的镜像：    

```
RootWindowContainer.performSurfacePlacementNoTrace
    RootWindowContainer.applySurfaceChangesTransaction
        DisplayContent.applySurfaceChangesTransaction
            DisplayContent.updateRecording
                ContentRecorder.updateRecording
                    ContentRecorder.startRecordingIfNeeded
                        SurfaceControl.mirrorSurface
```

```
    @VisibleForTesting void updateRecording() {
        if (isCurrentlyRecording() && (mDisplayContent.getLastHasContent()
                || mDisplayContent.getDisplayInfo().state == Display.STATE_OFF)) {
            pauseRecording();
        } else {
            // Display no longer has content, or now has a surface to write to, so try to start
            // recording.
            startRecordingIfNeeded();
        }
    }
```


对比 SurfaceFlinger Layer 层级：     

<img src="/images/android-window-system-move-task/no-content.png" width="869" height="401" />


<img src="/images/android-window-system-move-task/has-content.png" width="872" height="415" />

如果我们想把录屏显示这里去掉，那么直接把 `startRecordingIfNeeded();` 这里注释掉就行了。      

## 改进


上面的移栈方案是使用命令行进行的，我们可以改进成使用手指拖动到另外一个屏幕，并加上响应的衔接动画来实现。      
参考下面的博客和开源代码：
[Android T多屏多显——应用双屏间拖拽移动功能](https://www.jindouyun.cn/document/industry/details/242008)     
[AndroidT应用双屏间拖拽移动功能实现](https://blog.csdn.net/gitblog_09739/article/details/142943823)     
[gitcode](https://gitcode.com/open-source-toolkit/ca59c/?utm_source=tools_gitcode&index=top&type=card&&isLogin=1)      
[车载多屏互动联动动画版本同屏幕大小情况方案设计--众筹项目](https://blog.csdn.net/learnframework/article/details/130507022)     
[车载多屏互动联动动画版本图层设计--众筹项目](https://blog.csdn.net/learnframework/article/details/130522955)     

