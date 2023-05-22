---
title: Shader(三)-颜色
urlname: shader-color
date: 2023/05/05
tags:
  - Shader
---

颜色一般我们会使用 rgb 来读取，同样的，也可以使用 xyz 来读取:

```glsl
vec4 vector;
vector[0] = vector.r = vector.x = vector.s;
vector[1] = vector.g = vector.y = vector.t;
vector[2] = vector.b = vector.z = vector.p;
vector[3] = vector.a = vector.w = vector.q;
```

## 混合颜色

GLSL 中的 mix 函数是用于线性插值（Lerp）两个值的函数，它的原理是通过加权平均的方式将两个值混合在一起。

具体而言，对于两个值 x 和 y 以及一个插值因子 a，mix 函数的计算公式为：

```glsl
mix(x, y, a) = x * (1 - a) + y * a
```

其中，插值因子 a 的取值范围为[0, 1]，表示 x 和 y 在混合中的权重比例。当 a=0 时，mix 函数返回值为 x；当 a=1 时，返回值为 y；当 a 在[0, 1]之间时，返回值则为 x 和 y 的线性插值结果。

在颜色混合的应用中，通常将两个颜色值作为 x 和 y 输入到 mix 函数中，然后通过改变插值因子 a 的值来控制两个颜色之间的混合程度。例如，当 a=0.5 时，返回值为两个颜色值的平均值，即两个颜色的中间色。

所以可以理解 a 为 0 到 1，结果就是 x 到 y。

## 渐变

mix() 函数有更多的用处。我们可以输入两个互相匹配的变量类型而不仅仅是单独的 float 变量，在我们后面例子中用的是 vec3。这样我们便获得了混合颜色单独通道 .r，.g 和 .b 的能力。

本图来自 thebookofshaders

![](images/opengl/mix-vec.jpg)

举一个例子，现在我们想要实现一个这样的效果：

![](images/opengl/sun.gif)

它的效果是一座山，山后边是太阳光照射过来时形成的光晕。看到这样一个效果，你觉得应该从何入手？

首先，需要确定这个曲线的函数，曲线可以自己选择，这里选择一条 quaImpulse

```glsl
float quaImpulse(float k, float x) {
    return 2.0 * sqrt(k) * x / (2.0 + k * x * x);
}
```

然后在曲线附近进行渐变的处理，可以使用 smoothstep 来处理

```glsl
vec3 pct = vec3(st.x);
pct.b = quaImpulse(10.0, st.x);
color = mix(color, colorB, smoothstep(pct.b + 0.2, pct.b - 0.2, st.y));
```

这段代码的意思就是当在曲线 0.2 的上下附近时，是这两个颜色的渐变效果，否则各自一个颜色，这样就实现了绿色的山的效果。

其次需要实现后面的阳光的渐变效果，一个是需要根据时间来变化，我们通过 `abs(sin(iTime))`来获取，另一个是只需要上半部分的颜色进行渐变，山的颜色是不变的，因此首先需要限制为曲线的上半部分，然后根据时间转换后的比例来进行颜色变换，代码如下：

```glsl
float step = smoothstep(pct.b , 1., st.y);
color = mix(color, colorC, step * pctAbove);
```

