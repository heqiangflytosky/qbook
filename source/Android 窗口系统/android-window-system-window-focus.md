---
title: Android 窗口焦点
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口焦点
date: 2022-11-23 10:00:00
---

## 整体流程

Focus App的设置是从 WMS-> InputFlinger，Focus Window的设置是WMS -> SurfaceFlinger -> InputFlinger。     

<img src="/images/android-window-system-window-focus/focus-flow.png" width="932" height="1137"/>

## 查看焦点窗口

我们通过 WMS 、SurfaceFlinger和input端的日志都可以看窗口焦点的情况，WMS 侧的 mCurrentFocus 只是表面当前 surface 已经创建好，可以作为一个焦点窗口，具体能不能接收焦点，还要等 SurfaceFlinger 画好后再传导给 inputflinger 才能具体接收焦点。      

`adb shell dumpsys window displays`    

```
WINDOW MANAGER DISPLAY CONTENTS (dumpsys window displays)
  Display: mDisplayId=0 (organized)
    init=1080x2340 440dpi mMinSizeOfResizeableTaskDp=220 cur=1080x2340 app=1080x2179 rng=1080x988-2248x2179
    deferred=false mLayoutNeeded=false mTouchExcludeRegion=SkRegion((0,0,1080,2340))
......
  mCurrentFocus=Window{5c4591 u0 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity}
  mFocusedApp=ActivityRecord{b3a1ec9 u0 com.hq.android.androiddemo/.MainActivity t82}
......
```

这里打印的是 DisplayContent 中的下面两个变量：    

```
    /**
     * Window that is currently interacting with the user. This window is responsible for receiving
     * key events and pointer events from the user.
     */
    WindowState mCurrentFocus = null;

    /**
     * The foreground app of this display. Windows below this app cannot be the focused window. If
     * the user taps on the area outside of the task of the focused app, we will notify AM about the
     * new task the user wants to interact with.
     */
    ActivityRecord mFocusedApp = null;
```

分别是 WindowState 和 ActivityRecord 对象。    


`adb shell dumpsys SurfaceFlinger` 中查看 HWC Layer没看到 [Focused] 这一列中带 [*] 号的对应的是焦点窗口。    

```
Display 4630947064936706947 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Layer Class | Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity#694
  rel      0 |            1 |            0 |     DEVICE |          0 |    0    0 1080 2340 |    0.0    0.0 1080.0 2340.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
```

`adb shell dumpsys input`    

```
  FocusedDisplayId: 0
  FocusedApplications:
    displayId=0, name='ActivityRecord{b3a1ec9 u0 com.hq.android.androiddemo/.MainActivity t82}', dispatchingTimeout=5000ms
  FocusedWindows:
    displayId=0, name='5c4591 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity'
  FocusRequests:
    displayId=0, name='5c4591 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity' result='OK'

```

event log:    

```
adb logcat -b events |grep input
```


```
input_focus: [Focus leaving ......  //InputDispatcher::dispatchFocusLocked 中打印
// WMS 中打印，
input_focus: [Focus request ......
// sf 中打印，如果打印这个日志，说明focus已经到了input了。
input_focus: [Focus entering ...... //InputDispatcher::dispatchFocusLocked 中打印
```

ProtoLog：

查看 WindowManager 相关日志。    

```
adb shell wm logging enable-text WM_DEBUG_FOCUS WM_DEBUG_FOCUS_LIGHT 
```

如果需要结合 App 启动流程查看，还可以加上 `WM_DEBUG_APP_TRANSITIONS WM_DEBUG_APP_TRANSITIONS_ANIM WM_DEBUG_ANIM`

## 窗口焦点更新

焦点的更新是首先要设置焦点 App（mFocusedApp），然后再设置 焦点窗口（mCurrentFocus）。    
焦点窗口的更新是通过 `WindowManagerService.updateFocusedWindowLocked()` 进行，更新焦点窗口有不同的时机，这个时机通过 `WindowManagerService.updateFocusedWindowLocked()` 的第一个参数体现出来：    

