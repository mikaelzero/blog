---
title: VR相关技术词汇
date: 2023/01/02
tags:
  - vr
---

## HMD

HMD 是 Head Mount Display 的缩写, 也就是俗称的头显, 这是 VR 最核心的设备.

## VR 的分类

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/202211031418854.jpg)
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/202211031418230.jpg)
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/202211031419537.jpg)

## VR 基本原理

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/202211031419540.jpg)

1. **处理器**

计算的核心, 用来生成图像, 根据陀螺仪数据计算姿态定位 等. 为了防止眩晕, VR 眼镜要求图像刷新率达到 90Hz. 这对运算速度要求很高. 所以, 一个 VR 眼镜的处理器芯片性能指标至关重要.

2. **显示器**

分别向左右眼睛显示图像.

3. **凸透镜片**

如果把显示器直接贴在人眼前, 人眼是很难聚焦这么近的物体的. 凸透镜片的目的, 就是通过折射光线, 将显示器上的画面成像拉近到视网膜位置, 使人的眼睛能轻松看清几乎贴在眼前的显示屏.

4. **陀螺仪**

显示器里的景象, 需要陀螺仪来检测它是能检测到物体在空间中的姿态/朝向 即可.

## IMU

（英文 Inertial measurement unit, 简称 IMU）, 是测量物体三轴姿态角及加速度的装置. 一般 IMU 包括三轴陀螺仪及三轴加速度计, 某些 9 轴 IMU 还包括三轴磁力计.

- 陀螺仪能测量出 X, Y , Z 三轴的角速度
- 加速度计能测量出三轴的加速度, 无法求出水平方向的偏航角
- 磁力计就是一个指南针, 正好弥补了加速度计无法测量的水平方向的偏航角的问

## 欧拉角

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/eSWMNi.jpg)

在图片中绕 x 轴水平旋转的是横滚角 Roll, 第二次旋转是俯仰角 Pitch, 最后的称为偏航角 Yaw.

但是欧拉角描述刚体旋转会遇到一个著名的万向节锁问题

## 6DOF

dof：degree of freedom, 即自由度.

其中 3dof 是指有 3 个转动角度的自由度, 而 6dof 是指, 除了 3 个转动角度外, 再加上 上下、前后、左右等 3 个位置相关的自由度.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/YPiJTH.jpg)

**自由度（DoF）**

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ChGf43.jpg)

## 眼动追踪

这是一些头显用来跟踪用户注视的过程. 该信息可以为观看者所观看的内容提供更清晰、准确的信息.

## 视野 (FoV)

\*\*FOV 是显示设备边缘与观察点(眼睛)连线的夹角, 简单来说就是你能清晰看见画面+余光扫到的内容. 人的 FOV 一般为 200°（双眼视觉只覆盖 120° 左右）.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/hfViPF.jpg)

## 帧率 (fps)

为了防止用户眩晕和头痛, VR 需要保持高帧率（至少 90 fps）.

## 刷新率

刷新率是显示器可以在屏幕上重绘图像的次数. 一般刷新率应该与帧率相匹配, 否则显示器显示的图像将与计算机单元生成的图像不匹配.

## 由内而外的跟踪（Inside-out tracking）与由外而内的跟踪(Outside-in tracking)

- **由内而外的跟踪（Inside-out tracking）**
  使用 VR 头显内部的传感器追踪. 头显上的摄像头将记录真实环境中的一些固定点, 并以此作为参考点记录你的运动坐标. 无需标记点的由内向外跟踪也可以对一些移动物体进行位置跟踪, 但它缺乏足够高的准确性.
- **由外向内跟踪 (Outside-in tracking)**
  使用外部跟踪设备（如相机或灯塔）来模拟用户所处的“虚拟盒子”. 这种技术非常准确, 但无法跟踪传感器视线之外的任何东西. 简而言之, 保证了精确性但是量程受限.

## 分辨率

在 VR 设备上, 图像分辨率通常会导致细节层次较低, 因为渲染内容的表面看起来比平面矩形屏幕大得多, 这意味着图像被拉伸到更宽的区域. 而且, 对于立体视图, 每只眼睛只能看到实际分辨率的一半. 例如, 4K 视频, 单眼只能看到 2K 分辨率, 如果要实现真正的 4K 视频, 双眼需要都达到 4K

## TW / ATW

https://www.zhihu.com/question/58295603

**什么是 Timewarp. **时间扭曲是把渲染好的一帧图像显示之前, 根据场景被渲染之后头部又旋转角度加以矫正, 得出相对当前头部位置更正确的一帧图像的技术, 从而减少感知到的延迟. 在虚拟现实上使用也可以提高帧率.

简单来说, 在一个 vr 场景中, 当你转动眼镜时, TW 能够预先计算出你转动到某一个角度时应该是什么样的画面.

这种是只针对旋转的时间扭曲. 时间扭曲有一个比较大的好处, 它是 2D 的扭曲, 所以当和畸变（Distortion）结合的时候并不会消耗太多性能. 和相对渲染一帧复杂的场景相比, 时间扭曲只使用非常少的计算. **（此处需要一个 TW 算法：原理参考** https://www.youtube.com/watch?v=WvtEXMlQQtI**）**

## 反畸变

https://zhuanlan.zhihu.com/p/100679725

## 大小核

目前的手机 CPU 按照核心数和架构来说, 可以分为下面三类：

- 非大小核架构
- 大小核架构
- 大中小核架构

目前的大部分 CPU 都是大小核架构, 当然也有一些 CPU 是大中小核架构, 比如高通骁龙 855\865, 也有少部分 CPU 是非大小核架构

### 非大小核架构

很早的机器 CPU 只有双核心或者四核心的时候, 一般只有一种核心架构, 也就是说这四个核心或者两个核心是同构的, 相同的频率, 相同的功耗, 一起开启或者关闭；有些高通的中低端处理器也会使用同构的八核心处理器, 比如高通骁龙 636

现在的大部分机器已经不使用非大小核的架构了

### 大小核架构

现在的 CPU 一般采用 8 核心, 八个核心中, CPU 0-3 一般是小核心, CPU 4-7
小核心一般来说主频低, 功耗也低, 使用的一般是 arm A5X 系列, 比如高通骁龙 845, 小核心是由四个 A55 (最高主频 1.8GHz ) 组成
大核心一般来说最高主频比较高, 功耗相对来说也会比较高, 使用的一般是 arm A7X 系列, 比如高通骁龙 845, 大核心就是由四个 A75（最高主频 2.8GHz）组成

