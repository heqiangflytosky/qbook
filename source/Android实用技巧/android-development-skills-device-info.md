---
title: Android 查看硬件信息命令
categories: Android实用技巧
comments: true
tags: [Android]
description: 
date: 2015-3-26 10:00:00
---


## 查看CPU信息

### 查看 CPU 核心个数

```
adb shell cat  /proc/cpuinfo
```

输出：

```
processor	: 0
BogoMIPS	: 38.40
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 sm3 sm4 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint i8mm bf16 dgh bti ecv afp
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd80
CPU revision	: 1

......

processor	: 7
BogoMIPS	: 38.40
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 sm3 sm4 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint i8mm bf16 dgh bti ecv afp
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd82
CPU revision	: 1
```

这里可以到看到当前设备有 8 个核，索引是0-7。     


### 查看CPU 频率

然后可以来查看CPU频率相关信息。    

```
cd sys/devices/system/cpu   
```

可以看到一些 CPU 相关的信息：

```
cpu0 cpu1 cpu2 cpu3 cpu4 cpu5 cpu6 cpu7
```

#### 第一种方法

可以通过下面命令来查看每个CPU的最大频率：

```
cat cpu0/cpufreq/cpuinfo_max_freq
2265600
```

#### 第二种方法

```
cd /sys/devices/system/cpu/cpufreq/
```

可以看到 CPU 的分组情况：     

```
boost  policy0  policy2  policy5  policy7
```

当前 CPU 分组情况为：0-1，2-4，5-6，7        
下面命令可以查看每个分组的最大频率和最小频率。     

```
cat policy0/cpuinfo_min_freq
cat policy0/cpuinfo_max_freq
```

从而可以知道大小核情况。    






