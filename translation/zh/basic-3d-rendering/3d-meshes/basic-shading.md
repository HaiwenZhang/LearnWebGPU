基本着色<span class="bullet">🟡</span>
=============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/3d-meshes/basic-shading.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step056`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step056)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step056-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step056-vanilla)
````

从这个 3D 网格部分开始，**我们一直在作弊**，将金字塔尖端的颜色变暗，以获得一些几何形状的感觉。但实际上，关于我们周围物体的几何形状，我们获得的最强烈的**视觉线索**来自**照明效果**，特别是**阴影**。

在本章中，我们获得了关于如何对场景进行着色的**直觉**，但不遵循非常基于物理的方法（这将在稍后介绍）。

理论
------

让我们快速了解一下理论，因为我们在前一章中已经受够了。简单看一下这张图：

```{figure} /images/pexels-hatice-nogman-7961716.jpg
:align: center
:class: with-shadow
观察自然图像总是一个好主意。
```

该罐子具有或多或少均匀的材料。然而，它的不同侧面**看起来**不同，左侧的方形区域比右侧的更暗。

为什么会这样呢？因为他们有不同的**取向**。一个朝向光线，而另一个则面向不同的方向。

面部的方向表示为**垂直于面部的向量**。这称为“正规化”向量，它的长度始终为 1，因为我们关心的只是它的方向（称为“标准化”向量或“单位”向量）。

```{image} /images/normal-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>法向量表示方向。它垂直于其面，长度为 <span class="math">1</span>.</em></span>
</p>

```{note}
有**两个可能的**向量垂直于脸部并具有单位长度：一个及其相反（朝向脸部的另一侧）。 **按照惯例**，我们将法线指向对象的外部，但这对于未闭合的网格来说可能没有很好的定义。每当您遇到奇怪的阴影伪像时，请务必检查您的法线！
```

正常
------

### 数据

法线可以通过数学方法计算（使用三角形两侧的叉积），但通常**将它们存储在 3D 文件**格式中，因为有时我们故意使用假法线来给人三角形稍微弯曲的感觉。

我们将把这些法线信息添加到我们的小文件格式中，并添加一个新的顶点属性：

```
# 金字塔.txt
[积分]
# 我们添加法线向量 (nx, ny, nz)
# x y z nx ny nz r g b

# 基础
-0.5 -0.5 -0.3     0.0 -1.0 0.0    1.0 1.0 1.0
+0.5 -0.5 -0.3     0.0 -1.0 0.0    1.0 1.0 1.0
+0.5 +0.5 -0.3     0.0 -1.0 0.0    1.0 1.0 1.0
-0.5 +0.5 -0.3     0.0 -1.0 0.0    1.0 1.0 1.0

# 面的侧面有自己的顶点副本
# 因为它们有不同的法线向量。
-0.5 -0.5 -0.3  0.0 -0.848 0.53    1.0 1.0 1.0
+0.5 -0.5 -0.3  0.0 -0.848 0.53    1.0 1.0 1.0
+0.0 +0.0 +0.5  0.0 -0.848 0.53    1.0 1.0 1.0

+0.5 -0.5 -0.3   0.848 0.0 0.53    1.0 1.0 1.0
+0.5 +0.5 -0.3   0.848 0.0 0.53    1.0 1.0 1.0
+0.0 +0.0 +0.5   0.848 0.0 0.53    1.0 1.0 1.0

+0.5 +0.5 -0.3   0.0 0.848 0.53    1.0 1.0 1.0
-0.5 +0.5 -0.3   0.0 0.848 0.53    1.0 1.0 1.0
+0.0 +0.0 +0.5   0.0 0.848 0.53    1.0 1.0 1.0

-0.5 +0.5 -0.3  -0.848 0.0 0.53    1.0 1.0 1.0
-0.5 -0.5 -0.3  -0.848 0.0 0.53    1.0 1.0 1.0
+0.0 +0.0 +0.5  -0.848 0.0 0.53    1.0 1.0 1.0

[指数]
# 基础
 0  1  2
 0  2  3
# 双方
 4  5  6
 7  8  9
10 11 12
13 14 15
```

