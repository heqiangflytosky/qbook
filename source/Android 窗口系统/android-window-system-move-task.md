---
title: Android 窗口系统-移栈
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口层级移栈的流程
date: 2022-11-23 10:00:00
---


## 概述

前面我们在介绍构建窗口层级的意义时讲过，窗口层级的构建使得系统方便对窗口进行管理，这使得移栈这种操作也简单和便利。     
移动应用进程，其实移动就是就是对应Activity所在的Stack。     
本文就介绍一下窗口在不同的显示设备之间的移动流程。       


## 实现

在开发者选项中打开模拟辅助显示设备，就会开启另一个虚拟屏幕。    
用 `adb shell dumpsys activity containers` 此时的窗口层级情况是：    

```
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  #1 Display 0 name="内置屏幕" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1368,3192] bounds=[0,0][1368,3192]
  ......
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
        #3 Task=117 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
         #0 ActivityRecord{b2468e4 u0 com.hq.android.androiddemo/.MainActivity t117} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
          #0 161f95b com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  ......
  #0 Display 5 name="叠加视图 #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1920,1080] bounds=[0,0][1920,1080]
  ......
     #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
     ......
```

执行移栈命令： `adb shell am display move-stack 117 5`       
窗口层级树为：     

```
ACTIVITY MANAGER CONTAINERS (dumpsys activity containers)
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
  #1 Display 5 name="叠加视图 #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1920,1080] bounds=[0,0][1920,1080]
  ......
     #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
      #0 Task=117 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
       #0 ActivityRecord{b2468e4 u0 com.hq.android.androiddemo/.MainActivity t117} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]
        #0 3bdaa62 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1080]

  ......
  #0 Display 0 name="内置屏幕" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][1368,3192] bounds=[0,0][1368,3192]
  ......
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1368,3192]
```

可见，Task=117 从 Display 0 移动到了 Display 5 。    

## 代码流程

原理其实就是把 Task 容器重新挂载到新的 display 的 TaskDisplayArea 上。    

### 绑定Display和TextureView

前面说过，一个 DisplayContent 代表一个屏幕，那么我们创建虚拟屏幕就会创建一个新的 DisplayContent。    


因为我们手机就一个屏幕，无法用硬件设备来显示新建的 DisplayContent，那么就在手机上创建一个新的 Window 用来显示虚拟屏幕内容。    
承载虚拟屏幕显示的是 OverlayDisplayWindow 类。   
它会在主屏幕上新建一个窗口，布局中的 TextureView 来提供对应的 Surface 来显示对应虚拟屏幕数据。    

```
OverlayDisplayAdapter.registerLocked().ContentObserver
    OverlayDisplayAdapter.updateOverlayDisplayDevices()
        OverlayDisplayAdapter.updateOverlayDisplayDevicesLocked()
            new OverlayDisplayHandle
                OverlayDisplayHandle.showLocked()
                    mShowRunnable.run
                        new OverlayDisplayWindow
                            OverlayDisplayWindow.createWindow()
                                TextureView.setSurfaceTextureListener
                                    SurfaceTextureListener.onSurfaceTextureAvailable
                                        OverlayDisplayHandle.onWindowCreated()
                                            // 通知 SF 创建 Display
                                            displayToken = DisplayControl.createDisplay()
                                                DisplayControl.nativeCreateDisplay
                                            // 创建 DisplayDevice，传入 displayToken 和 surfaceTexture
                                            new OverlayDisplayDevice(displayToken,...,surfaceTexture)
                                            sendDisplayDeviceEventLocked(mDevice, DISPLAY_DEVICE_EVENT_ADDED)
                                                DisplayDeviceRepository.handleDisplayDeviceAdded
                                                    DisplayDeviceRepository.sendEventLocked(device, DISPLAY_DEVICE_EVENT_ADDED)
                                                        LogicalDisplayMapper.handleDisplayDeviceAddedLocked()
                                                            LogicalDisplayMapper.updateLogicalDisplaysLocked()
                                                                LogicalDisplayMapper.sendUpdatesForDisplaysLocked(LOGICAL_DISPLAY_EVENT_ADDED)
                                                                    DisplayManagerService.onLogicalDisplayEventLocked
                                                                        DisplayManagerService.handleLogicalDisplayAddedLocked
                                                                            DisplayManagerService.sendDisplayEventLocked(display, DisplayManagerGlobal.EVENT_DISPLAY_ADDED) // 转下面接收 Message 流程
                        OverlayDisplayWindow.show()
                            DisplayManager.registerDisplayListener
                                
```

