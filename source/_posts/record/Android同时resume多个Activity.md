---
title: Android同时resume多个Activity
urlname: resume-multiple-activities-simultaneously-after-waking_up-from-sleep.
date: 2023/03/28
tags:
  - Framework
  - AMS
---

> 之前在 [surfaceflinger 修改 layer 层级](/posts/surfaceflinger_change_layer_level) 文章中实现了底部的 Layer 能否显示在顶部，但是在休眠唤醒的时候会出现问题,因此这篇文章是 [surfaceflinger 修改 layer 层级](/posts/surfaceflinger_change_layer_level)的后续完善版本。

## 场景

一个 Activity A 和 Activity B,A 做为 Home，也就是桌面，当启动 B 的时候，A 同样能够显示在 B 的上面。A 其实是一个半透明的页面。

有人想到的方案可能是多窗口，但其实这和多窗口不是一回事，因为我们的架构设计中，都是虚拟屏幕。

当然一开始最快能想到的方案自然就是直接再次把 A 拉起来，放在栈顶，但是这样就会导致 B 会 pause，由于 B 大多数都是 3D 应用，一旦 pause，大部分程序是直接不渲染的，也就是黑屏。

同样的，想解决这个黑屏问题也可以直接不 pause B，但是休眠唤醒的时候就会有问题，唤醒的时候只会唤醒栈顶的 Activity，也就是 A，所以还必须同时唤醒 B 才能达到效果。并且改动拉起部分代码不少，所以选择了另外一个方案，逻辑其实还是类似的。

首先 A 还是 Launcher，B 还是普通的 3D 应用，正常启动 A，然后启动 B，只是在 pause 的时候，需要让 A 不能够 pause 且 A 的画面能够在顶层，这个在 [surfaceflinger 修改 layer 层级](/posts/surfaceflinger_change_layer_level) 一文中已经解释过，这个解决了第一个问题。

其次还是在休眠唤醒上，在 A 之上启动了一个 B 之后，栈关系是 B 在 A 之上，所以休眠唤醒时，还是只会唤醒 B，所以需要处理的是，同时唤醒 A。

## resume 多个 Activity

第一个问题就是，在启动了 B 之后，A 如何还保持着 resume 状态。由于我们的 A 是一个 launcher，一个 Home 类型的 Activity，并且是一个 Unity 程序，所以在处理过 Layer 的前提之上，还需要保持 A 为 resume 状态，否则 Unity 程序就会结束 VR 模式。

在启动一个新的 Activity 时，会在对应的 ActivityStack 中调用`resumeTopActivityInnerLocked`,在`resumeTopActivityInnerLocked`流程中，会调用

```java
boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
```

`pauseBackStacks`的作用就是 pause 掉当前正在 resume 的 Activity。在我们的场景下，启动 B 的时候，需要 pause 的就是 A。所以在这个地方，直接过滤掉我们的 A 即可，下面是修改代码：

```java
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
        boolean someActivityPaused = false;
        for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = mStacks.get(stackNdx);
            final ActivityRecord resumedActivity = stack.getResumedActivity();
            if (resumedActivity != null
                    && (stack.getVisibility(resuming) != STACK_VISIBILITY_VISIBLE
                        || !stack.isFocusable())) {
                //改动在这 判断是否为Launcher
                if (resumedActivity.isActivityTypeHome()){
                    return false;
                }
                someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                        dontWait);
            }
        }
        return someActivityPaused;
    }
```

这样就能让我们的 A 同时保持 resume 状态。

## 唤醒时 resume 多个 Activity

唤醒的时候，默认是 resume 在 top 的 Activity。比如我们现在的栈是 A -> B，那么休眠唤醒时，默认就是 resume B，所以我们需要同时 resume A。

在休眠唤醒的时候，流程如下:

```text
唤醒的时候应用resume的流程
ActivityStackSupervisor.java
SleepTokenImpl.release
-->removeSleepTokenLocked
-->updateSleepIfNeededLocked
-->applySleepTokens
-->resumeFocusedStacksTopActivities
-->resumeTopActivityUncheckedLocked
-->resumeTopActivityInnerLocked
```

在`applySleepTokens`流程中，会调用`resumeFocusedStacksTopActivities`来 resume 起在顶部的 Activity，那么我们需要在它之前，先 resume 起我们的 A，代码如下：

```java
ActivityRecord tmpTop = stack.topRunningActivityLocked();
final ActivityRecord r = getActivityDisplay(DEFAULT_DISPLAY).getHomeActivity();
if (r != null && !r.finishing) {
    boolean result = r.getActivityStack().resumeTopActivityUncheckedLocked(r, null);
}

resumeFocusedStacksTopActivities()
```

并且还需要在 resumeTopActivityUncheckedLocked 方法的 shouldBeVisible 条件中加入一个判断，因为如果不加判断，A 就会直接调用 completeResumeLocked 导致不走 resume