### 大中小核架构

部分 CPU 比较另辟蹊径, 选择了大中小核的架构, 比如高通骁龙 855 8 核 (1 个 A76 的大核+3 个 A76 的中核 + 4 个 A55 的小核）和之前的的 MTK X30 10 核 (2 个 A73 的大核 + 4 个 A53 的中核 + 4 个 A35 的小核）以及麒麟 980 8 核 (2 个 A76 的大核 + 2 个 A76 的中核 + 4 个 A55 的小核）

相比大小核架构, 大中小核架构中的大核可以理解为超大核(高通称之为 Gold +) , 这个超大核的个数一般比较少(1-2 个), 主频一般会比较高, 功耗相对也会高很多, 这个是用来处理一些比较繁重的任务

## 绑核

绑核, 顾名思义就是把某个任务绑定到某个或者某些核心上, 来满足这个任务的性能需求：

任务本身负载比较高, 需要在大核心上面才能满足时间要求
任务本身不想被频繁切换, 需要绑定在某一个核心上面
任务本身不重要, 对时间要求不高, 可以绑定或者限制在小核心上面运行
上面是一些绑核的例子, 目前 Android 中绑核操作一般是由系统来实现的, 常用的有三种方法

### 配置 CPUset

使用 CPUset 子系统可以限制某一类的任务跑在特定的 CPU 或者 CPU 组里面, 比如下面, Android 中会划分一些默认的 CPU 组, 厂商可以针对不同的 CPU 架构进行定制, 目前默认划分

system-background 一些低优先级的任务会被划分到这里, 只能跑到小核心里面
foreground 前台进程
top-app 目前正在前台和用户交互的进程
background 后台进程
foreground/boost 前台 boost 进程, 通常是用来联动的, 现在已经没有用到了, 之前的时候是应用启动的时候, 会把所有 foreground 里面的进程都迁移到这个进程组里面

每个 CPU 架构对应的 CPUset 的配置都不一样, 每个厂商也会有不同的策略在里面, 比如下面就是一个 Google 官方默认的配置, 各位也可以查看对应的节点来查看自己的 CPUset 组的配置

```bash
//官方默认配置
write /dev/CPUset/top-app/CPUs 0-7
write /dev/CPUset/foreground/CPUs 0-7
write /dev/CPUset/foreground/boost/CPUs 4-7
write /dev/CPUset/background/CPUs 0-7
write /dev/CPUset/system-background/CPUs 0-3
// 自己查看
adb shell cat /dev/CPUset/top-app/CPUs
0-7
```

对应的, 可以在每个 CPUset 组的 tasks 节点下面看有哪些进程和线程是跑在这个组里面的

```bash
adb shell cat /dev/CPUset/top-app/tasks
```

需要注意每个任务跑在哪个组里面, 是动态的, 并不是一成不变的, 有权限的进程就可以改

部分进程也可以在启动的时候就配置好跑到哪个进程里面, 下面是 lmkd 的启动配置, writepid /dev/CPUset/system-background/tasks 这一句把自己安排到了 system-background 这个组里面

```bash
service lmkd /system/bin/lmkd
    class core
    user lmkd
    group lmkd system readproc
    capabilities DAC_OVERRIDE KILL IPC_LOCK SYS_NICE SYS_RESOURCE BLOCK_SUSPEND
    critical
    socket lmkd seqpacket 0660 system system
    writepid /dev/CPUset/system-background/tasks
```

大部分 App 进程是根据状态动态去变化的,在 Process 这个类中有详细的定义

### 配置 affinity

使用 affinity 也可以设置任务跑在哪个核心上, 其系统调用的 taskset, taskset 用来查看和设定“CPU 亲和力”, 其实就是查看或者配置进程和 CPU 的绑定关系, 让某进程在指定的 CPU 核上运行, 即是“绑核”.

taskset 的用法
显示进程运行的 CPU

```bash
taskset -p pid
```

注意, 此命令返回的是十六进制的, 转换成二进制后, 每一位对应一个逻辑 CPU, 低位是 0 号 CPU, 依次类推. 如果每个位置上是 1, 表示该进程绑定了该 CPU. 例如, 0101 就表示进程绑定在了 0 号和 3 号逻辑 CPU 上了

绑核设定

```bash
taskset -pc 3  pid    表示将进程pid绑定到第3个核上
taskset -c 3 command   表示执行 command 命令, 并将 command 启动的进程绑定到第3个核上.
```

Android 中也可以使用这个系统调用, 把任务绑定到某个核心上运行. 部分较老的内核里面不支持 CPUset, 就会用 taskset 来设置

### 调度算法

在 Linux 的调度算法中修改调度逻辑, 也可以让指定的 task 跑在指定的核上面, 部分厂家的核调度优化就是使用的这种方法

## 锁频

### CPU 频率

CPU 的工作频率越高, 运算就越快, 但能耗也更高. 然而很多时候, 设备并不需要那么高的计算性能, 这个时候, 我们就希望能降低 CPU 的工作频率, 追求较低的能耗, 以此实现更长的待机时间.

基于此需求, 当前电子设备 的 CPU 都会存在 多个工作频率, 并能根据实际场景进行 CPU 频率的自动切换, 以此达到平衡计算性能与能耗的目的.

## SoC

SOC 称为系统级芯片, 也称片上芯片, 是一个专有目标的集成电路的产品, 其中包括完整系统并有嵌入软件的全部内容. 目前 SOC 更多的集成处理器(包括 CPU, GPU, DSP), 存储器, 基带, 各种接口控制模块, 各种互联总线等, 其典型代表为手机芯片.

SoC（System on Chip）： 称为系统级芯片, 也称为片上系统, 意指它是一个产品, 是一个有专有目标的集成电路, 其中包含完整系统并嵌入软件的全部内容.

CPU = 运算器 + 控制器, 现在几乎没有纯粹的 CPU 了, 都是 SoC.

## DRM

DRM 是目前主流的图形显示框架, Linux 内核中已经有 Framebuffer 驱动用于管理显示设备的 Framebuffer, Framebuffer 框架也可以实现 Linux 系统的显示功能, 但是缺点如下：

不支持 VSYNC
不支持 DMA-BUF
不支持异步更新
不支持 Fence 机制
不支持和 GPU 的通信
这些功能 DRM 框架都支持, 可以统一管理 GPU 和 Display 驱动, 使得软件架构更为统一, 方便管理和维护.

## Pipe 机制

Pipe 是 Linux 的一种系 统调用, Linux 会在内核地址空间中开辟一段共享内存, 并产生一个 Pipe 对象. 每个 Pipe 对象内部都 会自动创建两个文件描述符, 一个用于读, 另一个用于写. 应用程序可以调用 pipe()函数产生一个 Pipe 对象, 并获得该对象中的读、写文件描述符. 文件描述符是全局唯一的, 从而使得两个进程之间可以借 助这两个描述符, 一个往管道中写数据, 另一个从管道中读数据. 管道只能是单向的, 因此, 如果两个 进程要进行双向消息传递, 必须创建两个管道.

## android freeForm

Android N 引入了 Multi-Window, Freeform 自由窗口模式是其中的一种. 自由窗口模式下可以实现窗口的可以自由缩放, 自由移动.

要支持 Freeform 自由窗口模式, 主要有如下一些逻辑支持：

- 定义特有的 freeform stack, 所有的 freeform 模式的 Activity 会在特定的 Freeform stack 启动.
- 为 Freeform 窗口添加特殊的 layout, 用于控制窗口最大化, 关闭以及拖动等功能.
- 为 Freeform 提供一个控制窗口初始化位置和大小的类
- 为 Freeform 提供可以控制窗口移动和缩放处理的类

## BSP

bsp 是板级支持包, 并不是特定某个文件, 而是从功能上理解的一种硬件适配软件包, 它的核心就是：

1. linux 驱动
2. linux BSP （CPU, 电源管理比驱动更深入的硬件支持包）
3. Android HAL 层

## HLSL

高级着色器语言

## MTP

Motion To Photons
MTP 时延是指从头动到显示出相应画面的时间. MTP 时延太大容易引起眩晕, 目前公认的是 MTP 时延低于 20ms 就能大幅减少晕动症的发生

## 分辨率

分辨率(resolution)就是屏幕图像的精密度, 是指显示器所能显示的像素的多少. 由于屏幕上的点、线和面都是由像素组成的, 显示器可显示的像素越多, 画面就越精细, 同样的屏幕区域内能显示的信息也越多, 所以分辨率是个非常重要的性能指标之一. 可以把整个图像想象成是一个大型的棋盘, 而分辨率的表示方式就是所有经线和纬线交叉点的数目.

### 分类

分辨率可以从显示分辨率和图像分辨率两个方向分类.
显示分辨率就是屏幕上显示的像素个数, 分辨率 160×128 的意思是水平方向含有像素数为 160 个, 垂直方向像素数 128 个. 以分辨率为 1024×768 的屏幕来说, 即每一条水平线上包含有 1024 个像素点, 共有 768 条线, 即扫描列数为 1024 列, 行数为 768 行. 屏幕尺寸一样的情况下, 分辨率越高, 显示效果就越精细和细腻.

图像分辨率则是单位英寸中所包含的像素点数, 其定义更趋近于分辨率本身的定义.

### 特点

高分辨率是保证彩色显示器清晰度的重要前提. 显示器的点距是高分辨率的基础之一, 大屏幕彩色显示器的点距一般为 0.28, 0.26, 0.25. 高分辨率的另一方面是指显示器在水平和垂直显示方面能够达到的最大像素点, 一般有 320×240, 640×480, 1024×768, 1280×1024 等几种, 好的大屏幕彩色显示器通常能够达到 1600×1280 的分辨率. 较高的分辨率不仅意味着较高的清晰度, 也意味着在同样的显示区域内能够显示更多的内容. 比如在 640×480 分辨率下只能显示一页内容, 在 1600×1280 分辨率下则能同时显示两页.

### 单位

描述分辨率的单位有：(dpi 点每英寸)、lpi(线每英寸)和 ppi(像素每英寸). 但只有 lpi 是描述光学分辨率的尺度的. 虽然 dpi 和 ppi 也属于分辨率范畴内的单位, 但是他们的含义与 lpi 不同. 而且 lpi 与 dpi 无法换算, 只能凭经验估算

## 刷新率

刷新率是指电子束对屏幕上的图像重复扫描的次数. 刷新率越高, 所显示的图象(画面)稳定性就越好. 刷新率高低将直接决定其价格, 但是由于刷新率与分辨率两者相互制约, 因此只有在高分辨率下达到高刷新率这样的显示器才能称其为性能优秀. 注意, 虽然它的计算单位与垂直扫描频率都是 Hz, 但这是两个截然不同的概念. 75Hz 的画面刷新率是 VESA 规定的最基本标准, 这里的 75Hz 应是所有显示模式下的都能达到的标准.

### 影响因素

#### 硬件

影响刷新率最主要的因素就是显示器的带宽, 现在一般 17 寸的彩显带宽在 100 左右, 完全能上 85Hz, 屏幕越大, 带宽越大, 19 寸的在 200 左右, 21 寸的在 300 左右, 同品牌同尺寸的彩显, 带宽越高, 价格越贵. 其次影响刷新率的还有显卡, 显卡也有可用的刷新率和分辨率, 但是就刷新率来说, 这点现在完全可以忽略不计, 因为这主要针对老一代的显卡, 现在哪怕古董级的 TNT2 显卡, 也能支持 1024x768 分辨率下达到 85Hz 的效果, 1024x768 是 17 寸 CRT 显示器的标准分辨率. 所以, 影响刷新率最主要的还是显示器的带宽.

#### 软件

影响刷新率最大的是屏幕的分辨率, 举个例子, 同样是 17 寸彩显, 带宽 108, 将分辨率调至 1024x768, 最高能达到 85Hz, 调高至 1280x1024, 最高只能达到 70Hz, 调低至 800\*600, 却能达到 100Hz. 分辨率越高, 在带宽不变的情况下, 刷新率就越低, 要想保持高刷新率, 只有采用高的带宽, 所以大屏幕显示器的带宽都很高.

#### 行频

行频是指为像管中的电子枪每秒在屏幕上从左到右扫描的次数, 单位是 Hz, 场频是指每秒钟重复绘制显示画面的次数, 单位是 Hz, 行频和场频是一台显示器的基本的电器性能.

#### 带宽

带宽代表的是显示器的一个综合指标, 也是衡量一台显示器好坏的重要指标. 带宽是指每秒钟所扫描的图像个数, 也就是说在单位时间内, 每条扫描线上显示的频点数的总和, 单位是 Hz, 带宽大小是有一定的计算方法的, 大家在选择一款显示器时, 就可以根据一些参数计算带宽, 或者根据带宽来计算一些参数. 当显示器的刷新率提高一点的话, 它的带宽就会要提高很多.

### 刷新率和帧数的关系和区别

帧数, 就是画面改变的速度, 只要显卡够强, 帧数就能很高, 只要帧数高画面就流畅. 理论上, 每一帧都是不同的画面. 60fps 就是每秒钟显卡生成 60 张画面图片. 帧数 FPS 是由显卡所决定的.

刷新率, 顾名思义, 就是显卡将显示信号输出刷新的速度. 60 赫兹(hertz)就是每秒钟显卡向显示器输出 60 次信号. 刷新率是由显示器决定的.

假设帧数是刷新率的 1/2, 那么意思就是显卡每两次向显示器输出的画面是用一幅画面. 相反, 如果帧数是刷新率的 2 倍, 那么画面每改变两次, 其中只有 1 次是被显卡发送并在显示器上显示的.

所以高于刷新率的帧数都是无效帧数, 对画面效果没有任何提升, 反而可能导致画面异常. 所以许多游戏都有垂直同步选项, 打开垂直同步就是强制游戏帧数最大不超过刷新率.

在没有垂直同步的情况下, 帧数可以无限高, 但刷新率会受到显示器的硬件限制. 所以你可以在 60 赫兹的显示器上看到 200 多帧的画面, 但是这是 200 帧的画面流畅程度其实和 60 帧是一样的, 其余 140 帧全是无效帧数, 这 140 帧的显示信号根本没有被显卡输出到显示器.

由于传统 CRT 的显示原理, 画面每刷新一次, 显示器会变暗再亮, 所以刷新率过低, 会觉得屏幕在闪. 但 LCD 的工作原理不同, 画面刷新率过低不会导致屏幕闪烁. 而帧数无论何时, 越低画面越卡, 有效帧数越高, 画面越流畅.

通常 60fps 以上的情况下, 人眼无法识别出区别, 而大多数 LCD 刷新率都在 60 赫兹以上, 所以 LCD 也不像 CRT 那样强调刷新率. 但是最新的 3D 显示效果需要 LCD 支持 120 赫兹以上的刷新率, 因为是双画面同时输出的, 120 赫兹也只同等于普通的 60 赫兹刷新率.

所以刷新率的高低决定了有效帧数的多少.

## 延迟

由于数据传输、图形计算等因素的影响, 导致用户接收到的图片或文字产品拖延的现象称之为延迟. 在虚拟现实中, 延迟可导致用户产生眩晕感, 影响虚拟现实设备的使用效果.

在虚拟现实系统中, 用户需要通过头戴显示器(虚拟现实设备)感受虚拟世界. 虚拟现实设备可以将用户与周围的现实环境隔离开, 使用户产生强烈的沉浸感. 在使用虚拟现实设备时, 为了实时更新所要显示的虚拟环境, 必须使用位置跟踪器跟着用户的头部运动. 在理想的条件下, 虚拟现实设备响应用户头部运动及更新显示内容的时间应该为零. 但是由于数据传输、图形计算等因素的影响, 会导致一定的时间延迟, 导致用户在体验虚拟现实设备时产生眩晕感.

### 延迟产生的原因

(1) 位置跟踪系统造成的. 位置跟踪器通过六自由读书决定用户头部的位置和朝向. 这些数据用于计算用户在虚拟环境中的视点, 从而实时更新虚拟现实设备所现实的虚拟场景. 位置跟踪器需要一定的时间来采集和处理这些数据, 必然就会出现一定的延迟情况发生.

(2) 应用系统造成的. 应用系统造成的延迟包括接受和处理各种外部输入数据(如处理数据手套传感器数据)、仿真过程执行时间(如各种碰撞检测等).

(3) 生成图像造成的延迟. 虚拟现实系统需要生成虚拟现实左眼和右眼的图像. 在基于虚拟现实设备的虚拟现实系统中, 生成什么样的图像由头部位置跟踪数据和具体的仿真应用共同决定. 由于虚拟现实应用系统图像计算复杂, 就会导致延迟.

(4) 显示图像造成的延迟. 生成的图像需要再虚拟现实设备中显示, 就会由此造成一定的延迟.

### 延迟的解决办法

由于不同的硬件配置和应用系统, 造成延迟的时间有几毫秒到几十毫秒不等. 而延迟会造成图像不稳定, 使用户感到恶心, 头晕等, 严重影响了虚拟现实系统的沉浸感, 所以必须要借助一定的技术手段预测延迟时间的长短, 消除由延迟造成的影响,比如 ATW 来减少视觉上的延迟, 修改系统的图形系统通过 singleBuffer 减少合成延迟

### 目前虚拟现实头盔普遍的延迟时间是多少

目前虚拟现实的延迟参数普遍为 40 毫秒, Oculus DK2 的延迟参数为 25 毫秒已经让用户的虚拟现实体验大为提升.

## 晕动症

眩晕感最基本的问题就是“视错症”(motion-sickness problem)本质上和晕车晕船晕机没有什么区别, 当人眼前所接受的视觉信号与前庭平衡器官所接受的运动信号不匹配就会引发眩晕和恶心的症状. VR 的头戴眼镜的视觉反应跟不上头部速度, 就会产生严重的眩晕. 也就是说当你的大脑从视觉系统接收到运动的景象时, 会默认是因为你的身体运动造成的. 这样会指挥你的手脚进行相应的运动. 然后小脑放射特定的生物信号来在运动中保持平衡, 如果传递的信号不一样就会发生紊乱. 举个例子, 当你坐在车里或船上的时候, 你接受到的视觉信号其实是一直运动的, 但是你的身体一直没动. 大脑并没有支配手脚运动, 没有反射电流回馈到小脑. 但小脑接收到视觉信号是动的. 动了就要保持平衡啊. 所以这家伙就给大脑回馈错误的平衡信息. 大脑一听, 不干啊, 我是大脑, 你是小脑, 比我小一级, 我还能听你的, 所以, 就拒绝. 然后身体其他器官就紊乱了. 因为管事的意见不合啊. 这就是为什么人会晕船, 晕车. 但是并不是所有人都这样, 只有一本分人小脑和大脑老闹意见. 所以有人就不晕车.

### 产生眩晕原因

#### 设备原因

显示屏屏显不够, 目前最好的屏显就是 GearVR,因为是 2K 屏, Oculus 和 3Glasses 是 1K, 所以三星的眩晕感就比 Oculus 和 3Glasses 低.

2.作为沉浸式头盔, 你接受到的信息非常的真实, 但你实际上是不动的, 所以你晕.

3. 现在的头盔延迟都没达到理想的标准, 也就是 turn around tracking , 据我所知目前做的最好的是 OCULUS, 大概 25ms, Sumsung gearvr, VRONE, 3 Glasses,以及一些国外其他头盔大概在 28ms.

4.在 3D 场景中你没有参照物, 运动时足部没有力反馈, 所以你晕, 如果配相关的力反馈手套, 跑步机应该会降低眩晕程度.

#### 游戏内容原因

1. 大部分内容都是从 PC 上直接移植过来, 所有的 UI 界面, 玩法并不适合 VR 设备. 操作变扭. 且场景视角转换不合理.
2. 游戏光线色调, 太过强烈或昏暗. 因为以前是在屏幕上距离眼睛远, 所以问题不凸显, 但是现在在头显上问题就很明显了.
3. 视角和内容不合理, 打个比方, 现实中你一直低头看楼梯, 不停的爬, 你也晕. 所以内容设计上要避免这种情况. 还有视角变换太快, 或者抖动, 人体不舒服. 比方说, 我要是在头盔玩 EVE, RUSH, 就没有 TF2(军团要塞, FPS)晕.
4. 场景不符合 VR 设备的图形畸变算法, 看起来难受. 所以你晕.

VR 眼镜目前普遍使用的是 5 寸 1080P 屏幕作为显示器, 虽然以手机的标准来讲这个屏幕的 DPI 已经够高了, 但是 VR 眼镜会让人眼离屏幕更近, 所以需要 DPI 更高. 减少纱窗效应就需要更高的 DPI, 更高的 DPI 就意味着需要更高的分辨率.

解决眩晕这个问题, 就需要在 VR 眼镜的显示刷新率和头动跟踪能力上费大力气进行技术开发, 以及相应的在内容上进行针对性的优化.

### 虚拟现实眩晕解决方法

#### 硬件解决

1. 用 2K 屏, 4K 屏, 或者用双屏及 OLED 柔性屏, 全面提升屏显、刷新频率、延迟显示等
2. 提升陀螺仪‘传感器的精确度
3. 加强数据传递速度

#### 内容解决

1. 重新设计 UI 界面符合 VR 操作
2. 提升图像畸变算法, 符合 VR 视角
3. 对玩法、操作进行修正
4. 对贴图、渲染、建模技术进行修正

## 帧数

帧数就是在 1 秒钟时间里传输的图片的量, 也可以理解为图形处理器每秒钟能够刷新几次, 通常用 fps(Frames Per Second)表示. 每一帧都是静止的图象, 快速连续地显示帧便形成了运动的假象. 高的帧率可以得到更流畅、更逼真的动画. 帧数 (fps) 越高, 所显示的动作就会越流畅. 但是文件大小会变大.

### 人眼视觉残留

#### 说法 1

是因为人眼的视觉残留特性：是光对视网膜所产生的视觉在光停止作用后, 仍保留一段时间的现象, 其具体应用是电影的拍摄和放映. 原因是由视神经元的反应速度造成的. 其时值是二十四分之一秒. 是动画、电影等视觉媒体形成和传播的根据.

#### 说法 2

当物体在快速运动时, 当人眼所看到的影像消失后, 人眼仍能继续保留其影像 1/24 秒左右的图像, 这种现象被称为视觉暂留现象. 是人眼具有的一种性质. 人眼观看物体时, 成像于视网膜上, 并由视神经输入人脑, 感觉到物体的像. 但当物体移去时, 视神经对物体的印象不会立即消失, 而要延续 1/24 秒左右的时间, 人眼的这种性质被称为 “眼睛的视觉暂留”.

### 录制视频

对于手机来说, 因为涉及机器处理图片能力和存储能力的影响, 大多数手机的视频拍摄能力无论是 720P 还是 1080P 都只有 30 帧一秒.

但是随着手机的硬件不断刷新, 现在市面上也出现很多能够高速录像的手机, 例如：

iPhone 6s 在使用 4k 格式拍摄下, 甚至可以使用 135 帧每秒的超高速拍摄功能.

Find5 也因其高强的硬件, 在通过降低到 480P 之后甚至可以录制 120 帧每秒的视频.

当然, 其实在手机上使用的高速录像并不成熟, 因为受限于硬件和储存设备写入速度等等, 拍摄效果总是不够理想. 不过这是个好的开始, 起码厂商开始重视人们对高速录像的需求, 虽然这个需求量不是很大, 但起码有……

你们可能以为高速录像除了让视频看上去更顺畅之外就没有作用了. 其实现在我们看到很多的慢速播放视频都是由高速录像拍摄下来, 然后再通过软件调慢帧数播放, 就能表现出慢动作视频了.

上面 XDA 科技说过“肉眼在看超过 24 帧每秒的静态图片就会认为是连续动态视频”, 所以你能拍摄到 60 帧每秒的视频, 然后通过软件把每秒帧数调节到 24 帧左右, 那么你在一秒钟内拍摄到的图像就能通过慢速播放成两秒钟, 而且是连续的、不会卡顿的. 如果你能拍摄 90 帧每秒的视频, 那么你起码能把一秒钟拖慢到三秒的慢动作播放, 以此类推. 高速录像能拍摄到很多我们容易忽略的细节和精采的瞬间, 通过慢动作播放视频, 也会让视频更好玩有趣, 这就是为什么越来越多厂商开始注重高速录像这个功能的原因.

VR 游戏建议帧数为 90 帧, 60FPS 对于 VR 设备来说是远远不够的, 因为这个帧数根本无法得到良好的 VR 体验. 运行 VR 最好是 120FPS, 最低也得 90FPS.

## 焦距

焦距, 是光学系统中衡量光的聚集或发散的度量方式, 指平行 光入射时从透镜光心到光聚集之焦点的距离. 亦是照相机中, 从镜片中心到底片或 CCD 等成像平面的距离. 具有短焦距的光学系统比长焦距的光学系统有更佳聚集 光的能力. 简单的说焦距是焦点到面镜的中心点之间的距离.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ZwaMr9.jpg)

### 为什么要调整焦距

调整焦距其实和调整物距是一样的道理, 只不过调整焦距还可以让戴眼镜的人士更加方便佩戴虚拟现实头盔.

### 物距、焦距和瞳距的调整：

目前大多数虚拟现实设备都可以调整瞳距和物距、焦距, 调节方法目前也分为物理调整和软件调整. 比如市面上目前能见到的 Oculus Rift DK2 内置转轮让你调整焦距的, 而三星的 Gear VR 这款头盔是调节物距, 这俩可以使得虚拟环境在你眼前变得尽可能清晰. Oculus Rift 包含两副分开的镜头：一副是面向普通用户或轻度近视的用户, 另一副面向近视较深的用户.

## 物距

在物理学中, 物距就是指物体到透镜光心的距离. 用英文字母 u 表示. 对于透镜而言, 通过光心且与光轴垂直的平面, 即是物方主平面也是像方主平面重合. 物距与像距存在共轭关系, 物距越远, 像距越近;相反, 物距越近, 像距越远. 在进行光学计算时, 严格地讲, 物距应为被摄体平面与镜头前主面间的距离.

### 为什么要调整物距

调整物距可以让玩家在 VR 眼镜上看的虚拟环境在你眼前变得尽可能清晰, 根据每个人的视力不同来进行调节, 它是让屏幕与透镜直接产生距离.

### 物距、焦距和瞳距的调整

目前大多数虚拟现实设备都可以调整瞳距和物距、焦距, 调节方法目前也分为物理调整和软件调整. 比如市面上目前能见到的 Oculus Rift DK2 内置转轮让你调整焦距的, 而三星的 Gear VR 这款头盔是调节物距, 这俩可以使得虚拟环境在你眼前变得尽可能清晰. Oculus Rift 包含两副分开的镜头：一副是面向普通用户或轻度近视的用户, 另一副面向近视较深的用户.

## 瞳距

瞳距就是瞳孔的距离, 正常人的双眼注视同一物体, 物体分别在两眼视网膜处成像, 并在大脑视中枢重叠起来, 成为一个完整的、具有立体感的单一物体, 这个功能叫双眼单视. 但是, 婴幼儿在双眼单视形成过程中, 很容易受外界因素影响, 致使一眼注视目标, 另一眼偏斜而不能往同一目标上看, 于是就产生了斜视. 医学上将眼球注视物体时向内侧斜视, 称为内斜, 也就是人们俗称的“斗鸡眼”.

配戴眼镜时需要测量瞳距, 瞳距分为：远用瞳距, 近用瞳距, 常用瞳距. 测定时, 是按一定的距离测出这三种瞳距的.

对于近视眼或者远视眼患者, 配眼镜时, 需要考虑这个参数. 即两块镜片中心的距离(光学中心距离)应当与患者的瞳距相配合, 否则, 即使度数正确, 患者戴上眼镜后也会有不适的感觉, 并且影响视力.

## 色差

色差(Chromatic aberration;chromatic aberration)： 色差又称色像差, 是透镜成像的一个严重缺陷, 色差简单来说就是颜色的差别, 发生在以多色光为光源的情况下, 单色光不产生色差.

CA(Chromatic Aberration)即色差, CA(Area)值用来衡量图像的色差水平, 这个值越低说明品质越好. 0-0.5：可以忽略, 肉眼难以辨认出; 0.5-1.0：很低, 只有受过长期专业训练的人才能勉强发现;1.0-1.5：中等, 高倍率输出时时常看到, 中等镜头的表现;大于 1.5：严重, 高倍率输出时非常明显, 镜头表现糟糕.

### 色差分类

(一)不同波长的光将以不同的程度色散. 白光被色散为紫外波段、可见波段和红外波段范围的各种波长的光, 通过透镜时所成的像便带有彩色边缘, 即为色差. 光学系统的实际成像与理想成像的差别, 统称为像差. 色差是像差中的一种, 是因透射材料的透射率随波长不同而不同造成的, 故只有对多色光才显现出来. 用不同的玻璃材料制成的凹凸镜组合可以消除色差.

(二)定量表示的色知觉差异. 从明度、色调和彩度这三种颜色属性的差异来表示. 明度差表示深浅的差异, 色调差表示色相的差异(即偏红或偏蓝等), 彩度差表示鲜艳度的差异. 色差的评定在工业和商业中非常重要, 主要应用于生产中的配色和产品的颜色质量控制. 现代色差评定根据国际照明协会(CIE)推荐的标准色差公式并采用仪器和电脑测量计算, 用精确的数字来表示.

(三)染同一颜色的革, 其批与批之间出现颜色不一致, 或者同一转鼓、同一次染色的革出现几种颜色差别的现象称为色差. 特别是绒面革更易出现色差. 可指同一张皮革不同部位的色泽差别, 也可指同一批加工皮革之间存在的颜色差异, 还可指原定染同一颜色之不同批次皮革间的颜色差别.

## PPI

Pixels Per Inch 所表示的是每英寸所拥有的像素(Pixel)数目. 因此 PPI 数值越高, 即代表显示屏能够以越高的密度显示图像. 当然, 显示的密度越高, 拟真度就越高.

Pixels Per Inch 是图像分辨率的单位, 图像 PPI 值越高, 画面的细节就会越丰富, 因为单位面积的像素数量更多, 所以数码相机拍出来的图片因品牌或生产时间不同可能 有所不同, 常见的有 72PPI, 180PPI 和 300PPI,默认出来就是这么多(A710 拍出的是 180PPI). DPI(Dots Per Inch)是指输出分辨, 针对于输出设备而言的, 一般的激光打印机的输出分辨率是 300DPI-600DPI, 印刷的照排机达到 1200DPI-2400DPI, 常见的冲印一般在 150DPI 到 300DPI 之间.

## 畸变

畸变指畸形地变化. 在虚拟现实里指图像在最大化的覆盖人的视觉范围时有没有扭曲.

在虚拟现实系统中是指虚拟现实设备镜片畸变. 为了让用户在视觉上拥有真实的沉浸感, 虚拟现实设备就要尽可能的覆盖人眼的视觉范围, 因此就需要在虚拟现实设备装一个特定的球面弧度镜片, 但是利用弧形镜片将传统的图像投射到人的眼中时, 图像是扭曲的, 人眼就没有办法获得虚拟空间中的定位, 即在虚拟现实中你的周边都是哈哈镜的空间, 四周都是扭曲的图像. 要解决这个问题, 就要先扭转图像, 通过特定的算法生成畸变镜片对应的畸变图像, 然后这些畸变图像在经过畸变镜片投射到人眼之后, 就会变成正常的图像, 从而让人感觉到真实的位置投射以及大视角范围的覆盖.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ETnsTb.jpg)

## 视场角

视场角, 英文 field of view, 简称 FOV. 在显示系统中, 视场角就是显示器边缘与观察点(眼睛)连线的夹角.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/hfViPF.jpg)

