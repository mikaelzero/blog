EGL 是 OpenGL ES 渲染 API 和本地窗口系统(native platform window system)之间的一个中间接口层, 它主要由系统制造商实现. EGL 提供如下机制：

- 与设备的原生窗口系统通信
- 查询绘图表面的可用类型和配置
- 创建绘图表面
- 在 OpenGL ES 和其他图形渲染 API 之间同步渲染
- 管理纹理贴图等渲染资源

为了让 OpenGL ES 能够绘制在当前设备上, 我们需要 EGL 作为 OpenGL ES 与设备的**桥梁**.

- Display(EGLDisplay) 是对实际显示设备的抽象.
- Surface（EGLSurface）是对用来存储图像的内存区域
- FrameBuffer 的抽象, 包括 Color Buffer, Stencil Buffer , Depth Buffer. Context (EGLContext) 存储 OpenGL ES 绘图的一些状态信息.

### 使用 EGL 的绘图的一般步骤：

1. 获取 EGL Display 对象：eglGetDisplay()
2. 初始化与 EGLDisplay 之间的连接：eglInitialize()
3. 获取 EGLConfig 对象：eglChooseConfig()
4. 创建 EGLContext 实例：eglCreateContext()
5. 创建 EGLSurface 实例：eglCreateWindowSurface()
6. 连接 EGLContext 和 EGLSurface：eglMakeCurrent()
7. 使用 OpenGL ES API 绘制图形：gl\_\*()
8. 切换 front buffer 和 back buffer 送显：eglSwapBuffer()
9. 断开并释放与 EGLSurface 关联的 EGLContext 对象：eglRelease()
10. 删除 EGLSurface 对象
11. 删除 EGLContext 对象
12. 终止与 EGLDisplay 之间的连接

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/HAk3Pa.jpg)

### surface

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/LkOrhb.jpg)

#### Surface

Handle onto a raw buffer that is being managed by the screen compositor.即 Surface 是保存原始缓存区的句柄, 也就是显示的像素数据

#### SurfaceView

有 surface 的 view, 在 Client 端(App)它仍在 View hierachy 中, 但在 Server 端（WMS 和 SF）中, 它与宿主窗口是分离的, 好处是对这个 Surface 的渲染可以放到单独线程去做, 渲染时可以有自己的 GL context

#### SurfaceHolder

SurfaceHolder 是用来操作 surface 的接口, 通过 SurfaceView 的 getHolder 来获取

#### SurfaceTexture

它对图像流的处理并不直接显示, 而是转为 GL 外部纹理, 因此可用于图像流数据的二次处理（如 Camera 滤镜, 桌面特效等）.

SurfaceTexture 从图像流（来自 Camera 预览, 视频解码, GL 绘制场景等）中获得帧数据, 当调用 updateTexImage()时, 根据内容流中最近的图像更新 SurfaceTexture 对应的 GL 纹理对象, 接下来, 就可以像操作普通 GL 纹理一样操作它了

#### ANativeWindow

native 层的 ANativeWindow 就等价于 Java 层的 Surface, 可以通过接口 ANativeWindow_fromSurface ()从 surface 对象中获取到, 可以通过软件对其进行 lock, render, unlock-and-post 等操作

### PbufferSurface

- WindowSurface
  在屏幕上的一块显示区的封装, 渲染后即显示在界面上
- PbufferSurface
  在显存中开辟一个空间, 将渲染后的数据(帧)存放在这里
- PixmapSurface
  以位图的形式存放在内存中

### 共享纹理的两种实现

- 一种是 EGL 的 ShareContext 机制
- 一种是共享内存, 这里就是 EGLImageKHR

### 纹理流程

- 1.glGenTextures 生成一个纹理句柄
- 2.glBindTexture 绑定纹理句柄
- 3.glTexParameteri 设置纹理参数
- 4.glTexImage2D 输入到像素数据到纹理
- 5.glActiveTexture 激活纹理
- 6.glDrawElements 绘制到屏幕
- 7.eglSwapBuffers 交换缓冲区, 显示到屏幕

