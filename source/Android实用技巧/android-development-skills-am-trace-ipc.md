---
title: Android 查看 Binder 调用信息
categories: Android实用技巧
comments: true
tags: [Android, getprop, setprop, watchprops]
description: 
date: 2014-10-18 10:00:00
---

## 抓取 Binder 通信调用堆栈

开启追踪：

```
adb shell am trace-ipc start
```

结束追踪：

```
adb shell am trace-ipc stop --dump-file /data/local/tmp/ipc.txt
```

导出追踪文件

```
adb pull /data/local/tmp/ipc.txt ~/test/
```

查看：    
这里还好根据进程来进行分类。    

```
Binder transaction traces for all processes.

// 按进程分类
Traces for process: system
Count: 1
Trace: java.lang.Throwable
	at android.os.BinderProxy.transact(BinderProxy.java:553)
	at com.android.internal.statusbar.IStatusBar$Stub$Proxy.setTopAppHidesStatusBar(IStatusBar.java:1812)
	at com.android.server.statusbar.StatusBarManagerService$1.setTopAppHidesStatusBar(StatusBarManagerService.java:599)
	at com.android.server.wm.DisplayPolicy.updateSystemBarsLw(DisplayPolicy.java:2668)
	at com.android.server.wm.DisplayPolicy.updateSystemBarAttributes(DisplayPolicy.java:2515)
	at com.android.server.wm.DisplayPolicy.finishPostLayoutPolicyLw(DisplayPolicy.java:1874)
	at com.android.server.wm.DisplayContent.applySurfaceChangesTransaction(DisplayContent.java:5119)
	at com.android.server.wm.RootWindowContainer.applySurfaceChangesTransaction(RootWindowContainer.java:1018)
	at com.android.server.wm.RootWindowContainer.performSurfacePlacementNoTrace(RootWindowContainer.java:819)
	at com.android.server.wm.RootWindowContainer.performSurfacePlacement(RootWindowContainer.java:777)
	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacementLoop(WindowSurfacePlacer.java:177)
	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:126)
	at com.android.server.wm.WindowSurfacePlacer.performSurfacePlacement(WindowSurfacePlacer.java:115)
	at com.android.server.wm.WindowState.setupWindowForRemoveOnExit(WindowState.java:2592)
	at com.android.server.wm.WindowState.removeIfPossible(WindowState.java:2552)
	at com.android.server.wm.WindowManagerService.removeWindow(WindowManagerService.java:2018)
	at com.android.server.wm.Session.remove(Session.java:232)
	at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:666)
	at com.android.server.wm.Session.onTransact(Session.java:179)
	at android.os.Binder.execTransactInternal(Binder.java:1344)
	at android.os.Binder.execTransact(Binder.java:1275)
......
```

系统在 BinderProxy 的 transact 方法中实现了相关逻辑。只会打印 Java 层的binder通信。        
相关代码：     

```
// BinderProxy.java

    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    ......
        final boolean tracingEnabled = Binder.isStackTrackingEnabled();
        if (tracingEnabled) {
            final Throwable tr = new Throwable();
            Binder.getTransactionTracker().addTrace(tr);
            StackTraceElement stackTraceElement = tr.getStackTrace()[1];
            Trace.traceBegin(Trace.TRACE_TAG_ALWAYS,
                    stackTraceElement.getClassName() + "." + stackTraceElement.getMethodName());
        }
    .......    
            if (tracingEnabled) {
                Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
            }
```

## perfetto 中查看

开启 trace-ipc 后，不只会有日志打印，在 perfetto 中也会有相关的输出。    
开启前，binder 调用没有相关信息：    

<img src="/images/android-development-skills-am-trace-ipc/1.png" width="886" height="196"/>

开启后，可以看出具体什么方法发起了binder调用。   

<img src="/images/android-development-skills-am-trace-ipc/2.png" width="774" height="224"/>

但是在 server 端仍然看不到相关的调用。    

<img src="/images/android-development-skills-am-trace-ipc/3.png" width="548" height="216"/>

这时我们可以在开启 perfetto 时添加 aidi 参数，比如：    

```
./record_android_trace -o $(date +%Y%m%d_%H%M%S)_trace_file.perfetto-trace -t 5s -b 32mb sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory gfx view wm am ss video camera hal res sync idle binder_driver binder_lock ss aidl
```

然后在开启 trace-ipc 抓取 trace，就可以看到 server 端的调用情况了：    

<img src="/images/android-development-skills-am-trace-ipc/4.png" width="1164" height="287"/>

具体代码在：    

```
// Binder.java

    private boolean execTransactInternal(int code, Parcel data, Parcel reply, int flags,
            int callingUid) {
            
            ...
        final boolean tagEnabled = Trace.isTagEnabled(Trace.TRACE_TAG_AIDL);
        ...
        final boolean tracingEnabled = tagEnabled && transactionTraceName != null;
        ...
            if (tracingEnabled) {
                Trace.traceBegin(Trace.TRACE_TAG_AIDL, transactionTraceName);
            }
```

## 相关文章

https://blog.csdn.net/sinat_20059415/article/details/106158891

https://mp.weixin.qq.com/s/MS-TE1Z32F_4QkXm57e2Pg
https://mp.weixin.qq.com/s/incGzW8yG4nufMIbUm-pLw
