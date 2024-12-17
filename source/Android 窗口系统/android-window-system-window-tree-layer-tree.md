---
title: Android 窗口层级树-Layer树
categories: Android 窗口系统
comments: true
tags: [Android 窗口系统]
description: 介绍 Android 窗口层级树和Layer树的关系
date: 2022-11-23 10:00:00
---

## 概述

前面我们介绍了窗口层级树以及窗口的创建、布局、显示过程，但是我们知道，Android 真的的显示控制其实是在 SurfaceFlinger 层，在 SurfaceFlinger 也有一个类似的窗口树，只是我们叫它 Layer 树。    
下图是通过 Winscope 工具抓取的信息，可以看到在SurfaceFlinger层也有一个和窗口树应用的层级关系，以及当前图层对应的一些属性信息。    

<img src="/images/android-window-system-window-tree-layer-tree/layer.png" width="704" height="384"/>

有下面几个知识点：    

 - 触发创建Surface时就会触发创建出一个Layer，所以Surface和Layer是有着对应关系的，只不过在framework层侧重Surface，在SurfaceFlinger侧重Layer。    
 - 窗口层级树和Layer树基本上可以看做是一一对应，但也有不一致的情况，具体可以看下面“问题”这一章节。    
 - 在 WindowState 对应的图层下面还多了一级，多出来的这层右边的属性中有一项“activeBuffer  :  w:1080, h:2340, stride:268438272, format:1”。
 - 应用层只要有Surface，就可以将View的数据绘制保存到Surface中，也就可以显示到屏幕上    
 - Layer 有多种类型，有的Layer是有UI数据的，有的Layer是没有UI数据的。    
 - 根据 Layer 显示效果不同，可以分为 Layer(普通的窗口)，LayerDim（后面的窗口产生一个变暗的透明效果），LayerBlur（在LayerDim的基础上，背景会产生模糊的效果）    

## Surface 几种类型

在 SurfaceControl 中定义了下面几种 Surface 类型。    

```
    /**
     * Surface creation flag: Creates a normal surface.
     * This is the default.
     *
     * @hide
     */
    public static final int FX_SURFACE_NORMAL   = 0x00000000;

    /**
     * Surface creation flag: Creates a effect surface which
     * represents a solid color and or shadows.
     *
     * @hide
     */
    public static final int FX_SURFACE_EFFECT = 0x00020000;

    /**
     * Surface creation flag: Creates a container surface.
     * This surface will have no buffers and will only be used
     * as a container for other surfaces, or for its InputInfo.
     * @hide
     */
    public static final int FX_SURFACE_CONTAINER = 0x00080000;

    /**
     * @hide
     */
    public static final int FX_SURFACE_BLAST = 0x00040000;
```


FX_SURFACE_NORMAL，代表了一个标准Surface，这个是默认设置。    
FX_SURFACE_EFFECT，代表了一个有纯色或者阴影效果的Surface。    
FX_SURFACE_CONTAINER，代表了一个容器类Surface，这种Surface没有缓冲区，只是用来作为其他Surface的容器，或者是它自己的InputInfo的容器。     
FX_SURFACE_BLAST，结合下面的代码可以看到，FX_SURFACE_BLAST应该是等同于FX_SURFACE_NORMAL。     

```
            if ((mFlags & FX_SURFACE_MASK) == FX_SURFACE_NORMAL) {
                setBLASTLayer();
            }
```

`setContainerLayer()` 没有 UI 数据，不会以任何方式渲染，而是用作可渲染层的父级。     

```
        public Builder setBLASTLayer() {
            return setFlags(FX_SURFACE_BLAST, FX_SURFACE_MASK);
        }

        /**
         * Indicates whether a 'ContainerLayer' is to be constructed.
         *
         * Container layers will not be rendered in any fashion and instead are used
         * as a parent of renderable layers.
         *
         * @hide
         */
        public Builder setContainerLayer() {
            unsetBufferSize();
            return setFlags(FX_SURFACE_CONTAINER, FX_SURFACE_MASK);
        }
```

## Surface 创建

SurfaceFlinger没有这么复杂构建 Layer 树的逻辑，因为只要Framework创建一个“容器”类的同时也触发创建一个Surface，这样SurfaceFlinger层就也能同步构造出一个Layer（Surface）树。     
窗口层级树里面介绍的那些类都是容器，作为容器他们本身是没有UI数据的，真正有显示数据的就是带有Buffer的这层Layer。     
### DisplayContent的 Surface构建