createWindow() 时为 TextureView 注册监听，当 onSurfaceTextureAvailable 回调时调用 OverlayDisplayHandle.onWindowCreated() 创建一个 OverlayDisplayDevice，把 SurfaceTexture 作为参数传入。来显示对应 DisplayDevice 的内容。        

```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:background="#000000">
    <TextureView android:id="@+id/overlay_display_window_texture"
               android:layout_width="0px"
               android:layout_height="0px" />
    <TextView android:id="@+id/overlay_display_window_title"
               android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:layout_gravity="top|center_horizontal" />
</FrameLayout>
```


```
//OverlayDisplayAdapter.java

        public void performTraversalLocked(SurfaceControl.Transaction t) {
            if (mSurfaceTexture != null) {
                if (mSurface == null) {
                    // 
                    mSurface = new Surface(mSurfaceTexture);
                }
                setSurfaceLocked(t, mSurface);
            }
        }
    // 绑定 Surface 到 SF 创建的 Display，这样前面创建的 TextureView 就会显示虚拟Display的内容    
    public final void setSurfaceLocked(SurfaceControl.Transaction t, Surface surface) {
        if (mCurrentSurface != surface) {
            mCurrentSurface = surface;
            t.setDisplaySurface(mDisplayToken, surface);
        }
    }
```

### 创建新的 DisplayContent

收到创建虚拟设备消息后，RootWindowContainer 创建新的 DisplayContent

```
DisplayManagerGlobal.DisplayListenerDelegate.handleMessage
    EVENT_DISPLAY_ADDED
    RootWindowContainer.onDisplayAdded
        RootWindowContainer.getDisplayContentOrCreate
            new DisplayContent()
                // 构建层级树，前面已经讲过
                DisplayContent.configureSurfaces
```


### 移栈

```
ActivityManagerShellCommand.runDisplay
    ActivityManagerShellCommand.runDisplayMoveStack
        ActivityTaskManagerService.moveRootTaskToDisplay
            RootWindowContainer.moveRootTaskToDisplay
                // 获取或者创建 DisplayContent
                RootWindowContainer.getDisplayContentOrCreate()
                // 获取 DefaultTaskDisplayArea
                DisplayContent.getDefaultTaskDisplayArea()
                RootWindowContainer.moveRootTaskToTaskDisplayArea
                    Task.reparent(TaskDisplayArea)
                        WindowContainer.reparent(TaskDisplayArea)
                            WindowContainer.onBeforeParentChanged()
                                ......
                                    SurfaceFreezer.freeze()
                                        new Snapshot()
                                            TaskDisplayArea.makeAnimationLeash().setName("snapshot anim: ").build()
                                                WindowContainer.makeAnimationLeash()
                            TaskDisplayArea.addChild()
                                TaskDisplayArea.addChildTask()
                                    ActivityTaskSupervisor.updateTopResumedActivityIfNeeded()
                                        ActivityTaskManagerService.setLastResumedActivityUncheckLocked()
                    Task.resumeNextFocusAfterReparent()
                        Task.adjustFocusToNextFocusableTask()
                            Task.moveToFront()
                                TaskDisplayArea.positionChildAt()
                                    TaskDisplayArea.positionChildTaskAt()
                                        TaskDisplayArea.positionChildAt()
                                            WindowContainer.positionChildAt()
                                                RootWindowContainer.onChildPositionChanged()
                                                    ActivityTaskSupervisor.updateTopResumedActivityIfNeeded()
                                                        ActivityTaskManagerService.setLastResumedActivityUncheckLocked()
                        RootWindowContainer.resumeFocusedTasksTopActivities()
                        RootWindowContainer.ensureActivitiesVisible()
                    
```