```{caution}
我必须**复制一些点**，因为虽然它们具有相同的位置，但根据它们所属的面它们具有不同的法线。实际上，顶点一般应被视为**面角**而不是 3D 点。
```

```{note}
为了更好地看到阴影的影响，这次我给整个金字塔赋予了相同的基色。
```

### 加载中

我们不需要更改几何加载过程，只需更改每个顶点的浮动属性数量：

```c++
bool success = loadGeometry(RESOURCE_DIR "/pyramid.txt", vertexData, indexData, 6);
//                                                                              ^ This was a 3
```

`vertexData` 数组现在每个顶点包含 3 个属性。为了**更好地管理**我们的顶点属性，我们可以创建一个与 `MyUniforms` 结构类似的结构：

```C++
/**
 * A structure that describes the data layout in the vertex buffer
 * We do not instantiate it but use it in `sizeof` and `offsetof`
 */
struct VertexAttributes {
	vec3 position;
	vec3 normal;
	vec3 color;
};
```

此结构镜像 WGSL 着色器中的 `VertexInput` 结构：

```rust
结构体顶点输入{
@location(0) 位置：vec3f，
@location(1) normal: vec3f, // 新属性
@location(2) 颜色：vec3f，
};
```

**字段的顺序**不需要相同。 C++ 结构体 `VertexAttributes` 中字段的顺序由数据在加载文件中存储的顺序决定。

`VertexInput` 中的顺序并不重要，并且 `@location` 必须与属性定义匹配：

```C++
std::vector<VertexAttributes> vertexAttribs(3);
//                                         ^ This was a 2

// Position attribute
vertexAttribs[0].shaderLocation = 0;
vertexAttribs[0].format = VertexFormat::Float32x3;
vertexAttribs[0].offset = offsetof(VertexAttributes, position);

// Normal attribute
vertexAttribs[1].shaderLocation = 1;
vertexAttribs[1].format = VertexFormat::Float32x3;
vertexAttribs[1].offset = offsetof(VertexAttributes, normal);

// Color attribute
vertexAttribs[2].shaderLocation = 2;
vertexAttribs[2].format = VertexFormat::Float32x3;
vertexAttribs[2].offset = offsetof(VertexAttributes, color);

// [...]

vertexBufferLayout.arrayStride = sizeof(VertexAttributes);
//                               ^^^^^^^^^^^^^^^^^^^^^^^^ This was 6 * sizeof(float)
```

并且不要忘记更改设备限制：

```C++
// We changed the number of attributes
requiredLimits.limits.maxVertexAttributes = 3;
//                                          ^ This was a 2
requiredLimits.limits.maxBufferSize = 16 * sizeof(VertexAttributes);
//                                         ^^^^^^^^^^^^^^^^^^^^^^^^ This was 6 * sizeof(float)
//                                    ^^ This was 15
requiredLimits.limits.maxVertexBufferArrayStride = sizeof(VertexAttributes);
//                                                        ^^^^^^^^^^^^^^^^^^^^^^^^ This was 6 * sizeof(float)

// We changed the number of components transiting from vertex to fragment shader
// (to color.rgb we added normal.xyz, hence a total of 6 components)
requiredLimits.limits.maxInterStageShaderComponents = 6;
//                                                    ^ This was a 3
```

阴影
-------

### 光方向

现在，普通数据已从文件中加载，可供顶点着色器访问。但是着色发生在**片段着色器**中，所以我们需要转发法线属性：

```rust
结构体顶点输出{
@builtin(position) 位置：vec4f，
@位置（0）颜色：vec3f，
@location(1) normal: vec3f, // <--- 添加正常输出
};

// [...]

@顶点
fn vs_main(in: VertexInput) -> VertexOutput {
	// [...]
// 转发正常
输出.正常=输入.正常;
返回；
}
```

