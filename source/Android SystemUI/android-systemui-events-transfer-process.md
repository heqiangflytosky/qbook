---
title: SystemUI -- 下拉面板事件分发流程
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍下拉面板事件分发流程
date: 2021-11-6 10:00:00
---

## 概述

本文基于原生 Android S 代码。


## 事件分发流程

事件的处理主要有下面几个类：

PhoneStatusBarView 主要处理从状态栏下拉通知面板
OverviewProxyService 主要操作桌面下拉通知面板
NotificationShadeWindowViewController 主要处理锁屏切换下拉通知的操作。
NotificationPanelViewController & PanelViewController 主要处理 QS Panel 的整体操作，比如显示，隐藏和整体滑动等等。
NotificationStackScrollLayoutController 主要处理通知中心的滑动，它处理事件时会通知 NotificationPanelViewController 更新 QS 的可见高度。

### PhoneStatusBarView

从状态栏下拉时，事件由 PhoneStatusBarView 分发给 NotificationPanelView 来处理面板的整体滑动。

```
//PanelBar.java
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (!panelEnabled()) {
            if (event.getAction() == MotionEvent.ACTION_DOWN) {
                Log.v(TAG, String.format("onTouch: all panels disabled, ignoring touch at (%d,%d)",
                        (int) event.getX(), (int) event.getY()));
            }
            return false;
        }

        if (event.getAction() == MotionEvent.ACTION_DOWN) {
            final PanelViewController panel = mPanel;
            if (panel == null) {
                return true;
            }
            boolean enabled = panel.isEnabled();
            if (!enabled) {
                // panel is disabled, so we'll eat the gesture
                return true;
            }
        }
        // 直接给 NotificationPanelView 来分发和处理事件
        return mPanel == null || mPanel.getView().dispatchTouchEvent(event);
    }
```

### OverviewProxyService

OverviewProxyService 监听了 Launcher 里面的下滑事件，在处理DOWN事件时设置 shader view 可见，那么后面shader view就可以分发和处理事件了。
具体看下面桌面下拉部分的介绍。

### NotificationShadeWindowView

NotificationShadeWindowView 处理当它可见时对所有事件的分发，以及对锁屏状态下的下拉(非状态栏下拉)事件的处理。

### NotificationPanelView

NotificationPanelView 主要处理QS完全展开与QS折叠之间状态下的事件处理，可以调用通知中心接口来移动通知中心。PanelView 主要是进行QS面板的整体操作，比如显示，隐藏和整体滑动等等。 
NotificationPanelView 和它的父类 PanelView 分别设置了事件拦截器和OnTouchListener，事件的处理工作主要由 NotificationPanelViewController 和 PanelViewController 创建的 TouchHandler() 来处理。

```
    public void setOnTouchListener(PanelViewController.TouchHandler touchHandler) {
        super.setOnTouchListener(touchHandler);
        mTouchHandler = touchHandler;
    }
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        return mTouchHandler.onInterceptTouchEvent(event);
    }
```

NotificationPanelView 对事件拦截：

```
NotificationShadeWindowView.dispatchTouchEvent()
    PanelView.onInterceptTouchEvent()
        NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
            PhoneStatusBarView.panelEnabled() // 是否允许显示通知面板
                CommandQueue.panelsEnabled()
            NotificationPanelViewController.shouldQuickSettingsIntercept()
            !PanelViewController.isFullyExpanded() &&       // 在QS展开的前提情况下(1,2,3场景之间的切换)，
            NotificationPanelViewController.onQsIntercept() // QS 是否需要拦截，拦截后执行QS折叠动作
                MotionEvent.ACTION_DOWN
                    mKeyguardShowing && shouldQuickSettingsIntercept()//在锁屏界面而且时在锁屏的状态栏开始滑动时，
                        requestDisallowInterceptTouchEvent(true)// 禁止父组件再拦截事件，保证move事件可以下发下来
                        mQsTracking = true //开始跟踪事件
                MotionEvent.ACTION_MOVE
                    if mQsTracking
                        NotificationPanelViewController.setQsExpansion() // 更新QS的高度以及通知中心位置，详见后面方法介绍
                        return true
                    NotificationPanelViewController.onQsExpansionStarted() // 开始跟踪手势，实现通知中心上下滑
                    mQsTracking = true
                    NotificationPanelViewController.notifyExpandingFinished()
            PanelViewController.TouchHandler.onInterceptTouchEvent() // PanelViewController 是否拦截
                NotificationPanelViewController.canCollapsePanelOnTouch() // 是否可以折叠 QSPanel
                    NotificationStackScrollLayoutController.isScrolledToBottom() // 通知中心有没有滑动到底部，滑动到底部表示不能滑动就拦截，不能再折叠了，move事件处理成QS整体操作
```

在大部分场景下，Down 事件一般都不是 NotificationPanelView 消费的，如果 `NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()` 这里拦截了，就处理QSPanel的整体操作，比如显示和隐藏等。不拦截了就向下分发，处理通知中心折叠操作等。


```
NotificationPanelViewController.TouchHandler.onTouch()
    NotificationPanelViewController.handleQsTouch()// NotificationPanelView处理 touch事件，比如通知中心的空白区域的滑动，在QS上上划呼出通知中心，具体看后面介绍
    ACTION_DOWN
        event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyCollapsed() // 在QS完全折叠情况下，消费Down事件，那么后面的Move和UP事件也会在这里处理，比如状态栏下拉
        handle = true
    PanelViewController.TouchHandler.onTouch() // NotificationPanelView 不处理，就走到这里，执行下拉通知整体操作，看滑动QS部分介绍
    
```

关于 onTouch 部分的操作，可以看下面QS滑动部分有详细介绍。


### NotificationStackScrollLayout

处理三种类型的事件：1.单个通知的展开和收缩手势，2.通知中心的滑动和滚动（这里所说的滑动和滚动，意在区分不同场景下通知中心的滚动），3.左右滑动删除通知操作