开始移栈后，会把原来的Task做一个截图，等 Transition 准备好后，开始做截图动画：    

```
RootWindowContainer.performSurfacePlacementNoTrace
    BLASTSyncEngine.onSurfacePlacement
        BLASTSyncEngine$SyncGroup.tryFinish
            BLASTSyncEngine$SyncGroup.finishNow
                Transition.onTransactionReady
                    consumeWindowAnimation()
                        consumeWindowModeChangeAnimation
                        consumePinWindowAnimationIfNeed
                        consumeDisplaySwitchAnimation
                            WindowContainer.startAnimation
                                SurfaceAnimator.startAnimation
                                    LocalAnimationAdapter.startAnimation
                                        SurfaceAnimationRunner.startAnimation
                                            mChoreographer.postFrameCallback(this::startAnimations)
                                                SurfaceAnimationRunner.startAnimations
                                                    SurfaceAnimationRunner.startPendingAnimationsLocked
                                                        SurfaceAnimationRunner.startAnimationLocked
                                                            ValueAnimator.addUpdateListener
                                                                SurfaceAnimationRunner.applyTransformation()
                                                                    WindowAnimationSpec.apply()
                                                                SurfaceAnimationRunner.scheduleApplyTransaction()
                                                            ValueAnimator.start()
                                            SurfaceAnimationRunner.applyTransformation()
                                                WindowAnimationSpec.apply()
                                     SurfaceFreezer$Snapshot.startAnimation
```

```
// RootWindowContainer.java
// 参数：rootTaskId：对应被移动的任务栈
// taskDisplayArea：新的承载 stack 的 Display
// onTop：判断stack在new Display的显示层级，默认是显示在最上层

    void moveRootTaskToTaskDisplayArea(int rootTaskId, TaskDisplayArea taskDisplayArea,
            boolean onTop) {
        // 根据 TaskId 获取对应的 Task
        final Task rootTask = getRootTask(rootTaskId);
        if (rootTask == null) {
            throw new IllegalArgumentException("moveRootTaskToTaskDisplayArea: Unknown rootTaskId="
                    + rootTaskId);
        }
        // 获取 Task 当前所在的 TaskDisplayArea
        final TaskDisplayArea currentTaskDisplayArea = rootTask.getDisplayArea();
        if (currentTaskDisplayArea == null) {
            throw new IllegalStateException("moveRootTaskToTaskDisplayArea: rootTask=" + rootTask
                    + " is not attached to any task display area.");
        }

        if (taskDisplayArea == null) {
            throw new IllegalArgumentException(
                    "moveRootTaskToTaskDisplayArea: Unknown taskDisplayArea=" + taskDisplayArea);
        }

        if (currentTaskDisplayArea == taskDisplayArea) {
            throw new IllegalArgumentException("Trying to move rootTask=" + rootTask
                    + " to its current taskDisplayArea=" + taskDisplayArea);
        }
        // 把 Task 挂载到新的 TaskDisplayArea
        rootTask.reparent(taskDisplayArea, onTop);

        // Resume focusable root task after reparenting to another display area.
        rootTask.resumeNextFocusAfterReparent();

        // TODO(multi-display): resize rootTasks properly if moved from split-screen.
    }
```

## 虚拟屏显示内容

我们看到，当虚拟屏里面没有内容显示的时候，它和手机主屏显示的内容是一样的，当把一个task移过去后，它就显示了这个task内容，这部分流程是怎么执行的呢？     
通过分析可以看到，当 mDisplayContent.getLastHasContent() 判断不成立时，就会开启虚拟屏对主屏的录制，此时虚拟屏显示的是主屏的镜像：    

