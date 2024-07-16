---
title: Shader 编程学习之路 -- 临摹
date: 2022-10-10 10:50:00
tags: [Shader, Programming]
categories:
  - 计算机图形学
---

本文主要是积累一些自己编写的 Shader 习题示例。主要采用 vscode + shadertoy 扩展工具来实现，使用 GLSL 语言编写，不定期更新。

<!-- more -->
## 参考资料

本文的参考资料为：  
1.[《The Book of Shaders》](https://thebookofshaders.com/)
2.[《Nathan Vaughn 的 shader 教程》](https://inspirnathan.com/topics/shaders/)

## 程序示例

### 1.Piet Mondria - Tableau (1921)

![](/image/2022/shader-learn/PietMondria.png)

这个图案是该教程[形状](https://thebookofshaders.com/07/?lan=ch)这一章节展示的示例，我将它实现以用作绘制矩形的练习，代码如下：

``` GLSL
vec3 red = vec3(0.71, 0.14, 0.15);
vec3 yellow = vec3(0.99, 0.77, 0.2);
vec3 blue = vec3(0.0, 0.36, 0.6);
vec3 black = vec3(0.12, 0.14, 0.15);

/*
 * 绘制矩形
 *
 * st 当前 frag 归一化坐标
 * l 矩形长度
 * h 矩形高度
 * center 矩形中心点
 */
float rect(vec2 st, float l, float h, vec2 center) {
    float tl = (1.0 - l) * 0.5;
    float th = (1.0 - h) * 0.5;
    vec2 c = vec2(center.x - 0.5, 1.0 - center.y - 0.5);

    vec2 lb = step(vec2(tl + c.x, th + c.y), st);
    vec2 rt = step(st, vec2(1.0 - tl + c.x, 1.0 - th + c.y));

    return lb.x * lb.y * rt.x* rt.y;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    vec3 back = vec3(0.96, 0.93, 0.87);

    // 先混合背景色和主色块
    back = mix(back, red, rect(st, 0.2, 0.4, vec2(0.1, 0.2)));
    back = mix(back, yellow, rect(st, 0.05, 0.4, vec2(0.975, 0.2)));
    back = mix(back, blue, rect(st, 0.3, 0.1, vec2(0.85, 0.95)));

    // 接下来是几个覆盖的黑色线条 横向
    back = mix(back, black, rect(st, 1.0, 0.035, vec2(0.5, 0.2)));
    back = mix(back, black, rect(st, 1.0, 0.035, vec2(0.5, 0.4)));
    back = mix(back, black, rect(st, 0.8, 0.03, vec2(0.6, 0.9)));

    // 纵向
    back = mix(back, black, rect(st, 0.025, 0.4, vec2(0.06, 0.2)));
    back = mix(back, black, rect(st, 0.03, 1.0, vec2(0.2, 0.5)));
    back = mix(back, black, rect(st, 0.03, 1.0, vec2(0.7, 0.5)));
    back = mix(back, black, rect(st, 0.03, 1.0, vec2(0.95, 0.5)));

    fragColor = vec4(back, 1.0);
}
```
非常的简单，主要是矩形绘制和 mix 函数的应用。

### 2.画圆

效果如图所示：
![](/image/2022/shader-learn/Circle.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;

    float len = distance(st, vec2(0.5));
    float pct = 1.0 - smoothstep(0.45, 0.5, len);
    vec3 back = vec3(pct);

    fragColor = vec4(back, 1.0);
}
```
直接通过 distance 函数配合 smoothstep 绘制一个边界模糊的圆。

### 3.画正方形（通过shaderToy这个平台）

![](/image/2022/shader-learn/react.jpg)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    vec3 wihit = vec3(1.0);

    float pct = step(max(abs(st.x), abs(st.y)), 0.25);
    back = pct * wihit + (1.0 - pct) * back;

    fragColor = vec4(back, 1.0);
}
```
绘制矩形主要运用了 max(abs(x), abs(y)) =  r / 2 这个关系，其中 r 为矩形边长。

### 4.边缘模糊的圆环

![](/image/2022/shader-learn/CircleRing.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    vec3 white = vec3(1.0);

    float len = length(st);

    //if (len > 0.2 && len < 0.3)
    float pct = 1.0 - smoothstep(0.18, 0.2, len) * (1.0 - smoothstep(0.28, 0.3, len));
    back = pct * back + (1.0 - pct) * white; 

    fragColor = vec4(back, 1.0);
}
```
使用 smoothstep 来代替 if 语句。

### 5.奥运五环

![](/image/2022/shader-learn/Olympic.png)

``` GLSL
// 圆环，r 为半径，offset 为圆心偏移量
float circleRing(vec2 st, float r, vec2 offset)
{
    vec2 pos = st - offset;
    float len = length(pos);
    float sdf = smoothstep(r - 0.02, r, len) * (1.0 - smoothstep(r, r + 0.02, len));
    return sdf;
}

// 矩形，l 为长，h为高，中心点 (0, 0)
float rect(vec2 st, float l, float h) 
{
    float horizontal = 1.0 - smoothstep(l - 0.01, l, abs(st.x));
    float vertical = 1.0 - smoothstep(h - 0.01, h, abs(st.y));
    float sdf = horizontal * vertical;
    return sdf;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 black = vec3(0.0);
    vec3 blue = vec3(0.1, 0.1, 0.8);
    vec3 red = vec3(0.8, 0.1, 0.1);
    vec3 yellow = vec3(0.9, 0.9, 0.1);
    vec3 green = vec3(0.1, 0.8, 0.1);

    vec3 back = vec3(0.0);

    // 奥运五环
    back = mix(back, vec3(1.0), rect(st, 0.4, 0.3));
    back = mix(back, blue, circleRing(st, 0.1, vec2(-0.15, 0.08)));
    back = mix(back, black, circleRing(st, 0.1, vec2(0.0, 0.08)));
    back = mix(back, red, circleRing(st, 0.1, vec2(0.15, 0.08)));
    back = mix(back, yellow, circleRing(st, 0.1, vec2(-0.08, -0.04)));
    back = mix(back, green, circleRing(st, 0.1, vec2(0.08, -0.04)));

    fragColor = vec4(back, 1.0);
}
```

这个图案主要运用 mix 混合函数来实现，由于是纯 2D 表示，所以并没有考虑深度。也可以先单纯采用距离场的方式计算出图形的位置关系，然后再混合，这样也可以得出相同的图案：

``` GLSL
// 圆环，r 为外圈半径，offset 为圆心偏移量
float circleRing(vec2 st, float r, vec2 offset)
{
    vec2 pos = st - offset;
    float d = length(pos);
    float d1 = d - r;
    float d2 = d - (r + 0.01);
    float sdf = max(-d1, d2);

    return sdf;
}

// 矩形
float rect(vec2 st, float l, float h)
{
    float d1 = abs(st.x) - l;
    float d2 = abs(st.y) - h;
    float sdf = max(d1, d2);

    return sdf;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 black = vec3(0.0);
    vec3 blue = vec3(0.1, 0.1, 0.8);
    vec3 red = vec3(0.8, 0.1, 0.1);
    vec3 yellow = vec3(0.9, 0.9, 0.1);
    vec3 green = vec3(0.1, 0.8, 0.1);

    vec3 back = vec3(0.0);

    float sdf1 = rect(st, 0.4, 0.3);
    float sdf2 = circleRing(st, 0.1, vec2(-0.15, 0.08));
    float sdf3 = circleRing(st, 0.1, vec2(0.0, 0.08));
    float sdf4 = circleRing(st, 0.1, vec2(0.15, 0.08));
    float sdf5 = circleRing(st, 0.1, vec2(-0.08, -0.04));
    float sdf6 = circleRing(st, 0.1, vec2(0.08, -0.04));

    // 奥运五环
    back = mix(back, vec3(1.0), 1.0 - smoothstep(0.0, 0.008, sdf1));
    back = mix(back, blue, 1.0 - smoothstep(0.0, 0.008, sdf2));
    back = mix(back, black, 1.0 - smoothstep(0.0, 0.008, sdf3));
    back = mix(back, red, 1.0 - smoothstep(0.0, 0.008, sdf4));
    back = mix(back, yellow, 1.0 - smoothstep(0.0, 0.008, sdf5));
    back = mix(back, green, 1.0 - smoothstep(0.0, 0.008, sdf6));

    fragColor = vec4(back, 1.0);
}
```
这样更加简洁一些，更少地调用 smoothstep 这些内置函数。

### 6.图案的关系处理

#### 6.1求交集

两个图形求交集，效果如下：
![](/image/2022/shader-learn/Intersection.png)

``` GLSL
// 圆环，r 为外圈半径，offset 为圆心偏移量
float circleRing(vec2 st, float r, vec2 offset)
{
    vec2 pos = st - offset;
    float d = length(pos);
    float d1 = d - r;
    float d2 = d - (r + 0.04);
    float sdf = max(-d1, d2);

    return sdf;
}

// 矩形
float rect(vec2 st, float l, float h)
{
    float d1 = abs(st.x) - l;
    float d2 = abs(st.y) - h;
    float sdf = max(d1, d2);

    return sdf;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = max(d1, d2);

    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```
我们分别用三种颜色表示图形，较亮的那一部分为两图形的交集。求交集使用的函数是 max。

#### 6.2求并集

两个图形求交集，效果如下：
![](/image/2022/shader-learn/Union.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = min(d1, d2);

    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```
只需要调用 min 函数即可实现。

#### 6.3求补集

两个图形求交集，效果如下：
![](/image/2022/shader-learn/Comp.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = -d1;

    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```
直接对圆环距离场取负即可。用这个关系我们可以取矩形和环形补集的交集，可以得到如下图案：

![](/image/2022/shader-learn/Sub.png)

``` GLSL
float sdf = max(-d1, d2);
```

#### 6.4XOR 异或关系

根据异或关系 “同 0 异 1” 的原则，我们可以推断出两个图形异或的结果为其相交部分被去除：

![](/image/2022/shader-learn/Xor.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = min(max(-d1, d2), max(d1, -d2));

    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```

分别取圆环和矩形各自的补集相交，然后将结果合并即可。用逻辑关系表示为 A xor B = (A and not B) or (not A and B)，当然不止有这一种实现方法，读者自行探索。

#### 6.5平滑接缝

有时候我们可以通过特殊的函数将两个图形的交接处变得更平滑：

![](/image/2022/shader-learn/Smoothxor.png)

``` GLSL
// a 和 b 分别为两个图形的有向距离场，k 为平滑因子
float smin(float a, float b, float k) {
  float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0);
  return mix(b, a, h) - k * h * (1.0 - h);
}

// smooth max
float smax(float a, float b, float k) {
  return -smin(-a, -b, k);
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = smin(smax(-d1, d2, 0.1), smax(d1, -d2, 0.1), 0.1);

    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```

将原有的 max 和 min 替换为 smax 和 smin 即可实现效果。这两个函数直接取自 Nathan Vaughn 的博客 中实现的算法，数学原理暂不理解，还需要慢慢研究。

#### 6.6轴对称

![](/image/2022/shader-learn/SymX.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);

    vec2 symX = vec2(-st.x, st.y);

    float d1 = circleRing(st, 0.2, vec2(0.2));
    float d2 = circleRing(symX, 0.2, vec2(0.2));
    float d3 = min(d1, d2);
    float d4 = rect(st, 0.2, 0.2);
    float sdf = min(max(-d3, d4), max(d3, -d4));
 
    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```

轴对称操作的非常简单，直接将输入坐标的 x 值取负，然后就可以得到关于 x 轴对称的图形，然后使用 min 取并集即可获得结果。还有另一种方法，直接对输入的坐标 st 的 x 值取绝对值 abs(st.x)，这样可以直接得到以下的效果：

![](/image/2022/shader-learn/Sym.png)

``` GLSL
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);

    float d1 = circleRing(vec2(abs(st.x), st.y), 0.2, vec2(0.2));
    float d2 = rect(st, 0.2, 0.2);
    float sdf = min(max(-d1, d2), max(d1, -d2));
 
    back = mix(back, vec3(0.1, 0.2, 0.4), 1.0 - smoothstep(0.0, 0.005, d1));
    back = mix(back, vec3(0.1, 0.1, 0.2), 1.0 - smoothstep(0.0, 0.005, d2));
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - smoothstep(0.0, 0.005, sdf));
    fragColor = vec4(back, 1.0);
}
```
不过这样做会切除该图形在对称象限的那部分。

### 7.画心形

最简单的心形我们采用 (x^2 + y^2 - 1)^3 - x^2 * y^3 = 0 这个公式来构造距离场：

![](/image/2022/shader-learn/Heart.png)

``` GLSL
// 心形 (x^2 + y^2 - 1)^3 - x^2 * y^3 = 0
float heart(vec2 st, float size, vec2 offset)
{
    vec2 pos = st - offset;

    // 直接计算距离场
    float xx = dot(pos.x, pos.x);
    float yy = dot(pos.y, pos.y);
    float yyy = pos.y * yy;
    float group = xx + yy - size;
    float sdf = dot(group, group) * group - xx * yyy;

    return sdf;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);
    float sdf = heart(st, 0.05, vec2(0.0));
 
    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - step(0.0, sdf));
    fragColor = vec4(back, 1.0);
}
```
比较奇怪的是这个形状无法使用 smoothstep 来消除锯齿，可能是不规则图形的缘故，可能需要更高深的插值技巧，有待研究。

### 8.五角星

![](/image/2022/shader-learn/Star5.png)

``` GLSL
// 绘制五角星，同样出自 Inigo Quilez 的网站
float sdStar5(vec2 st, float r, float rf)
{
    const vec2 k1 = vec2(0.809016994375, -0.587785252292);
    const vec2 k2 = vec2(-k1.x, k1.y);
    st.x = abs(st.x);
    st -= 2.0 * max(dot(k1, st), 0.0) * k1;
    st -= 2.0 * max(dot(k2, st), 0.0) * k2;
    st.x = abs(st.x);
    st.y -= r;
    vec2 ba = rf * vec2(-k1.y, k1.x) - vec2(0, 1);
    float h = clamp(dot(st, ba) / dot(ba, ba), 0.0, r);
    return length(st - ba * h) * sign(st.y * ba.x - st.x * ba.y);
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x *= (iResolution.x / iResolution.y);
    st = 2.0 * st - 1.0;

    float sdf = sdStar5(st, 0.5, 0.4);
    vec3 back = vec3(1.0 - smoothstep(0.0, 0.02, sdf));

    fragColor = vec4(back, 1.0);
}
```

五角星的实现同样出自 [IQ](https://iquilezles.org/articles/) 的博客，原理暂时没搞明白，不过有了这个实现，可以很轻松地绘制一面五星红旗：

![](/image/2022/shader-learn/Flag.png)

``` GLSL
// 绘制五角星，同样出自 Inigo Quilez 的网站
float sdStar5(vec2 st, float r, float rf, vec2 offset)
{
    const vec2 k1 = vec2(0.809016994375, -0.587785252292);
    const vec2 k2 = vec2(-k1.x, k1.y);

    st -= offset;

    st.x = abs(st.x);
    st -= 2.0 * max(dot(k1, st), 0.0) * k1;
    st -= 2.0 * max(dot(k2, st), 0.0) * k2;
    st.x = abs(st.x);
    st.y -= r;

    vec2 ba = rf * vec2(-k1.y, k1.x) - vec2(0, 1);
    float h = clamp(dot(st, ba) / dot(ba, ba), 0.0, r);
    return length(st - ba * h) * sign(st.y * ba.x - st.x * ba.y);
}

