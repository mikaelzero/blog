---
title: SurfaceFlinger的主动刷新
date: 2022/06/06
tags:
  - android
  - framework
  - SurfaceFlinger
---

**Abstract**

在 SurfaceFlinger 中加入了对传感器数据的融合算法从而实现 AR 眼镜的头部跟踪功能之后, 发现只有使用 OpenGL 连续渲染的 APP 在头部转动时才会呈现流畅的画面, 而对于普通的 APP, 画面是卡顿的. 因此, 需要在 SurfaceFlinger 中实现主动刷新的功能.

研究发现, SurfaceFlinger 在收到来自硬件层的 Vsync 事件之后, 不是每次都会调用 SurfaceFlinger::handleMessageInvalidate()的. 而 SurfaceFlinger 中处理 Vsync 事件涉及到的类有：DispSyncThread, DispSync,SurfaceFlinger::DispSyncSource, EventThread, EventThread::Connection, MessageQueue. 本文首先研究这些类之间的关系, 以及自 Vysnc 事件发生后这些类的处理流程. 最终, 通过修改 EventThread::Connection 中的成员 count 解决了每个 Vsync 事件都会调用 SurfaceFlinger::handleMessageInvalidate()的问题.

接下来要解决的是从 SurfaceFlinger::handleMessageInvalidate()到最终绘制的问题. 研究发现, handleMessageInvalidate()方法经常返回 false. 通过修改强制调用 handleMessageRefresh()之后, 又需要修改 SurfaceFlinger::setUpHWComposer()中的 mMustRecompose 变量, 使得每一帧都能真正执行绘制操作.

**研究 EventThread.cpp 中的 waitForEvent()**

SurfaceFlinger 中处理来自 DispSyncThread 的 VSYNC 事件的线程是 EventThread. 而 EventThread 线程最耗时也最重要的方法是 waitForEvent().

**Subject**

（1）EventThread::onVSyncEvent()是否每一帧都会调用到？还是说必须要在 enableVSyncLocked()之后？

（2）EventThread::waitForEvent()中的 connection->count 怎么会变成-1 了？

（3）EventThread::waitForEvent()中如何从 enableVSyncLocked()转到 disableVSyncLocked()?

**Process**

**DispSyncSource 的 setVSyncEnabled()方法的作用**

DispSyncSource 中的 setVSyncEnabled()方法是唯一调用了 DispSync::addEventListener()的地方：

```cpp
status_t err = mDispSync->addEventListener(mPhaseOffset,
static_cast<DispSync::Callback\*>(this));
```

只有执行了该方法, 才会使 DispSyncThread 的成员 mEventListeners 有内容. 进而使得 DispSyncThread::threadLoop 执行 gatherCallbackInvocationsLocked()时, 返回的`Vector<CallbackInvocation> callbackInvocations`的大小大于 0, 最终在执行 fireCallbackInvocations 时, 能调用到 DispSyncSource 的 onDispSyncEvent()方法.

而在 DispSyncSource 的 onDispSyncEvent()方法中可以看到, 当 DispSyncSource 的 mCallback 非空时, 会调用到 EventThread 的 onVSyncEvent()方法：

```cpp
virtual void onDispSyncEvent(nsecs_t when) {
sp<VSyncSource::Callback> callback;
{
Mutex::Autolock lock(mMutex);
callback = mCallback;
...
}

if (callback != NULL) {
callback->onVSyncEvent(when);
}
}
```

因此, 如果没有调用这个 setVSyncEnabled(), 那么源自硬件的 Vsync 是不会传到 EventThread 的.

SurfaceFlinger.cpp 的子类 DispSyncSource 中的 setVSyncEnabled(bool enable)是在哪里被调用的？

**从 SurfaceFlinger.cpp 的 init()开始**

在 SurfaceFlinger.cpp 的 init()方法中, 通过如下的代码初始化了 DispSyncSource 和 EventThread, 并将 mSFEventThread 关联到成员变量 MessageQueue mEventQueue 中.

