---
title: Android显示原理
urlname: android_display_principle
date: 2023/01/06
tags:
  - android
  - graphic
---

> 相关的类的定义和功能可以查看 [Android 显示原理(手册)](/android/android_display_principle_manual)

显示流程, 表面意思就是在 android 系统中, 是如何将画面显示在手机上的整体流程.

android 系统的显示流水线,主要包含三个阶段, 绘制、合成和显示

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/gKItXw.jpg)

# 绘制

绘制阶段包括了窗口的概念, 涉及到 AMS, WMS, DMS, 大致流程可以分为下面几部分

1. 应用层创建 Activity, 通过 AMS 启动 Activity
2. 创建 Activity 时, 通过我们设置的 setContentView 附属到 DectorView 中,通过 WMS 创建对应的 Window，这个过程会创建一系列的对象
   - Activity
   - ViewRootImpl
   - Window
   - Surface
   - Layer
3. 窗口
4. 绘制

绘制阶段是比较长的一个阶段, 涉及到的模块也很多, 大部分在 framework 的 JAVA 层中, 我们做应用开发时经常接触到的就是 Activity、Window 以及 View, 涉及到 framework 中的服务就是 AMS、WMS、DMS 等, 你可能经常听到窗口, 一般口头上我们说到窗口或者窗口管理的时候,都是指所有的窗口,比如 Activity、dialog 等, Activity 委托 View 负责显示, 三者的关系可以简单理解为, view 组成 Activity, Activity 在窗口模块中被抽象为 Window

View 主要有三个方法, Measure、Layout 和 Draw, Draw 的时候才会开始真正分配 buffer

在 View 中有一个比较重要的 DectorView, 它是 Activity 的根 View, 窗口的添加就是通过 WindowManager.addView()把该 mDecorView 添加到 WindowManager 中, 所以一个 Activity 对应一个窗口, 当然你也可以直接通过 WindowManager 添加一个单独的 view, 这样也是一个窗口

## 控制端

WindowManagerService (服务端, 与绘制无关)

- 窗口的状态、属性, 如大小, 位置; (WindowState, 与上面的 DectorView 一一对应)
- View 增加、删除、更新
- 窗口顺序
- Input Event 消息收集和处理等(与绘制无关, 可不用关心)

SurfaceFlinger(服务端, 与绘制无关)

- Layer 的大小、位置(Layer 与上面的 WindowState 一一对应)
- Layer 的增加、删除、更新;
- Layer 的 zorder 顺序

## 内容端(绘制)

framework/base: Canvas

- SoftwareCanvas (skia/CPU)
- HardwareCanvas (hwui/GPU)

framework/base: Surface

- 区别于 WMS 的 Surface 概念, 与 Canvas 一一对应, 内容生产者

framework/native:

- Surface: 负责分配 buffer 与填充(由上面的 Surface 传下来)
- SurfaceFlinger
  - Layer 数据已填充好, 与上面提到的 Surface 同样是一一对应
  - 可见 Layer 这个概念即是控制端又是内容端, SF 更重要的是合成

## 概念梳理

> Window -> DecorView-> ViewRootImpl -> WindowState -> Surface -> Layer 是一一对应的
> 一般的 Activity 包括的多个 View 会组成 View hierachy 的树形结构, 只有最顶层的 DecorView, 也就是根结点视图, 才是对 WMS 可见的, 即有对应的 Window 和 WindowState.
> 一个应用程序窗口分别位于应用程序进程和 WMS 服务中的两个 Surface 对象有什么区别呢？
> 虽然它们都是用来操作位于 SurfaceFlinger 服务中的同一个 Layer 对象的, 不过, 它们的操作方式却不一样. 具体来说, 就是位于应用程序进程这一侧的 Surface 对象负责绘制应用程序窗口的 UI, 即往应用程序窗口的图形缓冲区填充 UI 数据, 而位于 WMS 服务这一侧的 Surface 对象负责设置应用程序窗口的属性, 例如位置、大小等属性.
> 这两种不同的操作方式分别是通过 C++层的 Surface 对象和 SurfaceControl 对象来完成的, 因此, 位于应用程序进程和 WMS 服务中的两个 Surface 对象的用法是有区别的. 之所以会有这样的区别, 是因为绘制应用程序窗口是独立的, 由应用程序进程来完即可, 而设置应用程序窗口的属性却需要全局考虑, 即需要由 WMS 服务来统筹安排.

