当我们看视频时会看到很多图像，这些图像的展现形式我们中学学习几何课程的时候就接触过，是由一个个像素点组成的线，又由一条条线组成面，这个面铺在屏幕上展现出来的就是我们看到的图像。

这些图像有黑白的，也有彩色的。这是因为图像输出设备支持的规格不同，所以色彩空间也有所不同，不同的色彩空间能展现的色彩明暗程度，颜色范围等也不同。下面是一些常见的色彩格式：

• GRAY 色彩空间
• YUV 色彩空间
• RGB 色彩空间
• HSL 和 HSV 色彩空间

## GRAY 灰度模式表示

这一模式为 8 位展示的灰度，取值 0 至 255，表示明暗程度，0 为最黑暗的模式，255 为最亮的模式,也就是黑白电视的显示模式。

由于每个像素点是用 8 位深展示的，所以一个像素点等于占用一个字节，一张图像占用的存储空间大小计算方式也比较简单:

`图像占用空间 = 图像宽度(W) x 图像宽度(H) x 1`

如果图像为 352x288 的分辨率，那么一张图像占用的存储空间应该是 352x288， 也就是 101376 个字节大小。

## YUV 色彩表示

在视频领域，通常以 YUV 的格式来存储和显示图像。其中 Y 表示视频的灰阶值，也可以理解 为亮度值，而 UV 表示色彩度，如果忽略 UV 值的话，我们看到的图像与前面提到的 GRAY 相同，为黑白灰阶形式的图像。YUV 最大的优点在于 **每个像素点的色彩表示值占用的带宽或者存储空间非常少** 。

下面是原图与 YUV 的 Y 通道、U 通道和 V 通道的图像示例

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/DGguAz.png)

为节省带宽起见，大多数 YUV 格式平均使用的每像素位数都少于 24 位。主要的色彩采样格 式有 YCbCr 4:2:0、YCbCr 4:2:2、YCbCr 4:1:1 和 YCbCr 4:4:4。YUV 的表示法 也称为 A:B:C 表示法。

YUV 存储方式主要分为两大类:Planar 和 Packed 两种。Planar 格式的 YUV 是先连续 存储所有像素点的 Y，然后接着存储所有像素点的 U，之后再存储所有像素点的 V，也可 以是先连续存储所有像素点的 Y，然后接着存储所有像素点的 V，之后再存储所有像素点 的 U。Packed 格式的 YUV 是先存储完所有像素的 Y，然后 U、V 连续的交错存储。

为了便于理解，下面我以 352x288 的图像大小为例，来详细介绍一下各个采样格式的区别。

### YUV 4:4:4 格式

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/pOhjrk.png)

yuv444 表示 4 比 4 比 4 的 yuv 取样，水平每 1 个像素(即 1x1 的 1 个像素)中 y 取样 1 个，u 取样 1 个，v 取样 1 个，所以每 1x1 个像素 y 占有 1 个字节，u 占有 1 个字节，v 占有 1 个字节，平均 yuv444 每个像素所占位数为:

> bpp: bits -per - pixel,代表每一个像素的数据量 <br>
> pix:pixel,像素

$$
\left ( 1+  1+  1 \right ) \times 8bit {\div} 1pix = 24bpp
$$

那么 352x288 分辨率的一帧图像占用的存储空间为:

$$
352\times 288\times 24\left ( bit \right ) {\div} 8\left ( bit \right ) = 304128\left ( 字节 \right )
$$

### YUV 4:2:2 格式

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/c8tFMk.png)

yuv422 表示 4 比 2 比 2 的 yuv 取样，水平每 2 个像素(即 2x1 的 2 个像素)中 y 取样 2 个，u 取样 1 个，v 取样 1 个，所以每 2x1 个像素 y 占有 2 个字节，u 占有 1 个字节，v 占有 1 个字节，平均 yuv422 每个像素所占位数为:

$$
\left ( 2+  1+  1 \right ) \times 8bit {\div} 2pix = 16bpp
$$

那么 352x288 分辨率的一帧图像占用的存储空间为:

$$
352\times 288\times 12\left ( bit \right ) {\div} 8\left ( bit \right ) = 152064\left ( 字节 \right )
$$

### YUV 4:2:0 格式

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/HVVYjZ.png)

yuv420 表示 4 比 2 比 0 的 yuv 取样，水平每 2 个像素与垂直每 2 个像素(即 2x2 的 2 个像 素)中 y 取样 4 个，u 取样 1 个，v 取样 1 个，所以每 2x2 个像素 y 占有 4 个字节，u 占有 1 个字节，v 占有 1 个字节，平均 yuv420 每个像素所占位数为:

$$
\left ( 4+  1+  1 \right ) \times 8bit {\div} 4pix = 12bpp
$$