```
RootWindowContainer.performSurfacePlacementNoTrace
    RootWindowContainer.applySurfaceChangesTransaction
        DisplayContent.applySurfaceChangesTransaction
            DisplayContent.updateRecording
                ContentRecorder.updateRecording
                    ContentRecorder.startRecordingIfNeeded
                        SurfaceControl.mirrorSurface
```

```
    @VisibleForTesting void updateRecording() {
        if (isCurrentlyRecording() && (mDisplayContent.getLastHasContent()
                || mDisplayContent.getDisplayInfo().state == Display.STATE_OFF)) {
            pauseRecording();
        } else {
            // Display no longer has content, or now has a surface to write to, so try to start
            // recording.
            startRecordingIfNeeded();
        }
    }
```


对比 SurfaceFlinger Layer 层级：     

<img src="/images/android-window-system-move-task/no-content.png" width="869" height="401" />


<img src="/images/android-window-system-move-task/has-content.png" width="872" height="415" />

如果我们想把录屏显示这里去掉，那么直接把 `startRecordingIfNeeded();` 这里注释掉就行了。      

## 改进


上面的移栈方案是使用命令行进行的，我们可以改进成使用手指拖动到另外一个屏幕，并加上响应的衔接动画来实现。      

```
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {

    ......
    
    // Multi-screen interaction test start
    //创建DoubleScreenMovePointerEventListener对象
    final DoubleScreenMovePointerEventListener mDoubleScreenMovePointerEventListener;
    // Multi-screen interaction test end
    
    ......
    
    DisplayContent(Display display, RootWindowContainer root) {
    
        ......
        //Multi-screen interaction test start
        //创建DoubleScreenMovePointerEventListener对象
        mDoubleScreenMovePointerEventListener = new DoubleScreenMovePointerEventListener(
                mWmService, this,mRootWindowContainer);
        //注册DoubleScreenMovePointerEventListener监听
        registerPointerEventListener(mDoubleScreenMovePointerEventListener);
        //Multi-screen interaction test end
        ......
    }

}
```