## 软件绘制

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/hcrZSM.jpg)

在 Android 应用程序进程这一侧, 每一个窗口都关联有一个 Surface. 每当窗口需要绘制 UI 时, 就会调用其关联的 Surface 的成员函数 lock 获得一个 Canvas, 其本质上是向 SurfaceFlinger 服务 dequeue 一个 Graphic Buffer.
Canvas 封装了由 Skia 提供的 2D UI 绘制接口, 并且都是在前面获得的 Graphic Buffer 上面进行绘制的, 这个 Canvas 的底层是一个 bitmap, 也就是说, 绘制都发生在这个 Bitmap 上. 绘制完成之后, Android 应用程序进程再调用前面获得的 Canvas 的成员函数 unlockAndPost 请求显示在屏幕中, 其本质上是向 SurfaceFlinger 服务 queue 一个 Graphic Buffer, 以便 SurfaceFlinger 服务可以对 Graphic Buffer 的内容进行合成, 以及显示到屏幕上去.

### surface.lockCanvas():

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/xAoqE1.jpg)

```cpp
//android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

    ......

    ANativeWindow_Buffer outBuffer;
    //调用了Surface的dequeueBuffer, 从SurfaceFlinger中申请内存GraphicBuffer,这个buffer是用来传递绘制的元数据的
    status_t err = surface->lock(&outBuffer, dirtyRectPtr);
    if (err < 0) {
        const char* const exception = (err == NO_MEMORY) ?
                OutOfResourcesException :
                "java/lang/IllegalArgumentException";
        jniThrowException(env, exception, NULL);
        return 0;
    }

    SkImageInfo info = SkImageInfo::Make(outBuffer.width, outBuffer.height,
                                         convertPixelFormat(outBuffer.format),
                                         outBuffer.format == PIXEL_FORMAT_RGBX_8888
                                                 ? kOpaque_SkAlphaType : kPremul_SkAlphaType);
    //新建了一个SkBitmap, 并进行了一系列设置
    SkBitmap bitmap;
    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
    bitmap.setInfo(info, bpr);
    if (outBuffer.width > 0 && outBuffer.height > 0) {
        //bitmap对graphicBuffer进行关联
        bitmap.setPixels(outBuffer.bits);
    } else {
        // be safe with an empty bitmap.
        bitmap.setPixels(NULL);
    }
    //构造一个native的Canvas对象（SKCanvas), 再返回这个Canvas对象, java层的Canvas对象其实只是对SKCanvas对象的一个简单包装, 所有绘制方法都是转交给SKCanvas来做.
    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
    //bitmap对下关联了获取的内存buffer, 对上关联了Canvas,把这个bitmap放入Canvas中
    nativeCanvas->setBitmap(bitmap);

    ......

    sp<Surface> lockedSurface(surface);
    lockedSurface->incStrong(&sRefBaseOwner);
    return (jlong) lockedSurface.get();
}
```

### canvas.drawXXX

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/PD7DnM.jpg)

### Skia 深入分析

SkCanvas 是按照 SkBitmap 的方法去关联 GraphicBuffer

一、渲染层级 从渲染流程上分, Skia 可分为如下三个层级：

指令层：SkPicture、SkDeferredCanvas->SkCanvas
这一层决定需要执行哪些绘图操作, 绘图操作的预变换矩阵, 当前裁剪区域, 绘图操作产生在哪些 layer 上, Layer 的生成与合并.

解析层：SkBitmapDevice->SkDraw->SkScan、SkDraw1Glyph::Proc
这一层决定绘制方式, 完成坐标变换, 解析出需要绘制的形体（点/线/规整矩形）并做好抗锯齿处理, 进行相关资源解析并设置好 Shader.

渲染层：SkBlitter->SkBlitRow::Proc、SkShader::shadeSpan 等
这一层进行采样（如果需要）, 产生实际的绘制效果, 完成颜色格式适配, 进行透明度混合和抖动处理（如果需要）.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/jNcy1G.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ULfbn2.jpg)

