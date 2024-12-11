---
title: Android 本地窗口动画流程
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 本地窗口动画添加窗口动画流程
date: 2022-11-23 10:00:00
---

## 代码流程

```
WindowSurfacePlacer.performSurfacePlacement()
    WindowSurfacePlacer.performSurfacePlacementLoop()
        RootWindowContainer.performSurfacePlacement()
            RootWindowContainer.performSurfacePlacementNoTrace()
                RootWindowContainer.applySurfaceChangesTransaction()
                    DisplayContent.applySurfaceChangesTransaction()
                        WindowContainer.forAllWindows
                            WindowState.forAllWindows
                                WindowState.applyInOrderWithImeWindows()
                                    //执行callback
                                    DisplayContent.mApplySurfaceChangesTransaction
                                        WindowStateAnimator.commitFinishDrawingLocked()
                                            WindowState.performShowLocked()
                                                WindowStateAnimator.applyEnterAnimationLocked()
                                                    WindowStateAnimator.applyAnimationLocked()
                                                        // 加载动画资源，两种方式，下面详细解释
                                                        AnimationUtils.loadAnimation()
                                                            AnimationUtils.createAnimationFromXml()
                                                        // 第二种方式
                                                        AppTransition.loadAnimationAttr()
                                                            TransitionAnimation.loadAnimationAttr()
                                                                TransitionAnimation.getCachedAnimations()
                                                                    TransitionAnimation.getAnimationStyleResId()
                                                                TransitionAnimation.loadAnimationSafely()
                                                                    AnimationUtils.loadAnimation()
                                                                        AnimationUtils.createAnimationFromXml()
                                                        // 开始动画
                                                        WindowState.startAnimation()
                                                            getPendingTransaction() //获取一个事务
                                                            WindowContainer.startAnimation()
                                                                SurfaceAnimator.startAnimation()
                                                                    // 创建动画 leash
                                                                    SurfaceAnimator.createAnimationLeash()
                                                                        WindowContainer.makeSurface()
                                                                        SurfaceControl.Builder.setContainerLayer()
                                                                        // 设置 mAnimationLeash
                                                                        mAnimationLeash = leash;
                                                                    WindowState.getAnimationLeashParent()
                                                                        WindowContainer.getAnimationLeashParent()
                                                                            WindowContainer.getParentSurfaceControl()
                                                                                WindowContainer.getSurfaceControl()
                                                                    SurfaceControl.setParent()
                                                                    SurfaceControl.setEffectLayer()
                                                                    // leash surface 调整
                                                                    WindowState.onAnimationLeashCreated()
                                                                        WindowContainer.reassignLayer
                                                                        WindowContainer.resetSurfacePositionForAnimationLeash()
                                                                    // 启动动画
                                                                    LocalAnimationAdapter.startAnimation()
                                                                        SurfaceAnimationRunner.startAnimation()
                                                                            // 创建RunningAnimation对象
                                                                            new RunningAnimation
                                                                            PendingAnimations.put(animationLeash, runningAnim)
                                                                            Choreographer.postFrameCallback(this::startAnimations)
                                                                                SurfaceAnimationRunner.startPendingAnimationsLocked()
                                                                                    SurfaceAnimationRunner.startAnimationLocked()
                                                                                        ValueAnimator.addUpdateListener
                                                                                            SurfaceAnimationRunner.applyTransformation()
                                                                                                // 执行动画的实际逻辑，也就是修改leash的参数
                                                                                                WindowAnimationSpec.apply()
                                                                                        ValueAnimator.start()
                                                                            // 初始化动画的状态
                                                                            SurfaceAnimationRunner.applyTransformation()
                                                                                WindowAnimationSpec.apply()
                                                            WindowContainer.commitPendingTransaction()
                                                                //从动画线程触发对 prepareSurfaces 的调用
                                                                // 协调动画和窗口的显示
                                                                WindowContainer.scheduleAnimation()
                                                                    WindowManagerService.scheduleAnimationLocked()
                                                                        WindowAnimator.scheduleAnimation()
                                                                            Choreographer.postFrameCallback(mAnimationFrameCallback)
                                                                                WindowAnimator.mAnimationFrameCallback
                                                                                    WindowAnimator.animate()
                                                                                        DisplayContent.updateWindowsForAnimator()
                                                                                        DisplayContent.prepareSurfaces()
```

`WindowStateAnimator.applyEnterAnimationLocked()` 之前的流程在前面文章中已经介绍过，现在我们主要介绍从这个方法开始之后的动画流程。     

## applyEnterAnimationLocked

这个方法会确定 transit 的值，并执行 applyAnimationLocked

```
    void applyEnterAnimationLocked() {
        final int transit;
        if (mEnterAnimationPending) {
            mEnterAnimationPending = false;
            transit = WindowManagerPolicy.TRANSIT_ENTER;
        } else {
            transit = WindowManagerPolicy.TRANSIT_SHOW;
        }

        // Application类型的窗口一般是由ActivityRecord控制，壁纸类型由WallpaperController控制。
        if (mAttrType != TYPE_BASE_APPLICATION && !mIsWallpaper) {
            applyAnimationLocked(transit, true);
        }

        if (mService.mAccessibilityController.hasCallbacks()) {
            mService.mAccessibilityController.onWindowTransition(mWin, transit);
        }
    }
```

mEnterAnimationPending 在 WindowManagerService.addWindow 方法中被设置为 true。     

```
// WindowManagerService.java
    public int addWindow(Session session, IWindow client, LayoutParams attrs, int viewVisibility,
        ......
            final WindowStateAnimator winAnimator = win.mWinAnimator;
            winAnimator.mEnterAnimationPending = true;
            winAnimator.mEnteringAnimation = true;
```

