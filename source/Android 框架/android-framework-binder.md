---
title: Android Binder
categories: Android 框架
comments: true
tags: [Android 框架]
description: Android Binder
date: 2022-11-23 10:00:00
---




## 概述

Binder 驱动是 Android 系统中跨进程通信（IPC）的核心，它运行在内核空间，作为用户空间进程间数据交换的枢纽。可以说，没有 Binder 驱动，Android 四大组件间的协作、系统服务的调用都将无法实现。    
Binder 驱动被实现为 Linux 内核的一个字符设备驱动（通常路径为 /dev/binder），但它并不操作物理硬件，而是专门为进程间通信提供支持。    
Linux 驱动设备分为三类，分别是字符设备、块设备和网络设备。字符设备中有一个比较特殊的 misc 杂项设备，设备号为 10，可以自动生成设备节点。Android 的 Ashmem、Binder 都属于 misc 杂项设备。       

## Binder 基础知识

### 缓冲区限制

Binder通信支持最大内存,1M - 8K。    

```
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
```

8K 可能是需要存储一些 Binder的元数据。     
为什么限制为1M？    
综合性能考虑，Binder的设计初衷本就不是为了传递大数据而设计。     


### 默认的最大线程数

默认的最大 binder 线程数。    

```
#define DEFAULT_MAX_BINDER_THREADS 15

static unique_fd open_driver(const char* driver, String8* error) {
    ....
    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
    result = ioctl(fd.get(), BINDER_SET_MAX_THREADS, &maxThreads);
}
```

所以，一个进程默认的最大 Binder 线程数是15+1，这个1是Binder的主线程。      

system_server 的最大 binder 线程数。    

```
    // maximum number of binder threads used for system_server
    // will be higher than the system default
    private static final int sMaxBinderThreads = 31;
```

```
    @Override
    public IBinder getSubService(String servicesName) {
        final long origId = Binder.clearCallingIdentity();
        try {
            synchronized (mSubServices) {
                return mSubServices.get(servicesName);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

### 关于Stub 和 Proxy

AIDL 生成的代码中，Stub 和 Proxy 分别代表什么？     
Stub： 服务端实体，继承自 Binder，实现了 onTransact() 方法，负责反序列化并调用服务端真正的实现方法。     
Proxy： 客户端代理，持有 BinderProxy（指向服务端的句柄），实现了 transact() 方法，负责将参数序列化后发送给驱动。     

```
  public static abstract class Stub extends android.os.Binder implements com.example.myapplication.IMyAidlInterface
  {
    ....
    public static com.example.myapplication.IMyAidlInterface asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.example.myapplication.IMyAidlInterface))) {
        return ((com.example.myapplication.IMyAidlInterface)iin);
      }
      return new com.example.myapplication.IMyAidlInterface.Stub.Proxy(obj);
    }
```

```
    private static class Proxy implements com.example.myapplication.IMyAidlInterface
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      ......
      @Override public java.lang.String getVersion() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.lang.String _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getVersion, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getVersion();
          }
          _reply.readException();
          _result = _reply.readString();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }

```

Proxy 类中的 mRemote 实际是一个 BpBinder（Native 层）或 BinderProxy（Java 层），它通过 transact() 方法与 Binder 驱动通信。    
Binder Proxy 的调用链：    
Java Proxy → BinderProxy → BpBinder → Binder 驱动 → BBinder（服务端）。     

### 通信模型

四大角色：Client、Server、ServiceManager（SM）、Binder 驱动。    
SM 像一个 DNS 服务器，负责将服务名称转换为 Binder 句柄；Binder 驱动是连接所有组件的“路由器”。    

### 和其他 IPC 对比

| IPC 方式 | 数据拷贝次数 | 是否面向对象 | 安全性（自动携带身份）| 适用场景|
|:-------------:|:-------------:|:-------------:|:-------------:|:-------------:|
| Binder | 1次| 是| 是| Android 核心 IPC|
| Socket| 2次| 否| 否|网络通信，跨设备；大量数据流传输，适合实时性要求高的连续数据| 
| 共享内存| 0次（需同步）| 否| 否（需额外机制）| 大量数据共享|
| 管道/消息队列	|2次 | 否| 否| 简单数据流|

### 其他

Android中的对象通过binder传到另外进程后还是同一个对象吗？     
不是同一个对象。Android的Binder跨进程通信（IPC）机制传递对象时，并不是直接传递原始对象的引用，而是通过以下方式之一：     

1.可序列化对象     

对象会被序列化（打包）：在发送进程中将对象转换为字节流。     
在接收进程反序列化（解包）：重新创建新对象，内容与原始对象相同。     
 结果：接收端得到的是一个全新的对象实例，位于不同进程的内存空间中。     

```
// 发送端
MyObject obj = new MyObject("data");
binder.sendObject(obj); // 序列化并传输

