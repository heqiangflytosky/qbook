---
title: Android 性能优化文章汇总
categories: Android性能优化
comments: true
tags: [Android性能优化]
description: 网络文章汇总
date: 2016-2-20 10:00:00
---
## 汇总文章

[Android 性能优化必知必会](https://www.androidperformance.com/2018/05/07/Android-performance-optimization-skills-and-tools/#)    

## 性能优化工具

[另一个Android性能剖析工具——simpleperf](https://zhuanlan.zhihu.com/p/25277481)    
[Simpleperf](https://developer.android.com/ndk/guides/simpleperf?hl=zh-cn)    
[Android application profiling](https://android.googlesource.com/platform/system/extras/+/refs/heads/main/simpleperf/doc/android_application_profiling.md)    
[Android Tech And Perf 的 Android Systrace 系列文章](https://www.androidperformance.com/2019/05/28/Android-Systrace-About/#)

## ANR

[字节跳动技术团队 ANR 系列](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI1MzYzMjE0MQ==&action=getalbum&album_id=1780091311874686979&scene=173&from_msgid=2247488116&from_itemidx=1&count=3&nolastread=1#wechat_redirect)    
 - [今日头条 ANR 优化实践系列 - 设计原理及影响因素 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488116&idx=1&sn=fdf80fa52c57a3360ad1999da2a9656b&chksm=e9d0d996dea750807aadc62d7ed442948ad197607afb9409dd5a296b16fb3d5243f9224b5763&scene=178&cur_album_id=1780091311874686979#rd)
 - [今日头条 ANR 优化实践系列 - 监控工具与分析思路 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488182&idx=1&sn=6337f1b51d487057b162064c3e24c439&chksm=e9d0d954dea75042193ed09f30eb8ba0acd93870227c5d33b33361b739a03562afb685df9215&scene=178&cur_album_id=1780091311874686979#rd)
 - [ 今日头条 ANR 优化实践系列分享 - 实例剖析集锦 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488243&idx=1&sn=1f948e0ef616c6dfe54513a2a94357be&chksm=e9d0d911dea75007f36b3701b51842b9fa40969fe8175c2cb4aecf96793504602c574945d636&scene=178&cur_album_id=1780091311874686979#rd)
 - [ 今日头条 ANR 优化实践系列 - Barrier 导致主线程假死 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488314&idx=1&sn=559e52288ae2730a580fcd550f22d895&chksm=e9d0d8d8dea751ceecb715d472796f0c678a9358abf91eb279cdb0576329595e87531e221438&scene=178&cur_album_id=1780091311874686979#rd)
 - [ 今日头条 ANR 优化实践系列 - 告别 SharedPreference 等待 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247488558&idx=1&sn=27dda3c3630116d37ab56a8c7bdf1382&chksm=e9d0dfccdea756daed46b340fb8021b57ea8cc300e58bdb59f0305f8290704984308a089bf2d&scene=178&cur_album_id=1780091311874686979#rd)
 - [ 西瓜视频稳定性治理体系建设三：Sliver 原理及实践 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247489902&idx=1&sn=bfdf9f48dc6dc973722b5dcab9cd5882&chksm=e9d0d28cdea75b9ad255eb5de227240d2e6f0e9d66e562d3f49cf69f8ed4127c9954ef21bb6d&scene=178&cur_album_id=1780091311874686979#rd)
 - [ 西瓜卡顿 & ANR 优化治理及监控体系建设 ](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247489949&idx=1&sn=01948c047c0ce203956a3cf81dd20e83&chksm=e9d0d27fdea75b697e70a665b4c6912a8081649700766cf007a7b75d420a57089fe06d2e85b0&scene=178&cur_album_id=1780091311874686979&poc_token=HCkNZ2WjlVAuvciKFnHtUseZAdIEwaKjmM2Hj8bZ)
 - [抖音 ANR 自动归因平台建设实践 ](https://mp.weixin.qq.com/s/ZMkj-VvG5sFfTCfIcFa4mg)

ANR 系列：    
 - [ANR系列之一：ANR显示和日志生成原理讲解](https://blog.csdn.net/rzleilei/article/details/120720918)
 - [ANR系列之二：Input类型ANR产生原理讲解](https://blog.csdn.net/rzleilei/article/details/127118071)
 - [ANR系列之三：broadcast类型ANR产生原理讲解](https://blog.csdn.net/rzleilei/article/details/127401168)
 - [ANR系列之四：ContentProvider类型ANR产生原理讲解](https://blog.csdn.net/rzleilei/article/details/128039319)
 - [ANR系列之五：Service类型ANR原理讲解](https://blog.csdn.net/rzleilei/article/details/128491937)

## 卡顿优化

[努比亚技术团队：简书](https://www.jianshu.com/u/167b54662111)    
 - [Android卡顿掉帧问题分析之原理篇](https://www.jianshu.com/p/386bbb5fa29a)
 - [Android卡顿掉帧问题分析之工具篇](https://www.jianshu.com/p/cf531a3af828)
 - [Android卡顿掉帧问题分析之实战篇](https://www.jianshu.com/p/f1a777551b70)