```
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (mTouchHandler != null && mTouchHandler.onInterceptTouchEvent(ev)) {
            return true;
        }
        return super.onInterceptTouchEvent(ev);
    }
```

```
    public boolean onTouchEvent(MotionEvent ev) {
        if (mTouchHandler != null && mTouchHandler.onTouchEvent(ev)) {
            return true;
        }

        return super.onTouchEvent(ev);
    }
```

```
        public boolean onInterceptTouchEvent(MotionEvent ev) {
            mView.initDownStates(ev);
            mView.handleEmptySpaceClick(ev);

            NotificationGuts guts = mNotificationGutsManager.getExposedGuts();
            boolean expandWantsIt = false;
            if (!mSwipeHelper.isSwiping()
                    && !mView.getOnlyScrollingInThisMotion() && guts == null) {
                // 处理单个通知的展开和收缩手势
                expandWantsIt = mView.getExpandHelper().onInterceptTouchEvent(ev);
            }
            boolean scrollWantsIt = false;
            if (!mSwipeHelper.isSwiping() && !mView.isExpandingNotification()) {
                // 处理通知中心的滚动
                scrollWantsIt = mView.onInterceptTouchEventScroll(ev);
            }
            boolean swipeWantsIt = false;
            if (!mView.isBeingDragged()
                    && !mView.isExpandingNotification()
                    && !mView.getExpandedInThisMotion()
                    && !mView.getOnlyScrollingInThisMotion()
                    && !mView.getDisallowDismissInThisMotion()) {
                // 处理左右滑动删除通知操作
                swipeWantsIt = mSwipeHelper.onInterceptTouchEvent(ev);
            }
            // Check if we need to clear any snooze leavebehinds
            boolean isUp = ev.getActionMasked() == MotionEvent.ACTION_UP;
            if (!NotificationSwipeHelper.isTouchInView(ev, guts) && isUp && !swipeWantsIt &&
                    !expandWantsIt && !scrollWantsIt) {
                mView.setCheckForLeaveBehind(false);
                mNotificationGutsManager.closeAndSaveGuts(true /* removeLeavebehind */,
                        false /* force */, false /* removeControls */, -1 /* x */, -1 /* y */,
                        false /* resetMenu */);
            }
            if (ev.getActionMasked() == MotionEvent.ACTION_UP) {
                mView.setCheckForLeaveBehind(true);
            }

            if (scrollWantsIt && ev.getActionMasked() != MotionEvent.ACTION_DOWN) {
                InteractionJankMonitor.getInstance().begin(mView,
                        CUJ_NOTIFICATION_SHADE_SCROLL_FLING);
            }
            return swipeWantsIt || scrollWantsIt || expandWantsIt;
        }
        
```

```
NotificationStackScrollLayout.onInterceptTouchEvent()
    NotificationStackScrollLayoutController.TouchHandler.onInterceptTouchEvent()
        ExpandHelper.onInterceptTouchEvent() // 处理单个通知的展开和收缩手势
        NotificationStackScrollLayout.onInterceptTouchEventScroll() // 处理通知中心的滚动
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayout.setIsBeingDragged()
                    requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截事件
        SwipeHelper.onInterceptTouchEvent() // 处理左右滑动删除通知操作
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayoutController.onBeginDrag()
                    NotificationStackScrollLayout.onSwipeBegin()
                        requestDisallowInterceptTouchEvent(true)
```

```
//NotificationStackScrollLayoutController.java
        @Override
        public boolean onTouchEvent(MotionEvent ev) {
            NotificationGuts guts = mNotificationGutsManager.getExposedGuts();
            boolean isCancelOrUp = ev.getActionMasked() == MotionEvent.ACTION_CANCEL
                    || ev.getActionMasked() == MotionEvent.ACTION_UP;
            mView.handleEmptySpaceClick(ev);
            boolean expandWantsIt = false;
            boolean onlyScrollingInThisMotion = mView.getOnlyScrollingInThisMotion();
            boolean expandingNotification = mView.isExpandingNotification();
            if (mView.getIsExpanded() && !mSwipeHelper.isSwiping() && !onlyScrollingInThisMotion
                    && guts == null) {
                ExpandHelper expandHelper = mView.getExpandHelper();
                if (isCancelOrUp) {
                    expandHelper.onlyObserveMovements(false);
                }
                boolean wasExpandingBefore = expandingNotification;
                // 处理单个通知的展开和收缩手势 
                expandWantsIt = expandHelper.onTouchEvent(ev);
                expandingNotification = mView.isExpandingNotification();
                if (mView.getExpandedInThisMotion() && !expandingNotification && wasExpandingBefore
                        && !mView.getDisallowScrollingInThisMotion()) {
                    mView.dispatchDownEventToScroller(ev);
                }
            }
            boolean scrollerWantsIt = false;
            if (mView.isExpanded() && !mSwipeHelper.isSwiping() && !expandingNotification
                    && !mView.getDisallowScrollingInThisMotion()) {
                // 处理通知中心的滚动
                scrollerWantsIt = mView.onScrollTouch(ev);
            }
            boolean horizontalSwipeWantsIt = false;
            if (!mView.isBeingDragged()
                    && !expandingNotification
                    && !mView.getExpandedInThisMotion()
                    && !onlyScrollingInThisMotion
                    && !mView.getDisallowDismissInThisMotion()) {
                // 处理左右滑动删除通知操作
                horizontalSwipeWantsIt = mSwipeHelper.onTouchEvent(ev);
            }

            // Check if we need to clear any snooze leavebehinds
            if (guts != null && !NotificationSwipeHelper.isTouchInView(ev, guts)
                    && guts.getGutsContent() instanceof NotificationSnooze) {
                NotificationSnooze ns = (NotificationSnooze) guts.getGutsContent();
                if ((ns.isExpanded() && isCancelOrUp)
                        || (!horizontalSwipeWantsIt && scrollerWantsIt)) {
                    // If the leavebehind is expanded we clear it on the next up event, otherwise we
                    // clear it on the next non-horizontal swipe or expand event.
                    checkSnoozeLeavebehind();
                }
            }
            if (ev.getActionMasked() == MotionEvent.ACTION_UP) {
                // Ensure the falsing manager records the touch. we don't do anything with it
                // at the moment.
                mFalsingManager.isFalseTouch(Classifier.SHADE_DRAG);
                mView.setCheckForLeaveBehind(true);
            }
            traceJankOnTouchEvent(ev.getActionMasked(), scrollerWantsIt);
            return horizontalSwipeWantsIt || scrollerWantsIt || expandWantsIt;
        }
    
```

