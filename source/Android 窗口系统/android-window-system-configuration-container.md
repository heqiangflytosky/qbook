---
title: Android ConfigurationContainer
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 ConfigurationContainer
date: 2022-11-23 10:00:00
---





## ConfigurationContainer

在前面的博客中也提到过，ConfigurationContainer 是窗口层级容器的父容器，其中包含了当前容器要使用的几种 Configuration，并且提供了请求更新 Configuration 配置项的机制。    
值得注意的是，作为窗口层级容器(WindowContainer)的父容器，ConfigurationContainer 本身并不参与层级结构的构建，而是主要类处理 Configuration 相关的一些公共逻辑。但是它却借助了 WindowContainer 构建起来的层级结构，从上到下进行了进行 Configuration 的分发。         

## Configuration

此类描述可能影响应用程序检索的资源的所有设备配置信息。这包括用户指定的配置选项（区域设置列表和缩放）以及设备配置（例如输入模式、屏幕大小和屏幕方向）。    
另外还有一个重要的 windowConfiguration，它是窗口状态相关的配置项，这时本文介绍的重点。     

```
    @TestApi
    public final WindowConfiguration windowConfiguration = new WindowConfiguration();
```

## WindowConfiguration

WindowConfiguration 包含了窗口容器的各种信息，比如显示区域，窗口模式等。    

通过 `adb shell dumpsys activity XXXX`得到的 WindowConfiguration 相关的信息：     

```
winConfig={ mBounds=Rect(0, 0 - 1080, 2340) mAppBounds=Rect(0, 92 - 1080, 2271) mMaxBounds=Rect(0, 0 - 1080, 2340) mDisplayRotation=ROTATION_0 mWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0}
```

mBounds，这个是最常用的 bounds，代表了 container 的边界。    
mAppBounds，用到的地方不多，代表的是App的可用边界，对比mBounds，一个很明显的区别就是mAppBounds的计算是考虑到了insets的，而mBounds的边界是不考虑insets的。     
看上面的 mBounds 代表的是屏幕的大小，而mAppBounds为屏幕高度减去了状态栏和导航栏的高度。     
mMaxBounds，代表了一个 container 能够得到的最大bounds，比如分屏下，一个 ActivityRecord 的各个bounds可能是：   

```
mBounds=Rect(0, 1184 - 1080, 2340) mAppBounds=Rect(0, 1276 - 1080, 2271) mMaxBounds=Rect(0, 0 - 1080, 2340)
```

大部分时候，mMaxBounds代表的都是屏幕的尺寸，但是少部分特殊模式下，它可能会小于屏幕的实际物理尺寸。     


mRotation 表示当前 Container 的旋转角度。     
mWindowingMode 表示当前 Container 的窗口模式。    
mActivityType 表示容器的 Activity的类型，有下面几种：    

```
    @IntDef(prefix = { "ACTIVITY_TYPE_" }, value = {
            ACTIVITY_TYPE_UNDEFINED,
            ACTIVITY_TYPE_STANDARD,
            ACTIVITY_TYPE_HOME,
            ACTIVITY_TYPE_RECENTS,
            ACTIVITY_TYPE_ASSISTANT,
            ACTIVITY_TYPE_DREAM,
    })
```

Activity 类型的计算在 ActivityRecord.setActivityType() 方法中。    
作为Task容器的TaskDisplayArea，也创建了一些专门的成员变量来存放它们的引用。    

```
    private Task mRootHomeTask;
    private Task mRootPinnedTask;
```

mAlwaysOnTop 表示当前 Container是否总是处于顶层。比如对于 PIP 模式的容器，是需要在 TaskDisplayArea 的顶层显示的。即要位于其他 Task 之上。      

## ConfigurationContainer 中的 Configuration 详解

### mRequestedOverrideConfiguration

作用：存储当前容器显式请求的覆盖配置。例如，当应用调用 Activity.setRequestedOrientation() 强制横屏时，该请求会被记录在此变量中。    
特点：

 - 表示用户或组件直接设置的配置覆盖，尚未经过系统处理。
 - 可能包含未经调整的值（如未考虑设备硬件限制或系统策略）。

看一下 ConfigurationContainer 的 setBounds() 方法实现，如果我们想修改容器的显示区域，可以用下面的方法。     

```
// ConfigurationContainer.java
    public int setBounds(Rect bounds) {
        ......
        mRequestsTmpConfig.setTo(getRequestedOverrideConfiguration());
        mRequestsTmpConfig.windowConfiguration.setBounds(bounds);
        onRequestedOverrideConfigurationChanged(mRequestsTmpConfig);
        ......
    }
```