### mSurface.unlockCanvasAndPost(canvas):

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/zcX55c.jpg)

最后一个步骤会通知消费者的 onFrameAvailable 接口, 进一步调用 SurfaceFlinger 的 signalLayerUpdate 发起更新操作

## HardwareRenderer 硬件绘制

具体参考:https://www.jianshu.com/p/40f660e17a73

CPU 更擅长复杂逻辑控制, 而 GPU 得益于大量 ALU 和并行结构设计, 更擅长数学运算.
页面由各种基础元素（DisplayList）构成, 渲染时需要进行大量浮点运算.
硬件加速条件下, CPU 用于控制复杂绘制逻辑, 构建或更新 DisplayList；GPU 用于完成图形计算, 渲染 DisplayList.
硬件加速条件下, 刷新界面尤其是播放动画时, CPU 只重建或更新必要的 DisplayList, 进一步提高渲染效率.

开启硬件绘制条件是在 ViewRootImpl 的 draw 中

```java
private void draw(boolean fullRedrawNeeded) {
    ...
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        //关键点1 是否开启硬件加速
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
             ...
            dirty.setEmpty();
            mBlockResizeBuffer = false;
            //关键点2 硬件加速绘制
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
          ...
           //关键点3 软件绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }
}
```

硬件加速渲染和软件渲染一样, 在开始渲染之前, 都是要先向 SurfaceFlinger 服务 dequeue 一个 Graphic Buffer. 不过对硬件加速渲染来说, 这个 Graphic Buffer 会被封装成一个 ANativeWindow, 并且传递给 OpenGL 进行硬件加速渲染环境初始化.
在 Android 系统中, ANativeWindow 和 Surface 可以是认为等价的, 只不过是 ANativeWindow 常用于 Native 层中, 而 Surface 常用于 Java 层中. OpenGL 获得了一个 ANativeWindow, 并且进行了硬件加速渲染环境初始化工作之后, Android 应用程序就可以调用 OpenGL 提供的 API 进行 UI 绘制了, 绘制出来内容就保存在前面获得的 Graphic Buffer 中.
当绘制完毕, Android 应用程序再调用 libegl 库(一般由第三方提供)的 eglSwapBuffer 接口请求将绘制好的 UI 显示到屏幕中, 其本质上与软件渲染过程是一样的.

硬件加速绘制包括两个阶段：构建阶段 + 绘制阶段, 所谓构建就是递归遍历所有视图, 将需要的操作缓存下来, 之后再交给单独的 Render 线程利用 OpenGL 渲染. 在 Android 硬件加速框架中, View 视图被抽象成 RenderNode 节点, View 中的绘制都会被抽象成一个个 DrawOp（DisplayListOp）, 比如 View 中 drawLine, 构建中就会被抽象成一个 DrawLintOp, drawBitmap 操作会被抽象成 DrawBitmapOp, 每个子 View 的绘制被抽象成 DrawRenderNodeOp, 每个 DrawOp 有对应的 OpenGL 绘制命令, 同时内部也握着绘图所需要的数据. 如下所示：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ZqBcV5.jpg)

如此以来, 每个 View 不仅仅握有自己 DrawOp List, 同时还拿着子 View 的绘制入口, 如此递归, 便能够统计到所有的绘制 Op, 很多分析都称为 Display List, 源码中也是这么来命名类的, 不过这里其实更像是一个树, 而不仅仅是 List, 示意如下：

![硬件加速](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/EbcdXD.jpg)

构建完成后, 就可以将这个绘图 Op 树交给 Render 线程进行绘制, 这里是同软件绘制很不同的地方, 软件绘制时, View 一般都在主线程中完成绘制, 而硬件加速, 除非特殊要求, 一般都是在单独线程中完成绘制, 如此以来就分担了主线程很多压力, 提高了 UI 线程的响应速度.

![硬件加速模型](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/BZSaMu.jpg)

引进 Display List 的概念有什么好处呢？主要是两个好处。
第一个好处是在下一帧绘制中，如果一个 View 的内容不需要更新，那么就不用重建它的 Display List，也就是不需要调用它的 onDraw 成员函数。
第二个好处是在下一帧中，如果一个 View 仅仅是一些简单的属性发生变化，例如位置和 Alpha 值发生变化，那么也无需要重建它的 Display List，只需要在上一次建立的 Display List 中修改一下对应的属性就可以了，这也意味着不需要调用它的 onDraw 成员函数。这两个好处使用在绘制应用程序窗口的一帧时，省去很多应用程序代码的执行，也就是大大地节省了 CPU 的执行时间。