## 渲染

渲染(Render)在电脑绘图中是指：用软件从模型生成图像的过程. 模型是用严格定义的语言或者数据结构对于三维物体的描述, 它包括几何、视点、纹理以及照明信息. 图像是数字图像或者位图图像. 渲染这个术语类似于“艺术家对于场景的渲染”. 另外渲染也用于描述：计算视频编辑文件中的效果, 以生成最终视频输出的过程.

## OLED

有机发光二极管又称为有机电激光显示(Organic Light-Emitting Diode, OLED), OLED 显示技术具有自发光的特性, 采用非常薄的有机材料涂层和玻璃基板, 当有电流通过时, 这些有机材料就会发光.

## Micro LED

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/ZZQZRu.jpg)

Micro-LED, 顾名思义, 就是特别小的 LED. 又称微型发光二极管, 是指高密度集成的 LED 阵列, 阵列中的 LED 像素点距离在 10 微米量级, 每一个 LED 像素都能自发光.

## 余晖效应

视觉暂留现象（Visual staying phenomenon, duration of vision）又称“余晖效应”. 人眼在观察景物时, 光信号传入大脑神经, 需经过一段短暂的时间, 光的作用结束后, 视觉形象并不立即消失, 这种残留的视觉称“后像”, 视觉的这一现象则被称为“视觉暂留”.