为了**检查所有内容是否正确连接**，您可以尝试仅使用法线向量的坐标作为输出颜色。由于这些坐标在 $(-1,1)$ 范围内，我通常添加 ` * 0.5 + 0.5` 将它们重新映射到 $(0,1)$ 范围内，这是颜色输出所期望的：

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让颜色 = in.normal * 0.5 + 0.5；
	// [...]
}
```

<figure class="align-center">
<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
<source src="../../_static/shading-fwd-normal.mp4" type="video/mp4">
</video>
<figcaption>
<p><span class="caption-text">这些颜色是正常“调色板”的典型颜色。</span></p>
</figcaption>
</figure>

现在让我们用这个法线做一些实验：

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让颜色 = in.color * in.normal.x;
	// [...]
}
```

```{figure} /images/pyramid-axis-lights.png
:align: center
:class: with-shadow
将颜色乘以法线轴可创建轴对齐的定向光。
```

为了应用来自任意方向的照明，我们再次使用不同轴的线性组合：

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让阴影= 0.5 * in.normal.x - 0.9 * in.normal.y + 0.1 * in.normal.z;
让颜色 = in.color * 阴影；
	// [...]
}
```

```{figure} /images/pyramid-first-light.png
:align: center
:class: with-shadow
混合多个轴可以创建来自任何方向的定向光。
```

系数 $(0.5, -0.9, 0.1)$ 实际上是 **光线方向** 这种组合称为 **点积**：

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让 lightDirection = vec3f(0.5, -0.9, 0.1);
让 shading = dot(lightDirection, in.normal);
让颜色 = in.color * 阴影；
	// [...]
}
```

```{note}
术语“方向”表明这是一个**归一化**向量（即长度为 $1$ 的向量。这里我们实际上通过向量的幅度（即长度）对方向加上光的**强度**进行编码。
```

### 多个灯

添加多个光源就像对多个方向的贡献求和一样简单。但有一件重要的事情：

```rust
让 shading = max(0.0, dot(lightDirection, in.normal));
// ^^^^^^^^ 这将负值限制为 0.0
```

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让 lightDirection1 = vec3f(0.5, -0.9, 0.1);
让 lightDirection2 = vec3f(0.2, 0.4, 0.3);
让 shading1 = max(0.0, dot(lightDirection1, in.normal));
让 shading2 = max(0.0, dot(lightDirection2, in.normal));
让阴影=阴影1+阴影2；
让颜色 = in.color * 阴影；
	// [...]
}
```

```rust
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
让 lightColor1 = vec3f(1.0, 0.9, 0.6);
让 lightColor2 = vec3f(0.6, 0.9, 1.0);
	// [...]
让 shading = shading1 * lightColor1 + shading2 * lightColor2;
让颜色 = in.color * 阴影；
	// [...]
}
```

<figure class="align-center">
<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
<source src="../../_static/shading01.mp4" type="video/mp4">
</video>
<figcaption>
<p><span class="caption-text">两个彩色灯.</span></p>
</figcaption>
</figure>

变换
---------

在上一部分中，光线方向随着物体的方向而变化。要应用固定的全局光照，我们需要转换法线。模型变换，但不是视图变换（因此有区别）。

```rust
// 在顶点着色器中
out.normal = (uMyUniforms.modelMatrix * vec4f(in.normal, 0.0)).xyz;

// [...]

// 在片段着色器中
让正常=正常化（in.正常）；
//（并在 shading1/shading2 中将 in.normal 替换为正常）
```

<figure class="align-center">
<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
<source src="../../_static/shading02.mp4" type="video/mp4">
</video>
<figcaption>
<p><span class="caption-text">固定光线方向.</span></p>
</figcaption>
</figure>

结论
----------

我们将在[光照和材质](/basic-3d-rendering/lighting-and-material/index.md) 章节中看到更准确的材质模型。

````{tab} With webgpu.hpp
*结果代码：* [`step056`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step056)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step056-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step056-vanilla)
````
