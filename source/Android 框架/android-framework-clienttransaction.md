---
title: Android AMS 之 ClientTransaction 生命周期管理机制
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 Android AMS 之 ClientTransaction 生命周期管理机制
date: 2022-11-23 10:00:00
---

## 相关类简介

ClientTransaction：从Android 9.0开始，Android系统引入了 ClientTransaction 来简化 system_server 与 App 进程之间处理 Activity 启动相关的任务。这一改变使得代码更加简洁，提高了系统的效率和可维护性。      
ClientTransaction 相当于封装一系列消息的容器。 在 Android 系统中用于在 Activity 启动过程中负责在客户端（如App进程）和服务端（如 system_server 进程）之间传递请求和响应，从而启动和管理 Activity 的生命周期。      

```
// ClientTransaction.java
    // app端需要执行的 ClientTransactionItem 列表
    @UnsupportedAppUsage
    private List<ClientTransactionItem> mActivityCallbacks;

    // 执行完transaction后 activity 的最终的生命周期状态
    private ActivityLifecycleItem mLifecycleStateRequest;

    // 对应的app的ApplicationThread，它是一个Binder对象
    private IApplicationThread mClient;

    // app端activity的binder对象
    private IBinder mActivityToken;
```

ClientTransactionItem 对象，一个回调消息，实现了 Parcelable 接口，可以被用于进程间传递，`system_server` 创建，App 端执行。      
ActivityLifecycleItem 继承自 ClientTransctionItem，主要的子类有 ResumeActivityItem、PauseActivityItem、StopActivityItem、DestoryActivityItem。      
另外介绍一下 LaunchActivityItem，它不是继承自 ActivityLifecycleItem，而是继承自 ClientTransctionItem，顾名思义就是主要是来启动 Activity 的（冷启动）。     

```
public abstract class ActivityLifecycleItem extends ActivityTransactionItem {

    @IntDef(prefix = { "UNDEFINED", "PRE_", "ON_" }, value = {
            UNDEFINED,
            PRE_ON_CREATE,
            ON_CREATE,
            ON_START,
            ON_RESUME,
            ON_PAUSE,
            ON_STOP,
            ON_DESTROY,
            ON_RESTART
    })
```

<img src="/images/android-framework-clienttransaction/1.png" width="1009" height="300"/>

Android 10 新增了一项 TopResumedActivityChangeItem 事务，该事务用于向客户端报告在顶层处于已恢复状态的更改。     
该状态的相关信息存储在客户端，每次 Activity 转换为 RESUMED 或 PAUSED 时，系统还会检查是否应该调用 onTopResumedActivityChanged() 回调。这可以在服务器和客户端之间就生命周期状态和“在顶层处于 RESUMED 状态”这一状态进行通信时实现某些分离。     

ActivityConfigurationChangeItem:当配置旋转屏幕不重建 Activity时通知Activity调用 onConfigurationChanged      

ClientLifecycleManager 是管理 Activity 生命周期的，在 ActivityTaskManagerService 里面提供 getLifecycleManager 来获取此对象。     
TransactionExecutor 用来执行 ClientTransaction，定义在 ActivityThread 应用端。     

## 代码分析

以 Launch Activity 为例来介绍。     

system_server：

```
ActivityTaskSupervisor.realStartActivityLocked()
    ClientTransaction.obtain()
    LaunchActivityItem.obtain()
    ClientTransaction.addCallback()
    ResumeActivityItem.obtain()
    ClientTransaction.setLifecycleStateRequest()
    ClientLifecycleManager.scheduleTransaction()
        ClientTransaction.schedule()
            IApplicationThread.scheduleTransaction() // 跨进程到 App 进程执行
```

App：

```
ActivityThread.ApplicationThread.scheduleTransaction
    ClientTransactionHandler.scheduleTransaction()
        ClientTransaction.preExecute()
            LaunchActivityItem.preExecute()
            ResumeActivityItem.preExecute()
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)
            ActivityThread.H.handleMessage()
                TransactionExecutor.execute()
                    TransactionExecutor.executeCallbacks()
                        LaunchActivityItem.execute()
                            ActivityThread.handleLaunchActivity()
                                ActivityThread.performLaunchActivity()
                                    Instrumentation.callActivityOnCreate()
                                        Activity.performCreate()
                                            Activity.onCreate()
                                    setState(ON_CREATE)
                        LaunchActivityItem.postExecute()
                    TransactionExecutor.executeLifecycleState()
                        TransactionExecutor.cycleToPath()
                            TransactionExecutorHelper.getLifecyclePath()
                            TransactionExecutor.performLifecycleSequence()
                                    ActivityThread.handleStartActivity()
                                        Activity.performStart()
                                            Instrumentation.callActivityOnStart()
                                                Activity.onStart()
                                        setState(ON_START)
                        ResumeActivityItem.execute()
                            ActivityThread.handleResumeActivity()
                                ActivityThread.performResumeActivity()
                                    Activity.performResume()
                                        Instrumentation.callActivityOnResume()
                                            Activity.onResume()
                                    setState(ON_RESUME)
                        ResumeActivityItem.postExecute()
```

