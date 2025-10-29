---
title: Android WindowContainerTransaction
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 WindowContainerTransaction
date: 2022-11-23 10:00:00
---

## 简介

用于存储对 WindowContainer 修改的集合的类。由于应用层无法直接操作 WindowContainer，需要通过系统层进行修改，因此 WindowContainerTransaction 类实现了 Parcelable 接口，支持跨进程传输。     
这种机制确保了应用层可以通过系统层安全地修改 WindowContainer。    
这些变更主要包含 Change 相关和 HierarchyOp。    
比如，切换分屏模式时，在 SystemUI 侧对 Task 的层级、位置和宽高边界做修改，那么就应用了 WindowContainerTransaction。这些属性的变更需要通过 `WindowOrganizer.startTransition(WindowContainerTransaction t)` ，`WindowOrganizer.finishTransition(WindowContainerTransaction t)` 或者 `WindowOrganizer.applyTransaction(WindowContainerTransaction t)` 传入到 system_server 侧来对对应的窗口进行应用。     

## Change

Change 包含了 WindowContainer 的变更，包括 Configuration 的变化。    

```
    public static class Change implements Parcelable {
        public static final int CHANGE_FOCUSABLE = 1;
        public static final int CHANGE_BOUNDS_TRANSACTION = 1 << 1;
        public static final int CHANGE_PIP_CALLBACK = 1 << 2;
        public static final int CHANGE_HIDDEN = 1 << 3;
        public static final int CHANGE_BOUNDS_TRANSACTION_RECT = 1 << 4;
        public static final int CHANGE_IGNORE_ORIENTATION_REQUEST = 1 << 5;
        public static final int CHANGE_FORCE_NO_PIP = 1 << 6;
        public static final int CHANGE_FORCE_TRANSLUCENT = 1 << 7;
        public static final int CHANGE_DRAG_RESIZING = 1 << 8;
        public static final int CHANGE_RELATIVE_BOUNDS = 1 << 9;

        private final Configuration mConfiguration = new Configuration();
        private boolean mFocusable = true;
        private boolean mHidden = false;
        private boolean mIgnoreOrientationRequest = false;
        private boolean mForceTranslucent = false;
        private boolean mDragResizing = false;

        // 针对 Change 中定义的属性的修改标记
        private int mChangeMask = 0;  
        // 针对 Configuration 修改的标记
        private @ActivityInfo.Config int mConfigSetMask = 0;  
        // 针对 WindowConfiguration 属性修改的标记
        private @WindowConfiguration.WindowConfig int mWindowSetMask = 0;  

        private Rect mPinnedBounds = null;
        private SurfaceControl.Transaction mBoundsChangeTransaction = null;
        private Rect mBoundsChangeSurfaceBounds = null;
        @Nullable
        private Rect mRelativeBounds = null;
        private boolean mConfigAtTransitionEnd = false;

        private int mActivityWindowingMode = -1;
        private int mWindowingMode = -1;

```

这里面的一些成员变量都是可以修改属性，其中最为常用的就是Configuration。     
但是，WindowContainerTransaction 并不支持对 Configuration 中所有的属性进行修改，主要是screenSize、windowingMode和bounds等。    
mChangeMask、mConfigSetMask 和 mWindowSetMask 均为标记，后续有不同类型的修改都会通过该变量进行标记。    

用 getOrCreateChange 方法去获取 mChanges 的 Change 对象或者去创建一个新的，很多方法调用了这个方法。        
调用该方法时，参数 IBinder 对象 token 传递的是 `WindowContainerToken` 对象的` asBinder()`。    
也就是说，这个方法就是通过传递过来的不同的容器所对应的WindowContainerToken，然后找到其对应着的Change；如果这个Change为空则创建一个Change，并将其放到mChanges集合中，使WindowContainerToken和Change一一对应。    

```
    private Change getOrCreateChange(IBinder token) {
        Change out = mChanges.get(token);
        if (out == null) {
            out = new Change();
            mChanges.put(token, out);
        }
        return out;
    }

    public WindowContainerTransaction setHidden(
            @NonNull WindowContainerToken container, boolean hidden) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mHidden = hidden;
        chg.mChangeMask |= Change.CHANGE_HIDDEN;
        return this;
    }
```

<img src="/images/android-window-system-window-container-transaction/0.png" width="670" height="436"/>

