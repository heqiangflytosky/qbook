---
title: Android 动画实践
categories: Android
comments: true
tags: [Attribute和Style]
description: 动画实践
date: 2016-7-2 10:00:00
---


## 概述

介绍实现 Android 动画的一些常用方式。     
 
## 动画实践

### ValueAnimator

```
        ValueAnimator valueAnimator= ValueAnimator.ofFloat(0, 1f);
        valueAnimator.setDuration(300);
        valueAnimator.start();
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float value = (float)(animation.getAnimatedValue());
            }
        });
        valueAnimator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
            }
        });
```

### ObjectAnimator

指定 propertyName 来做一些属性动画。    

```
    private void test1() {
        ObjectAnimator notificationDown = ObjectAnimator.ofFloat(mView,"translationY",200)
                .setDuration(1000);
        notificationDown.setInterpolator(TRANSLATION_Y_INTERPOLATOR);

        // 缩放动画先缩放到0.5f然后会回到1f
        ObjectAnimator scaleX = ObjectAnimator.ofFloat(mView,"scaleX", 1f, 0.5f, 1f);
        ObjectAnimator scaleY = ObjectAnimator.ofFloat(mView,"scaleY", 1f, 0.5f, 1f);
        AnimatorSet scaleSet = new AnimatorSet();
        scaleSet.setInterpolator(TRANSLATION_Y_INTERPOLATOR);
        scaleSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
            }
        });
        scaleSet.setDuration(3000);
        scaleSet.playTogether(scaleX, scaleY);// 缩放动画一起做并行动画

        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.playSequentially(notificationDown,scaleSet);//位移动画和缩放动画做串行动画
        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
                mView.setTranslationY(0); //动画结束回到原来的位置
            }
        });
        animatorSet.start();
    }
```

自定义 Property 来做一些自定义的属性动画。    

```
    public static final IntProperty<View> BACKGROUND_ALPHA =
            new IntProperty<View>("backgroundAlpha") {
                @Override
                public void setValue(View view, int value) {
                    Drawable drawable = view.getBackground();
                    drawable.setAlpha(value);
                }

                @Override
                public Integer get(View view) {
                    return view.getBackground().getAlpha();
                }
            };
    public void test2() {
        ObjectAnimator animator = ObjectAnimator.ofInt(mView, BACKGROUND_ALPHA, 10);
        animator.setInterpolator(ALPHA_INTERPOLATOR);
        animator.setDuration(2000);
        animator.start();
    }
```

### PropertyValuesHolder

通过PropertyValuesHolder创建复合属性动画

```
            PropertyValuesHolder pvh1 = PropertyValuesHolder.ofFloat("scaleX", 1f, 0f, 1f);
            PropertyValuesHolder pvh2 = PropertyValuesHolder.ofFloat("scaleY", 1f, 0f, 1f);
            PropertyValuesHolder pvh3 = PropertyValuesHolder.ofFloat("translationY", 0f, 300f, 0f);
            ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, pvh1, pvh2, pvh3);
            animator.setDuration(10000).start();
```

```
            PropertyValuesHolder scaleX = PropertyValuesHolder.ofFloat(
                    "scaleX", 1f, 0.8f);
            PropertyValuesHolder scaleY = PropertyValuesHolder.ofFloat(
                    "scaleY", 1f, 0.8f);
            PropertyValuesHolder alpha = PropertyValuesHolder.ofFloat(
                    "alpha", 1f, 0f);
            ObjectAnimator scaleAnim = ObjectAnimator.ofPropertyValuesHolder(ActivatableNotificationView.this, scaleX, scaleY);
            scaleAnim.setDuration(DURATION_SCALE_OUT);
            ObjectAnimator alphaAnim = ObjectAnimator.ofPropertyValuesHolder(ActivatableNotificationView.this, alpha);
            alphaAnim.setDuration(450);
            AnimatorSet animatorSet = new AnimatorSet();
            animatorSet.setInterpolator(mNotificationTransYInterpolator);
            animatorSet.playTogether(scaleAnim,alphaAnim);
            animatorSet.start();
```


ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(mNotificationGuide,"translationY",49,0).setDuration(1000);
objectAnimator.start();


### View.animate() 
https://xiaozhuanlan.com/topic/7032651948
Alpha动画调用的不是View的setAlpha方法，而是 mView.setAlphaNoInvalidation。

### SpringAnimation

SpringAnimation三个构造函数，一个是传入FloatValueHolder对象，这个和我们使用ValueAnimator的作用类似，第二个传入Object和FloatPropertyCompat，这里的Object是任何对象，但对我UI而言就是View对象;FloatPropertyCompat就是我们要改变的属性，这里系统为我们提供了常见的View属性值动画：ALPHA,TRANSLATION_X、TRANSLATION_Y、TRANSLATION_Z等，我们直接使用即可；当然我们也可以设置自定义的属性。