// 绘制矩形
float sdBox(vec2 st)
{
    float sdf = max(abs(st.x) - 1.0, abs(st.y) - 0.7);
    return sdf;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x *= (iResolution.x / iResolution.y);
    st = 2.0 * st - 1.0;

    float star1 = sdStar5(st, 0.2, 0.4, vec2(-0.7, 0.2));
    float star2 = sdStar5(st, 0.06, 0.4, vec2(-0.35, 0.5));
    float star3 = sdStar5(st, 0.06, 0.4, vec2(-0.3, 0.3));
    float star4 = sdStar5(st, 0.06, 0.4, vec2(-0.3, 0.1));
    float star5 = sdStar5(st, 0.06, 0.4, vec2(-0.35, -0.1));
    float star = min(star1, star2);
    star = min(star, star3);
    star = min(star, star4);
    star = min(star, star5);

    float flag = sdBox(st);

    vec3 back = vec3(1.0 - smoothstep(0.0, 0.02, flag), 0.0, 0.0);
    back = mix(back, vec3(1.0, 1.0, 0.0), 1.0 - smoothstep(0.0, 0.01, star));

    fragColor = vec4(back, 1.0);
}
```

### 9.线段

![](/image/2022/shader-learn/Line.png)

``` GLSL
// 线段距离场，w 为线段宽度
float line(vec2 p, vec2 a, vec2 b, float w)
{
    vec2 ap = p - a;
    vec2 ab = b - a;

    // |t| / |ab| 结果必然小于 1.0
    float t = clamp(dot(ap, ab) / dot(ab, ab), 0.0, 1.0);
    vec2 at = t * ab;
    float sdf = length(ap - at);

    return sdf - w;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);    
    float sdf = line(st, vec2(-0.2), vec2(0.2), 0.005);

    back = mix(back, vec3(0.2, 1.0, 1.0), 1.0 - step(0.0, sdf));
    fragColor = vec4(back, 1.0);
}
```
利用向量的方式来求任意一点到线段的距离。

### 10.贝赛尔曲线

![](/image/2022/shader-learn/Bezier.png)

``` GLSL
float dot2(vec2 v) {
    return dot(v, v);
}