```
NotificationStackScrollLayout.onTouchEvent()
    NotificationStackScrollLayoutController.TouchHandler.onTouchEvent()
        ExpandHelper.onTouchEvent() //处理单个通知的展开和收缩手势 
        NotificationStackScrollLayout.onScrollTouch() // 处理通知中心的滚动
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayout.overScrollDown() // 下滑
                NotificationStackScrollLayout.overScrollUp() //上滑
                NotificationStackScrollLayout.customOverScrollBy() //滚动
            MotionEvent.ACTION_UP
                NotificationStackScrollLayout.onOverScrollFling(true) // 展开QS
                NotificationStackScrollLayout.onOverScrollFling(false) // 通知中心滑动，回到原来位置
                NotificationStackScrollLayout.fling() // 处理通知中心放手后的惯性滚动，注意：不是回弹效果。
                OverScroller.springBack()
                NotificationStackScrollLayout.animateScroll()
        SwipeHelper.onTouchEvent() // 处理左右滑动删除通知操作
```


## QSPanel 的几种显示场景

NotificationStackScrollLayout 的三级折叠

0.QSPanel隐藏
1.通知中心全部显示
2.显示QQS和通知中心
3.显示QS，不显示通知中心

2到1的切换有个条件就是通知中心在2状态下时满屏的，可以滚动，这时的通知栏滑动处理成ScrollView的滚动问题。
一个比较重要的方法就是 `NotificationPanelViewController.canCollapsePanelOnTouch()`

```
    protected boolean canCollapsePanelOnTouch() {
        if (!isInSettings() && mBarState == KEYGUARD) {
            return true;
        }
        // 判断通知中心是否可以滚动
        if (mNotificationStackScrollLayoutController.isScrolledToBottom()) {
            return true;
        }

        return !mShouldUseSplitNotificationShade && (isInSettings() || mIsPanelCollapseOnQQS);
    }
```

在 `PanelViewController.TouchHandler.onInterceptTouchEvent()` 中相应 `MotionEvent.ACTION_MOVE` 做判断，如果通知中心不能滚动，就拦截 `MotionEvent.ACTION_MOVE` 事件，做 QSPanel 上移或者隐藏的动作。
如果通知中心可以滚动，那么就不拦截`MotionEvent.ACTION_MOVE` 事件，交给通知中心做滚动处理。

```
// PanelViewController.java
        public boolean onInterceptTouchEvent(MotionEvent event) {
                case MotionEvent.ACTION_MOVE:
                    final float h = y - mInitialTouchY;
                    addMovement(event);
                    if (canCollapsePanel || mTouchStartedInEmptyArea || mAnimatingOnDown) {
                        float hAbs = Math.abs(h);
                        float touchSlop = getTouchSlop(event);
                        if ((h < -touchSlop || (mAnimatingOnDown && hAbs > touchSlop))
                                && hAbs > Math.abs(x - mInitialTouchX)) {
                            cancelHeightAnimator();
                            startExpandMotion(x, y, true /* startTracking */, mExpandedHeight);
                            return true;
                        }
                    }
                    break;
```

NotificationStackScrollLayout.onInterceptTouchEventScroll 收到 `MotionEvent.ACTION_MOVE` 事件后，会做相应的判断，做事件的拦截：

```
    boolean onInterceptTouchEventScroll(MotionEvent ev) {
            case MotionEvent.ACTION_MOVE: {
                ......

                final int y = (int) ev.getY(pointerIndex);
                final int x = (int) ev.getX(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                final int xDiff = Math.abs(x - mDownX);
                if (yDiff > getTouchSlop(ev) && yDiff > xDiff) {
                    // 设置可以拖动 mIsBeingDragged 为true
                    setIsBeingDragged(true);
                    mLastMotionY = y;
                    mDownX = x;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                }
                break;
            }
        return mIsBeingDragged;
    }
```

然后事件就交给 `NotificationStackScrollLayoutController.TouchHandler.onTouchEvent() -> NotificationStackScrollLayout.onScrollTouch() -> NotificationStackScrollLayout.customOverScrollBy()` 处理上划滚动。


2->3 通知中心下滑，这时 QSPanel不对 `MotionEvent.ACTION_MOVE` 做拦截，这时交给 NotificationStackScrollLayout 来处理向下滚动的 `MotionEvent.ACTION_MOVE` 和 `MotionEvent.ACTION_UP` 事件。


2->0 

`MotionEvent.ACTION_UP` 事件后包含两种场景，一个是QSPanel消失，另外一个时回弹回2场景。这两种情况都是由 `PanelViewController.TouchHandler.onTouch` 中对 `MotionEvent.ACTION_UP` 的处理。
然后在 `PanelViewController.endMotionEvent` 方法中通过 `flingExpands()` 来计算是否 expand，根据expand来计算最终的位置。如果expand为true就回原位，否则上划消失。