因此，这里执行 `transit = WindowManagerPolicy.TRANSIT_ENTER`，顺便看一下其他几种类型：    

```
    /** Window has been added to the screen. */
    public static final int TRANSIT_ENTER = 1;
    /** Window has been removed from the screen. */
    public static final int TRANSIT_EXIT = 2;
    /** Window has been made visible. */
    public static final int TRANSIT_SHOW = 3;
    /** Window has been made invisible.
     * TODO: Consider removal as this is unused. */
    public static final int TRANSIT_HIDE = 4;
    /** The "application starting" preview window is no longer needed, and will
     * animate away to show the real window. */
    public static final int TRANSIT_PREVIEW_DONE = 5;
```


## applyAnimationLocked 加载动画并启动动画

参数 transit 表示动画的类型，isEntrance 相同类型的动画上次是否已经执行过。    

```
// WindowStateAnimator.java
    boolean applyAnimationLocked(int transit, boolean isEntrance) {
        if (mWin.isAnimating() && mAnimationIsEntrance == isEntrance) {
            // 检查是否正在执行窗口动画，并且正在执行的动画和要启动的动画是同一类型
            return true;
        }

        ......

        // 判断是否可以进行动画，比如屏幕是否冻结，是否开启屏幕等
        if (mWin.mToken.okToAnimate()) {
            // 获取动画的资源
            int anim = mWin.getDisplayContent().getDisplayPolicy().selectAnimation(mWin, transit);
            int attr = -1;
            Animation a = null;
            // 不同的资源选择不同的加载方法
            if (anim != DisplayPolicy.ANIMATION_STYLEABLE) {
                //ANIMATION_NONE表示不做动画
                if (anim != DisplayPolicy.ANIMATION_NONE) {
                    Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "WSA#loadAnimation");
                    // 加载动画
                    a = AnimationUtils.loadAnimation(mContext, anim);
                    Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
                }
            } else {
                // 非ANIMATION_STYLEABLE情况下根据 transit 来确定动画资源
                switch (transit) {
                    case WindowManagerPolicy.TRANSIT_ENTER:
                        attr = com.android.internal.R.styleable.WindowAnimation_windowEnterAnimation;
                        break;
                    case WindowManagerPolicy.TRANSIT_EXIT:
                        attr = com.android.internal.R.styleable.WindowAnimation_windowExitAnimation;
                        break;
                    case WindowManagerPolicy.TRANSIT_SHOW:
                        attr = com.android.internal.R.styleable.WindowAnimation_windowShowAnimation;
                        break;
                    case WindowManagerPolicy.TRANSIT_HIDE:
                        attr = com.android.internal.R.styleable.WindowAnimation_windowHideAnimation;
                        break;
                }
                if (attr >= 0) {
                    // 加载动画
                    a = mWin.getDisplayContent().mAppTransition.loadAnimationAttr(
                            mWin.mAttrs, attr, TRANSIT_OLD_NONE);
                }
            }
            ......
            if (a != null) {
                Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "WSA#startAnimation");
                // 启动动画
                mWin.startAnimation(a);
                Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
                mAnimationIsEntrance = isEntrance;
            }
        } else {
            mWin.cancelAnimation();
        }

        return mWin.isAnimating(0 /* flags */, ANIMATION_TYPE_WINDOW_ANIMATION);
    }
```

### selectAnimation 选择动画的资源

确定动画所使用的资源，返回 `ANIMATION_STYLEABLE` 以使用定义动画的样式资源，返回 `ANIMATION_NONE` 表示无动画，或者返回特定的资源ID。     
一些特殊定制的动画可以在这里设置。     

```
//DisplayPolicy.java
    int selectAnimation(WindowState win, int transit) {
        if (transit == TRANSIT_PREVIEW_DONE) {
            if (win.hasAppShownWindows()) {
                if (win.isActivityTypeHome()) {
                    // Dismiss the starting window as soon as possible to avoid the crossfade out
                    // with old content because home is easier to have different UI states.
                    return ANIMATION_NONE;
                }
                return R.anim.app_starting_exit;
            }
        }
        return ANIMATION_STYLEABLE;
    }
```


### 加载动画资源

有两种加载动画资源的方式。    
第一种方式：     
AnimationUtils.loadAnimation主要用于加载XML文件中定义的常规动画，这些动画可以用于任何View对象。     

```
//AnimationUtils.java
    public static Animation loadAnimation(Context context, @AnimRes int id)
            throws NotFoundException {

        XmlResourceParser parser = null;
        try {
            // 获取动画资源，根据解析的xml创建Animation对象
            parser = context.getResources().getAnimation(id);
            return createAnimationFromXml(context, parser);
        } catch (XmlPullParserException | IOException ex) {
            throw new NotFoundException(
                    "Can't load animation resource ID #0x" + Integer.toHexString(id), ex);
        } finally {
            if (parser != null) parser.close();
        }
    }
```

createAnimationFromXml就是根据应用侧定义XML，进行解析并创建动画。     