```
    boolean updateFocusedWindowLocked(int mode, boolean updateInputWindows) {
```

具体定义在：    

```
    static final int UPDATE_FOCUS_NORMAL = 0;
    /** Caller will assign layers */
    static final int UPDATE_FOCUS_WILL_ASSIGN_LAYERS = 1;
    /** Caller is performing surface placement */
    static final int UPDATE_FOCUS_PLACING_SURFACES = 2;
    /** Caller will performSurfacePlacement */
    static final int UPDATE_FOCUS_WILL_PLACE_SURFACES = 3;
    /** Indicates we are removing the focused window when updating the focus. */
    static final int UPDATE_FOCUS_REMOVING_FOCUS = 4;
```

而焦点 App 的设置主要是通过 `DisplayContent.setFocusedApp(ActivityRecord newFocus)`，所以，焦点窗口和焦点 APP 可能是不属于同一个应用的。    

### 窗口焦点更新流程

我们先来看一下 `WindowManagerService.updateFocusedWindowLocked()`，然后再看触发更新的时机。    

```
WindowManagerService.updateFocusedWindowLocked
    RootWindowContainer.updateFocusedWindowLocked
        // 针对每个DisplayContent，进行焦点窗口更新
        DisplayContent.updateFocusedWindowLocked
            DisplayContent.findFocusedWindowIfNeeded()
                DisplayContent.findFocusedWindow()
                    DisplayContent.forAllWindows()
                        ......
                        WindowState.applyInOrderWithImeWindows.callback.apply()
                            mFindFocusedWindow.apply()
                                'Looking for focus:' //日志
                                WindowState.canReceiveKeys()
                                    WindowState.isVisibleRequestedOrAdding()
                                    // ActivityRecord 是否可以获得焦点
                                    ActivityRecord.windowsAreFocusable()
                                    // 是否可以接收 input 事件
                                    Task.cantReceiveTouchInput()
            `Changing focus from` // 日志
            // 更新焦点窗口
            mCurrentFocus = newFocus
            //通知DisplayPolicy焦点窗口发生变化
            DisplayPolicy.focusChangedLw()
            BackNavigationController.onFocusChanged()
            // 下面根据不同的mode执行不同动作 -->
            DisplayContent.performLayout
            DisplayContent.assignWindowLayers
            // <-----
            //向InputMonitor中设置焦点窗口
            InputMonitor.setInputFocusLw()
                `Input focus has changed to` // 日志
                InputMonitor.setUpdateInputWindowsNeededLw()
                // 更新焦点窗口，这个方法在很多地方调用
                InputMonitor.updateInputWindowsLw()
                    InputMonitor.scheduleUpdateInputWindows()
                        UpdateInputWindows.run()
                            // 核心的工作在这个方法里面做
                            InputMonitor.UpdateInputForAllWindowsConsumer.updateInputWindows()
                                DisplayContent.forAllWindows(callback)
                                    InputMonitor.UpdateInputForAllWindowsConsumer.accept()
                                        // 将WindowState的部分属性填充给inputWindowHandle
                                        InputMonitor.populateInputWindowHandle
                                        // InputWindowHandle通过SurfaceFlinger设置给了InputDispatcher
                                        InputMonitor.setInputWindowInfoIfNeeded()
                                            InputWindowHandleWrapper.applyChangesToSurface
                                                SurfaceControl.Transaction.setInputWindowInfo()  --> surfaceFlinger
                                                    ----->jni
                                                    nativeSetInputWindowInfo
                                                        android_view_SurfaceControl::nativeSetInputWindowInfo
                                                            android_view_InputWindowHandle_getHandle()
                                                            NativeInputWindowHandle.updateInfo
                                                            // 设置 windowinfo 等待 ApplyTransaction 时传递给 surfaceFlinger 
                                                            SurfaceComposerClient::Transaction::setInputWindowInfo()
                                                                SurfaceComposerClient::Transaction::getLayerState()
                                                                layer_state_t->windowInfoHandle = new WindowInfoHandle(info)
                                InputMonitor.updateInputFocusRequest()
                                    // 如果焦点已经设置给 App，但是窗口还没有绘制好，那么就需要先将 mInputFocus 置空
                                    `Requesting to set focus to null window` // 日志
                                    // 
                                    InputMonitor.requestFocus()
                                        mInputFocus = focusToken;
                                        SurfaceControl.Transaction.setFocusedWindow() --> surfaceFlinger
                                        'Focus request ****' // event 日志
                                        'Focus requested for window' // WindowManager 日志
            // 调整ime窗口
            DisplayContent.adjustForImeIfNeeded()
            DisplayContent.updateKeepClearAreas()
            DisplayContent.scheduleToastWindowsTimeoutIfNeededLocked()
        // 更新焦点屏幕
        PhoneWindowManager.setTopFocusedDisplay()
```

