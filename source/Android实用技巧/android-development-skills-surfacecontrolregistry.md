---
title: Android 使用 SurfaceControlRegistry 查看 SurfaceControl 调用记录
categories: Android实用技巧
comments: true
tags: [Android,SurfaceControlRegistry]
description: 介绍 SurfaceControlRegistry 的使用
date: 2014-10-18 10:00:00
---


## 概述

Android 中提供了 SurfaceControlRegistry 来查看 SurfaceControl 的操作记录，它在 SurfaceControl 的方法里面都加了调用栈，来帮助我们来调试以及跟踪和识别SurfaceControl的方法调用情况，也可以跟踪 SurfaceControl 可能发生的泄露。        

## 方法

比如我们想看 SurfaceControl.Transaction 的show 和hide 方法，就输入下面的命令：    
```
adb shell setprop persist.wm.debug.sc.tx.log_match_call show,hide
```
这样就打印所有的 show 和hide 方法的调用栈。    
如果只想看某个或几个 Surface 的打印，就可以继续添加下面的 命令：    
```
adb shell setprop persist.wm.debug.sc.tx.log_match_name com.hq.android.androiddemo
```
这里，如果要设置多个名称，那么就要设置 Surface 的全称，如果设置了一个，那么就可以设置部分字段就行了。    

然后重启手机。    

输出的结果就是取上面两个命令的交集，日志 TAG=SurfaceControlRegistry。    

## 原理

```
public final class SurfaceControl implements Parcelable {
    public static class Transaction implements Closeable, Parcelable {
        public Transaction show(SurfaceControl sc) {
            ...
            if (SurfaceControlRegistry.sCallStackDebuggingEnabled) {
                SurfaceControlRegistry.getProcessInstance().checkCallStackDebugging(
                        "show", this, sc, null);
            }
            nativeSetFlags(mNativeObject, sc.mNativeObject, 0, SURFACE_HIDDEN);
            return this;
        }
```

```
public class SurfaceControlRegistry {
    final static void initializeCallStackDebugging() {
        ......

        sCallStackDebuggingInitialized = true;
        // SurfaceControl 的方法过滤
        sCallStackDebuggingMatchCall =
                SystemProperties.get("persist.wm.debug.sc.tx.log_match_call", null)
                        .toLowerCase();
        // Surface 的名称过滤
        sCallStackDebuggingMatchName =
                SystemProperties.get("persist.wm.debug.sc.tx.log_match_name", null)
                        .toLowerCase();
        // 上面任意一个过滤设置了就可以开启调用栈调试
        sCallStackDebuggingEnabled = (!sCallStackDebuggingMatchCall.isEmpty()
                || !sCallStackDebuggingMatchName.isEmpty());
        if (sCallStackDebuggingEnabled) {
            Log.d(TAG, "Enabling transaction call stack debugging:"
                    + " matchCall=" + sCallStackDebuggingMatchCall
                    + " matchName=" + sCallStackDebuggingMatchName);
        }
    }
```

```
    final void checkCallStackDebugging(@NonNull String call,
            @Nullable SurfaceControl.Transaction tx, @Nullable SurfaceControl sc,
            @Nullable String details) {
        if (!sCallStackDebuggingEnabled) {
            return;
        }
        // 进行 sCallStackDebuggingMatchCall 和 sCallStackDebuggingMatchName 匹配
        if (!matchesForCallStackDebugging(sc != null ? sc.getName() : null, call)) {
            return;
        }
        final String txMsg = tx != null ? "tx=" + tx.getId() + " ": "";
        final String scMsg = sc != null ? " sc=" + sc.getName() + "": "";
        final String msg = details != null
                ? call + " (" + txMsg + scMsg + ") " + details
                : call + " (" + txMsg + scMsg + ")";
        // 日志打印
        Log.e(TAG, msg, new Throwable());
    }
```

匹配规则：    

```
    public final boolean matchesForCallStackDebugging(@Nullable String name, @NonNull String call) {
        final boolean matchCall = !sCallStackDebuggingMatchCall.isEmpty();
        // sCallStackDebuggingMatchCall 包含该方法名称
        if (matchCall && !sCallStackDebuggingMatchCall.contains(call.toLowerCase())) {
            return false;
        }
        final boolean matchName = !sCallStackDebuggingMatchName.isEmpty();
        if (!matchName) {
            return true;
        }
        if (name == null) {
            return false;
        }
        // sCallStackDebuggingMatchName 配置中包含了该 Surface 的名称或者该 Surface 的名称包含sCallStackDebuggingMatchName 配置
        // 所以这里，如果要设置多个名称，那么就要设置 Surface 的全称，如果设置了一个，那么就可以设置部分字段就行了
        return sCallStackDebuggingMatchName.contains(name.toLowerCase()) ||
                        name.toLowerCase().contains(sCallStackDebuggingMatchName);
    }
```
