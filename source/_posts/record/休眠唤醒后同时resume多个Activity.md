---
title: 休眠唤醒后同时resume多个Activity
urlname: resume-multiple-activities-simultaneously-after-waking_up-from-sleep.
date: 2023/03/24
tags:
  - android
  - framework
  - AMS
password: hello1111111sadjadk
---

之前在 「surfaceflinger 修改 layer 层级」文章中实现了底部的 Layer 能否显示在顶部，但是在休眠唤醒的时候会出现问题。

## 场景

一个 Activity A 和 Activity B,A 做为 Home，也就是桌面，当启动 B 的时候，A 同样能够显示在 B 的上面。A 其实是一个半透明的页面。

当然一开始最快能想到的方案自然就是直接再次把 A 拉起来，放在栈顶，但是这样就会导致 B 会 pause，由于 B 大多数都是 3D 应用，一旦 pause，大部分程序是直接不渲染的，也就是黑屏。同样的，想解决这个问题也可以直接不 pause B，但是休眠唤醒的时候就会有问题，唤醒的时候只会唤醒栈顶的 Activity，也就是 A，所以还必须同时唤醒 B 才能达到效果。并且改动拉起部分代码不少，所以选择了另外一个方案，逻辑是类似的。

首先 A 还是 Launcher，B 还是普通的 3D 应用，正常启动 A，然后启动 B，只是在 pause 的时候，需要让 A 不能够 pause 且 A 的画面能够在顶层，这个在「surfaceflinger 修改 layer 层级」一文中已经解释过，这个解决了第一个问题。

其次还是在休眠唤醒上，在 A 之上启动了一个 B 之后，栈关系是 B 在 A 之上，所以休眠唤醒时，还是只会唤醒 B，所以需要处理的是，同时唤醒 A。

## 异常时的调用栈

一开始尝试的是，在 RootActivityContainer 中的 applySleepTokens 方法中，在 resumeFocusedStacksTopActivities 之前，resume 起 Launcher,代码如下

```java
ActivityRecord tmpTop = stack.topRunningActivityLocked();
final ActivityRecord r = getActivityDisplay(DEFAULT_DISPLAY).getHomeActivity();
if (r != null && !r.finishing) {
    boolean result = r.getActivityStack().resumeTopActivityUncheckedLocked(r, null);
}
```

这里的 r 就是 Launcher。

删除了 Layer 逻辑正常，说明 WMS 部分在现实 Launcher 时窗口时不正确，导致 SurfaceView 部分的 layer 不显示。