```
//AnimationUtils.java
    private static Animation createAnimationFromXml(
            Context c, XmlPullParser parser, AnimationSet parent, AttributeSet attrs)
            throws XmlPullParserException, IOException, InflateException {

        Animation anim = null;

        // Make sure we are on a start tag.
        int type;
        int depth = parser.getDepth();
        // 循环读取节点，并根据节点类型创建对应的动画    
        while (((type = parser.next()) != XmlPullParser.END_TAG || parser.getDepth() > depth)
                && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            String  name = parser.getName();

            if (name.equals("set")) {
                anim = new AnimationSet(c, attrs);
                createAnimationFromXml(c, parser, (AnimationSet)anim, attrs);
            } else if (name.equals("alpha")) {
                anim = new AlphaAnimation(c, attrs);
            } else if (name.equals("scale")) {
                anim = new ScaleAnimation(c, attrs);
            }  else if (name.equals("rotate")) {
                anim = new RotateAnimation(c, attrs);
            }  else if (name.equals("translate")) {
                anim = new TranslateAnimation(c, attrs);
            } else if (name.equals("cliprect")) {
                anim = new ClipRectAnimation(c, attrs);
            } else if (name.equals("extend")) {
                anim = new ExtendAnimation(c, attrs);
            } else {
                throw new InflateException("Unknown animation name: " + parser.getName());
            }

            if (parent != null) {
                parent.addAnimation(anim);
            }
        }

        return anim;

    }
```

比如应用创建的动画：

```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha android:fromAlpha="1.0"
        android:toAlpha="0"
        android:duration="1000"/>
</set>
```

可以通过loadAnimation加载XML    

```
ImageView img_show = (ImageView) findViewById(R.id.img_show);
//加载前面定义的exit.xml
Animation animation = AnimationUtils.loadAnimation(this, R.anim.exit);
img_show.startAnimation(animation);
```

第二种方法：

根据 attribute Id 加载动画，这种方式一般对应于应用界面的转场动画。    
`attr = com.android.internal.R.styleable.WindowAnimation_windowEnterAnimation;` 对应的资源为:frameworks/base/core/res/res/values/attrs.xml     

```
    <declare-styleable name="WindowAnimation">
        <!-- The animation used when a window is being added. -->
        <attr name="windowEnterAnimation" format="reference" />
        <!-- The animation used when a window is being removed. -->
        <attr name="windowExitAnimation" format="reference" />
        <!-- The animation used when a window is going from INVISIBLE to VISIBLE. -->
        <attr name="windowShowAnimation" format="reference" />
        <!-- The animation used when a window is going from VISIBLE to INVISIBLE. -->
        <attr name="windowHideAnimation" format="reference" />
```

对应我们应用侧设置的 sytle

```
    <style name="MyWindowAnimation">
        <item name="android:windowEnterAnimation">@anim/enter</item>
        <item name="android:windowExitAnimation">@anim/exit</item>
    </style>
```

attr是资源的一个ID，一般来说，添加窗口时的动画通过android:windowEnterAnimation属性设置时，attr的值为0x0；移除窗口时的动画通过android:windowExitAnimation属性设置时，attr的值为0x1。attr的值只与transit有关，后面调用loadAnimationAttr方法时会用到。    

```
// TransitionAnimation.java
    public Animation loadAnimationAttr(LayoutParams lp, int animAttr, int transit) {
        int resId = Resources.ID_NULL;
        Context context = mContext;
        if (animAttr >= 0) {
            // 获取动画资源
            AttributeCache.Entry ent = getCachedAnimations(lp);
            if (ent != null) {
                context = ent.context;
                resId = ent.array.getResourceId(animAttr, 0);
            }
        }
        // 更新具有透明效果的动画资源 resId
        resId = updateToTranslucentAnimIfNeeded(resId, transit);
        //加载动画
        if (ResourceId.isValid(resId)) {
            return loadAnimationSafely(context, resId, mTag);
        }
        return null;
    }
```

getCachedAnimations 方法主要是获取动画资源ID。

```
    private AttributeCache.Entry getCachedAnimations(LayoutParams lp) {
        ......
        // 判断应用侧是否设置了 LayoutParams 和 windowAnimations
        if (lp != null && lp.windowAnimations != 0) {
            // 判断是应用资源还是系统资源
            String packageName = lp.packageName != null ? lp.packageName : DEFAULT_PACKAGE;
            // 获取动画资源 ID
            int resId = getAnimationStyleResId(lp);
            if ((resId & 0xFF000000) == 0x01000000) {
                packageName = DEFAULT_PACKAGE;
            }
            if (mDebug) {
                Slog.v(mTag, "Loading animations: picked package=" + packageName);
            }
            // 这里面对资源做了缓存
            return AttributeCache.instance().get(packageName, resId,
                    com.android.internal.R.styleable.WindowAnimation);
        }
        return null;
    }
```

getAnimationStyleResId 方法会把应用侧设置的动画给 resId。  

```
        lp.windowAnimations = R.style.MyWindowAnimation
```

注意下面代码中的注释，对应应用的启动动画，一般是不允许应用来定制的。    

```
// TransitionAnimation.java
    public int getAnimationStyleResId(@NonNull LayoutParams lp) {
        int resId = lp.windowAnimations;
        if (lp.type == LayoutParams.TYPE_APPLICATION_STARTING) {
            // 请注意，我们不希望应用程序自定义启动窗口动画。由于此窗口专门用于在应用程序启动时显示，
            // 因此应用程序不应直接更改其动画。在这种情况下，它将使用系统资源来获取默认动画。
            resId = mDefaultWindowAnimationStyleResId;
        }
        return resId;
    }
```

## 启动动画

