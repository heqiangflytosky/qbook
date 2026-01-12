---
title: Android LetterBox
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android LetterBox
date: 2022-11-23 10:00:00
---


## 概述


Letterbox 主要是系统用来对一些不用尺寸的设备进行UI适配，它可以在应用的显示区域之外显示一些样式。当应用的宽高比和它的容器比例不兼容的时候，就会以 Letterbox 模式打开。        
一般情况下，以横屏模式下打开一个强制竖屏的界面，就出出现 Letterbox。     

设置手机横屏模式：   

```
adb shell wm fixed-to-user-rotation enabled
adb shell wm user-rotation -d 0 lock 1
```

恢复：

```
adb shell wm user-rotation -d 0 free 0
adb shell wm fixed-to-user-rotation default
```

## 相关类和方法

```
// ActivityRecord.java
    final LetterboxUiController mLetterboxUiController;
```

```
//LetterboxUiController.java
    private Letterbox mLetterbox;
    // 是否需要显示 Letterbox
    boolean shouldShowLetterboxUi(WindowState mainWindow) {
        if (mIsRelaunchingAfterRequestedOrientationChanged) {
            return mLastShouldShowLetterboxUi;
        }

        final boolean shouldShowLetterboxUi =
                (mActivityRecord.isInLetterboxAnimation() || mActivityRecord.isVisible()
                        || mActivityRecord.isVisibleRequested())
                && mainWindow.areAppWindowBoundsLetterboxed()
                // Check for FLAG_SHOW_WALLPAPER explicitly instead of using
                // WindowContainer#showWallpaper because the later will return true when this
                // activity is using blurred wallpaper for letterbox background.
                && (mainWindow.getAttrs().flags & FLAG_SHOW_WALLPAPER) == 0;

        mLastShouldShowLetterboxUi = shouldShowLetterboxUi;

        return shouldShowLetterboxUi;
    }
```

```
// Letterbox.java
    private final LetterboxSurface mTop = new LetterboxSurface("top");
    private final LetterboxSurface mLeft = new LetterboxSurface("left");
    private final LetterboxSurface mBottom = new LetterboxSurface("bottom");
    private final LetterboxSurface mRight = new LetterboxSurface("right");

    public class LetterboxSurface {
        private final Rect mSurfaceFrameRelative = new Rect();
        private final Rect mLayoutFrameGlobal = new Rect();
        private final Rect mLayoutFrameRelative = new Rect();
        private void createSurface(SurfaceControl.Transaction t) {
            mSurface = mSurfaceControlFactory.get()
                    .setName("Letterbox - " + mType)
                    .setFlags(HIDDEN)
                    .setColorLayer()
                    .setCallsite("LetterboxSurface.createSurface")
                    .build();

            t.setLayer(mSurface, TASK_CHILD_LAYER_LETTERBOX_BACKGROUND)
                    .setColorSpaceAgnostic(mSurface, true);
        }
    }
```


```
//LetterboxConfiguration.java
    //获取 Letterbox 颜色：
    Color getLetterboxBackgroundColor() {
        if (mLetterboxBackgroundColorOverride != null) {
            return mLetterboxBackgroundColorOverride;
        }
        int colorId = mLetterboxBackgroundColorResourceIdOverride != null
                ? mLetterboxBackgroundColorResourceIdOverride
                : R.color.config_letterboxBackgroundColor;
        // Query color dynamically because material colors extracted from wallpaper are updated
        // when wallpaper is changed.
        return Color.valueOf(mContext.getResources().getColor(colorId));
    }
```

## 风格设置

可以通过 `adb shell wm -h` 查看如何通过命令来设置 Letterbox 的样式：

