---
title: AMS拦截并启动虚拟屏
urlname: AMS-intercepts-and-starts-the-virtual-screen
date: 2023/04/11
tags:
  - Android
  - Framework
  - DMS
  - AMS
---

## 需求场景

1. Unity 中存在一个面板，这个面板是用来显示 2D 应用的画面的
2. Unity 中的面板是 Unity 创建的，它接收的是一个 TextureID，并创建 Shader 后根据纹理 ID 来渲染
3. 在 android 系统层面，需要拦截所有的 2D 应用并将它们显示到 Unity 的面板中
4. 且这些 2D 应用都是显示到虚拟屏幕中，也就是 VirtualDisplay

先说下这部分我的艰苦历程吧，这个问题其实拖了非常久，因为一开始想到的方案就是多进程共享 Texture 的思路。首先 Unity 应用是一个进程，系统又是一个进程，所以自然而然就想到了多进程共享纹理的思路。一开始直接用固定的纹理 ID 测试后，发现无法实现久没怎么研究了，主要是网上搜到的多进程共享纹理的方案多少有点复杂，就没怎么去看，然后最近才开始重视这个问题，发现其实共享 Surface 就行了。

## SurfaceTexture

## VirtualDisplay

每个 2D 应用在 VR 系统中都是无法直接显示的，因为都会直接显示到主屏幕中，就无法根据 VR 眼镜的转动而渲染。

所以一个解决方案是将所有的 2D 应用显示到虚拟屏幕中，每个虚拟屏都会有一个 DisplayId 和 Surface，Surface 的作用就是将虚拟屏幕上的画面数据设置 Surface 中。

创建虚拟屏幕时有一些参数：

```java

```

其中的 Surface，是我们可以手动操作的地方，我们可以在将 2D 启动到虚拟屏幕后，手动

## Unity 部分

Unity 部分比较简单，使用自带的接口根据 TextureId 来创建 2D 纹理并且创建对应的 Shader 即可

DMS::handleDisplayDeviceAddedLocked
DMS::addLogicalDisplayLocked
sendDisplayEventLocked(displayId, DisplayManagerGlobal.EVENT_DISPLAY_ADDED);这里是一个异步，发会送`EVENT_DISPLAY_ADDED`
回调 RootActivityContainer 的 onDisplayAdded，通过 getActivityDisplayOrCreate 创建 ActivityDisplay

1. 通过接收 onDisplayAdded 来重复启动 Activity
2. 主动调用 getActivityDisplayOrCreate