`WindowStateAnimator.applyAnimationLocked()` 方法中的 `mWin.startAnimation(a);` 用于启动动画，参数a为前面调用 loadAnimationAttr 方法后的值，即应用侧传递的动画。   
创建了一个AnimationAdapter对象，把动画包装到了WindowAnimationSpec中，最后通过startAnimation传递这个AnimationAdapter的对象。    
`WindowAnimationSpec` 的作用主要是定义窗口动画的各种属性和行为。这个类包含了一些字段，这些字段定义了动画的类型、持续时间、插值器等。    
当WMS需要为一个窗口启动动画时，它会创建一个WindowAnimationSpec对象，并将这个对象传递给 `SurfaceAnimationRunner`。然后，`SurfaceAnimationRunner` 会根据这个 `WindowAnimationSpec` 对象的属性来创建和执行动画。    
`commitPendingTransaction();` 方法会最终调用到 `WindowAnimator.scheduleAnimation()` 方法，协调动画与窗口的显示。    

```
// WindowState.java
    void startAnimation(Animation anim) {

        // Insets类型的动画由InsetsSourceProvider相关处理,这个类用于控制窗口动画的边界和剪裁。
        if (mControllableInsetProvider != null) {
            return;
        }
        // 动画参数的初始化
        final DisplayInfo displayInfo = getDisplayInfo();
        anim.initialize(mWindowFrames.mFrame.width(), mWindowFrames.mFrame.height(),
                displayInfo.appWidth, displayInfo.appHeight);
        anim.restrictDuration(MAX_ANIMATION_DURATION);
        anim.scaleCurrentDuration(mWmService.getWindowAnimationScaleLocked());
        /创建一个新的本地动画适配器（LocalAnimationAdapter），把初始化的anim设置到其中，该类用于管理动画的特定细节。
        final AnimationAdapter adapter = new LocalAnimationAdapter(
                new WindowAnimationSpec(anim, mSurfacePosition, false /* canSkipFirstFrame */,
                        0 /* windowCornerRadius */),
                mWmService.mSurfaceAnimationRunner);
        // 启动动画
        // 通过getPendingTransaction()获取一个事务
        startAnimation(getPendingTransaction(), adapter);
        commitPendingTransaction();
    }
```

### 传递启动动画的参数

启动动画通过调用 WindowState 和 WindowContainer 的 startAnimation 方法，传入一些参数，最后调用 SurfaceAnimator.startAnimation 来执行启动动画操作。    

```
// WindowState.java
    private void startAnimation(Transaction t, AnimationAdapter adapter) {
        startAnimation(t, adapter, mWinAnimator.mLastHidden, ANIMATION_TYPE_WINDOW_ANIMATION);
    }
    
// WindowContainer.java
    void startAnimation(Transaction t, AnimationAdapter anim, boolean hidden,
            @AnimationType int type) {
        startAnimation(t, anim, hidden, type, null /* animationFinishedCallback */);
    }
    void startAnimation(Transaction t, AnimationAdapter anim, boolean hidden,
            @AnimationType int type,
            @Nullable OnAnimationFinishedCallback animationFinishedCallback) {
        startAnimation(t, anim, hidden, type, animationFinishedCallback,
                null /* adapterAnimationCancelledCallback */, null /* snapshotAnim */);
    }
    void startAnimation(Transaction t, AnimationAdapter anim, boolean hidden,
            @AnimationType int type,
            @Nullable OnAnimationFinishedCallback animationFinishedCallback,
            @Nullable Runnable animationCancelledCallback,
            @Nullable AnimationAdapter snapshotAnim) {
        ProtoLog.v(WM_DEBUG_ANIM, "Starting animation on %s: type=%d, anim=%s",
                this, type, anim);

        // TODO: This should use isVisible() but because isVisible has a really weird meaning at
        // the moment this doesn't work for all animatable window containers.
        mSurfaceAnimator.startAnimation(t, anim, hidden, type, animationFinishedCallback,
                animationCancelledCallback, snapshotAnim, mSurfaceFreezer);
    }
```

我们来看一下这个调用流程添加的一些参数：   

 - Transaction t：事务对象，通过getPendingTransaction()获取。用于描述一系列的窗口操作，例如移动、调整大小、绘制等。这些操作在WMS中排队，并在适当的时机应用到窗口上。
 - AnimationAdapter anim：前面传入的 LocalAnimationAdapter 对象，对动画进行封装的类。在WindowContainer的构造方法中初始化的    
 - int type：ANIMATION_TYPE_WINDOW_ANIMATION 类型动画。
 - OnAnimationFinishedCallback animationFinishedCallback，Runnable animationCancelledCallback：动画完成和取消的回调，均为 null
 - SurfaceFreezer freezer：在动画开始时冻结窗口的更新，以防止在动画过程中窗口的内容闪烁。在WindowContainer的构造方法中初始化的。        

SurfaceAnimator的作用主要是控制窗口动画，它是窗口动画的中控，通过操控mLeash对象来实现窗口的大小、位置、透明度等动画属性的改变。     