基于以前的 mRequestedOverrideConfiguration 构建新要修改的 Configuration。在新的Configuration上设置Bounds。用新的Configuration更新以前的 mRequestedOverrideConfiguration。    

### mResolvedOverrideConfiguration

作用：保存经过系统处理后的有效覆盖配置。系统会根据父容器、设备状态、权限或其他约束条件，调整 mRequestedOverrideConfiguration 的值。      
特点：

 - 是mRequestedOverrideConfiguration的“实际生效”版本。
 - 例如，若设备禁用旋转，即使请求横屏，此处可能仍保持竖屏。

ConfigurationContainer 中默认行为是设置成 mRequestedOverrideConfiguration，但是它的子类会重载这个方法。    
```
// ConfigurationContainer.java
    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration);
        mResolvedOverrideConfiguration = "+mResolvedOverrideConfiguration);
```

比如 TaskFragment 会加入对 Bounds 和 WindowMode 的调整。    

```
// TaskFragment.java
    @Override
    void resolveOverrideConfiguration(Configuration newParentConfig) {
        mTmpBounds.set(getResolvedOverrideConfiguration().windowConfiguration.getBounds());
        super.resolveOverrideConfiguration(newParentConfig);
        final Configuration resolvedConfig = getResolvedOverrideConfiguration();

        if (mRelativeEmbeddedBounds != null && !mRelativeEmbeddedBounds.isEmpty()) {
            // For embedded TaskFragment, make sure the bounds is set based on the relative bounds.
            resolvedConfig.windowConfiguration.setBounds(translateRelativeBoundsToAbsoluteBounds(
                    mRelativeEmbeddedBounds, newParentConfig.windowConfiguration.getBounds()));
        }
        ...
        if (getActivityType() == ACTIVITY_TYPE_HOME && windowingMode == WINDOWING_MODE_UNDEFINED) {
            windowingMode = WINDOWING_MODE_FULLSCREEN;
            resolvedConfig.windowConfiguration.setWindowingMode(windowingMode);
        }

        ...
        if (!supportsMultiWindow()) {
            final int candidateWindowingMode =
                    windowingMode != WINDOWING_MODE_UNDEFINED ? windowingMode : parentWindowingMode;
            if (WindowConfiguration.inMultiWindowMode(candidateWindowingMode)
                    && candidateWindowingMode != WINDOWING_MODE_PINNED) {
                resolvedConfig.windowConfiguration.setWindowingMode(WINDOWING_MODE_FULLSCREEN);
            }
        }

        ...
        computeConfigResourceOverrides(resolvedConfig, newParentConfig);
    }
```

比如 ActivityRecord会在 MultiWindowMode 下对 Orientation 进行调整，以及 Letterbox 和 smallestScreenWidthDp 的调整。    

```
// ActivityRecord.java
    @Override
    void resolveOverrideConfiguration(Configuration newParentConfiguration) {
        final Configuration requestedOverrideConfig = getRequestedOverrideConfiguration();
        if (requestedOverrideConfig.assetsSeq != ASSETS_SEQ_UNDEFINED
                && newParentConfiguration.assetsSeq > requestedOverrideConfig.assetsSeq) {
            requestedOverrideConfig.assetsSeq = ASSETS_SEQ_UNDEFINED;
        }
        super.resolveOverrideConfiguration(newParentConfiguration);
        final Configuration resolvedConfig = getResolvedOverrideConfiguration();

        applyLocaleOverrideIfNeeded(resolvedConfig);

        ...
        final CompatDisplayInsets compatDisplayInsets = getCompatDisplayInsets();
        if (compatDisplayInsets != null) {
            resolveSizeCompatModeConfiguration(newParentConfiguration, compatDisplayInsets);
        } else if (inMultiWindowMode() && !isFixedOrientationLetterboxAllowed) {
            resolvedConfig.orientation = Configuration.ORIENTATION_UNDEFINED;
            if (!matchParentBounds()) {
                computeConfigByResolveHint(resolvedConfig, newParentConfiguration);
            }
        }
        ...

        if (isFixedOrientationLetterboxAllowed || compatDisplayInsets != null
                // In fullscreen, can be letterboxed for aspect ratio.
                || !inMultiWindowMode()) {
            updateResolvedBoundsPosition(newParentConfiguration);
        }

        boolean isIgnoreOrientationRequest = mDisplayContent != null
                && mDisplayContent.getIgnoreOrientationRequest();
        if (compatDisplayInsets == null
                && (mLetterboxBoundsForFixedOrientationAndAspectRatio != null
                        || (isIgnoreOrientationRequest && mIsAspectRatioApplied))) {
            resolvedConfig.smallestScreenWidthDp =
                    Math.min(resolvedConfig.screenWidthDp, resolvedConfig.screenHeightDp);
        }

        ......
    }
```

