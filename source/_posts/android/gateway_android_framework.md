---
title: 使用「Gateway」IDE来开发Android Framework
date: 2023/03/07
tags:
  - android
  - framework
  - Gateway
  - 源码
---

> 前提：你的源码是在服务器上，并且通过 ssh 来连接，Gateway 是一个远程开发工具。
> 优点：和在 AS 上开发普通 app 一样顺畅。
> 缺点：断开连接后无法保持你代码的位置，会重新定位到开头。

可以使用 Jetbrains 的 ToolBox 来下载「Gateway」

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/WJ142e.png)

在服务器上使用 Aidegen

首先进入 android 根目录

```bash
m aidegen
```

生成 framework 的 idea 工程文件

```bash
aidegen-dev -n frameworks
```

> 如果想打开其他文件夹，frameworks 替换为其他目录，应该也是可以的

用 Gateway 打开 frameworks 目录，注意不是根目录而是要 frameworks 目录

> 注意提示的 `JDK "JDK18" is missing` 不要去配置