```
package com.android.server.wm;

import android.util.Slog;
import android.view.MotionEvent;
import android.view.WindowManagerPolicyConstants;

public class DoubleScreenMovePointerEventListener implements WindowManagerPolicyConstants.PointerEventListener {

    private final String TAG = "DoubleScreenMovePointerEventListener";
    //手指数
    private final int FINGERS = 2;
    //移动标志位
    boolean shouldBeginMove = false;

    //初始双击的两个横坐标点
    int point1XFirst = 0;
    int point2XFirst = 0;

    //拖动放手后的两个横坐标点
    int point1XLast = 0;
    int point2XLast = 0;

    //动作触发阈值，最少移动为10个像素才可以
    //移动大于150像素则往对侧移动
    //在10~150之间回到原来的屏幕
    final int START_GAP = 10;
    final int AUTO_MOVE_GAP = 150;

    private final WindowManagerService mService;
    DoubleScreenMoveController mDoubleScreenMoveController;

    public DoubleScreenMovePointerEventListener(WindowManagerService mService,
                                                DisplayContent mDisplayContent,
                                                RootWindowContainer mRootWindowContainer) {
        this.mService = mService;
        mDoubleScreenMoveController = new DoubleScreenMoveController(
        mService, mDisplayContent,mRootWindowContainer);
    }

    @Override
    public void onPointerEvent(MotionEvent motionEvent) {
        //屏幕数小于2，不调用onPointerEvent
        if (mService.mRoot.getChildCount() != 2) {
            Slog.i(TAG,"mRootWindowContainer.getChildCount() != 2");
            return;
        }
        switch (motionEvent.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
            case MotionEvent.ACTION_POINTER_DOWN:
                //手指数不为FINGERS，则shouldBeginMove标志位置为false
                if (motionEvent.getPointerCount() != FINGERS) {
                    shouldBeginMove = false;
                }

                if (motionEvent.getPointerCount() == FINGERS &&
                        point1XFirst == 0 && point2XFirst == 0) {
                    //获取两个手指的初始坐标
                    point1XFirst = (int) motionEvent.getX(0);
                    point2XFirst = (int) motionEvent.getX(1);
                    Slog.i(TAG, "onPointerEvent ACTION_POINTER_DOWN point1XFirst = " +
                            point1XFirst + " point2XFirst = " + point2XFirst);
                }
                break;
            case MotionEvent.ACTION_MOVE:
                if (motionEvent.getPointerCount() == FINGERS) {
                    //获取两个手指在移动后的坐标
                    point1XLast = (int) motionEvent.getX(0);
                    point2XLast = (int) motionEvent.getX(1);
                    //计算移动后的差值
                    int offsetX1 = point1XLast - point1XFirst;
                    int offsetX2 = point2XLast - point2XFirst;
                    Slog.i(TAG, "onPointerEvent ACTION_MOVE " +
                            "point1XFirst = " + point1XFirst + " point1XLast = " + point1XLast);
                    if (!shouldBeginMove &&
                            //取两手指移动距离的绝对值，其中一个手指移动距离大于10即可
                            ((Math.abs(offsetX1) > START_GAP) || (Math.abs(offsetX2) > START_GAP)) &&
                            //判断两手指移动的方向是否相等，保证两手指移动的方向一致
                            (getMovingOrientation(offsetX1, 0).equals(
                                    getMovingOrientation(offsetX2, 0)))) {
                        Slog.i(TAG, "onPointerEvent ACTION_MOVE start moving");
                        shouldBeginMove = true;
                        //调用doTestMoveTaskToOtherDisplay移动Task到另一屏
                        mDoubleScreenMoveController.doTestMoveTaskToOtherDisplay();
                    }
                    Slog.i(TAG, "onPointerEvent ACTION_MOVE shouldBeginMove=" + shouldBeginMove);
                    if (shouldBeginMove) {
                        int offsetX = Math.abs(point1XLast - point1XFirst);
                        //调用startMoveCurrentScreenTask，使手拖动时移动过程平滑显示
                        mDoubleScreenMoveController.startMoveCurrentScreenTask(offsetX, 0);
                    }
                } else {
                    Slog.i(TAG, "onPointerEvent ACTION_MOVE forbid moving!");
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_POINTER_UP:
            case MotionEvent.ACTION_CANCEL:
                if (shouldBeginMove) {
                    int offsetX = Math.abs(point1XLast - point1XFirst);
                    Slog.i(TAG, "onPointerEvent ACTION_POINTER_UP point1XLast = " + point1XLast
                            + " point1XFirst = " + point1XFirst + "offsetX = " + offsetX);
                    //调用startAutoMove，在放手后播放Task自动移动的动画
                    mDoubleScreenMoveController.startAutoMove(offsetX, offsetX > AUTO_MOVE_GAP);
                }
                shouldBeginMove = false;
                point1XFirst = 0;
                point2XFirst = 0;
                break;
        }
    }

    /**
     * 根据距离差判断 滑动方向
     *
     * @param dx X轴的距离差
     * @param dy Y轴的距离差
     * @return 滑动的方向
     */
    private String getMovingOrientation(float dx, float dy) {
        String orientation = null;
        Slog.i(TAG, "========X轴距离差：" + dx);
        Slog.i(TAG, "========Y轴距离差：" + dy);
        if (Math.abs(dx) >= Math.abs(dy)) {
            //X轴移动
            if (dx > 0) {
                orientation = "right";
            } else {
                orientation = "left";
            }
        } else {
            //Y轴移动
            if (dy > 0) {
                orientation = "up";
            } else {
                orientation = "down";
            }
        }
        return orientation;
    }

}
```