# 深度技术

## 力反馈

所谓力反馈（Force Feedback）, 本来是应用于军事上的一种虚拟现实技术, 它利用机械表现出的反作用力, 将游戏数据通过力反馈设备表现出来, 可以让用户身临其境地体验游戏中的各种效果.

## Sdk

软件开发工具包(外语首字母缩写：SDK、外语全称：Software Development Kit)一般都是一些软件工程师为特定的软件包、软件框架、硬件平台、操作系统等建立应用软件时的开发工具的集合.

## ATW

异步时间扭曲(Asynchronous Timewarp 简称 ATW)是一种生成中间帧的技术, 当游戏不能保持足够帧率的时候, ATW 能产生中间帧, 从而有效减少游戏画面的抖动.

实现 ATW 是有挑战性的, 主要有两个原因：

1： 它需要 GPU 硬件支持合理的抢占粒度.

2： 它要求操作系统和驱动程序支持使 GPU 抢占.

让我们从抢占粒度开始, 在 90 赫兹, 帧之间的间隔大约是 11ms(1/90), 这意味着为了使 ATW 有机生成一帧, 它必须能够抢占渲染线程并且运行 时间少于 11ms, 然而 11ms 实际上不够好, 如果 ATW 在一帧时间区间内任意随机点开始运行, 那么起潜伏期(执行和帧扫描之间的时间)也将随机, 我们 需要确保我们不跳跃任何游戏渲染的帧.