```
SystemServer.main
    SystemServer.run
        SystemServer.startOtherServices
            ActivityManagerService.setWindowManager
                ActivityTaskManagerService.setWindowManager
                    RootWindowContainer.setWindowManager
                        DisplayContent.<init>
                            DisplayContent.configureSurfaces
```

```
    private void configureSurfaces(Transaction transaction) {
        final SurfaceControl.Builder b = mWmService.makeSurfaceBuilder(mSession)
                .setOpaque(true)
                // 设置 Container 类型
                .setContainerLayer()
                .setCallsite("DisplayContent");
        mSurfaceControl = b.setName(getName()).setContainerLayer().build();


    String getName() {
        return "Display " + mDisplayId + " name=\"" + mDisplayInfo.name + "\"";
    }
```

### 其他容器Surface的创建

窗口层级树里面其他各个容器创建的时候也会触发创建出对应的一个Surface，WindowContainer addChild 时候其实都会触发 WindowContainer.onParentChanged，从而触发 DisplayContent 中创建对应的 SurfaceControl，而 setContainerLayer() 创建出来的都是容器，而不是真正可以绘制的SurfaceControl。     
以 ActivityRecord 的创建为例，具体的调用链如下：     


```
    WindowContainer.addChild
        WindowContainer.setParent
            ActivityRecord.onParentChanged
                WindowContainer.onParentChanged
                    WindowToken.createSurfaceControl
                        WindowContainer.createSurfaceControl
                            WindowContainer.makeSurface()
                                WindowContainer.makeChildSurface
                                    ......
                                    DisplayContent.makeChildSurface
                                        WindowManagerService.makeSurfaceBuilder
                                            // 设置 container类型
                                            Builder.setContainerLayer()
                            WindowContainer.setInitialSurfaceControlProperties
                                SurfaceControl.Builder.build
                                    SurfaceControl()
                                        nativeCreate()
```


### 创建带 buffer 的 Surface

那么哪里才是真正的绘制画面的 SurfaceControl 呢？      
App 调用到 WMS 的 relayout 才可以获取可以绘制画面的 SurfaceControl，而且是 WMS 端创建好传递回去的。           


```
WindowManagerService.relayoutWindow
    //创建“Buff” 类型Surface
    WindowManagerService.createSurfaceControl
        WindowStateAnimator.createSurfaceLocked()
            new WindowSurfaceController
                SurfaceControl.Builder()
                Builder.setBLASTLayer()
                Builder.build()
    //给应用端Surface赋值
    WindowSurfaceController.getSurfaceControl
```

relayoutWindow 的 outSurfaceControl 参数就是 WMS 端写入数据，然后 app 端的 ViewRootImpl 来进行读取的。

```
    public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
            int lastSyncSeqId, ClientWindowFrames outFrames,
            MergedConfiguration outMergedConfiguration, SurfaceControl outSurfaceControl,
            InsetsState outInsetsState, InsetsSourceControl.Array outActiveControls,
            Bundle outSyncIdBundle) {
```

```
    WindowSurfaceController(String name, int format, int flags, WindowStateAnimator animator,
            int windowType) {
        mAnimator = animator;

        title = name;

        mService = animator.mService;
        final WindowState win = animator.mWin;
        mWindowType = windowType;
        mWindowSession = win.mSession;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "new SurfaceControl");
        final SurfaceControl.Builder b = win.makeSurface()
                //挂载到 WindowState 下面
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(METADATA_WINDOW_TYPE, windowType)
                .setMetadata(METADATA_OWNER_UID, mWindowSession.mUid)
                .setMetadata(METADATA_OWNER_PID, mWindowSession.mPid)
                .setCallsite("WindowSurfaceController");

        final boolean useBLAST = mService.mUseBLAST && ((win.getAttrs().privateFlags
                & WindowManager.LayoutParams.PRIVATE_FLAG_USE_BLAST) != 0);

        if (useBLAST) {
            //设置为“Buff”图层
            b.setBLASTLayer();
        }

        mSurfaceControl = b.build();

        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
```

所以我们才会在上面的 Winscope 抓取的Layer层级里面看到 WindowState 下面多出的一层 Layer，就是我们上面创建的这层。      

