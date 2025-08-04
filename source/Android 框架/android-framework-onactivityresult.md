---
title: Activity 之 onActivityResult 原理
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 Activity 之 onActivityResult 原理
date: 2022-11-23 10:00:00
---


## 概述

Android Activity 中提供 setResult 和 onActivityResult 方法，可以使 Activity 之间进行数据传递，本文从源码角度来看一下他们的工作流程。    
本文基于 Android 15。    

## 应用方法

### 两个 Activity 之间传值

启动 Activity 时传入 requestCode:

```
startActivityForResult(intent,101);
```

在第一个Activity实现 onActivityResult 方法：

```
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if(requestCode == 101){
            when(resultCode){
                Activity.RESULT_OK -> {
                    Log.e(tag,"onActivityResult RESULT_OK")
                }
                Activity.RESULT_CANCELED -> {
                    Log.e(tag,"onActivityResult RESULT_CANCELED")
                }
            }
        }
    }
```

必须在第二个 Activity 的finish() 方法调用之前设置 Result。    

```
    override fun finish() {
        setResult(RESULT_OK, intent)
        super.finish()
    }
```

注意：如果启动Activity时添加了 `intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)`，在启动时会直接返回 `Activity.RESULT_CANCELED`，不会返回 Result。

### 多个 Activity 之间传值

使用场景：
ActivityA --> ActivityB --> ActivityC 三个 Activity 之间跳转，A 启动 B 时需要回传数据，B 添加数据后，如果启动C，C也需要添加数据，返回后一起交给 A。     
这种场景下，可以在 B 启动 C 时，在 Intent 中添加 `intent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT)`，这个 Flag 的作用就是C的 Result 可以不经过 B，而直接传给 A。    

ActivityA：

```
startActivityForResult(intent,101);
```

```
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if(requestCode == 101){
            when(resultCode){
                Activity.RESULT_OK -> {
                    Log.e(tag,"onActivityResult RESULT_OK ${data?.getStringExtra("result")}, ${data?.getStringExtra("result1")}")
                }
                Activity.RESULT_CANCELED -> {
                    Log.e(tag,"onActivityResult RESULT_CANCELED")
                }
            }
        }
    }
```

ActivityB:

启动ActivityC时：

```
    fun button1(view: View) {
        var intent = Intent(this, CommonTestActivity2::class.java)
        // 添加 FLAG_ACTIVITY_FORWARD_RESULT
        intent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT)

        // 添加自己的数据
        setResult(RESULT_OK, intent.putExtra("result","CommonTestActivity"))

        startActivity(intent)
        finish()
    }
```

ActivityC：

```
    fun button1(view: View) {
        // 需要在同一个 intent 中添加自己数据，才能把两个 Activity 的数据一起传给 ActivityA
        intent.putExtra("result1","CommonTestActivity2")
        setResult(RESULT_OK, intent)
        finish()
    }
```

## 框架流程

```
ActivityTaskManagerService.startActivity
    ActivityTaskManagerService.startActivityAsUser
        ActivityTaskManagerService.startActivityAsUser
            ActivityStarter.setResultTo()
            ActivityStarter.execute()
                ActivityStarter.executeRequest()
                    // 配置 resultRecord
                    resultRecord = sourceRecord
                    resultRecord = sourceRecord.resultTo;
                    ActivityRecord.Builder.setResultTo(resultRecord)
                    ActivityRecord.Builder.build()
                        new ActivityRecord(mResultTo,mRequestCode)
                    ActivityStarter.startActivityUnchecked
                        ActivityStarter.startActivityInner
                            ActivityStarter.setInitialState
                                ActivityStarter.sendNewTaskResultRequestIfNeeded()
                                    // 如果设置 FLAG_ACTIVITY_NEW_TASK，直接发送 RESULT_CANCELED
                                    if (mStartActivity.resultTo != null && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
                                    mStartActivity.resultTo.sendResult(RESULT_CANCELED)
```

```
//ActivityStarter.java

    private int executeRequest(Request request) {
    ...
        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            sourceRecord = ActivityRecord.isInAnyTask(resultTo);
            if (DEBUG_RESULTS) {
                Slog.v(TAG_RESULTS, "Will send result to " + resultTo + " " + sourceRecord);
            }
            if (sourceRecord != null) {
                //当 requestCode >= 0时，才会为 resultRecord 赋值
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
        // 处理 FLAG_ACTIVITY_FORWARD_RESULT Flag
        final int launchFlags = intent.getFlags();
        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
            // Transfer the result target from the source activity to the new one being started,
            // including any failures.
            if (requestCode >= 0) {
                SafeActivityOptions.abort(options);
                return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
            }
            // 这里将 resultRecord 值修改为 sourceRecord.resultTo，也就是上一个Activity的resultTo
            resultRecord = sourceRecord.resultTo;
            if (resultRecord != null && !resultRecord.isInRootTaskLocked()) {
                resultRecord = null;
            }
            resultWho = sourceRecord.resultWho;
            requestCode = sourceRecord.requestCode;
            sourceRecord.resultTo = null;
            if (resultRecord != null) {
                resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
            }
            if (sourceRecord.launchedFromUid == callingUid) {
                callingPackage = sourceRecord.launchedFromPackage;
                callingFeatureId = sourceRecord.launchedFromFeatureId;
            }
        }
    ...
    }
```