wms --> surfaceFlinger

```
android_view_SurfaceControl::nativeApplyTransaction
    SurfaceComposerClient::Transaction::apply()
        sf->setTransactionState --> surfaceFlinger
```

#### 判断是否可以接收事件    

```
    public boolean canReceiveKeys(boolean fromUserTouch) {
        if (mActivityRecord != null && mTransitionController.shouldKeepFocus(mActivityRecord)) {
            // During transient launch, the transient-hide windows are not visibleRequested
            // or on-top but are kept focusable and thus can receive keys.
            return true;
        }
        final boolean canReceiveKeys = isVisibleRequestedOrAdding()
                && (mViewVisibility == View.VISIBLE) && !mRemoveOnExit
                && ((mAttrs.flags & WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE) == 0)
                && (mActivityRecord == null || mActivityRecord.windowsAreFocusable(fromUserTouch))
                // can it receive touches
                && (mActivityRecord == null || mActivityRecord.getTask() == null
                        || !mActivityRecord.getTask().getRootTask().shouldIgnoreInput());

        if (!canReceiveKeys) {
            return false;
        }
        // Do not allow untrusted virtual display to receive keys unless user intentionally
        // touches the display.
        return fromUserTouch || getDisplayContent().isOnTop()
                || getDisplayContent().isTrusted();
    }
```

从上面方法中可以看到一个 WindowState 可以接收 input 事件，需要满足下面的条件：    

 - isVisibleRequestedOrAdding() 为true，表示WindowState可见或处于添加过程中。
 - mViewVisibility 为 View.VISIBLE
 - mRemoveOnExit 表示设置了退出动画后从窗口管理器中完全删除该窗口
 - mActivityRecord.windowsAreFocusable 表示 ActivityRecord 可以获得焦点
 - Task.cantReceiveTouchInput() 表示 是否可以接收 Touch 事件。

应用切换过程中，判断是否可以接收事件主要在 isVisibleRequestedOrAdding() 方法中的 isVisibleRequested() 条件，下面来看一下 mVisibleRequested 变量的设置流程：      

```
ActivityClientController.activityPaused
    ActivityRecord.activityPaused
        TaskFragment.completePause
            RootWindowContainer.ensureActivitiesVisible
                DisplayContent.ensureActivitiesVisible
                    WindowContainer.forAllRootTasks
                        Task.forAllRootTasks // 新启动应用所在 Task
                            Task.ensureActivitiesVisible
                                Task.forAllLeafTasks
                                    TaskFragment.updateActivityVisibilities
                                        EnsureActivitiesVisibleHelper.process
                                            EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                                EnsureActivitiesVisibleHelper.makeVisibleAndRestartIfNeeded
                                                    ActivityRecord.setVisibility(true)
                                                        `setAppVisibility` // 日志
                                                        ActivityRecord.setVisibleRequested(true)
                                                            WindowContainer.setVisibleRequested(true)
                                                                mVisibleRequested = true
                        Task.forAllRootTasks // 桌面所在 Task
                            Task.ensureActivitiesVisible
                                Task.forAllLeafTasks
                                    TaskFragment.updateActivityVisibilities
                                        EnsureActivitiesVisibleHelper.process
                                            EnsureActivitiesVisibleHelper.setActivityVisibilityState
                                                EnsureActivitiesVisibleHelper.makeInvisible
                                                    ActivityRecord.setVisibility(false)
                                                        `setAppVisibility` // 日志
                                                        ActivityRecord.setVisibleRequested(false)
                                                            WindowContainer.setVisibleRequested(false)
                                                                mVisibleRequested = false
```