## BufferQueue

和 BufferQueue 有关的几个类分别是:

1.  BufferBufferCore：BufferQueue 的实际实现
2.  BufferSlot：用来存储 GraphicBuffer, 在 BufferQueue 的成员变量中, mSlots ：BufferSlot 数组, 默认大小是 64 个, BufferQueueProducer::mSlots 和 BufferQueueConsumer::mSlots 是 BufferQueueCore::mSlots 的映射/引用, 其实就是一个东西
3.  BufferState：表示 GraphicBuffer 的状态
4.  mQueue：BufferQueue 里的一个 BufferItem 类型的数组, Producer 调用 queueBuffer 后, 其实就是 queue 到这个数组里面,队列数量一般 2 个或者三个
5.  BufferItem 队列：具体双缓冲或者三缓冲的实现；
6.  IGraphicBufferProducer：BufferQueue 的生产者接口, 实现类是 BufferQueueProducer
7.  IGraphicBufferConsumer：BufferQueue 的消费者接口, 实现类是 BufferQueueConsumer
8.  GraphicBuffer：表示一个 Buffer, 可以填充图像数据,代表一帧
9.  ANativeWindow_Buffer：GraphicBuffer 的父类
10. ConsumerBase：实现了 ConsumerListener 接口, 在数据入队列时会被调用到, 用来通知消费者

BufferQueue 是生产者和消费者的桥梁:

1.  Producer 准备绘图时, 调用 dequeueBuffer 向 BufferQueue 请求一块 buffer;
2.  BufferQueue 收到 dequeueBuffer 的请求后会去自己的 BufferSlot 数组中寻找一个 FREE 状态的, 然后返回它的 index;
3.  Producer 拿到可用的 buffer 后就可以准备填充数据了；
4.  Producer 填充数据完毕, 调用 queueBuffer 将 bffer 再返还给 BufferQueue;
5.  BufferQueue 收到 queueBuffer 请求后, 将指定的 buffer 包装为 BufferItem 对象, 放入 mQueue, 并通知 Consumer 来消费；
6.  Consumer 收到 onFrameAvailable 的通知后, 调用 acquireBuffer, 获取一个 buffer;
7.  Consumer 拿到 buffer 进行消费, 一般是指 SurfaceFlinger 做合成以及显示;
8.  Consumer 完成处理, 调用 releaseBuffer 把 buffer 返还给 BufferQueue, 这个 buffer 之后就可以再次在新的循环中使用了.

BufferQueue 收到 queueBuffer 请求后, 将指定的 buffer 包装为 BufferItem 对象, 放入 mQueue, 并通知 Consumer 来消费;

BufferQueueLayer 接收到回调 onFrameAvailable 触发 mFlinger->signalLayerUpdate()

signalLayerUpdate 会 requestNextVsync 请求 vsync 信号, 触发 INVALIDATE 消息

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/202211031429327.jpg)

先处理 handleMessageTransaction, handleMessageTransaction 主要处理 Layer 属性变化, 显示设备变化等情况, 最终将变化的信息 mCurrentState 提交到 mDrawingState, 等待合成处理.

再调用 handleMessageInvalidate, 有 BufferQueue 过来, 但是还没有到显示时间(mLayersWithQueuedFrames 为空), 或者没有获取到 Buffer, 则重新触发一次更新(signalLayerUpdate)

触发 latchBuffer, 调用 acquireBuffer, 获取一个 buffer;

取出 GraphicBuffer 后通过事务 Transaction 来向 SurfaceFlinger 提交 Buffer 与图层的属性并且触发 REFRESH 事件

# 合成

## SurfacFlinger 中的服务

SurfacFlinger 进程中主要 4 个服务:

- startHidlServices 主要是启动 allocator
- DisplayService, 主要负责 DisplayEvent 的处理
- SurfaceFlinger, 主要显示相关的, 最重要的服务
- GpuService, GPU 相关的服务, 为访问 GPU

