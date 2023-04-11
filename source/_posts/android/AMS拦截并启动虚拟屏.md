---
title: AMS拦截并启动虚拟屏
urlname: AMS-intercepts-and-starts-the-virtual-screen
date: 2023/03/07
tags:
  - Android
  - Framework
  - DMS
  - AMS
---

为什么 ActivityDisplay 延迟添加了？

DMS::handleDisplayDeviceAddedLocked
DMS::addLogicalDisplayLocked
sendDisplayEventLocked(displayId, DisplayManagerGlobal.EVENT_DISPLAY_ADDED);这里是一个异步，发会送`EVENT_DISPLAY_ADDED`
回调 RootActivityContainer 的 onDisplayAdded，通过 getActivityDisplayOrCreate 创建 ActivityDisplay

1. 通过接收 onDisplayAdded 来重复启动 Activity
2. 主动调用 getActivityDisplayOrCreate