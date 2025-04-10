---
title: Android 壁纸管理服务
categories: Android 框架
comments: true
tags: [Android 框架]
description: 介绍 WallpaperManagerService 及其相关类
date: 2022-11-23 10:00:00
---


## 概述

adb shell dumpsys wallpaper

## 动态壁纸实现方法

参考前面博客：[Android 动态壁纸](https://www.heqiangfly.com/qbook/source/Android%20Demo/android-demo-live-wallpaper-snow.html)

## WallpaperManager

```
    // 系统壁纸
    public static final int FLAG_SYSTEM = 1 << 0;

    // 锁屏壁纸
    public static final int FLAG_LOCK = 1 << 1;
```

## WallpaperWindowToken

mShowWhenLocked：是否可以在锁屏时显示，也就是锁屏壁纸

通过 `void setShowWhenLocked(boolean showWhenLocked)` 来设置。    

WallpaperManagerService.updateEngineFlags 方法中设置：

```
    private void updateEngineFlags(WallpaperData wallpaper) {
        if (wallpaper.connection == null) {
            return;
        }
        wallpaper.connection.forEachDisplayConnector(
                connector -> {
                    try {
                        if (connector.mEngine != null) {
                            connector.mEngine.setWallpaperFlags(wallpaper.mWhich);
                            mWindowManagerInternal.setWallpaperShowWhenLocked(
                                    connector.mToken, (wallpaper.mWhich & FLAG_LOCK) != 0);
                        }
                    } catch (RemoteException e) {
                        Slog.e(TAG, "Failed to update wallpaper engine flags", e);
                    }
                });
    }
```

## WallpaperController

它负责协调壁纸窗口的显示、生命周期以及与 Activity 窗口的交互逻辑。    
它作为 DisplayContent 的成员变量存在。    

```
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
    .....
    WallpaperController mWallpaperController;
    ......
    }
```

### mWallpaperTarget

如果为非 null，则这是与壁纸关联的当前可见窗口。      
解释一下壁纸的目标窗口，比如一个 Activity 的窗口在显示的时候被设置为支持壁纸显示，比如 Launcher 的窗口，那么这个窗口就可以作为壁纸的目标窗口。如果我们遍历所有的窗口后，找不到一个窗口可以作为壁纸的目标窗口，那么就说明所有的窗口都不支持壁纸显示，那壁纸也就会被设置为不可见。如果可以找到一个壁纸的目标窗口，那么这个目标窗口就会被保存到成员变量 WallpaperController.mWallpaperTarget中。    

### mFindResults

FindWallpaperTargetResult 的实例，见下面解释：    

### FindWallpaperTargetResult

它定义在WallpaperController里，用来保存寻找壁纸目标窗口操作的结果。   

mTopWallpaper：

#### TopWallpaper

mTopHideWhenLockedWallpaper和mTopShowWhenLockedWallpaper都可以设置为壁纸窗口，且同一时间它们两个中间只能有一个被设置：     
1）、如果 mTopHideWhenLockedWallpaper 被设置（即不为空），说明此时壁纸只能在Home界面可见，锁屏界面不可见。     
2）、如果 mTopShowWhenLockedWallpaper 被设置（即不为空），说明此时壁纸在Home界面以及锁屏界面均可见。     
mNeedsShowWhenLockedWallpaper 如果在一个Activity界面可以在锁屏界面上显示，比如通话界面，如果这个Activity或者窗口不是全屏的，那么就会把mNeedsShowWhenLockedWallpaper 的值设置为 true。在这种场景下，即使我们没有为锁屏壁纸找到一个目标窗口，那么我们可能也是需要将锁屏壁纸显示出来的。      
useTopWallpaperAsTarget 如果经过我们遍历所有窗口后，我们找不到任何一个窗口可以作为壁纸的目标窗口，但是在一些特殊场景下，我们又是需要将壁纸显示出来的，那么我们就将这个值设置为true，表示我们将TopWallpaper中的mTopHideWhenLockedWallpaper或者mTopShowWhenLockedWallpaper保存的壁纸WindowState本身作为壁纸的目标窗口，至于从这两者中的哪个里面取，则是看当前是否处于锁屏。     

wallpaperTarget 在寻找壁纸的目标窗口阶段，我们将寻找的结果保存在FindWallpaperTargetResult.wallpaperTarget中，后续在WallpaperController.updateWallpaperWindowsTarget中，我们就从FindWallpaperTargetResult.wallpaperTarget中拿寻找的结果。    

### adjustWallpaperWindows()

作用：根据当前窗口状态调整壁纸窗口的可见性和布局。是壁纸更新的起点。       
逻辑：遍历所有窗口，判断是否有全屏窗口覆盖壁纸，动态调整壁纸的 Visibility 和 Z-Order。    

```
    void adjustWallpaperWindows() {
        mDisplayContent.mWallpaperMayChange = false;
        // 用来寻找壁纸的目标窗口，将寻找的结果放到成员变量WallpaperController.mFindResults中。
        findWallpaperTarget();
        // 根据成员变量WallpaperController.mFindResults更新成员变量WallpaperController.mWallpaperTarget。
        updateWallpaperWindowsTarget(mFindResults);
        WallpaperWindowToken token = getTokenForTarget(mWallpaperTarget);

        // The window is visible to the compositor...but is it visible to the user?
        // That is what the wallpaper cares about.
        final boolean visible = token != null;

        if (visible) {
            if (mWallpaperTarget.mWallpaperX >= 0) {
                token.mWallpaperX = mWallpaperTarget.mWallpaperX;
                token.mWallpaperXStep = mWallpaperTarget.mWallpaperXStep;
            }
            if (mWallpaperTarget.mWallpaperY >= 0) {
                token.mWallpaperY = mWallpaperTarget.mWallpaperY;
                token.mWallpaperYStep = mWallpaperTarget.mWallpaperYStep;
            }
            if (mWallpaperTarget.mWallpaperDisplayOffsetX != Integer.MIN_VALUE) {
                token.mWallpaperDisplayOffsetX = mWallpaperTarget.mWallpaperDisplayOffsetX;
            }
            if (mWallpaperTarget.mWallpaperDisplayOffsetY != Integer.MIN_VALUE) {
                token.mWallpaperDisplayOffsetY = mWallpaperTarget.mWallpaperDisplayOffsetY;
            }
        }
        // 更新壁纸的可见性。
        updateWallpaperTokens(visible, mDisplayContent.isKeyguardLocked());

        .....

        if (visible && mLastFrozen != mFindResults.isWallpaperTargetForLetterbox) {
            mLastFrozen = mFindResults.isWallpaperTargetForLetterbox;
            sendWindowWallpaperCommand(
                    mFindResults.isWallpaperTargetForLetterbox ? COMMAND_FREEZE : COMMAND_UNFREEZE,
                    /* x= */ 0, /* y= */ 0, /* z= */ 0, /* extras= */ null, /* sync= */ false);
        }

        ......
    }
```

### findWallpaperTarget()

用来寻找壁纸的目标窗口，为 FindWallpaperTargetResult.TopWallpaper.wallpaperTarget 赋值。    

```
    private void findWallpaperTarget() {
        //重置FindWallpaperTargetResult的保存的所有信息
        mFindResults.reset();
        if (mService.mAtmService.mSupportsFreeformWindowManagement
                && mDisplayContent.getDefaultTaskDisplayArea()
                .isRootTaskVisible(WINDOWING_MODE_FREEFORM)) {
            // 如果当前有WINDOWING_MODE_FREEFORM类型的App显示，
            // 那么就设置FindWallpaperTargetResult.useTopWallpaperAsTarget 为 true
            mFindResults.setUseTopWallpaperAsTarget(true);
        }
        //第一次遍历窗口，寻找可以在锁屏或解锁显示的壁纸，
        // 为 mTopHideWhenLockedWallpaper 和 mTopShowWhenLockedWallpaper 赋值。    
        findWallpapers();
        // 通过 mFindWallpaperTargetFunction 回调对所有的窗口进行轮询。
        // 后面详细介绍
        mDisplayContent.forAllWindows(mFindWallpaperTargetFunction, true /* traverseTopToBottom */);
        if (mFindResults.mNeedsShowWhenLockedWallpaper) {
            // 如果在 mFindWallpaperTargetFunction 中为 mNeedsShowWhenLockedWallpaper 赋值
            mFindResults.setUseTopWallpaperAsTarget(true);
        }
        // 如果FindWallpaperTargetResult.wallpaperTarget为空，说明通过两次遍历我们没有找到壁纸的目标窗口，
        // 但是FindWallpaperTargetResult.useTopWallpaperAsTarget为true，又说明我们的确想显示壁纸，
        // 那么就调用FindWallpaperTargetResult.getTopWallpaper看看能不能返回一个壁纸窗口，
        // 如果可以，那么就调用FindWallpaperTargetResult.setWallpaperTarget将壁纸的目标窗口设置为壁纸本身。
        // getTopWallpaper 返回 mTopHideWhenLockedWallpaper 或者  mTopShowWhenLockedWallpaper
        if (mFindResults.wallpaperTarget == null && mFindResults.useTopWallpaperAsTarget) {
            mFindResults.setWallpaperTarget(
                    mFindResults.getTopWallpaper(mDisplayContent.isKeyguardLocked()));
        }
    }
```

调用 findWallpapers 寻找可以在锁屏或解锁显示的壁纸，为 mTopHideWhenLockedWallpaper 和 mTopShowWhenLockedWallpaper 赋值。    

```
    private void findWallpapers() {
        // 遍历所有壁纸窗口
        for (int i = mWallpaperTokens.size() - 1; i >= 0; i--) {
            final WallpaperWindowToken token = mWallpaperTokens.get(i);
            // 是否可以显示在锁屏
            final boolean canShowWhenLocked = token.canShowWhenLocked();
            for (int j = token.getChildCount() - 1; j >= 0; j--) {
                final WindowState w = token.getChildAt(j);
                // 如果不是 TYPE_WALLPAPER 类型的窗口，就忽略，继续查询其他窗口
                if (!w.mIsWallpaper) continue;
                if (canShowWhenLocked && !mFindResults.hasTopShowWhenLockedWallpaper()) {
                    // 可以显示在锁屏，而且mTopShowWhenLockedWallpaper还没有赋值
                    mFindResults.setTopShowWhenLockedWallpaper(w);
                } else if (!canShowWhenLocked && !mFindResults.hasTopHideWhenLockedWallpaper()) {
                    // 如果是系统壁纸，而且 mTopHideWhenLockedWallpaper 还没有赋值
                    mFindResults.setTopHideWhenLockedWallpaper(w);
                }
            }
        }
    }
```

通过 mFindWallpaperTargetFunction 回调对所有的窗口进行轮询。    

```
    private final ToBooleanFunction<WindowState> mFindWallpaperTargetFunction = w -> {
        final boolean useShellTransition = w.mTransitionController.isShellTransitionsEnabled();
        if (!useShellTransition) {
            // 没有开启ShellTransition的情况下
            // 如果此窗口的应用程序识别码处于隐藏状态且未添加动画效果
            if (w.mActivityRecord != null && !w.mActivityRecord.isVisible()
                    && !w.mActivityRecord.isAnimating(TRANSITION | PARENTS)) {
                ......
                return false;
            }
        } else {
            final ActivityRecord ar = w.mActivityRecord;
            // 如果动画窗口处于过渡状态，它仍然可以在屏幕上看到，因此我们应该检查这个窗口是否可以成为壁纸目标，
            // 即使 visibleRequested 为 false。
            if (ar != null && !ar.isVisibleRequested() && !ar.isVisible()) {
                // 不可见的 activity 不会成为 Wallpaper Target
                return false;
            }
        }
        ......

        final WindowContainer animatingContainer = w.mActivityRecord != null
                ? w.mActivityRecord.getAnimatingContainer() : null;
        if (!useShellTransition && animatingContainer != null
                && animatingContainer.isAnimating(TRANSITION | PARENTS)
                && AppTransition.isKeyguardGoingAwayTransitOld(animatingContainer.mTransit)
                && (animatingContainer.mTransitFlags
                & TRANSIT_FLAG_KEYGUARD_GOING_AWAY_WITH_WALLPAPER) != 0) {
            // Keep the wallpaper visible when Keyguard is going away.
            mFindResults.setUseTopWallpaperAsTarget(true);
        }

        if (mService.mPolicy.isKeyguardLocked()) {
        // 锁屏的情况
            if (w.canShowWhenLocked()) {
            // 可以在锁屏界面显示的窗口
                if (mService.mPolicy.isKeyguardOccluded() || (useShellTransition
                        ? w.inTransition() : mService.mPolicy.isKeyguardUnoccluding())) {
                    // 如果现在锁屏的状态为“occluded”
                    // 当前该在锁屏界面上的那个窗口处于Transition，那一般就是open或者close
                    // 该窗口或者该窗口对应的ActivityRecord是否是全屏的，
                    // 如果不是，那么将mNeedsShowWhenLockedWallpaper设置为true
                    // 可以这样理解：如果是一个非全屏的窗口盖在锁屏界面上，如果不显示锁屏壁纸，
                    // 那么屏幕上没有被这个非全屏窗口覆盖的部分就会由于没有内容显示从而黑屏。
                    mFindResults.mNeedsShowWhenLockedWallpaper = !isFullscreen(w.mAttrs)
                            || (w.mActivityRecord != null && !w.mActivityRecord.fillsParent());
                }
            } else if (w.hasWallpaper() && mService.mPolicy.isKeyguardHostWindow(w.mAttrs)
                    && w.mTransitionController.hasTransientLaunch(mDisplayContent)) {
                // If we have no candidates at all, notification shade is allowed to be the target
                // of last resort even if it has not been made visible yet.
                .....
                mFindResults.setWallpaperTarget(w);
                return false;
            }
        }

        final boolean animationWallpaper = animatingContainer != null
                && animatingContainer.getAnimation() != null
                && animatingContainer.getAnimation().getShowWallpaper();
        final boolean hasWallpaper = w.hasWallpaper() || animationWallpaper;
        if (isRecentsTransitionTarget(w) || isBackNavigationTarget(w)) {
            ...
            mFindResults.setWallpaperTarget(w);
            return true;
        } else if (hasWallpaper && w.isOnScreen()
                && (mWallpaperTarget == w || w.isDrawFinishedLw())) {
            ...
            mFindResults.setWallpaperTarget(w);
            mFindResults.setIsWallpaperTargetForLetterbox(w.hasWallpaperForLetterboxBackground());
            ...
            if (w.mActivityRecord == null && mDisplayContent.isKeyguardGoingAway()) {
                return false;
            }
            return true;
        }
        return false;
    };
```

### updateWallpaperTokens

updateWallpaperTokens 方法用来设置壁纸窗口的可见性

```
    private void updateWallpaperTokens(boolean visibility, boolean keyguardLocked) {
        ...
        WindowState topWallpaper = mFindResults.getTopWallpaper(keyguardLocked);
        WallpaperWindowToken topWallpaperToken =
                topWallpaper == null ? null : topWallpaper.mToken.asWallpaperToken();
        for (int curTokenNdx = mWallpaperTokens.size() - 1; curTokenNdx >= 0; curTokenNdx--) {
            final WallpaperWindowToken token = mWallpaperTokens.get(curTokenNdx);
            token.updateWallpaperWindows(visibility && (token == topWallpaperToken));
        }
    }
```

### updateWallpaperOffset()

作用：更新壁纸的偏移量，通常由 WallpaperManagerService 调用。

### isWallpaperVisible()

作用：判断当前壁纸是否对用户可见（例如未被全屏 Activity 完全覆盖）。

### setWindowWallpaperDisplayOffset()

作用：设置窗口与壁纸的对齐偏移（例如锁屏壁纸居中显示时的偏移计算）。

## WallpaperManagerService

```
    private final ComponentName mDefaultWallpaperComponent;

    // 存储系统壁纸
    private final SparseArray<WallpaperData> mWallpaperMap = new SparseArray<WallpaperData>();
    // 存储锁屏壁纸
    private final SparseArray<WallpaperData> mLockWallpaperMap = new SparseArray<WallpaperData>();

    protected WallpaperData mFallbackWallpaper;
```


## WallpaperService

实现壁纸的应用需要继承 WallpaperService，因此它运行在 App  进程。    


## WallpaperData

## WallpaperObserver

## 设置壁纸流程

设置壁纸流程：    

```
WallpaperManagerService.setWallpaper // 设置静态壁纸时调用这个方法
    WallpaperManagerService.getWallpaperSafeLocked
        WallpaperManagerService.loadSettingsLocked
            WallpaperDataParser.loadSettingsLocked()
            mWallpaperMap.put()
            mLockWallpaperMap.put()
            mLockWallpaperMap.remove()
        // 如果前面没有找到 wallpaper，这里就创建一个 WallpaperData
        new WallpaperData()
        mLockWallpaperMap.put()
        mWallpaperMap.put()
```

加载壁纸文件后触发：    

```
FileObserver$ObserverThread.onEvent
    WallpaperManagerService$WallpaperObserver.onEvent
        WallpaperManagerService$WallpaperObserver.updateWallpapers
            WallpaperManagerService.loadSettingsLocked
            WallpaperManagerService.bindWallpaperComponentLocked
                // 新建 WallpaperConnection
                newConn = new WallpaperConnection
                // 绑定壁纸服务
                Context.bindServiceAsUser()
                // detach 上一个壁纸的 WallpaperData
                WallpaperManagerService.maybeDetachLastWallpapers
                wallpaper.connection = newConn
                // 更新 mLastWallpaper
                updateCurrentWallpapers()
            WallpaperManagerService$WallpaperDestinationChangeHandler.complete
                WallpaperManagerService.updateEngineFlags
                    WindowManagerService$LocalService.setWallpaperShowWhenLocked
                        // 根据 FLAG_LOCK 来设置是否可以显示在锁屏
                        WallpaperWindowToken.setShowWhenLocked
            WallpaperManagerService.saveSettingsLocked
                
```

连接壁纸服务成功后进行创建 WallpaperWindowToken、attach壁纸服务等工作。    

```
WallpaperManagerService$WallpaperConnection.onServiceConnected
    WallpaperManagerService.attachServiceLocked
        WallpaperManagerService$WallpaperConnection.forEachDisplayConnector
            WallpaperManagerService$DisplayConnector.connectLocked
                WindowManagerService$LocalService.addWindowToken()
                    WindowManagerService.addWindowToken
                        // 创建 WallpaperWindowToken
                        new WallpaperWindowToken()
                            WallpaperController.addWallpaperToken()
                                mWallpaperTokens.add()
                WindowManagerService$LocalService.setWallpaperShowWhenLocked
                    WallpaperWindowToken.setShowWhenLocked
                // attach 壁纸服务
                WallpaperConnection.mService.attach()
                    -------> 壁纸进程
```

attch 壁纸服务调用到壁纸进程：    

```
IWallpaperService$Stub.onTransact
    WallpaperService$IWallpaperServiceWrapper.attach()
        HandlerCaller.obtainMessage(DO_ATTACH)
        HandlerCaller.sendMessage()
// 处理消息
WallpaperService$IWallpaperEngineWrapper.executeMessage
    case DO_ATTACH:
        WallpaperService$IWallpaperEngineWrapper.doAttachEngine
            WallpaperService$Engine.attach
                WallpaperService$Engine.updateSurface
                    // 添加窗口
                    Session.addToDisplay()
                        --------> system_server
                            new WindowState()
```

## 设置动态壁纸流程

还有动态壁纸时不一样的地方：

```
WallpaperManagerService.setWallpaperComponentChecked
    WallpaperManagerService.setWallpaperComponent
        WallpaperManagerService.setWallpaperComponentInternal
            WallpaperManagerService.bindWallpaperComponentLocked
            // 后面的流程和静态壁纸一样
```

## 息屏时壁纸切换流程

SystemUI:

```
KeyguardService.IKeyguardService.Stub.onStartedGoingToSleep()
    KeyguardLifecyclesDispatcher.dispatch(KeyguardLifecyclesDispatcher.STARTED_GOING_TO_SLEEP)
        Handler.sendMessage(STARTED_GOING_TO_SLEEP)
            KeyguardLifecyclesDispatcher.KeyguardLifecycleHandler.dispatchStartedGoingToSleep()
                WallpaperManagerService.notifyGoingToSleep()
```

system_server:

```
WallpaperManagerService.notifyGoingToSleep()
    WallpaperManagerService.DisplayConnector.IWallpaperEngine.dispatchWallpaperCommand()
```


壁纸进程：

```
WallpaperService.IWallpaperEngineWrapper.dispatchWallpaperCommand()
    WallpaperService.Engine.BaseIWindow.dispatchWallpaperCommand()
        HandlerCaller.sendMessage(WallpaperCommand)
            WallpaperService.IWallpaperEngineWrapper.executeMessage
                case MSG_WALLPAPER_COMMAND
                Engine.doCommand()
```

## 相关文章

[【问题分析】锁屏界面调起google语音助手后壁纸不可见【Android 14】 ](https://juejin.cn/post/7367229542933250102)
[Android窗口管理服务WindowManagerService对壁纸窗口（Wallpaper Window）的管理分析](https://blog.csdn.net/luoshengyang/article/details/8550820/)
