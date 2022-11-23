---
title: Android 图形系统 -- 实现毛玻璃的几种方法
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 实现毛玻璃的几种方法
date: 2022-11-23 10:00:00
---


在Android开发中，我们一般需要用到模糊的地方有下面两种场景：

 - Window 级模糊：将某个半透明 Window 的背景内容进行模糊处理，常见于通知栏下拉之类的场景。
 - View 级模糊：在某一个App页面内，某控件的背景内容进行模糊处理。或者对某个View内容进行模糊。

下面介绍几种实现毛玻璃的方法。    

## 使用 RenderScript

[官方文档](https://developer.android.google.cn/guide/topics/renderscript/compute)

RenderScript 是 Android 自带一个高效的计算框架，能够自动利用 CPU、GPU、DSP 来做并行计算，能在处理图片、数学模型计算等场景提供高效的计算能力。RenderScriptIntrinsics提供了一些可以帮助我们快速实现各种图片处理的操作类，例如，ScriptIntrinsicBlur，可以简单高效实现高斯模糊效果。

Android 3.0(api 11)以后支持 RenderScript， 4.2(api 17) 支持 ScriptIntrinsicBlur。
从 Android 12 开始，RenderScript API 已被弃用。建议 对 RenderScript 进行迁移至 Toolkit.blur。 [迁移文档](https://developer.android.google.cn/guide/topics/renderscript/migrate)    
如果以 Android 12（API 级别 31）及更高版本为目标平台，请考虑使用 RenderEffect 类（而非 Toolkit.blur()）。    

```
    private Drawable getDrawable(Bitmap bitmap) {
        int width = bitmap.getWidth();
        int height = bitmap.getHeight();
        // 可以设置图片缩放，可以减少内存，也可以调节图片模糊度，
        Matrix matrix = new Matrix();
        matrix.postScale(0.5f, 0.5f);
        bitmap = Bitmap.createBitmap(bitmap, 0, 0, width, height, matrix, false);

        // 如果图片类型是 Bitmap.Config.HARDWARE 这里需要copy成 ARGB_8888，ScriptIntrinsicBlur 对 HARDWARE 类型图片不支持
        Bitmap inputBitmap = bitmap.copy(Bitmap.Config.ARGB_8888, true);

        RenderScript rs = RenderScript.create(this);
        ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
        Allocation tmpIn = Allocation.createFromBitmap(rs, inputBitmap);
        Allocation tmpOut = Allocation.createTyped(rs,tmpIn.getType());
        // 设置渲染的模糊程度, 25是最大模糊度
        blurScript.setRadius(25);
        blurScript.setInput(tmpIn);
        blurScript.forEach(tmpOut);
        tmpOut.copyTo(inputBitmap);
        BitmapDrawable bitmapDrawable = new BitmapDrawable(inputBitmap);
        // 可以设置 ColorFilter，使图片的颜色暗一些
        bitmapDrawable.setColorFilter(0x66000000, PorterDuff.Mode.SRC_ATOP);
        return bitmapDrawable;
    }
```

## OpenGL处理

参考这篇博客[OpenGL ES -- 图像处理 ](http://www.heqiangfly.com/2020/05/06/opengl-es-image-effect/)

## RenderEffect

可以使用 RenderEffect 来实现 View 级别模糊。    

Android 12 版本添加的高斯模糊接口：

 - createBitmapEffect
 - createBlendModeEffect
 - createBlurEffect
 - createChainEffect
 - createColorFilterEffect
 - createOffsetEffect
 - createRuntimeShaderEffect
 - createShaderEffect

我们可以使用 View.setRenderEffect() 方法来对一个View实现模糊效果。setRenderEffect 方法只能对View本身做模糊，不能对 View 底下的内容产生影响。    
另外，还可以使用 RenderNode.setRenderEffect() 方法将模糊、色彩滤镜等效果应用于 RenderNode。

## View.getViewRootImpl().createBackgroundBlurDrawable()

Android 12 开始，系统内置了一个 com.android.internal.graphics.drawable.BackgroundBlurDrawable，给当前的 View 设置此背景后，可使自身区域变为毛玻璃区域，对当前窗口以下的图层进行跨图层模糊。提供了下面的功能：：

 - 设置圆角值
 - 设置透明度
 - 设置模糊程度
 - 设置前景叠加色（如半透明的白色）

可以通过 View.getViewRootImpl().createBackgroundBlurDrawable() 来获取BackgroundBlurDrawable，设置给 View 后实现窗口层级的模糊。

```
view.addOnAttachStateChangeListener(new OnAttachStateChangeListener() {
    @Override
    public void onViewAttachedToWindow(@NonNull View v) {
        BackgroundBlurDrawable drawable = v.getViewRootImpl().createBackgroundBlurDrawable();
        v.setBackground(drawable);
    }

    @Override
    public void onViewDetachedFromWindow(@NonNull View v) {

    }
});
```

## Window.setBackgroundBlurRadius()

 Android 12 中 Window 提供了处理跨窗口模糊效果的方法。我们可以使用 setBackgroundBlurRadius() 实现窗口模糊处理效果（例如背景模糊和背后模糊）。
 窗口模糊或交叉窗口模糊用于模糊给定窗口后面的屏幕。窗口模糊有两种类型，可以用来实现不同的视觉效果：
  - 背景模糊允许您创建背景模糊的窗口，从而创建磨砂玻璃效果。
  - Blur behind允许您模糊（对话框）窗口后面的整个屏幕，创建景深效果。

两种效果可以单独使用也可以组合使用，如下图所示： 
 具体可以参考[官方文档](https://source.android.google.cn/docs/core/display/window-blurs)。

## SurfaceControl.Transaction.setBackgroundBlurRadius()

参考 SystemUI 中 NotificationShadeDepthController 实现窗口级毛玻璃的方法。    
具体实现在 BlurUtils.applyBlur。    

```
    open fun createTransaction(): SurfaceControl.Transaction {
        return SurfaceControl.Transaction()
    }
    
    
    fun applyBlur(viewRootImpl: ViewRootImpl?, radius: Int, opaque: Boolean) {
        if (viewRootImpl == null || !viewRootImpl.surfaceControl.isValid) {
            return
        }
        createTransaction().use {
            if (supportsBlursOnWindows()) {
                it.setBackgroundBlurRadius(viewRootImpl.surfaceControl, radius)
                if (lastAppliedBlur == 0 && radius != 0) {
                    it.setEarlyWakeupStart()
                }
                if (lastAppliedBlur != 0 && radius == 0) {
                    it.setEarlyWakeupEnd()
                }
                lastAppliedBlur = radius
            }
            it.setOpaque(viewRootImpl.surfaceControl, opaque)
            it.apply()
        }
    }
```

```
        SurfaceControl.Transaction transaction = new SurfaceControl.Transaction();
        transaction.setBackgroundBlurRadius(view.getViewRootImpl().getSurfaceControl(), radus).apply();
```

## 相关链接

https://mp.weixin.qq.com/s/9-D5wlMnklThhhR2nqs-pA
https://www.jianshu.com/p/433d58163e73
https://source.android.com/docs/core/display/window-blurs