```
// ActivityTaskSupervisor.java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
                // 创建 launch activit 的 ClientTransaction
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.token);
                ...
                // Set desired final state.
                // 设置目标状态
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(isTransitionForward,
                            r.shouldSendCompatFakeFocus());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                // 执行 transaction
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```

在这个方法中，会创建一个 ClientTransaction 对象，它会被传递到客户进程，同时给 ClientTransaction 实例添加 LaunchActivityItem 形式的 CallBack 和设置 setLifecycleStateRequest。    
启动Activity就是创建了一个LaunchActivityItem并且设置了对应的LifecycleStateRequest，最后是通过LifecycleManager调用scheduleTransaction来执行。    


ClientTransaction 的获取是从一个对象池中获取的，分别设置了mClient和 mActivityToken 这里不做过多介绍。    

```
    public static ClientTransaction obtain(IApplicationThread client, IBinder activityToken) {
        ClientTransaction instance = ObjectPool.obtain(ClientTransaction.class);
        if (instance == null) {
            instance = new ClientTransaction();
        }
        instance.mClient = client;
        instance.mActivityToken = activityToken;

        return instance;
    }
```

调度执行：

```
//ClientLifecycleManager.java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
```

```
//ClientTransaction.java
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }

```

然后就转到 App 端执行。     

```
//ActivityThread.java
//ApplicationThread
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

这里的 ClientTransaction 就是在 system_server 中创建的对象。    
ActivityThread是ClientTransactionHandler的子类，所以调用了 ClientTransactionHandler的scheduleTransaction方法。    
```
//ClientTransactionHandler.java
    void scheduleTransaction(ClientTransaction transaction) {
        // 执行执行 preExecute 方法，其实就是调用 activityCallback的preExecute方法，
        // 以及mLifecycleStateRequest的preExecute方法，这里分别指的是 LaunchActivityItem 和 ResumeActivityItem
        transaction.preExecute(this);
        // 发送消息，执行 execute
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

```
//ActivityThread.java
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
```

调用 TransactionExecutor来执行ClientTransaction。     

```
    public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

        final IBinder token = transaction.getActivityToken();
        if (token != null) {
            ......
        }

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

```
//TransactionExecutor.java
    public void executeCallbacks(ClientTransaction transaction) {
        ......

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            ......

            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            if (r == null) {
                // Launch activity request will create an activity record.
                r = mTransactionHandler.getActivityClient(token);
            }

            if (postExecutionState != UNDEFINED && r != null) {
                // Skip the very last transition and perform it by explicit state request instead.
                final boolean shouldExcludeLastTransition =
                        i == lastCallbackRequestingState && finalState == postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
            }
        }
    }
    
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ......

        // Cycle to the state right before the final requested state.
        // 获取 ResumeActivityItem 的 getTargetState，这里 excludeLastState 设置为true，从而让Activity进入ON_START状态而不是ON_RESUME状态。
        // 因为后面还会调用ResumeActivityItem.execute来设置 ON_RESUME 状态，具体见下面代码分析。    
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```
这里其实就是执行了 ClientTransaction 设置的 Callbacks 和 executeLifecycleState的 execute 和 postExecute 方法，对应 LaunchActivityItem 和 ResumeActivityItem 的 的 execute 和 postExecute 方法。    

```
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mActivityOptions, mIsForward, mProfilerInfo,
                client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
                mTaskFragmentToken);
        client.handleLaunchActivity(r, pendingActions, mDeviceId, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.countLaunchingActivities(-1);
    }