```

        SpringAnimation scaleAnimator = new SpringAnimation(new FloatValueHolder(), 100f);
        scaleAnimator.setStartValue(0f);
        scaleAnimator.getSpring().setDampingRatio(0.85f);
        scaleAnimator.getSpring().setStiffness(180);
        scaleAnimator.addEndListener((dynamicAnimation, b, v, v1) -> {

        });
        scaleAnimator.addUpdateListener(new DynamicAnimation.OnAnimationUpdateListener() {
            @Override
            public void onAnimationUpdate(DynamicAnimation dynamicAnimation, float v, float v1) {

            }
        });
        scaleAnimator.start();
```

```
        SpringAnimation scaleXAnimation = new SpringAnimation(row, SpringAnimation.SCALE_X, 0.75f);
        scaleXAnimation.setStartValue(1);
        scaleXAnimation.getSpring().setDampingRatio(0.725f);
        scaleXAnimation.getSpring().setStiffness(100);
        scaleXAnimation.addEndListener((dynamicAnimation, b, v, v1) -> {

        });
        scaleXAnimation.start();

```

```
    private static final FloatPropertyCompat TRANSLATION_Y = new FloatPropertyCompat<ExpandableNotificationRow>("translation_y") {

        @Override
        public float getValue(ExpandableNotificationRow view) {
            return view.getTranslationY();
        }

        @Override
        public void setValue(ExpandableNotificationRow view, float v) {
            view.setCustomTranslationY(v);
        }

    };
    
        SpringAnimation mTranYAnimator = new SpringAnimation(view, TRANSLATION_Y, view.getYTranslation());
        mTranYAnimator.setStartValue(100);
        mTranYAnimator.getSpring().setDampingRatio(0.8f);
        mTranYAnimator.getSpring().setStiffness(250);
        mTranYAnimator.addUpdateListener(new DynamicAnimation.OnAnimationUpdateListener() {
            @Override
            public void onAnimationUpdate(DynamicAnimation dynamicAnimation, float v, float v1) {

            }
        });
        mTranYAnimator.addEndListener((dynamicAnimation, b, v, v1) -> {

        });
        mTranYAnimator.start();
```

```
        val scaleAnimator = SpringAnimation(FloatValueHolder())
        scaleAnimator.setStartValue(2f)
        scaleAnimator.spring = SpringForce().setDampingRatio(0.92f).setStiffness(300f).setFinalPosition(10f)
        scaleAnimator.addEndListener { dynamicAnimation: DynamicAnimation<*>?, b: Boolean, v: Float, v1: Float ->
            Log.d("Test","end value = $v")
        }
        scaleAnimator.addUpdateListener { dynamicAnimation, v, v1 ->
            Log.d("Test","update value = $v")
        }
        scaleAnimator.start()
```

### Transition 动画