// 接收端
MyObject receivedObj = binder.receiveObject(); // 反序列化新对象
System.out.println(obj == receivedObj); // false（不同进程）
System.out.println(obj.equals(receivedObj)); // 可能为true（若内容相同）
```

```
// 发送端（Service进程）
public class MyService extends Service {
    private final IMyInterface.Stub binder = new IMyInterface.Stub() {
        @Override
        public void doSomething() { /* 原始对象逻辑 */ }
    };
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder; // 传递Binder引用
    }
}

// 接收端（Client进程）
IMyInterface proxy = IMyInterface.Stub.asInterface(binder); // 获取代理对象
proxy.doSomething(); // 通过代理跨进程调用原始对象
// proxy != 原始binder对象，但可远程调用其方法
```

## App 端常用的两种方式


 - 通过创建 Service，通过 bindService 获取 IBinder。     
 - 通过 ServiceManager.getService() 方式来获取 IBinder。     

那么主要的过程就是必须要获取一个 IBinder 对象，然后转化为代理对象后调用方法，这个过程中涉及到 Binder 的 transact() 进行跨进程通信。     
其实，这里面都绕不开 ServiceManager.getService()，因为通过 Service.bindService 也是也涉及到 App 和 AMS 进行跨进程通信的，需要通过 ServiceManager 来获取 AMS 服务。      



## Binder 通信实例

[千里马Android Framework实战开发-am命令怎么编译生成及native程序与java程序的binder通信实战](https://blog.csdn.net/learnframework/article/details/120029213)      
[千里马Android Framework实战开发-native程序之间binder通信实战案例分析](https://blog.csdn.net/learnframework/article/details/119987323)      


native <--> native

Client:

```
#include <binder/IServiceManager.h>
#include <binder/IBinder.h>
#include <binder/Parcel.h>
#include <binder/ProcessState.h>
#include <binder/IPCThreadState.h>
#include <private/binder/binder_module.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>
#include <binder/Binder.h>

using namespace android;
#ifdef LOG_TAG
#undef LOG_TAG
#endif

#define LOG_TAG "binderCallbackClient"
#define SAMPLE_SERIVCE_DES "my_hello"
#define SAMPLE_CB_SERIVCE_DES "android.os.SampleCallback"
#define SRV_CODE 1
#define CB_CODE 1
class SampeCallback : public BBinder
{
public:
  SampeCallback()
  {
    ALOGE("Client ------------------------------ %d",__LINE__);
    mydescriptor = String16(SAMPLE_CB_SERIVCE_DES);
  }
  virtual ~SampeCallback() {
  }
  virtual const String16& getInterfaceDescriptor() const{
    return mydescriptor;
  }
protected:

  void callbackFunction(int val) {
    ALOGE(" -----------callback Client ok------------------- %d val = %d",__LINE__,val);
  }

  virtual status_t onTransact( uint32_t code, const Parcel& data,Parcel* reply,uint32_t flags = 0){
    ALOGD( "my_hello Client onTransact, line = %d, code = %d",__LINE__,code);
    int val_1,val_2,val_3;
    String8 str_1,str_2,str_3;
    switch (code){
    case CB_CODE:
      //1.读取int32类型数据
      val_1 = data.readInt32();
      val_2 = data.readInt32();
      val_3 = data.readInt32();
      //2.读取String8类型字符串;str_1.string()-->String8转换char类型数组
      str_1 = data.readString8();
      str_2 = data.readString8();
      str_3 = data.readString8();

      callbackFunction(1234567);
      break;

    default:
      return BBinder::onTransact(code, data, reply, flags);
    }
    return 0;
  }
private:
  String16 mydescriptor;
};

