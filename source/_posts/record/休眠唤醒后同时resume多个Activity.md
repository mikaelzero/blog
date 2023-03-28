---
title: 休眠唤醒后同时resume多个Activity
urlname: resume-multiple-activities-simultaneously-after-waking_up-from-sleep.
date: 2023/03/24
tags:
  - android
  - framework
  - AMS
---

> 之前在 [surfaceflinger 修改 layer 层级](/posts/surfaceflinger_change_layer_level) 文章中实现了底部的 Layer 能否显示在顶部，但是在休眠唤醒的时候会出现问题。

## 场景

一个 Activity A 和 Activity B,A 做为 Home，也就是桌面，当启动 B 的时候，A 同样能够显示在 B 的上面。A 其实是一个半透明的页面。

当然一开始最快能想到的方案自然就是直接再次把 A 拉起来，放在栈顶，但是这样就会导致 B 会 pause，由于 B 大多数都是 3D 应用，一旦 pause，大部分程序是直接不渲染的，也就是黑屏。同样的，想解决这个问题也可以直接不 pause B，但是休眠唤醒的时候就会有问题，唤醒的时候只会唤醒栈顶的 Activity，也就是 A，所以还必须同时唤醒 B 才能达到效果。并且改动拉起部分代码不少，所以选择了另外一个方案，逻辑是类似的。

首先 A 还是 Launcher，B 还是普通的 3D 应用，正常启动 A，然后启动 B，只是在 pause 的时候，需要让 A 不能够 pause 且 A 的画面能够在顶层，这个在 [surfaceflinger 修改 layer 层级](/posts/surfaceflinger_change_layer_level) 一文中已经解释过，这个解决了第一个问题。

其次还是在休眠唤醒上，在 A 之上启动了一个 B 之后，栈关系是 B 在 A 之上，所以休眠唤醒时，还是只会唤醒 B，所以需要处理的是，同时唤醒 A。

## resume 多个 Activity

第一个问题就是，在启动了 B 之后，A 如何还保持着 resume 状态。由于我们的 A 是一个 launcher，一个 Home 类型的 Activity，并且是一个 Unity 程序，所以在处理过 Layer 的前提之上，还需要保持 A 为 resume 状态，否则 Unity 程序就会结束 VR 模式。

在启动一个新的 Activity 时，会在对应的 ActivityStack 中调用`resumeTopActivityInnerLocked`,在`resumeTopActivityInnerLocked`流程中，会调用

```java
boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
```

`pauseBackStacks`的作用就是 pause 掉当前正在 resume 的 Activity。在我们的场景下，启动 B 的时候，需要 pause 的就是 A。所以在这个地方，直接过滤掉我们的 A 即可。

```java
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
        boolean someActivityPaused = false;
        for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = mStacks.get(stackNdx);
            final ActivityRecord resumedActivity = stack.getResumedActivity();
            if (resumedActivity != null
                    && (stack.getVisibility(resuming) != STACK_VISIBILITY_VISIBLE
                        || !stack.isFocusable())) {
                //改动在这
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

在`applySleepTokens`流程中，会调用`resumeFocusedStacksTopActivities`来 resume 起在顶部的 Activity，那么我们需要在它之前，先 resume 起我们的 A。

```java
ActivityRecord tmpTop = stack.topRunningActivityLocked();
final ActivityRecord r = getActivityDisplay(DEFAULT_DISPLAY).getHomeActivity();
if (r != null && !r.finishing) {
    boolean result = r.getActivityStack().resumeTopActivityUncheckedLocked(r, null);
}
```

## Layer 异常

上述操作改完后，发现唤醒时，A 无法显示，只能显示 B，且通过下面命令查看 Activity 的状态时，发现都是 resume 的状态。

```bash
adb shell dumpsys activity activities | grep -E 'Stack|TaskRecord|Hist|Display|state'
```

所以问题不会出现在生命周期上，于是排查下 Layer 有什么问题，得到了下面的输出。

正常情况下，一个 Unity 程序的 Layer 有两个，一个是 Unity 程序的壳，一个是使用 SurfaceView 渲染的 Layer,如下所示：

```text
  - Output Layer 0x7564882000 (Composition layer 0x7564786898) (com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity#0)
        Region visibleRegion (this=0x7564882028, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=1
      hwc: layer=0x0812 composition=DEVICE (2)
  - Output Layer 0x7560edde00 (Composition layer 0x7571e40598) (SurfaceView - com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity#0)
        Region visibleRegion (this=0x7560edde28, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=2
      hwc: layer=0x0813 composition=DEVICE (2)
```

可以看到 Layer 的名字，一个是 com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity，一个是 SurfaceView - com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity，只有 SurfaceView - com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity 这个 Layer 输出，Unity 程序才能正常显示。当出现问题的时候，日志是这样的：

```text
   1 Layer  - Output Layer 0x6f6890d000 (Composition layer 0x6ffe117698) (com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity#0)
        Region visibleRegion (this=0x6f6890d028, count=1)
    [  0,   0, 5088, 2544]
      forceClientComposition=false clearClientTarget=true displayFrame=[0 0 5088 2544] sourceCrop=[0.000000 0.000000 5088.000000 2544.000000] bufferTransform=0 (0) z-index=0
      hwc: layer=0x087 composition=DEVICE (2)
```

那么，就是少了 SurfaceView - com.mikaelzero.dock/com.unity3d.player.UnityPlayerActivity 这个 Layer。那么问题就应该出现在这。

在排查这个问题时，耗费了非常多的时间，因为问题给的线索很少，一开始所有的状态都是正确的，唯独这个 Layer 不显示。而且我对 Unity 程序是信心满满，一直没怀疑 Unity 那边有什么问题，虽然确实不是 Unity 的问题，但是也是从 Unity 那部分排查得出的结果。

OnApplicationPause

```java
    @Override public void onWindowFocusChanged(boolean hasFocus)
    {
        android.util.Log.d("qqq","dock UnityPlayerActivity onWindowFocusChanged hasFocus="+hasFocus);
        super.onWindowFocusChanged(hasFocus);
        mUnityPlayer.windowFocusChanged(hasFocus);
    }
```