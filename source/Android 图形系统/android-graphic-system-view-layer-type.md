---
title: Android Hardware Layer 详解
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android Hardware Layer 
date: 2016-9-5 10:00:00
---




## 硬件加速和 Hardware Layer 区别

### 硬件加速和软件加速

有些开发者可能会把硬件加速和 Hardware Layer 搞混，硬件加速也叫 GPU 加速，硬件加速和软件加速的区别主要是图形的绘制是 GPU 还是 CPU。目前 Android 的版本中，默认都是开启硬件加速的。     

#### 硬件加速

硬件加速情况下，App 存在主线程和渲染线程，一帧的绘制是主线程和渲染线程一起配合执行的。      
主线程负责 input 事件，animation，UI 的 measure， layout，draw。draw 主要记录各个组件的绘制命令，序列化存储到 DisplayList。      
渲染线程使用 GPU 来执行实际的绘制。然后与 SurfaceFlinger 通信，获取 Buffer，调用 OpenGl 的绘制接口，执行真正的 DrawOP。最后将绘制好的 Buffer Swap 给 SurfaceFlinger 进行合成。     
GPU 的真正介入是在 RenderThread 中的部分操作中。     


<img src="/images/android-graphic-system-view-layer-type/1.png" width="739" height="295"/>

#### 软件加速

在 AndroidManifest 中使能软件加速：     

```
    <application
        .....
        android:hardwareAccelerated="false"
```

<img src="/images/android-graphic-system-view-layer-type/2.png" width="756" height="132"/>

可以看到，软件渲染下，只有主线程在工作，所有的渲染工作，都在主线程完成。       

### Hardware Layer 和 Software Layer

Software Layer 和 Hardware Layer , 这两个概念主要是针对 View 的说的， 与此时 App 是硬件渲染还是软件渲染没有直接关系，但是有依赖关系。     
可以为 View 设置不同类型的 Layer，这个 Layer 也叫“离屏缓冲”，也就是把当前绘制的内容保存在一个单独的缓冲层，后面针对 View 的操作可以复用这个缓冲层，比如平移，旋转，透明度变换等，而不必重新绘制。     
但是，如果有 View 的更新，比如删除 View 或者修改 View 的显示内容，这会使得 FBO（离屏渲染 Buffer）失效，在某些场景下反而性能会更差，这个问题后面会详细介绍。       

Layer 有下面三种类型：     

 - LAYER_TYPE_NONE
 - LAYER_TYPE_SOFTWARE：
 - LAYER_TYPE_HARDWARE

```
    public void buildLayer() {
        if (mLayerType == LAYER_TYPE_NONE) return;

        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo == null) {
            throw new IllegalStateException("This view must be attached to a window first");
        }

        if (getWidth() == 0 || getHeight() == 0) {
            return;
        }

        switch (mLayerType) {
            case LAYER_TYPE_HARDWARE:
                updateDisplayListIfDirty();
                if (attachInfo.mThreadedRenderer != null && mRenderNode.hasDisplayList()) {
                    attachInfo.mThreadedRenderer.buildLayer(mRenderNode);
                }
                break;
            case LAYER_TYPE_SOFTWARE:
                buildDrawingCache(true);
                break;
        }
    }
```

#### LAYER_TYPE_NONE

表示这个 View 没有 对应的 Layer。     
默认情况下，所有的 View 都是这个 layerType，这种情况下，这个 View 不会做任何的特殊处理。      

绘制过程：     
每帧: onDraw() → GPU渲染 → 显示      

#### LAYER_TYPE_SOFTWARE

表示这个 View 有一个软件实现的 Layer，使用一个 Bitmap 来缓冲。     

```
    private void buildDrawingCacheImpl(boolean autoScale) {
        ......
            try {
                bitmap = Bitmap.createBitmap(mResources.getDisplayMetrics(),
                        width, height, quality);
```

Software Layer 是对 Hardware Layer 的一个补充，如果 App 处于某种情况不能使用 Hardware Layer ，那么 Software Layer 就会派上用场 。 Hardware Layer 不支持的 API 的实现也得用 Software Layer 来实现。      