```java
            if (shouldBeVisible(next)||next.isActivityTypeHome()) {
                notUpdated = !mRootActivityContainer.ensureVisibilityAndConfig(next, mDisplayId,
                        true /* markFrozenIfConfigChanged */, false /* deferResume */);
            }
```

## Layer 异常

上述操作改完后，发现唤醒时，A 无法显示，只能显示 B，且通过下面命令查看 Activity 的状态时，发现都是 resume 的状态。

```bash
adb shell dumpsys activity activities | grep -E 'Stack|TaskRecord|Hist|Display|state'
```

所以问题不会出现在生命周期上，于是排查下 Layer 是否有什么问题。这里的 Layer 就是指使用 `adb shell dumps SurfaceFlinger` 时的 Layer，也是指每一个 app 对应的 Layer。

正常情况下，一个 Unity 程序的 Layer 有两个，一个是 Unity 程序的壳，一个是使用 SurfaceView 渲染的 Layer,如下所示：

```text
  - Output Layer 0x7564882000 (Composition layer 0x7564786898) (com.mikaelzero.dock/UnityActivity#0)
        Region visibleRegion (this=0x7564882028, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=1
      hwc: layer=0x0812 composition=DEVICE (2)
  - Output Layer 0x7560edde00 (Composition layer 0x7571e40598) (SurfaceView - com.mikaelzero.dock/UnityActivity#0)
        Region visibleRegion (this=0x7560edde28, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=2
      hwc: layer=0x0813 composition=DEVICE (2)
```

可以看到 Layer 的名字，一个是 `com.mikaelzero.dock/UnityActivity`，一个是 `SurfaceView - com.mikaelzero.dock/UnityActivity`，只有 `SurfaceView - com.mikaelzero.dock/UnityActivity` 这个 Layer 输出，Unity 程序才能正常显示。

Layer 输出的意思就是指：指定的 Layer 出现在使用 `adb shell dumps SurfaceFlinger` 时,Output Layer 的部分。Output Layer 出现的 Layer 就是最终显示在显示屏上的 Layer。

而当出现问题的时候，日志是这样的：

```text
   1 Layer  - Output Layer 0x6f6890d000 (Composition layer 0x6ffe117698) (com.mikaelzero.dock/UnityActivity#0)
        Region visibleRegion (this=0x6f6890d028, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=0
      hwc: layer=0x087 composition=DEVICE (2)
```

那么，就是少了 SurfaceView - com.mikaelzero.dock/UnityActivity 这个 Layer。那么问题就应该出现在这。

在排查这个问题时，耗费了非常多的时间，因为问题给出的线索很少，一开始所有的状态都是正确的，唯独这个 Layer 不显示。排查了 WMS 处理窗口的部分时，也没发现特别之处，所以开始怀疑问题点并不在系统。

而且我对 Unity 程序是信心满满，一直没怀疑 Unity 那边有什么问题，虽然确实不是 Unity 的问题，但是也是从 Unity 那部分排查得出的结果。

后来在 Unity 程序的 Log 中，发现了一些端倪，也就是在唤醒的时候，Unity 的日志居然一点也不输出(在做这个需求的时候，自己在 adb 时都是完全过滤掉了 Unity 的日志，所以一开始都没发现)。

不输出日志，说明在唤醒时 Unity 就是处于不工作的状态，查看代码发现 OnApplicationPause 也不回调，按理来说，如果 Activity resume 了，那么就会回调 OnApplicationPause，且参数为 false，这是 Unity 自带的函数，所以应该也不会有什么问题。

直到我在 Unity 导出的 Android 工程的项目中 Acitvity 的生命周期打印一些日志，顺便在这个函数打印了一下：

```java
    @Override public void onWindowFocusChanged(boolean hasFocus)
    {
        super.onWindowFocusChanged(hasFocus);
        mUnityPlayer.windowFocusChanged(hasFocus);
    }
```

发现当启动 B 的时候，A 会回调 onWindowFocusChanged，且参数为 false，而唤醒时却没有回调。

在测试了正常流程下的 Unity 程序，只有在有 focus 的情况下，才会回调 OnApplicationPause。

onWindowFocusChanged 这个函数，是在焦点变换时调用的，而且是在 resume 之后调用，当我存在 A 时，且调用了 B，A 的焦点变成了 false，这是正常的，而且唤醒的时候，焦点应该也是只有栈顶的 Acitivty 才会获得，由于我们的 A 并不是栈顶，所以显然 A 并不会获得栈顶。既然如此，那就手动给 A 获取焦点，这个函数仅仅是一个回调而已，并不会影响系统的逻辑功能。

所以，最后一步，就是在 Activity 的 resume 中，如果是 Home 类型的 Acitivty，就手动调用 onWindowFocusChanged 方法。

```java
public void performResume(...){
    ......
    onWindowFocusChanged(true);
}
```

到此为止，所有的功能就基本完善了。