```
  set-letterbox-style
    Sets letterbox style using the following options:
      --aspectRatio aspectRatio
        Aspect ratio of letterbox for fixed orientation. If aspectRatio <= 1.0
        both it and R.dimen.config_fixedOrientationLetterboxAspectRatio will
        be ignored and framework implementation will determine aspect ratio.
      --minAspectRatioForUnresizable aspectRatio
        Default min aspect ratio for unresizable apps which is used when an
        app is eligible for the size compat mode.  If aspectRatio <= 1.0
        both it and R.dimen.config_fixedOrientationLetterboxAspectRatio will
        be ignored and framework implementation will determine aspect ratio.
      --cornerRadius radius
        Corners radius (in pixels) for activities in the letterbox mode.
        If radius < 0, both R.integer.config_letterboxActivityCornersRadius
        and it will be ignored and corners of the activity won't be rounded.
      --backgroundType [reset|solid_color|app_color_background
          |app_color_background_floating|wallpaper]
        Type of background used in the letterbox mode.
      --backgroundColor color
        Color of letterbox which is be used when letterbox background type
        is 'solid-color'. Use (set)get-letterbox-style to check and control
        letterbox background type. See Color#parseColor for allowed color
        formats (#RRGGBB and some colors by name, e.g. magenta or olive).
      --backgroundColorResource resource_name
        Color resource name of letterbox background which is used when
        background type is 'solid-color'. Use (set)get-letterbox-style to
        check and control background type. Parameter is a color resource
        name, for example, @android:color/system_accent2_50.
      --wallpaperBlurRadius radius
        Blur radius for 'wallpaper' letterbox background. If radius <= 0
        both it and R.dimen.config_letterboxBackgroundWallpaperBlurRadius
        are ignored and 0 is used.
      --wallpaperDarkScrimAlpha alpha
        Alpha of a black translucent scrim shown over 'wallpaper'
        letterbox background. If alpha < 0 or >= 1 both it and
        R.dimen.config_letterboxBackgroundWallaperDarkScrimAlpha are ignored
        and 0.0 (transparent) is used instead.
      --horizontalPositionMultiplier multiplier
        Horizontal position of app window center. If multiplier < 0 or > 1,
        both it and R.dimen.config_letterboxHorizontalPositionMultiplier
        are ignored and central position (0.5) is used.
      --verticalPositionMultiplier multiplier
        Vertical position of app window center. If multiplier < 0 or > 1,
        both it and R.dimen.config_letterboxVerticalPositionMultiplier
        are ignored and central position (0.5) is used.
      --isHorizontalReachabilityEnabled [true|1|false|0]
        Whether horizontal reachability repositioning is allowed for 
        letterboxed fullscreen apps in landscape device orientation.
      --isVerticalReachabilityEnabled [true|1|false|0]
        Whether vertical reachability repositioning is allowed for 
        letterboxed fullscreen apps in portrait device orientation.
      --defaultPositionForHorizontalReachability [left|center|right]
        Default position of app window when horizontal reachability is.
        enabled.
      --defaultPositionForVerticalReachability [top|center|bottom]
        Default position of app window when vertical reachability is.
        enabled.
      --persistentPositionForHorizontalReachability [left|center|right]
        Persistent position of app window when horizontal reachability is.
        enabled.
      --persistentPositionForVerticalReachability [top|center|bottom]
        Persistent position of app window when vertical reachability is.
        enabled.
      --isEducationEnabled [true|1|false|0]
        Whether education is allowed for letterboxed fullscreen apps.
      --isSplitScreenAspectRatioForUnresizableAppsEnabled [true|1|false|0]
        Whether using split screen aspect ratio as a default aspect ratio for
        unresizable apps.
      --isTranslucentLetterboxingEnabled [true|1|false|0]
        Whether letterboxing for translucent activities is enabled.
      --isUserAppAspectRatioSettingsEnabled [true|1|false|0]
        Whether user aspect ratio settings are enabled.
      --isUserAppAspectRatioFullscreenEnabled [true|1|false|0]
        Whether user aspect ratio fullscreen option is enabled.
      --isCameraCompatRefreshEnabled [true|1|false|0]
        Whether camera compatibility refresh is enabled.
      --isCameraCompatRefreshCycleThroughStopEnabled [true|1|false|0]
        Whether activity "refresh" in camera compatibility treatment should
        happen using the "stopped -> resumed" cycle rather than
        "paused -> resumed" cycle.
  reset-letterbox-style [aspectRatio|cornerRadius|backgroundType
      |backgroundColor|wallpaperBlurRadius|wallpaperDarkScrimAlpha
      |horizontalPositionMultiplier|verticalPositionMultiplier
      |isHorizontalReachabilityEnabled|isVerticalReachabilityEnabled
      |isEducationEnabled|defaultPositionMultiplierForHorizontalReachability
      |isTranslucentLetterboxingEnabled|isUserAppAspectRatioSettingsEnabled
      |persistentPositionMultiplierForHorizontalReachability
      |persistentPositionMultiplierForVerticalReachability
      |defaultPositionMultiplierForVerticalReachability]
    Resets overrides to default values for specified properties separated
    by space, e.g. 'reset-letterbox-style aspectRatio cornerRadius'.
    If no arguments provided, all values will be reset.
  get-letterbox-style
    Prints letterbox style configuration.

```