```cpp
sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
sPropertiesState.mAppVSyncOffset, true, "app");
mEventThread = new EventThread(vsyncSrc);
sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
sPropertiesState.mSfVSyncOffset, true, "sf");
mSFEventThread = new EventThread(sfVsyncSrc);
mEventQueue.setEventThread(mSFEventThread);
```

在 EventThread.cpp 中可以看到当 EventThread 的实例创建时, 在其 onFirstRef()中会开启线程：

```cpp
void EventThread::onFirstRef() {
run("EventThread", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
}
```

注意此时 DispSyncSource 的 setVSyncEnabled()还没有被调用到.

而在 MessageQueue::setEventThread()中, 会创建 EventThread::Connection

```cpp
void MessageQueue::setEventThread(const sp<EventThread> &eventThread)
{
    mEventThread = eventThread;
    mEvents = eventThread->createEventConnection();
    mEventTube = mEvents->getDataChannel();
    mLooper->addFd(mEventTube->getFd(), 0, Looper::EVENT_INPUT,
                   MessageQueue::cb_eventReceiver, this);
}
sp<EventThread::Connection> EventThread::createEventConnection() const
{
    return new Connection(const_cast<EventThread\*>(this));
}
EventThread::Connection::Connection(
    const sp<EventThread> &eventThread)
    : count(-1), mEventThread(eventThread), mChannel(new BitTube())
```

注意 count 的初始值是-1.

在 EventThread::Connection 的 onFirstConnection 中, 会将 Connection 的实例传给 EventThread, 并加到其成员变量 mDisplayEventConnections：

```cpp
void EventThread::Connection::onFirstRef() {
// NOTE: mEventThread doesn't hold a strong reference on us
mEventThread->registerDisplayEventConnection(this);
}
status_t EventThread::registerDisplayEventConnection(
const sp<EventThread::Connection>& connection) {
Mutex::Autolock \_l(mLock);
mDisplayEventConnections.add(connection);
mCondition.broadcast();
return NO_ERROR;
}
```

**第一次执行 EventThread::waitForEvent()**

当 EventThread::threadLoop()运行起来后, 首先会执行 EventThread::waitForEvent()方法：

```cpp
// This will return when (1) a vsync event has been received, and (2) there was
// at least one connection interested in receiving it when we started waiting.
Vector<sp<EventThread::Connection>> EventThread::waitForEvent(
    DisplayEventReceiver::Event\*event)
{
    Mutex::Autolock \_l(mLock);
    Vector<sp<EventThread::Connection>> signalConnections;

    do
    {
        bool eventPending = false;
        bool waitForVSync = false;

        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        for (int32_t i = 0; i < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES; i++)
        {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp)
            {
                // we have a vsync event to dispatch
                \*event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }

        if (!timestamp)
        {
            // no vsync event, see if there are some other event
            eventPending = !mPendingEvents.isEmpty();
            if (eventPending)
            {
                // we have some other event to dispatch
                \*event = mPendingEvents[0];
                mPendingEvents.removeAt(0);
            }
        }

        // find out connections waiting for events
        size_t count = mDisplayEventConnections.size();
        for (size_t i = 0; i < count; i++)
        {
            sp<Connection> connection(mDisplayEventConnections[i].promote());
            if (connection != NULL)
            {
                bool added = false;
                if (connection->count >= 0)
                {
                    // we need vsync events because at least
                    // one connection is waiting for it
                    waitForVSync = true;
                    if (timestamp)
                    {
                        // we consume the event only if it's time
                        // (ie: we received a vsync event)
                        if (connection->count == 0)
                        {
                            // fired this time around
                            connection->count = -1;
                            signalConnections.add(connection);
                            added = true;
                        }
                        else if (connection->count == 1 ||
                                 (vsyncCount % connection->count) == 0)
                        {
                            // continuous event, and time to report it
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }

                if (eventPending && !timestamp && !added)
                {
                    signalConnections.add(connection);
                }
            }
            else
            {
                mDisplayEventConnections.removeAt(i);
                --i;
                --count;
            }
        }
        if (timestamp && !waitForVSync)
        {
            disableVSyncLocked();
        }
        else if (!timestamp && waitForVSync)
        {
            enableVSyncLocked();
        }
        if (!timestamp && !eventPending)
        {
            if (waitForVSync)
            {
                bool softwareSync = mUseSoftwareVSync;
                nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
                if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT)
                {
                    if (!softwareSync)
                    {
                        ALOGW("Timed out waiting for hw vsync; faking it");
                    }
                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                    mVSyncEvent[0].vsync.count++;
                }
            }
            else
            {
                mCondition.wait(mLock);
            }
        }
    } while (signalConnections.isEmpty());
    return signalConnections;
}
```

