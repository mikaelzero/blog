---
title: virtualDisplay原理研究
urlname: principle_virtualDisplay
date: 2022/11/11
tags:
  - VirtualDisplay
---

问题：

1. A 进程（为 app 端）创建虚拟屏, 将 B 启动到虚拟屏上
2. A 进程创建 textureName 后, 如何拿到 B 进程在 virtualDisplay 的数据

思路记录：

1. 创建虚拟屏的流程是什么样的
2. 创建后, surface 的流程是怎样的
3. 图像的生产和消费模型是怎样的
4. 数据流是怎样的
5. 如果是投屏, 如何拿到主屏的画面的

## 创建虚拟屏的流程

1. 经过 binder 到 DMS 中, 通过 SurfaceFlinger 的 createDisplay 创建 DisplayToken 并且添加到 mCurrentState.displays 列表中
2. 创建 VirtualDisplayDevice, 通过一系列回调和通知, 会触发 SurfaceFlinger 的 handleTransactionLocked 和 processDisplayChangesLocked,在这其中会创建 VirtualDisplaySurface,先通过以下逻辑判断 display 是否添加
   - 在 mCurrentState 中的 DisplayDeviceState 中有但是在 mDrawingState 中没有, 那么就说明在上一个 VSYNC 中新增了 Display
   - 在 mDrawingState 中的 DisplayDeviceState 中有但是在 mCurrentState 中没有, 那么就说明在上一个 VSYNC 中有 Display 被移除了
3. 根据 VirtualDisplaySurface 的 producer 创建 native window 即 cpp 层的 Surface, 根据这些创建 DisplayDeivce 并添加到 SurfaceFlinger 的 mDisplays 列表中, VirtualDisplay 才算正常创建完成

## Surface 的数据流

首先 A 进程创建 Surface 传递给 server 端, 一直传到 SurfaceFlinger, 在 setupNewDisplayDeviceInternal 流程中创建 Surface 时是根据 bufferProducer 来创建的,而这个 bufferProducer 为 VirtualDisplaySurface, 且 VirtualDisplaySurface 中包含 A 进程传递过来的 Surface, 传递过来的作用是作为在 dequeueBuffer 和 queueBuffer 时, 能够转交到 client 端

## 生产消费模型

创建 VirtualDisplaySurface 的之前,先创建 BufferQueue,bqProducer,bqConsumer,且根据 client 端的 BufferProducer 创建 VirtualDisplaySurface
VirtualDisplaySurface 作为生产者也是消费者,且在 dequeue 和 queue buffer 的时候,调用的都是 client 端的 BufferProducer 的 dequeue 和 queue

client 端:Surface 为生产者, SurfaceTexture 为消费者.

## 数据流

每次合成的时候, SurfaceFlinger 对每个 DisplayDevice 依次调用 doDisplayComposition(), 在 VirtualDisplay 情况下 的 doDisplayComposition() 中, 会调用 dequeueBuffer() 给接下来的合成（目前看 VirtualDisplay 都是 GPU 合成）申请 Buffer,转交到 client 端的 dequeueBuffer, 拿到申请的 buffer 后, 经过合成, SurfaceFlinger 合成再通过 queueBuffer() 将渲染完的 Buffer 还给 client 端, app 端接收到 onFrameAvailable 后, 通过 surfaceTexture 的 updateTexImage 去 acquireBuffer 拿去合成后的 buffer 数据, 就可以做纹理处理或丢给 unity 做处理

## 投屏画面

Android 支持多个屏幕, layer 可以定制化的只显示到某个显示屏幕上. 其中就是靠 layerStack(Layer 栈)来实现的
Layer 的 stack 值如果和 DisplayDevice 的 stack 值一样, 说明这个 layer 是属于这个显示屏幕的,前面创建的 VirtualDisplay 的 layerStack 对应就是主屏的 layerStack, 也就是说, 此方法会把所有要显示到主屏的 layer 全部都保存到 VirtualDisplay 中, 这样 VirtualDisplay 就拿到了所有要显示到主屏的内容
在 SurfaceFliner 刷新所有的 Layer 到屏幕上显示之前, 需要确定那些 Layer 应该显示到哪个屏幕上, 以上逻辑是通过 rebuildLayerStacks()来进行实现的
