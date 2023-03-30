---
title: android与unity的纹理
urlname: android_unity_texture
date: 2022/03/22
tags:
  - Android
  - Unity
  - texture2D
---

## 普通纹理

Android 下的普通纹理, 比如一张图片, 与 unity 使用有两个方案, 一个是直接跑在 unity 的渲染线程中, 在 unity 的渲染线程里进行渲染, 二是通过共享 context 的方式进行纹理共享

## 视频流 RTT 类的纹理

这类纹理代表的就是 camera 的视频流数据或者是手机投屏的纹理数据, 同样有两种方案, 以手机投屏为例

### 离屏渲染

首先创建出 textureID 以及创建出对应的 surfacetexture 和 surface, 再与 virtual display 进行绑定, 这样渲染的纹理就和 textureid 绑定在一起

其次, 由于 unity 无法直接使用 opengl es 的 OES 的格式, 在渲染的时候, 需要创建一个新的类型为 texture2D 的纹理 id, 再通过 FBO 的方式绑定这个 id, 这样在 OES 下绘制的就会绘制到 texture2D 的纹理 ID 上, 之后再把这个纹理 ID 给 unity, unity 进行渲染

### 让 unity 处理

unity 无法直接使用 opengl es 的 OES 的格式, 但是 unity 可以接收 OES 格式, 只要 unity 自己通过 shader 的方式处理, 所以流程会更加简单, 因为渲染都在 unity 做

首先和上面一样要拿到 virtual display 的纹理数据

其次直接把这个纹理 ID 给 unity, unity 在脚本中绑定一个 shader, 通过这个 shader 进行渲染
