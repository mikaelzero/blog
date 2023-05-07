---
title: ShaderToy(三)-颜色
urlname: ShaderToy-color
date: 2023/05/05
tags:
  - Shader
---

GLSL 中的 mix 函数是用于线性插值（Lerp）两个值的函数，它的原理是通过加权平均的方式将两个值混合在一起。

具体而言，对于两个值 x 和 y 以及一个插值因子 a，mix 函数的计算公式为：

```glsl
mix(x, y, a) = x * (1 - a) + y * a
```

其中，插值因子 a 的取值范围为[0, 1]，表示 x 和 y 在混合中的权重比例。当 a=0 时，mix 函数返回值为 x；当 a=1 时，返回值为 y；当 a 在[0, 1]之间时，返回值则为 x 和 y 的线性插值结果。

在颜色混合的应用中，通常将两个颜色值作为 x 和 y 输入到 mix 函数中，然后通过改变插值因子 a 的值来控制两个颜色之间的混合程度。例如，当 a=0.5 时，返回值为两个颜色值的平均值，即两个颜色的中间色。

所以可以理解 a 为 0 到 1，结果就是 x 到 y。

在图形学和计算机图形学中，HSB 是一种颜色表示方式，它由三个参数组成：色相（Hue）、饱和度（Saturation）和亮度（Brightness），其中 HSB 是缩写，也称为 HSV（Value 代替了 Brightness）。HSB 通常用于交互式应用程序和用户界面中，因为它提供了一种直观的方式来控制颜色。

以下是 HSB 参数的解释：

1. 色相（Hue）：它表示颜色的基本色调。色相以 0 到 360 度的角度表示，其中红色在 0 度，绿色在 120 度，蓝色在 240 度。

2. 饱和度（Saturation）：它表示颜色的纯度或强度。饱和度的值从 0%（灰色）到 100%（全彩色）变化。

3. 亮度（Brightness 或 Value）：它表示颜色的明暗程度，取值范围从 0%（黑色）到 100%（白色）。

在计算机图形学中，HSB 通常被转换为 RGB（红、绿、蓝）颜色空间，因为大多数图形硬件都使用 RGB 颜色空间来显示颜色。通常使用以下公式将 HSB 转换为 RGB：

- 计算饱和度和亮度的百分比值。
- 根据色相计算对应的红、绿、蓝分量的值。
- 将所有三个分量的值限制在 0 到 255 的范围内，以保证它们在 RGB 颜色空间中的有效值。

HSB 在计算机图形学中的应用非常广泛，它被用于创建渐变、调整色调、增强图像等，同时还可以帮助设计师和开发人员更好地理解和控制颜色。在着色器编程中，HSB 可以作为一种方便的方式来控制颜色，例如创建更自然的渐变效果、色彩过渡等。

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