### VAO VBO FBO

- gen
- bind
- draw
- active
- unbind

### FBO

https://blog.csdn.net/xiajun07061225/article/details/7283929/

https://zhuanlan.zhihu.com/p/451871889

### eglMakeCurrent

EGLBoolean eglMakeCurrent(EGLDisplay display,EGLSurface draw,EGLSurface read,EGLContext context);

- Display 是一个连接, 用于连接设备上的底层窗口系统
- Context 仅仅是个容器, 是设计来存储渲染相关的输入数据
- Surface 则是设计来存储渲染相关的输出数据

eglMakeCurrent 把 context 绑定到当前的渲染线程以及 draw 和 read 指定的 Surface. draw 用于除数据回读(glReadPixels、glCopyTexImage2D 和 glCopyTexSubImage2D)之外的所有 GL 操作. 回读操作作用于 read 指定的 Surface 上的帧缓冲(frame buffer).

因此, 当我们在线程 T 上调用 GL 指令, OpenGL ES 会查询 T 线程绑定是哪个 Context C, 进而查询是哪个 Surface draw 和哪个 Surface read 绑定到了这个 Context C 上.

### glFinish 与 glFlush

OpenGL ES 驱动和 GPU 以并行/异步的机制运行. 发起 GL 调用时, 为了得到最好的性能, 驱动会尝试尽快地把调用指令发送给 GPU. 但 GPU 并不会马上执行这些指令, 这些指令只是添加到了 GPU 的指令队列里等待 GPU 执行. 如果在短时间内发送大量的 GL 指令给 GPU, GPU 的指令队列可能满了, 以至于驱动需要把这些指令保存在 Context 的调用缓存里(即上文里提到的 Context 内的调用缓存).

那么问题来了, 这些等待中的指令何时会发送给 GPU 呢？通常来说, 大多数 OpenGL ES 驱动的实现可能会在发起新的(下一个)GL 指令发起的时候发送这些指令. 如果想要主动执行这个操作, 那就调用 glFlush. 这个操作将会阻塞当前线程, 直到所有的指令都发送给了 GPU. glFinish 这个命令更加强大, 它会阻塞当前线程, 直到所有的指令都发送给了 GPU, 并执行完毕. 需要注意的是, 应用程序的性能会因此下降.

glFlush 和 glFinish 被称为显式同步操作. 某些情况下也会发生隐式同步操作. 调用 eglSwapBuffers 时, 就可能发生这种情况. 由于这个操作是由驱动直接执行的, 此时 GPU 可能把所有待执行的 glDraw*绘制指令, 作用在一个不符合预期的 surface 缓冲上(如果之前前端缓冲和后端缓冲已经交换过了). 为了防止这种情形, 在交换缓冲前, 驱动必须阻塞当前线程, 等待所有的影响当前 surface 的 glDraw*指令执行完毕.

当然, 使用双重缓冲的 surfaces 时, 不需要主动调用 glFlush 或 glFinish：因为 eglSwapBuffers 进行了隐式同步操作. 但在使用单缓冲 surfaces(如上文提到的第二个线程里)的情况, 需要及时调用 glFlush, 例如：在线程退出前, 必须调用 glFlush, 否则, GL 指令可能从未发送到 GPU.

### glViewPort

glViewport(GLint x,GLint y,GLsizei width,GLsizei height)

x, y 以像素为单位, 指定了视口的左下角位置, width, height 表示这个视口矩形的宽度和高度, 根据窗口的实时变化重绘窗口.

在默认情况下, 视口被设置为占据打开窗口的整个像素矩形, 窗口大小和设置视口大小相同, 所以为了选择一个更小的绘图区域, 就可以用 glViewport 函数来实现这一变换, 在窗口中定义一个像素矩形, 最终将图像映射到这个矩形中. 例如可以对窗口区域进行划分, 在同一个窗口中显示分割屏幕的效果, 以显示多个视图.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/86R0Wg.jpg)
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/2SWLJM.jpg)