绘制过程：     
首帧: onDraw() --> 缓存到 Bitmap     
后续: 复用 Bitmap + 变换 --> 显示     

#### LAYER_TYPE_HARDWARE

标识这个 View 有一个硬件实现的 Layer，通常是OpenGL硬件上的帧缓冲对象或FBO（离屏渲染 Buffer）。       
这里 Hardware layerType 是依赖硬件加速的，如果硬件加速开启，那么才会有 FBO 或者帧缓冲 ； 如果硬件加速关闭，那么就算你设置一个 View 的 LayerType 是 Hardware Layer ，也会按照 Software Layer 去做处理。      
绘制过程：     
首帧: onDraw() --> 缓存到GPU纹理       
后续: 复用纹理 + 变换 --> 显示         
(invalidate时重新缓存)     

由于 Hardware Layer 的特性，属性动画( alpha \ translation \ scale \ rotation \ )过程中只更新 View 的 property，不会每一帧都去销毁和重建 FBO，其动画性能会有很大的提升。当然这里要注意属性动画的过程中( 比如 AnimationUpdate 回调中)，不要做除了上述属性更新之外的其他事情，比如添加删除子 View、修改 View 的显示内容等，这会使得 FBO 失效，性能反而变差。      
在某些情况下，实际上 Hardware Layer 可能要做非常多的工作，而不仅仅是渲染视图。缓存一个层需要花费时间，因为这一步要划分为两个过程：首先，视图渲染入 GPU 上的一个层中，然后，GPU 再渲染那个层到窗口，如果 View 的渲染十分简单（比如一个纯色），那么在初始化的时候设置 Hardware Layer 可能增加不必要的开销。     
对所有缓存来讲，存在一个缓存失效的可能性。动画运行时，如果某个地方调用了 `View.invalidate( )`，那么 Layer 就不得不重新渲染一遍。倘若不断地失效，你的 Hardware Layer 实际上要比不添加任何 Layer 性能更差(下面的例子可以佐证)，因为 Hardware Layer 在设置缓存的时候增加了开销。如果你不断的重缓存 Layer，会对性能造成极大地负担(做动画的 View 越复杂，带来的负担就越重)。      

## 使用场景分析

下面通过一个 Demo 来测试一下设置 View 不同的 Layer 在动画时对性能的影响。     
这里需要注意一下，由于动画组内设置了 Alpha 动画，如果不显性地设置一下 `textView.setLayerType(View.LAYER_TYPE_NONE, null);`，它还是会以 `LAYER_TYPE_HARDWARE` 的方式进行离屏渲染，如果去掉 alpha 动画，默认就是不会进行离屏渲染了。      
下面的代码是一个动画的场景来分别测试使用不同的 Layer 时对动画性能的影响。       
在动画开始前设置 layerType，在动画结束后，将 layerType 重新设置为 LAYER_TYPE_NONE , 设置回来的原因是 Hardware Layer 使用的是 Video Memory，设置为 NONE 之后这部分使用的内存将会回收 。        

```
    private void initView() {
        textView = findViewById(R.id.text_view);
        textView.setText("准备动画......");


        animatorSet = new AnimatorSet();
        objectAnimator1 = ObjectAnimator.ofFloat(textView, View.TRANSLATION_X, 350);
        objectAnimator2 = ObjectAnimator.ofFloat(textView, View.ALPHA, 0.6f);
        objectAnimator3 = ObjectAnimator.ofFloat(textView, View.TRANSLATION_Y, 150);
        objectAnimator4 = ObjectAnimator.ofFloat(textView, View.SCALE_X, 5);
        objectAnimator5 = ObjectAnimator.ofFloat(textView, View.SCALE_Y, 5);
        animatorSet.playTogether(objectAnimator1, objectAnimator2, objectAnimator3, objectAnimator4, objectAnimator5);
        animatorSet.setDuration(500);

        objectAnimator1.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animator) {
                //textView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
                //textView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
                //textView.setLayerType(View.LAYER_TYPE_NONE, null);
                count = 0;

            }

            @Override
            public void onAnimationEnd(Animator animator) {
                //textView.setLayerType(View.LAYER_TYPE_NONE, null);
            }

            @Override
            public void onAnimationCancel(Animator animator) {

            }

            @Override
            public void onAnimationRepeat(Animator animator) {

            }
        });

        objectAnimator1.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(@NonNull ValueAnimator valueAnimator) {
                textView.setText("这时第 "+String.valueOf(count++)+" 次更新");
            }
        });
    }
```