比如设置 Letterbox 颜色为红色：      

```
adb shell wm set-letterbox-style --backgroundColor red
```

## 计算

```
ActivityRecord.onConfigurationChanged
  WindowContainer.onConfigurationChanged
    ConfigurationContainer.onConfigurationChanged
      ActivityRecord.resolveOverrideConfiguration
        ActivityRecord.resolveFixedOrientationConfiguration
          resolvedBounds =getResolvedOverrideConfiguration().windowConfiguration.getBounds()
          ActivityRecord.applyAspectRatio(resolvedBounds)
```

```
    private void resolveFixedOrientationConfiguration(@NonNull Configuration newParentConfig) {
    
        final Rect resolvedBounds =
                getResolvedOverrideConfiguration().windowConfiguration.getBounds();
    
        // Apply aspect ratio to resolved bounds
        mIsAspectRatioApplied = applyAspectRatio(resolvedBounds, containingBoundsWithInsets,
                containingBounds, desiredAspectRatio);    
    }
```

## layout

```
ActivityRecord.layoutLetterboxIfNeeded
  LetterboxUiController.layoutLetterboxIfNeeded
    LetterboxUiController.shouldShowLetterboxUi
    Letterbox.layout
      Letterbox$LetterboxSurface.layout
    Letterbox.hide
      Letterbox.layout
        Letterbox$LetterboxSurface.layout
```

```
ActivityRecord.updateLetterboxSurfaceIfNeeded
  LetterboxUiController.updateLetterboxSurfaceIfNeeded
    LetterboxUiController.updateLetterboxSurfaceIfNeeded
      LetterboxUiController.layoutLetterboxIfNeeded
        Letterbox.layout
          Letterbox$LetterboxSurface.layout
```



## 显示或隐藏 Letterbox：


```
RootWindowContainer.performSurfacePlacementNoTrace:813
  RootWindowContainer.applySurfaceChangesTransaction:1019
    DisplayContent.applySurfaceChangesTransaction:5195
      WindowContainer.forAllWindows:1846
        WindowState.forAllWindows:4632
          WindowState.applyInOrderWithImeWindows:4767
            WindowContainer$ForAllWindowsConsumerWrapper.apply:2853
              ActivityRecord.updateLetterboxSurfaceIfNeeded:1950
                LetterboxUiController.updateLetterboxSurfaceIfNeeded:806
                  Letterbox.applySurfaceChanges:230
                    Letterbox$LetterboxSurface.applySurfaceChanges:458
                      if (!mSurfaceFrameRelative.isEmpty())
                      SurfaceControl.Transaction.show
                      else
                      SurfaceControl.Transaction.hide
```


```
        public void applySurfaceChanges(@NonNull SurfaceControl.Transaction t,
                @NonNull SurfaceControl.Transaction inputT) {
            if (!needsApplySurfaceChanges()) {
                // Nothing changed.
                return;
            }
            mSurfaceFrameRelative.set(mLayoutFrameRelative);
            if (!mSurfaceFrameRelative.isEmpty()) {
                if (mSurface == null) {
                    createSurface(t);
                }

                mColor = mColorSupplier.get();
                mParentSurface = mParentSurfaceSupplier.get();
                t.setColor(mSurface, getRgbColorArray());
                t.setPosition(mSurface, mSurfaceFrameRelative.left, mSurfaceFrameRelative.top);
                t.setWindowCrop(mSurface, mSurfaceFrameRelative.width(),
                        mSurfaceFrameRelative.height());
                t.reparent(mSurface, mParentSurface);

                mHasWallpaperBackground = mHasWallpaperBackgroundSupplier.getAsBoolean();
                updateAlphaAndBlur(t);

                t.show(mSurface);
            } else if (mSurface != null) {
                t.hide(mSurface);
            }
            if (mSurface != null && mInputInterceptor != null) {
                mInputInterceptor.updateTouchableRegion(mSurfaceFrameRelative);
                inputT.setInputWindowInfo(mSurface, mInputInterceptor.mWindowHandle);
            }
        }

```