```
    protected boolean flingExpands(float vel, float vectorVel, float x, float y) {
        if (mFalsingManager.isUnlockingDisabled()) {
            return true;
        }

        @Classifier.InteractionType int interactionType = vel > 0
                ? QUICK_SETTINGS : (
                        mKeyguardStateController.canDismissLockScreen() ? UNLOCK : BOUNCER_UNLOCK);

        if (isFalseTouch(x, y, interactionType)) {
            return true;
        }
        if (Math.abs(vectorVel) < mFlingAnimationUtils.getMinVelocityPxPerSecond()) {
            // 根据滑动的距离来判断
            return shouldExpandWhenNotFlinging();
        } else {
            // 根据滑动速度来判断
            return vel > 0;
        }
    }
    
    protected boolean shouldExpandWhenNotFlinging() {
        return getExpandedFraction() > 0.5f;
    }
```

0->2 切换时也由PanelViewController处理，此时满足 expand 条件，就会去做 expand 动画。

3->2 切换，在QS上做上划动作，NotificationStackScrollLayout 不消费DOWN事件，被NotificationPanelView子View消费，后续MOVE 和 UP事件事件再经过 NotificationPanelView 被拦截，被 NotificationPanelView.onTouch() -> NotificationPanelViewController.handleQsTouch()->onQsTouch() 处理。


## 状态栏下拉

### 单指滑动

从状态栏下拉时，事件由 PhoneStatusBarView 分发给 NotificationPanelView 来处理面板的整体滑动。此时的事件不经过NotificationShadeWindowView分发。
在这种情况下，PanelViewController.TouchHandler.onInterceptTouchEvent() 和 NotificationPanelViewController.TouchHandler.onTouch()在Down事件时返回true，消费该事件，那么后面的Move和UP事件也会在这里或者父类的onTouch()中处理。

```
StatusBarWindowView.dispatchTouchEvent()
    PhoneStatusBarView.onTouchEvent()
        PanelBar.onTouchEvent()
            NotificationPanelView.dispatchTouchEvent()
                NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
                    PanelViewController.TouchHandler.onInterceptTouchEvent()
                        ACTION_MOVE
                            return true 因为此时View并不可见。
                NotificationPanelViewController.TouchHandler.onTouch()
                    ACTION_DOWN
                    Down事件时返回true
                    PanelViewController.TouchHandler.onTouch()
                        处理Move和UP事件，执行整体下来操作。
                        ACTION_MOVE
                            PanelViewController.setExpandedHeightInternal() //设置QS展开的高度，更新shaderview可见性，具体看后面文章介绍
                        ACTION_UP
                            PanelViewController.endMotionEvent()
                                PanelViewController.fling()
                                   NotificationPanelViewController.flingToHeight()//下拉面板整体fling到指定位置，具体看后面文章介绍
                        
```

```
// PanelViewController.java
   public class TouchHandler implements View.OnTouchListener {
        public boolean onInterceptTouchEvent(MotionEvent event) {
            ......
            return (mView.getVisibility() != View.VISIBLE);
        }
```

### 双指滑动

这个场景指的是双指从状态栏下拉，和单指下拉的区别是单指下拉后的状态是显示QQS和通知中心，双指下拉的最终状态时显示QS。
具体的事件分发流程就和上面单指状态栏下拉是一样的，这里就不多介绍。只介绍一下双指下拉的不一样的流程。
这个场景中应用的两个比较重要的变量是 mTwoFingerQsExpandPossible 和 mQsExpandImmediate。

在下面的方法中进行赋值：

```
private boolean handleQsTouch(MotionEvent event) {
        ......
        if (action == MotionEvent.ACTION_DOWN && isFullyCollapsed() && isQsExpansionEnabled()) {
            // mTwoFingerQsExpandPossible 赋值为true表示可以进行双指下拉操作
            mTwoFingerQsExpandPossible = true;
        }
        if (mTwoFingerQsExpandPossible && isOpenQsEvent(event) && event.getY(event.getActionIndex())
                < mStatusBarMinHeight) {
            mMetricsLogger.count(COUNTER_PANEL_OPEN_QS, 1);
            mQsExpandImmediate = true; //双指操作的情况下接收DONW事件时赋值 mQsExpandImmediate 为true。
            mNotificationStackScrollLayoutController.setShouldShowShelfOnly(true);
            requestPanelHeightUpdate();
            ......
        }

}
```

```
    private boolean isOpenQsEvent(MotionEvent event) {
        final int pointerCount = event.getPointerCount();
        final int action = event.getActionMasked();

        final boolean
                twoFingerDrag =
                action == MotionEvent.ACTION_POINTER_DOWN && pointerCount == 2;

        final boolean
                stylusButtonClickDrag =
                action == MotionEvent.ACTION_DOWN && (event.isButtonPressed(
                        MotionEvent.BUTTON_STYLUS_PRIMARY) || event.isButtonPressed(
                        MotionEvent.BUTTON_STYLUS_SECONDARY));

        final boolean
                mouseButtonClickDrag =
                action == MotionEvent.ACTION_DOWN && (event.isButtonPressed(
                        MotionEvent.BUTTON_SECONDARY) || event.isButtonPressed(
                        MotionEvent.BUTTON_TERTIARY));

        return twoFingerDrag || stylusButtonClickDrag || mouseButtonClickDrag;
    }
```



滑动过程中
move 的时候用getMaxPanelHeight()计算 panel 高度，双指滑动时计算最大高度是用 calculatePanelHeightQsExpanded()，单指滑动是用 calculatePanelHeightShade()。具体看后面文章介绍。
onHeightUpdated 时双指滑动会同时调用 positionClockAndNotifications()  来更新通知中心位置以及setQsExpansion()来设置QS展开高度。单指滑动时在这里只调用positionClockAndNotifications()不调用setQsExpansion()。
这里的 setQsExpansion() 就是用QQS高度加上展开进度乘以最大展开高度（完全展开QS）来计算的。

