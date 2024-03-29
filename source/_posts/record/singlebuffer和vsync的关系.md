---
title: singlebuffer和vsync的关系
urlname: between_singlebuffer_vsync
date: 2022/11/11
tags:
  - SurfaceFlinger
  - Vsync
  - Singlebuffer
---

单缓冲可以有效地减少延时, 因为双缓冲是需要等待 vsync 再进行提交数据, 而单缓冲窗口随时可以自己控制提交的时机, 单缓冲的最大问题就是画面撕裂.

**单缓冲减少的时间在哪？首先双缓冲有两个缓冲区, 前缓冲区绘制结束之后并不会马上交给后缓冲区, 而是会等待 vsync 信号, 到了之后再进行交换, 这样做的原因是, 绘制速度是非常快的, 而显示速率比较慢, 所以不能绘制完就提交, 这样有可能绘制了好几次, 显示器才拿了一次数据, 所以为了减少资源占用和浪费, 需要等待 vsync 信号再进行缓冲区交换, 而单缓冲就是减少了这部分时间, 自己控制时机**

android 中 Vysnc + 三重缓冲 的框架, 显示延迟至少一个 Vysnc 周期. 所以就需要 Single Buffer + Direct Render. 这个就要求在一个 Vsync 周期里完成 应用渲染 + ATW 反畸变, 反色差后处理送帧. VR 应用 和 普通 android 应用 的区别：

1. VR 应用都是 OpenGL 3D 渲染的, 而且还是双目渲染, GPU loading 是比普通的 andrid 应用高不少
2. 普通 android 平台, 应用渲染完 SurfaceFlinger 直接送显就行了, VR 还需要反畸变、反色差后处理,业界基本上是拿 GPU 做的, 这让 GPU loading 进一步提高
3. 分辨率和刷新率的提升, 加重了系统的负载
4. 普通 android 场景偶尔有几次丢帧关系不大, 但是在 VR 里面就会对用户造强烈的不舒适,VR 要求不能丢帧

要显示屏幕动画, 我们要设置两个时机：

- 时机一：生成帧, 产生了新的画面（帧）, 将其填充到 framebuffer 中, 这个过程由 CPU（计算绘制需求）和 GPU（完成数据绘制）完成；
- 时机二：显示帧, 显示屏显示完一帧图像之后, 等待一个固定的间隔后, 从 framebuffer 中取下一帧图像并显示, 这个过程由 GPU 完成.

为 framebuffer 空间进行扩展, 使得其至少为 2 倍屏幕空间,通常会将其扩展为 双缓冲 或 三缓冲
图像的缓冲区不光只有 framebuffer, 对 framebuffer 扩展为多缓冲包含两个意思：

- 现在的 framebuffer 很多本身都实现了至少两个缓冲区
- 在 Android 显示系统中, 在将帧数据写入 framebuffer 之前对于帧数据的绘制和生成也提供了多级缓冲, 即所谓的绘制缓冲区, 它们和 framebuffer 一起组成了多缓冲的结构

framebuffer 双缓冲一般有两种实现方法, 其共同点是都会把缓冲分为两部分：

- 背景缓冲（backbuffer）：用于保存正在生成的帧.
- 前景缓冲（frontbuffer）：用于保存即将显示的帧.