我们真的期望 ATW 运行一直非常的短, 短到在视频卡产生新的一帧之前结束, 刚好有足够的时间来完成中间帧的生成, 缺少自定义的同步 ATW 中断例程, 我们可以获得高优先级抢占粒度和调度, 在最长 2ms 或更少的时间内.

原来, 对现在的图形卡和驱动实现来说, 2ms 抢占是一个艰巨的任务, 虽然许多 GPU 支持有限的形式的抢占, 但执行存在显著差异.

1： 一些显卡实现厂商和驱动程序允许抢占任一批处理或回执调用粒度, 虽然有帮助, 但不是十分完美(举一个极端的例子, 一个复杂的并包含很多绘制指令着色器可以很容易在 10ms 完成).

2： 其他显卡实现厂商和驱动程序允许抢占计算着色器, 但需要特定扩展来支持.

如果抢占操作不是很快, 则 ATW 将无法抢在画面同步之前生成中间帧. 这样, 最后一帧将会再显示, 将导致抖动, 这意味着一个正确的实现应该能够抢占和恢复任意渲染操作, 和管线状态. 理论上讲, 甚至三角抢占(triangle-granularity) 不够好, 因为我们不知道一个复杂着色器执行将花多长时间. 我们正与 GPU 制造商来实现更好的抢占, 但是在这之前确实要因为这个问题花费一定时间.