```
// SurfaceAnimator.java
    void startAnimation(Transaction t, AnimationAdapter anim, boolean hidden,
            @AnimationType int type,
            @Nullable OnAnimationFinishedCallback animationFinishedCallback,
            @Nullable Runnable animationCancelledCallback,
            @Nullable AnimationAdapter snapshotAnim, @Nullable SurfaceFreezer freezer) {
        mAnimation = anim;
        mAnimationType = type;
        mSurfaceAnimationFinishedCallback = animationFinishedCallback;
        mAnimationCancelledCallback = animationCancelledCallback;
        //获取当前窗口的SurfaceControl
        final SurfaceControl surface = mAnimatable.getSurfaceControl();
        if (surface == null) {
            //没有surface，则取消当前的动画
            Slog.w(TAG, "Unable to start animation, surface is null or no children.");
            cancelAnimation();
            return;
        }
        //调用SurfaceFreezer中takeLeashForAnimation()获取mLeash，但是SurfaceFreezer中没有被初始化，所以这里的mLeash还是为null
        mLeash = freezer != null ? freezer.takeLeashForAnimation() : null;
        if (mLeash == null) {
            //创建mLeash
             mLeash = createAnimationLeash(mAnimatable, surface, t, type,
                                              mAnimatable.getSurfaceWidth(), mAnimatable.getSurfaceHeight(), 0 /* x */,
                                              0 /* y */, hidden, mService.mTransactionFactory);
            //创建动画“leash”后执行的一些操作，包括重置图层、重新分配图层以及重置Surface的位置
            mAnimatable.onAnimationLeashCreated(t, mLeash);
        }
        //处理动画开始时进行一些设置和准备工作
        mAnimatable.onLeashAnimationStarting(t, mLeash);
        if (mAnimationStartDelayed) {
            ProtoLog.i(WM_DEBUG_ANIM, "Animation start delayed for %s", mAnimatable);
            return;
        }
        //将leash传给AnimationAdapter，执行动画
        mAnimation.startAnimation(mLeash, t, type, mInnerAnimationFinishedCallback);
        if (ProtoLogImpl.isEnabled(WM_DEBUG_ANIM)) {
            StringWriter sw = new StringWriter();
            PrintWriter pw = new PrintWriter(sw);
            mAnimation.dump(pw, "");
            ProtoLog.d(WM_DEBUG_ANIM, "Animation start for %s, anim=%s", mAnimatable, sw);
        }
        //获取一个快照，并使用该快照来执行动画，我们这里snapshotAnim为null，因此不涉及
        if (snapshotAnim != null) {
            mSnapshot = freezer.takeSnapshotForAnimation();
            if (mSnapshot == null) {
                Slog.e(TAG, "No snapshot target to start animation on for " + mAnimatable);
                return;
            }
            mSnapshot.startAnimation(t, snapshotAnim, type);
        }
    }

```

### 创建 Leash 

入参：    

 - Animatable animatable：当前窗口
 - SurfaceControl surface：当前窗口的surface
 - Transaction t：一个事务对象，用于执行一系列操作
 - @AnimationType int type：动画类型

```
// SurfaceAnimator.java
    static SurfaceControl createAnimationLeash(Animatable animatable, SurfaceControl surface,
            Transaction t, @AnimationType int type, int width, int height, int x, int y,
            boolean hidden, Supplier<Transaction> transactionFactory) {
        ProtoLog.i(WM_DEBUG_ANIM, "Reparenting to leash for %s", animatable);
        //1.创建 Leash
        final SurfaceControl.Builder builder = animatable.makeAnimationLeash()
                //把leash的父节点设置为当前动画窗口的父节点
                .setParent(animatable.getAnimationLeashParent())
                // 设置名称，在winscope里面看到的名称
                .setName(surface + " - animation-leash of " + animationTypeToString(type))
                // TODO(b/151665759) Defer reparent calls
                // We want the leash to be visible immediately because the transaction which shows
                // the leash may be deferred but the reparent will not. This will cause the leashed
                // surface to be invisible until the deferred transaction is applied. If this
                // doesn't work, you will can see the 2/3 button nav bar flicker during seamless
                // rotation.
                .setHidden(hidden)
                .setEffectLayer()
                .setCallsite("SurfaceAnimator.createAnimationLeash");
        final SurfaceControl leash = builder.build();
        //2.设置 Leash 的大小、位置、可见性；
        t.setWindowCrop(leash, width, height);
        t.setPosition(leash, x, y);
        t.show(leash);
        t.setAlpha(leash, hidden ? 0 : 1);
        //3.设置 Leash 为当前动画 Surface 的 parent
        t.reparent(surface, leash);
        return leash;
    }
```

前面说过animatable为Animatable接口对象，WindowContainer为该接口的实现类，即animatable表示当前窗口，这个方法主要就是把创建好的leash图层加入到启动的Test窗口和它父亲节点WindowToken之间。    

```
// WindowContainer.java
    public Builder makeAnimationLeash() {
        return makeSurface().setContainerLayer();
    }
    Builder makeSurface() {
        final WindowContainer p = getParent();
        return p.makeChildSurface(this);
    }
```

```
// SurfaceControl.java
        public Builder setContainerLayer() {
            unsetBufferSize();
            return setFlags(FX_SURFACE_CONTAINER, FX_SURFACE_MASK);
        }
```

### Leash 层级调整

首先把leash的父节点设置为当前动画窗口的父节点。    

```
//SurfaceControl.java
        public Builder setParent(@Nullable SurfaceControl parent) {
            mParent = parent;
            return this;
        }
```

```
//WindowState.java
    public SurfaceControl getAnimationLeashParent() {
        if (isStartingWindowAssociatedToTask()) {
            return mStartingData.mAssociatedTask.mSurfaceControl;
        }
        return super.getAnimationLeashParent();
    }
```

```
// WindowContainer.java
    public SurfaceControl getAnimationLeashParent() {
        return getParentSurfaceControl();
    }
    public SurfaceControl getParentSurfaceControl() {
        final WindowContainer parent = getParent();
        if (parent == null) {
            return null;
        }
        return parent.getSurfaceControl();
    }
    public SurfaceControl getSurfaceControl() {
        return mSurfaceControl;
    }
```

看上面代码，getAnimationLeashParent 就是获取当前窗口的父SurfaceControl。    
那么合起来setParent(animatable.getAnimationLeashParent())的意思就是，把当前新创建的SurfaceControl(leash)的父亲设置为当前窗口父亲的SurfaceControl。
即此时leash图层和当前窗口TestWindow的父亲均是WindowToken，两人还在当兄弟。      