```

```

#### 核心切换逻辑

```
        private void updateInputWindows(boolean inDrag) {
```

#### 焦点切换

我们通过下面的命令来打开焦点相关的 ProtoLog，来看一下 DisplayContent.updateFocusedWindowLocked 中的相关日志打印，从而看一下在冷启动一个应用时焦点窗口的切换。    

```
adb shell wm logging enable-text WM_DEBUG_FOCUS WM_DEBUG_FOCUS_LIGHT
```

```
D WindowManager: Changing focus from Window{da4ca87 u0 com.***.launcher/com.android.launcher3.uioverrides.QuickstepLauncher} to null displayId=0 Callers=com.android.server.wm.RootWindowContainer.updateFocusedWindowLocked:473 com.android.server.wm.WindowManagerService.updateFocusedWindowLocked:6235

....

D WindowManager: Changing focus from null to Window{7c0dadb u0 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity} displayId=0 Callers=com.android.server.wm.RootWindowContainer.updateFocusedWindowLocked:473 com.android.server.wm.WindowManagerService.updateFocusedWindowLocked:6235 com.android.server.wm.WindowManagerService.relayoutWindow:2506 com.android.server.wm.Session.relayout:250 
```

这个过程中，执行了两次 `DisplayContent.updateFocusedWindowLocked` ，两次焦点切换分别从 Launcher 窗口  -->  null --> App 窗口。    

相关event 日志打印：    
```
I input_focus: [Focus leaving da4ca87 com.**.launcher/com.android.launcher3.uioverrides.QuickstepLauncher (server),reason=NO_WINDOW]
I input_focus: [Focus request 1103c8 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity,reason=UpdateInputWindows]
I input_focus: [Focus entering 1103c8 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity (server),reason=Window became focusable. Previous reason: NOT_VISIBLE]
```

那么为什么经历了一次焦点为 null 的过程呢？我们来看一下第一次 `Changing focus from` 日志之前的 `Looking for focus:` 此时所有的窗口都无法获取焦点，`canReceive=false`，切换为 App 窗口焦点之前的 `Changing focus from` 日志里面，App 窗口可以获得焦点，因此寻找焦点窗口成功，我们来看看两次不同的日志情况：    

```
D WindowManager: addWindow: win=Window{2b0fdce u0 Splash Screen com.hq.android.androiddemo} Callers=com.android.server.wm.ActivityRecord.addWindow:4543 com.android.server.wm.WindowManagerService.addWindow:1793 com.android.server.wm.Session.addToDisplayAsUser:215 android.view.IWindowSession$Stub.onTransact:621 com.android.server.wm.Session.onTransact:179
......
V WindowManager: Looking for focus: Window{2b0fdce u0 Splash Screen com.hq.android.androiddemo}, flags=-2130509544, canReceive=false, reason=fromTouch= false isVisibleRequestedOrAdding=true mViewVisibility=0 mRemoveOnExit=false flags=-2130509544 appWindowsAreFocusable=true canReceiveTouchInput=true displayIsOnTop=true displayIsTrusted=true transitShouldKeepFocus=false

.....

D WindowManager: addWindow: win=Window{7c0dadb u0 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity} Callers=com.android.server.wm.ActivityRecord.addWindow:4543 com.android.server.wm.WindowManagerService.addWindow:1793 com.android.server.wm.Session.addToDisplayAsUser:215 android.view.IWindowSession$Stub.onTransact:621 com.android.server.wm.Session.onTransact:179 

.....
V WindowManager: Looking for focus: Window{2b0fdce u0 Splash Screen com.hq.android.androiddemo}, flags=-2130509544, canReceive=false, reason=fromTouch= false isVisibleRequestedOrAdding=true mViewVisibility=0 mRemoveOnExit=false flags=-2130509544 appWindowsAreFocusable=true canReceiveTouchInput=true displayIsOnTop=true displayIsTrusted=true transitShouldKeepFocus=false

V WindowManager: Looking for focus: Window{7c0dadb u0 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity}, flags=-2122252032, canReceive=true, reason=fromTouch= false isVisibleRequestedOrAdding=true mViewVisibility=0 mRemoveOnExit=false flags=-2122252032 appWindowsAreFocusable=true canReceiveTouchInput=true displayIsOnTop=true displayIsTrusted=true transitShouldKeepFocus=false
01-02 09:46:54.064  2685  9220 V WindowManager: findFocusedWindow: Found new focus @ Window{7c0dadb u0 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity}
```

