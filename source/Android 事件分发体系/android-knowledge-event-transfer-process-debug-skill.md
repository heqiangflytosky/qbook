---
title: Android Input事件相关Debug技巧
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: Android Input事件相关Debug技巧
date: 2022-8-31 10:00:00
---

## Event 日志


input_cancel：当有cancle事件发出时，就会打印出 input_cancel 日志，可以看到接受cancel的window以及cancel的原因。

```
input_cancel: [de8f568 NotificationShade (server),reason=transferring touch focus from this window to another window]
```

input_focus：当输入焦点发生变化时，会打印出input_focus日志，可以看到焦点发生变化的window以及原因。

```
input_focus: [Focus request 3309b96 NotificationShade,reason=UpdateInputWindows]
input_focus: [Focus leaving a2b17b9 com.android.settings/com.android.settings.Settings (server),reason=setFocusedWindow]
input_focus: [Focus entering 3309b96 NotificationShade (server),reason=setFocusedWindow]

```

input_interaction：事件流转过哪些window。

```
input_interaction: Interaction with: a2b17b9 com.android.settings/com.android.settings.Settings (server), [Gesture Monitor] swipe-up (server), [Gesture Monitor] edge-swipe (server), PointerEventDispatcher0 (server), 
```

```
input_interaction: Interaction with: 9845e1f com.meizu.flyme.launcher/com.android.launcher3.uioverrides.QuickstepLauncher (server), [Gesture Monitor] swipe-up (server), [Gesture Monitor] edge-swipe (server), 145a397 com.meizu.flyme.sdkstage.wallpaper.video.m2191.VideoWallpaperV1 (server), 3309b96 NotificationShade (server), PointerEventDispatcher0 (server), 
```

view_enqueue_input_event:

```

```

## Main 日志


InputManager-JNI：input JNI 层相关日志。    
InputEventReceiver:    
SystemGesture-XXXXX:SystemGesture相关日志，比如 SystemGesture-GestureManager，GestureManager-MzPhoneWindowManager等等。    


## Dumpsys

`adb shell dumpsys input` 通过dump查看触控事件处理系统框架部分：EventHub、InputReader、InputDispatcher的工作状态：    

InputDispatcher：    

## 开发者选项输入相关

打开 开发者选项 -> 触摸事件 开关，就可以额外获取一些触摸事件日志。对应的其实就是 `sys.inputlog.enabled` 系统属性，也可以通过命令来打开和关闭：`adb shell setprop sys.inputlog.enabled true`    

InputTransport：    

打开 开发者选项 -> 指针位置 开关，触摸屏幕能看到小白点实时显示，能够主观感受屏幕滑动跟手度等状况。屏幕上会显示当前的触控数据    
打开 开发者选项 -> 显示点按操作反馈 开关，点击屏幕能看到点按操作的视觉反馈。    

## 其他上层日志开关

需要动态或者静态的打开Log 开关，通过log 来分析定位。    

比如我们在 InputDispatcher::finishDispatchCycleLocked() 方法中会看到下面的日志开关：    
```
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
                                                const std::shared_ptr<Connection>& connection,
                                                uint32_t seq, bool handled, nsecs_t consumeTime) {
    if (DEBUG_DISPATCH_CYCLE) {
        ALOGD("channel '%s' ~ finishDispatchCycle - seq=%u, handled=%s",
              connection->getInputChannelName().c_str(), seq, toString(handled));
    }
```

DEBUG_DISPATCH_CYCLE 定义在 `frameworks/native/services/inputflinger/dispatcher/DebugConfig.h` 中：    

```
/**
 * Log detailed debug messages about each outbound event processed by the dispatcher.
 * Enable this via "adb shell setprop log.tag.InputDispatcherOutboundEvent DEBUG" (requires restart)
 */
const bool DEBUG_OUTBOUND_EVENT_DETAILS =
        __android_log_is_loggable(ANDROID_LOG_DEBUG, LOG_TAG "OutboundEvent", ANDROID_LOG_INFO);

/**
 * Log debug messages about the dispatch cycle.
 * Enable this via "adb shell setprop log.tag.InputDispatcherDispatchCycle DEBUG" (requires restart)
 */
const bool DEBUG_DISPATCH_CYCLE =
        __android_log_is_loggable(ANDROID_LOG_DEBUG, LOG_TAG "DispatchCycle", ANDROID_LOG_INFO);

.........

/**
 * Log debug messages about hover events.
 * Enable this via "adb shell setprop log.tag.InputDispatcherHover DEBUG" (requires restart)
 */
const bool DEBUG_HOVER =
        __android_log_is_loggable(ANDROID_LOG_DEBUG, LOG_TAG "Hover", ANDROID_LOG_INFO);

```