在 init 函数中, 主要做了一下几件事:

- 创建两个 EventThread 及其 DispSyncSource, 主要是用以处理和分发如 Vsync; App 和 SurfaceFlinger 的如 Vsync 是分开的, 不是同一个.
- 创建了自己的消息队列 mEventQueue, SurfaceFlinger 的消息队列
- 初始化了 Client 合成模式(GPU)合成时, 需要用到的 RenderEngine
- 初始化 HWComposer, 注册回调接口 registerCallback, HAL 会回调一些方法.
- 如果是 VR 模式, 创建 mVrFlinger
- 创建 mEventControlThread, 处理 Event 事件, 如 Vsync 事件和 hotplug 事件.
- 初始化显示设备 initializeDisplays

## 合成方式

目前 SurfaceFlinger 中支持两种合成方式, 一种是 Device 合成, 一种是 Client 合成. SurfaceFlinger 在收集可见层的所有缓冲区之后, 便会询问 Hardware Composer 应如何进行合成.

### Client 合成

Client 合成方式是相对与硬件合成来说的, 其合成方式是, 将各个 Layer 的内容用 GPU 渲染到暂存缓冲区中, 最后将暂存缓冲区传送到显示硬件. 这个暂存缓冲区, 我们称为 FBTarget, 每个 Display 设备有各自的 FBTarget. Client 合成, 之前称为 GLES 合成, 我们也可以称之为 GPU 合成. Client 合成, 采用 RenderEngine 进行合成.

### Device 合成

就是用专门的硬件合成器进行合成 HWComposer, 所以硬件合成的能力就取决于硬件的实现. 其合成方式是将各个 Layer 的数据全部传给显示硬件, 并告知它从不同的缓冲区读取屏幕不同部分的数据. HWComposer 是 Devicehec 的抽象.

rebuildLayerStacks

Android 支持多个屏幕, 每个屏幕的显示数据并不是完全一样的, 每个 Display 是分开合成的; 也就是说, layersSortedByZ 中的 layer 需要根据显示屏的特性, 分别进行合成, 合成后的数据, 送给各自的显示屏.

### 虚拟屏幕合成

虚拟屏幕合成与外部屏幕合成类似. 虚拟屏幕合成与物理屏幕合成之间的区别在于, 虚拟屏幕将输出发送到 Gralloc 缓冲区, 而不是发送到屏幕. 硬件混合渲染器 (HWC) 将输出内容写入缓冲区, 提供完成栅栏信号, 并将缓冲区发送给使用方(例如视频编码器、GPU、CPU 等). 如果显示通道写入内存, 虚拟屏幕便可使用 2D/位块传送器或叠加层.

### 模式

在 SurfaceFlinger 调用 validateDisplay() HWC 方法之后, 每个帧处于以下三种模式之一:

- GLES - GPU 合成所有图层, 直接写入输出缓冲区. HWC 不参与合成.
- MIXED - GPU 将一些图层合成到帧缓冲区, 由 HWC 合成帧缓冲区和剩余的图层, 直接写入输出缓冲区.
- HWC - HWC 合成所有图层并直接写入输出缓冲区.

### 输出格式

虚拟屏幕的缓冲区输出格式取决于其模式:

- GLES 模式 - EGL 驱动程序在 dequeueBuffer() 中设置输出缓冲区格式, 通常为 RGBA_8888. 使用方必须能够接受该驱动程序设置的输出格式, 否则便无法读取缓冲区.
- MIXED 和 HWC 模式 - 如果使用方需要 CPU 访问权限, 则由使用方设置格式. 否则, 格式将为 IMPLEMENTATION_DEFINED, Gralloc 会根据使用标记设置最佳格式. 例如, 如果使用方是视频编码器, 则 Gralloc 会设置 YCbCr 格式, 而 HWC 可以高效地写入该格式.

## Layer

Layer, 即图层, SurfaceFlinger 的合成就是将所有图层按照顺序和特定属性合成一帧画面

