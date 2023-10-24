---
title: Android 性能优化工具篇 -- SimplePerf
categories: Android性能优化
comments: true
tags: [ANR]
description: SimplePerf 的使用技巧
date: 2018-01-16 10:00:00
---

Simpleperf 也叫火焰图，是一个通用的命令行 CPU 性能剖析工具。我们通常使用 AS 中的 CPU Profile 来分析 CPU，但是这个比较受限于固件，如果是量产固件可能无法连接调试。    
Simpleperf 的优点：
 - 不受限于固件和APP类型
 - 这个工具由于硬件直接支持，对性能的影响非常小
 - 可以看到native的堆栈信息

Simpleperf 提供了命令行抓取 CPU 运行数据的方法，对于我们分析一些卡顿问题很有帮助。    
如需了解更多 Simpleperf 的用法，可以使用 `adb shell simpleperf -h` 查看。    

[Android 官方文档](https://developer.android.com/ndk/guides/simpleperf?hl=zh-cn)

## 手机端执行脚本

### 采样

手机端执行采样前需要先执行 `adb root`    

采样：    
adb shell simpleperf record -p 30639 -g --no-cut-samples --duration 5 -o /sdcard/perf_sysui.data    

-p 后面添加需要采样的进程 ID。

数据转换：    
adb shell simpleperf report-sample --show-callchain --protobuf -i /sdcard/perf_sysui.data -o /sdcard/perf_sysui.trace    

### 分析

把转换后的 perf_sysui.trace 拖动到 Android Studio里面，就自动允许 Profile 来加载数据进行可视化呈现。

## 其他用法

### simpleperf report

查找执行时间最长的共享库    
您可以运行此命令来查看哪些 .so 文件占用了最大的执行时间百分比（基于 CPU 周期数）。启动性能分析会话时，首先运行此命令是个不错的选择。    

```
adb shell simpleperf report --sort dso -i /sdcard/perf_sysui.data
```

查找执行时间最长的函数    
当您确定占用最多执行时间的共享库后，就可以运行此命令来查看执行该 .so 文件的函数所用时间的百分比。    

```
adb shell simpleperf report --dsos /system/lib64/libhwui.so --sort symbol -i /sdcard/perf_sysui.data
```

查找线程中所用时间的百分比    
.so 文件中的执行时间可以跨多个线程分配。您可以运行此命令来查看每个线程所用时间的百分比。    

```
simpleperf report --sort tid,comm
```


查找对象模块中所用时间的百分比    
在找到占用大部分执行时间的线程之后，可以使用此命令来隔离在这些线程上占用最长执行时间的对象模块。    

```
simpleperf report --tids threadID --sort dso
```

## 相关文章

https://zhuanlan.zhihu.com/p/25277481
https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md#Android-application-profiling
https://developer.android.com/ndk/guides/simpleperf?hl=zh-cn