另外一方面是操作系统对抢占的支持, 在 Windows8 之前, Windiows 显示驱动模型(WDDM)支持使用“批处理队列”粒度的有限抢占, 对于内奸的图形驱动程序, 很不幸, 图形驱动程序趋向于大批量渲染效率, 导致支持 ATW 太粗糙.

对于 Windows8, 改善了 WDDM1.2 支持更细的抢占粒度, 然而, 这些抢占模式不被图形驱动程序普遍支持, 渲染管线将在 Windows 10 或 DirectX12 中得到显著提升. 这为开发人员提供了较低级别的渲染控制, 这是一个好消息, 但直到 Windows10 变 为主流之前, 我们还是没有标准的方式来支持渲染抢占, 造成的结果是, ATW 需要特定显卡驱动的扩展.

ATW 是有用的, 但不是万能的.

一旦我们普遍实现了 GPU 渲染管线管理和任务抢占, ATW 可能成为另一种工具来帮助开发人员提高性能和减少虚拟现实的抖动, 然而, 由于我们这里 列出的挑战的问题, ATW 不是万能的, VR 的应用本身最好是维持较高的帧率, 以提供最好的渲染质量. 最坏的情况, ATW 生成的中间帧也可以导致用户有 不舒服的感受, 换句话说, ATW 无法根本解决这种不舒服.

