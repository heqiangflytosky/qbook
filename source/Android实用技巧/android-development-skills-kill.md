---
title: Android实用技巧之adb命令：kill 命令的使用
categories: Android实用技巧
comments: true
tags: [Android,kill]
description: 介绍 adb kill 的使用
date: 2014-10-18 10:00:00
---

## kill

kill {pid} 可以杀掉指定PID的进程。    

为了方便，可以写成脚本来执行：    

```
#!/bin/bash
adb shell ps |grep com.android.systemui|awk '{print $2}'|adb shell xargs  kill
```

另外pidof命令可以得到指定进程名称的pid：

```
#!/bin/bash
adb shell pidof com.android.systemui|awk '{print $1}'|adb shell xargs  kill
```

## kill all

kill all {进程名称} 用于杀掉指定名称的进程。

```
adb shell killall com.android.systemui
```

## kill -3 {Pid}

使用kill -3 {Pid} 命令，生成对应进程的 dump 文件，就像我们在分析ANR时的那种信息，可以得到当前进程各个线程的调用栈信息、CPU使用等 java trace 信息。    
执行命令后去 /data/anr/ 目录下找到类似 trace_00     这种文件pull出来后就可以分析了。     

另外 adb shell debuggerd -j {Pid} 也可以打印对应进程的 java trace 信息。
