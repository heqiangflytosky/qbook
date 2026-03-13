---
title: Android ServiceManager 进程介绍
categories: Android 框架
comments: true
tags: [Android 框架]
description: Android Binder
date: 2022-11-23 10:00:00
---




## 简介

ServiceManager 是 Binder 的服务总管，负责 Binder 服务的注册和查找。       
它是由 init 进程启动的。       
它是一个特殊的 Binder 服务（同时也是 Binder 的 Server）。Server 需要向 ServiceManager 注册服务。Client 通过服务名向 SM 查询服务句柄。      
ServiceManager 的 Binder 句柄是 0（全局固定），所有进程都知道如何找到它。       
前面我们介绍过 App 端常用的两种跨进程通信的方式：    
 - 通过创建 Service，通过 bindService 获取 IBinder。
 - 通过Android系统服务 ServiceManager.getService() 方式来获取 IBinder。

那么主要的过程就是必须要获取一个 IBinder 对象，然后转化为代理对象后调用方法，这个过程中涉及到 Binder 的 transact 进行跨进程通信。            
上面两种方法都是通过已经建立Binder联系的 Client 和 Server 进行传递 IBinder 对象实现跨进程通信。      
其实 Service.bindService 这里面都绕不开 ServiceManager.getService()，因为通过 Service.bindService 也是也涉及到 App 和 AMS 通过 Activity.startService() 进行跨进程通信的，需要通过 ServiceManager 来获取 AMS 服务。       
那么，App 通过 ServiceManager.getService() 获取系统服务最终也是需要通过 ServiceManager 进程来事件的。     
system_server 先通过 getIServiceManager().addService() 把服务注册进 ServiceManager 进程，然后应用进程就通过 getIServiceManager().getService() 来获取系统服务 IBinder 对象。     
所以说，ServiceManager 是 Binder 架构中必不可少的一环。     

## 启动

[Android Framework实战课程-binder专题之ServiceManager启动及运行篇](https://blog.csdn.net/learnframework/article/details/118465668)      



```
///frameworks/native/cmds/servicemanager/servicemanager.rc      
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    file /dev/kmsg w
    onrestart setprop servicemanager.ready false
    onrestart restart --only-if-running apexd
    onrestart restart audioserver
    onrestart restart gatekeeperd
    onrestart class_restart --only-enabled main
    onrestart class_restart --only-enabled hal
    onrestart class_restart --only-enabled early_hal
    task_profiles ProcessCapacityHigh
    shutdown critical
```

```
main()
  driver = "/dev/binder";
  // 打开驱动
  binder_open(driver, 128*1024)
    bs->fd = open(driver, O_RDWR | O_CLOEXEC)
    //映射内存
    bs->mapped = mmap()
  //把自己成为servicemanager
  binder_become_context_manager(bs)
    //ioctl调用到驱动，告诉驱动我就是manager
    ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0)
  //进入循环
  binder_loop(bs, svcmgr_handler)
    //告诉binder驱动，已经进入loop状态
    readbuf[0] = BC_ENTER_LOOPER;
    for (;;) {
    // 一直循环读取binder驱动中的发送过来的数据
    ioctl(bs->fd, BINDER_WRITE_READ, &bwr)
      // ----> 进入到内核态，下一小节分析
    // 解析数据
    binder_parse()
      //这里解析了驱动发送过来数据，调用func即svcmgr_handler
      func()
        svcmgr_handler
          //获取handle即binder实体对象的引用
          bio_get_ref(msg)
            do_add_service()
              //寻找是否已经有了
              find_svc()
                //没有添加过，就需要构造对象进行添加到链表
                svcinfo si = malloc()
                si->handle = handle;
                .....
```

## 和 ServiceManager 通信

ServiceManager 其实也是属于一个普通的应用程序，它也需要与Binder驱动进行通信，他在跨进程通信中CS模式中扮演的 Server端,普通进程需要添加获取Service就是 Client 端。     

```
  sp<IServiceManager> sm = defaultServiceManager();
  SampleService* samServ = new SampleService();
  // 添加
  status_t ret = sm->addService(String16(SAMPLE_SERIVCE_DES), samServ);
```

Server 端(和ServiceManager通信过程中也代表Client端)获取到 ServiceManager 的代理对象，然后调用 addService 向ServiceManager注册自己定义的 IBinder，这样，其他应用程序只要获取到这个 IBinder 就可以和 Server 端跨进程通信。       

```
defaultServiceManager()
  ProcessState::self()->getContextObject(NULL) // ProcessState::getContextObject
    ProcessState::getStrongProxyForHandle(0)
      //直接就可以创建对应的BpBinder对象
      b = new BpBinder(handle);
IServiceManager::addService(IBinder)
  BpBinder::transact
    IPCThreadState::transact
      IPCThreadState::writeTransactionData
      IPCThreadState::waitForResponse
        IPCThreadState::talkWithDriver
          ioctl 
            // ---> 进入内核态
```

可以看出其实它最后就是new BpBinder(0),这个就成了最后的 ServiceManager 的本地代理对象IBinder，ps：因为系统默认servicemanager的handle固定就是0，所以本地代理创建就非常非常简单。      

内核态：      

```
binder_ioctl
  case BINDER_WRITE_READ:
    binder_ioctl_write_read
      // 拷贝用户空间数据到内核
      copy_from_user(&bwr, ubuf, sizeof(bwr))
      binder_thread_write
        binder_transaction
          //这里发起唤醒服务进程等待队列
          binder_proc_transaction
            //对方的进程中寻找到一个线程进行传输
            binder_select_thread_ilocked
            //把对应任务放入线程执行队列
            binder_enqueue_thread_work_ilocked
            //如果同步调用，则唤醒目标线程
            binder_wakeup_thread_ilocked
      copy_to_user
```

用户态：      

前面唤醒的目标线程就是servicemanager的 loop 的那个主线程，他是一直ioctl方式在读取驱动数据，所以它的进程应该执行的是：     

```
binder_thread_read
  //取出任务
  binder_dequeue_work_head_ilocked
  //拷贝cmd到用户空间
  put_user
  //拷贝真实实体数据
  copy_to_user
```

后面就是 ServiceManager 读取到数据。这部分上面有介绍过。       



