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

绘制过程：    
首帧: onDraw() --> 缓存到 Bitmap     
后续: 复用 Bitmap + 变换 --> 显示     

#### LAYER_TYPE_HARDWARE

使用GPU来缓冲
绘制过程：     
首帧: onDraw() --> 缓存到GPU纹理       
后续: 复用纹理 + 变换 --> 显示         
(invalidate时重新缓存)     

## 使用场景分析

下面通过一个 Demo 来测试一下设置 View 不同的 Layer 在动画时对性能的影响。     
这里需要注意一下，由于动画组内设置了 Alpha 动画，如果不显性地设置一下 `textView.setLayerType(View.LAYER_TYPE_NONE, null);`，它还是会以 LAYER_TYPE_HARDWARE 的方式进行离屏渲染，如果去掉 alpha 动画，默认就是不会进行离屏渲染了。      

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

### 不正确使用 Software Layer 引起的性能问题 


### 不正确使用 Hardware Layer 引起的性能问题


在某一帧的绘制中发生了这么多的 Hardware Layer 重建，对内存和耗时都会有影响，应该着手去看是否可以优化一下：

<img src="/images/android-graphic-system-view-layer-type/example_draw_layers.png" width="645" height="68"/>


## 性能分析方法

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

## 相关文章

[Android 中的 Hardware Layer 详解](https://www.androidperformance.com/2019/07/27/Android-Hardware-Layer/)
