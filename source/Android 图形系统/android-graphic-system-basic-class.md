---
title: Android 图形系统基础知识和相关类
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统基础知识和相关类
date: 2022-11-23 10:00:00
---


## 基础知识

先简单介绍下Android图形栈的流程：

1.  Android APP 的 UI 线程负责 input 事件，animation，UI 的 measure， layout，draw。
2.  draw 主要记录各个组件的绘制命令，序列化存储到 DisplayList。
3.  再 post 给 RenderThread 线程，通过使用 GPU 绘制为 Bitmap 位图数据。Surface简单理解为管理着这个 Bitmap。
4.  RenderThread 通过 Surface 把 Bitmap 传输给 SurfaceFlinger 进程。
5.  SurfaceFlinger 进程给作为客户端的 APP进程传递过来的 Surface 数据（就是 Bitmap 位图）创建一个layer。
6.  SurfaceFlinger 按照 Z-Order 排序对各个 APP 传递过来的 layer 做可见性裁剪等优化后，再通过 GPU 混合合成一张 Bitmap。【SurfaceFlinger 进程名字就是各个 Surface 的 Flinger (混合）】
7.  把合成的这个Bitmap 写到frame buffer中。
8.  display把这个数据绘制到屏幕上。


## 相关类


## 相关文章

[Android图形架构缺陷](https://mp.weixin.qq.com/s/M61-uXnZz3mllMrn5HRNLg)      
[Android GPU渲染屏幕绘制显示基础概念（1）](https://zhangphil.blog.csdn.net/article/details/138585120)      
[Android GPU渲染SurfaceFlinger合成RenderThread的dequeueBuffer/queueBuffer与fence机制（2](https://blog.csdn.net/zhangphil/article/details/138628225)      



