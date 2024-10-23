---
title: Android 窗口层级树-添加窗口
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍添加窗口流程
date: 2022-11-23 10:00:00
---

## 窗口加载

添加窗口的流程会根据不同的窗口类型而有所区别，主要设计的窗口容器的创建、WindowState初始化以及把WindowState加入到WIndowToken（或者 ActivityRecord）。    

## Activity 窗口加载

从桌面启动 Activity 后，通过对比前后的窗口层级树：    

<img src="/images/android-window-system-add-window/1.png" width="893" height="145"/>

我们会发现，应用窗口 WindowState 挂载到了 ActivityRecord 下面，ActivityRecord 挂载到了 Task 下面，Task 挂载到了 DefaultTaskDisplayArea。    
在这个过程分别创建了 WindowState，Task，ActivityRecord。    


```
ActivityTaskManagerService.startActivity
    ActivityTaskManagerService.startActivityAsUser
        ActivityStarter.execute
            ActivityStarter.executeRequest
                ActivityRecord$Builder.build //创建ActivityRecord
                    ActivityRecord.<init>
                ActivityStarter.startActivityUnchecked()
                    ActivityStarter.startActivityInner
                        ActivityStarter.getOrCreateRootTask // 创建或者获取Task
                            RootWindowContainer.getOrCreateRootTask
                                TaskDisplayArea.getOrCreateRootTask
                                    // 为 Task 设置父窗口 DefaultTaskDisplayArea
                                    Task.Builder.setParent.build() 
                                        Task.buildInner()
                                            Task.<init>
                                        // 把 Task 添加到父窗口 DefaultTaskDisplayArea
                                        TaskDisplayArea.addChild 
                                            TaskDisplayArea.addChildTask
                                                WindowContainer.addChild
                        ActivityStarter.setNewTask    //将task与activityRecord 绑定
                            ActivityStarter.addOrReparentStartingActivity
                                Task.addChild
                                    TaskFragment.addChild
                                        WindowContainer.addChild // 把 ActivityRecord 添加到 Task 上面
                                        // 把 ActivityRecord 的父窗口设置为 Task
                                        WindowContainer.setParent 
                                            ActivityRecord.onDisplayChanged()
                                                WindowToken.onDisplayChanged()
                                                    DisplayContent.reParentWindowToken
                                                        DisplayContent.addWindowToken
                                                            // 把 ActivityRecord 添加到 mTokenMap 列表
                                                            mTokenMap.put 
                                        
```



```
应用侧
WindowManagerImpl::addView
    // 创建ViewRootImpl
    root = new ViewRootImpl()
    WindowManagerGlobal::addView   
        ViewRootImpl::setView     
            //--- 与WMS通信 addWindow
            Session.addToDisplayAsUser

wms侧
WindowManagerService.addWindow
    DisplayContent.getWindowToken  // 获取应用窗口对应的 ActivityRecord
        mTokenMap.get
    win = new WindowState
    WindowState.attach()
    win.mToken.addWindow(win)//WindowToken.addWindow
        ActivityRecord.addChild
            WindowToken.addChild
                WindowContainer.addChild // WindowState 添加到 ActivityRecord 上面
                WindowContainer.setParent // WindowState 的父节点设置为 ActivityRecord
```

## 非 Activity 窗口加载

首先通过 `windowManager.addView` 方式添加一个 `lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY` 的窗口。    

```
    fun addLocalWindow(view:View){
        var windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        var lp = WindowManager.LayoutParams()
        lp.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        lp.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
        lp.format = PixelFormat.RGBA_8888
        lp.gravity = Gravity.CENTER
        lp.width = WindowManager.LayoutParams.WRAP_CONTENT
        lp.height = WindowManager.LayoutParams.WRAP_CONTENT
        lp.windowAnimations = R.style.MyWindowAnimation
        lp.title = "WMSTestActivity 测试窗口"

        windowView = LayoutInflater.from(this).inflate(R.layout.window, null);
        windowManager.addView(windowView, lp)
        hasAddedWindow = true
    }
```

通过对比前后的窗口层级树：      

<img src="/images/android-window-system-add-window/2.png" width="932" height="167"/>

发现在对应层级的 Leaf 节点下面添加了 WindowToken 节点，添加的窗口 WindowState 挂载到了 WindowToken 节点。   
具体流程如下：    

