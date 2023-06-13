---
title: Shader(二)-函数曲线
urlname: shader-function-curve
date: 2023/05/03
tags:
  - Shader
  - OpenGL
---

先看看实现函数曲线的 glsl 代码：

```glsl
float plot(vec2 st, float pct){
  return  smoothstep( pct-0.02, pct, st.y) -
          smoothstep( pct, pct+0.02, st.y);
}

void main() {
    vec2 st = gl_FragCoord.xy/iResolution.xy;
    float y = smoothstep(0.1,0.9,st.x);

    vec3 color = vec3(y);

    float pct = plot(st,y);
    color = (1.0-pct)*color+pct*vec3(0.0,1.0,0.0);

    gl_FragColor = vec4(color,1.0);
}
```

其中`float y = smoothstep(0.1,0.9,st.x);`代表的是一个曲线函数，可以替换为其他任何的函数，比如`float y = pow(st.x, 5.0);`或者`float y = st.x;`

## 颜色混合

```glsl
color = (1.0-pct)*color+pct*vec3(0.0,1.0,0.0);
```

这段代码是将一个颜色值(color)和另一个颜色值(vec3(0.0,1.0,0.0))按照一定比例混合在一起，从而生成一个新的颜色值。

使用线性插值计算最终颜色。基本颜色是 color，而绿色线条的颜色是 vec3(0.0, 1.0, 0.0)。插值系数为 pct，当 pct 接近 1 时，最终颜色趋向于绿色。

这个式子中的"pct"是一个介于 0 和 1 之间的比例系数，用于控制两个颜色之间的混合比例。当"pct"等于 0 时，结果就是原来的颜色值(color)；当"pct"等于 1 时，结果就是另一个颜色值(vec3(0.0,1.0,0.0))，也就是纯绿色；当"pct"在 0 和 1 之间时，结果就是两个颜色按照"pct"的比例混合在一起的结果。

在 GLSL 中，我们可以通过这种方式将两个颜色进行加权混合。颜色值(color)和颜色向量(vec3)都是由三个浮点数(RGB)组成的向量。对于每个分量，我们将原颜色值乘以一个系数(1.0-pct)，再加上另一个颜色值乘以"pct"，这相当于对每个分量进行了线性插值。最后得到的结果是一个新的颜色向量，其中每个分量都是两个颜色按照一定比例混合在一起的结果。

## smoothstep 函数

`smoothstep` 是一个常用的 GLSL 函数，用于在两个给定的阈值之间进行平滑插值。它的原理基于 Hermite 插值，该插值可以通过计算一个三次多项式来实现。

![](images/opengl/shader_toy_smoothstep.png)

```glsl
float smoothstep(float edge0, float edge1, float x)
vec2 smoothstep(vec2 edge0, vec2 edge1, vec2 x)
vec3 smoothstep(vec3 edge0, vec3 edge1, vec3 x)
vec4 smoothstep(vec4 edge0, vec4 edge1, vec4 x)

vec2 smoothstep(float edge0, float edge1, vec2 x)
vec3 smoothstep(float edge0, float edge1, vec3 x)
vec4 smoothstep(float edge0, float edge1, vec4 x)
```

`smoothstep` 函数的计算过程可以通过 Hermite 插值的计算公式来推导得出。Hermite 插值是一种基于多项式的插值方法，它可以通过一些已知点的函数值和导数值，来推导出一个多项式函数，从而在两个已知点之间进行插值。

在 `smoothstep` 函数中，我们需要计算一个在给定阈值之间的平滑插值。具体来说，我们希望当输入值 `x` 小于某个阈值时，输出值为 0；当 `x` 大于另一个阈值时，输出值为 1；而在两个阈值之间时，输出值应该在 0 到 1 之间平滑过渡。

为了实现这个平滑过渡，我们可以使用 Hermite 插值的三次多项式函数，它可以通过以下公式计算得出：

```
f(x) = (2t³ - 3t² + 1) * f(a) + (t³ - 2t² + t) * m(a, b) + (-2t³ + 3t²) * f(b) + (t³ - t²) * m(b, a)
```

其中，`a` 和 `b` 是两个已知的阈值，`f(a)` 和 `f(b)` 分别是在 `a` 和 `b` 处的函数值，`m(a, b)` 和 `m(b, a)` 分别是在 `a` 和 `b` 处的导数值，`t` 是一个在 0 到 1 之间的插值参数，它可以表示 `x` 在 `a` 和 `b` 之间的位置。

为了让 `f(x)` 在 `a` 处为 0，我们需要将 `f(a)` 设为 0，将 `m(a, b)` 设为 0。同理，为了让 `f(x)` 在 `b` 处为 1，我们需要将 `f(b)` 设为 1，将 `m(b, a)` 设为 0。这样，我们就可以得到一个简化后的插值函数：

```
f(x) = 3t² - 2t³
```

其中，`t` 的取值范围为 0 到 1，它表示 `x` 在 `a` 和 `b` 之间的位置。

具体来说，对于一个输入值 `x`，`smoothstep` 函数的计算过程如下：

```glsl
genType t;  /* Or genDType t; */
t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0);
return (3.0 - 2.0 * t) * t * t;
```

1. 将 `x` 限制在两个给定的值之间。
2. 计算 `t = x * x * (3.0 - 2.0 * x)`，这是一个三次多项式，它在 `x=0` 时为 0，在 `x=1` 时为 1，在 `x=0.5` 时为最大值 0.5。
3. 返回 `t` 值。

在实际应用中，`smoothstep` 函数通常用于将一个值从一个阈值平滑地过渡到另一个阈值。例如，可以使用 `smoothstep` 函数将一个值从 0.0 到 1.0 的范围内经过平滑插值，转换为在 0.1 到 0.9 之间的范围内的值，如下所示：