根据生成中间帧的复杂性来说, ATW 很显然表明, 甚至是位置时间扭曲, ATW 不会成为一个完美的通用的解决方案, 这意味着只有方向 ATW 和位 置 ATW 还算是可以的, 填充帧时偶尔会有跳跃. 为了产生一个舒适, 令人信服的虚拟现实, 开发人员仍然需要保持帧率在 90 赫兹.

试图支持传统显示器和 VR 双模式将会面临很大性能困难, 这种巨大的性能要求是对引擎的伸缩性的考验, 对于开发人员遇到的这种情况, ATW 可能看起来很有吸引力, 如果达到 90 赫兹的频率, 将使 VR 具有很好的舒适性, 这是 VR 存在的真正魅力.

## 虚拟全景

虚拟全景又称三维全景虚拟现实（也称实景虚拟）是基于全景图像的真实场景虚拟现实技术. 全景（英文名称是 Panorama）是把相机环 360 度拍摄的一组或多组照片拼接成一个全景图像, 通过计算机技术实现全方位互动式观看的真实场景还原展示方式.

## 全息投影

全息投影技术一般指全息投影. 全息投影技术（front-projected holographic display）也称虚拟成像技术是利用干涉和衍射原理记录并再现物体真实的三维图像的技术. 全息投影技术不仅可以产生立体的空中幻像, 还可以使幻像与表演者产生互动, 一起完成表演, 产生令人震撼的演出效果. 适用范围产品展览、汽车服装发布会、舞台节目、互动、酒吧娱乐、场所互动投影等.

