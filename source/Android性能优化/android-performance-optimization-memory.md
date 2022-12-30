
android-performance-optimization-memory.md

---
title: Android 性能优化实战篇 -- 内存优化
categories: Android性能优化
comments: true
tags: [内存优化]
description: 介绍内存优化的一般方法和步骤
date: 2018-10-10 10:00:00
---


## 概述

前面几篇文章介绍了内存优化的意义以及一些理论方法和编码过程中的一些注意事项，本文就着重从内存优化的实践方面介绍如果对内存进行优化。

## 内存优化步骤

### 检查

通过一系列的工具来掌握应用的内存使用情况，一般我们先使用 `adb shell dumpsys meminfo -a <进程名称>` 来大概看一下内存的总体使用情况，看看那部分内存占用偏高，然后针对每个部分逐个突破。    
关于 `dumpsys meminfo` 的使用在前面有专门的介绍。    
如果发现 Java Heap 和 Native Heap 占用内存较高，可以通过 Memory Profiler 来分析内存中有那些占用大内存的对象或者是没有及时释放的对象。    
如果发现 Code 占用内存较大，可以通过 showmap 来看进程中有没有哪些库占用内存比较多。然后也可以着手对 APK 进行一些瘦身操作。    


### 代码优化

代码优化分两个方法，一个是平时我们编码是要养成良好的素养和习惯。二是发现问题后的问题优化。主要包括节省内存开销和避免内存泄漏两个目标。    
关于编码喜欢的问题可以参考前面的文章。    
编码过程中通过运行lint检查工具发现可能影响内存性能的地方。运行 lint 结果中关注 `Android Lint:Performance` 中的警告项可以帮助我们进一步优化我们的代码。Android Studio 中菜单项 Analyze -> Inspect Code 可以启动静态代码检查。    

### 监控

可以借助一些监控工具来实时监测内存使用异常，比如内存泄漏监测工具 LeakCanary。    