| 属性         | 描述                                                                                                                                                 |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Positional   | 定义层在其屏幕上的显示位置. 包括层边缘的位置及其相对于其他层的 Z 顺序(指示该层在其他层之前还是之后)等信息.                                           |
| Content      | 定义应如何在定位属性定义的边界内呈现层上显示的内容. 包括诸如剪裁(用来扩展内容的一部分以填充层的边界)和转换(用来显示旋转或翻转的内容)等信息.          |
| Composition  | 定义层应如何与其他层合成. 包括混合模式和用于 Alpha 合成的全层 Alpha 值等信息.                                                                        |
| Optimization | 提供对于正确合成层并非绝对必要但可由硬件混合渲染器 (HWC) 设备用来优化合成执行方式的信息. 包括层的可见区域以及层的哪个部分自上一帧以来已经更新等信息. |

- 一个 Layer 对应一个 BufferQueue, 一个 BufferQueue 中有多个 Buffer, 一般是 2 个或者 3 个.
- 一个 Layer 有一个 Producer, 一个 Consumer, 一个 SurfaceControl
- 一个 Surface 和一个 Layer 也是一一对应的, 和窗口也是一一对应的.

## LayerStack

Android 系统支持多种显示设备, 比如说, 输出到手机屏幕, 或者通过 WiFi 投射到电视屏幕. Android 用 DisplayDevice 类来表示这样的设备. 不是所有的 Layer 都会输出到所有的 Display, 比如说, 我们可以只将 Video Layer 投射到电视, 而非整个屏幕. LayerStack 就是为此设计, LayerStack 是一个 Display 对象的一个**数值**, 而类 Layer 里成员 State 结构体也有成员变量 mLayerStack, 只有两者的 layerStack 值相同, Layer 才会被输出到给该 Display 设备.

## OutputLayer

createSurface

```cpp
sp<SurfaceControl> SurfaceComposerClient::createSurface(
        const String8& name,
        uint32_t w,
        uint32_t h,
        PixelFormat format,
        uint32_t flags,
        SurfaceControl* parent,
        uint32_t windowType,
        uint32_t ownerUid)
{
    sp<SurfaceControl> sur;
    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        sp<IGraphicBufferProducer> gbp;

        if (parent != nullptr) {
            parentHandle = parent->getHandle();
        }
        status_t err = mClient->createSurface(name, w, h, format, flags, parentHandle,
                windowType, ownerUid, &handle, &gbp);
        ALOGE_IF(err, "SurfaceComposerClient::createSurface error %s", strerror(-err));
        if (err == NO_ERROR) {
            sur = new SurfaceControl(this, handle, gbp);
        }
    }
    return sur;
}
```

- name
  name 就是图层的名字, 没有实际意义, 主要是便于 debug, 有有名字, 就可以知道, 我们创建的图层是哪个了.
- w, h
  这个是图层的宽高, 我们用只是从 DisplayInfo 中读出来的, 跟屏幕的大小一样.
- format
  图层的格式, 应用中, 我们用的格式是 PIXEL_FORMAT_RGBX_8888.

createSurface 时, 可用的 flag 如下:

```cpp
* frameworks/native/libs/gui/include/gui/ISurfaceComposerClient.h

    // flags for createSurface()
    enum { // (keep in sync with Surface.java)
        eHidden = 0x00000004,
        eDestroyBackbuffer = 0x00000020,
        eSecure = 0x00000080,
        eNonPremultiplied = 0x00000100,
        eOpaque = 0x00000400,
        eProtectedByApp = 0x00000800,
        eProtectedByDRM = 0x00001000,
        eCursorWindow = 0x00002000,

        eFXSurfaceNormal = 0x00000000,
        eFXSurfaceColor = 0x00020000,
        eFXSurfaceMask = 0x000F0000,
    };
```

| Flag               | 作用             | 说明                                               |
| ------------------ | ---------------- | -------------------------------------------------- |
| eHidden            | 隐藏             | 表示 Layer 是不可见的                              |
| eDestroyBackbuffer | 销毁 Back Buffer | 表示 Layer 不保存 Back Buffer                      |
| eSecure            | 安全 layer       | 表示安全的 Layer, 只能显示在 Secure 屏幕上         |
| eNonPremultiplied  | 非自左乘         | 针对 Alpha 通道的, 如果没有 Alpha 通道没有实际意义 |
| eOpaque            | 非透明的         | 表示 Layer 不是透明的                              |
| eProtectedByApp    | 应用保护         | 应用将请求一个安全的硬件通道传输到外显             |
| eProtectedByDRM    | DRM 保护         | 应用请求一个安全的通道传输 DRM                     |
| eCursorWindow      | Cursor Layer     | 其实这 Layer 就是鼠标一类的图标                    |
| eFXSurfaceNormal   | 普通的 Layer     | 系统中大多数 layer 都是此类                        |
| eFXSurfaceColor    | 颜色 layer       | 在之前的版本中, 这个被称为 Dim Layer               |

