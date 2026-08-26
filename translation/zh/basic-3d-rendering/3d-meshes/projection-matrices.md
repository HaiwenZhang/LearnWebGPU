投影矩阵 <span class="bullet">🟡</span>
===================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/3d-meshes/projection-matrices.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step055`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step055)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step055-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step055-vanilla)
````

现在我们已经熟悉了矩阵的概念，我们看到了如何使用它们来表示投影，尽管**透视投影**既不是线性变换也不是仿射变换，因此从数学上来说，它并不完全是矩阵应该表示的。

我们还在第二部分中介绍了使用 GLM 库从 C++ 代码管理变换和投影矩阵的**典型方法**。

正投影
-----------------------

到目前为止，我们在将 3D 场景投影到 2D 屏幕上做了哪些工作？输出位置 `out.position` 的 $x$ 和 $y$ 坐标映射到窗口坐标，而 $z$ 坐标不影响我们几何体的像素位置，因此这是 **沿 Z 轴的正交投影**：

```rust
out.position = vec4f(position.x,position.y * 比例,position.z * 0.5 + 0.5, 1.0);
```

请记住，我们必须将 $z$ 坐标重新映射到范围 $(0,1)$，因为该范围之外的任何内容都会被 **剪掉**，就像沿 $x$ 和 $y$ 轴超出范围 $(-1,1)$ 的任何内容都会落在窗口之外。

```{image} /images/clip-space-light.svg
:align: center
:class: only-light
```

```{image} /images/clip-space-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>标准化裁剪体积.</em></span>
</p>

我们称之为**剪切体积**。只有在顶点着色器之后位于该体积内的几何体才能产生**可见片段**。

```{caution}
输出 $z$ 坐标的预期范围因图形 API 而异。所有现代 API（DirectX 12、Metal、Vulkan、WebGPU）都使用 $(0,1)$，但 OpenGL 和 WebGL 期望 $(-1,1)$。因此，投影矩阵的定义略有不同。
```

当然，这个正交投影可以很容易地表示为矩阵：

```rust
让 P = 转置(mat4x4f(
	1.0,  0.0,  0.0, 0.0,
0.0, 比率, 0.0, 0.0,
	0.0,  0.0,  0.5, 0.5,
	0.0,  0.0,  0.0, 1.0,
));

让同类位置 = vec4f(位置, 1.0);
输出位置 = P * 同质位置；
```

请注意，上面矩阵中的系数 $0.5$ 来自我们想要从 $(-1,1)$ 范围重新映射 Z 坐标的事实。一般来说，如果模型的 $z$ 坐标在 $(n,f)$ 范围内，则得到 $z_{\text{out}} = \frac{z - n}{f - n} = \frac{z}{f - n} - \frac{n}{f - n}$，因此系数变为 $p_{zz} = \frac{1}{f - n}$ 和 $p_{zw} = \frac{- n}{f - n}$。

我们还可以通过划分场景的 XY 尺寸来更改视图范围，以使其更大的部分适合视锥体。

```rust
// 正交投影矩阵的更通用的表达式
让附近=-1.0；
让远= 1.0；
让比例= 1.0；
让 P = 转置(mat4x4f(
1.0/刻度、0.0、0.0、0.0、
0.0, 比例/比例, 0.0, 0.0,
0.0, 0.0, 1.0 /（远-近），-近/（远-近），
	    0.0,          0.0,           0.0,                  1.0,
));
```

透视投影
----------------------

### 焦点

透视投影（或多或少）是**实际相机**或人眼中发生的投影。它不是将场景投影到平面上，而是**投影到单个点**，称为**焦点**。

屏幕的像素对应于不同的**传入方向**，几何元素从这些方向投影。

```{image} /images/perspective-light.svg
:align: center
:class: only-light
```

```{image} /images/perspective-dark.svg
:align: center
:class: only-dark
```

如果我们想要将透视**视锥体**（即可见方向集）映射到上述标准化剪辑空间，我们**需要将** XY 位置除以 $z$ 坐标。让我们举个例子：

```{image} /images/perspective2-light.svg
:align: center
:class: only-light
```