前面的寻找焦点窗口时，真正的 APP 窗口还没有添加，此时只是添加了 StartingWindow 窗口，那么当然找不到可以接收焦点的窗口了，后面一次时，App Window 已经添加，`canReceive=true,` 就找到了真正可以接收焦点的窗口了。      

### SurfaceFlinger 端相关流程

前面在介绍 《Android Input 系统简介》 相关知识的时候我们也提到过，上层 WMS 中的窗口信息是会同步到 SurfaceFlinger 侧的，每次 vSync 信号来临时都会调用 updateInputFlinger 方法，若满足一定条件则会根据上层WMS传来的信息来更新 SurfaceFlinger 侧的窗口信息。因此，底层 InputDispatcher 中的窗口列表信息是从 SurfaceFlinger 处更新而来的。    
SurfaceFlinger 中通过调用 buildWindowInfo 来更新 Layer 信息并构建新的窗口列表，然后通过 WindowInfosListener 的 windowInfosChanged 方法将新的窗口列表信息同步至 InputDispatcher 中。     

这里我们就简单介绍一下焦点更新流程，具体参考前面的文章。    
这里涉及到 SurfaceFlinger 和 inputFlinger 的通信，一种是通过 inputFlinger 调用，SurfaceFlinger 在 bootFinished() 方法中初始化了 inputFlinger：     

```
    sp<IBinder> input(defaultServiceManager()->getService(String16("inputflinger")));

    static_cast<void>(mScheduler->schedule([=]() FTL_FAKE_GUARD(kMainThreadContext) {
        if (input == nullptr) {
            ALOGE("Failed to link to input service");
        } else {
            mInputFlinger = interface_cast<os::IInputFlinger>(input);
        }
```

另外一种是通过 WindowInfosListenerInvoker(SurfaceFlinger端)通知 WindowInfosListenerReporter(inputFlinger 端)，他们的关系建立在 InputDispatcher 初始化的时候，具体可以参考前面input相关文章：    

```
InputDispatcher::InputDispatcher(InputDispatcherPolicyInterface& policy,
                                 std::chrono::nanoseconds staleEventTimeout)
                                 {
    // 初始化mWindowInfoListener
    mWindowInfoListener = sp<DispatcherWindowListener>::make(*this);
#if defined(__ANDROID__)
    //向SurfaceComposerClient注册mWindowInfoListener
    SurfaceComposerClient::getDefault()->addWindowInfosListener(mWindowInfoListener);
#endif
```


回调注册流程：   

```
**************************** system_server 进程
SurfaceComposerClient.addwindowInfosListener
    WindowInfosListenerReporter.addwindowInfosListener
**************************** surfaceflinger 进程
SurfaceFlinger.addWindowInfosListener
    WindowInfosListenerInvoker.addWindowInfosListener
```

焦点更新流程：    

wms -- > SurfaceFlinger

```
SurfaceFlinger::setTransactionState
    TransactionHandler::queueTransaction
        mLocklessTransactionQueue.push
    SurfaceFlinger::setTransactionFlags
        Scheduler::modulateVsync
        SurfaceFlinger::scheduleCommit
            
```


```
SurfaceFlinger::commit
    SurfaceFlinger::updateInputFlinger
        SurfaceFlinger::buildWindowInfos
        WindowInfosListenerInvoker.windowInfosChanged
            WindowInfosListenerInvoker::windowInfosChanged()
                WindowInfosListenerReporter::onWindowInfosChanged  --> inputFlinger
        InputManager::setFocusedWindow  --> inputFlinger
```

