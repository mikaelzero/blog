---
title: VirtualDisplay中APP的生命周期处理
urlname: life-cycle-processing-of-app-in-VirtualDisplay
date: 2023/04/20
tags:
  - VirtualDisplay
---

当我们想将一个 Activity 或者是 APP 切换到后台时，可以使用 moveStack 的方法或者直接回到 Home，这样 Activity 就会从 resumed 状态变更为 stopped 状态。但是在虚拟屏上的逻辑可不是这样，当我们切换到 Home，只会将 Home 所在的 Display 的 Stack 进行切换，而 Home 类型的 ActivityStack 只会在默认屏幕上。因此，如果想在主屏幕上切换到 Home 时，同时也想把虚拟屏上的生命周期进行处理，就需要自己手动进行切换。

VirtualDisplay 中有一个方法`setDisplayState`，是用来设置 on/off 的状态，参数为 boolean

```java
    public void setDisplayState(boolean isOn) {
        if (mToken != null) {
            mGlobal.setVirtualDisplayState(mToken, isOn);
        }
    }
```

通过该方法可以切换 VirtualDisplay 的显示状态，从而来暂停或者恢复 VirtualDisplay 中的 Activity 的状态。接下来看看它的原理是怎样的。

该函数的调用链如下:

```c++
DisplayManagerGlobal::setVirtualDisplayState
DMS::setVirtualDisplayStateInternal
VirtualDisplayAdapter::setVirtualDisplayStateLocked
VirtualDisplayDevice::setDisplayState
DisplayAdapter::sendDisplayDeviceEventLocked(this, DISPLAY_DEVICE_EVENT_CHANGED)
DMS::handleDisplayDeviceChanged
```

handleDisplayDeviceChanged 中会发送一个`EVENT_DISPLAY_CHANGED` 消息，会回调所有的监听者，监听的方法为 onDisplayChanged。RootActivityContainer 注册了该 Linstener，在 RootActivityContainer 的 onDisplayChanged 中会循环调用 ActivityDisplay 的 onDisplayChanged 方法，如下：

```java
    void onDisplayChanged() {
        // The window policy is responsible for stopping activities on the default display.
        final int displayId = mDisplay.getDisplayId();
        if (displayId != DEFAULT_DISPLAY) {
            final int displayState = mDisplay.getState();
            if (displayState == Display.STATE_OFF && mOffToken == null) {
                mOffToken = mService.acquireSleepToken("Display-off", displayId);
            } else if (displayState == Display.STATE_ON && mOffToken != null) {
                mOffToken.release();
                mOffToken = null;
            }
        }
        ......
    }
```

如果 displayState == Display.STATE_OFF 成立，则会调用 ATMS(ActivityTaskManagerService) 的 acquireSleepToken，该方法中会创建需要 sleep 的所有 Token，之后在 updateSleepIfNeededLocked 中，调用 applySleepTokens

```java
mRootActivityContainer.applySleepTokens(true /* applyToStacks */);
```

applySleepTokens 会根据上面生成的 token 列表中，来判断是否需要休眠，如果是的话，则会调用对应 ActivityStack 的 goToSleepIfPossible，该方法就会将 ActivityStack 下的所有 Activity 切换为 stopped 状态。

所以，虚拟屏的最终处理其实和主屏幕类似，也是通过屏幕的状态更改来做处理。