int main()
{
    // 获取 ServiceManager
    sp<IServiceManager> sm = defaultServiceManager();
    // 得到 IBinder
    sp<IBinder> ibinder = sm->getService(String16(SAMPLE_SERIVCE_DES));
    if (ibinder == NULL){
        return -1;
    }
    Parcel _data,_reply;
    SampeCallback *callback = new SampeCallback();
    //写入客户端的callback，callback 继承自 BBinder
    _data.writeStrongBinder(sp<IBinder>(callback));
    _data.writeInterfaceToken(String16(SAMPLE_CB_SERIVCE_DES));
    // transact 携带 callback，
    int ret = ibinder->transact(SRV_CODE, _data, &_reply, 0);
    // 
    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```

Server:

```
#include <binder/IServiceManager.h>
#include <binder/IBinder.h>
#include <binder/Parcel.h>
#include <binder/ProcessState.h>
#include <binder/IPCThreadState.h>

using namespace android;
#ifdef LOG_TAG
#undef LOG_TAG
#endif

#define LOG_TAG "sampleService"
#define SAMPLE_SERIVCE_DES "my_hello"
#define SAMPLE_CB_SERIVCE_DES "android.os.SampleCallback"
#define SRV_CODE 1
#define CB_CODE 1

class SampleService: public BBinder {
public:
  SampleService() {
    mydescriptor = String16(SAMPLE_SERIVCE_DES);
  }

  virtual ~SampleService() {
  }

  virtual const String16& getInterfaceDescriptor() const {
    return mydescriptor;
  }

protected:

  virtual status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) {
    ALOGD( "Service onTransact,line = %d, code = %d",__LINE__, code);
    switch (code) {
    case SRV_CODE:
      //读取Client传过来的IBinder对象
      callback = data.readStrongBinder();

      if(callback != NULL)
       {
         Parcel _data, _reply;
	 _data.writeInt32(1);
	 _data.writeInt32(2);
	 _data.writeInt32(3);

	 //2.String8类型
	 _data.writeString8(String8("who..."));
	 _data.writeString8(String8("are..."));
	 _data.writeString8(String8("you..."));
	 //使用客户端发送过来的 callback 回调客户端
         int ret = callback->transact(CB_CODE, _data, &_reply, 0);
       }
      // server 端 dosomething
      break;
    default:
      return BBinder::onTransact(code, data, reply, flags);
    }
    return 0;
  }

private:
  String16 mydescriptor;
  sp<IBinder> callback;
};

int main() {
  sp<IServiceManager> sm = defaultServiceManager();
  SampleService* samServ = new SampleService();
  // 添加
  status_t ret = sm->addService(String16(SAMPLE_SERIVCE_DES), samServ);
  printf("server before joinThreadPool \n");
  IPCThreadState::self()->joinThreadPool( true);
  printf("server after joinThreadPool \n");
  return 0;
}
```

native <--> Java

## 通信流程

Client 部分：

```
ITelephony$Stub$Proxy.getEmergencyCallbackMode
  BinderProxy.transact
    BinderProxy.transactNative
      android_os_BinderProxy_transact()
        BpBinder::transact()
          IPCThreadState::transact()
            IPCThreadState::waitForResponse()
              IPCThreadState::talkWithDriver()
                ioctl()
                  // -----> 进入内核态
```

Server 部分：

```
AndroidRuntime::javaThreadShell
  Thread::_threadLoop
    PoolThread::threadLoop()
      IPCThreadState::joinThreadPool
        IPCThreadState::getAndExecuteCommand()
          IPCThreadState::talkWithDriver
          IPCThreadState::executeCommand
            case BR_TRANSACTION
              BBinder::transact
                JavaBBinder::onTransact
                  
```





整体：

客户端：
```
Stub.Proxy -> BpBinder.transact -> IPCThreadState.transact -> ioctl  --
                                                                      |
                                                                 Binder 驱动
                                                                      |
  Stub <- BBinder.transact <-- getAndExecuteCommand <--  ioctl  <-----
```

## Binder 性能优化

https://mp.weixin.qq.com/s/nheKHn6ghj1FYhpEkhzAWg     

`dumpsys cacheinfo` 命令用于查看 Android 系统中 PropertyInvalidatedCache 的运行状况和统计数据。PropertyInvalidatedCache 是 Android framework 层用于优化跨进程通信的一种内部缓存机制，主要用来缓存一些通过 Binder 调用获取的系统服务数据，以减少 IPC 调用次数，提升系统性能。     
PropertyInvalidatedCache 运行在 Framework 层，是一种更上层的、业务相关的缓存。它的核心逻辑是缓存某些通过 Binder 调用从系统服务获取的数据值。当系统服务的某些配置（如“电池是否低电量”）发生变化时，会通过一个“失效键”通知所有进程，使它们的本地缓存失效，从而保证数据的一致性。     
主要用于优化那些读取频繁但变化不频繁的系统属性或服务数据。例如，一个应用可能需要频繁检查当前设备的充电状态。如果没有缓存，每次检查都会触发一次 Binder 调用，造成性能浪费。有了 PropertyInvalidatedCache，第一次读取后值就被缓存在应用进程本地，后续读取直接从缓存返回，速度极快。     


## 文章


[为什么Android要采用Binder作为IPC机制？](https://www.zhihu.com/question/39440766/answer/89210950)     

[彻底理解Android Binder通信架构](https://gityuan.com/2016/09/04/binder-start-service/)     

[Binder | 代理对象的泄露及其检测 ](https://juejin.cn/post/7024432171779620894)     

[如何解决Binder泄漏问题](https://blog.csdn.net/weiqifa0/article/details/100588966)     