类似有很多这种日志，可以通过 `adb shell setprop log.tag.InputDispatcherOutboundEvent DEBUG` 打开，然后重启 system_server 进程即可生效。


```
// frameworks/native/services/inputflinger/reader/Macros.cpp 
DEBUG_RAW_EVENTS  
DEBUG_VIRTUAL_KEYS DEBUG_POINTERS DEBUG_POINTER_ASSIGNMENT 
DEBUG_GESTURES DEBUG_VIBRATOR DEBUG_STYLUS_FUSION 

//frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
DEBUG_INPUT_READER_POLICY 
DEBUG_INPUT_DISPATCHER_POLICY

//frameworks/native/services/inputflinger/dispatcher/inputDispatcher.cpp
DEBUG_FOCUS
DEBUG_INJECTION
```

开启ViewRootImpl/View/ViewGroup中input event的处理过程的log开关。 去确认以下怀疑点：Input 事件传递到了哪个window？ 有没有被正确的view处理？有没有被drop?等等。    

```
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java  
DEBUG_INPUT = true;

//frameworks/base/core/java/android/view/KeyEvent.java 
DEBUG = true;

//frameworks/base/core/java/android/view/ViewRootImpl.java 
DEBUG_INPUT_RESIZE = true;
DEBUG_INPUT_STAGES = true;

//frameworks/base/core/java/android/view/ViewDebug.java
DEBUG_POSITIONING = true;

//PhoneWindowManagerInjectImpl.java
    /**
     * 使用命令
     * adb shell dumpsys window -p
     * 打开或者关闭 DEBUG_INPUT
     */
    private static /*final*/ boolean DEBUG_INPUT = false;
```

## 查看支持的触控输入设备节点

```
# cd /dev/input
/dev/input # ls -l
total 0
crw-rw---- 1 root input 13,  64 1970-07-12 04:58 event0
crw-rw---- 1 root input 13,  65 1970-07-12 04:58 event1
crw-rw---- 1 root input 13,  66 1970-07-12 04:58 event2
crw-rw---- 1 root input 13,  67 1970-07-12 04:58 event3
crw-rw---- 1 root input 13,  68 1970-07-12 04:58 event4
crw-rw---- 1 root input 13,  69 1970-07-12 04:58 event5
crw-rw---- 1 root input 13,  70 1970-07-12 04:58 event6
crw-rw---- 1 root input 13,  71 1970-07-12 04:58 event7
crw-rw---- 1 root input 13,  72 1970-07-12 04:58 event8
```

## adb shell input

`adb shell input [<source>] [-d DISPLAY_ID] <command> [<arg>...]` 具体命令可以通过 `adb shell input -h`     
比如 `adb shell input motionevent DOWN 400 800` 可以模拟屏幕(400,800)点处的Down事件。    
`adb shell input keyevent key-value` 可以模拟进行按键的点击，将点击事件直接通过InputDispatcher的injectInputEvent方法发送到Native层，若这边有问题表示InputDispatcher的拦截和分发存在问题。

## adb shell getevent 命令查看屏幕报点情况

adb shell getevent -ltr 可以滑动查看屏幕触控报点是否正常和均匀：     