然后通过 setEffectLayer EffectLayer。它是一种特殊类型的SurfaceControl层，它默认表现得像一个容器层，但可以支持颜色填充、阴影和/或模糊效果。 这个EffectLayer主要就是用于实现一些视觉效果。     

最后通过 Transaction.reparent 把当前窗口的surface重新绑定到新创建的leash上。    

```
// SurfaceControl.Transaction.java
        public Transaction reparent(@NonNull SurfaceControl sc,
                @Nullable SurfaceControl newParent) {
            checkPreconditions(sc);
            long otherObject = 0;
            if (newParent != null) {
                newParent.checkNotReleased();
                otherObject = newParent.mNativeObject;
            }
            nativeReparent(mNativeObject, sc.mNativeObject, otherObject);
            mReparentedSurfaces.put(sc, newParent);
            return this;
        }
```

这个方法主要就是把当前窗口的SurfaceControl的父亲，修改为leash。此时leash图层变成了当前窗口TestWindow图层的父亲。刚才的兄弟节点现在变成了父子节点。     

应该注意：此时虽然图层关系发生了变化，但是WMS的窗口层级树关系并未发生改变，TestWindow 窗口的父节点仍然是 WindowToken。

### leash surface 调整

这里主要用来调整 surface 的位置，并初始化动画相关的一些状态。    

```
// WindowContainer.java
    public void onAnimationLeashCreated(Transaction t, SurfaceControl leash) {
        mLastLayer = -1;
        mAnimationLeash = leash;
        reassignLayer(t);
        // Leash is now responsible for position, so set our position to 0.
        resetSurfacePositionForAnimationLeash(t);
    }
    
    void assignChildLayers(Transaction t) {
        int layer = 0;

        // We use two passes as a way to promote children which
        // need Z-boosting to the end of the list.
        for (int j = 0; j < mChildren.size(); ++j) {
            final WindowContainer wc = mChildren.get(j);
            wc.assignChildLayers(t);
            if (!wc.needsZBoost()) {
                wc.assignLayer(t, layer++);
            }
        }
        for (int j = 0; j < mChildren.size(); ++j) {
            final WindowContainer wc = mChildren.get(j);
            if (wc.needsZBoost()) {
                wc.assignLayer(t, layer++);
            }
        }
        if (mOverlayHost != null) {
            mOverlayHost.setLayer(t, layer++);
        }
    }
    
    void resetSurfacePositionForAnimationLeash(Transaction t) {
        t.setPosition(mSurfaceControl, 0, 0);
        final SurfaceControl.Transaction syncTransaction = getSyncTransaction();
        if (t != syncTransaction) {
            // Avoid restoring to old position if the sync transaction is applied later.
            syncTransaction.setPosition(mSurfaceControl, 0, 0);
        }
        mLastSurfacePosition.set(0, 0);
    }
```

onLeashAnimationStarting 方法主要就是做了两件事：     
1.根据mAnimatingActivityRegistry的值判断，是否需要把有动画效果的Activity添加到列表中 
2.根据mNeedsAnimationBoundsLayer的值判断，否需要创建一个动画边界层。      

AnimationBoundsLayer的主要作用是限制动画的显示区域，以确保动画不会影响到应用程序的其他部分。    

```
    public void onLeashAnimationStarting(Transaction t, SurfaceControl leash) {
        if (mAnimatingActivityRegistry != null) {
            //1.将正在启动或者有动画效果的Activity添加到列表中，以便于管理和控制这些Activity的动画效果。
            mAnimatingActivityRegistry.notifyStarting(this);
        }

        if (mNeedsLetterboxedAnimation) {
            updateLetterboxSurface(findMainWindow(), t);
            mNeedsAnimationBoundsLayer = true;
        }

        // If the animation needs to be cropped then an animation bounds layer is created as a
        // child of the root pinned task or animation layer. The leash is then reparented to this
        // new layer.
        //2.否需要创建一个动画边界层
        if (mNeedsAnimationBoundsLayer) {
            mTmpRect.setEmpty();
            if (getDisplayContent().mAppTransitionController.isTransitWithinTask(
                    getTransit(), task)) {
                task.getBounds(mTmpRect);
            } else {
                final Task rootTask = getRootTask();
                if (rootTask == null) {
                    return;
                }
                // Set clip rect to root task bounds.
                rootTask.getBounds(mTmpRect);
            }
            //创建动画边界层
            mAnimationBoundsLayer = createAnimationBoundsLayer(t);

            // Crop to root task bounds.
            t.setLayer(leash, 0);
            t.setLayer(mAnimationBoundsLayer, getLastLayer());

            if (mNeedsLetterboxedAnimation) {
                final int cornerRadius = mLetterboxUiController
                        .getRoundedCornersRadius(findMainWindow());

                final Rect letterboxInnerBounds = new Rect();
                getLetterboxInnerBounds(letterboxInnerBounds);

                t.setCornerRadius(mAnimationBoundsLayer, cornerRadius)
                        .setCrop(mAnimationBoundsLayer, letterboxInnerBounds);
            }

            //重新将leash的父节点设置为动画边界层。
            t.reparent(leash, mAnimationBoundsLayer);
        }
    }
```

### 执行动画

接下来将leash传给LocalAnimationAdapter，执行动画。    
入参：    

 - SurfaceControl animationLeash:创建的动画 Leash
 - Transaction t:事务
 - int type:动画类型
 - OnAnimationFinishedCallback finishCallback:动画结束时的回调