```{image} /images/perspective2-dark.svg
:align: center
:class: only-dark
```

点 $A$ 和 $C$ 沿相同方向投影，因此它们应该具有相同的 $y_\text{out}$ 坐标。同时，点$A$和$B$具有相同的输入$y$坐标。

问题是它们有**不同的深度**，而且我们知道**远处的物体看起来更小**。这是通过除以深度来建模的：

$$
y_\text{out} = \frac{y}{z}
$$

由于 $A$ 和 $B$ 具有不同的 $z$，因此它们最终位于不同的 $y_\text{out}$ 坐标，这意味着我们看到它们的方向略有不同（即最终图像中的不同像素）。

我们可以尝试一下：

```rust
// [...] 应用模型和视图变换，但不应用正交投影

// 我们移动视点，使所有 Z 坐标 > 0
//（这与正交投影没有区别
// 但现在确实如此。）
让 focusPoint = vec3f(0.0, 0.0, -2.0);
位置=位置-焦点；

// 我们除以 Z 坐标
位置.x /= 位置.z;
位置.y /= 位置.z;

// 应用正交矩阵来重新映射 Z 并处理比率
// 近和远必须为正数
让近= 0.0；
让远= 100.0；
让 P = /* ... */;
out.position = P * vec4f(位置, 1.0);
```

<figure class="align-center">
<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
<source src="../../_static/perspective.mp4" type="video/mp4">
</video>
<figcaption>
<p><span class="caption-text">我们的第一视角.</span></p>
</figcaption>
</figure>

塔达姆，它有效！

### 焦距

除以原始 $z$ 坐标有点随意，上面的公式有点可疑，因为它的术语**不可通约**：$y_\text{out}$ 是**长度**（例如，以米或英尺为单位），但 $y / z$ 是**比率**（无单位）。事实上我们可以缩放除法因子：

```rust
位置.x /= 0.5 * 位置.z;
位置.y /= 0.5 * 位置.z;
```

这对应于在公式中引入**焦距** $l$：

$$
y_\text{out} = l\frac{y}{z}
$$

```{note}
焦距可以看作是输出窗口对应的**焦点与虚拟传感器之间的距离**（可以使用泰勒斯定理进行验证）。
```

```rust
让焦距= 2.0；
位置.x /= 位置.z / 焦距；
位置.y /= 位置.z / 焦距；
```

```{image} /images/perspective3-light.svg
:align: center
:class: only-light
```

```{image} /images/perspective3-dark.svg
:align: center
:class: only-dark
```

焦距是一个用户参数，对应于我们虚拟相机的**变焦级别**。

```{figure} /images/pexels-alexandru-g-stavrica-2204008.jpg
:align: center
:class: with-shadow
该镜头的焦距范围为 18mm 至 55mm，具体取决于变焦环的转动方式。
```

```{note}
投影仅取决于**传感器尺寸**与实际**焦距**之间的**比率**（很容易看出，如果我们将传感器尺寸乘以 2 并将其移动到距焦点两倍的距离，我们会得到相同的图像）。

因此，商业焦距通常是针对给定的**标准传感器尺寸**来表示的，即沿对角线的 35 毫米（称为“全画幅”的尺寸）。

由于我们的剪辑空间的宽度为 2 个单位（从 -1 到 1），因此如果我们想要商业 50 毫米镜头的视觉外观，我们需要将 `focalLength` 设置为 `2 * 50 / 35 = 2.86`。实际上，使用 `640/480` 比率，剪辑空间的对角线为 `2.5`，因此 `focalLength` 最终为 `3.57`。
```

### 透视矩阵

不幸的是，透视投影**不是线性变换**，因为除以 $z$。然而，由于这种划分非常常用，所以它被嵌入**固定管道**！

我们怎么还没注意到呢？因为为了获得更大的灵活性，它不会除以 `out.position.z` ，而是除以 `out.position.w` 。

我们希望 $w$ 为 `position.z / focalLength`，因此在投影矩阵 `P` 中，我们将系数 $p_{wz}$ 设置为 `1.0 / focalLength`，并将最后一个对角线系数 $p_{ww}$ 设置为 $0$ 而不是 $1$。