## 挂载 Surface

也就是把前面创建的 Surface 组成 Surface 树。         
主要通过 WindowContainer 的 makeChildSurface 来完成。        

```
    Builder makeSurface() {
        final WindowContainer p = getParent();
        // 传递当前容器过去
        return p.makeChildSurface(this);
    }

    /**
     * @param child The WindowContainer this child surface is for, or null if the Surface
     *              is not assosciated with a WindowContainer (e.g. a surface used for Dimming).
     */
    Builder makeChildSurface(WindowContainer child) {
        // 拿到父容器，再调用父容器的 makeChildSurface
        final WindowContainer p = getParent();
        return p.makeChildSurface(child)
                //走完DisplayContent.makeChildSurface后然后层层设置 Parent，
                //相当于层层移动该 SurfaceControl，直到移动到正确位置
                .setParent(mSurfaceControl);
    }
```

最后递归调用到了 DisplayContent.makeChildSurface     

```
    SurfaceControl.Builder makeChildSurface(WindowContainer child) {
        SurfaceSession s = child != null ? child.getSession() : getSession();
        // 根据当前容器创建一个容器类型 Surface的Builder
        final SurfaceControl.Builder b = mWmService.makeSurfaceBuilder(s).setContainerLayer();
        if (child == null) {
            return b;
        }

        // 设置名称
        return b.setName(child.getName())
        // 这里先设置为 DisplayContent 对应的 mSurfaceControl，后面层层递归调用会经过多次逐步重新设置，然后设置为正确的parent。
                .setParent(mSurfaceControl);
    }
```


## Surface 返回给应用端

应用端View的绘制信息都是保存到Surface上的，因为必定要有一个"Buff"类型的Surface，也就是上面流程中创建的这个 Surface。     
应用端的 ViewRootImpl 触发 WMS 的 relayoutWindow 会传递一个出参 ：outSurfaceControl过来， 现在WMS会通过以下方法将刚刚创建好是Surface传递到应用端。     
这样一来应用端就有了可以保持绘制数据的Surface，然后就可以执行 View.draw。     


```
WindowSurfaceController.java

    void getSurfaceControl(SurfaceControl outSurfaceControl) {
        outSurfaceControl.copyFrom(mSurfaceControl, "WindowSurfaceController.getSurfaceControl");
    }
```

## 创建 Layer

接下来根据 Surface 在 SurfaceFlinger 端来创建对应的 Layer。     

```
system_server进程
SurfaceControl.init()
    android_view_SurfaceControl::nativeCreate
        SurfaceComposerClient::createSurfaceChecked
            Client::createSurface
            -------跨进程调用到 SurfaceFlinger
SurfaceFlinger::createLayer
    // 根据不同的surface创建不同的 Layer
    createBufferStateLayer
    createEffectLayer
    createContainerLayer
    SurfaceFlinger::addClientLayer
```

然后可以根据 `adb shell dumpsys SurfaceFlinger` 可以看到这样的 Layer 信息：     

SurfaceFlinger 端的 Layer 情况，可以看到，基本和 WMS 窗口层级树是对应关系的。      

