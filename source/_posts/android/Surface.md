---
title: Surface
urlname: surace
date: 2023/04/18
tags:
  - Android
  - Framework
  - Surface
  - SurfaceFlinger
---

```java

final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);

WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);

                    final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
```

WMS 中创建 WindowState 时会根据 display

SurfaceComposerClient::createDisplay

SurfaceControl::setDisplaySurface

```c++
static void nativeSetDisplaySurface(JNIEnv* env, jclass clazz,
        jlong transactionObj,
        jobject tokenObj, jlong nativeSurfaceObject) {
    sp<IBinder> token(ibinderForJavaObject(env, tokenObj));
    if (token == NULL) return;
    sp<IGraphicBufferProducer> bufferProducer;
    sp<Surface> sur(reinterpret_cast<Surface *>(nativeSurfaceObject));
    if (sur != NULL) {
        bufferProducer = sur->getIGraphicBufferProducer();
    }
    status_t err = NO_ERROR;
    {
        auto transaction = reinterpret_cast<SurfaceComposerClient::Transaction*>(transactionObj);
        err = transaction->setDisplaySurface(token,
                bufferProducer);
    }
}
```

SurfaceComposerClient::Transaction::setDisplaySurface()

```c++
status_t SurfaceComposerClient::Transaction::setDisplaySurface(const sp<IBinder>& token,
        const sp<IGraphicBufferProducer>& bufferProducer) {
    if (bufferProducer.get() != nullptr) {
        status_t err = bufferProducer->setAsyncMode(true);
        ...
    }
    DisplayState& s(getDisplayState(token));
    s.surface = bufferProducer;
    s.what |= DisplayState::eSurfaceChanged;
    return NO_ERROR;
}
```

`IGraphicBufferProducer`是 Android 中的一个接口，用于与系统的图像缓冲区进行通信。主要作用是创建、管理和向`GraphicBuffer`（Android 中用于处理图像数据的类）对象发送数据。

在 Android 中，由于多个应用需要同时使用相同的图形资源，因此需要一个统一的机制来管理和分配图形资源。`IGraphicBufferProducer`就是一个完成这个任务的重要组件，它可以将多个应用或线程产生的图像数据通过共享缓冲区的方式提高图像数据的效率和处理速度。

作为一个接口，`IGraphicBufferProducer`提供了一系列方法，包括：

- `connect()`：连接`GraphicBuffer`生产者和消费者。
- `lockBuffer()`：锁定`GraphicBuffer`对象中的数据，并返回用于修改此缓冲区数据的指针。
- `queueBuffer()`：提交图像数据缓冲区，并等待系统呈现到目标设备，此时缓冲区可重新用于写入新的数据。
- `cancelBuffer()`：将缓冲区设置为已取消状态，并立即解除锁定。
- `detachBuffer()`：解除属性属于此生产者的缓冲区。
- `getMinUndequeuedBufferCount()`：返回仍未提交的缓冲区的最小数量。

在 Android 中，`Surface`就是一个基于`IGraphicBufferProducer`的应用程序和系统之间的接口，`SurfaceView`和`TextureView`是将`Surface`嵌入到 View 层次结构中的两个视图容器，它们是基于`Surface`实现的，可以用于提供高效的显示功能。

在 Android 中，SurfaceFlinger 的方法 setDisplayStateLocked() 通常在以下情况下被调用：

1. 当系统启动或读取屏幕配置时，SurfaceFlinger 会设置初始的 DisplayState，并调用 setDisplayStateLocked() 来启用或禁用特定的显示器或屏幕。

2. 当屏幕被创建或销毁时，SurfaceFlinger 会更新对应的 DisplayState，并调用 setDisplayStateLocked() 告诉系统屏幕的状态。

3. 当屏幕的某些属性发生变化时，例如分辨率、旋转方向等，SurfaceFlinger 也会更新相应的 DisplayState，并调用 setDisplayStateLocked() 告知系统。

总之，setDisplayStateLocked() 是 SurfaceFlinger 用于更新显示器状态的核心方法，它可以通过更改 DisplayState 中定义的属性来控制显示器的输出行为。

```c++
uint32_t SurfaceFlinger::setDisplayStateLocked(const DisplayState& s) {
    const ssize_t index = mCurrentState.displays.indexOfKey(s.token);
    if (index < 0) return 0;

    uint32_t flags = 0;
    DisplayDeviceState& state = mCurrentState.displays.editValueAt(index);

    const uint32_t what = s.what;
    if (what & DisplayState::eSurfaceChanged) {
        if (IInterface::asBinder(state.surface) != IInterface::asBinder(s.surface)) {
            state.surface = s.surface;
            flags |= eDisplayTransactionNeeded;
        }
    }
```

将 DisplayState 中的 Surface （即 App 端创建的 BufferProducer） 传递给 DisplayDeviceState，同时将 eSurfaceChanged 转换为 eDisplayTransactionNeeded。这一下，不仅完成了 DisplayState 的内容传递到 DisplayDeviceState，还完成了 state 转为 Transaction.