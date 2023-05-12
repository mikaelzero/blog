---
title: Shader(一)-如何使用
urlname: play-shadertoy
date: 2023/05/02
tags:
  - Shader
---

## 什么是 Shadertoy？

Shadertoy 是一个在线的着色器编辑器和社区，旨在让人们分享和发现基于片段着色器的图像和动画效果。它允许用户编写自己的着色器代码，并将其应用于屏幕上的几何体，以创建各种视觉效果，包括光影、材质、粒子效果、折射和反射等。Shadertoy 的编辑器支持 GLSL（OpenGL 着色语言），并且具有实时预览功能，使用户可以即时看到他们的效果。

你可以看到很多可能只有一两百行或者四五百行的代码就能做出非常酷炫的效果，非常具有挑战性。

最著名的大佬莫过于 iq，比如下图，就是 800 行的代码实现，除了背景是一些素材外，其他都是代码生成。

![](images/opengl/shadertoy_demo.png)

## 如何使用

Shadertoy 提供了一些着色器使用，图像着色器、声音着色器、虚拟现实着色器。

### 图像着色器

为了通过计算每个像素的颜色来生成过程图像着色器需要执行 mainImage()函数。每生成一个像素点时此函数都会被调用，并且此程序的责任是提供正确的输入并从中获得输出颜色并将其分配给屏幕像素。设计原型:

`void mainImage( out vec4 fragColor, in vec2 fragCoord );`

其中 fragCoord 包括着色器用来计算颜色的像素坐标。坐标是以像素为单位，精确范围从 0.5 到-0.5, 覆盖渲染表面，其中分辨率通过 iResolution 参数(详见下文)传递给着色器。

产生的结果颜色由四位向量储存在 fragColor 里, 最后一位被接受端忽略。 在未来用于叠加多个渲染目标时，结果作为“输出”参数储存。

### 声音着色器

声音由 GLSL 着色器通过 mainSound()函数产生, 遵循以下设计原型:

`vec2 mainSound( float time )`

其中 time 变量变量是用于计算波幅的声音样本的时间（以秒为单位）。这个时间将以 iSampleRateuniform 指定的采样率进行采样，根据应用程序，该采样率通常为 44100 或 48000。

通过 mainSound 函数的返回值，将期望的波幅输出为立体声（左和右声道）声音的一对值。
由 mainSound 入口点生成的着色器将被自动标记为声音着色器，并且可以通过声音输出限定词的过滤器系统进行搜索。

### 虚拟现实着色器

Shadertoy 着色器可以为虚拟现实（VR）无缝生成图像。这些着色器需要实现 mainVR()函数, 此方程不是着色器必须实现方程但遵循以下设计原型:

`void mainVR( out vec4 fragColor, in vec2 fragCoord, in vec3 fragRayOri, in vec3 fragRayDir )`

其中 fragCoord 和 fracColor 工作方式和普通 2D 图像着色器一样：他们包括像素在表面空间的坐标系数和输出像素颜色。

变量 fragRayOri 和 fragRayDir 包含在跟踪器空间中给出的经过虚拟空间里的像素的射线原点和射线方向。 如果移动的相机是通过虚拟空间移动，那么着色器将把这些值转换到相机空间。射线原点是一个变量并且在相机不作为针孔相机的 VR 系统中不是 uniform 参数。

由 mainVR 入口点生成的着色器将被自动标记为 VR，并且可以通过 VR 限定词的过滤器系统进行搜索。

### 统一变量输入

着色器可以通过使用以下统一变量来提供不同类型的每帧静态信息:

```glsl
uniform vec3 iResolution;
uniform float iTime;
uniform float iTimeDelta;
uniform float iFrame;
uniform float iChannelTime[4];
uniform vec4 iMouse;
uniform vec4 iDate;
uniform float iSampleRate;
uniform vec3 iChannelResolution[4];
uniform samplerXX iChanneli;
```

## VSCode 中使用 ShaderToy

需要下载两个插件：

名称: Shader Toy
VS Marketplace 链接: https://marketplace.visualstudio.com/items?itemName=stevensona.shader-toy

名称: WebGL GLSL Editor
VS Marketplace 链接: https://marketplace.visualstudio.com/items?itemName=raczzalan.webgl-glsl-editor

Shader Toy 插件使用来识别 ShaderToy 上预制的函数和输入输出，WebGL GLSL Editor 插件用来语法识别和提示。

关于 Texture 的输入，Shader Toy 插件官网中有详细的解释。