WindowContainerTransaction 提供了很多针对 Change 的操作的方法：    

 - scheduleFinishEnterPip()
 - setBoundsChangeTransaction
 - setActivityWindowingMode
 - setWindowingMode
 - setFocusable
 - setHidden
 - setForceTranslucent
 - ......
 - setDragResizing

这些方法设置了 Change 中对应的 CHANGE_FOCUSABLE 等这些属性，并且在 mChangeMask 中做了标记。     

另外还有设置容器Configuration相关属性的方法：

```
    public WindowContainerTransaction setBounds(
            @NonNull WindowContainerToken container,@NonNull Rect bounds) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.windowConfiguration.setBounds(bounds);
        chg.mConfigSetMask |= ActivityInfo.CONFIG_WINDOW_CONFIGURATION;
        chg.mWindowSetMask |= WindowConfiguration.WINDOW_CONFIG_BOUNDS;
        return this;
    }

    @NonNull
    public WindowContainerTransaction setAppBounds(
            @NonNull WindowContainerToken container,@NonNull Rect appBounds) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.windowConfiguration.setAppBounds(appBounds);
        chg.mConfigSetMask |= ActivityInfo.CONFIG_WINDOW_CONFIGURATION;
        chg.mWindowSetMask |= WindowConfiguration.WINDOW_CONFIG_APP_BOUNDS;
        return this;
    }

    public WindowContainerTransaction setScreenSizeDp(
            @NonNull WindowContainerToken container, int w, int h) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.screenWidthDp = w;
        chg.mConfiguration.screenHeightDp = h;
        chg.mConfigSetMask |= ActivityInfo.CONFIG_SCREEN_SIZE;
        return this;
    }
    
    public WindowContainerTransaction setDensityDpi(@NonNull WindowContainerToken container,
            int densityDpi) {
        Change chg = getOrCreateChange(container.asBinder());
        chg.mConfiguration.densityDpi = densityDpi;
        chg.mConfigSetMask |= ActivityInfo.CONFIG_DENSITY;
        return this;
    }
    
```

这些方法都是通过传递的WindowContainerToken，获取对应的Change，然后再通过Configuration对象修改对应的属性值，保存在Change中，并且在mConfigSetMask和mWindowSetMask中保存了修改标记。     

## HierarchyOp

HierarchyOp 相关的修改，主要是修改层级结构的，通过WindowOrganizerController.applyHierarchyOp方法进行实现。

```
    public static final class HierarchyOp implements Parcelable {
        public static final int HIERARCHY_OP_TYPE_REPARENT = 0;
        public static final int HIERARCHY_OP_TYPE_REORDER = 1;
        public static final int HIERARCHY_OP_TYPE_CHILDREN_TASKS_REPARENT = 2;
        public static final int HIERARCHY_OP_TYPE_SET_LAUNCH_ROOT = 3;
        public static final int HIERARCHY_OP_TYPE_SET_ADJACENT_ROOTS = 4;
        public static final int HIERARCHY_OP_TYPE_LAUNCH_TASK = 5;
        public static final int HIERARCHY_OP_TYPE_SET_LAUNCH_ADJACENT_FLAG_ROOT = 6;
        public static final int HIERARCHY_OP_TYPE_PENDING_INTENT = 7;
        public static final int HIERARCHY_OP_TYPE_START_SHORTCUT = 8;
        public static final int HIERARCHY_OP_TYPE_RESTORE_TRANSIENT_ORDER = 9;
        public static final int HIERARCHY_OP_TYPE_ADD_INSETS_FRAME_PROVIDER = 10;
        public static final int HIERARCHY_OP_TYPE_REMOVE_INSETS_FRAME_PROVIDER = 11;
        public static final int HIERARCHY_OP_TYPE_SET_ALWAYS_ON_TOP = 12;
        public static final int HIERARCHY_OP_TYPE_REMOVE_TASK = 13;
        public static final int HIERARCHY_OP_TYPE_FINISH_ACTIVITY = 14;
        public static final int HIERARCHY_OP_TYPE_CLEAR_ADJACENT_ROOTS = 15;
        public static final int HIERARCHY_OP_TYPE_SET_REPARENT_LEAF_TASK_IF_RELAUNCH = 16;
        public static final int HIERARCHY_OP_TYPE_ADD_TASK_FRAGMENT_OPERATION = 17;
        public static final int HIERARCHY_OP_TYPE_MOVE_PIP_ACTIVITY_TO_PINNED_TASK = 18;
        private final int mType;

        // 表示当前操作容器的WindowContainerToken。
        private IBinder mContainer;

        // 表示父容器的WindowContainerToken。
        private IBinder mReparent;
        
        // 表示reparent/reorder操作后是否需要将子容器移动到父容器之上。true：表示移动到父容器上方，false表示移动到父容器下方。
        private boolean mToTop;
        
        // 启动操作项，如分屏时会调用addActivityOptions设置操作项。
        private Bundle mLaunchOptions;
        
        public static HierarchyOp createForReparent()
        
        public static HierarchyOp createForReorder()
        
        public static HierarchyOp createForChildrenTasksReparent()
        
        ......
```

