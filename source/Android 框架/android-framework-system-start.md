---
title: Android 系统启动流程
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 Android 的系统启动流程
date: 2022-11-23 10:00:00
---

## init 进程

Android 设备的启动必须经历三个阶段：Boot Loader、Linux Kernel 和 Android 系统服务。严格来说，Android 系统实际是运行在 Linux 内核之上的一系列“服务进程”，而这些服务进程的祖先就是 init 进程。     
当按下启动电源时，系统启动后会加载引导程序，引导程序又启动 Linux 内核，在 Linux 内核加载完成后，第一件事情就是启动 init 进程。      
Boot Loader 是在操作系统内核运行之前的一段小程序。通过这段小程序，可以初始化硬件设备、建立内存空间的映射图，从而将系统的软硬件环境带到一个合适的状态，以便为最终调用操作系统内核准备好正确的环境。      
Linux内核加载完毕之后，就会在用户空间启动 init 进程。     
init 进程是 Android 系统中用户空间的第一个进程，PID（进程号）为 1，是 Android 系统启动流程中一个关键的步骤，作为第一个进程，被赋予了很多极其重要的工作职责，比如创建 Zygote 进程、启动bootanimation、启动ServiceManager等。      
它通过解析 /system/core/rootdir/init.rc 文件来构建出系统的初始运行状态，即 Android 系统服务大多是在 init.rc 脚本文件中有描述并按照一定的条件启动。      
init 进程启动做了很多工作，总的来说主要做了以下三件事：    
 - 创建（mkdir）和挂载（mount）启动所需的文件目录； 
 - 初始化和启动属性服务（property service）；
 - 解析 init.rc 配置文件并启动 Zygote 进程；

init 进程的源码在 /system/core/init 目录下面。       
init进程的入口是 /system/core/init/main.cpp 的main函数。      
Init进程启动后，首先挂载文件系统、再挂载相应的分区，启动SELinux安全策略，启动属性服务，解析rc文件，并启动相应属性服务进程，初始化epoll，依次设置signal、property、keychord这3个fd可读时相对应的回调函数。进入无线循环，用来响应各个进程的变化与重建。      

## Zygote 启动

在Android系统中，普通应用程序进程以及运行系统的服务 system_server 进程都是由 Zygote 进程来fork的。也叫做孵化器。它通过linux中的fork形式创建应用程序进程和 system_server 。由于zygote进程在启动的时候会创建java虚拟机环境，因此通过fork而创建的应用程序进程或者system_server进程可以在内部获得java虚拟机环境，不需要单独为每一个进程创建java虚拟机环境。      

Zygote 进程是通过init 进程在 system/core/rootdir/init.zygote32.rc 启动的。     

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart --only-if-running media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
    setenv PROCESS_FROM_ZYGOTE true