### mFullConfiguration

作用：最终的完整配置，对应于应用了自身 mResolvedOverrideConfiguration 的完整父级配置。      
特点：
 - 是真正应用到组件的配置。
 

```
    public Configuration getConfiguration() {
        return mFullConfiguration;
    }
```

怎么来理解呢？我们来看看 mFullConfiguration 的配置过程。      

```
    public void onConfigurationChanged(Configuration newParentConfig) {
    
    ....
        mFullConfiguration.setTo(newParentConfig);
        mFullConfiguration.windowConfiguration.unsetAlwaysOnTop();
        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
    ...
        for (int i = getChildCount() - 1; i >= 0; --i) {
            dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
        }
```

newParentConfig 为参数传进来的一个 congfig，其实也就是父容器的 mFullConfiguration。

```
    public void onRequestedOverrideConfigurationChanged(Configuration overrideConfiguration) {
        updateRequestedOverrideConfiguration(overrideConfiguration);
        // Update full configuration of this container and all its children.
        final ConfigurationContainer parent = getParent();
        onConfigurationChanged(parent != null ? parent.getConfiguration() : Configuration.EMPTY);
    }
```

首先将 mFullConfiguration 设置为传进来的 newParentConfig，然后再用 mResolvedOverrideConfiguration 去更新 mFullConfiguration。    
就是首先设置成传进来的父容器的 mFullConfiguration ，然后再结合自身的 mResolvedOverrideConfiguration 就是自身调整的 config 组成自己的 mFullConfiguration。    

### mMergedOverrideConfiguration

作用：表示合并后的 Override 配置。当存在层级关系（如Activity属于某个Task）时，当前容器的覆盖配置会与父容器的配置合并。      
特点：

 - 解决多层级配置冲突，确保子容器继承并合并父容器的配置。
 - 例如，父容器强制暗色主题，子容器请求横屏，合并后包含两者。

```
    void onMergedOverrideConfigurationChanged() {
        final ConfigurationContainer parent = getParent();
        if (parent != null) {
            mMergedOverrideConfiguration.setTo(parent.getMergedOverrideConfiguration());
            mMergedOverrideConfiguration.windowConfiguration.unsetAlwaysOnTop();
            mMergedOverrideConfiguration.updateFrom(mResolvedOverrideConfiguration);
        } else {
            mMergedOverrideConfiguration.setTo(mResolvedOverrideConfiguration);
        }
    }
```

那么 mMergedOverrideConfiguration 和 mFullConfiguration 有什么区别呢？看起来都是父容器的配置和自身 mResolvedOverrideConfiguration 的一个合并。     
其实他们的区别来源于 RootWindowContainer，RootWindowContainer 的 mFullConfiguration 和 mResolvedOverrideConfiguration 初始化就不一样。    

在下面的 Configuration 的更新流程可以看到，RootWindowContainer.onConfigurationChanged 的参数其实来源于 ActivityTaskManagerService.getGlobalConfiguration()。    

```
// ActivityTaskManagerService.java
    Configuration getGlobalConfiguration() {
        return mRootWindowContainer != null ? mRootWindowContainer.getConfiguration()
                : new Configuration();
    }
```

也就是 RootWindowContainer.mFullConfiguration，那么所有容器的 mFullConfiguration 也就来源于 RootWindowContainer.mFullConfiguration。       

但是容器的 mMergedOverrideConfiguration 来源于 RootWindowContainer.mResolvedOverrideConfiguration，可以在 `onMergedOverrideConfigurationChanged()` 看到。    
mResolvedOverrideConfiguration，代表的是container自己的覆盖配置，这个覆盖配置是当前container的mFullConfiguration区别于父container的mFullConfiguration的部分，那对于某个container来说，它的mMergedOverrideConfiguration收集的信息就是从RootWindowContainer到当前container这条路径上的所有container的mResolvedOverrideConfiguration之和，或者说override config之和。    


### 临时Config

mRequestsTmpConfig
mResolvedTmpConfig

### mLastReportedConfiguration



## MergedConfiguration


```
public class MergedConfiguration implements Parcelable {

    private final Configuration mGlobalConfig = new Configuration();
    private final Configuration mOverrideConfig = new Configuration();
    private final Configuration mMergedConfig = new Configuration();

```