```
WindowManagerService.addWindow
    WindowToken$Builder.build()
        WindowToken.<init>
            DisplayContent.addWindowToken
                //把创建的 WindowToken放到 mTokenMap 列表
                mTokenMap.put(binder, token)
                // 找到对应的层级
                DisplayContent.findAreaForToken
                    DisplayContent.findAreaForWindowType
                        DisplayAreaPolicyBuilder$Result.findAreaForWindowType
                            RootDisplayArea.getWindowLayerFromTypeLw
                                // 找出 TYPE_APPLICATION_OVERLAY 对应的层级 11
                                WindowManagerPolicy.getWindowLayerFromTypeLw 
                                return mAreaForLayer[windowLayerFromType]
                DisplayArea.Tokens.addChild()
    win = new WindowState
    WindowState.attach()
    mWindowMap.put() // 保存WindowState和客户端窗口的映射关系
    win.mToken.addWindow(win)//WindowToken.addWindow
        ActivityRecord.addChild
            WindowToken.addChild
                WindowContainer.addChild // WindowState 添加到 WindowToken 上面
                WindowContainer.setParent // WindowState 的父节点设置为 WindowToken
```

## addWindow

```
WindowManagerService.addWindow
    DisplayContent.getWindowToken  // 获取应用窗口对应的 ActivityRecord
        mTokenMap.get
    WindowToken$Builder.build() // 或者新建 WindowToken
    win = new WindowState // 创建 WindowState
    WindowState.attach()
    mWindowMap.put() // 保存WindowState和客户端窗口的映射关系
    win.mToken.addWindow(win)//WindowToken.addWindow
        ActivityRecord.addChild
            WindowToken.addChild
                WindowContainer.addChild // WindowState 添加到 ActivityRecord 上面
                WindowContainer.setParent // WindowState 的父节点设置为 ActivityRecord
                    WindowContainer.onParentChanged()
                        // 调整窗口Z-order
                        mParent.assignChildLayers() // WindowContainer.assignChildLayers()
```

```
//WindowManagerService.java

    public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
            ......
            //判断mWindowMap中是否已经存在当前客户端的key,如果有则已经将当前客户端的window添加了，无需重复添加
            if (mWindowMap.containsKey(client.asBinder())) {
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }
            ......
            // 根据客户端传递过来的 token获取windowToken
            // 如果创建 Activity 窗口时，那么在 startActivity 时会创建 ActivityRecord，
            // 并加入到 mTokenMap 列表；
            // 创建非 Activity 窗口时 hasParent 为false，如果没有设置token，这里通过 displayContent 获取的token为空；
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            ......
            if (token == null) {
                if (!unprivilegedAppCanCreateTokenWith(parentWindow, callingUid, type,
                        rootType, attrs.token, attrs.packageName)) {
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (hasParent) {
                    // Use existing parent window token for child windows.
                    token = parentWindow.mToken;
                } else if (mWindowContextListenerController.hasListener(windowContextToken)) {
                    ......
                } else {
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    // 创建 WindowToken
                    token = new WindowToken.Builder(this, binder, type)
                            .setDisplayContent(displayContent)
                            .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                            .setRoundedCornerOverlay(isRoundedCornerOverlay)
                            .build();
                }
            } else if (rootType >= FIRST_APPLICATION_WINDOW
                    && rootType <= LAST_APPLICATION_WINDOW) {
                //当前窗口为应用窗口，通过token，获取ActivityRecord
                activity = token.asActivityRecord();
            }
            ......
            //创建WindowState
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
            ......
            win.attach();
            //将客户端与WindowState加入到mWindowMap中
            mWindowMap.put(client.asBinder(), win);
            win.initAppOpsState();
            ......
            //将WindowState加入到WindowToken
            win.mToken.addWindow(win);
                    
```

### 调整 Z-order

```
    void assignChildLayers(Transaction t) {
        //1.初始化layer=0，代表着z-order。
        int layer = 0;

        // 2.遍历mChildren数组，判断Children是否需要提高到顶部（判断标志位mNeedsZBoost）。
        //如果不需要则调用Children的assignLayer方法调整其z-order为layer，
        //并将layer++。如果需要则执行下一遍循环。
        for (int j = 0; j < mChildren.size(); ++j) {
            final WindowContainer wc = mChildren.get(j);
            wc.assignChildLayers(t);
            if (!wc.needsZBoost()) {
                wc.assignLayer(t, layer++);
            }
        }
        //3. 再次遍历mChildren数组，判断Children是否需要提高到顶部。
        // 如果需要则则调用Children的assignLayer方法调整其z-order为layer，
        // 并将layer++。如果不需要则执行下一次循环。
        for (int j = 0; j < mChildren.size(); ++j) {
            final WindowContainer wc = mChildren.get(j);
            if (wc.needsZBoost()) {
                wc.assignLayer(t, layer++);
            }
        }
        if (mOverlayHost != null) {
            mOverlayHost.setLayer(t, layer++);
        }
    }
```







