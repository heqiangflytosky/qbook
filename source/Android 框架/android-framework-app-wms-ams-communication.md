---
title: Android APP 和 WMS AMS 通信
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 Android APP 和 WMS AMS 通信
date: 2022-11-23 10:00:00
---





## App -> WMS 和 WMS -> App

IWindowSession 和 IWindow    

App 可以通过 IWindowSession 接口通信到 server 的 Session，至于 IWindowSession 的实例 App 可以通过 WindowManagerGlobal 获取到 。    

```
// WindowManagerGlobal.java
    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    // Emulate the legacy behavior.  The global instance of InputMethodManager
                    // was instantiated here.
                    // TODO(b/116157766): Remove this hack after cleaning up @UnsupportedAppUsage
                    InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                    IWindowManager windowManager = getWindowManagerService();
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            });
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
```

从这里可以看到，IWindowSession 在 App 进程中是单例的。           

比如，添加窗口时：


```
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
            ......
                    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibleTypes(), inputChannel, mTempInsets,
                            mTempControls, attachedFrame, compatScale);
```

mWindow 是在 App 侧的 IWindow.aidl 实现，传递到 WMS，方便 WMS 与 App 通信。   
它的实现在 App 的 ViewRootImpl中，server 调用 IWindow 接口的方法，就可以调用到这里来。      

```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {
        ......
    static class W extends IWindow.Stub implements WindowStateTransactionItem.TransactionListener {
    ......
        @Override
        public void dispatchAppVisibility(boolean visible) {
            final ViewRootImpl viewAncestor = mViewAncestor.get();
            if (viewAncestor != null) {
                viewAncestor.dispatchAppVisibility(visible);
            }
        }
```

Session 是 server 端 IWindowSession 的实现。    


```
class Session extends IWindowSession.Stub
```


```
// WindowManagerService.java
    public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            int displayId, int requestUserId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
            ......
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
```

把 IWindow 保存在 WindowState 的 mClient 变量中。     

```
    WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState parentWindow, int appOp, WindowManager.LayoutParams a, int viewVisibility,
            int ownerId, int showUserId, boolean ownerCanAddInternalSystemWindow) {
        super(service);
        mTmpTransaction = service.mTransactionFactory.get();
        mSession = s;
        mClient = c;
```

WMS 如果想和 App 通信，就可以通过 WindowState.mClient 来调用接口。    
比如：    

```
    void sendAppVisibilityToClients() {
        ......

        try {
            ......
            mClient.dispatchAppVisibility(clientVisible);
```



## App -> WMS AMS

IActivityManager

App 调用 server 可以通过 IActivityManager 接口的方法：

```
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
```

通过 ActivityManager.getService() 获取 IActivityManager 实例，就可以调到 server 的AMS 的相关方法。     
调用 attachApplication 把 `ApplicationThread mAppThread` 实例传递过去，server 就可以和 App 进行通信了。     


IApplicationThread 在 App 侧的实现：

server 端调用 IApplicationThread 接口方法就会调用到这里来：    

```

public final class ActivityThread extends ClientTransactionHandler
        implements ActivityThreadInternal {
        
    private class ApplicationThread extends IApplicationThread.Stub {
    
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

## WMS AMS -> App

IApplicationThread

WMS

WindowState 中通过 mSession 可以获取到 WindowProcessController 对象，然后获取 IApplicationThread。     

```
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState,
        InsetsControlTarget, InputTarget {
    final Session mSession;
```

```
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    ......
    final WindowProcessController mProcess;
```

ActivityRecord 中有 app 变量可以获取到 WindowProcessController 的 IApplicationThread 对象。    

```
public final class ActivityRecord extends WindowToken implements WindowManagerService.AppFreezeListener {

    WindowProcessController app;  
```

```
public class WindowProcessController extends ConfigurationContainer<ConfigurationContainer>
        implements ConfigurationContainerListener {
        ......
    private IApplicationThread mThread;
```

IApplicationThread 初始化时机：

```
ActivityManagerService.attachApplication
    ActivityManagerService.attachApplicationLocked
        ProcessRecord.makeActive
            WindowProcessController.setThread
```

WindowProcessController 初始化时机：

```
ActivityManagerService.attachApplication
    ActivityManagerService.attachApplicationLocked
        ActivityManagerService.finishAttachApplicationInner
            ActivityTaskManagerService$LocalService.attachApplication
                RootWindowContainer.attachApplication
                    ActivityTaskSupervisor.realStartActivityLocked
                        ActivityRecord.setProcess
                            WindowProcessController app = proc
```

WindowState WindowProcessController 初始化时机，通过初始化 Session：

```
Session.addToDisplayAsUser
    WindowManagerService.addWindow
        new WindowState
            Session mSession = s
```

通信举例：

比如 ClientTransaction 的实现其实也是借助于 IApplicationThread 和 App 进行通信的：    

```
public class ClientTransaction implements Parcelable, ObjectPoolItem {
    private IApplicationThread mClient;
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
```

## 相关文章

[Android 应用程序建立与WMS服务之间的通信过程](https://www.pianshen.com/article/295225034/)       
[App、WMS、AMS如何通信](https://blog.csdn.net/weixin_41126214/article/details/132926615)       


