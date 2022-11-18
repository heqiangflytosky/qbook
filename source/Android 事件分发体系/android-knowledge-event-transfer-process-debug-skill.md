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


InputDispatcher：