### inputFlinger

sf--> inputFlinger

```
WindowInfosListenerReporter::onWindowInfosChanged
    InputDispatcher::onWindowInfosChanged
        InputDispatcher::setInputWindowsLocked
            FocusResolver.setInputWindows()
                FocusResolver::getFocusRequest()
                FocusResolver::getFocusedWindowToken()
                FocusResolver::updateFocusedWindow()
            InputDispatcher::onFocusChangedLocked
                mPolicy.notifyFocusChanged --> IMS InputManagerService.notifyFocusChanged
```

sf--> inputFlinger

```
InputDispatcher::setFocusedWindow
    FocusResolver.setFocusedWindow()
        //容器中保存
        mFocusRequestByDisplay[displayId] = request
        mLastFocusResultByDisplay[displayId] = result
    InputDispatcher::onFocusChangedLocked
        InputDispatcher::enqueueFocusEventLocked
            InboundQueue.insert(EventEntry::Type::FOCUS)
                InputDispatcher::dispatchFocusLocked
                    `Focus entering` 或 `Focus leaving` // event 日志
                    // 分发给 App
                    InputDispatcher::dispatchEventLocked()    --> App
```


### WMS

```
InputManagerService.notifyFocusChanged
    InputManagerCallback.notifyFocusChanged
        WindowManagerService.reportFocusChanged
            lastTarget = getInputTargetFromToken(oldToken)
            newTarget = getInputTargetFromToken(newToken);
            `Focus changing:` //日志 
            mFocusedInputTarget = newTarget;
            AnrController.onFocusChanged()
            WindowState.reportFocusChangedSerialized()
                
```

## 焦点 App 切换时机

对于焦点 App 的切换，是从 WMS 直接到 inputFlinger，并没有经过 SurfaceFlinger。    

冷启动：    

```
ActivityManagerService.attachApplication
    ActivityManagerService.attachApplicationLocked
        ActivityManagerService.finishAttachApplicationInner
            ActivityTaskManagerService$LocalService.attachApplication
                RootWindowContainer.attachApplication
                    RootWindowContainer$AttachApplicationHelper.process
                        WindowContainer.forAllRootTasks
                            RootWindowContainer$AttachApplicationHelper.accept
                                WindowContainer.forAllActivities
                                    RootWindowContainer$AttachApplicationHelper.test
                                        ActivityTaskSupervisor.realStartActivityLocked
                                            Task.minimalResumeActivityLocked
                                                ActivityRecord.setState(RESUMED)
                                                    TaskFragment.onActivityStateChanged
                                                        TaskFragment.setResumedActivity
                                                            ActivityTaskSupervisor.updateTopResumedActivityIfNeeded
                                                                ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                                                    DisplayContent.setFocusedApp
                                                                        `setFocusedApp` // 日志
                                                                        mFocusedApp = newFocus
                                                                        InputMonitor.setFocusedAppLw()
                                                                            InputManagerService.setFocusedApplication()
                                                                                NativeInputManagerService.setFocusedApplication()
                                                                                    ----------jni----------
                                                                                    nativeSetFocusedApplication()
                                                                                        NativeInputManager::setFocusedApplication
                                                                                            InputDispatcher::setFocusedApplication()
                                                                                                InputDispatcher::setFocusedApplicationLocked()
                                                                                                    mFocusedApplicationHandlesByDisplay[displayId] = inputApplicationHandle
                                                                                                    // 重置 ANR 计时
                                                                                                    InputDispatcher::resetNoFocusedWindowTimeoutLocked()
                                                                                                    
```


热启动：    