```rust
让焦距= 2.0；
// 我们不再在这里划分！
//位置.x /= 位置.z / focusLength;
//位置.y /= 位置.z / focusLength;

让 P = /* ... */;
让同类位置 = vec4f(位置, 1.0);
输出位置 = P * 同质位置；

// 我们改为更改 w：
out.position.w = 位置.z / focusLength;
```

```{important}
$z$ 坐标本身也除以 $w$。
```

```{figure} /images/divide-w.png
:align: center
:class: with-shadow
投影是相同的，但是由于$z$坐标也除以$w$，所以深度信息被搞乱了。
```

在解决这个问题之前，我们可以注意到，由于硬编码除法，我们的透视投影**可以完全编码为矩阵**！

```rust
让焦距= 2.0；
让近= 0.0；
让远= 100.0；
//（既然我们有了focalLength，就不需要比例参数了）
让 P = 转置(mat4x4f(
	1.0,  0.0,       0.0,          0.0,
0.0, 比率, 0.0, 0.0,
0.0, 0.0, p_zz, p_zw,
0.0, 0.0, 1.0 / 焦距, 0.0,
));
让同类位置 = vec4f(位置, 1.0);
输出位置 = P * 同质位置；
```

系数 `p_zz` 和 `p_zw` 过去分别是 `1.0 / (far - near)` 和 `-near / (far - near)`，因此 $z_\text{out}$ 位于 $(0,1)$ 范围内。现在我们需要它在 $(0, w_\text{out}) = (0, \frac{z_\text{in}}{l})$ 范围内，以便在通过 $w_\text{out}$ 标准化后它最终会在 $(0,1)$ 中：