完整代码可参考:[sun.glsl](https://github.com/mikaelzero/Shader/blob/main/src/3.sun.glsl)

## 彩虹和五彩旗

为了增加对 mix 函数的掌握程度，可以想一想，一个彩虹效果用 mix 应该如何实现呢？

![](images/opengl/shader_rainbow.png)

完整代码可参考:[rainbow.glsl](https://github.com/mikaelzero/Shader/blob/main/src/4.rainbow.glsl)

或者一个五彩旗

![](images/opengl/shader_flag.png)

完整代码可参考:[flag.glsl](https://github.com/mikaelzero/Shader/blob/main/src/5.flag.glsl)

## HSB

HSB 是一种颜色表示方式，它由三个参数组成：色相（Hue）、饱和度（Saturation）和亮度（Brightness），其中 HSB 是缩写，也称为 HSV（Value 代替了 Brightness）。HSB 通常用于交互式应用程序和用户界面中，因为它提供了一种直观的方式来控制颜色。

以下是 HSB 参数的解释：

1. 色相（Hue）：它表示颜色的基本色调。就是平常所说的颜色名称，如红色、黄色等。色相以 0 到 360 度的角度表示，其中**红色在 0 度，绿色在 120 度，蓝色在 240 度**。
2. 饱和度（Saturation）：它表示颜色的纯度或强度。饱和度的值从 0%（灰色）到 100%（全彩色）变化。
3. 亮度（Brightness 或 Value）：它表示颜色的明暗程度，取值范围从 0%（黑色）到 100%（白色）。

在计算机图形学中，HSB 通常被转换为 RGB（红、绿、蓝）颜色空间，因为大多数图形硬件都使用 RGB 颜色空间来显示颜色。通常使用以下公式将 HSB 转换为 RGB：

- 计算饱和度和亮度的百分比值。
- 根据色相计算对应的红、绿、蓝分量的值。
- 将所有三个分量的值限制在 0 到 255 的范围内，以保证它们在 RGB 颜色空间中的有效值。

```glsl
//  Function from Iñigo Quiles
//  https://www.shadertoy.com/view/MsS3Wc
vec3 hsb2rgb( in vec3 c ){
    vec3 rgb = clamp(abs(mod(c.x*6.0+vec3(0.0,4.0,2.0),
                             6.0)-3.0)-1.0,
                     0.0,
                     1.0 );
    rgb = rgb*rgb*(3.0-2.0*rgb);
    return c.z * mix(vec3(1.0), rgb, c.y);
}

void main(){
    vec2 st = gl_FragCoord.xy/u_resolution;
    vec3 color = vec3(0.0);

    // We map x (0.0 - 1.0) to the hue (0.0 - 1.0)
    // And the y (0.0 - 1.0) to the brightness
    color = hsb2rgb(vec3(st.x,1.,st.y));

    gl_FragColor = vec4(color,1.0);
}
```

这段代码来自 [thebookofshaders.com](https://thebookofshaders.com/06/?lan=ch),作用就是将 x 方向映射为色相，y 方向映射为亮度，效果如下:

![](images/opengl/shader_hsb.png)

## HSB 转 RGB

```glsl
// Function from Iñigo Quiles
// https://www.shadertoy.com/view/MsS3Wc
vec3 hsb2rgb(in vec3 c) {
    vec3 rgb = clamp(abs(mod(c.x * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0) - 1.0, 0.0, 1.0);
    rgb = rgb * rgb * (3.0 - 2.0 * rgb);
    return c.z * mix(vec3(1.0), rgb, c.y);
}
```

这个函数实现了 HSB 到 RGB 的转换:

1. 通过映射 Hue 计算主色索引
2. 根据索引和饱和度求得 RGB 值
3. 再根据亮度缩放 RGB,得到最终结果

下面具体分析下源码的逻辑:

1. Hue 值通常的范围是 0 到 1，但为了方便计算 RGB 值,我们需要将 Hue 值映射到 0 到 6 之间,对应色相环上 6 个主色的索引。，首先传入的 vec3 都是 0 到 1 的范围，这个在传参时已经确定,代表了 HSB 三个值
2. 之后 c.x \* 6.0, 把 Hue 从 0-1 映射到 0-6
   1. 将 Hue 从 0-1 映射到 0-6 ,是为了便于计算出对应的 RGB 值
   2. 色相环是一个将色彩等间隔分布在圆周上的模型。其中主要有 6 种主色:红绿蓝青紫橙。将 Hue 映射到 0-6 ,就代表色相环上这 6 种主色的索引。通过这个索引,便可以计算出对应的 RGB 值。
   3. 色相环模型是这样的:红色为 0°（360°）；黄色为 60°；绿色为 120°；青色为 180°；蓝色为 240°；品红色为 300°
3. 加上 vec3(0.0, 4.0, 2.0)代表了这个分量每个都加上前面 c.x \* 6.0 的值
4. 取余数 mod(......, 6) ,将向量的每个分量都限制在 0-6 的数值,这就是 Hue 映射到 0-6 后,对应的色轮主色角度索引。然后,Hue 值代表色相环上的角度。如:
   1. 1/6 到 2/6 之间,索引为 1(对应黄色)
   2. 2/6 到 3/6 之间,索引为 2(对应绿色)
   3. 3/6 到 4/6 之间,索引为 3(对应青色)
   4. 4/6 到 5/6 之间,索引为 4(对应蓝色)
   5. 5/6 到 1 之间,索引为 5(对应紫色)
5. 根据索引,使用分段函数计算出 RGB 值，S 代表饱和度:
   1. 如果索引为 0:R = 1,G = 1-S,B = 1-S(对应红色)
   2. 如果索引为 1:R = 1-S,G = 1,B = 1-S(对应黄色)
   3. 如果索引为 5:R = 1-S,G = 1-S, B = 1(对应紫色)
6. 计算出一个范围在-3 到 3 之间的调整后索引: mod(...) - 3.0
7. 根据这个调整后索引,计算出一个 0 到 2 之间的 RGB 值: abs(...) - 1.0
8. 将 RGB 值限制在 0 到 1 范围内: clamp(...)
9. 根据饱和度 S 缩放 RGB 值: rgb _ rgb _ (3.0 - 2.0 \* rgb)
10. 最后根据亮度 B 缩放 RGB 值: c.z \* mix(vec3(1.0), rgb, c.y)

## 色轮

![](images/opengl/shader_color_wheel.png)

极坐标下的色轮表示

![](images/opengl/polar_coordinate.svg)

下面的代码就是实现了一个色轮

```glsl
#define PI 3.1415
#define TWO_PI 6.28318530718

// Function from Iñigo Quiles
// https://www.shadertoy.com/view/MsS3Wc
vec3 hsb2rgb(in vec3 c) {
    vec3 rgb = clamp(abs(mod(c.x * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0) - 1.0, 0.0, 1.0);
    rgb = rgb * rgb * (3.0 - 2.0 * rgb);
    return c.z * mix(vec3(1.0), rgb, c.y);
}

float circle(float r, vec2 st) {
    return smoothstep(r, r + 0.008, length(st - vec2(0.5)));
}

void main() {
    vec2 st = gl_FragCoord.xy / iResolution.xy;
    vec3 color = vec3(0.0);

    // Use polar coordinates instead of cartesian
    vec2 toCenter = vec2(0.5) - st;
    float angle = atan(toCenter.y, toCenter.x);
    float radius = length(toCenter) * 2.0;

    color = hsb2rgb(vec3((angle + PI) / TWO_PI, radius, 1.0));
    color = mix(color, vec3(1.0), circle(0.3, st));
    gl_FragColor = vec4(color, 1.0);
}

```

简单来说，当我们每次判断坐标时，判断坐标到中心点的距离，如果超出，则显示为白色，否则显示为原来的颜色

```glsl
vec2 toCenter = vec2(0.5)-st;
```

这段代码的作用是计算从当前像素到屏幕中心的向量，用于将当前像素的坐标从笛卡尔坐标系转换为极坐标系。

具体来说，`vec2(0.5)`代表屏幕中心的坐标，而`st`代表当前像素在屏幕上的归一化坐标。因此，`vec2(0.5)-st`计算出的向量就是从当前像素到屏幕中心的向量。

这个向量的长度和方向可以用来计算当前像素在极坐标系中的位置，从而将笛卡尔坐标系中的像素坐标转换为极坐标系中的坐标。这个过程在接下来的代码中完成，可以看到`angle`和`radius`变量的计算都是基于这个向量的长度和方向进行的。

```glsl
float angle = atan(toCenter.y,toCenter.x);
```

这段代码的作用是计算当前像素在极坐标系中的角度，即极角。

`toCenter`变量是一个向量，表示从当前像素到屏幕中心的向量。在笛卡尔坐标系中，向量的 x 和 y 分量分别表示向量的水平和竖直方向的大小。在 GLSL 中，可以使用`atan(y, x)`函数来计算向量的极角，其中`y`和`x`分别表示向量的竖直和水平方向分量。

```glsl
float radius = length(toCenter)*2.0;
```

这段代码的作用是计算当前像素在极坐标系中的半径。

```glsl
color = hsb2rgb(vec3((angle + PI) / TWO_PI, radius, 1.0));
```

由于`atan()`函数的返回值范围是`[-π, π]`，因此需要进行额外的步骤来将极角的范围映射到`[0, 2π]`，以便后续的计算。具体来说，需要将极角加上`π`，然后将结果除以`2π`，得到的值就是从 0 到 1 之间的一个数，用于表示 HSB 颜色空间中的色相。

总的来说，H 表现在色轮的一个轮盘上，从最上方开始顺时针走动。

## 颜色的计算

### 相加

两个颜色相加，如果两个颜色向量都是 RGB 颜色向量，它们的每个分量都表示红、绿、蓝三个颜色通道的亮度值。在这种情况下，两个颜色向量相乘通常表示颜色通道之间的混合或调制。比如红色混合绿色变成了黄色，有很多在线网址可以预览颜色的混合，比如 https://gradients.app/zh/mix

### 相减

### 相乘