```
ActivityTaskManagerService.startActivity
    ActivityTaskManagerService.startActivityAsUser
        ActivityStarter.execute
            ActivityStarter.executeRequest
                ActivityStarter.startActivityUnchecked
                    ActivityStarter.startActivityInner
                        ActivityStarter.recycleTask
                            ActivityStarter.setTargetRootTaskIfNeeded
                                Task.moveTaskToFront
                                    ActivityRecord.moveFocusableActivityToTop
                                        ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                            DisplayContent.setFocusedApp
                                                `setFocusedApp` // 日志
                                                mFocusedApp = newFocus
                                                InputMonitor.setFocusedAppLw()
                                                    InputManagerService.setFocusedApplication()
                                                        NativeInputManagerService.setFocusedApplication()
                                                            ----------jni----------
                                                            nativeSetFocusedApplication()
                                                                NativeInputManager::setFocusedApplication
                                                                    InputDispatcher::setFocusedApplication()
                                                                        InputDispatcher::setFocusedApplicationLocked()
                                                                            mFocusedApplicationHandlesByDisplay[displayId] = inputApplicationHandle
                                                                            
```

应用内启动 Activity ：    

```
ActivityClientController.activityPaused
    ActivityRecord.activityPaused
        TaskFragment.completePause
            RootWindowContainer.resumeFocusedTasksTopActivities
                RootWindowContainer.resumeFocusedTasksTopActivities
                    Task.resumeTopActivityUncheckedLocked
                        Task.resumeTopActivityInnerLocked
                            TaskFragment.resumeTopActivity
                                ActivityTaskSupervisor.startSpecificActivity
                                    ActivityTaskSupervisor.realStartActivityLocked
                                        Task.minimalResumeActivityLocked
                                            ActivityRecord.setState
                                                TaskFragment.onActivityStateChanged
                                                    TaskFragment.setResumedActivity
                                                        ActivityTaskSupervisor.updateTopResumedActivityIfNeeded
                                                            ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                                                DisplayContent.setFocusedApp
```

## 焦点窗口更新时机

```
ActivityTaskManagerService.startActivity()
    ActivityTaskManagerService.startActivityAsUser()
        ActivityStartController.obtainStarter()
        ActivityStarter.execute()
            ActivityStarter$Request.resolveActivity() //解析启动请求参数
            ActivityStarter.executeRequest()
                ActivityStarter.startActivityUnchecked()
                    ActivityStarter.startActivityInner
                        ActivityStarter.recycleTask
                            ActivityStarter.setTargetRootTaskIfNeeded
                                Task.moveTaskToFront
                                    ActivityRecord.moveFocusableActivityToTop
                                        ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                            WindowManagerService.updateFocusedWindowLocked(UPDATE_FOCUS_NORMAL)
```


```
Session.relayout
    WindowManagerService.relayoutWindow
        WindowManagerService.updateFocusedWindowLocked(UPDATE_FOCUS_NORMAL)
```

```
ActivityClientController.activityPaused
    ActivityRecord.activityPaused
        TaskFragment.completePause
            RootWindowContainer.resumeFocusedTasksTopActivities
                Task.resumeTopActivityUncheckedLocked
                    Task.resumeTopActivityInnerLocked
                        TaskFragment.resumeTopActivity
                            ActivityTaskSupervisor.startSpecificActivity
                                ActivityTaskSupervisor.realStartActivityLocked
                                    Task.minimalResumeActivityLocked
                                        ActivityRecord.setState(RESUMED)
                                            TaskFragment.onActivityStateChanged
                                                TaskFragment.setResumedActivity
                                                    ActivityTaskSupervisor.updateTopResumedActivityIfNeeded
                                                        ActivityTaskManagerService.setLastResumedActivityUncheckLocked
                                                            WindowManagerService.updateFocusedWindowLocked(UPDATE_FOCUS_NORMAL)
```

```
Session.addToDisplayAsUser
    WindowManagerService.addWindow
        WindowManagerService.updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS)
```

```
Session.remove
    WindowManagerService.removeWindow
        WindowState.removeIfPossible
            WindowState.setupWindowForRemoveOnExit
                WindowManagerService.updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES)
```




## 相关文章

[Android R WindowManagerService模块(5) 焦点窗口和InputWindows的更新](https://juejin.cn/post/6955857985151336484)      
[Android焦点之SurfaceFlinger传递给InputFinger](https://blog.csdn.net/wenwang88/article/details/140401230)      
