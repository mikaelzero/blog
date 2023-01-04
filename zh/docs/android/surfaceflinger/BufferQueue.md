转自：https://zhuanlan.zhihu.com/p/62813895?utm_id=0

**1\. 背景**

对业务开发来说, 无法接触到 BufferQueue, 甚至不知道 BufferQueue 是什么东西. 对系统来说, BufferQueue 是很重要的传递数据的组件, Android 显示系统依赖于 BufferQueue, 只要显示内容到“屏幕”(此处指抽象的屏幕, 有时候还可以包含编码器), 就一定需要用到 BufferQueue, 可以说在显示/播放器相关的领悟中, BufferQueue 无处不在. 即使直接调用 Opengl ES 来绘制, 底层依然需要 BufferQueue 才能显示到屏幕上.

弄明白 BufferQueue, 不仅可以增强对 Android 系统的了解, 还可以弄明白/排查相关的问题, 如为什么 Mediacodec 调用 dequeueBuffer 老是返回-1？为什么普通 View 的 draw 方法直接绘制内容即可, SurfaceView 在 draw 完毕后还需要 unlockCanvasAndPost？

注：本文分析的代码来自于 Android6.0.1.

**2\. BufferQueue 内部运作方式**

BufferQueue 是 Android 显示系统的核心, 它的设计哲学是生产者-消费者模型, 只要往 BufferQueue 中填充数据, 则认为是生产者, 只要从 BufferQueue 中获取数据, 则认为是消费者. 有时候同一个类, 在不同的场景下既可能是生产者也有可能是消费者. 如 SurfaceFlinger, 在合成并显示 UI 内容时, UI 元素作为生产者生产内容, SurfaceFlinger 作为消费者消费这些内容. 而在截屏时, SurfaceFlinger 又作为生产者将当前合成显示的 UI 内容填充到另一个 BufferQueue, 截屏应用此时作为消费者从 BufferQueue 中获取数据并生产截图.

以下是 Android 官网对其的介绍：

