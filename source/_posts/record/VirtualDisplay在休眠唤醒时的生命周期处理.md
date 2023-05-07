---
title: VirtualDisplay在休眠唤醒时的生命周期处理
urlname: lifecycle-processing-of-VirtualDisplay-when-waking-up-from-sleep
date: 2023/04/18
tags:
  - Framework
  - VirtualDisplay
---

## 问题描述

当启动一个 APP 到 VirtualDisplay 上的时候，APP 的画面会正常显示。当按电源键进行休眠时，在 VirtualDisplay 上的 app 的状态还是为 resumed 状态，不会变成 stopped 状态。

## 休眠时的处理

在设备休眠时，会调用到`RootActivityContainer`的`applySleepTokens`方法，该方法会循环所有的`ActivityDisplay`，并且调用 ActivityDisplay 的 shouldSleep 来判断是否休眠，如果是的话，在循环 ActivityDisplay 中对应的 ActivityStack 时，会调用 ActivityStack 的 goToSleepIfPossible。在 goToSleepIfPossible 中，就会开始调用 startPausingLocked 来 stopped 当前 resumed 状态的 Activity。

shouldSleep 的逻辑是这样的：

```java
    boolean shouldSleep() {
        return (mStacks.isEmpty() || !mAllSleepTokens.isEmpty())
                && (mService.mRunningVoice == null);
    }
```

其中 mStacks 代表这个屏上的 Stack 数量，现在的情况下不会为空，mService.mRunningVoice 主要处理的是音频相关，基本为空，mAllSleepTokens 用于将此堆栈上的活动置于休眠状态的所有令牌（包括 mOffToken）。

测试后发现，VirtualDisplay 上的 mAllSleepTokens 都是为空，所以第一步就是要排查这个 mAllSleepTokens 是如何添加的。

## Token 添加

SleepToken 只有在一个地方添加，就是 RootActivityContainer 中的 createSleepToken，createSleepToken 有两个参数，一个是描述 sleep 的文案，一个是 displayId

```java
ActivityTaskManagerInternal.SleepToken createSleepToken(String tag, int displayId) {
  final ActivityDisplay display = getActivityDisplay(displayId);
  ...
  display.mAllSleepTokens.add(token);
}
```

显然，肯定是这里的 displayId 没有包含 VirtualDisplay 对应的 displayId。

一路找调用，调用的发起者是 PhonwWindowManager 中的 updateScreenOffSleepToken 方法。

```java
    // TODO (multidisplay): Support multiple displays in WindowManagerPolicy.
    private void updateScreenOffSleepToken(boolean acquire) {
            if (acquire) {
                if (mScreenOffSleepToken == null) {
                    mScreenOffSleepToken = mActivityTaskManagerInternal.acquireSleepToken(
                            "ScreenOff", DEFAULT_DISPLAY);
                }
            } else {
                if (mScreenOffSleepToken != null) {
                    mScreenOffSleepToken.release();
                    mScreenOffSleepToken = null;
                }
            }
    }
```

发现调用 acquireSleepToken 时，仅仅用了 DEFAULT_DISPLAY，也就是主屏幕。而且这个方法还有个 TODO，这个 TODO 在 Android 13 的版本还在，说明 Google 官方还没处理这个 multidisplay 的情况。

## multidisplay 支持

所以就需要修改 updateScreenOffSleepToken 的逻辑，让它能够支持多 Display 的情况。因为这里的 mScreenOffSleepToken 是一个对象，如果是多屏幕的情况，就需要有多个 mScreenOffSleepToken，所以需要一个 Map 类型来存储，Map 的 Key 为 DisplayId,Value 为 SleepToken。

其次需要循环所有的 Display，然后修改传给 acquireSleepToken 的参数，代码如下：

```java
    Map<Integer,SleepToken> mSleepTokenMap;

    private void updateScreenOffSleepToken(boolean acquire) {
        for (int i = 0; i < mDisplayManager.getDisplays().length; i++) {
            Display display = mDisplayManager.getDisplays()[i];
            if (acquire) {
                if (mSleepTokenMap.get(display.getDisplayId()) == null) {
                    SleepToken sleepToken = mActivityTaskManagerInternal.acquireSleepToken(
                            "ScreenOff", display.getDisplayId());
                    mSleepTokenMap.put(display.getDisplayId(), sleepToken);
                }
            } else {
                if (mSleepTokenMap.get(display.getDisplayId()) != null) {
                    mSleepTokenMap.get(display.getDisplayId()).release();
                    mSleepTokenMap.put(display.getDisplayId(), null);
                }
            }
        }
    }
```

这样支持了多屏幕的休眠唤醒问题。