## 立体显示

立体显示是虚拟现实的一个实现方式. 立体显示主要有以下几种方式：双色眼镜、主动立体显示、被动同步的立体投影设备、立体显示器、真三维立体显示、其它更高级的设备.

## 眼球追踪

眼球追踪是一项科学应用技术, 用户无需触摸屏幕即可翻动页面. 从原理上看, 眼球追踪主要是研究眼球运动信息的获取、建模和模拟, 用途颇广. 而获取眼球运动信息的设备除了红外设备之外, 还可以是图像采集设备, 甚至一般电脑或手机上的摄像头, 其在软件的支持下也可以实现眼球跟踪.

## 人机交互

人机交互技术（Human-Computer Interaction Techniques）是指通过计算机输入、输出设备, 以有效的方式实现人与计算机对话的技术.

## 动作捕捉

动作捕捉, 英文 Motion capture,简称 Mocap. 简单来说就是把人物动作数字化, 把数字化动作应用到不同行业, 比如拍电影, 做动画, 做游戏”.

动作捕捉技术涉及尺寸测量、物理空间里物体的定位及方位测定等方面可以由计算机直接理领域. 解处理的数据. 在运动物体的关键部位设置跟踪器, 由 Motion capture 系统捕捉跟踪器位置, 再经过计算机处理后得到三维空间坐标的数据. 当数据被计算机识别后, 可以应用在动画制作, 步态分析, 生物力学, 人机工程等.

## 可视化

可视化(Visualization)是利用计算机图形学和图像处理技术, 将数据转换成图形或图像在屏幕上显示出来, 并进行交互处理的理论、方法和技术. 它涉及到计算机图形学、图像处理、计算机视觉、计算机辅助设计等多个领域, 成为研究数据表示、数据处理、决策分析等一系列问题的综合技术.

# 虚拟现实引擎

## 游戏引擎

游戏引擎是指一些已编写好的可编辑电脑游戏系统或者一些交互式实时图像应用程序的核心组件. 这些系统为游戏设计者提供各种编写游戏所需的各种工具, 其目的在于让游戏设计者能容易和快速地做出游戏程式而不用由零开始. 大部分都支持多种操作平台, 如 Linux、Mac OS X、微软 Windows.

游戏引擎是一个游戏的重要核心, 它既是建立游戏的基础, 也是控制游戏每一个细节的指挥官, 不论是游戏场景中的一个不起眼亮点, 还是气势宏伟的场景视觉特效. 不同的游戏引擎所能实现的功能也不尽相同, 而且用不同的引擎所制作出来的游戏对于运行的系统平台性能需求也有较大的差异.

## Unity

Unity 是由 Unity Technologies 开发的一个让玩家轻松创建诸如三维视频游戏、建筑可视化、实时三维动画等类型互动内容的综合型游戏开发创作工具. Unity(游戏引擎)一般指 Unity3D.

Unity 是由 Unity Technologies 开发的一个让玩家轻松创建诸如三维视频游戏、建筑可视化、实时三维动画等类型互动内容的综合型游戏开发创作工具, 是一个全面整合的专业游戏引擎. Unity 类似于 Director, Blender, Virtools 或 Torque Game Builder 等利用交互的图型化开发环境为首要方式的软件其编辑器运行在 Windows 和 Mac OS X 下, 可发布游戏至 Windows、Mac、Wii、iPhone、Windows phone 8 和 Android 平台. 也可以利用 Unity web player 插件发布网页游戏, 支持 Mac 和 Windows 的网页浏览. 它的网页播放器也被 Mac widgets 所支持. Unity 分成 Free 与 Pro 版. Free 版提供试用 30 天 Pro 版的功能.

## Unreal

Unreal：UNREAL ENGINE 的简写, 是目前世界最知名授权最广的顶尖游戏引擎, 占有全球商用游戏引擎 80%的市场份额.

## Source

Source 引擎是由 Valve 电子软件公司开发的 3D 绘图引擎, 开发时是为了使用它制作半条命 2, 其中与用了大量的 3D 设置, 并且对其他的游戏开发者开放授权. 这个引擎提供关于渲染、声效、动画、消锯齿、界面、美工创意和物理模拟方面的支持.

## Virtools

Virtools 是一套整合软件, 可以将现有常用的档案格式整合在一起, 如 3D 的模型、2D 图形或是音效等. Virtools 是一套具备丰富的互动行为模块的实时 3D 环境虚拟实境编辑软件, 可以让没有程序基础的美术人员利用内置的行为模块快速制作出许多不同用途的 3D 产品, 如网际网络、计算机游戏、多媒体、建筑设计、交互式电视、教育训练、仿真与产品展示等 .

## Unigine

该引擎包含了逼真的三维渲染, 强大的物理模块, 对象具有非常丰富的图书馆导向的脚本系统, 全功能的 GUI 模块, 声音子系统, 以及灵活的工具. 高效率和良好架构的框架, 支持多核系统, 使 Unigine 具有一个高度可扩展的解决方案, 对其中的多平台类型游戏的影响颇多.

## Converse3D

Converse3D 虚拟现实引擎是由北京中天灏景网络科技有限公司自主研发的具有完全知识产权的一款三维虚拟现实平台软件, 可广泛的应用于视景仿真、城市规划、室内设计、工业仿真、古迹复原、娱乐、艺术与教育等行业. 该软件适用性强、操作简单、功能强大、Converse3D 虚拟现实引擎的问世给中国的虚拟现实技术领域注入了新的生命力.

## GetReal3D For Unity

GetReal3D For UnityTM 软件是美国 Mechdyne 公司针对 Unity 虚拟交互引擎专门定制开发的功能扩展插件, 它能够实现将 Unity 游戏引擎所创建的交互内容与虚拟现实、仿真训练等应用的无缝对接, 支持将 Unity 引擎开发的应用程序发布到沉浸式显示系统.

## Project Tango

Project Tango 是谷歌公司的一项研究项目, 2014 年 2 月谷歌已经成功为该项目研发出了一款 Android 手机原型机, 配备了一系列摄像头、传感器和芯片, 能实时为用户周围的环境进行 3D 建模.