![](https://pic4.zhimg.com/80/v2-6a617ddb116d922b24f416582a5bf013_1440w.webp)

以下是常见的 BufferQueue 使用步骤：

1.  初始化一个 BufferQueue
2.  图形数据的生产者通过 BufferQueue 申请一块 GraphicBuffer, 对应图中的 dequeueBuffer 方法
3.  申请到 GraphicBuffer 后, 获取 GraphicBuffer, 通过函数 requestBuffer 获取
4.  获取到 GraphicBuffer 后, 通过各种形式往 GraphicBuffer 中填充图形数据后, 然后将 GraphicBuffer 入队到 BufferQueue 中, 对应上图中的 queueBuffer 方法
5.  在新的 GraphicBuffer 入队 BufferQueue 时, BufferQueue 会通过回调通知图形数据的消费者, 有新的图形数据被生产出来了
6.  然后消费者从 BufferQueue 中出队一个 GraphicBuffer, 对应图中的 acquireBuffer 方法
7.  待消费者消费完图形数据后, 将空的 GraphicBuffer 还给 BufferQueue 以便重复利用, 此时对应上图中的 releaseBuffer 方法
8.  此时 BufferQueue 再通过回调通知图形数据的生产者有空的 GraphicBuffer 了, 图形数据的生产者又可以从 BufferQueue 中获取一个空的 GraphicBuffer 来填充数据
9.  一直循环 2-8 步骤, 这样就有条不紊的完成了图形数据的生产-消费

当然图形数据的生产者可以不用等待 BufferQueue 的回调再生产数据, 而是一直生产数据然后入队到 BufferQueue, 直到 BufferQueue 满为止. 图形数据的消费者也可以不用等 BufferQueue 的回调通知, 每次都从 BufferQueue 中尝试获取数据, 获取失败则尝试, 只是这样效率比较低, 需要不断的轮训 BufferQueue（因为 BufferQueue 有同步阻塞和非同步阻塞两种机种, 在非同步阻塞机制下获取数据失败不会阻塞该线程直到有数据才唤醒该线程, 而是直接返回-1）.

同时使用 BufferQueue 的生产者和消费者往往处在不同的进程, BufferQueue 内部使用共享内存和 Binder 在不同的进程传递数据, 减少数据拷贝提高效率.

和 BufferQueue 有关的几个类分别是：

1.  BufferBufferCore：BufferQueue 的实际实现
2.  BufferSlot：用来存储 GraphicBuffer
3.  BufferState：表示 GraphicBuffer 的状态
4.  IGraphicBufferProducer：BufferQueue 的生产者接口, 实现类是 BufferQueueProducer
5.  IGraphicBufferConsumer：BufferQueue 的消费者接口, 实现类是 BufferQueueConsumer
6.  GraphicBuffer：表示一个 Buffer, 可以填充图像数据
7.  ANativeWindow_Buffer：GraphicBuffer 的父类
8.  ConsumerBase：实现了 ConsumerListener 接口, 在数据入队列时会被调用到, 用来通知消费者

BufferQueue 中用 BufferSlot 来存储 GraphicBuffer, 使用数组来存储一系列 BufferSlot, 数组默认大小为 64.

GraphicBuffer 用 BufferState 来表示其状态, 有以下状态：

1.  FREE：表示该 Buffer 没有被生产者-消费者所使用, 该 Buffer 的所有权属于 BufferQueue
2.  DEQUEUED：表示该 Buffer 被生产者获取了, 该 Buffer 的所有权属于生产者
3.  QUEUED：表示该 Buffer 被生产者填充了数据, 并且入队到 BufferQueue 了, 该 Buffer 的所有权属于 BufferQueue
4.  ACQUIRED：表示该 Buffer 被消费者获取了, 该 Buffer 的所有权属于消费者

为什么需要这些状态呢？ 假设不需要这些状态, 实现一个简单的 BufferQueue, 假设是如下实现：

```cpp
BufferQueue{
 vector<GraphicBuffer> slots;
 void push(GraphicBuffer slot){
 slots.push(slot);
 }
 GraphicBuffer pull(){
 return slots.pull();
 }
}

```

生产者生产完数据后, 通过调用 BufferQueue 的 push 函数将数据插入到 vector 中. 消费者调用 BufferQueue 的 pull 函数出队一个 Buffer 数据.

上述实现的问题在于, 生产者每次都需要自行创建 GraphicBuffer, 而消费者每次消费完数据后的 GraphicBuffer 就被释放了, GraphicBuffer 没有得到循环利用. 而在 Android 中, 由于 BufferQueue 的生产者-消费者往往处于不同的进程, GraphicBuffer 内部是需要通过共享内存来连接生成者-消费者进程的, 每次创建 GraphicBuffer, 即意味着需要创建共享内存, 效率较低.

而 BufferQueue 中用 BufferState 来表示 GraphicBuffer 的状态则解决了这个问题. 每个 GraphicBuffer 都有当前的状态, 通过维护 GraphicBuffer 的状态, 完成 GraphicBuffer 的复用.

由于 BufferQueue 内部实现是 BufferQueueCore, 下文均用 BufferQueueCore 代替 BufferQueue. 先介绍下 BufferQueueCore 内部相应的数据结构, 再介绍 BufferQueue 的状态扭转过程和生产-消费过程.

以下是 Buffer 的入队/出队操作和 BufferState 的状态扭转的过程, 这里只介绍非同步阻塞模式.

2.1 BufferQueueCore 内部数据结构

核心数据结构如下：

```cpp
BufferQueueDefs::SlotsType mSlots：用数组存放的Slot, 数组默认大小为BufferQueueDefs::NUM_BUFFER_SLOTS,具体是64, 代表所有的Slot
std::set<int> mFreeSlots：当前所有的状态为FREE的Slot, 这些Slot没有关联上具体的GraphicBuffer, 后续用的时候还需要关联上GraphicBuffer
std::list<int> mFreeBuffers：当前所有的状态为FREE的Slot, 这些Slot已经关联上具体的GraphicBuffer, 可以直接使用
Fifo mQueue：一个先进先出队列, 保存了生产者生产的数据

```

在 BufferQueueCore 初始化时, 由于此时队列中没有入队任何数据, 按照上面的介绍, 此时 mFreeSlots 应该包含所有的 Slot, 元素大小和 mSlots 一致, 初始化代码如下：

```cpp
for (int slot = 0; slot < BufferQueueDefs::NUM_BUFFER_SLOTS; ++slot) {
 mFreeSlots.insert(slot);
 }

```

**2.2 生产者 dequeueBuffer**

当生产者可以生产图形数据时, 首先向 BufferQueue 中申请一块 GraphicBuffer. 调用函数是 BufferQueueProducer.dequeueBuffer, 如果当前 BufferQueue 中有可用的 GraphicBuffer, 则返回其对用的索引, 如果不存在, 则返回-1, 代码在 BufferQueueProducer, 流程如下：

```cpp
status_t BufferQueueProducer::dequeueBuffer(int *outSlot,
 sp<android::Fence> *outFence, bool async,
 uint32_t width, uint32_t height, PixelFormat format, uint32_t usage) {
 //1. 寻找可用的Slot, 可用指Buffer状态为FREE
 status_t status = waitForFreeSlotThenRelock("dequeueBuffer", async,
 &found, &returnFlags);
 if (status != NO_ERROR) {
 return status;
 }
 //2.找到可用的Slot, 将Buffer状态设置为DEQUEUED, 由于步骤1找到的Slot状态为FREE,因此这一步完成了FREE到DEQUEUED的状态切换
 *outSlot = found;
 ATRACE_BUFFER_INDEX(found);
 attachedByConsumer = mSlots[found].mAttachedByConsumer;
 mSlots[found].mBufferState = BufferSlot::DEQUEUED;
 //3. 找到的Slot如果需要申请GraphicBuffer, 则申请GraphicBuffer, 这里采用了懒加载机制, 如果内存没有申请, 申请内存放在生产者来处理
 if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
 status_t error;
 sp<GraphicBuffer> graphicBuffer(mCore->mAllocator->createGraphicBuffer(width, height, format, usage, &error));
 graphicBuffer->setGenerationNumber(mCore->mGenerationNumber);
 mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
 }
}

```

关键在于寻找可用 Slot, waitForFreeSlotThenRelock 的流程如下：

```cpp
status_t BufferQueueProducer::waitForFreeSlotThenRelock(const char* caller,
 bool async, int* found, status_t* returnFlags) const {
 //1. mQueue 是否太多
 bool tooManyBuffers = mCore->mQueue.size()> static_cast<size_t>(maxBufferCount);
 if (tooManyBuffers) {
 } else {
 // 2. 先查找mFreeBuffers中是否有可用的, 由2.1介绍可知, mFreeBuffers中的元素关联了GraphicBuffer, 直接可用
 if (!mCore->mFreeBuffers.empty()) {
 auto slot = mCore->mFreeBuffers.begin();
 *found = *slot;
 mCore->mFreeBuffers.erase(slot);
 } else if (mCore->mAllowAllocation && !mCore->mFreeSlots.empty()) {
 // 3. 再查找mFreeSlots中是否有可用的, 由2.1可知, 初始化时会填充满这个列表, 因此第一次调用一定不会为空. 同时用这个列表中的元素需要关联上GraphicBuffer才可以直接使用, 关联的过程由外层函数来实现
 auto slot = mCore->mFreeSlots.begin();
 // Only return free slots up to the max buffer count
 if (*slot < maxBufferCount) {
 *found = *slot;
 mCore->mFreeSlots.erase(slot);
 }
 }
 }
 tryAgain = (*found == BufferQueueCore::INVALID_BUFFER_SLOT) ||
 tooManyBuffers;
 //4. 如果找不到可用的Slot或者Buffer太多（同步阻塞模式下）, 则可能需要等
 if (tryAgain) {
 if (mCore->mDequeueBufferCannotBlock &&
 (acquiredCount <= mCore->mMaxAcquiredBufferCount)) {
 return WOULD_BLOCK;
 }
 mCore->mDequeueCondition.wait(mCore->mMutex);
 }
}
waitForFreeSlotThenRelock函数会尝试寻找一个可用的Slot, 可用的Slot状态一定是FREE（因为是从两个FREE状态的列表中获取的）, 然后dequeueBuffer将状态改变为DEQUEUED, 即完成了状态的扭转.

```

waitForFreeSlotThenRelock 返回可用的 Slot 分为两种：

1.  从 mFreeBuffers 中获取到的, mFreeBuffers 中的元素关联了 GraphicBuffer, 直接可用
2.  从 mFreeSlots 中获取到的, 没有关联上 GraphicBuffer, 因此需要申请 GraphicBuffer 并和 Slot 关联上, 通过 createGraphicBuffer 申请一个 GraphicBuffer, 然后赋值给 Slot 的 mGraphicBuffer 完成关联

小结 dequeueBuffer：尝试找到一个 Slot, 并完成 Slot 与 GraphicBuffer 的关联（如果需要）, 然后将 Slot 的状态由 FREE 扭转成 DEQUEUED. 返回 Slot 在 BufferQueueCore 中 mSlots 对应的索引.

**2.3 生产者 requestBuffer**

dequeueBuffer 函数获取到了可用 Slot 的索引后, 通过 requestBuffer 获取到对应的 GraphicBuffer. 流程如下：

```cpp
status_t BufferQueueProducer::requestBuffer(int slot, sp<GraphicBuffer>* buf) {
 // 1. 判断slot参数是否合法
 if (slot < 0 || slot >= BufferQueueDefs::NUM_BUFFER_SLOTS) {
 BQ_LOGE("requestBuffer: slot index %d out of range [0, %d)",
 slot, BufferQueueDefs::NUM_BUFFER_SLOTS);
 return BAD_VALUE;
 } else if (mSlots[slot].mBufferState != BufferSlot::DEQUEUED) {
 BQ_LOGE("requestBuffer: slot %d is not owned by the producer "
 "(state = %d)", slot, mSlots[slot].mBufferState);
 return BAD_VALUE;
 }
 //2. 将mRequestBufferCalled置为true
 mSlots[slot].mRequestBufferCalled = true;
 *buf = mSlots[slot].mGraphicBuffer;
 return NO_ERROR;
}

```

这一步不是必须的. 业务层可以直接通过 Slot 的索引获取到对应的 GraphicBuffer.

**2.4 生产者 queueBuffer**

上文 dequeueBuffer 获取到一个 Slot 后, 就可以在 Slot 对应的 GraphicBuffer 上完成图像数据的生产了, 可以是 View 的主线程 Draw 过程, 也可以是 SurfaceView 的子线程绘制过程, 甚至可以是 MediaCodec 的解码过程.

填充完图像数据后, 需要将 Slot 入队 BufferQueueCore（数据写完了, 可以传给生产者-消费者队列, 让消费者来消费了）, 入队调用 queueBuffer 函数. queueBuffer 的流程如下：

```cpp
status_t BufferQueueProducer::queueBuffer(int slot,
 const QueueBufferInput &input, QueueBufferOutput *output) {
 // 1. 先判断传入的Slot是否合法
 if (slot < 0 || slot >= maxBufferCount) {
 BQ_LOGE("queueBuffer: slot index %d out of range [0, %d)",
 slot, maxBufferCount);
 return BAD_VALUE;
 }
 //2. 将Buffer状态扭转成QUEUED, 此步完成了Buffer的状态由DEQUEUED到QUEUED的过程
 mSlots[slot].mFence = fence;
 mSlots[slot].mBufferState = BufferSlot::QUEUED;
 ++mCore->mFrameCounter;
 mSlots[slot].mFrameNumber = mCore->mFrameCounter;
 //3. 入队mQueue
 if (mCore->mQueue.empty()) {
 mCore->mQueue.push_back(item);
 frameAvailableListener = mCore->mConsumerListener;
 }
 // 4. 回调frameAvailableListener, 告知消费者有数据入队了
 if (frameAvailableListener != NULL) {
 frameAvailableListener->onFrameAvailable(item);
 } else if (frameReplacedListener != NULL) {
 frameReplacedListener->onFrameReplaced(item);
 }
}
从上面的注释可以看到, queueBuffer的主要步骤如下：

```

1.  将 Buffer 状态扭转成 QUEUED, 此步完成了 Buffer 的状态由 DEQUEUED 到 QUEUED 的过程
2.  将 Buffer 入队到 BufferQueueCore 的 mQueue 队列中
3.  回调 frameAvailableListener, 告知消费者有数据入队, 可以来消费数据了, frameAvailableListener 是消费者注册的回调

小结 queueBuffer：将 Slot 的状态扭转成 QUEUED, 并添加到 mQueue 中, 最后通知消费者有数据入队.

2.5 消费者 acquireBuffer

在消费者接收到 onFrameAvailable 回调时或者消费者主动想要消费数据, 调用 acquireBuffer 尝试向 BufferQueueCore 获取一个数据以供消费. 消费者的代码在 BufferQueueConsumer 中, acquireBuffer 流程如下：

```cpp
status_t BufferQueueConsumer::acquireBuffer(BufferItem* outBuffer,
 nsecs_t expectedPresent, uint64_t maxFrameNumber) {
 //1. 如果队列为空, 则直接返回
 if (mCore->mQueue.empty()) {
 return NO_BUFFER_AVAILABLE;
 }
 //2. 取出mQueue队列的第一个元素, 并从队列中移除
 BufferQueueCore::Fifo::iterator front(mCore->mQueue.begin());
 int slot = front->mSlot;
 *outBuffer = *front;
 mCore->mQueue.erase(front);
 //3. 处理expectedPresent的情况, 这种情况可能会连续丢几个Slot的“显示”时间小于expectedPresent的情况,这种情况下这些Slot已经是“过时”的, 直接走下文的releaseBuffer消费流程,代码比较长, 忽略了

 //4. 更新Slot的状态为ACQUIRED
 if (mCore->stillTracking(front)) {
 mSlots[slot].mAcquireCalled = true;
 mSlots[slot].mNeedsCleanupOnRelease = false;
 mSlots[slot].mBufferState = BufferSlot::ACQUIRED;
 mSlots[slot].mFence = Fence::NO_FENCE;
 }
 //5. 如果步骤3有直接releaseBuffer的过程, 则回调生产者, 有数据被消费了
 if (listener != NULL) {
 for (int i = 0; i < numDroppedBuffers; ++i) {
 listener->onBufferReleased();
 }
 }
}

```

从上面的注释可以看到, acquireBuffer 的主要步骤如下：

1.  从 mQueue 队列中取出并移除一个元素
2.  改变 Slot 对应的状态为 ACQUIRED
3.  如果有丢帧逻辑, 回调告知生产者有数据被消费, 生产者可以准备生产数据了

小结 acquireBuffer：将 Slot 的状态扭转成 ACQUIRED, 并从 mQueue 中移除, 最后通知生产者有数据出队.

**2.6 消费者 releaseBuffer**

消费者获取到 Slot 后开始消费数据（典型的消费如 SurfaceFlinger 的 UI 合成）, 消费完毕后, 需要告知 BufferQueueCore 这个 Slot 被消费者消费完毕了, 可以给生产者重新生产数据, releaseBuffer 流程如下：

```cpp
status_t BufferQueueConsumer::releaseBuffer(int slot, uint64_t frameNumber,
 const sp<Fence>& releaseFence, EGLDisplay eglDisplay,EGLSyncKHR eglFence) {
 //1. 检查Slot是否合法
 if (slot < 0 || slot >= BufferQueueDefs::NUM_BUFFER_SLOTS ||
 return BAD_VALUE;
 }
 //2. 容错处理：如果要处理的Slot存在于mQueue中, 那么说明这个Slot的来源不合法, 并不是从2.5的acquireBuffer获取的Slot, 拒绝处理
 BufferQueueCore::Fifo::iterator current(mCore->mQueue.begin());
 while (current != mCore->mQueue.end()) {
 if (current->mSlot == slot) {
 return BAD_VALUE;
 }
 ++current;
 }
 // 3. 将Slot的状态扭转为FREE, 之前是ACQUIRED, 并将该Slot添加到BufferQueueCore的mFreeBuffers列表中（mFreeBuffers的定义参考2.1的介绍）
 if (mSlots[slot].mBufferState == BufferSlot::ACQUIRED) {
 mSlots[slot].mEglDisplay = eglDisplay;
 mSlots[slot].mEglFence = eglFence;
 mSlots[slot].mFence = releaseFence;
 mSlots[slot].mBufferState = BufferSlot::FREE;
 mCore->mFreeBuffers.push_back(slot);
 listener = mCore->mConnectedProducerListener;
 BQ_LOGV("releaseBuffer: releasing slot %d", slot);
 }
 // 4. 回调生产者, 有数据被消费了
 if (listener != NULL) {
 listener->onBufferReleased();
 }
}

```

从上面的注释可以看到, releaseBuffer 的主要步骤如下：

1.  将 Slot 的状态扭转为 FREE
2.  将被消费的 Slot 添加到 mFreeBuffers 供后续的生产者 dequeueBuffer 使用
3.  回调告知生产者有数据被消费, 生产者可以准备生产数据了

小结 releaseBuffer：将 Slot 的状态扭转成 FREE, 并添加到 BufferQueueCore mFreeBuffers 队列中, 最后通知生产者有数据出队.

总结下状态变化的过程：

![](https://pic3.zhimg.com/80/v2-dbf15c50de4a4a1976afc594139458ba_1440w.webp)

上面主要介绍了 BufferQueue 的设计思想和内部实现.

下面将继续介绍 BufferQueue, 着重介绍 Android 中对于 BufferQueue 的常用封装, 以及 SurfaceView 中使用 BufferQueue 的具体实现.

**3.BufferQueue 常用封装类**

在实际应用中, 除了直接使用 BuferQueue 外, 更多的是使用 Surface/SurfaceTexture, 其对 BufferQueue 做了包装, 方便业务更方便的使用 BufferQueue. Surface 作为 BufferQueue 的生产者, SurfaceTexture 作为 BufferQueue 的消费者.

3.1 Surface

Surface 的构造函数如下：

Surface::Surface(

const sp<IGraphicBufferProducer>& bufferProducer,

bool controlledByApp)

: mGraphicBufferProducer(bufferProducer),

mGenerationNumber(0)

构造函数需要传入一个生产者的引用, 和 BufferQueue 的交互均有这个生产者的引用来完成. dequeueBuffer 的流程如下：

```cpp
int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
 // 1. 调用mGraphicBufferProducer的dequeueBuffer方法, 尝试获取一个Slot索引
 int buf = -1;
 sp<Fence> fence;
 status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, swapIntervalZero,
 reqWidth, reqHeight, reqFormat, reqUsage);
 if (result < 0) {
 ALOGV("dequeueBuffer: IGraphicBufferProducer::dequeueBuffer(%d, %d, %d, %d, %d)"
 "failed: %d", swapIntervalZero, reqWidth, reqHeight, reqFormat,
 reqUsage, result);
 return result;
 }
 // 2. 调用mGraphicBufferProducer的requestBuffer方法, 尝试获取Slot
 sp<GraphicBuffer>& gbuf(mSlots[buf].buffer);
 if ((result & IGraphicBufferProducer::BUFFER_NEEDS_REALLOCATION) || gbuf == 0) {
 result = mGraphicBufferProducer->requestBuffer(buf, &gbuf);
 if (result != NO_ERROR) {
 ALOGE("dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: %d", result);
 mGraphicBufferProducer->cancelBuffer(buf, fence);
 return result;
 }
 }
 // 3. 返回GraphicBuffer
 *buffer = gbuf.get();
}

```

queueBuffer 也是如下, 流程如下：

```cpp
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
 IGraphicBufferProducer::QueueBufferOutput output;
 IGraphicBufferProducer::QueueBufferInput input(timestamp, isAutoTimestamp,
 mDataSpace, crop, mScalingMode, mTransform ^ mStickyTransform,
 mSwapIntervalZero, fence, mStickyTransform);
 // 1. 直接调用mGraphicBufferProducer的queueBuffer方法即可
 status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
 if (err != OK) {
 ALOGE("queueBuffer: error queuing buffer to SurfaceTexture, %d", err);
 }
}

```

Surface 还提供了 lock 函数, 用来支持双缓冲, 内部也是调用 dequeueBuffer 方法获取最新的 Buffer：

```cpp
status_t Surface::lock(
 ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
 ANativeWindowBuffer* out;
 int fenceFd = -1;
 //1. 获取实际Buffer
 status_t err = dequeueBuffer(&out, &fenceFd);
 //2. 处理双缓冲
 if (canCopyBack) {
 // copy the area that is invalid and not repainted this round
 const Region copyback(mDirtyRegion.subtract(newDirtyRegion));
 if (!copyback.isEmpty())
 copyBlt(backBuffer, frontBuffer, copyback);
 }
}

```

Surface 也提供了 unlockAndPost 方法, 将数据给到 BufferQueue：

```cpp
status_t Surface::unlockAndPost()
{
 if (mLockedBuffer == 0) {
 ALOGE("Surface::unlockAndPost failed, no locked buffer");
 return INVALID_OPERATION;
 }
 int fd = -1;
 status_t err = mLockedBuffer->unlockAsync(&fd);
 ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);
 //1. 将生产好的数据给到BufferQueue
 err = queueBuffer(mLockedBuffer.get(), fd);
 ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
 mLockedBuffer->handle, strerror(-err));
 mPostedBuffer = mLockedBuffer;
 mLockedBuffer = 0;
 return err;
}

```

**3.2 SurfaceTexture**

SurfaceTexture 作为 BufferQueue 的消费者, 其初始化代码如下：

```cpp
static void SurfaceTexture_init(JNIEnv* env, jobject thiz, jboolean isDetached,
 jint texName, jboolean singleBufferMode, jobject weakThiz)
{
 sp<IGraphicBufferProducer> producer;
 sp<IGraphicBufferConsumer> consumer;
 //1. 创建一个BufferQueue
 BufferQueue::createBufferQueue(&producer, &consumer);
 if (singleBufferMode) {
 consumer->disableAsyncBuffer();
 consumer->setDefaultMaxBufferCount(1);
 }
 //2. 创建一个消费者实例surfaceTexture
 sp<GLConsumer> surfaceTexture;
 if (isDetached) {
 surfaceTexture = new GLConsumer(consumer, GL_TEXTURE_EXTERNAL_OES,
 true, true);
 } else {
 surfaceTexture = new GLConsumer(consumer, texName,
 GL_TEXTURE_EXTERNAL_OES, true, true);
 }
 //3. 将消费者实例和该BufferQueue对应的生产者保存到java层, 这样Surface构造时, 就可以获取到该BufferQueue对应的生产者了
 SurfaceTexture_setSurfaceTexture(env, thiz, surfaceTexture);
 SurfaceTexture_setProducer(env, thiz, producer);
}

```

消费的方法是 updateTexImage, 流程如下：

```cpp
static void SurfaceTexture_updateTexImage(JNIEnv* env, jobject thiz)
{
 // 1. 先获取到初始化时构造的消费者
 sp<GLConsumer> surfaceTexture(SurfaceTexture_getSurfaceTexture(env, thiz));
 // 2. 调用消费者的updateTexImage方法
 status_t err = surfaceTexture->updateTexImage方法();
 if (err == INVALID_OPERATION) {
 jniThrowException(env, IllegalStateException, "Unable to update texture contents (see "
 "logcat for details)");
 } else if (err < 0) {
 jniThrowRuntimeException(env, "Error during updateTexImage (see logcat for details)");
 }
}

```

GLConsumer 的 updateTextImage 实现如下：

```cpp
status_t GLConsumer::updateTexImage() {
 BufferItem item;
 //1. 调用自身的acquireBufferLocked方法
 err = acquireBufferLocked(&item, 0);:updateTexImage() {
 // Release the previous buffer.
 err = updateAndReleaseLocked(item);
 if (err != NO_ERROR) {
 glBindTexture(mTexTarget, mTexName);
 return err;
 }
}

```

acquireBufferLocked 方法, 最终走到了 ConsumerBase 的 acquireBufferLocked 方法.

```cpp
status_t ConsumerBase::acquireBufferLocked(BufferItem *item,
 nsecs_t presentWhen, uint64_t maxFrameNumber) {
 //1. 最终还是走到了消费者的acquireBuffer方法, 消费者对应上面的BufferQueueConsumer
 status_t err = mConsumer->acquireBuffer(item, presentWhen, maxFrameNumber);
 if (err != NO_ERROR) {
 return err;
 }
 return OK;
}

```

同理, 消费者消费数据的方法是 releaseTexImage, 最终也会走到 BufferQueueConsumer 的 releaseBufferLocked 方法, 这里不再描述了.

**4.BufferQueue 的实例**

上述介绍了 BufferQueue 的内部实现, 以及常用的封装类. 接下来将介绍一个具体的实例.

Android 中, SurfaceView 作为系统提供的组件, 因为可以在子线程中绘制提高性能, SurfaceView 拥有自身的 Surface, 不需要和 Activity 的 Surface 共享, 在 SurfaceFlinger 中, Activity 的 Surface 和 SurfaceView 的 Surface 是平级且互相独立的, 可以独立的进行合成. 那我们来看一下 SurfaceView 是怎么使用 BufferQueue 的.

4.1 数据的生产过程

SurfaceView 的 Surface 创建过程, 这里不关注, 有兴趣的可以参考 android SurfaceView 绘制实现原理解析 这篇文章, 我们主要关注其中与 BufferQueue 相关的绘制和显示步骤.

使用 SuerfaceView 绘制伪码如下：

Canvas canvas = null;

try {

canvas = holder.lockCanvas(null);

//实际的 draw

}catch (Exception e) {

// TODO: handle exception

e.printStackTrace();

}finally {

if(canvas != null) {

holder.unlockCanvasAndPost(canvas);

}

需要调用 lockCanvas 和 unlockCanvasAndPost 方法, 这两个方法的作用是什么呢？

先看下 lockCanvas, 调用流程是：

1.  SurfaceHolder.lockCanvas
2.  SurfaceHolder.internalLockCanvas
3.  Surface.lockCanvas
4.  Surface.nativeLockCanvas

nativeLockCanvas 实现如下：

```cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
 jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
 sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
 ANativeWindow_Buffer outBuffer;
 //1. 通过Surface::lock方法, 获取一个合适的Buffer
 status_t err = surface->lock(&outBuffer, dirtyRectPtr);
 //2. 构造一个Bitmap, 地址指向步骤1获取的Buffer的地址, 这样在这个Bitmap上绘制的内容, 直接绘制到了GraphicBuffer, 如果GraphicBuffer的内存是SurfaceFlinger通过共享内存申请的, 那么SurfaceFlinger就能直接看到绘制的图形数据
 SkImageInfo info = SkImageInfo::Make(outBuffer.width, outBuffer.height,
 convertPixelFormat(outBuffer.format),
 kPremul_SkAlphaType);
 SkBitmap bitmap;
 ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
 bitmap.setInfo(info, bpr);
 if (outBuffer.width > 0 && outBuffer.height > 0) {
 bitmap.setPixels(outBuffer.bits);
 } else {
 // be safe with an empty bitmap.
 bitmap.setPixels(NULL);
 }
 // 3. 将创建的Bitmap设置给Canvas, 作为画布
 Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
 nativeCanvas->setBitmap(bitmap);
}

```

从这里可以看到, nativeLockCanvas 的步骤主要如下：

1.  通过调用 Surface::lock 方法（内部也是调用 dequeueBuffer 和 requestBuffer 方法）, 获取到一个 GraphicBuffer
2.  将步骤 1 获取的 GraphicBuffer 构造成一个 Bitmap, 设置给 Canvas
3.  应用通过这个 Canvas 就可以绘制图形了

在绘制图形完成后, 调用 unlockCanvasAndPost 方法, 调用流程是：

1.  SurfaceHolder.unlockCanvasAndPost
2.  Surface.unlockCanvasAndPost
3.  Surface.nativeUnlockCanvasAndPost

nativeUnlockCanvasAndPost 的实现如下：

static void nativeUnlockCanvasAndPost(JNIEnv\* env, jclass clazz,

jlong nativeObject, jobject canvasObj) {

sp<Surface> surface(reinterpret_cast<Surface \*>(nativeObject));

if (!isSurfaceValid(surface)) {

return;

}

// detach the canvas from the surface

Canvas\* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);

nativeCanvas->setBitmap(SkBitmap());

// 直接调用 Surface 的 unlockAndPost 方法, 有上文可知 unlockAndPost 内部最终也会调用到 qeueBuffer 方法

status_t err = surface->unlockAndPost 方法, 有上文可知 unlockAndPost 内部最终也会调用到 qeueBuffer 方法();

if (err < 0) {

doThrowIAE(env);

}

}

从注释可以看到, 这个方法, 最终会调用到 Surface 的 unlockAndPost 方法方法, 而该方法内部最终也会调用到 BufferQueueProducer 的 queueBuffer 方法. 即完成了数据的生产和入队.

4.2 数据的消费过程

SurfaceView 绘制的数据, 传递过 BufferQueue 后, 最终由 SurfaceFlinger 进行合成消费. SurfaceFlinger 的消费由 SurfaceFlingerConsumer 实现, 流程如下：

```cpp
status_t SurfaceFlingerConsumer::updateTexImage(BufferRejecter* rejecter,
 const DispSync& dispSync, uint64_t maxFrameNumber)
{
 BufferItem item;
 // 1. 调用acquireBufferLocked获取一个Slot
 err = acquireBufferLocked(&item, computeExpectedPresent(dispSync),
 maxFrameNumber);
 if (err != NO_ERROR) {
 return err;
 }
 //2. 消费完毕, 释放Slot
 err = updateAndReleaseLocked(item);
 if (err != NO_ERROR) {
 return err;
 }
}

```

acquireBufferLocked 的实现如下：

```cpp
status_t SurfaceFlingerConsumer::acquireBufferLocked(BufferItem* item,
 nsecs_t presentWhen, uint64_t maxFrameNumber) {
 //1. 调用 GLConsumer::acquireBufferLocked, 最终会调用到BufferQueueConsumer的acquireBuffer方法
 status_t result = GLConsumer::acquireBufferLocked(item, presentWhen,
 maxFrameNumber);
 if (result == NO_ERROR) {
 mTransformToDisplayInverse = item->mTransformToDisplayInverse;
 mSurfaceDamage = item->mSurfaceDamage;
 }
 return result;
}

```

而 updateAndReleaseLocked 方法的流程如下：

-

```cpp
status_t GLConsumer::updateAndReleaseLocked(const BufferItem& item)
{
 // Do whatever sync ops we need to do before releasing the old slot.
 err = syncForReleaseLocked(mEglDisplay);
 if (err != NO_ERROR) {
 //1. releaseBufferLocked释放Slot, 最终会调用到BufferQueueConsumer的releaseBuffer方法
 releaseBufferLocked(buf, mSlots[buf].mGraphicBuffer,
 mEglDisplay, EGL_NO_SYNC_KHR);
 return err;
 }
}

```

**5\. 总结**

本文对 BufferQueue 的内部实现做了介绍, 结合入队/出对说明了 BufferQueue 内部 Slot 的状态扭转过程, 并介绍了常用的 BufferQueue 封装类, 最后介绍了一个基于 BufferQueue 的例子.