```
package com.android.server.wm;

import android.animation.Animator;
import android.animation.AnimatorListenerAdapter;
import android.animation.ValueAnimator;
import android.annotation.NonNull;
import android.graphics.Matrix;
import android.os.Build;
import android.os.Handler;
import android.os.Message;
import android.util.Slog;
import android.view.SurfaceControl;
import android.view.animation.AccelerateInterpolator;

public class DoubleScreenMoveController {

    /**
     * Multi-screen interaction test start
     */
    private final static String TAG = "DoubleScreenMoveController";
    //移动时根节点图层，镜像图层的父图层
    private SurfaceControl copyTaskSc = null;
    //镜像图层
    private SurfaceControl copyTaskBuffer = null;
    //应用的图层
    private SurfaceControl realWindowStateBuffer = null;
    //保存当前TaskId
    private int mCurrentTaskId = -1;
    //保存另一屏
    private DisplayContent mOtherDisplay = null;
    //保存当前Activity
    private ActivityRecord mCurrentRecord = null;

    private final DisplayContent mDisplayContent;
    private final WindowManagerService mWmService;
    private final RootWindowContainer mRootWindowContainer;

    public DoubleScreenMoveController(WindowManagerService mWmService,
                                      DisplayContent mDisplayContent,
                                      RootWindowContainer mRootWindowContainer) {
        this.mWmService = mWmService;
        this.mDisplayContent = mDisplayContent;
        this.mRootWindowContainer = mRootWindowContainer;
    }


    public void doTestMoveTaskToOtherDisplay() {
        Slog.i(TAG, "doTestMoveTaskToOtherDisplay " +
                "mRootWindowContainer.getChildCount() = " + mRootWindowContainer.getChildCount());
        if (mRootWindowContainer.getChildCount() == 2) {
            //获取另一个屏幕，mRootWindowContainer.getChildAt(0)获取需要移动应用的屏幕
            //判断是否和当前屏幕相等，mRootWindowContainer.getChildAt(1)表示另一屏
            mOtherDisplay = mRootWindowContainer.getChildAt(0) == mDisplayContent ?
                    mRootWindowContainer.getChildAt(1) :
                    mRootWindowContainer.getChildAt(0);
        } else {
            Slog.i(TAG, "mRootWindowContainer.getChildCount() is not 2!");
        }
        //判断另一屏和当前屏幕是否是同一个屏幕
        if (mOtherDisplay != mDisplayContent && mOtherDisplay != null) {
            try {
                //获取当前屏幕上最顶层的应用Task
                Task rootTask = mDisplayContent.getTopRootTask();
                //判断Task是否是桌面
                if (rootTask.isActivityTypeHome()) {
                    return;
                }
                //获取TaskId
                int rootTaskId = rootTask.getRootTaskId();
                mCurrentTaskId = rootTaskId;
                if (copyTaskSc == null) {
                    //创建图层，并使其父节点为WindowedMagnification
                    //这个图层为移动时根节点图层，镜像图层的父图层
                    copyTaskSc = mDisplayContent.makeChildSurface(null)
                            .setName("copyTaskSc")
                            .setParent(mDisplayContent.getWindowingLayer())
                            .build();
                    Slog.i(TAG, "doTestMoveTaskToOtherDisplay mDisplayContent.getWindowingLayer() = "
                            + mDisplayContent.getWindowingLayer());
                }

                if (copyTaskBuffer == null) {
                    //创建当前应用Task的镜像图层
                    copyTaskBuffer = SurfaceControl.mirrorSurface(rootTask.getSurfaceControl());
                }
                //获取当前应用的图层
                realWindowStateBuffer = rootTask.getSurfaceControl();
                SurfaceControl.Transaction transaction = mWmService.mTransactionFactory.get();
                //把镜像图层copyTaskBuffer挂载到copyTaskSc图层节点上
                transaction.reparent(copyTaskBuffer, copyTaskSc);
                transaction.show(copyTaskSc);
                transaction.show(copyTaskBuffer);
                transaction.apply();

                Slog.i(TAG, "doTestMoveTaskToOtherDisplay " +
                        "moveRootTaskToDisplay  rootTask = " + rootTask + " rootTaskId = "
                        + rootTaskId + " otherDisplay.mDisplayId = " + mOtherDisplay.mDisplayId);
                //保证底部activity显示
                ensureOtherDisplayActivityVisible(mOtherDisplay);
                //防止另一屏闪烁Task
                //移动前需要调用一下startMoveCurrentScreenTask，目的是为了把另一屏的Task坐标移动到屏幕外，
                //不然可能会产生开始拖拉时候，另一屏会有显示一瞬间闪烁的task画面，这里提前把偏移设置好
                startMoveCurrentScreenTask(0, 0);
                //移动Task至另一屏
                mRootWindowContainer.moveRootTaskToDisplay(
                        mCurrentTaskId, mOtherDisplay.mDisplayId, true);
            } catch (Exception e) {
                Slog.i(TAG, "doTestMoveTaskToOtherDisplay Exception", e);
            }
        } else {
            Slog.i(TAG, "otherDisplay is this or otherDisplay is null!");
        }
    }


    /**
     * 在双屏拖拽时可能会出现另一屏桌面或者其他顶层Activity界面黑屏的现象，因此需要通过该配置使其保持显示。
     * @param otherDisplay 另一屏
     */
    void ensureOtherDisplayActivityVisible(DisplayContent otherDisplay) {
        //获取最顶部的Activity
        ActivityRecord otherTopActivity = otherDisplay.getTopActivity(false, false);
        Slog.i(TAG, "ensureOtherDisplayActivityVisible otherDisplay = " +
                otherDisplay + "otherTopActivity = " + otherTopActivity);
        if (otherTopActivity != null) {
            //mLaunchTaskBehind = true,表示当前允许activity显示在最下方
            otherTopActivity.mLaunchTaskBehind = true;
            //保存最顶部的Activity
            mCurrentRecord = otherTopActivity;
        }
    }

    /**
     * 重置mLaunchTaskBehind，更新Activity显示状态
     */
    void resetState() {
        if (mCurrentRecord != null) {
            //恢复正常状态，让mLaunchTaskBehind变成false
            mCurrentRecord.mLaunchTaskBehind = false;
            //更新Activity显示状态
            mRootWindowContainer.ensureActivitiesVisible(null, 0, false);
        }
    }

    /**
     * 移动Task图层
     * @param offsetX 左右偏移量
     * @param offsetY 上下偏移量（不涉及）
     */
    public void startMoveCurrentScreenTask(int offsetX, int offsetY) {
        Slog.i(TAG, "startMoveCurrentScreenTask getDisplayId() = " + mDisplayContent.mDisplayId);
        if (realWindowStateBuffer != null && realWindowStateBuffer != null) {
            SurfaceControl.Transaction transaction = mWmService.mTransactionFactory.get();
            Matrix matrix = new Matrix();
            //获取屏幕宽度
            int width = mDisplayContent.getDisplayInfo().logicalWidth;
            //第一屏的DisplayId一般的都是0
            if (mDisplayContent.mDisplayId == 0) {
                Slog.i(TAG, "isMovingToRight!");
                //Display1，移动镜像图层
                matrix.reset();
                //根据计算的偏移量进行移动
                matrix.postTranslate(offsetX + (width - offsetX), offsetY);
                transaction.setMatrix(copyTaskBuffer, matrix, new float[9]);
                //Display2，移动真实图层
                matrix.reset();
                //-(width-offsetX)，根据计算的偏移量进行移动
                matrix.postTranslate(offsetX - width, offsetY);
                transaction.setMatrix(realWindowStateBuffer, matrix, new float[9]);
            } else {
                Slog.i(TAG, "isMovingToLeft!");
                //Display2，移动镜像图层
                matrix.reset();
                //-(offsetX + (width - offsetX))，根据计算的偏移量进行移动
                matrix.postTranslate(-offsetX - width + offsetX, offsetY);
                transaction.setMatrix(copyTaskBuffer, matrix, new float[9]);
                //Display1，移动真实图层
                matrix.reset();
                //根据计算的偏移量进行移动
                matrix.postTranslate(width - offsetX, offsetY);
                transaction.setMatrix(realWindowStateBuffer, matrix, new float[9]);
            }
            transaction.apply();
        }
    }

    /**
     * 在放手后播放Task自动移动的动画
     * @param offsetX 移动时的偏移量
     * @param toOther 是否移动到另一屏,
     *                移动大于AUTO_MOVE_GAP像素则往对侧移动,
     *                在START_GAP和AUTO_MOVE_GAP之间回到原来的屏幕
     */
    public void startAutoMove(int offsetX, boolean toOther) {
        //获取屏幕宽度
        int width = mDisplayContent.getDisplayInfo().logicalWidth;
        //如果是移动到另一屏则会移动距离区间为(offsetX，width)
        //否则移动区间为(offsetX，0)
        int endX = toOther ? width : 0;
        Slog.i(TAG, "startAutoMove onAnimationUpdate offsetX = " + offsetX
                + " endX = " + endX);
        //创建一个在(offsetX, endX)区间内的ValueAnimator
        ValueAnimator valueAnimator = ValueAnimator.ofInt(offsetX, endX);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {

            valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                //动画更新时的监听
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    int currentX = (int) animation.getAnimatedValue();
                    Slog.i(TAG, "startAutoMove onAnimationUpdate currentX = " + currentX);
                    //在动画更新过程中调用移动方法，使图层在区间(offsetX, endX)内移动
                    startMoveCurrentScreenTask(currentX, 0);
                }
            });

            valueAnimator.addListener(new AnimatorListenerAdapter() {
                //动画结束时的监听
                @Override
                public void onAnimationEnd(Animator animation) {
                    super.onAnimationEnd(animation);
                    SurfaceControl.Transaction transaction = mWmService.mTransactionFactory.get();
                    //判断是否移动到另一屏幕
                    if (toOther) {
                        //如果移动另一屏幕，重置mLaunchTaskBehind，更新Activity显示状态
                        resetState();
                    } else {
                        Slog.i(TAG, "startAutoMove move to mDisplayContent.mDisplayId = " +
                                mDisplayContent.mDisplayId);
                        //如果不是移动到另一屏幕，使Task回到当前屏幕
                        mRootWindowContainer.moveRootTaskToDisplay(
                                mCurrentTaskId, mDisplayContent.mDisplayId, true);
                        mCurrentTaskId = -1;
                        //重置realWindowStateBuffer
                        if (realWindowStateBuffer != null) {
                            Matrix matrix = new Matrix();
                            matrix.reset();
                            transaction.setMatrix(realWindowStateBuffer, matrix, new float[9]);
                        }
                        transaction.apply();
                    }

                    //移除镜像图层以及其根节点
                    if (copyTaskBuffer != null && copyTaskSc != null) {
                        Slog.i(TAG, "startAutoMove copyTaskBuffer and copyTaskSc is remove!");
                        transaction.remove(copyTaskBuffer);
                        transaction.remove(copyTaskSc);
                        transaction.apply();
                        copyTaskBuffer = null;
                        copyTaskSc = null;
                    } else {
                        Slog.i(TAG, "copyTaskBuffer or copyTaskSc is null!");
                    }
                }
            });
            //设置动画参数并播放
            valueAnimator.setInterpolator(new AccelerateInterpolator(1.0f));
            valueAnimator.setDuration(500);
            valueAnimator.start();
        } else {
            Slog.i(TAG, "Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB");
        }
    }

    /**
     * Multi-screen interaction test end
     */
}
```

参考下面的博客和开源代码：
[Android T多屏多显——应用双屏间拖拽移动功能](https://www.jindouyun.cn/document/industry/details/242008)     
[AndroidT应用双屏间拖拽移动功能实现](https://blog.csdn.net/gitblog_09739/article/details/142943823)     
[gitcode](https://gitcode.com/open-source-toolkit/ca59c/?utm_source=tools_gitcode&index=top&type=card&&isLogin=1)      
[车载多屏互动联动动画版本同屏幕大小情况方案设计--众筹项目](https://blog.csdn.net/learnframework/article/details/130507022)     
[车载多屏互动联动动画版本图层设计--众筹项目](https://blog.csdn.net/learnframework/article/details/130522955)     

