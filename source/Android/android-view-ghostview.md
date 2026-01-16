---
title: Android View -- GhostView
categories: Android
comments: true
tags: [View]
description: 介绍 GhostView
date: 2017-8-26 10:00:00
---



## 概述

GhostView 


GhostView 的核心特性可以归纳为以下几点：

 - 非入侵性代理绘制：它的核心功能是在 Overlay 里面绘制另外一个 View（目标View），而且不会修改目标 View 在原来的父子层级关系。目标视图依然留在它原来的布局位置。    
 - 绘制位置的转移：目标视图本身不会被其父视图绘制（因为它被设为 INVISIBLE），但其视觉内容会通过 GhostView 的渲染节点，在 GhostView 自身所在的位置（通常是 Overlay 覆盖层）被绘制出来。
 - 可见性同步与互斥：GhostView 和目标视图的可见性状态是绑定且互斥的，形成一种“双控开关”机制：
  - 当 GhostView 可见 时，目标视图变为 不可见。
  - 当 GhostView 不可见 时，目标视图恢复为 可见。
  - 这种设计确保了同一时刻，同一个视觉内容只在一个地方被绘制，避免重复绘制或视觉错乱。
 - 主要应用场景：这种特性使其非常适合用于实现平滑的动画过渡。例如，在视图需要从一个布局位置移动到另一个布局位置的动画过程中，可以创建一个 GhostView 来“接管”原视图的绘制，并在覆盖层上进行动画。原视图本身可以变为不可见或进行其他布局操作，从而实现视觉上连续、但逻辑上分离的动画效果。

简单来说，GhostView 就像一个“视觉替身”或“全息投影”。它捕获一个视图的外观，并在另一个地方（Overlay上）显示出来，同时通过精妙的可见性控制，确保“本体”和“替身”不会同时出现。这是一个由系统内部使用的、用于支持高级视觉效果（特别是动画）的底层视图。    
目标View 只是在 GhostView 里面被绘制出来，这个绘制出来的视图不具备原视图的一些特性，比如：响应点击事件等。     
GhostView 是 View 的成员变量。    

## 实践

添加 GhostView ：    

```
    private void addGhostView() {
        mTargetView = findViewById(R.id.btn_3);

        ViewGroup decorView = (ViewGroup) getWindow().getDecorView();

        // 首先在 decorView 的overlay 中添加一个 ColorDrawable
        ColorDrawable colorDrawable = new ColorDrawable(0x880000ff);
        colorDrawable.setBounds(200, 600, 400, 400);
        decorView.getOverlay().add(colorDrawable);

        // 然后再添加 GhostView
        Matrix matrix = new Matrix();
        matrix.setTranslate(200, 600);
        mGhostView = GhostView.addGhost(mTargetView, (ViewGroup) decorView,matrix);

        // 强制使原视图和GhostView绘制的视图一起显示
        //mTargetView.setTransitionVisibility(View.VISIBLE);
        //((ViewGroup)(mTargetView.getParent())).invalidate();
    }
```


修改GhostView的可见性：    

```
    private void changeGhostViewVisibility() {
        if (mGhostView != null) {
            if (mGhostView.getVisibility() == View.VISIBLE) {
                mGhostView.setVisibility(View.INVISIBLE);
            } else {
                mGhostView.setVisibility(View.VISIBLE);
            }
            ((ViewGroup)(mTargetView.getParent())).invalidate();
        }
    }
```

删除 GhostView：    

```
    private void removeGhostView() {
        GhostView.removeGhost(mTargetView);
        mGhostView = null;
    }
```

在 addGhost之后强制设置目标 View 的setTransitionVisibility为View.VISIBLE，可以使原视图和GhostView绘制的视图一起显示。     

## 原理

```
GhostView.addGhost(View view, ViewGroup viewGroup, Matrix matrix)
  viewGroup.getOverlay()
  ViewGroupOverlay overlay = viewGroup.getOverlay()
  ViewOverlay.OverlayViewGroup overlayViewGroup = overlay.mOverlayViewGroup
  // 创建 GhostView
  ghostView = new GhostView(view)
    // 赋值给 GhostView 的成员变量 mView
    mView = view;
    // 将原视图设置为不可见
    mView.setTransitionVisibility(View.INVISIBLE)
  // 将变换矩阵应用于 ghostView
  ghostView.setMatrix(matrix);
  // 创建 GhostView 的parent
  FrameLayout parent = new FrameLayout(view.getContext());
  parent.addView(ghostView);
  // 将 GhostView 移动到顶层
  GhostView.moveGhostViewsToTop
  // 将 GhostView 添加到  Overlay
  GhostView.insertIntoOverlay(overlay.mOverlayViewGroup, parent, ghostView, tempViews, firstGhost)
    // 添加到 Overlay 的容器 mOverlayViewGroup
    viewGroup.addView(ghostView)
```

绘制原理：    
 - 使用硬件加速渲染方式（RecordingCanvas + RenderNode）把原视图的显示列表绘制到 GhostView 上。    
 - mView.updateDisplayListIfDirty() 会生成或更新原视图的 RenderNode，然后 GhostView 对其进行绘制。    

```
    protected void onDraw(Canvas canvas) {
        if (canvas instanceof RecordingCanvas) {
            RecordingCanvas dlCanvas = (RecordingCanvas) canvas;
            mView.mRecreateDisplayList = true;
            RenderNode renderNode = mView.updateDisplayListIfDirty();
            if (renderNode.hasDisplayList()) {
                dlCanvas.enableZ(); // enable shadow for this rendernode
                dlCanvas.drawRenderNode(renderNode);
                dlCanvas.disableZ(); // re-disable reordering/shadows
            }
        }
    }
```

可见性设置：    
 - 当 GhostView 变为可见 (visibility == 0, 即 View.VISIBLE) 时，会将原视图的过渡可见性设为 INVISIBLE(4)；
 - 当 GhostView 不可见时，则反过来把原视图的过渡可见性恢复为 VISIBLE(0)。
 - 这样可以保证在动画中，只看到 GhostView 显示，原视图不会与其冲突。

```
    public void setVisibility(@Visibility int visibility) {
        super.setVisibility(visibility);
        if (mView.mGhostView == this) {
            int inverseVisibility = (visibility == View.VISIBLE) ? View.INVISIBLE : View.VISIBLE;
            mView.setTransitionVisibility(inverseVisibility);
        }
    }
```



## 相关文章

[GhostView源码分析 ](https://juejin.cn/post/7470731561501425702)      
[GhostView](https://blog.csdn.net/weixin_38020796/article/details/64922899)    
[ Android Activity共享元素动画分析 ](https://juejin.cn/post/7144621475503276045#heading-5)     