```glsl
float x = 0.5; // 假设 x 的值在 0.0 到 1.0 之间
float y = smoothstep(0.1, 0.9, x); // 将 x 平滑插值到 0.1 到 0.9 之间
```

在上述示例中，`y` 的值将在 `x=0.1` 时为 0，在 `x=0.5` 时为 0.5，在 `x=0.9` 时为 1，这样就可以实现平滑的过渡效果。

而在 0.0 到 0.1 和 0.9 到 1.0 的区间，曲线都是为直线，因为都是固定的 0.0 或者 1.0。

## 绘制曲线

得到了曲线函数之后，如何把曲线部分的颜色值变为绿色？

假设现在绘制的函数是 `float y = st.x;`，那么该函数图像就是一个斜线

![](images/opengl/shadertoy_fx.png)

当 x 为 0.5 时，y 也是 0.5。由于绘制的线需要一个宽度，因为坐标已经归化为 0 到 1 了，所以我们暂且定为宽度为 0.04。

那么当 y 为 0.5 时，0.502 和 0.498 这两个 y 的颜色值也是为绿色，且当 x 为 0.5 时，假设现在 y 是从上到下来计算的话，当 y 为 0.502 到 0.498 时，颜色为绿色的渐变。

在 plot 函数中，它使用了两个 smoothstep 函数，一个用于绘制峰值的上侧，另一个用于绘制下侧。具体来说，下侧的 smoothstep 函数使用 pct-0.02 作为起始点，pct 作为终点，st.y 作为插值因子，生成一个在下侧缘处渐变的值，也就是 0.5 到 0.502 这个区间。上侧的 smoothstep 函数则使用 pct 作为起始点，pct+0.02 作为终点，同样使用 st.y 作为插值因子，生成一个在上侧缘处渐变的值，也就是 0.492 到 0.5 这个区间。然后，将这两个函数的结果相减，在 0.502 和 0.498 区间的一个颜色插值，而在这个区间外的则为 0 或者 1。

最终在计算颜色值时，`color = (1.0 - pct) * color + pct * vec3(0.0, 1.0, 0.0);`，通过这个插值再计算颜色值，就得到了一个渐变的绿色。效果如下:

![](images/opengl/shader_fx1.png)

如果将 0.02 的值放大一些，就更能看出渐变的效果:

![](images/opengl/shader_fx2.png)

## step 函数

如果曲线不想要渐变色，而是纯绿色，则可以使用 step 函数

step() 插值函数需要输入两个参数。第一个是极限或阈值，第二个是我们想要检测或通过的值。对任何小于阈值的值，返回 0.0，大于阈值，则返回 1.0。

```glsl
float step(float edge, float x)
vec2 step(vec2 edge, vec2 x)
vec3 step(vec3 edge, vec3 x)
vec4 step(vec4 edge, vec4 x)

vec2 step(float edge, vec2 x)
vec3 step(float edge, vec3 x)
vec4 step(float edge, vec4 x)
```

函数图像如下:

![](images/opengl/shader_step.png)

定义如下：

```glsl
float step(float edge, float x) {
    return (x < edge) ? 0.0 : 1.0;
}
```

其中，edge 是阈值，x 是输入值。如果 x 小于阈值 edge，函数返回 0.0，否则返回 1.0。

step 函数的原理非常简单，它使用了三目运算符（? :）来判断是否满足条件，如果条件成立则返回一个值，否则返回另一个值。在这里，如果 x 小于阈值 edge，则条件为 true，返回 0.0；否则条件为 false，返回 1.0。这样就产生了一个在阈值上产生阶跃输出的效果。

在图形学中，step 函数常用于实现阴影映射（shadow mapping）和距离场（distance field）等技术。例如，阴影映射中需要判断一个像素是否在阴影中，可以使用 step 函数将深度值与阈值进行比较，从而得到一个二值化的阴影图像。距离场中则可以使用 step 函数将距离值与等值线进行比较，从而生成一个平滑的距离场效果。

所以得到一个下面的函数：

```glsl
float plot2(vec2 st,float pct) {
    return  step(pct-0.2, st.y) * (1.0 - step(pct+0.2, st.y));
}
```

第一个 step 检查 st.y 是否大于等于 pct-0.2，如果是则返回 1.0，否则返回 0.0。第二个 step 检查 st.y 是否大于等于 pct+0.2，如果是则返回 0.0，否则返回 1.0。将这两个函数的结果相乘，就得到了一个在区间[pct-0.2, pct+0.2]内为 1.0，否则为 0.0 的函数值。

效果如下:

![](images/opengl/shader_step2.png)

## sin 曲线

正弦曲线是一种周期性的函数，其公式如下：

```
y = A * sin(ωx + φ) + k
```

其中，A 是振幅，ω 是角频率，φ 是相位差，k 是垂直方向的偏移量。

振幅 A 表示波峰和波谷的差值，角频率 ω 表示波形的周期，相位差 φ 表示波形在 x 轴上的位置偏移量，垂直方向的偏移量 k 表示整个波形曲线在 y 轴上的位置偏移量。

如果你想将一个 sin 函数映射到 0 到 1，那么前两个参数应该写成-1 和 1，即

```glsl
float y = smoothstep(-1.,1.,sin(st.x));
```

smoothstep 是将给定的范围和值映射到 0 到 1 的范围，而 sin 函数的域为 [-1,1]。当然如果修改了 sin 函数的振幅，前两个参数也需要修改。

如果加上一个时间来修改 k,那么这个曲线就会动起来

```glsl
float y = smoothstep(-1.,1.,sin(20.*st.x+iTime));
```