```
//LocalAnimationAdapter.java
    public void startAnimation(SurfaceControl animationLeash, Transaction t,
            @AnimationType int type, @NonNull OnAnimationFinishedCallback finishCallback) {
        mAnimator.startAnimation(mSpec, animationLeash, t,
                () -> finishCallback.onAnimationFinished(type, this));
    }
```

这里会通过 SurfaceAnimationRunner 来启动动画。mSpec是WindowAnimationSpec对象，也就是窗口动画的各种属性和行为。 这两个参数都是在LocalAnimationAdapter构造方法中初始化的。        

```
    void startAnimation(AnimationSpec a, SurfaceControl animationLeash, Transaction t,
            Runnable finishCallback) {
        synchronized (mLock) {
            //创建RunningAnimation对象，把传递的参数赋值给RunningAnimation
            final RunningAnimation runningAnim = new RunningAnimation(a, animationLeash,
                    finishCallback);
            boolean requiresEdgeExtension = requiresEdgeExtension(a);

            if (requiresEdgeExtension) {
                ......
            }

            if (!requiresEdgeExtension) {
                mPendingAnimations.put(animationLeash, runningAnim);
                // mAnimationStartDeferred 主要是用来控制动画的播放时机的
                if (!mAnimationStartDeferred && mPreProcessingAnimations.isEmpty()) {
                    //通过Choreographer请求Vsync信号，接收到Vsync信号回调this::startAnimations
                    //即使用postFrameCallback在下一帧绘制完成后回调this::startAnimations
                    mChoreographer.postFrameCallback(this::startAnimations);
                }
            }

            // Some animations (e.g. move animations) require the initial transform to be
            // applied immediately.
            applyTransformation(runningAnim, t, 0 /* currentPlayTime */);
        }
    }
```

在 SurfaceAnimationRunner 类中设置mAnimationStartDeferred的值的方法有两个，deferStartingAnimations()和continueStartingAnimations()。    

```
//SurfaceAnimationRunner.java
    void deferStartingAnimations() {
        synchronized (mLock) {
            mAnimationStartDeferred = true;
        }
    }
    void continueStartingAnimations() {
        synchronized (mLock) {
            mAnimationStartDeferred = false;
            if (!mPendingAnimations.isEmpty() && mPreProcessingAnimations.isEmpty()) {
                mChoreographer.postFrameCallback(this::startAnimations);
            }
        }
    }
```

这两个方法，是在前面窗口添加流程中handleAppTransitionReady()方法里调用:   

```
	void handleAppTransitionReady() {
        ......
        mService.mSurfaceAnimationRunner.deferStartingAnimations();
        try {
            ......
        } finally {
            mService.mSurfaceAnimationRunner.continueStartingAnimations();
        }

       ......
    }
```

可以推测出，deferStartingAnimations() 方法用来将动画的启动时间推迟到 continueStartingAnimations() 方法被调用。
这意味着，当你调用deferStartingAnimations() 后，所有新的动画不会立即开始，它们会被放入待处理列表（在 mPendingAnimations 中）。
continueStartingAnimations() 方法则用于恢复动画的启动。当这个方法被调用时，动画的启动不再被推迟，所有待处理的动画会开始播放。
这两个方法通常用于动画的复杂控制，例如，在数据加载或其他后台任务完成之前，你可能希望延迟动画的播放，直到这些任务完成。
简单来说，这两个方法的主要作用是控制何时开始播放动画。    

接下来就在 `SurfaceAnimationRunner.startAnimationLocked()` 方法中启动动画并设置监听。    

```
    private void startAnimationLocked(RunningAnimation a) {
        final ValueAnimator anim = mAnimatorFactory.makeAnimator();

        // Animation length is already expected to be scaled.
        anim.overrideDurationScale(1.0f);
        anim.setDuration(a.mAnimSpec.getDuration());
        // 动画更新监听，在surface上apply动画效果
        anim.addUpdateListener(animation -> {
            synchronized (mCancelLock) {
                if (!a.mCancelled) {
                    final long duration = anim.getDuration();
                    long currentPlayTime = anim.getCurrentPlayTime();
                    if (currentPlayTime > duration) {
                        currentPlayTime = duration;
                    }
                    //根据动画的当前时间点获取相应的变换效果，并将这些效果应用到指定的Surface对象（leash图层）上，包括位置、透明度、裁剪和圆角等。
                    applyTransformation(a, mFrameTransaction, currentPlayTime);
                }
            }

            //执行下一帧动画
            scheduleApplyTransaction();
        });

        anim.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animation) {
                synchronized (mCancelLock) {
                    if (!a.mCancelled) {
                        // TODO: change this back to use show instead of alpha when b/138459974 is
                        // fixed.
                        mFrameTransaction.setAlpha(a.mLeash, 1);
                    }
                }
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                synchronized (mLock) {
                    mRunningAnimations.remove(a.mLeash);
                    synchronized (mCancelLock) {
                        if (!a.mCancelled) {

                            // Post on other thread that we can push final state without jank.
                            mAnimationThreadHandler.post(a.mFinishCallback);
                        }
                    }
                }
            }
        });
        a.mAnim = anim;
        mRunningAnimations.put(a.mLeash, a);

        anim.start();
        if (a.mAnimSpec.canSkipFirstFrame()) {
            // If we can skip the first frame, we start one frame later.
            anim.setCurrentPlayTime(mChoreographer.getFrameIntervalNanos() / NANOS_PER_MS);
        }

        // 手动执行一帧动画，否则，开始时间将仅在下一帧中设置，从而导致延迟。
        anim.doAnimationFrame(mChoreographer.getFrameTime());
    }
```

