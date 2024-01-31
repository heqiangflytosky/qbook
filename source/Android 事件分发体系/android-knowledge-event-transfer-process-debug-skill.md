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


InputManager-JNI：
InputEventReceiver:


## Dumpsys

`adb shell dumpsys input` 通过dump查看触控事件处理系统框架部分：EventHub、InputReader、InputDispatcher的工作状态：

InputDispatcher：

## 开发者选项输入相关

打开 开发者选项 -> 触摸事件 开关，就可以额外获取一些触摸事件日志。

InputTransport：

打开 开发者选项 -> 指针位置 开关，触摸屏幕能看到小白点实时显示，能够主观感受屏幕滑动跟手度等状况。屏幕上会显示当前的触控数据    
打开 开发者选项 -> 显示点按操作反馈 开关，点击屏幕能看到点按操作的视觉反馈。    

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

## adb getevent 命令查看屏幕报点情况

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