而单指滑动这时因为最终状态是不显示QS的，因此没有主动调用的必要。也就是NotificationPanelViewController.在onLayoutChange()会附带的调用一下，设置的高度也是定值mQsMinExpansionHeight。
```
//NotificationPanelViewController.java
    private boolean handleQsTouch(MotionEvent event) {
        ....
        if (!mQsExpandImmediate && mQsTracking) {
            onQsTouch(event);
            if (!mConflictingQsExpansionGesture) {
                return true;
            }
        }
```

```
//NotificationPanelViewController.java
    protected void onHeightUpdated(float expandedHeight) {
    
        if (mQsExpandImmediate || mQsExpanded && !mQsTracking && mQsExpansionAnimator == null
                && !mQsExpansionFromOverscroll) {
            float t;
            if (mKeyguardShowing) {

                // On Keyguard, interpolate the QS expansion linearly to the panel expansion
                t = expandedHeight / (getMaxPanelHeight());
            } else {
                // In Shade, interpolate linearly such that QS is closed whenever panel height is
                // minimum QS expansion + minStackHeight
                float
                        panelHeightQsCollapsed =
                        mNotificationStackScrollLayoutController.getIntrinsicPadding()
                                + mNotificationStackScrollLayoutController.getLayoutMinHeight();
                float panelHeightQsExpanded = calculatePanelHeightQsExpanded();
                t =
                        (expandedHeight - panelHeightQsCollapsed) / (panelHeightQsExpanded
                                - panelHeightQsCollapsed);
            }
            float
                    targetHeight =
                    mQsMinExpansionHeight + t * (mQsMaxExpansionHeight - mQsMinExpansionHeight);
            setQsExpansion(targetHeight);
        }
```

```
    @Override
    protected int getMaxPanelHeight() {
        int min = mStatusBarMinHeight;
        if (!(mBarState == KEYGUARD)
                && mNotificationStackScrollLayoutController.getNotGoneChildCount() == 0) {
            int minHeight = mQsMinExpansionHeight;
            min = Math.max(min, minHeight);
        }
        int maxHeight;
        if (mQsExpandImmediate || mQsExpanded || mIsExpanding && mQsExpandedWhenExpandingStarted
                || mPulsing) {
            // 双指操作或者时显示QQS到显示QS场景变换时走这里
            maxHeight = calculatePanelHeightQsExpanded();
        } else {
            // 其他场景走这里计算
            maxHeight = calculatePanelHeightShade();
        }
        maxHeight = Math.max(min, maxHeight);
        ....
        return maxHeight;
    }
```

fling 的时候用getMaxPanelHeight()计算 panel 高度，双指滑动时计算最大高度是用 calculatePanelHeightQsExpanded()，单指滑动是用 calculatePanelHeightShade()。具体看后面文章介绍。

因此，我们可以看到，关键位置主要是根据 mQsExpandImmediate 在Move的时候额外调用了  setQsExpansion() 来设置QS展开，在Fling时计算 getMaxPanelHeight() 时根据 mQsExpandImmediate 做了处理。

## 桌面下拉

桌面下拉事件处理逻辑分为两种情况，一种是 OverviewProxyService 自己处理，另外一种是 OverviewProxyService + NotificationPanelViewController 来处理。
首先 OverviewProxyService 接收DOWN事件后，设置shade view可见。
第一种情况是 OverviewProxyService 自己处理，这种情况下可能是在手势很快的情况下，事件没有分发到 NotificationPanelView，那么就OverviewProxyService自己处理。
OverviewProxyService 只处理 DONW, UP和CANCEL事件。

```
OverviewProxyService.onStatusBarMotionEvent()
    ACTION_DOWN
        PanelViewController.startExpandLatencyTracking()
        StatusBar.onInputFocusTransfer()
            NotificationPanelViewController.startWaitingForOpenPanelGesture()
                NotificationPanelViewController.onTrackingStarted()
                    PanelViewController.onTrackingStarted()
                        mTracking = true
                        PanelViewController.notifyExpandingStarted()
                            NotificationPanelViewController.onExpandingStarted()
                                NotificationStackScrollLayoutController.onExpansionStarted()
                                mIsExpanding = true
                                NotificationPanelViewController.onQsExpansionStarted()
                        PanelViewController.notifyBarPanelExpansionChanged() // 可以看后面详细介绍
                            PhoneStatusBarView.panelExpansionChanged()
                                PanelBar.panelExpansionChanged()
                                    PanelBar.updateVisibility()
                                        NotificationPanelView.setVisibility()//设置 NotificationPanelView 可见
                                    PhoneStatusBarView.onPanelPeeked()
                                        StatusBar.makeExpandedVisible() //设置Shade可见，然后可以接收事件
                                            NotificationShadeWindowControllerImpl.setPanelVisible()
                                                NotificationShadeWindowControllerImpl.apply()
                                                    NotificationShadeWindowControllerImpl.applyVisibility()
                                                        NotificationShadeView.setVisibility() // 设置NotificationShadeView可见
                NotificationPanelViewController.updatePanelExpanded()
    ACTION_UP
    ACTION_CANCEL
        StatusBar.onInputFocusTransfer()
            NotificationPanelViewController.stopWaitingForOpenPanelGesture()
                 对 mExpectingSynthesizedDown 判断是否执行fling，如果PanelViewController处理down事件，这里收到cancel事件时就不会处理fling，否则在up事件时处理fling
                 NotificationPanelViewController.fling() // 这里也可以处理fling动画
```

第二种情况是OverviewProxyService + NotificationPanelViewController 来处理。因为已经设置了shade view可见，然后它就可以接收事件开始处理动画了。此时后面的Down，Move，Up事件都由 shade view 处理了，那么它会OverviewProxyService 收到一个CANCEL事件。
接下来就是 NotificationPanelViewController 接收Down和Move事件来处理通知面板的整体滑动操作。和状态栏下拉处理比较类似，具体看上面介绍。这种情况下会在 shouldGestureWaitForTouchSlop() 中设置 mExpectingSynthesizedDown = false，那么OverviewProxyService收到cancel事件时就不会处理fling。
如果 OverviewProxyService 没有收到 CANCEL事件，表示 NotificationPanelViewController 没有来得及去处理事件，那么就在OverviewProxyService.ACTION_UP中处理下拉面板的状态，这就是第一种情况。