当第二个 Activity finish 时，

```
ActivityClientController.finishActivity
    ActivityRecord.finishIfPossible
        ActivityRecord.finishActivityResults
            // 将Result添加到第一个 Activity，
            // 此时的 ActivityRecord 为 resultTo
            ActivityRecord.addResultLocked()
                new ActivityResult()
                results.add(r)

```

当第一个 Activity 执行 resume 时，

```
ActivityClientController.activityPaused
    ActivityRecord.activityPaused
        TaskFragment.completePause
            RootWindowContainer.resumeFocusedTasksTopActivities
                RootWindowContainer.resumeFocusedTasksTopActivities
                    Task.resumeTopActivityUncheckedLocked
                        Task.resumeTopActivityInnerLocked
                            TaskFragment.resumeTopActivity
                                ActivityResultItem.obtain
                                ClientLifecycleManager.scheduleTransactionItem()
                                    ClientLifecycleManager.getOrCreatePendingTransaction()
                                    ClientTransaction.addTransactionItem()
                                        mTransactionItems.add(item)
        ActivityTaskManagerService.continueWindowLayout
            WindowSurfacePlacer.continueLayout
                WindowSurfacePlacer.performSurfacePlacement
                    WindowSurfacePlacer.performSurfacePlacement
                        WindowSurfacePlacer.performSurfacePlacementLoop
                            RootWindowContainer.performSurfacePlacement
                                RootWindowContainer.performSurfacePlacementNoTrace
                                    ClientLifecycleManager.dispatchPendingTransactions
                                        // 调度执行 ClientTransaction
                                        ClientLifecycleManager.scheduleTransaction
                                            ClientTransaction.schedule
                                                IApplicationThread$Stub$Proxy.scheduleTransaction
```

system_server 创建 ActivityResultItem

```
//TaskFragment.java

    final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean skipPause) {
            ......
                // Deliver all pending results.
                final ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int size = a.size();
                    if (!next.finishing && size > 0) {
                        ....
                        final ActivityResultItem activityResultItem = ActivityResultItem.obtain(
                                next.token, a);
                        if (transaction == null) {
                            mAtmService.getLifecycleManager().scheduleTransactionItem(
                                    appThread, activityResultItem);
                        } else {
                            transaction.addTransactionItem(activityResultItem);
                        }
                    }
                }
            ....
```

App 端：


```
ActivityThread$ApplicationThread.scheduleTransaction
    ClientTransactionHandler.scheduleTransaction
        ClientTransaction.preExecute
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)
            ActivityThread$H.handleMessage
                TransactionExecutor.execute
                    TransactionExecutor.executeTransactionItems
                        TransactionExecutor.executeNonLifecycleItem
                            ActivityTransactionItem.execute
                                ActivityResultItem.execute
                                    ActivityThread.handleSendResult
                                        ActivityThread.deliverResults
                                           Activity.dispatchActivityResult
                                               Activity.internalDispatchActivityResult
                                                   Activity.onActivityResult
```


ActivityResult 和 `FLAG_ACTIVITY_NEW_TASK` 的关系：     

```
    private void sendNewTaskResultRequestIfNeeded() {
        if (mStartActivity.resultTo != null && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            // 无论出于何种原因，这项活动正在启动到一项新任务中......然而，呼叫者已要求返回结果。
            // 好吧，这很混乱，所以立即发回取消并让新任务继续正常启动，而不依赖于其发起者。
            // For whatever reason this activity is being launched into a new task...
            // yet the caller has requested a result back.  Well, that is pretty messed up,
            // so instead immediately send back a cancel and let the new task continue launched
            // as normal without a dependency on its originator.

            Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
            mStartActivity.resultTo.sendResult(INVALID_UID, mStartActivity.resultWho,
                    mStartActivity.requestCode, RESULT_CANCELED,
                    null /* data */, null /* callerToken */, null /* dataGrants */);
            mStartActivity.resultTo = null;
        }
    }
```