```
$ adb shell getevent -ltr 
add device 1: /dev/input/event8
  name:     "kalama-mtp-snd-card USBC Jack"
add device 2: /dev/input/event7
  name:     "main_touch"
add device 3: /dev/input/event6
  name:     "ndt"
add device 4: /dev/input/event5
  name:     "pmic_resin"
add device 5: /dev/input/event4
  name:     "pmic_pwrkey"
add device 6: /dev/input/event3
  name:     "pmic_pwrkey"
add device 7: /dev/input/event2
  name:     "qcom-hv-haptics"
add device 8: /dev/input/event1
  name:     "qbt_key_input"
add device 9: /dev/input/event0
  name:     "gpio-keys"
0[  615041.773137] /dev/input/event7: EV_ABS       ABS_MT_TRACKING_ID   000011bb            
[  615041.773137] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    00000382            
[  615041.773137] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000338            
[  615041.773137] /dev/input/event7: EV_ABS       ABS_MT_TOUCH_MAJOR   0000002c            
[  615041.773137] /dev/input/event7: EV_KEY       BTN_TOUCH            DOWN                
[  615041.773137] /dev/input/event7: EV_SYN       SYN_REPORT           00000000            
[  615041.826618] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    00000383            
[  615041.826618] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 18
[  615041.834138] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    0000038d            
[  615041.834138] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000335            
[  615041.834138] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 132
[  615041.842614] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    0000039c            
[  615041.842614] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000331            
[  615041.842614] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 117
[  615041.851267] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003ad            
[  615041.851267] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    0000032c            
[  615041.851267] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 115
[  615041.859259] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003b9            
[  615041.859259] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    0000032a            
[  615041.859259] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 125
[  615041.867622] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003c2            
[  615041.867622] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000328            
[  615041.867622] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 119
[  615041.875947] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003c9            
[  615041.875947] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 120
[  615041.884493] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003cf            
[  615041.884493] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 117
[  615041.892968] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003d2            
[  615041.892968] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000327            
[  615041.892968] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 117
[  615041.901355] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003d4            
[  615041.901355] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 119
[  615041.909625] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003d6            
[  615041.909625] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 120
[  615041.934326] /dev/input/event7: EV_ABS       ABS_MT_POSITION_X    000003d9            
[  615041.934326] /dev/input/event7: EV_ABS       ABS_MT_POSITION_Y    00000326            
[  615041.934326] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 40
[  615041.942456] /dev/input/event7: EV_ABS       ABS_MT_TRACKING_ID   ffffffff            
[  615041.942456] /dev/input/event7: EV_KEY       BTN_TOUCH            UP                  
[  615041.942456] /dev/input/event7: EV_SYN       SYN_REPORT           00000000             rate 123
```



## Systrace上查看触控事件分发

InputDispatcher进行事件分发是会有一些处理队列，源码都加了一些trace tag的的打印和计数，如InputReader读取到触控事件后唤醒InputDispatcher会放入“iq”队列中，然后进行事件分发时每个目标窗口都有对应的队列“oq”和等待目标窗口事件处理的“wq”队列，最后应用这边收到触控事件后还有对应的“aq”队列，从systrace上看如下图所示：    
system_server进程的InputDispatcher和InputReader：    

<img src="/images/android-knowledge-event-transfer-process-debug-skill/system_server_input_dispatcher.jpg" width="633" height="119"/>

InputReader 从EventHub 中读取屏幕驱动上报的Input触控事件，并唤醒交给InputDispatcher线程进行分发。    
InputDispatcher 被唤醒后，先对Input触控事件进行封装，然后寻找到当前前台焦点窗口，并将Input 事件发送到焦点窗口所属的应用。    
当有input未及时处理而引发ANR时，InputDispatcher 会报 notifyWindowUnresponsive，上图中的绿色方块。    

<img src="/images/android-knowledge-event-transfer-process-debug-skill/system_server_iq_oq_wq.png" width="721" height="351"/>

system_server进程的    
iq (InboundQueue)队列表示InputReader读取到了触控事件。    
oq (OutboundQueue)对应着一个可见窗口的事件分发处理队列。存放的这些事件是即将要被派发给目标窗口 App，但是此时还未发送成功的事件。    
wq (WaitQueue)表示触控事件在某个目标窗口中等待处理的耗时状态。这个队列里面记录的是已经派发给 App，但是 App 还在处理没有返回处理成功的事件。如果应用处理完成并反馈后就会从队列中移除。    

目标窗口应用App进程：     

<img src="/images/android-knowledge-event-transfer-process-debug-skill/app_deliver_input_event.png" width="721" height="153"/>

deliverInputEvent 标识 App UI Thread 被 Input 事件唤醒。    
aq (PendingInputEventQueue) 队列中记录的是应用需要处理的Input事件，这里可以看到input事件已经传递到了应用进程。

## 参考文章

https://juejin.cn/post/6956500920108580878