- windowType
  窗口类型, 可以参照 WindowManager.java 中的定义, 如 FIRST_SYSTEM_WINDOW, TYPE_STATUS_BAR 等. 默认值为 0.
- ownerUid
  可以指定这个是哪个 UID 的 Layer.

- 一个 Layer 对应一个 BufferQueue, 一个 BufferQueue 中有多个 Buffer, 一般是 2 个或者 3 个.
- 一个 Layer 有一个 Producer, 一个 Consumer, 一个 SurfaceControl
- 结合前面的分析, 一个 Surface 和一个 Layer 也是一一对应的, 和窗口也是一一对应的.

ContainerLayer
ColorLayer
BufferLayer

Composition layers

## 显示系统各组件交互流程

来自 [github](https://github.com/huanzhiyazi/articles/blob/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger.md#ch6)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/lAJeo6.jpg)

一个是图层(Layer), 图层是 Surface 在其服务方的表示, 在 SurfaceFlinger 中叫 Layer, 在 HWC 中叫 hwLayer, HWComposer 是 SurfaceFlinger 中用于与 HWC HAL 进行沟通的代理

在客户端, 各个不同渲染源将借助 Surface 绘制单元窗口数据, 作为生产者将数据传送到 BufferQueue. 不同的渲染源可以是 Skia(通常的窗口布局绘制工具)、OpenGL(如游戏渲染)、视频解码等

作为窗口缓冲区的 BufferQueue, 将借助 Gralloc 进行缓冲区的申请和回收, Gralloc 借助 ion 对缓冲区以共享内存的方式进行访问. 具体而言, 面向单元窗口的缓冲区, 来自于预分配的内存池; 而面向普通合成窗口的缓冲区, 来自于 framebuffer. Gralloc 屏蔽了这些共享内存的来源细节, SurfaceFlinger 只需要告诉 Gralloc 申请的内存用于何种用途即可, Gralloc 将据此决定具体的内存申请源头.

SurfaceFlinger 执行合成的流程如下:

1.  HWC 触发 vsync 信号：vsync 信号将以回调的方式通知 SurfaceFlinger, SurfaceFlinger 收到 vsync 回调后开始执行下一帧的合成.
2.  SurfaceFlinger 更新图层：SurfaceFlinger 遍历各个有效图层, 从其对应的 BufferQueue 中获取最新的单元窗口绘制数据, 以对图层进行更新. 这一步的 BufferQueue 中的缓冲区来自于预分配内存.
3.  HWC 决策合成类型：SurfaceFlinger 更新并准备好所有图层后, 将图层参数告知 HWC HAL, HWC HAL 决定哪些图层可以执行 Device 合成.
4.  SurfaceFlinger 执行 Client 合成：如果有 HWC 不能处理的图层, SurfaceFlinger 统一将它们交给 OpenGL 执行合成, 其合成的数据作为一个普通合成窗口也插入到其对应的 BufferQueue 中, 同时 SurfaceFlinger 还充当该 BufferQueue 的消费者将普通合成窗口取出并作为一个新的合成图层与其它普通图层一起准备交与 HWC 进行 Device 合成. 注意, 这一步 BufferQueue 中的缓冲区来自于 framebuffer, 也就是说 Client 合成数据已经直接 post 到 framebuffer 中了.
5.  HWC 执行 Device 合成：HWC 将其余的图层连同 Client 合成图层一起进行 Device 合成.
6.  HWC 将合成的帧 post 到 framebuffer 显示：要将帧显示出来, 最终还是要将其 post 到 framebuffer 的 frontbuffer 中, 这样显示控制器（display controller）才能从 framebuffer 中读取帧数据进行扫描.