### glScissor

不想 OpenGL 绘制全屏画布, 只需要填充一部分, 则需要用到 glScissor()

glScissor 以左下角为坐标原点(0,0),前两个参数 x、y 指定了裁剪窗口的左下角. width 和 height 指定了窗口的宽和高

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/e5RQGt.jpg)

### EGL 的双缓冲

其内部有两个 FrameBuffer（帧缓冲区, 可以理解为是一个图像的存储区域）, 当 EGL 将一个 FrameBuffer 显示到屏幕上的时候, 另外一个 FrameBuffer 就在后台等待 OpenGL ES 进行渲染输出了. 直到调用函数 eglSwapBuffers 这条指令的时候, 才会把前台的 FrameBuffer 和后台的 FrameBuffer 进行交换, 这样用户就可以在屏幕上看到刚才 OpenGL ES 渲染输出的结果了.

如何将 EGL 和设备的屏幕连接起来呢？答案是使用 EGLSurface, Surface 实际上是一个 FrameBuffer, 通过 EGL 库提供的 eglCreateWindowSurface 可以创建一个可实际显示的 Surface, 通过 EGL 库提供的 eglCreatePbufferSuface 可以创建一个 OffScreen 的 Surface

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/n6mk6e.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/HCpRgP.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/TH2MJD.jpg)

### 模板缓冲(Stencil Buffer)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/Iead4A.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/2cuRwz.jpg)

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/GeFna0.jpg)

### 齐次坐标

- 若 w==1, 则向量(x, y, z, 1)为空间中的点.
- 若 w==0, 则向量(x, y, z, 0)为方向.

（请务必将此牢记在心. ）

二者有什么区别呢？对于旋转, 这点区别倒无所谓. 当旋转 点和方向 时, 结果是一样的. 但对于平移（将点沿着某个方向移动）情况就不同了. ”平移一个方向”是毫无意义的.

齐次坐标使得我们可以用同一个公式对点和方向作运算.

### 深度缓冲与 Z-buffer

在投影时, 每个投影点的深度也会被记录下来, 写入 深度缓冲.

其实是：在光栅化时, 每个片元的深度会计算出来, 写入深度缓冲

深度缓冲最主要的一个应用是 “消隐”

若不做消隐处理, 后绘制之物总会挡住先绘制之物

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/G8vMD9.jpg)

Z-buffer 消隐算法

1. 在把显示对象的每个面上每一点的颜色值填入帧 缓冲器相应单元前, 要把这点的深度值和 Z-buffer 中相应单元的值进行比较.
2. 只有前者小于后者时才改变帧缓冲器的那一单元 的值, 同时 Z-buffer 中相应单元的值也要改成这点的深度值；否则帧缓冲器相应单元的值不变

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/6cbFYv.jpg)

比如默认缓冲区下的 z-buffer 值为-5, 整体的效果如下：

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/TQuoHM.jpg)

### FBO 伪代码

```cpp
glBindTexture(GL_TEXTURE_2D, color_tex);
......
glTexImage2D(......);
glGenFramebuffers(1, &fb);
glBindFramebuffer(GL_FRAMEBUFFER, fb);
glFramebufferTexture2D(......)
......
glBindFramebuffer(GL_FRAMEBUFFER, fb);
//进行渲染, 比如给图片加个彩色
render();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
......
//使用FBO的texture进行渲染
glBindTexture(GL_TEXTURE_2D, color_tex);

```

### glGetIntegerv

glGetIntegerv(GL_FRAMEBUFFER_BINDING, &\_fbo);

保存当前的 FBO 句柄, 后续可以通过

glBindFramebuffer(GL_FRAMEBUFFER, \_fbo)

使用该句柄

### MRT 伪代码