## 滑动QS

这个场景表示在下面场景下的操作：
1.显示QQS和通知中心的情况下滑动通知中心以外区域
2.全部显示QS时在QS面板上面滑动。

这两种情况下，NotificationStackScrollLayout对DONW事件都不做处理。NotificationPanelViewController 对 Move 事件拦截，然后 Move和Up事件由 NotificationPanelViewController 或者 PanelViewController 处理。
有涉及到通知中心位移时就在 NotificationPanelViewController 中处理。处理下拉面板整体操作时就在 PanelViewController 中处理。

主要进行一下的逻辑操作：
1.设置QS显示高度
2.更新QQS的可见性
3.设置QS的绘制区域
4.更新通知中心的偏移
5.更新面板可见性

```
PanelView.onTouchEvent()
    NotificationPanelViewController.TouchHandler.onTouch()
        NotificationPanelViewController.handleQsTouch() // QS处理，用来更新 QS 的显示高度以及更新通知中心的位置。
                                                        // 更新QQS和QS可见性
                                                        //比如QS上面上划呼出通知中心，或者下滑隐藏通知中心。看后面对该方法的详细介绍
            ACTION_DOWN
                mQsTracking = true
                NotificationPanelViewController.onQsExpansionStarted()
                    NotificationPanelViewController.setQsExpansion() // 设置QS显示高度，更新QS和QQS的可见性, 设置QS的绘制区域,更新通知中心的偏移，后面详细介绍
            NotificationPanelViewController.onQsTouch()
        PanelViewController.TouchHandler.onTouch() // handleQsTouch 不处理，就走到这里，执行下拉通知整体操作
            MotionEvent.ACTION_MOVE
                NotificationPanelViewController.onTrackingStarted()
                    PanelViewController.onTrackingStarted()
                        PanelViewController.notifyBarPanelExpansionChanged() // 可以看后面详细介绍
                            PhoneStatusBarView.panelExpansionChanged()
                                PanelBar.panelExpansionChanged()
                                    PhoneStatusBarView.onPanelPeeked()
                                        StatusBar.makeExpandedVisible()
                                            CommandQueue.panelsEnabled() // 是否允许显示通知面板
                                            NotificationShadeWindowControllerImpl.setPanelVisible(true) // 设置面板可见
                                                NotificationShadeWindowControllerImpl.apply()
                                                    NotificationShadeWindowControllerImpl.applyVisibility()
                                                        NotificationShadeWindowView.setVisibility() // 更新面板可见性
                    PanelViewController.setExpandedHeightInternal() // 看后面介绍
                        NotificationPanelViewController.onHeightUpdated()
                            NotificationPanelViewController.positionClockAndNotifications()
                                NotificationPanelViewController.requestScrollerTopPaddingUpdate()
                                    NotificationStackScrollLayoutController.updateTopPadding() // 更新通知中心位置
            MotionEvent.ACTION_UP:
            MotionEvent.ACTION_CANCEL:
                PanelViewController.endMotionEvent()
                    PanelViewController.flingExpands() //判断是否expand，决定qspanel是消失还是展开。
                    PanelViewController.fling() // 开始 fling 动画
                        NotificationPanelViewController.flingToHeight()
                            PanelViewController.flingToHeight()
                                PanelViewController.createHeightAnimator()
                                    AnimatorUpdateListener
                                        PanelViewController.setExpandedHeightInternal() // 设置最终QS展开的高度
                                AnimatorListenerAdapter.onAnimationEnd()
                                    PanelViewController.onFlingEnd()
                                        PanelViewController.notifyBarPanelExpansionChanged()// 可以看后面详细介绍
                                ValueAnimator.start()
                    NotificationPanelViewController.onTrackingStopped()
                        PanelViewController.onTrackingStopped()
                            PhoneStatusBarView.onTrackingStopped()
                                PanelBar.onTrackingStopped()
                                    StatusBarKeyguardViewManager.showBouncer(false) // 设置BouncerView可见性
                            PanelViewController.notifyBarPanelExpansionChanged() // 可以看后面详细介绍
                                PhoneStatusBarView.panelExpansionChanged()
                                    PanelBar.panelExpansionChanged()
                                        PanelBar.onPanelCollapsed()
                                            PhoneStatusBarView.onPanelCollapsed()
                                                post(mHideExpandedRunnable)
                                                    StatusBar.makeExpandedInvisible()
                                                        NotificationShadeWindowControllerImpl.setPanelVisible(false) // 隐藏面板
                                                            NotificationShadeWindowControllerImpl.apply()
                                                                NotificationShadeWindowControllerImpl.applyVisibility()
                                                                    NotificationShadeWindowView.setVisibility() // 更新面板可见性
```


## 滑动通知中心

1.上滑
2.下滑
3.包含先上滑后再下滑：和第二种情况一样

前面也讲过，滑动通知中心也分两种情况，一种是上划做下拉面板的整体操作，一种是下滑，隐藏通知中心显示QS操作。
这两种情况下的Down事件，要么被 ExpandableNotificationRow 消费，满足下面条件且设置了ExpandableNotificationRow 设置了 setOnClickListener。

```
// ExpandableNotificationRow.java
    public boolean onTouchEvent(MotionEvent event) {
        if (event.getActionMasked() != MotionEvent.ACTION_DOWN
                || !isChildInGroup() || isGroupExpanded()) {
            return super.onTouchEvent(event);
        } else {
            return false;
        }
    }
```

要么ExpandableNotificationRow没消费，会被 NotificationStackScrollLayoutController.onTouchEvent() 消费。总之，NotificationStackScrollLayout 及其子View是可以消费Down事件的，那么后面的事件还会向它做分发。