// 贝塞尔曲线 f(u) = p0 * (1 - u) ^ 2 + 2 * p1 * u * (1 - u) + p2 * u ^ 2
// A、p1 和 p2 分别三个控制点
float bezier(vec2 pos, vec2 p0, vec2 p1, vec2 p2)
{
    vec2 a = p1 - p0;
    vec2 b = p0 - 2.0 * p1 + p2;
    vec2 c = a * 2.0;
    vec2 d = p0 - pos;
    float kk = 1.0 / dot(b, b);
    float kx = kk * dot(a, b);
    float ky = kk * (2.0 * dot(a,a) + dot(d, b)) / 3.0;
    float kz = kk * dot(d, a);
    float res = 0.0;
    float p = ky - kx * kx;
    float p3 = p * p * p;
    float q = kx * (2.0 * kx * kx - 3.0 * ky) + kz;
    float h = q * q + 4.0 * p3;
    if(h >= 0.0) { 
        h = sqrt(h);
        vec2 x = (vec2(h, -h) - q) / 2.0;
        vec2 uv = sign(x) * pow(abs(x), vec2(1.0 / 3.0));
        float t = clamp(uv.x + uv.y - kx, 0.0, 1.0);
        res = dot2(d + (c + b * t) * t);
    } else {
        float z = sqrt(-p);
        float v = acos(q / (p * z * 2.0)) / 3.0;
        float m = cos(v);
        float n = sin(v) * 1.732050808;
        vec3  t = clamp(vec3(m + m, -n - m, n - m) * z - kx, 0.0, 1.0);
        res = min(dot2(d + (c + b * t.x) * t.x),
                  dot2(d + (c + b * t.y) * t.y));
        // the third root cannot be the closest
        // res = min(res,dot2(d+(c+b*t.z)*t.z));
    }
    return sqrt(res) - 0.01;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x = st.x * (iResolution.x / iResolution.y);
    st -= 0.5;

    vec3 back = vec3(0.0);    
    float sdf = bezier(st, vec2(-0.2), vec2(0.0, 0.4), vec2(0.2, -0.2));

    back = mix(back, vec3(0.02, 1.0, 1.0), 1.0 - step(0.0, sdf));
    fragColor = vec4(back, 1.0);
}
```

贝塞尔曲线的 shader 实现直接引用自 [Inigo Quilez's](https://iquilezles.org/) 大神的实现，其中的数学原理还需要慢慢学习研究。

### 11.棋盘纹理

![](/image/2022/shader-learn/Check.png)

``` GLSL
vec2 tile(vec2 st, vec2 zoom)
{
    st *= zoom;
    return fract(st);
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x *= (iResolution.x / iResolution.y);
    st = 4.0 * st;

    st = tile(st, vec2(1.0));
    vec3 back = vec3(0.0);

    vec2 oe = step(vec2(0.5), mod(st, vec2(2.0)));
    back = vec3(min((1.0 - min(oe.x, oe.y)), max(oe.y, oe.x)));

    fragColor = vec4(back, 1.0);
}
```
使用 y = mod(x, 2.0) 来判断奇偶性。

### 12.随机噪声

![](/image/2022/shader-learn/Noise.png)

``` GLSL
float random(vec2 st)
{
    float r = fract(
        sin(
            dot(st, vec2(10.0, 10.3))
        ) * 300000.0
    );
    return r;
}

void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
    vec2 st = fragCoord / iResolution.xy;
    st.x *= (iResolution.x / iResolution.y);
    st = 2.0 * st;

    float rd = random(st);
    vec3 back = vec3(rd);

    fragColor = vec4(back, 1.0);
}
```
采用 y = fract(sin(x) * n) 来生成伪随机数。