举例：    

```
    public WindowContainerTransaction reparent(@NonNull WindowContainerToken child,
            @Nullable WindowContainerToken parent, boolean onTop) {
        mHierarchyOps.add(HierarchyOp.createForReparent(child.asBinder(),
                parent == null ? null : parent.asBinder(),
                onTop));
        return this;
    }

        public static HierarchyOp createForReparent(
                @NonNull IBinder container, @Nullable IBinder reparent, boolean toTop) {
            return new HierarchyOp.Builder(HIERARCHY_OP_TYPE_REPARENT)
                    .setContainer(container)
                    .setReparentContainer(reparent)
                    .setToTop(toTop)
                    .build();
        }
```

WindowContainerTransaction 提供了很多针对 HierarchyOp 操作的方法：

 - reparent：用来调整容器父子层级关系
 - reorder：Task重排序，对当前容器以及其父容器的顺序进行调整，这里传入的就是当前需要调整顺序的容器。     
 - reparentTasks：Tasks父级变更，把currentParent中的所有子容器，全部reparent到newParent下，即交换父亲。    
 - setLaunchRoot
 - setAdjacentRoots
 - setLaunchAdjacentFlagRoot
 - ......
 - startTask：通过taskId来启动一个task，这个task是已经启动过的，比如最近任务中的task。这个里面传递的options，存放了一些启动操作项。例如，分屏操作中调用的addActivityOptions(options1, mSideStage);,就是在options1中存放了关联SideStage的WindowContainerToken与ActivityOptions的KEY_LAUNCH_ROOT_TASK_TOKEN的映射关系。     
 - removeTask：移除task

## WindowContainerTransaction 的发送

应用层可以通过下面几种方式来想系统发送 WindowContainerTransaction 。    

### WindowOrganizer.startTransition 和 WindowOrganizer.finishTransition

ShellTransitions 动画的开始和结束时可以传递 WindowContainerTransaction 参数，如果参数不为空，WMCore 就会 apply 这些变更。     

```
public class WindowOrganizer {
    // 
    public void startTransition(@NonNull IBinder transitionToken,
            @Nullable WindowContainerTransaction t) {
        ....
    }
    public void finishTransition(@NonNull IBinder transitionToken, @Nullable WindowContainerTransaction t) {
        ....
    }
}
```

### WindowOrganizer.applyTransaction发送

通过 applyTransaction 发送异步 WindowContainerTransaction。     
通过 startTransition 或者 startNewTransition 发起 ShellTransition 动画，参考 Transitions.startTransition()。      

```
public class WindowOrganizer {

    // 分屏场景
    @RequiresPermission(value = android.Manifest.permission.MANAGE_ACTIVITY_TASKS)
    public void applyTransaction(@NonNull WindowContainerTransaction t) {
        ...
    }

    // 分屏，PIP等场景，通过 SyncTransactionQueue 发送
    public int applySyncTransaction(@NonNull WindowContainerTransaction t,
            @NonNull WindowContainerTransactionCallback callback) {
        ...
    }
    
    // 多任务场景，PIP 退出动画场景
    public IBinder startNewTransition(int type, @Nullable WindowContainerTransaction t) {
        ....
    }
    
    // 分屏 场景
    public void startTransition(@NonNull IBinder transitionToken,
            @Nullable WindowContainerTransaction t) {
        ....
    }
}
```

### SyncTransactionQueue 发送

发送同步 WindowContainerTransaction，最终也是通过 WindowOrganizer.applySyncTransaction 执行。    
// 分屏场景

```
SyncTransactionQueue.queue(WindowContainerTransaction wct)
SyncTransactionQueue.runInSync(TransactionRunnable runnable)
```

## 相关文章

[WindowContainerTransaction概念详解 ](https://juejin.cn/post/7301955975338131495)    
[Android U system_server侧WindowContainerTransaction处理](https://blog.csdn.net/yimelancholy/article/details/144694952)    
