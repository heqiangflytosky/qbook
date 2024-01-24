---
title: Android实用技巧之adb命令：wm 命令的使用
categories: Android实用技巧
comments: true
tags: [Android, wm]
description: 介绍 wm 的使用
date: 2015-3-28 10:00:00
---

## 概述

wm 命令是和 Android WindowManagerService 相关联的，可以获取设备屏幕相关信息，比如：分辨率、像素密度等。甚至还可以修改这些参数，可以很方便地查看 APP 在不同像分辨率和素密度设备上的显示效果。

## 命令参数

使用 `adb shell wm` 可以查看wm支持的一些参数：

```
Window manager (window) commands:
  help
      Print this help text.
  size [reset|WxH|WdpxHdp] [-d DISPLAY_ID]
    Return or override display size.
    width and height in pixels unless suffixed with 'dp'.
  density [reset|DENSITY] [-d DISPLAY_ID]
    Return or override display density.
  folded-area [reset|LEFT,TOP,RIGHT,BOTTOM]
    Return or override folded area.
  scaling [off|auto] [-d DISPLAY_ID]
    Set display scaling mode.
  dismiss-keyguard
    Dismiss the keyguard, prompting user for auth if necessary.
  disable-blur [true|1|false|0]
  user-rotation [-d DISPLAY_ID] [free|lock] [rotation]
    Print or set user rotation mode and user rotation.
  dump-visible-window-views
    Dumps the encoded view hierarchies of visible windows
  fixed-to-user-rotation [-d DISPLAY_ID] [enabled|disabled|default]
    Print or set rotating display for app requested orientation.
  set-ignore-orientation-request [-d DISPLAY_ID] [true|1|false|0]
  get-ignore-orientation-request [-d DISPLAY_ID] 
    If app requested orientation should be ignored.
  set-multi-window-config
    Sets options to determine if activity should be shown in multi window:
      --supportsNonResizable [configValue]
        Whether the device supports non-resizable activity in multi window.
        -1: The device doesn't support non-resizable in multi window.
         0: The device supports non-resizable in multi window only if
            this is a large screen device.
         1: The device always supports non-resizable in multi window.
      --respectsActivityMinWidthHeight [configValue]
        Whether the device checks the activity min width/height to determine 
        if it can be shown in multi window.
        -1: The device ignores the activity min width/height when determining
            if it can be shown in multi window.
         0: If this is a small screen, the device compares the activity min
            width/height with the min multi window modes dimensions
            the device supports to determine if the activity can be shown in
            multi window.
         1: The device always compare the activity min width/height with the
            min multi window dimensions the device supports to determine if
            the activity can be shown in multi window.
  get-multi-window-config
    Prints values of the multi window config options.
  reset-multi-window-config
    Resets overrides to default values of the multi window config options.
  reset [-d DISPLAY_ID]
    Reset all override settings.
  tracing (start | stop)
    Start or stop window tracing.
  logging (start | stop | enable | disable | enable-text | disable-text)
    Logging settings.

```

 - `wm size`：查看屏幕的分辨率, 单位: px。
 - `wm size <WxH|WdpxHdp>`：修改置屏幕的分辨率：wm size 720x1280（单位px），wm size 360dpx640dp（单位dp）。
 - `wm size reset`：重置屏幕的分辨率，撤销对屏幕分辨率的修改（改回真实的物理分辨率）。
 - `wm density`：查看屏幕的像素密度, 单位: dpi（dots per inch）。
 - `wm density <DENSITY>`：修改屏幕的像素密度：wm density 360。
 - `wm density reset`：重置屏幕的像素密度，撤销对屏幕像素密度的修改（改回真实的像素密度）
 - `wm overscan`：该命令用来设置、重置LCD的显示区域。四个参数分别是显示边缘距离LCD左、上、右、下的像素数。例如，对于分辨率为540x960的屏幕，通过执行 命令wm overscan 0,0,0,420可将显示区域限定在一个540x540的矩形框里。
 - `wm scaling`：设置缩放模式
 - `wm screen-capture`：屏幕截图
 - `wm dismiss-keyguard`：取消锁屏
 - `wm logging`：打开或者关闭一些wm相关的日志：比如：`adb shell wm logging enable-text WM_DEBUG_ORIENTATION` 就能打开WMS中类似 `ProtoLog.v(WM_DEBUG_ORIENTATION......`的这种日志。