```

通过执行 app_process 来启动 zygote 进程，app_process 的代码在 `frameworks/base/cmds/app_process/`。     

```
//app_main.cpp
main()
  AppRuntime::start("com.android.internal.os.ZygoteInit")
    AndroidRuntime::start()
      // 启动java需要的jvm环境，才可以运行java代码
      AndroidRuntime::startVm
      // 获取 ZygoteInit 的 main 方法
      startMeth = env->GetStaticMethodID(startClass, "main",
      // 执行 ZygoteInit 的 main 方法
      env->CallStaticVoidMethod(startClass, startMeth, strArray);
```

现在回到 Java 中执行 frameworks/base/core/java/com/android/internal/os/ZygoteInit.java 的main 方法。     

```
ZygoteInit.main()
  ZygoteInit.preload
    ZygoteInit.preloadClasses()
  Runnable r = ZygoteInit.forkSystemServer()
    Zygote.forkSystemServer()
      nativeForkSystemServer()
        // -----> jni
        com_android_internal_os_Zygote_nativeForkSystemServer()
          zygote::ForkCommon
            pid = fork()
    if (pid == 0)
      // 这里运行的是 fork 处理的 systen_server 进程
      // 由于SystemServer是复制Zygote的进程，因此也会包含Zygote的zygoteServer，
      // 对于SystemServer没有其他作用，需要先将其关闭
      ZygoteServer.closeServerSocket()
      ZygoteInit.handleSystemServerProcess
        String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        //创建PathClassLoader
        ZygoteInit.getOrCreateSystemServerClassLoader()
        ZygoteInit.zygoteInit()
          //启动一下binder线程池
          ZygoteInit.nativeZygoteInit()
            RuntimeInit.applicationInit()
              //寻找 "com.android.server.SystemServer" 中的main方法，
              //然后构造一个Runable，对应的run方法就是调用main方法
              RuntimeInit.findStaticMain()
  // systen_server 进程运行 Runnable，也就是执行 SystemServer.main
  r.run()
  // 原来的 Zygote 进入runSelectLoop来循环等待 AMS 创建新进程的请求
  ZygoteServer.runSelectLoop
    Os.poll
    //创建客户端连接
    ZygoteServer.acceptCommandPeer
    ZygoteConnection.processCommand
      // fork 进程
      Zygote.forkAndSpecialize
        nativeForkAndSpecialize
      if (pid == 0)
        // 如果是创建的子进程
        ZygoteInit.zygoteInit
      else
        // 如果是 Zygote 进程，进行循环等待
        return null
  if (caller != null) {
    // 如果是 Zygote fork的进程，就执行 Runnable
    // Zygote 进程会在前面循环等待，不会走到这里
    caller.run()
```

## SystemServer 启动

Zygote fork 出 system_server 进程后，开始执行 SystemServer.main。     

```
ZygoteInit.main()
  RuntimeInit$MethodAndArgsCaller.run
    Method.invoke
      SystemServer.main
        SystemServer.run
          // 设置system_server Binder 最大线程数
          BinderInternal.setMaxThreads(sMaxBinderThreads);
          //启动引导服务
          SystemServe.startBootstrapServices()
            SystemServiceManager.startService(ActivityTaskManagerService.Lifecycle.class).getService()
              // 启动 AMS
              ActivityManagerService.Lifecycle.startService()
          //启动核心服务
          SystemServer.startCoreServices
          //启动其他服务
          SystemServer.startOtherServices
            // 启动 WMS
            WindowManagerService.main()
              new WindowManagerService()
            // 将 WMS 添加到 ServiceManager
            ServiceManager.addService(wms)
            ActivityManagerService.systemReady
              // 启动 FallbackHome
              ActivityTaskManagerService$LocalService.startHomeOnAllDisplays
                RootWindowContainer.startHomeOnAllDisplays
                  RootWindowContainer.startHomeOnDisplay
                    ......
                      RootWindowContainer.startHomeOnTaskDisplayArea
                        ActivityStartController.startHomeActivity
          // 启动SystemUI
          SystemServer.startSystemUi
            PackageManagerInternal.getSystemUiServiceComponent
            Context.startServiceAsUser
```

收到Intent.ACTION_USER_UNLOCKED 广播后 finish FallbackHome。      

```
// FallbackHome.java
    private BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            maybeFinish();
        }
    };
    
    private void maybeFinish() {
        if (getSystemService(UserManager.class).isUserUnlocked()) {
            final Intent homeIntent = new Intent(Intent.ACTION_MAIN)
                    .addCategory(Intent.CATEGORY_HOME);
            final ResolveInfo homeInfo = getPackageManager().resolveActivity(homeIntent, 0);
            if (Objects.equals(getPackageName(), homeInfo.activityInfo.packageName)) {
                Log.d(TAG, "User unlocked but no home; let's hope someone enables one soon?");
                mHandler.sendEmptyMessageDelayed(0, 500);
            } else {
                Log.d(TAG, "User unlocked and real home found; let's go!");
                getSystemService(PowerManager.class).userActivity(
                        SystemClock.uptimeMillis(), false);
                finish();
            }
        } else {
            Log.d(TAG, "User not yet unlocked");
        }
    }
```

启动普通桌面。     

```
Binder.execTransact
  ActivityClientController.onTransact
    ActivityClientController.finishActivity
      ActivityRecord.finishIfPossible
        ActivityRecord.completeFinishing
          ActivityRecord.addToFinishingAndWaitForIdle
            RootWindowContainer.resumeFocusedTasksTopActivities(
              Task.resumeTopActivityUncheckedLocked
                Task.resumeTopActivityInnerLocked
                  Task.resumeNextFocusableActivityWhenRootTaskIsEmpty
                    RootWindowContainer.resumeHomeActivity
                      // 启动普通桌面
                      RootWindowContainer.startHomeOnTaskDisplayArea
                        ActivityStartController.startHomeActivity
```


## 关于 FallbackHome

FallbackHome 的优先级为 -1000，比普通的桌面要低，因此，开机解锁后，就会优先启动普通桌面。     
只有在开机过程中，因为它的 application 配置了 `android:directBootAware="true"`，所以才会启动。     

```
        <activity android:name=".FallbackHome"
                  android:excludeFromRecents="true"
                  android:label=""
                  android:taskAffinity="com.android.settings.FallbackHome"
                  android:exported="true"
                  android:theme="@style/FallbackHome"
                  android:screenOrientation="portrait"
                  android:permission="android.permission.DEVICE_POWER"
                  android:configChanges="keyboardHidden">
            <intent-filter android:priority="-1000">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.HOME" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
```


## 相关文章

[Android启动流程——1序言、bootloader引导与Linux启动](https://www.jianshu.com/p/9f978d57c683)      
[安卓init进程详解](https://blog.csdn.net/qq_45649553/article/details/139344278)      
[千里马Android系统启动相关文章](https://blog.csdn.net/learnframework/article/details/116177288)      



