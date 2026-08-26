一个简单的例子 <span class="bullet">🟡</span>
================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/3d-meshes/a-simple-example.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step050`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step050)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step050-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step050-vanilla)
````

让我们深入了解您在这里很可能要做的事情：渲染 **3D 形状**！

```{note}
我暂时回滚了有关动态制服的代码部分。我还将 `offset` 设置为 `vec2f(0.0)`；
```

切换到 3D 数据
--------------------

我们首先需要的是在我们的点位置上有一个第三列！

这是一个简单的形状，您可以将其保存在 `resources/pyramid.txt` 中：

```
[积分]
# 我们添加一个 Z 坐标
# x y z r g b

# 基础
-0.5 -0.5 -0.3    1.0 1.0 1.0
+0.5 -0.5 -0.3    1.0 1.0 1.0
+0.5 +0.5 -0.3    1.0 1.0 1.0
-0.5 +0.5 -0.3    1.0 1.0 1.0

# 金字塔尖
+0.0 +0.0 +0.5    0.5 0.5 0.5

[指数]
# 基础
 0  1  2
 0  2  3
# 双方
 0  1  4
 1  2  4
 2  3  4
 3  0  4
```

当然，我们需要调整 `loadGeometry` 函数来处理这个额外的维度。我添加了一个 `int dimensions` 参数，该参数应该是 2 或 3，具体取决于我们是在 2D 还是 3D 中：

```C++
bool loadGeometry(const fs::path& path, std::vector<float>& pointData, std::vector<uint16_t>& indexData, int dimensions) {
	// [...]

	// Get x, y, z, r, g, b
	for (int i = 0; i < dimensions + 3; ++i) {
	//                  ^^^^^^^^^^^^^^ This was a 5

	// [...]
}
```

我们现在可以加载几何图形，如下所示：

```C++
loadGeometry(RESOURCE_DIR "/pyramid.txt", pointData, indexData, 3 /* dimensions */);
```

由于这个新维度，我们需要更新顶点缓冲区步幅、位置属性格式和颜色属性偏移量：

```C++
// Position attribute
vertexAttribs[0].format = VertexFormat::Float32x3;
//                                              ^ This was a 2

// Color attribute
vertexAttribs[1].offset = 3 * sizeof(float);
//                        ^ This was a 2

// The buffer stride
vertexBufferLayout.arrayStride = 6 * sizeof(float);
//                               ^ This was a 5
```

````{note}
我们还需要增加顶点数组的最大步幅：

```C++
requiredLimits.limits.maxVertexBufferArrayStride = 6 * sizeof(float);
//                                                 ^ This was a 5
```
````

并且不要忘记更新 **着色器** 中的顶点输入结构！

```rust
结构体顶点输入{
@location(0) 位置：vec3f，
// ^ 这是一个 2
@location(1) 颜色：vec3f，
};
```

现在它有点起作用了，我们可以猜测这里有一个金字塔，但我还不会称它为 3D。将 `in.position.z` 添加到 `out.position.z` 到目前为止不会改变任何内容：

```{figure} /images/pyramid-base.png
:align: center
:class: with-shadow
金字塔……从上面看，没有透视。
```

```{note}
我特意为金字塔的尖端设置了不同的颜色，以便我们看得更清楚。当引入基本的**着色**时，这个问题会得到更好的解决。
```

基本变换
---------------

*这是对三角学的简单介绍。如果您熟悉这个概念，您可以跳过。*

从上面看，这座金字塔看起来很无聊，就像一个正方形。我们可以**旋转**这个吗？改变视角的一个非常基本的方法是交换轴：

```rust
var 位置 = vec3f(
在.位置.x,
in.position.z, // 交换 Y 轴和 Z 轴
位置 y,
);
out.position = vec4f(position.x,position.y * 比率, 0.0, 1.0);
```

```{figure} /images/pyramid-side.png
:align: center
:class: with-shadow
从侧面看到的金字塔（仍然没有透视）。
```

中间的轮换又如何呢？这个想法是**混合轴**，在 y 坐标中添加一点 z，在 z 坐标中添加一点 y。

```rust
var 位置 = vec3f(
在.位置.x,
in.position.y + 0.5 * in.position.z, // 在 Y 中添加一点 Z...
in.position.z + 0.5 * in.position.y, // ...以及 Z 中的一点 Y。
);
out.position = vec4f(position.x,position.y * 比率, 0.0, 1.0);
```

```{figure} /images/pyramid-tilted.png
:align: center
:class: with-shadow
从倾斜的视角看金字塔。
```

当然，在某些时候，我们必须从 Y 中删除一些 `in.position.y` ，以便在四分之一圈后我们达到 `Y = 0.0 * in.position.y + 1.0 * in.position.z` ，如上例所示。所以更一般地说，我们的变换是这样写的，其中 `alpha` 和 `beta` 取决于旋转角度：

```rust
让角度= uMyUniforms.time； // 你可以乘以它来旋转得更快
让 alpha: f32 = /* ??? */;
让 beta: f32 = /* ??? */;
var 位置 = vec3f(
在.位置.x,
α * in.position.y + beta * in.position.z,
α * in.position.z - beta * in.position.y，
);
out.position = vec4f(position.x,position.y * 比率, 0.0, 1.0);
```

```{note}
如果您密切关注上面的代码片段，您应该注意到第二个 `beta` 之前有一个**减号** `-`。它在我们的金字塔上不可见，因为它是对称的，但交换轴也会翻转对象。为了**平衡**这一点，我们可以更改其中一个维度的符号。因此四分之一圈后的 Z 坐标必须是 `-in.position.y` 而不是 `in.position.y`。
```

事实证明，这些权重`alpha`和`beta`用关于角度的基本运算来说并不容易表达。于是数学家们给它们起了一个专用的名字：**余弦**和**正弦**！好消息是这些是 WGSL 中的**内置操作**：

```rust
让角度= uMyUniforms.time； // 你可以乘以它来旋转得更快
让 alpha = cos(角度);
让 beta = sin(角度);
var 位置 = vec3f(
在.位置.x,
α * in.position.y + beta * in.position.z,
α * in.position.z - beta * in.position.y，
);
out.position = vec4f(position.x,position.y * 比率, 0.0, 1.0);
```

<figure class="align-center">
<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
<source src="../../_static/pyramid-ryz.mp4" type="video/mp4">
</video>
<figcaption>
<p><span class="caption-text">YZ 平面中的旋转</span></p>
</figcaption>
</figure>

```{image} /images/trigo-light.svg
:align: center
:class: only-light
```

```{image} /images/trigo-dark.svg
:align: center
:class: only-dark
```

恭喜，您已经了解了有关计算机图形学**三角学**的大部分知识！

```{hint}
**如果你不记得** `alpha` 和 `beta` 中哪一个是 $cos$，哪一个是 $sin$（别担心！每个人都会遇到这种情况），**仅举一个非常简单的旋转示例**：`angle = 0`。在这种情况下，我们需要 `alpha = 1` 和 `beta = 0`。如果您查看 $sin$ 和 $cos$ 函数的绘图，您很快就会发现 $cos(0) = 1$ 和 $sin(0) = 0$
```

```{important}
三角函数的自变量是**角度**，但要注意它必须用**弧度**表示。一整圈总共有 $2\pi$ 弧度，这导致了以下基本交叉乘法规则：

$$
\frac{r \text{ 弧度}}{d \text{ 度}} = \frac{2\pi \text{ 弧度}}{360 \text{ 度}}
$$

因此，要将角度 $d$（以度为单位）转换为等效的 $r$（以弧度为单位），我们只需执行以下操作：

$$
r = d \times \frac{\pi}{180}
$$
```

结论
----------

我们有一个开始。通过这种旋转，它开始看起来像 3D，但仍然有一些重要的问题需要关注：

- **深度战斗** 如下图所示，三角形未按正确顺序重叠。
- **变换**我们有基础知识，但有点手动，而且仍然**没有视角**！
- **着色** 将金字塔尖端设置为较深颜色的技巧对于开始来说很好，但我们可以做得更好。

这些要点按顺序是接下来 4 章的主题（变换分为 2 章）。

```{figure} /images/pyramid-zissue.png
:align: center
:class: with-shadow
深度有问题。
```

````{tab} With webgpu.hpp
*结果代码：* [`step050`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step050)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step050-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step050-vanilla)
````