## 协调动画和窗口的显示

这部分逻辑在 `WindowContainer.commitPendingTransaction()` 方法中执行。最终调用到 WindowAnimator.scheduleAnimation()   

```
//WindowAnimator.java
    void scheduleAnimation() {
        if (!mAnimationFrameCallbackScheduled) {
            mAnimationFrameCallbackScheduled = true;
            mChoreographer.postFrameCallback(mAnimationFrameCallback);
        }
    }
```
mAnimationFrameCallback 在构造方法中初始化。    
```
    WindowAnimator(final WindowManagerService service) {
        mService = service;
        mContext = service.mContext;
        mPolicy = service.mPolicy;
        mTransaction = service.mTransactionFactory.get();
        service.mAnimationHandler.runWithScissors(
                () -> mChoreographer = Choreographer.getSfInstance(), 0 /* timeout */);

        mAnimationFrameCallback = frameTimeNs -> {
            synchronized (mService.mGlobalLock) {
                mAnimationFrameCallbackScheduled = false;
                animate(frameTimeNs);
                if (mNotifyWhenNoAnimation && !mLastRootAnimating) {
                    mService.mGlobalLock.notifyAll();
                }
            }
        };
    }
```

通过 prepareSurfaces 把前面的绘制的Surface提交到到SurfaceFlinger进行合成显示。     

```
    private void animate(long frameTimeNs) {
        if (!mInitialized) {
            return;
        }

        // Schedule next frame already such that back-pressure happens continuously.
        scheduleAnimation();

        final RootWindowContainer root = mService.mRoot;
        mCurrentTime = frameTimeNs / TimeUtils.NANOS_PER_MS;
        mBulkUpdateParams = 0;
        root.mOrientationChangeComplete = true;

        //开始一个 surface 事务
        mService.openSurfaceTransaction();
        try {
            // Remove all deferred displays, tasks, and activities.
            root.handleCompleteDeferredRemoval();

            final AccessibilityController accessibilityController =
                    mService.mAccessibilityController;
            final int numDisplays = root.getChildCount();
            //更新每个显示内容的窗口
            for (int i = 0; i < numDisplays; i++) {
                final DisplayContent dc = root.getChildAt(i);
                // 实际就是调用了WindowContainer中的forAllWindows
                // 该方法根据traverseTopToBottom决定遍历的顺序，来对所有的容器执行callback回调函数
                dc.updateWindowsForAnimator();
                //显示Surface
                dc.prepareSurfaces();
            }

            for (int i = 0; i < numDisplays; i++) {
                final DisplayContent dc = root.getChildAt(i);
                //检查App中的所有窗口是否都已绘制，并在需要时显示它们。
                dc.checkAppWindowsReadyToShow();
                if (accessibilityController.hasCallbacks()) {
                    accessibilityController.drawMagnifiedRegionBorderIfNeeded(dc.mDisplayId,
                            mTransaction);
                }
            }

            cancelAnimation();
            // 绘制水印
            if (mService.mWatermark != null) {
                mService.mWatermark.drawIfNeeded();
            }

        } catch (RuntimeException e) {
            Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
        }

        final boolean hasPendingLayoutChanges = root.hasPendingLayoutChanges(this);
        final boolean doRequest = (mBulkUpdateParams != 0 || root.mOrientationChangeComplete)
                && root.copyAnimToLayoutParams();
        if (hasPendingLayoutChanges || doRequest) {
            mService.mWindowPlacerLocked.requestTraversal();
        }

        final boolean rootAnimating = root.isAnimating(TRANSITION | CHILDREN /* flags */,
                ANIMATION_TYPE_ALL /* typesToCheck */);
        if (rootAnimating && !mLastRootAnimating) {
            Trace.asyncTraceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "animating", 0);
        }
        if (!rootAnimating && mLastRootAnimating) {
            mService.mWindowPlacerLocked.requestTraversal();
            Trace.asyncTraceEnd(Trace.TRACE_TAG_WINDOW_MANAGER, "animating", 0);
        }
        mLastRootAnimating = rootAnimating;

        final boolean runningExpensiveAnimations =
                root.isAnimating(TRANSITION | CHILDREN /* flags */,
                        ANIMATION_TYPE_APP_TRANSITION | ANIMATION_TYPE_SCREEN_ROTATION
                                | ANIMATION_TYPE_RECENTS /* typesToCheck */);
        if (runningExpensiveAnimations && !mRunningExpensiveAnimations) {
            // Usually app transitions put quite a load onto the system already (with all the things
            // happening in app), so pause snapshot persisting to not increase the load.
            mService.mSnapshotController.setPause(true);
            mTransaction.setEarlyWakeupStart();
        } else if (!runningExpensiveAnimations && mRunningExpensiveAnimations) {
            mService.mSnapshotController.setPause(false);
            mTransaction.setEarlyWakeupEnd();
        }
        mRunningExpensiveAnimations = runningExpensiveAnimations;
        //将当前事务合并到全局事务中
        SurfaceControl.mergeToGlobalTransaction(mTransaction);
        //应用全局事务，最终执行合成显示。
        mService.closeSurfaceTransaction("WindowAnimator");
        ProtoLog.i(WM_SHOW_TRANSACTIONS, "<<< CLOSE TRANSACTION animate");

        mService.mAtmService.mTaskOrganizerController.dispatchPendingEvents();
        executeAfterPrepareSurfacesRunnables();

    }
```


## 参考

Android T 窗口动画（本地动画）显示流程其二——添加流程 ：https://juejin.cn/post/7365348844992200742