```

```
//ResumeActivityItem.java
    @Override
    public void execute(ClientTransactionHandler client, ActivityClientRecord r,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        client.handleResumeActivity(r, true /* finalStateRequest */, mIsForward,
                mShouldSendCompatFakeFocus, "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        // TODO(lifecycler): Use interface callback instead of actual implementation.
        ActivityClient.getInstance().activityResumed(token, client.isHandleSplashScreenExit(token));
    }
```

各自方法里面分别执行了 ActivityThread 里面的 handleLaunchActivity() 和 handleResumeActivity() 等方法。    

另外在 executeLifecycleState 方法中执行 lifecycleItem.execute 前先执行了 cycleToPath 方法。    
首先获取 ResumeActivityItem 的 getTargetState，然后这里 excludeLastState 设置为true，从而让Activity进入ON_START状态而不是ON_RESUME状态。     
因为后面还会调用ResumeActivityItem.execute来设置 ON_RESUME 状态。     

```
    private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        // 当前的状态是刚刚设置的 ON_CREATE
        final int start = r.getLifecycleState();
        ......
        此时的 finish 状态为 ON_RESUME，由于 excludeLastState 为true，因此 path 就是 ON_RESUME ON_START
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
    }
```

```
    public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
        if (start == UNDEFINED || finish == UNDEFINED) {
            throw new IllegalArgumentException("Can't resolve lifecycle path for undefined state");
        }
        if (start == ON_RESTART || finish == ON_RESTART) {
            throw new IllegalArgumentException(
                    "Can't start or finish in intermittent RESTART state");
        }
        if (finish == PRE_ON_CREATE && start != finish) {
            throw new IllegalArgumentException("Can only start in pre-onCreate state");
        }

        mLifecycleSequence.clear();
        if (finish >= start) {
            if (start == ON_START && finish == ON_STOP) {
                // A case when we from start to stop state soon, we don't need to go
                // through the resumed, paused state.
                mLifecycleSequence.add(ON_STOP);
            } else {
                // just go there
                for (int i = start + 1; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        } else { // finish < start, can't just cycle down
            if (start == ON_PAUSE && finish == ON_RESUME) {
                // Special case when we can just directly go to resumed state.
                mLifecycleSequence.add(ON_RESUME);
            } else if (start <= ON_STOP && finish >= ON_START) {
                // Restart and go to required state.

                // Go to stopped state first.
                for (int i = start + 1; i <= ON_STOP; i++) {
                    mLifecycleSequence.add(i);
                }
                // Restart
                mLifecycleSequence.add(ON_RESTART);
                // Go to required state
                for (int i = ON_START; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            } else {
                // Relaunch and go to required state

                // Go to destroyed state first.
                for (int i = start + 1; i <= ON_DESTROY; i++) {
                    mLifecycleSequence.add(i);
                }
                // Go to required state
                for (int i = ON_CREATE; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        }

        // Remove last transition in case we want to perform it with some specific params.
        if (excludeLastState && mLifecycleSequence.size() != 0) {
            mLifecycleSequence.remove(mLifecycleSequence.size() - 1);
        }

        return mLifecycleSequence;
    }
```

得到执行的path后再去执行 ActivityThread 中对应的方法。    

```
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            Context.DEVICE_ID_INVALID, null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions,
                            null /* activityOptions */);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r, false /* finalStateRequest */,
                            r.isForward, false /* shouldSendCompatFakeFocus */,
                            "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */,
                            false /* autoEnteringPip */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
```






<!--
```
@startuml
ObjectPoolItem <|-- BaseClientRequest
BaseClientRequest <|-- ClientTransactionItem
Parcelable <|-- ClientTransactionItem
ClientTransactionItem <|-- ActivityTransactionItem
ClientTransactionItem <|-- LaunchActivityItem

ActivityTransactionItem <|-- TopResumedActivityChangeItem
ActivityTransactionItem <|-- RefreshCallbackItem
ActivityTransactionItem <|-- EnterPipRequestedItem
ActivityTransactionItem <|-- NewIntentItem

ActivityTransactionItem <|-- ActivityLifecycleItem
ActivityLifecycleItem <|-- StartActivityItem
ActivityLifecycleItem <|-- PauseActivityItem
ActivityLifecycleItem <|-- ResumeActivityItem
ActivityLifecycleItem <|-- StopActivityItem
ActivityLifecycleItem <|-- DestroyActivityItem

ActivityTransactionItem <|-- ActivityConfigurationChangeItem
ActivityTransactionItem <|-- ActivityRelaunchItem
ActivityTransactionItem <|-- ActivityResultItem
ActivityTransactionItem <|-- MoveToDisplayItem


@enduml
```

-->
