---
title: VR中的延迟
date: 2022/12/11
tags:
  - vr
  - latency
---

VR 中的”延迟”, 特指”Motion-To-Photon Latency”

## Motion-To-Photon Latency

什么是 Motion-To-Photon Latency,指的是从用户运动开始到相应画面显示到屏幕上所花的时间,如果在你进行运动（例如向左看）时需要 100 毫秒才能将您的运动反映在虚拟现实耳机的屏幕上, 则 100 毫秒就是 Motion-To-Photon 的延迟.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ejkyya.jpg)

## 延迟数据采集

一般通过代码层面的收集,都是不准确的,因为本身处理器和外部的传感器以及显示设备都会造成延迟,所以一种有效的测试方式是记录高速视频,同时捕捉起始的物理运动和最终的显示更新. 然后, 可以通过单步执行视频并计算两个事件之间的视频帧数来确定系统延迟.

一些常用的减少延迟的技术

## Pixel Switching Time

像素切换时间被称为更新显示器上所有像素所需的时间. 要在显示器上看到图像, 必须绘制它. 绘制图像意味着显示器上的每个像素都必须更新以反映该图像. 最好的说明如下：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/k5eQkl.jpg)

在此示例中, 此 (3 x 3) 显示器的像素切换时间为 3 毫秒. 这是因为更新一行像素需要 1ms, 更新整个显示需要 3 次更新.

LCD 显示器的像素切换时间较差. 目前, Oculus Rift 耳机和许多可与 Google Cardboard 配合使用的智能手机都是 LCD 显示器. OLED 等其他技术可提供几乎瞬时的响应时间.

要减少像素切换时间, 请使用具有更快像素切换时间的更快显示技术. 例如, 如果使用 OLED 代替 LCD 显示器, 我们可以减少延迟.

## 刷新率

刷新率是显示器从显卡获取要绘制的新图像的频率, 它决定了每幅图像之间等待的延迟时间. 对于 60 Hz 显示器, 延迟计算为 (1000ms / 60 Hz =) 16.67ms. 这意味着即使在图形卡上绘制图像的延迟为零, 并且像素切换时间为零, 图像也只能以每 16.67 毫秒更新 1 张图像的速度快速更新.

为了减少获取图像的延迟, 需要更高的刷新率显示, 因为延迟 (ms) = 1000ms / 刷新率 (Hz). 例如, 120 Hz 会将此延迟减少到 8.33 毫秒.

## CPU, GPU, Game Engine

游戏或 VR 应用程序的经典处理模型是：

- 读取用户输入（I）
- 运行模拟 (S)
- 发出渲染命令 (R)
- 图形绘制（G）
- 等待垂直同步（‘’）
- 扫描输出 (V)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/Ytxpxh.jpg)

从上图看出一个简单的 pipeline
CPU1 读取用户输入并运行模拟, CPU2 发出渲染命令, GPU 绘制场景并转换为绘制命令显示, 并将其写入显示屏(V, 扫描出)
整个渲染过程需要 4 帧, 所以流水线渲染至少需要 48ms ~ 64ms

## Timewarp

了解过延迟渲染的人应该都知道, 我们可以利用 ZBuffer 的深度数据, 逆向推导出屏幕上每个像素的世界坐标

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/bL3aO8.jpg)

这就意味着, 我们可以把所有像素变换到世界空间, 再根据新的摄像机位置, 重新计算每个像素的屏幕坐标, 生成一幅新的图像:

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/8A45wp.jpg)

Timewarp 正是利用了这个特性, 在保证位置不变的情况下, 把渲染完的画面根据最新获取的传感器朝向信息计算出一帧新的画面, 再提交到显示屏. 由于角度变化非常小, 所以边缘不会出大面积的像素缺失情况.

## SingleBuffer

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/CAbIiH.jpg)

## Phase Sync（相位同步）

> 通过动态调整帧定时来帮助最小化延迟

说是可以可大大降低延迟.
Phase Sync（相位同步）是 Oculus PC SD 首次引入的一种帧定时管理技术, 它可以通过动态调整帧定时来帮助最小化姿态延迟. 它代替了传统的 Fixed-latency 模式
Fixed-latency(固定延迟) 模式就是尽可能早地合成帧以避免丢失当前帧并需要重新使用之前的帧. 之前的帧会对用户体验产生负面影响.

与 Fixed-latency 模式相比, Phase Sync（相位同步）根据应用的工作负载自适应地处理帧定时. 目标是在合成器需要完成帧之前完成帧的渲染, 这样可以尽可能地节省延迟, 同时不会丢失任何帧. 对于一个典型的多线程 VR 应用, Phase Sync（相位同步）和 Fixed-latency 模式之间的区别可参见下图说明：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/v6z3In.jpg)

我们可以看到, Frame N 的完整时间与 Phase Sync（相位同步）下的合成时间非常接近, 这是 Phase Sync（相位同步）节省延迟的来源.以上只是一个简单说明.

## Warp

异步空间扭曲（Asynchronous Space Warp）
应用空间扭曲（Application Space Warp）
运动平滑（Motion Smoothing）
同步空间扭曲（Synchronous Space Warp；SSW）

参考:
https://blog.csdn.net/xoyojank/article/details/50667507
http://www.chioka.in/what-is-motion-to-photon-latency/