但是 PanelViewController 在分发MOVE事件时，由于满足了下面的条件，对MOVE事件做了拦截和消费。那么接下来就是由 PanelViewController 来处理事件进行下拉面板的整体操作。

```
PanelViewController.java
    public class TouchHandler implements View.OnTouchListener {
        public boolean onInterceptTouchEvent(MotionEvent event) {
                ......
                case MotionEvent.ACTION_MOVE:
                    final float h = y - mInitialTouchY;
                    addMovement(event);
                    if (canCollapsePanel || mTouchStartedInEmptyArea || mAnimatingOnDown) {
                        float hAbs = Math.abs(h);
                        float touchSlop = getTouchSlop(event);
                        if ((h < -touchSlop || (mAnimatingOnDown && hAbs > touchSlop))
                                && hAbs > Math.abs(x - mInitialTouchX)) {
                            cancelHeightAnimator();
                            startExpandMotion(x, y, true /* startTracking */, mExpandedHeight);
                            return true;
                        }
                    }
                ......

    }
```

这种情况下和上面介绍的 QS 滑动流程类似，就不再介绍。

第二种情况时，NotificationStackScrollLayout 在处理 ACTION_MOVE 事件时，由于满足了下面的条件，因此 NotificationStackScrollLayout 对Move事件做了拦截。而且这种情况下会设置 requestDisallowInterceptTouchEvent(true)，阻止父组件对事件的拦截。那么后面的Move和Up事件就由NotificationStackScrollLayout来处理通知中心的滑动。
由于阻止了父组件的事件拦截，因此如果我们先滑动通知下滑然后再上滑是无法收起下拉面板的。

```
NotificationStackScrollLayout.java

    boolean onInterceptTouchEventScroll(MotionEvent ev) {
        ....
        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                ....
                final int yDiff = Math.abs(y - mLastMotionY);
                final int xDiff = Math.abs(x - mDownX);
                if (yDiff > getTouchSlop(ev) && yDiff > xDiff) {
                    setIsBeingDragged(true);
                    mLastMotionY = y;
                    mDownX = x;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                }
                break;
            }
        return mIsBeingDragged;

    }
```

```
NotificationShadeWindowView.dispatchTouchEvent()
    NotificationStackScrollLayout.onInterceptTouchEvent()
        NotificationStackScrollLayoutController.onInterceptTouchEvent()
            NotificationStackScrollLayout.onInterceptTouchEventScroll()
                MotionEvent.ACTION_MOVE
                    NotificationStackScrollLayout.setIsBeingDragged()
                        mIsBeingDragged = isDragged
                        requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截事件
    NotificationStackScrollLayout.onTouchEvent()
        NotificationStackScrollLayout.TouchHandler.onTouchEvent()
            NotificationStackScrollLayout.onScrollTouch()
                MotionEvent.ACTION_MOVE
                    NotificationStackScrollLayout.overScrollUp()
                    NotificationStackScrollLayout.overScrollDown()
                        NotificationStackScrollLayout.setOverScrolledPixels()
                            NotificationStackScrollLayout.setOverScrollAmount()
                                NotificationStackScrollLayout.setOverScrollAmountInternal
                                    NotificationStackScrollLayout.notifyOverscrollTopListener()
                                        NotificationPanelViewController.OnOverscrollTopChangedListener.onOverscrollTopChanged()
                                            NotificationPanelViewController.setQsExpansion() // 设置QS可见高度以及通知中心位置
                    NotificationStackScrollLayout.customOverScrollBy() // 通知栏展示到顶部时
                MotionEvent.ACTION_UP
                    NotificationStackScrollLayout.shouldOverScrollFling()
                    NotificationStackScrollLayout.onOverScrollFling(true) // 通知中心弹开，全部展开QS
                        NotificationPanelViewController.OnOverscrollTopChangedListener.flingTopOverscroll()
                            NotificationPanelViewController.flingSettings() // QS 或者 QQS的动画，可以处理面板折叠和展开的情况，看后面方法详解
                                ValueAnimator.start()
                                    AnimatorUpdateListener.onAnimationUpdate()
                                        NotificationPanelViewController.setQsExpansion() // 设置QS可见高度以及通知中心位置
                    NotificationStackScrollLayout.fling() // 处理通知中心放手后的惯性滚动，注意：不是回弹效果。
                        OverScroller.fling() 
                    NotificationStackScrollLayout.onOverScrollFling(false) // 通知中心滚动，回到原来位置
                    NotificationStackScrollLayout.animateScroll() // 通知中心列表中通知的滚动
```


## 锁屏下拉通知栏

### 状态栏下滑
这时的最终的状态可以是(3)显示QS，不显示通知中心。处理QS的滑动。

NotificationPanelViewController 没有拦截事件，NotificationStackScrollLayout 也没有消费，那么Down事件还是给 NotificationPanelView 来消费，以及后面的 Move 事件也被NotificationPanelView 来消费。
虽然在 NotificationPanelViewController.onInterceptTouchEvent 没有拦截，但是在 NotificationPanelViewController.onQsIntercept 处理Down事件时设置了 `mView.getParent().requestDisallowInterceptTouchEvent(true)`，那么它的父组件们就不会再拦截了。
下面来看一下Down事件的分发流程

```
NotificationShadeWindowView.dispatchTouchEvent()
    PanelView.onInterceptTouchEvent()
        NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
            NotificationPanelViewController.onQsIntercept()
                NotificationPanelViewController.shouldQuickSettingsIntercept() // QS是否需要拦截，如果需要就进行下面的设置
                View.getParent().requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截后面的事件
```

那么在什么情况下会拦截Down事件呢？

