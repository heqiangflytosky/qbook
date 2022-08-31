---
title: Android 窗口事件相关设置
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: Android 窗口事件相关设置
date: 2022-8-31 10:00:00
---


## FLAG_NOT_FOCUSABLE，FLAG_NOT_TOUCHABLE,FLAG_NOT_TOUCH_MODAL

可以为窗口的 `LayoutParams.flags |= LayoutParams.FLAG_NOT_FOCUSABLE;`  设置上面的Flag，那么该窗口就不会接收事件，而把事件透传给它下面的窗口。    

## FLAG_SLIPPERY

允许跨窗口传递 Touch 事件。    

```
Window window = getWindow();
window.addFlags(WindowManager.LayoutParams.FLAG_SPLIT_TOUCH | WindowManager.LayoutParams.FLAG_SLIPPERY);
```

当有一个界面被覆盖时，这个事件就会在当前页面出发cancle事件，把事件传给覆盖的窗口，进行处理。需要注意这里仅限系统层开发。FLAG_SLIPPERY 这个 api 是hide的。    

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