初次运行 waitForEvent()时, 由于 EventThread::onVSyncEvent()和 EventThread::onHotplugReceived()方法都没有被调用. 因此经过循环：

for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {

之后, timestamp 的值还是 0. 同样地, mPendingEvents 为空, 因此 eventPending 的值为 false.

前面提到, 在 Connection 的 onFirstRef()方法中, 会将自己加到 EventThread 类的 mDisplayEventConnections 中. 因此在这里, mDisplayEventConnections.size()的值为 1. 而 EventThread::Connection 的构造方法中, 其成员 count 的初始值为-1. 因此代码块：

```cpp
if (connection->count >= 0) {
...
}
```

不会执行到, 进而 waitForVSync 的值还是 false. 代码块：

```cpp
if (timestamp && !waitForVSync) {
disableVSyncLocked();
} else if (!timestamp && waitForVSync) {
enableVSyncLocked();
}
```

也不会被执行到.

而条件：

if (!timestamp && !eventPending) {

是成立的, 因此会执行代码：

mCondition.wait(mLock);

EventThread 线程会一直等待下去. （这里没有 Timeout）

EventThread::Connection 类是 BnDisplayEventConnection 的服务端, 其客户端 IDisplayEventConnection 开放出来的接口只有三个：

```cpp
class IDisplayEventConnection : public IInterface
{
public:

DECLARE_META_INTERFACE(DisplayEventConnection);

/\* \* getDataChannel() returns a BitTube where to receive the events from
\*/
virtual sp<BitTube> getDataChannel() const = 0;

/\* \* setVsyncRate() sets the vsync event delivery rate. A value of \* 1 returns every vsync events. A value of 2 returns every other events, \* etc... a value of 0 returns no event unless requestNextVsync() has \* been called.
\*/
virtual void setVsyncRate(uint32_t count) = 0;

/\* \* requestNextVsync() schedules the next vsync event. It has no effect \* if the vsync rate is > 0.
\*/
virtual void requestNextVsync() = 0; // asynchronous
};
```

其中用于控制 EventThread 的接口是 setVsyncRate()和 requestNextVsync().

需要注意这里的注释：

（1）传入 setVsyncRate()的参数 count 的意义是 vsync rate. 它指定了每 N 个 VSYNC 事件进行一次传递. 当传入的值是 1 时, 就是每次 VSYNC 事件都进行传递；当传入值是 2 时, 就是每两次 VSYNC 事件才进行一次传递；以此类推.

（2）requestNextVsync()用于 VSYNC 事件单次的传递, 而它也只是在 vsync rate 小于 0 时起作用.

再来看 EventThread::Connection 中这两个接口的实现：

（1）EventThread::setVsyncRate()

```cpp
void EventThread::setVsyncRate(uint32_t count,
const sp<EventThread::Connection>& connection) {
if (int32_t(count) >= 0) { // server must protect against bad params
Mutex::Autolock \_l(mLock);
const int32_t new_count = (count == 0) ? -1 : count;
if (connection->count != new_count) {
connection->count = new_count;
mCondition.broadcast();
}
}
}
```

（2）EventThread::requestNextVsync()

```cpp
void EventThread::requestNextVsync(
const sp<EventThread::Connection>& connection) {
Mutex::Autolock \_l(mLock);
if (connection->count < 0) {
connection->count = 0;
mCondition.broadcast();
}
}
```

这两个方法都会设置 Connection 类中的 count 成员变量（即 vsync rate）. 先研究 requestNextVsync()被调用后 waitForEvent()的情况. setVsyncRate()会在后面的小节《Extension：如果 Connection 的成员 count 的初始值为 1 会怎么样？》中涉及.

**EventThread::requestNextVsync()之后执行 waitForEvent()**

EventThread::requestNextVsync()会把 EventThread::Connection 的 count 的值赋为 0 并唤醒 EventThread 线程. 由于 waitForEvent()方法的主体是一个 do...while(signalConnections.isEmpty())循环, 而 signalConnections 依然为空, 所以当线程被唤醒之后, 还是从 waitForEvent()继续.

此时 timestamp 依然为 0, eventPending 也依然为 false. 但由于 connection->count 的值变成了 0, 因此会执行如下的代码块：

```cpp
if (connection->count >= 0) {
// we need vsync events because at least
// one connection is waiting for it
waitForVSync = true;

if (timestamp) {
// we consume the event only if it's time
// (ie: we received a vsync event)
if (connection->count == 0) {
// fired this time around
connection->count = -1;
signalConnections.add(connection);
added = true;
} else if (connection->count == 1 ||
(vsyncCount % connection->count) == 0) {
// continuous event, and time to report it
signalConnections.add(connection);
added = true;
}
}
}
```

可以看到此时 waitForVSync 的值变成了 true. 由于 timestamp 值为 0 而 waitForVSync 的值为 true, 因此满足条件：

else if (!timestamp && waitForVSync) {

从而会执行 EventThread::enableVSyncLocked().

接下来由于条件:

if (!timestamp && !eventPending) {

和

if (waitForVSync) {

都满足, 因此会执行：

```cpp
bool softwareSync = mUseSoftwareVSync;
nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
if (!softwareSync) {
ALOGW("Timed out waiting for hw vsync; faking it");
}
mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
mVSyncEvent[0].vsync.count++;
}
```

由于 mUseSoftwareVSync 的值为 false, 因此这里设置 timeout 为 1 秒钟并执行：

mCondition.waitRelative(mLock, timeout)

这里有三种情况：

（1）在超时之前又有 EventThread::requestNextVsync()被调用, 从 EventThread::requestNextVsync()的实现可以看到, 由于此时 EventThread::Connection 的 count 成员的值依然为 0, 那么该方法是不能唤醒 EventThread 线程的.

（2）在超时之前发生了 VSYNC 事件, 下面的小节《Vsync 发生后》会研究这种情况.

（3）发生了超时. 此时会生成一个模拟的 VSYNC 信号. 这样当来到新的 do...while()循环中时, 局部变量 timestamp 就会有值, 因此会执行如下的代码块：

```cpp
waitForVSync = true;
if (timestamp) {
// we consume the event only if it's time
// (ie: we received a vsync event)
if (connection->count == 0) {
// fired this time around
connection->count = -1;
signalConnections.add(connection);
added = true;
} else if (connection->count == 1 ||
(vsyncCount % connection->count) == 0) {
// continuous event, and time to report it
signalConnections.add(connection);
added = true;
}
}
```

这里会把 count 变量重置为-1, 并向 signalConnections 中添加 Connection 的实例.

接下来的条件：

```cpp
if (timestamp && !waitForVSync) {
else if (!timestamp && waitForVSync) {
if (!timestamp && !eventPending) {
```

都不满足, 因此直接跳出 do...while()循环并且返回 signalConnections. 然后回到 EventThread::threadLoop(), 执行 Connection::postEvent()方法, 向 Connection 的成员变量

```cpp
sp<BitTube> const mChannel;
status_t EventThread::Connection::postEvent(
const DisplayEventReceiver::Event& event) {
...
ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);
return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

发送消息.

由于 EventThread 继承自 system/core 中的 Thread 类, 因此当 postEvent()方法执行完, 然后 threadLoop()方法返回 true 之后, 会继续执行 threadLoop()方法. 此时 waitForEvent()方法中, timestamp 的值为 0, count 的值为-1, 进而 waitForVSync 的值为 false, 那么会执行到：

mCondition.wait(mLock);

EventThread 会一直等待下去. 直到下一个 requestNextVsync()发生.

**PS: EventThread::enableVSyncLocked()执行的操作**

上一节由于 requestNextVsync()使得 enableVSyncLocked()被调用：

```cpp
void EventThread::enableVSyncLocked() {
if (!mUseSoftwareVSync) {
// never enable h/w VSYNC when screen is off
if (!mVsyncEnabled) {
mVsyncEnabled = true;
mVSyncSource->setCallback(static_cast<VSyncSource::Callback\*>(this));
mVSyncSource->setVSyncEnabled(true);
}
}
mDebugVsyncEnabled = true;
sendVsyncHintOnLocked();
}
```

可以看到, 通过 EventThread::enableVSyncLocked()指定了 DispSyncSource 的 mCallback 为当前的 mEventThread 实例, 同时调用了 DispSyncSource 的 setVSyncEnabled()方法.

**Vsync 发生后**

当 setVSyncEnable()方法执行后, Vsync 事件就可以从 DispSyncThread 起, 顺序执行：

\ DispSyncThread 的 fireCallbackInvocations()

\2. DispSyncSource 的 onDispSyncEvent()

\3. EventThread 的 onVSyncEvent()

在 EventThread::onVSyncEvent()中, 设置了 mVSyncEvent 并唤醒了 EventThread 线程. 此时再执行 waitForEvent()时, timestamp 有值, 而 Connection 的 count 值不确定：

（1）count 的值为-1. 说明在 onVSyncEvent()被调用之前已经发生了 TIMEOUT, EventThread 处于一直等待的状态. 此时满足条件：

if (timestamp && !waitForVSync) {

因此会执行 disableVSyncLocked()：

```cpp
void EventThread::disableVSyncLocked() {
if (mVsyncEnabled) {
mVsyncEnabled = false;
mVSyncSource->setVSyncEnabled(false);
mDebugVsyncEnabled = false;
}
}
```

该方法调用了 DispSyncSource 的 setVSyncEnabled(false)方法, 最终会执行 DispSyncThread 的 removeEventListener()方法, 从 mEventListeners 中移除了 DispSyncSource 这个 Callback, 那么 DispSyncThread::fireCallbackInvocations()方法是不会传给 EventThread 了.

继续执行 waitForEvent(), 由于 signalConnections 内容还是为空, 因此还是在 do...while()的循环里面. 再次开始循环时, timestamp 的值已经是 0. 这是因为上一个循环中, 当给 timestamp 赋值之后, 会设置 mVSyncEvent[i].header.timestamp 的值为 0.

```cpp
timestamp = mVSyncEvent[i].header.timestamp;
if (timestamp) {
...
mVSyncEvent[i].header.timestamp = 0;
```

此时 waitForEvent()最终会走到：

mCondition.wait(mLock);

从而 EventThread 线程一直等待下去.

（2）count 的值为 0. 说明 waitForEvent()此时的状态是停留在：

mCondition.waitRelative(mLock, timeout)

并且还没有超时. 当本线程被唤醒之后, 会重新执行 do...while()循环. 此时 timestamp 的值非 0, 而 count 的值为 0, 此时会把 waitForVSync 赋为 true, count 的值赋为-1, 并把当前 Connection 的实例添加到 signalConnections 中. 这样 signalConnections 非空, 走出 do...while()循环. 然后 EventThread::threadLoop()调用 postEvent()方法. 接下来再次执行 waitForEvent(), 最终会执行：

mCondition.wait(mLock);

EventThread 会一直等待下去. 直到下一个 requestNextVsync()发生.

**小结**

\ 从 SurfaceFlinger 的 init()发生到第一次执行 EventThread 的 waitForEvent(), EventThread 线程会执行到 mCondition.wait()从而一直等待. 由于此时尚未 enableVSyncLocked(), 那么 EventThread 是不会执行 onVSyncEvent()方法的.

\2. requestNextVsync()发生后, 会执行 enableVSyncLocked(), waitForEvent()会执行到 mCondition.waitRelative(), 接下来有三种情况：

2.1 requestNextVsync()比 onVSyncEvent()先发生：此时无变化.

2.2 onVSyncEvent()比超时先发生：waitForEvent()会从 do...while()中跳出来, 并执行 EventThread::postEvent()方法. 再次执行 waitForEvent()时, 该线程会执行到 mCondition.wait()一直等待下去.

2.3 onVSyncEvent()比超时晚发生：当超时发生时, waitForEvent()会从 do...while()中跳出来, 并执行 EventThread::postEvent()方法. 再次执行 waitForEvent()时, 该线程会执行到 mCondition.wait 一直等待下去. 接下来 onVSyncEvent()发生时, 会执行 disableVSyncLocked(), 从此阻止 onVSyncEvent(). 然后 waitForEvent()会执行到 mCondition.wait()一直等待下去.

\3. EventThread 线程等待下一个 requestNextVsync()的发生.

PS：由于 onVSyncEvent()和 requestNextVsync()方法中都有自动锁：

Mutex::Autolock \_l(mLock);

因此是不会发生冲突的.

**如果 Connection 的成员 count 的初始值为 1 会怎么样？**

\ 第一次执行 waitForEvent()时, 会执行 enableVSyncLocked(). 而无须等待 requestNextVsync()的发生. 然后 waitForEvent()会执行到 mCondition.waitRelative(). 接下来有种情况：

1 requestNextVsync()发生了：无变化.

2 onVSyncEvent()比超时先发生：waitForEvent()会从 do...while()中跳出来, 并执行 EventThread::postEvent()方法. 再次执行 waitForEvent()时, 会执行到 mCondition.waitRelative().

3 onVSyncEvent()比超时晚发生：当超时发生时, waitForEvent()会从 do...while()中跳出来, 并执行 EventThread::postEvent()方法. 再次执行 waitForEvent()时, 会执行到 mCondition.waitRelative(), 而非 mCondition.wait(). 接下来 onVSyncEvent()发生时, 情况与 2 相同.

可以看到, 当 count 的值为 1 时, 产生的效果是消除了对 requestNextVsync()方法的依赖.

重启系统之后, 可以看到每次 VSYNC 事件到来之后, 都能最终调用到 SurfaceFlinger::handleMessageInvalidate().

在前面的小节《第一次执行 EventThread::waitForEvent()》中提到, IDisplayEventConnection 的接口方法 setVsyncRate()可以用来设置 Connection 的 count 成员. 纵观整个 frameworks, 该方法都没有被调用过. 我的修改是在 MessageQueue.cpp 的 setEventThread()方法中创建完 sp<IDisplayEventConnection> mEvents 后, 调用 setVsyncRate()方法：

```cpp
void MessageQueue::setEventThread(const sp<EventThread>& eventThread)
{
mEventThread = eventThread;
mEvents = eventThread->createEventConnection();
// willie change start 2016-11-2
mEvents->setVsyncRate(1);
// willie change end 2016-11-2
mEventTube = mEvents->getDataChannel();
mLooper->addFd(mEventTube->getFd(), 0, Looper::EVENT_INPUT,
MessageQueue::cb_eventReceiver, this);
}
```