[BasicTransition 动画示例代码](https://github.com/android/animation-samples/tree/main/BasicTransition)     
[CustomTransition 动画示例代码](https://github.com/android/animation-samples/tree/main/CustomTransition)      
[使用转场动效为布局变化添加动画效果](https://developer.android.google.cn/develop/ui/views/animations/transitions?hl=zh-cn)     

Transition框架是 Android 4.4.2 (API level 19) 引入的，借助它我们可以实现一些酷炫的过渡动画效果。    

那为什么要引入Transition动画呢？由于在Android引入了Metrial Desigon之后，动画的场面越来越大，比如以前我们制作一个动画可能涉及到的View就一个，或者就那么几个，如果我们一个动画中涉及到了当前Activity视图树中的各个View，那么情况就复杂了。比如我们要一次针对视图树中的10个View进行动画，这些View的效果都不同，可能有的是平移，有的是旋转，有的是淡入淡出，那么不管是使用之前哪种方式的动画，我们都需要为每个View定义一个开始状态和结束状态【关键帧，比如放缩，我们得设置fromXScale和toXScale 】，随着View个数的增加，这个情况会越来越复杂。这个时候如果使用一堆Animator去实现这一连串动画，代码将会又臭又长，Transition的出现大大减轻了开发的工作。     
#### 关键类
三个核心的类，分别是Scene、Transition和TransitionManager。    

##### Scene
Scene 场景，用于保存布局中所有View的属性值，创建Scene的方式可以通过getSceneForLayout方法：    

```
mScene1 = Scene.getSceneForLayout(mSceneRoot, R.layout.scene1, getContext());
```

也可以直接new Scene(ViewGroup sceneRoot, View layout)。    

```
View view1 = inflater.inflate(R.layout.scene1, container, false);
mScene2 = new Scene(mSceneRoot, view1);
```

Scene.getSceneForLayout 方式每次动画后都会回到 layout 配置的属性，`new Scene` 会保持在这个场景中所有视图的属性，通过下面的例子会看出来。如果我们在 mScene2 中修改了TextView的大小，那么到   mScene1 时会修改成 `R.layout.scene1` 配置的属性，再回到 mScene2 TextView 大小又会被修改成上次的大小。        
两种方式都需要传SceneRoot，即该场景的根节点。    

#####  Transition
Transition过渡动画，前面创建了两个场景，分别保存了视图的一些属性，比如Visibility、position等，Transition就是对于这些属性值的改变定义过渡的效果。从上图可以看到系统内置了一些常用的Transition，Transition的创建可以通过加载xml，如：

res/transition/fade_transition.xml

```
<fade xmlns:android="http://schemas.android.com/apk/res/android" />
```

然后在代码中：     
```
Transition mFadeTransition =
TransitionInflater.from(this).
inflateTransition(R.transition.fade_transition);
```
或者直接在代码中：     
```	
Transition mFadeTransition = new Fade();
```

##### TransitionManager

TransitionManager 用于将Scene和Transition联系起来，它提供了一系列的方法如setTransition(Scene fromScene, Scene toScene, Transition transition)指明起始场景和结束场景、他们的过渡动画是什么，go(Scene scene, Transition transition)，到指定的场景所使用的过渡动画是什么，beginDelayedTransition(ViewGroup sceneRoot, Transition transition)，在当前场景到下一帧的过渡效果是什么。     

```
TransitionManager.go(mScene3);
TransitionManager.go(mScene3,new Fade(Fade.IN));
```

##### 系统自带 Transition

xml 放在 `frameworks/base/core/res/res/transition`    

相关类：
 - AutoTransition:系统默认动画效果
 - ChangeBounds
 - ChangeClipBounds
 - ChangeImageTransform
 - ChangeScroll
 - ChangeTransform
 - Explode
 - Fade
 - Slide
 - TransitionSet
 - PathMotion

另外还可以自定义 Transition效果，需要继承Transition:    

```
public class CustomTransition extends Transition {
@Override
public void captureStartValues(TransitionValues values) {}
 
@Override
public void captureEndValues(TransitionValues values) {}
 
@Override
public Animator createAnimator(ViewGroup sceneRoot,
TransitionValues startValues,
TransitionValues endValues) {}
}
```

#### 针对单个视图的过渡

```
        TransitionManager.beginDelayedTransition(mRoot);
        // Then, we can just change view properties as usual.
        View textView = mRoot.findViewById(R.id.tv_animator);
        ViewGroup.LayoutParams params = textView.getLayoutParams();
        params.width = 400;
        params.height = 400;
        textView.setLayoutParams(params);
```

#### 针对多个视图的过渡


```
        mSceneRoot = findViewById(R.id.scene_root);
        mScene1 = new Scene(mSceneRoot, (ViewGroup) mSceneRoot.findViewById(R.id.scene_container));
        mScene2 = Scene.getSceneForLayout(mSceneRoot, R.layout.scene_2, this);
```

```
    public void onClick1(View view) {
        TransitionManager.go(mScene1);
    }

    public void onClick2(View view) {
        TransitionManager.go(mScene2);
    }
```

####  自定义 Transition


res/transition/scene3_transition_manager.xml    
```
<?xml version="1.0" encoding="utf-8"?>
<transitionManager xmlns:android="http://schemas.android.com/apk/res/android">
    <transition
        android:toScene="@layout/scene_3"
        android:transition="@transition/changebounds_slide_together"/>
</transitionManager>
```

res/transition/changebounds_slide_together.xml    
```
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
    <changeBounds/>
    <slide android:duration="1000" android:interpolator="@android:interpolator/accelerate_cubic" android:slideEdge="left">
        <targets>
            <target android:targetId="@id/tv_custom" />
        </targets>
    </slide>
</transitionSet>
```

```
        mScene3 = Scene.getSceneForLayout(mSceneRoot, R.layout.scene_3, this);
        mTransitionManagerForScene3 = TransitionInflater.from(this)
                .inflateTransitionManager(R.transition.scene3_transition_manager, mSceneRoot);
```

```
    public void onClick3(View view) {
        mTransitionManagerForScene3.transitionTo(mScene3);
    }
```

#### 注意事项

 - 对于 SurfaceView可能不起效果，因为SurfaceView的实例是在非UI线程更新的，因此会造成和其他视图动画不同步。
 - 某些特定的转换类型在应用到TextureView时可能不会产生所需的动画效果。
 - 继承自AdapterView的如ListView，与该框架不兼容。
 - 不要对包含文本的视图的大小进行动画

### 共享元素动画

参考[共享元素相关博客]()    

## 相关资料

[Android Docs Guide--Animations and Transitions](https://developer.android.com/training/animation)
[Android Docs Reference About Transition](https://developer.android.com/reference/android/transition/package-summary)
[Android Developer 动画和过渡](https://developer.android.google.cn/develop/ui/views/animations?hl=zh-cn)

android:stateListAnimator   
参考systemui中的MediaPlayer.SessionAction.Secondary    

通过和 AnimatorSet 配合来实现一些多个动画协同的动画效果。   

## 相关博客