那么 352x288 分辨率的一帧图像占用的存储空间为:

$$
352\times 288\times 12\left ( bit \right ) {\div} 8\left ( bit \right ) = 152064\left ( 字节 \right )
$$

为了方便理解 YUV 在内存中的存储方式，我们以宽度为 6、高度为 2 的 yuv420 格式为例， 一帧图像读取和存储在内存中的方式如图:

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/jm2g3l.png)

## YUV 类型和存储方式

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/MXXOJQ.png)

## RGB 色彩表示

三原色光模式(RGB color model)，又称 RGB 颜色模型或红绿蓝颜色模型，是一种加色模 型，将红(Red)、绿(Green)、蓝(Blue)三原色的色光按照不同的比例相加，来合成各种色彩光。

每象素 24 位编码的 RGB 值:使用三个 8 位无符号整数(0 到 255)表示红色、绿色和蓝色 的强度。这是当前主流的标准表示方法，用于交换真彩色和 JPEG 或者 TIFF 等图像文件格式 里的通用颜色。它可以产生一千六百万种颜色组合，对人类的眼睛来说，其中有许多颜色已经 无法确切地分辨了。

使用每原色 8 位的全值域，RGB 可以有 256 个级别的白 - 灰 - 黑深浅变化，255 个级别的红 色、绿色和蓝色以及它们等量混合的深浅变化，但是其他色相的深浅变化相对要少一些。

RGB 常见的展现方式分为 16 位模式和 32 位模式(32 位模式中主要用其中 24 位来表示 RGB)。16 位模式(RGB565、BGR565、ARGB1555、ABGR1555)分配给每种原色各为 5 位，其中绿色为 6 位，因为人眼对绿色分辨的色调更敏感。但某些情况下每种原色各占 5 位，余下的 1 位不使用或者表示 Alpha 通道透明度。

32 位模式(ARGB8888)，实际就是 24 位模式，余下的 8 位不分配到象素中，这种模式是 为了提高数据处理的速度。同样在一些特殊情况下，在有些设备中或者图像色彩处理内存中， 余下的 8 位用来表示象素的透明度(Alpha 通道透明度)。

但是，需要注意的 是 RGB 图像像素中 R、G、B 三个值并不一定是按 R、G、B 顺序排列的，也有可能是 B、 G、R 顺序排列。

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/8MDuIw.png)

## HSL 与 HSV 色彩表示

HSL 和 HSV 是将 RGB 色彩模型中的点放在圆柱坐标系中的表示法，在视觉上会比 RGB 模型更加直观

HSL，就是色相(Hue)、饱和度( Saturation)、亮度( Lightness)。HSV 是色相 (Hue)、饱和度( Saturation)和明度(Value)。色相(H)是色彩的基本属性，就是平常 我们所说的颜色名称，如红色、黄色等;饱和度(S)是指色彩的纯度，越高色彩越纯，低则 逐渐变灰，取 0~100% 的数值;明度(V)和亮度(L)，同样取 0~100% 的数值。

## 图像的色彩空间

了解了视频和图像的集中色彩表示方式，那我们是不是用相同的数据格式就能输出颜色完全一样的图像呢?不一定，你可以观察一下电视中的视频图像、电脑屏幕中的视频图像、打印机打印出来的视频图像，同一张图像会有不同的颜色异，甚至不同的电脑屏幕看到的视频图像、不同的电视看到的视频图像，有时也会存在颜色差异

主 要是因为图像受到了色彩空间参数的影响。我们这里说的色彩空间也叫色域，指某种表色模式 用所能表达的颜色构成的范围区域。而这个范围，不同的标准支持的范围则不同，下面，我们 来看三种范围，分别为基于 CIE 模型表示的 BT.601、BT.709 和 BT.2020 范围。

## YUV 与 MP4、H.264、RTMP 之间是什么样的关系

- YUV 表示原始数据.
- H.264 表示视频的编码格式.
- MP4 表示封装格式，可以直接观看.
- RTMP:常用的直播传输协议.

## RGB 和 YUV 之间的转换

采集到的原始图像、给显示器渲染的最终图像都是 RGB 图像，但是视频编码一般用的是 YUV 图像,那么这中间一定少不了两者的相互转换

### Color Range

Color Range 分为两种，一种是 Full Range，一种 是 Limited Range。Full Range 的 R、G、B 取值范围都是 0~255。而 Limited Range 的 R、G、B 取值范围是 16~235

BT709 和 BT601 定义了一个 RGB 和 YUV 互转的标准规范。只有我们都按照标准 来做事，那么不同厂家生产出来的产品才能对接上。BT601 是标清的标准，而 BT709 是高清的标准。

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/TPwCFf.png)

## 总结

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/8Wxgjz.png)