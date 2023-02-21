---
title: Android 卡顿优化工具篇 -- Perfetto 的使用
categories: Android性能优化
comments: true
tags: [卡顿优化]
description: 介绍卡顿优化工具 Perfetto 的使用
date: 2018-10-10 10:00:00
---


## Perfetto


## 录制trace方法

### 手机界面抓取

打开 设置-》辅助功能-》开发者选项-》系统跟踪。    
或者通过命令打开界面：    
```
adb shell am start com.android.traceur/com.android.traceur.MainActivity
```
类别项可以选择需要录制的项。    
打开长期跟踪项。    
点击上方的录制跟踪记录后开始录制，再次点击开关后停止录制，可以把trace文件保存到你想要保存的地方。    
可以从 /data/local/traces  导出录制好的trace文件。    

### 命令行抓取

````
adb shell perfetto -o /data/misc/perfetto-traces/trace -t 10s sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory  
````
如果遇到目录权限问题，换个目录试试：    
````
adb shell perfetto -o /data/local/traces/trace_file.perfetto-trace -t 10s sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory  
````

也可以从 https://ui.perfetto.dev/ 复制已经配置好的配置文件，然后到终端执行。    
下面是一个默认的配置：    

```
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace \
<<EOF

buffers: {
    size_kb: 63488
    fill_policy: DISCARD
}
buffers: {
    size_kb: 2048
    fill_policy: DISCARD
}
data_sources: {
    config {
        name: "android.gpu.memory"
    }
}
data_sources: {
    config {
        name: "android.power"
        android_power_config {
            battery_poll_ms: 1000
            battery_counters: BATTERY_COUNTER_CAPACITY_PERCENT
            battery_counters: BATTERY_COUNTER_CHARGE
            battery_counters: BATTERY_COUNTER_CURRENT
            collect_power_rails: true
        }
    }
}
data_sources: {
    config {
        name: "linux.process_stats"
        target_buffer: 1
        process_stats_config {
            scan_all_processes_on_start: true
            proc_stats_poll_ms: 1000
        }
    }
}
data_sources: {
    config {
        name: "android.log"
        android_log_config {
        }
    }
}
data_sources: {
    config {
        name: "android.surfaceflinger.frametimeline"
    }
}
data_sources: {
    config {
        name: "android.game_interventions"
    }
}
data_sources: {
    config {
        name: "linux.sys_stats"
        sys_stats_config {
            meminfo_period_ms: 1000
            vmstat_period_ms: 1000
            stat_period_ms: 1000
            stat_counters: STAT_CPU_TIMES
            stat_counters: STAT_FORK_COUNT
            cpufreq_period_ms: 1000
        }
    }
}
data_sources: {
    config {
        name: "android.heapprofd"
        target_buffer: 0
        heapprofd_config {
            sampling_interval_bytes: 4096
            shmem_size_bytes: 8388608
            block_client: true
        }
    }
}
data_sources: {
    config {
        name: "android.java_hprof"
        target_buffer: 0
        java_hprof_config {
        }
    }
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "power/suspend_resume"
            ftrace_events: "sched/sched_wakeup"
            ftrace_events: "sched/sched_wakeup_new"
            ftrace_events: "sched/sched_waking"
            ftrace_events: "power/cpu_frequency"
            ftrace_events: "power/cpu_idle"
            ftrace_events: "power/gpu_frequency"
            ftrace_events: "gpu_mem/gpu_mem_total"
            ftrace_events: "raw_syscalls/sys_enter"
            ftrace_events: "raw_syscalls/sys_exit"
            ftrace_events: "regulator/regulator_set_voltage"
            ftrace_events: "regulator/regulator_set_voltage_complete"
            ftrace_events: "power/clock_enable"
            ftrace_events: "power/clock_disable"
            ftrace_events: "power/clock_set_rate"
            ftrace_events: "mm_event/mm_event_record"
            ftrace_events: "kmem/rss_stat"
            ftrace_events: "ion/ion_stat"
            ftrace_events: "dmabuf_heap/dma_heap_stat"
            ftrace_events: "kmem/ion_heap_grow"
            ftrace_events: "kmem/ion_heap_shrink"
            ftrace_events: "sched/sched_process_exit"
            ftrace_events: "sched/sched_process_free"
            ftrace_events: "task/task_newtask"
            ftrace_events: "task/task_rename"
            ftrace_events: "lowmemorykiller/lowmemory_kill"
            ftrace_events: "oom/oom_score_adj_update"
            ftrace_events: "ftrace/print"
            atrace_apps: "*"
        }
    }
}
duration_ms: 10000

EOF

```

### Perfetto UI 网站抓取

打开 https://ui.perfetto.dev/ 首先配对设备，如有需要就修改配置问题。下面介绍几个配置选项。    

<img src="/images/android-performance-optimization-tools-perfetto/4.png" width="450" height="362"/>


然后点击 "Start Recording",点击 Stop 后在配置文件设置的输出目录中找到 trace 文件，从手机 pull 出来后在 https://ui.perfetto.dev/ 中打开。    
如果遇到 “It looks like you didn't add any probes. Please add at least one to get a non-empty trace.”    可以执行命令：`adb shell setprop persist.traced.enable 1`

## 分析

### VSYNC-app

相较于 systrace，perfetto 不会直接把 VSync 帧在图形界面标注出来，所以我们第一步最好是直接找到 SurfaceFlinger 进程的 VSYNC-app 数据，并钉在顶部，方便我们能分析出app每一帧的运行时间。    
perfetto 也不会直接高亮显示出应用的问题帧，而需要你通过应用主线程和 RenderThread 线程配合 SurfaceFlinger 的 VSYNC-app 数据配合分析出当前应用的问题帧。    

### Cpu0--Cpu7

Cpu X Frequency:
Cpu X Max Freq Limit:
Cpu X Min Freq Limit:

### Android logs

perfetto可以实时记录log，然后将log和trace信息一一对应。    
生成的perfetto文件，滑动下方的android log，可以看到有一根竖线，对应到trace的tag，日志和trace tag的一一对应。    

<img src="/images/android-performance-optimization-tools-perfetto/3.png" width="829" height="333"/>

### Actual timeline

可以看到SF某一帧是合成的APP的哪一帧，已经合成的状态。    

### Android missed frames

### Slice Details：

## 实践

### CPU 大小核策略问题

ID： 1178862

这里介绍一个在打开相机时下拉通知栏卡顿的问题。    
首先看一下 trace，发现 systemui 渲染耗时正常，DrawFrames大部分时间都在 dequeueBuffer，应该是和相机竞争导致sf合成失速。    

<img src="/images/android-performance-optimization-tools-perfetto/1.png" width="796" height="356"/>

但是此时sf跑在0，1，2小核。这个问题就可以调整一下 sf 的CPU策略。    

<img src="/images/android-performance-optimization-tools-perfetto/2.png" width="829" height="333"/>