## Configuration 的更新流程


更新 Configuration 的一些场景：    

```
WindowManagerService.setForcedDisplayDensityForUser
  WindowManagerService.setForcedDensityLockedInternal
    DisplayContent.setForcedDensity
      DisplayContent.reconfigureDisplayLocked
        DisplayContent.sendNewConfiguration()
          DisplayContent.updateDisplayOverrideConfigurationLocked
            ActivityTaskManagerService.updateGlobalConfigurationLocked
              RootWindowContainer.onConfigurationChanged
```

```
UiModeManagerService$Stub.setNightModeActivated
  UiModeManagerService$Stub.setNightModeActivatedForModeInternal
    UiModeManagerService.applyConfigurationExternallyLocked
      ActivityTaskManagerService.updateConfiguration(Configuration values)
        ActivityTaskManagerService.updateConfigurationLocked
          ActivityTaskManagerService.updateGlobalConfigurationLocked
            RootWindowContainer.onConfigurationChanged
```




```
ActivityManagerService.updatePersistentConfiguration
  ActivityManagerService.updatePersistentConfigurationWithAttribution
    ActivityTaskManagerService.updatePersistentConfiguration
      ActivityTaskManagerService.updateConfigurationLocked
        ActivityTaskManagerService.updateGlobalConfigurationLocked
```

```
WindowOrganizerController.lambda$startTransition$4
  WindowOrganizerController.applyTransaction
    Transition.applyDisplayChangeIfNeeded
      DisplayContent.sendNewConfiguration
        DisplayContent.updateDisplayOverrideConfigurationLocked
          ActivityTaskManagerService.updateGlobalConfigurationLocked
            RootWindowContainer.onConfigurationChanged
```

Configuration 更新的起点基本上是 `DisplayContent.sendNewConfiguration` 或者 `ActivityTaskManagerService.updateConfiguration`。        


Configuration 的更新流程：       

```
DisplayContent.sendNewConfiguration
    DisplayContent.updateDisplayOverrideConfigurationLocked
        values = new Configuration()
        DisplayContent.computeScreenConfiguration(values)
        DisplayContent.updateDisplayOverrideConfigurationLocked(values)
            ActivityTaskManagerService.updateGlobalConfigurationLocked
                mTempConfig.setTo(getGlobalConfiguration())
                    mTempConfig.updateFrom(values)
                RootWindowContainer.onConfigurationChanged(mTempConfig)
                    ConfigurationContainer.onConfigurationChanged(newParentConfig)
                        ConfigurationContainer.resolveOverrideConfiguration
                            mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration)
                        mFullConfiguration.setTo(newParentConfig)
                        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
                        ConfigurationContainer.onMergedOverrideConfigurationChanged
                            mMergedOverrideConfiguration.setTo(parent.getMergedOverrideConfiguration())
                            mMergedOverrideConfiguration.updateFrom(mResolvedOverrideConfiguration)
                        RootWindowContainer.dispatchConfigurationToChild
                            DisplayContent.performDisplayOverrideConfigUpdate
                                ConfigurationContainer.onRequestedOverrideConfigurationChanged
                                    ConfigurationContainer.updateRequestedOverrideConfiguration
                                        mRequestedOverrideConfiguration.setTo(overrideConfiguration)
```

在实战“自定义尺寸窗口中打开Activity”中WMShell修改 Task Bounds 到 WMCore 中更新配置的流程：      

```
          Configuration c = new Configuration(container.getRequestedOverrideConfiguration())
          Configuration.setTo(change.getConfiguration(), configMask, windowMask)
          WindowContainer(Task).onRequestedOverrideConfigurationChanged(WindowConfiguration c)
            ConfigurationContainer.onRequestedOverrideConfigurationChanged
              ConfigurationContainer.updateRequestedOverrideConfiguration
                //把 shell 传递来的配置给 mRequestedOverrideConfiguration
                mRequestedOverrideConfiguration.setTo(overrideConfiguration)
              // 执行 onConfigurationChanged
              Task.onConfigurationChanged(getParent().getConfiguration())
                Task.onConfigurationChangedInner
                  TaskFragment.onConfigurationChanged
                    WindowContainer.onConfigurationChanged
                      ConfigurationContainer.onConfigurationChanged
                        TaskFragment.resolveOverrideConfiguration
                          ConfigurationContainer.resolveOverrideConfiguration
                            // 把 mRequestedOverrideConfiguration 设置给 mResolvedOverrideConfiguration
                            mResolvedOverrideConfiguration.setTo(mRequestedOverrideConfiguration)
                          Task.resolveLeafTaskOnlyOverrideConfigs
                        // 把 mResolvedOverrideConfiguration 设置给 mFullConfiguration
                        mFullConfiguration.updateFrom(mResolvedOverrideConfiguration);
                        ConfigurationContainer.onMergedOverrideConfigurationChanged()
                          // 把 mResolvedOverrideConfiguration 设置给 mMergedOverrideConfiguration
                          mMergedOverrideConfiguration.updateFrom(mResolvedOverrideConfiguration)
                        // 分发给子容器
                        for (int i = getChildCount() - 1; i >= 0; --i) {
                        dispatchConfigurationToChild(getChildAt(i), mFullConfiguration);
                        }
```