### 不动态更新 View + None Layer

<img src="/images/android-graphic-system-view-layer-type/none_layer_no_update.png" width="620" height="182"/>

### 不动态更新 View + Hardware Layer


<img src="/images/android-graphic-system-view-layer-type/hardware_no_update.png" width="638" height="96"/>

### 不动态更新 View + Software Layer

<img src="/images/android-graphic-system-view-layer-type/software_no_update.png" width="712" height="178"/>

### 动态更新 View + None Layer

<img src="/images/android-graphic-system-view-layer-type/none_layer_update.png" width="686" height="187"/>

### 动态更新 View + Hardware Layer

<img src="/images/android-graphic-system-view-layer-type/hardware_update.png" width="754" height="86"/>

### 动态更新 View + Software Layer

<img src="/images/android-graphic-system-view-layer-type/software_update.png" width="737" height="165"/>

## 离屏渲染的正确使用

Hardware Layer 对动画性能确实有很大的提升，但是如果你用不好，那么还不如不用，如果通过 Systrace 发现你做动画的时候每一帧都在 buildDrawingCache/SW(主线程) 或者 buildLayer(渲染线程)，那么请查看你的代码的逻辑。          

### 不正确使用 Software Layer 引起的性能问题 


### 不正确使用 Hardware Layer 引起的性能问题


在某一帧的绘制中发生了这么多的 Hardware Layer 重建，对内存和耗时都会有影响，应该着手去看是否可以优化一下：       

<img src="/images/android-graphic-system-view-layer-type/example_draw_layers.png" width="645" height="68"/>

### 正确使用方式

 - 如果只是单纯的做动画，不动态修改 View 的内容，那么性能表现为 ：Hardware Layer >= Software Layer > None Layer       
 - 如果做动画同时动态修改 View 的内容，那么性能表现为 ：Normal Layer > Software Layer = Hardware Layer      

## 性能分析方法

### Debug 工具

可以在 设置 - 辅助功能 - 开发者选项 - 显示硬件层更新（Show hardware layers updates） 这个工具来追踪硬件层更新导致的性能问题 。       
当 View 渲染 Hardware Layer 的时候整个界面会闪烁绿色，正常情况下，它应该在动画开始的时候闪烁一次（也就是 Layer 渲染初始化的时候），后续的动画不应该再有绿色出现；如果你的 View 在整个动画期间保持绿色不变，这就是持续的缓存失效问题了。      

### Systrace 分析

查看 Perfetto 时可以关注以下 Trace 标签：      
 - `buildDrawingCache/SW Layer for XXXView`：Software Layer 重建
 - `buildLayer`：Hardware Layer 重建
 - `flush layers`：
 - `flush commands`：RenderThread 中 GPU 命令提交耗时

drawLayer、 flush layers、 flush commands      

<img src="/images/android-graphic-system-view-layer-type/trace_tag.png" width="765" height="100"/>

## 开启离屏渲染的方式

除了通过 `View.setLayerType()` 来设置 View 的离屏缓冲外，还可以通过 `Canvas.saveLayer()` 来设置一个离屏缓冲。      

```
ViewPropertyAnimator.withLayer()

view.animate().alpha(0f).withEndAction {
    view.setLayerType(View.LAYER_TYPE_NONE, null)
}
```


## 相关文章

[Android 中的 Hardware Layer 详解](https://www.androidperformance.com/2019/07/27/Android-Hardware-Layer/)        
