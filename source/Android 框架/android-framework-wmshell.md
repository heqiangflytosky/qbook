---
title: WM Shell 架构
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 WM Shell 架构
date: 2022-11-23 10:00:00
---

## 概述

我看先来看一下 Android Bootcamp 2020 对 WM Shell 架构的描述：    

<img src="/images/android-framework-wmshell/0.png" width="613" height="345"/>

WM Core 属于 WMS 的一部分，运行在 system server 进程，主要源码位于 `frameworks/base/services/core` 中。    
WM Core 主要包含窗口策略相关的代码。

WM Shell 是把一些和窗口动画和UI等相关的代码从 WMS 中剥离出来，以动态库的形式提供给 SystemUI，运行在 SystemUI 进程。    
比如：SplitScreen、StartingWindow，PIP等。    
代码在 `frameworks/base/libs/WindowManager/Shell/`    

```
//Android.bp
android_library {
    name: "WindowManager-Shell",
    srcs: [
        ":wm_shell_protolog_src",
        // TODO(b/168581922) protologtool do not support kotlin(*.kt)
        ":wm_shell-sources-kt",
        ":wm_shell-aidls",
    ],
```

编译成动态库被 SystemUI 引用：

SystemUI/Android.bp:

```
    static_libs: [
        "WifiTrackerLib",
        "WindowManager-Shell",
```

我们可以通过下面命令来查看 WMShell 的一些信息：    

```
adb shell dumpsys activity service SystemUIService WMShell
```


打开 WMShell 日志，比如 STARTING_WINDOW 相关日志：    

```
adb shell dumpsys activity service SystemUIService WMShell protolog  enable-text WM_SHELL_STARTING_WINDOW
```

相关 TAG 在 `frameworks/base/libs/WindowManager/Shell/src/com/android/wm/shell/protolog/ShellProtoLogGroup.java` 去看。    


## 相关类

WMShell

TaskOrganizer:ActivityTaskManager/WindowManager委派任务控制的接口，主要用于监听Task变化响应。提供一系列 Task 操作的 API，比如：onTaskAppeared，onTaskInfoChanged，addStartingWindow，removeStartingWindow，copySplashScreenView等。    
在TaskOrganizer中定义了一个变量：mInterface，该变量的类型为ITaskOrganizer：    

```
    private final ITaskOrganizer mInterface = new ITaskOrganizer.Stub() {
        @Override
        public void addStartingWindow(StartingWindowInfo windowInfo) {
            mExecutor.execute(() -> TaskOrganizer.this.addStartingWindow(windowInfo));
        }

        @Override
        public void removeStartingWindow(StartingWindowRemovalInfo removalInfo) {
            mExecutor.execute(() -> TaskOrganizer.this.removeStartingWindow(removalInfo));
        }

        @Override
        public void copySplashScreenView(int taskId)  {
            mExecutor.execute(() -> TaskOrganizer.this.copySplashScreenView(taskId));
        }

        @Override
        public void onAppSplashScreenViewRemoved(int taskId) {
            mExecutor.execute(() -> TaskOrganizer.this.onAppSplashScreenViewRemoved(taskId));
        }

        @Override
        public void onTaskAppeared(ActivityManager.RunningTaskInfo taskInfo, SurfaceControl leash) {
            mExecutor.execute(() -> TaskOrganizer.this.onTaskAppeared(taskInfo, leash));
        }

        .......
    };
```

在执行 TaskOrganizer.registerOrganizer() 时会把 mInterface 传递到服务端的 TaskOrganizerController，保存在 mTaskOrganizers 列表中，实现服务端和应用端的进程间通信。    

ShellTaskOrganizer：WMShell 中对 Task 的统一管理器。    

```
@startuml
WindowOrganizer <|-- DisplayAreaOrganizer
WindowOrganizer <|-- TaskOrganizer
WindowOrganizer <|-- TaskFragmentOrganizer
TaskOrganizer <|-- ShellTaskOrganizer
@enduml
```


WMCore:
WindowOrganizerController:服务端用来组织管理窗口的实现; 实现来自 shell 侧的窗口调用。    

TaskOrganizerController:用于存储与给定窗口关联的 ITaskOrganizer 及其关联状态；实现 Server 到 Shell 的调用。        
mTaskOrganizers 存储 ITaskOrganizer。
TaskOrganizerState:提供了add、remove、dispose等操作，用于管理Task，然后将Task的状态信息通过TaskOrganizerCallbacks上报到对应的TaskOrganizer；    


## 进程间通信

在执行 Shell 端 ITaskOrganizer 创建好后，TaskOrganizer.registerOrganizer() 时会把 mInterface 传递到服务端的 TaskOrganizerController，保存在 mTaskOrganizers 列表中，实现服务端到应用端的进程间通信。    

Shell:

```
ShellInterfaceImpl.onInit()
    ShellController.handleInit()
        ShellTaskOrganizer.onInit()
            ShellTaskOrganizer.registerOrganizer()
                TaskOrganizer.registerOrganizer()
                    ITaskOrganizerController.registerTaskOrganizer()
```

Server:

```
TaskOrganizerController.onTransact()
    TaskOrganizerController.registerTaskOrganizer()
        mTaskOrganizers.add()
```

而 TaskOrganizer 构造时会通过进程间通信拿到 TaskOrganizerController，这样可以实现由 Shell 到 Core 的进程间调用。    

```
    private ITaskOrganizerController getController() {
        try {
            return getWindowOrganizerController().getTaskOrganizerController();
        } catch (RemoteException e) {
            return null;
        }
    }
```

WindowOrganizer 中也提供了 getWindowOrganizerController() 来实现 Client 到 Server 的调用。    

```
    static IWindowOrganizerController getWindowOrganizerController() {
        return IWindowOrganizerControllerSingleton.get();
    }
```

## 相关文章

https://www.bilibili.com/opus/949475161829015593