```
RootWindowContainer.moveActivityToPinnedRootTask
    ConfigurationContainer.setWindowingMode
        ActivityRecord.onRequestedOverrideConfigurationChanged
            WindowContainer.onRequestedOverrideConfigurationChanged
                ConfigurationContainer.onRequestedOverrideConfigurationChanged
                    ConfigurationContainer.updateRequestedOverrideConfiguration
```



Configuration 变化对比：    

```
ActivityRecord.ensureActivityConfiguration
  ActivityRecord.updateReportedConfigurationAndSend
    ActivityRecord.getConfigurationChanges
      Configuration.diff
      SizeConfigurationBuckets.filterDiff
```


```
    public int diff(Configuration delta, boolean compareUndefined, boolean publicOnly) {
        int changed = 0;
        if ((compareUndefined || delta.fontScale > 0) && fontScale != delta.fontScale) {
            changed |= ActivityInfo.CONFIG_FONT_SCALE;
        }
        if ((compareUndefined || delta.mcc != 0) && mcc != delta.mcc) {
            changed |= ActivityInfo.CONFIG_MCC;
        }
        if ((compareUndefined || delta.mnc != 0) && mnc != delta.mnc) {
            changed |= ActivityInfo.CONFIG_MNC;
        }
        fixUpLocaleList();
        delta.fixUpLocaleList();
        ......

        if ((compareUndefined || delta.mGrammaticalGender != GRAMMATICAL_GENDER_UNDEFINED)
                && mGrammaticalGender != delta.mGrammaticalGender) {
            changed |= ActivityInfo.CONFIG_GRAMMATICAL_GENDER;
        }
        return changed;
    }
```

```
    public static int filterDiff(int diff, @NonNull Configuration oldConfig,
            @NonNull Configuration newConfig, @Nullable SizeConfigurationBuckets buckets) {
        if (buckets == null) {
            return diff;
        }

        final boolean nonSizeLayoutFieldsUnchanged =
                areNonSizeLayoutFieldsUnchanged(oldConfig.screenLayout, newConfig.screenLayout);
        if ((diff & CONFIG_SCREEN_SIZE) != 0) {
            final boolean crosses = buckets.crossesHorizontalSizeThreshold(oldConfig.screenWidthDp,
                    newConfig.screenWidthDp)
                    || buckets.crossesVerticalSizeThreshold(oldConfig.screenHeightDp,
                    newConfig.screenHeightDp);
            if (!crosses) {
                diff &= ~CONFIG_SCREEN_SIZE;
            }
        }
        if ((diff & CONFIG_SMALLEST_SCREEN_SIZE) != 0) {
            final int oldSmallest = oldConfig.smallestScreenWidthDp;
            final int newSmallest = newConfig.smallestScreenWidthDp;
            if (!buckets.crossesSmallestSizeThreshold(oldSmallest, newSmallest)) {
                diff &= ~CONFIG_SMALLEST_SCREEN_SIZE;
            }
        }
        if ((diff & CONFIG_SCREEN_LAYOUT) != 0 && nonSizeLayoutFieldsUnchanged) {
            if (!buckets.crossesScreenLayoutSizeThreshold(oldConfig, newConfig)
                    && !buckets.crossesScreenLayoutLongThreshold(oldConfig.screenLayout,
                    newConfig.screenLayout)) {
                diff &= ~CONFIG_SCREEN_LAYOUT;
            }
        }
        return diff;
    }
```
        

SizeConfigurationBuckets 如何创建      


## ResourcesManager.mResConfiguration


## 相关文章

[Android 14 WMS-Configuration 与ConfigurationContainer类](https://blog.csdn.net/u012627628/article/details/130124649)          
[【Android 12】ConfigurationContainer类 ](https://juejin.cn/post/7182894778408206394)      
[配置项容器ConfigurationContainer](https://zhuanlan.zhihu.com/p/655432749)     