```
Visible layers (count = 108)
Composition layers
* Layer 0xb400007783e854d0 (Light Sensor Capture#3)
      ...... 
* Layer 0xb400007783e7f9b0 (Display 0 name="内置屏幕"#4)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e2c810, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb400007783eaa520 (WindowedMagnification:0:31#5)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e25570, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb400007783eb88f0 (HideDisplayCutout:0:14#6)
      ......
* Layer 0xb400007783e9eee0 (OneHanded:0:14#7)
      ......
* Layer 0xb400007783eb2dd0 (FullscreenMagnification:0:12#8)
      ...... 
* Layer 0xb400007783ebb680 (Leaf:0:1#9)
      ...... 
......
* Layer 0xb400007783fcfa10 (Task=13#144)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e6bf50, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb400007784008950 (94bb650 ActivityRecordInputSink com.hq.android.androiddemo/.MainActivity#154)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e63c50, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb400007783fddde0 (ActivityRecord{9bde77c u0 com.hq.android.androiddemo/.MainActivity t13}#145)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e70900, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb40000778400e470 (3812819 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity#155)
      isSecure=false geomUsesSourceCrop=false geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 -1 -1] geomContentCrop=[0 0 -1 -1] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e6e010, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 0.000000 0.000000]       shadowRadius=0.000000 
      blend=INVALID (0) alpha=1.000000 backgroundBlurRadius=0 composition type=INVALID (0) 
      buffer: buffer=0x0 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=UNKNOWN (0) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false 
* Layer 0xb400007783fec1b0 (com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity#156)
      isSecure=false geomUsesSourceCrop=true geomBufferUsesDisplayInverseTransform=false geomLayerTransform (ROT_0) (IDENTITY)

      geomBufferSize=[0 0 1080 2340] geomContentCrop=[0 0 1080 2340] geomCrop=[0 0 -1 -1] geomBufferTransform=0 
        Region transparentRegionHint (this=0xb4000075e3e68e30, count=1)
    [  0,   0,   0,   0]
      geomLayerBounds=[0.000000 0.000000 1080.000000 2340.000000]       shadowRadius=0.000000 
      blend=NONE (1) alpha=1.000000 backgroundBlurRadius=0 composition type=DEVICE (2) 
      buffer: buffer=0xb4000076a3e36470 
      sideband stream=0x0 
      color=[0.000000 0.000000 0.000000] 
      isOpaque=true hasProtectedContent=false isColorspaceAgnostic=false dataspace=V0_SRGB (142671872) hdr metadata types=0 dimming enabled=true colorTransform=[[1.000,0.000,0.000,0.000][0.000,1.000,0.000,0.000][0.000,0.000,1.000,0.000][0.000,0.000,0.000,1.000]] caching hint=Enabled 
      internalOnly=false
```

其中，androiddemo 这个应用的对应的窗口层级为：    

```
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
        #2 Task=13 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
         #0 ActivityRecord{9bde77c u0 com.hq.android.androiddemo/.MainActivity t13} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
          #0 3812819 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2340]
```

可以和 sf 的 Layer 对比一下看看。      
有些 Layer 是不参与绘制的，有的参与绘制，参与绘制的这些 Layer 的 buffer是不为空的：
```
      blend=NONE (1) alpha=1.000000 backgroundBlurRadius=0 composition type=DEVICE (2) 
      buffer: buffer=0xb4000076a3e36470 
```

hw blend mode 有这几种：

```
/* Blend modes, settable per layer */
typedef enum {
    HWC2_BLEND_MODE_INVALID = 0,

    /* colorOut = colorSrc */
    HWC2_BLEND_MODE_NONE = 1,

    /* colorOut = colorSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_PREMULTIPLIED = 2,

    /* colorOut = colorSrc * alphaSrc + colorDst * (1 - alphaSrc) */
    HWC2_BLEND_MODE_COVERAGE = 3,
} hwc2_blend_mode_t;
```

Display *****(active) HWC layers:
表示真正参与合成的 layer。

```
Display 4630947064936706947 (active) HWC layers:
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Layer name
           Z |  Window Type |  Layer Class | Comp Type |  Transform |   Disp Frame (LTRB) |          Source Crop (LTRB) |     Frame Rate (Explicit) (Seamlessness) [Focused]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 com.hq.android.androiddemo/com.hq.android.androiddemo.MainActivity#156
  rel      0 |            1 |            0 |     DEVICE |          0 |    0    0 1080 2340 |    0.0    0.0 1080.0 2340.0 |                                              [*]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 StatusBar#104
  rel      0 |         2000 |            0 |     DEVICE |          0 |    0    0 1080   92 |    0.0    0.0 1080.0   92.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 NavigationBar0#100
  rel      0 |         2019 |            0 |     DEVICE |          0 |    0 2271 1080 2340 |    0.0    0.0 1080.0   69.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlay#77
  rel      0 |         2024 |            0 |     DEVICE |          0 |    0    0 1080  144 |    0.0    0.0 1080.0  144.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 ScreenDecorOverlayBottom#81
  rel      0 |         2024 |            0 |     DEVICE |          0 |    0 2196 1080 2340 |    0.0    0.0 1080.0  144.0 |                                              [ ]
---------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## 问题

窗口层级树和 sf 的Layer 树的层级一定是一样的吗？要想回答这个问题可以看看这篇文章[android12/13/14版本wms最新面试题：dumpsys window和sf一定会一致么](https://www.bilibili.com/read/cv39212132/)     