```
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!isQsExpansionEnabled() || mCollapsedOnDown || (mKeyguardShowing
                && mKeyguardBypassController.getBypassEnabled())) {
            return false;
        }
        // 如果时锁屏界面，那么header区域就是状态栏，否则就是QuickStatusBarHeader的区域。
        View header = mKeyguardShowing || mQs == null ? mKeyguardStatusBar : mQs.getHeader();
        // 根据header设置拦截区域
        mQsInterceptRegion.set(
                /* left= */ (int) mQsFrame.getX(),
                /* top= */ header.getTop(),
                /* right= */ (int) mQsFrame.getX() + mQsFrame.getWidth(),
                /* bottom= */ header.getBottom());
        // Also allow QS to intercept if the touch is near the notch.
        mStatusBarTouchableRegionManager.updateRegionForNotch(mQsInterceptRegion);
        //判读事件是否在header区域
        final boolean onHeader = mQsInterceptRegion.contains((int) x, (int) y);

        if (mQsExpanded) {
            return onHeader || (yDiff < 0 && isInQsArea(x, y));
        } else {
            return onHeader;
        }
    }

    private boolean onQsIntercept(MotionEvent event) {
        ......
        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
                ......
                if (mKeyguardShowing
                        && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, 0)) {
                    // Dragging down on the lockscreen statusbar should prohibit other interactions
                    // immediately, otherwise we'll wait on the touchslop. This is to allow
                    // dragging down to expanded quick settings directly on the lockscreen.
                    mView.getParent().requestDisallowInterceptTouchEvent(true);
                }
                ......
    }
```

此时 NotificationPanelViewController.TouchHandler.onTouch() 中的 handleQsTouch 是返回 true 的。
handleQsTouch() 方法来更新 QS 的显示高度以及更新通知中心的位置。

### 通知栏和其他区域下滑

这时的最终的状态可以是(2)显示QQS和通知中心，处理面板的整体滑动。

这种场景下的Down事件依旧是由 NotificationStackScrollLayout 或者其子 View ExpandableNotificationRow 消费。
但是 NotificationShadeWindowView 会拦截 `ACTION_MOVE` 事件，那么由 NotificationShadeWindowView.onTouchEvent() 来处理和消费 `ACTION_MOVE` 事件。

```
NotificationShadeWindowView.onInterceptTouchEvent()
    NotificationShadeWindowViewController.InteractionEventHandler.shouldInterceptTouchEvent()
        LockscreenShadeTransitionController.DragDownHelper.onInterceptTouchEvent()
            ACTION_MOVE
                startingChild != null || LockscreenShadeTransitionController.isDragDownAnywhereEnabled // 这里返回true，拦截事件
```

如果触电在通知上，或者 isDragDownAnywhereEnabled 为true的情况下，都是可以下拉的。

```
    internal val isDragDownAnywhereEnabled: Boolean
        get() = (statusBarStateController.getState() == StatusBarState.KEYGUARD &&
                !keyguardBypassController.bypassEnabled &&
                qS.isFullyCollapsed)
```

```
StatusBar.getStatusBarWindowTouchListener
    NotificationShadeWindowView.onTouchEvent()
        NotificationShadeWindowViewController.setupExpandedStatusBar().handleTouchEvent()
            LockscreenShadeTransitionController.DragDownHelper.onTouchEvent()
                ACTION_MOVE
                    LockscreenShadeTransitionController.dragDownAmount
                        NotificationPanelViewController.setTransitionToFullShadeAmount()
                            NotificationPanelViewController.updateQsExpansion()
                                QSFragment.setQsExpansion()
                                    QSContainerImpl.setTranslationY()
                                    QSFragment.updateQsBounds()
                                        NonInterceptingScrollView.setClipBounds() // 设置绘制区域
                        QSFragment.setTransitionToFullShadeAmount()
                            QSFragment.updateShowCollapsedOnKeyguard()
                                QSFragment.updateQsState()
                                    QuickStatusBarHeader.setVisibility()
                                    QuickStatusBarHeader.setExpanded()
                                    QSFooter.setVisibility()
                                    QSFooter.setExpanded()
                                    QSPanelController.setVisibility() // 设置QSPanel的可见性
                    LockscreenShadeTransitionController.onCrossedThreshold()
                        NotificationStackScrollLayoutController.setDimmed() //设置通知中心透明度
                            NotificationStackScrollLayout.setDimmed()
                                NotificationStackScrollLayout.animateDimmed()
                                NotificationStackScrollLayout.setDimAmount()
                ACTION_UP
                    LockscreenShadeTransitionController.DragDownHelper.stopDragging()
```

## 锁屏上滑解锁

### 通知栏上滑

这个场景其实和上面介绍的滑动通知中心场景下上滑收起下拉面板的流程是一样的。
NotificationStackScrollLayout 或其子 View 消费了 DOWN 事件，但是PanelViewController 拦截并消费了 MOVE 事件，后面的 UP 事件也由它进行处理。

```
NotificationShadeWindowView.dispatchTouchEvent()
    NotificationPanelViewController.TouchHandler.onTouch()
        PanelViewController.TouchHandler.onTouch()
            MotionEvent.ACTION_MOVE
                PanelViewController.setExpandedHeightInternal() //设置QS展开的高度
            MotionEvent.ACTION_UP
                PanelViewController.fling()
                    NotificationPanelViewController.flingToHeight() // 滚动到指定高度，后面文章详细介绍
                
```

### 其他区域上滑

NotificationPanelView 的子 View 对 Down 事件都没有消费，那么最终是 NotificationPanelViewController.TouchHandler.onTouch() 消费了 Down 事件，PanelViewController 处理后面的 Move 和 Up 事件。
这个场景和滑动 QS 的流程是一样的。

### 解锁



## 点击导航栏收起

```
StatusBar.onReceive()
    Intent.ACTION_CLOSE_SYSTEM_DIALOGS
        ShadeControllerImpl.animateCollapsePanels()
            PanelBar.collapsePanel()
                NotificationPanelViewController.collapse()
                    PanelViewController.collapse()
                        PanelViewController.fling()
                            NotificationPanelViewController.flingToHeight()
```