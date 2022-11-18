---
title: Android 窗口事件相关设置
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: Android 窗口事件相关设置
date: 2022-8-31 10:00:00
---


## FLAG_NOT_FOCUSABLE，FLAG_NOT_TOUCHABLE

可以为窗口的 `LayoutParams.flags |= LayoutParams.FLAG_NOT_FOCUSABLE;` 此窗口永远不会获得按键输入焦点，因此用户无法向其发送按键或其他按钮事件。那些按钮事件会转到它后面的任何可聚焦窗口。 这个标志还将启用`FLAG_NOT_TOUCH_MODAL` ，无论 `FLAG_NOT_TOUCH_MODAL` 是否被明确设置。
或者 `LayoutParams.flags |= LayoutParams.FLAG_NOT_TOUCHABLE;`  设置上面的Flag，那么该窗口就不会接收事件，而把事件透传给它下面的窗口。    

## FLAG_NOT_TOUCH_MODAL

即使当前的窗口可聚焦（未设置FLAG_NOT_FOCUSABLE时），也允许将窗口外部的任何指针事件发送到其后面的窗口。 否则，无论指针事件是否在窗口内，窗口它自己都将消耗所有指针事件。    

## FLAG_WATCH_OUTSIDE_TOUCH

如果你已经设置了 `FLAG_NOT_TOUCH_MODAL`，那么你可以设置 `FLAG_WATCH_OUTSIDE_TOUCH` 这个flag。这样一个点击事件如果发生在你的窗口之外的范围，你就会接收到一个特殊的 `MotionEvent：MotionEvent.ACTION_OUTSIDE`。注意，你只会接收到点击事件的第一下，而之后的 `DOWN/MOVE/UP` 等手势全都不会接收到。

## FLAG_SLIPPERY

允许跨窗口传递 Touch 事件。    

```
Window window = getWindow();
window.addFlags(WindowManager.LayoutParams.FLAG_SPLIT_TOUCH | WindowManager.LayoutParams.FLAG_SLIPPERY);
```

当有一个界面被覆盖时，就会给当前页面出发cancle事件，把事件传给覆盖的窗口，进行处理。
或者触摸区域划出当前窗口时，就会给当前页面出发cancle事件，把事件传给下面的窗口，进行处理。
需要注意这里仅限系统层开发。FLAG_SLIPPERY 这个 api 是hide的。    

## OnComputeInternalInsetsListener

如果我们为 `View.getViewTreeObserver().addOnComputeInternalInsetsListener()` 设置了 `OnComputeInternalInsetsListener`，在 `onComputeInternalInsets` 中为 `ViewTreeObserver.InternalInsetsInfo.touchableRegion.set()` 设置可点击区域，那么点击该区域以外时 View 就会收到 ACTION_OUTSIDE 事件。如果直接 return 表示整个 View 可以接收事件。        

下面的例子是SystemUI中为悬浮通知设置可点击区域的例子。    

```
    private void updateTouchableRegion() {
        ....
        if (shouldObserve) {
            mNotificationShadeWindowView.getViewTreeObserver()
                    .addOnComputeInternalInsetsListener(mOnComputeInternalInsetsListener);
            mNotificationShadeWindowView.requestLayout();
        } else {
            mNotificationShadeWindowView.getViewTreeObserver()
                    .removeOnComputeInternalInsetsListener(mOnComputeInternalInsetsListener);
        }
        ....
    }
    
    private final OnComputeInternalInsetsListener mOnComputeInternalInsetsListener =
            new OnComputeInternalInsetsListener() {
        @Override
        public void onComputeInternalInsets(ViewTreeObserver.InternalInsetsInfo info) {
            if (mIsStatusBarExpanded || mCentralSurfaces.isBouncerShowing()) {
                // The touchable region is always the full area when expanded
                return;
            }

            // Update touch insets to include any area needed for touching features that live in
            // the status bar (ie: heads up notifications)
            info.setTouchableInsets(ViewTreeObserver.InternalInsetsInfo.TOUCHABLE_INSETS_REGION);
            info.touchableRegion.set(calculateTouchableRegion());
        }
    };
```