$$
\左\{
\开始{对齐}
p_{zz} & = \frac{f}{l(f - n)} \\
p_{zw} & = -\frac{fn}{l(f - n)}
\结束{对齐}
\对。
$$

```{topic} Proof
对于 $z_\text{in} = n$，结果为 $z_\text{out} = 0$，对于 $z_\text{in} = f$，结果为 $z_\text{out} = \frac{z_\text{in}}{l} = \frac{f}{l}$。

$$
\左\{
\开始{对齐}
n p_{zz} + p_{zw} & = 0 \quad\quad (L_1)\\
f p_{zz} + p_{zw} & = \frac{f}{l} \quad\quad (L_2)
\结束{对齐}
\对。
$$

减去 $L_2 - L_1$ 和 $f L_1 - n L_2$：

$$
\左\{
\开始{对齐}
f p_{zz} - n p_{zz} & = \frac{f}{l} \\
f p_{zw} - n p_{zw} & = -\frac{fn}{l}
\结束{对齐}
\对。
$$

除以 $f - n$ 就得到上面的结果。
```

```{attention}
（TODO：解释）如果 $n$ 不为空**，则此方法不起作用。因此，我们必须将 `near` 设置为一个小但非零的值。
```

```rust
让焦距= 2.0；
让附近= 0.01；
让远= 100.0；
让除法 = 1.0 / (focalLength * (远 - 近));
让 P = 转置(mat4x4f(
	1.0,  0.0,        0.0,                  0.0,
0.0, 比率, 0.0, 0.0,
0.0、0.0、远*除、-远*近*除、
0.0, 0.0, 1.0 / 焦距, 0.0,
));
让同类位置 = vec4f(位置, 1.0);
输出位置 = P * 同质位置；
```

```{figure} /images/focal0.5.PNG
:align: center
:class: with-shadow
我们又回到了手动除法，只是这次都是矩阵！
```

最后一个代码块中定义的矩阵 `P` 是一个 **透视投影矩阵**。

与我们最后一个公式相比，投影矩阵通常全局乘以 `focalLength` ：

```rust
让 P = 转置(mat4x4f(
焦距, 0.0, 0.0, 0.0,
0.0, 焦距*比例, 0.0, 0.0,
0.0、0.0、远/（远 - 近）、-远 * 近/（远 - 近）、
	    0.0,             0.0,                1.0,                   0.0,
));
```

这不会影响最终结果，因为它还会缩放 $w$ 坐标。同样，将 `out.position` 乘以任何值都不会更改顶点的结束像素。

```{note}
由固定管道计算的值 `out.position / out.position.w` 称为***标准化**设备坐标* (NDC)。该 NDC 必须落入上述标准化裁剪体积内。
```

```{seealso}
从数学上讲，考虑到两个向量是**等价的**，当它们是彼此的倍数时（就像我们在这里使用 `out.position` 所做的那样）定义了一个[投影空间](https://en.wikipedia.org/wiki/Projective_space)，即方向空间。它的元素由**齐次坐标**表示，这样称呼是为了提醒人们它们不是唯一的，因此它们不形成规则的（欧几里得）坐标系。
```

矩阵制服
---------------

### 坐标系

我们不是为场景中每个对象的每个顶点构建相同的矩阵，而是构建一次并将它们存储在**统一缓冲区**中。

因此我们可以扩展我们的统一结构：

```C++
// C++ side
struct MyUniforms {
	std::array<float, 16> projectionMatrix;
	std::array<float, 16> viewMatrix;
	std::array<float, 16> modelMatrix;
	std::array<float, 4> color;
	float time;
	float _pad[3];
};
```

```rust
// WGSL 端
结构体 MyUniforms {
投影矩阵：mat4x4f，
视图矩阵：mat4x4f，
模型矩阵：mat4x4f，
颜色：vec4f，
时间：f32，
};
```

```{caution}
请记住**对齐**规则：将矩阵放在第一位，因为它们是较大的结构。
```

这是更正式地介绍**分割变换的典型方式**的机会。我们可以只存储一个矩阵 $M$ ，它将对从输入位置到输出剪辑位置的整个变换进行编码，但我们将其分成 **模型** 矩阵的乘积，然后是 **视图** 矩阵，然后是 **投影** 矩阵：

```rust
// 请记住，这是向后读取的
mat4x4 M = 投影矩阵 * 视图矩阵 * 模型矩阵；
```

更改**投影**矩阵对应于更改捕获场景的虚拟相机。这种情况很少发生（除非我们创建放大/缩小效果）。

更改**视图**矩阵对应于移动和旋转相机。每当用户通常与您的工具/游戏交互时，这种情况几乎总是会发生。

更改**模型**矩阵对应于相对于全局场景（通常称为**世界**）移动对象。

因此，我们为位置变换时所经过的中间**坐标系**命名：

- `in.position` 是对象的**本地**坐标，或**模型坐标**。它描述了几何形状，就好像对象是单独的并且以原点为中心。
- `modelMatrix * in.position` 给出点的**世界**坐标，告诉它相对于全局静态框架的位置。
- `viewMatrix * modelMatrix * in.position` 给出 **相机** 坐标，或 **视图** 坐标。这是从相机看到的点的坐标。您可以将其想象为我们实际上不是移动眼睛，而是向相反方向移动和旋转整个场景。
- 最后乘以 `projectionMatrix` 应用正交或透视投影，得到 **clip** 坐标。
- 之后，固定管道将剪辑坐标除以其 $w$，从而给出 **NDC** （标准化设备坐标）。

```{note}
为了简化符号，我在上面省略了我们实际上使用齐次坐标 `vec4f(in.position, 1.0)` 作为变换的输入的事实。
```

### 预计算

目前，矩阵的内容**在CPU上预先计算**，然后上传，但这也可以在计算着色器中完成，正如我们将在本文档的[计算部分](/basic-compute/index.md)中看到的那样。

确保解除对统一缓冲区大小的设备限制，并为矩阵定义一个值：

```C++
requiredLimits.limits.maxUniformBufferBindingSize = 16 * 4 * sizeof(float);

// Upload the initial value of the uniforms
MyUniforms uniforms;
uniforms.projectionMatrix = /* ... */;
uniforms.viewMatrix = /* ... */;
uniforms.modelMatrix = /* ... */;
// [...]
```

```{warning}
请记住，我们始终添加了 `transpose` 操作。确保与我们上面的定义相比，沿矩阵翻转系数。
```

> 😒 咳咳，这有点烦人，我们难道不能定义这个 `transpose` 操作吗？那么矩阵乘法呢？

是的，我们可以，或者我们甚至可以重用已经完成的事情！这将我们引向 GLM 库。

### GLM

[GLM](https://github.com/g-truc/glm) 库再现了着色器中可用的矩阵/向量类型和操作，以便我们可以在 C++ 和着色器之间**轻松地移植代码**。

它最初被设计为尽可能接近 GLSL 语法，其功能与 WGSL 接近（尽管类型名称略有不同）。它被广泛使用，支持多个平台，经过战场测试，仅标头（因此易于集成）。

#### 整合

这是 GLM 的精简版本：[glm.zip](../../../../data/glm-0.9.9.8-light.zip)（392 KB，而不是官方版本的 5.5 MB）。将其直接解压缩到源代码树中。您可以按如下方式包含它：

```C++
#include <glm/glm.hpp> // all types inspired from GLSL
```

````{note}
确保将主源目录添加到 `CMakeLists.txt` 中的包含路径中，因为某些编译器要求它在包含指令中使用 `<...>` 括号：

```CMake
target_include_directories(App PRIVATE .)
```
````

#### 基本用法

GLM 定义的所有内容都包含在 `glm` 命名空间中。您可以通过 `using namespace glm;` 全局使用它，也可以导入单个类型：

```C++
using glm::mat4x4;
using glm::vec4;

struct MyUniforms {
	mat4x4 projectionMatrix;
	mat4x4 viewMatrix;
	mat4x4 modelMatrix;
	vec4 color;
	float time;
	float _pad[3];
};
```

```{note}
GLM 的 `mat4x4` 类型对应于 WGSL 的 `mat4x4f`。 `mat4x4<f64>` 的等效项是 `dmat4x4`，前缀 `d` 代表 `double`。它还有一个名为 `mat4` 的别名，与 GLSL 相对应，您可能会喜欢它，因为它需要输入的字符较少。对于向量（`vec3` 是 `vec3f`）、整数（`ivec2` 是 WGLS 的 `vec2<i32>`）等也是如此。
```

因此很容易重现我们在 WGSL 中所做的事情。让我们从 **模型** 转换开始：

```C++
constexpr float PI = 3.14159265358979323846f;

// Scale the object
mat4x4 S = transpose(mat4x4(
	0.3,  0.0, 0.0, 0.0,
	0.0,  0.3, 0.0, 0.0,
	0.0,  0.0, 0.3, 0.0,
	0.0,  0.0, 0.0, 1.0
));

// Translate the object
mat4x4 T1 = transpose(mat4x4(
	1.0,  0.0, 0.0, 0.5,
	0.0,  1.0, 0.0, 0.0,
	0.0,  0.0, 1.0, 0.0,
	0.0,  0.0, 0.0, 1.0
));

// Rotate the object
float angle1 = (float)glfwGetTime();
float c1 = cos(angle1);
float s1 = sin(angle1);
mat4x4 R1 = transpose(mat4x4(
	 c1,  s1, 0.0, 0.0,
	-s1,  c1, 0.0, 0.0,
	0.0, 0.0, 1.0, 0.0,
	0.0, 0.0, 0.0, 1.0
));

uniforms.modelMatrix = R1 * T1 * S;
```

然后是**视图**转换。不要忘记包括焦点的平移（我们没有将其表示为上面的矩阵乘积，但转换很简单）：

```C++
using glm::vec3;

// Translate the view
vec3 focalPoint(0.0, 0.0, -2.0);
mat4x4 T2 = transpose(mat4x4(
	1.0,  0.0, 0.0, -focalPoint.x,
	0.0,  1.0, 0.0, -focalPoint.y,
	0.0,  0.0, 1.0, -focalPoint.z,
	0.0,  0.0, 0.0,     1.0
));

// Rotate the view point
float angle2 = 3.0 * PI / 4.0;
float c2 = cos(angle2);
float s2 = sin(angle2);
mat4x4 R2 = transpose(mat4x4(
	1.0, 0.0, 0.0, 0.0,
	0.0,  c2,  s2, 0.0,
	0.0, -s2,  c2, 0.0,
	0.0, 0.0, 0.0, 1.0
));

uniforms.viewMatrix = T2 * R2;
```

最后是投影：

```C++
float ratio = 640.0f / 480.0f;
float focalLength = 2.0;
float near = 0.01f;
float far = 100.0f;
float divider = 1 / (focalLength * (far - near));
uniforms.projectionMatrix = transpose(mat4x4(
	1.0, 0.0, 0.0, 0.0,
	0.0, ratio, 0.0, 0.0,
	0.0, 0.0, far * divider, -far * near * divider,
	0.0, 0.0, 1.0 / focalLength, 0.0
));
```

顶点着色器简单地变成：

```rust
fn vs_main(in: VertexInput) -> VertexOutput {
var out：顶点输出；
out.position = uMyUniforms.projectionMatrix * uMyUniforms.viewMatrix * uMyUniforms.modelMatrix * vec4f(in.position, 1.0);
输出颜色=输入颜色；
返回；
}
```

我不会再次放置图像，您仍然应该获得相同的结果。只是这一次它的能耗要少得多，因为矩阵仅计算一次，而不是每个顶点和每帧计算一次（在实际场景中，这很容易达到数百万甚至更多）。

#### 扩展

**原子矩阵**的构造，如平移、旋转、缩放或透视是非常常见的。然而，它不是 WGSL 内置函数的一部分，因为正如我们刚才所看到的，我们不应该在着色器代码中执行此操作。

由于 GLM 打算重现着色器语言的类型，因此它也不包括这些类型。至少不在 `glm/glm.hpp` 中。但它在它的**扩展**中确实如此，我们可以像这样包含它：

```C++
#include <glm/ext.hpp>
```

模型和视图矩阵的构造变得如此简单：

```C++
S = glm::scale(mat4x4(1.0), vec3(0.3f));
T1 = glm::translate(mat4x4(1.0), vec3(0.5, 0.0, 0.0));
R1 = glm::rotate(mat4x4(1.0), angle1, vec3(0.0, 0.0, 1.0));
uniforms.modelMatrix = R1 * T1 * S;

R2 = glm::rotate(mat4x4(1.0), -angle2, vec3(1.0, 0.0, 0.0));
T2 = glm::translate(mat4x4(1.0), -focalPoint);
uniforms.viewMatrix = T2 * R2;
```

请注意，GLM 提供的变换函数都采用输入矩阵进行变换，以避免矩阵乘法。这里我们总是使用**恒等**矩阵 `mat4x4(1.0)` 来构建原子变换，但上面的模型矩阵也可以这样构建：

```C++
mat4x4 M(1.0);
M = glm::rotate(M, angle1, vec3(0.0, 0.0, 1.0));
M = glm::translate(M, vec3(0.5, 0.0, 0.0));
M = glm::scale(M, vec3(0.3f));
uniforms.modelMatrix = M;
```

但我个人觉得阅读起来比较困难，因为我们必须反向应用这些操作。

```{note}
`rotate` 函数使人们能够绕任何轴（第二个参数）转动，而不是像我们在手动构建旋转矩阵时那样仅限于 X、Y 和 Z 轴。
```

GLM 扩展还提供了构建**投影矩阵**的过程，特别是透视投影：

```C++
float near = 0.001f;
float far = 100.0f;
float ratio = 640.0f / 480.0f;
float fov = ???;
uniforms.projectionMatrix = glm::perspective(fov, ratio, near, far);
```

它具有与我们使用的参数几乎相同的参数，只是它使用 **视野** 参数，即 **fov** 来代替 **焦距**。

````{important}
实际上，`perspective` 函数依赖于两个“隐藏”设置，我们必须注意它们。这两个设置都是通过在包含 GLM 之前定义预处理器变量来全局启用的：

```C++
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#define GLM_FORCE_LEFT_HANDED
#include <glm/ext.hpp>
```

第一个是 `GLM_FORCE_DEPTH_ZERO_TO_ONE`，它告诉 GLM 剪辑体积的 Z 范围是 $(0,1)$。默认情况下，它假设它是 $(-1,1)$，因为这是 OpenGL 使用的约定，与 WebGPU 不同。

第二个是 `GLM_FORCE_LEFT_HANDED` ，表示我们的视图空间使用**左手坐标系**。这是我们迄今为止隐式采用的选择，因为剪辑空间是左手的。可以切换到右手系统，在这种情况下，请注意相机朝视图空间的 -Z 方向而不是 +Z 方向看。
````

```{note}
您还可以在 `CMakeLists.txt` 中使用 `target_compile_definition` 全局定义这些设置，以确保它们在所有文件中保持一致。
```

回到视野：它与焦距有直接关系：

```{image} /images/fov-light.svg
:align: center
:class: only-light
```

```{image} /images/fov-dark.svg
:align: center
:class: only-dark
```

从图中我们可以看出$\tan\frac{\alpha}{2} = \frac{1}{l}$，这给我们提供了以下焦距和视场之间的**转换规则**：

$$
\开始{对齐}
l & = \frac{1}{\tan(\alpha/2)} = \cot\frac{\alpha}{2} \\
\alpha & = 2 \arctan\frac{1}{l}
\结束{对齐}
$$

您很可能会使用 fov 或焦距并坚持使用它，这样就不需要转换！我们仍然可以验证我们的公式是否给出相同的结果：

```C++
float fov = 2 * glm::atan(1 / focalLength);
uniforms.projectionMatrix = glm::perspective(fov, ratio, near, far);
```

```{caution}
`glm::perspective` 预期的视野必须以**弧度**表示。如果要将其设置为$45\deg$（这是一个常见值），则必须设置`fov = 45 * PI / 180`。
```

```{figure} /images/focal0.5.PNG
:align: center
:class: with-shadow
看起来仍然一样......但我们使我们的代码库更加强大！
```

> 😟 嘿，它不再为我转身......

您需要在主循环中更新模型矩阵！

````{tab} With webgpu.hpp
```C++
// Update view matrix
angle1 = uniforms.time;
R1 = glm::rotate(mat4x4(1.0), angle1, vec3(0.0, 0.0, 1.0));
uniforms.modelMatrix = R1 * T1 * S;
queue.writeBuffer(uniformBuffer, offsetof(MyUniforms, modelMatrix), &uniforms.modelMatrix, sizeof(MyUniforms::modelMatrix));
```
````

````{tab} Vanilla webgpu.h
```C++
// Update view matrix
angle1 = uniforms.time;
R1 = glm::rotate(mat4x4(1.0), angle1, vec3(0.0, 0.0, 1.0));
uniforms.modelMatrix = R1 * T1 * S;
wgpuQueueWriteBuffer(queue, uniformBuffer, offsetof(MyUniforms, modelMatrix), &uniforms.modelMatrix, sizeof(MyUniforms::modelMatrix));
```
````

<!--
```{caution}
For some reason the developers of the WebGPU standard [deemed the assignments to *swizzles* as "unnecessary"](https://github.com/gpuweb/gpuweb/issues/737), so we cannot compactly write `position.yz = ...`, we need to use this temporary `tmp` variable. I personally find this **very annoying**, and quite limiting for productivity, I hope they might change that eventually...
```
-->

结论
----------

在这个相当数学的章节中，我们看到了基本点：

- 由于固定管道执行的坐标归一化（除以 $w$），**投影**（正交或透视）可以**编码为矩阵**。
- **透视**投影通过**焦距**或**视野**进行参数化。
- 变换矩阵（模型、视图、投影）应计算一次并存储在**统一缓冲区**中，以避免不必要的昂贵计算。
- GLM 库为我们提供了在 CPU 端轻松计算这些矩阵所需的一切。

```{seealso}
GLM 库专注于 4 维以下的向量和矩阵。对于更高维度的线性代数，我通常转而使用 [Eigen](https://eigen.tuxfamily.org) 库，但我们在这里不需要它。
```

````{tab} With webgpu.hpp
*结果代码：* [`step055`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step055)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step055-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step055-vanilla)
````