```c++
Shader ourShader, ourShaderFBO
ourShader.use()
unsigned int defaultTexture
.......
glBindFramebuffer(GL_FRAMEBUFFER, fb);
glGenTextures(ATTACHMENT_NUM, m_AttachTexIds);
for (int i = 0; i < ATTACHMENT_NUM; ++i) {
    glBindTexture(GL_TEXTURE_2D, m_AttachTexIds[i]);
    glTexParameterf......
    glTexImage2D......
    glFramebufferTexture2D......
}
glDrawBuffers(ATTACHMENT_NUM, attachments);
...... glGenRenderbuffers ......

//draw
glBindFramebuffer(GL_FRAMEBUFFER, fb);
ourShader.use();
glBindTexture(GL_TEXTURE_2D, defaultTexture);
.......glDrawElements......
ourShaderFBO.use();
for (int i = 0; i < ATTACHMENT_NUM; ++i) {
    glActiveTexture(GL_TEXTURE0 + i);
    glBindTexture(GL_TEXTURE_2D, m_AttachTexIds[i]);
}
........glDrawElements......
```

### OpenGL 和 OpenGLES 坐标差异

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/0YNXy1.jpg)

### VAO、VBO、EBO、FBO

#### VBO-Vertex Buffer Objects-顶点缓冲对象

VBO 是在显存中开辟出的一块内存缓存区, 用于存储顶点的各类属性信息. 在渲染时, 可以直接从 VBO 中取出顶点的各类属性数据, VBO 存储在显存而不是内存, 不需要从 CPU 传输数据, 处理效率更高.

#### VAO-Vertex Arrary Object-顶点数组对象

VBO 保存了一个模型的顶点属性信息, 每次绘制模型之前需要绑定顶点的所有信息, 当数据量很大时, 重复这样的动作变得非常麻烦. VAO 可以把这些所有的配置都存储在一个对象中, 每次绘制模型时, 只需要绑定这个 VAO 对象就可以了.

VAO 是一个保存了所有顶点数据属性的状态结合, 它存储了顶点数据的格式以及顶点数据所需的 VBO 对象的引用.

VAO 本身并没有存储顶点的相关属性数据, 这些信息是存储在 VBO 中的, VAO 相当于是对很多个 VBO 的引用, 把一些 VBO 组合在一起作为一个对象统一管理.

#### EBO-Element Buffer Object-索引缓冲对象

EBO 中存储的内容就是顶点位置的索引 indices, EBO 跟 VBO 类似, 也是在显存中的一块内存缓冲器, 只不过 EBO 保存的是顶点的索引.

#### FBO-帧缓冲区对象

一般作用于离屏渲染.

有两种类型的“帧缓存关联图像”：纹理图像（texture images）和渲染缓存图像（renderbuffer images）. 如果纹理对象的图像数据关联到帧缓存, OpenGL 执行的是“渲染到纹理”（render to texture）操作. 如果渲染缓存的图像数据关联到帧缓存, OpenGL 执行的是离线渲染（offscreen rendering）.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/fflN57.jpg)

FBO 提供了一种高效的切换机制；将前面的帧缓存关联图像从 FBO 分离, 然后把新的帧缓存关联图像关联到 FBO. 在帧缓存关联图像之间切换比在 FBO 之间切换要快得多. FBO 提供了 glFramebufferTexture2DEXT()来切换 2D 纹理对象和 glFramebufferRenderbufferEXT()来切换渲染缓存对象.

FBO 本身没有图像存储区. 我们必须帧缓存关联图像（纹理或渲染对象）关联到 FBO. 这种机制允许 FBO 快速地切换（分离和关联）帧缓存关联图像. 切换帧缓存关联图像比在 FBO 之间切换要快得多. 而且, 它节省了不必要的数据拷贝和内存消耗. 比如, 一个纹理可以被关联到多个 FBO 上, 图像存储区可以被多个 FBO 共享.
