基于图像的照明 (<span class="bullet">🟠</span>WIP)
====================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/lighting-and-material/ibl.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step115`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step115)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step115-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step115-vanilla)
````

```{figure} /images/autumn_park.webp
:align: center
:class: with-shadow
环境贴图是高动态范围内的 360° 图像，我们将其用作全向光源。
```

```{figure} /images/ibl-coords.png
:align: center
对于每条相机光线（即每个像素），我们计算表面法线反射的方向，以对环境贴图中对照明贡献最大的部分进行采样。
```

IBL 采样
------------

```{admonition} 🚧 WIP
从本章开始，该指南不再是最新的。我目前正在逐章刷新，这就是我目前工作的地方！
```

待办事项

将环境贴图 [`autumn_park_4k.jpg`](../../../../data/autumn_park_4k.jpg) 添加到加载的纹理列表中：

```C++
// [...]
if (!initTexture(RESOURCE_DIR "/autumn_park_4k.jpg")) return false;
```

```{note}
您可以在 [PolyHaven](https://polyhaven.com/hdris) 和 [ambientCG](https://ambientcg.com/list?type=HDRI)] 上找到更多环境贴图示例。这些是 CC0 许可的，允许您在任何情况下使用它们！
```

我们在比船更简单的模型上进行测试：[`suzanne.obj`](../../../../data/suzanne.obj)。

### 着色器

```rust
@group(0) @binding(5) var<uniform> uLighting: LightingUniforms;
// ^ // 这是 6
// [...]
@group(0) @binding(4) varenvironmentTexture:texture_2d<f32>;

// [...]

// 我们不对光源进行循环，而是对环境贴图进行采样
// 在反射方向上获得镜面反射阴影。
让 ibl_direction = -reflect(V, N);
让 Pi = 3.14159265359；

// 将方向转换为球坐标
让 theta = acos(ibl_direction.z / length(ibl_direction));
让 phi = atan2(ibl_direction.y, ibl_direction.x);

// 将球面坐标映射到 (0,1) 范围以适合 UV 空间
让 ibl_uv = vec2<f32>(phi / (2.0 * Pi) + 0.5, theta / Pi);

// 对纹理进行采样
让 ibl_sample =textureSample(environmentTexture,textureSampler,ibl_uv).rgb;

var 漫反射 = vec3<f32>(0.0);
让镜面反射 = ibl_sample;

//（删除此处的 for 循环）
```

```{figure} /images/envmap-ldr.png
:align: center
:class: with-shadow
待办事项
```

IBL预过滤
----------------

我们可以通过采样不同的 MIP 级别来赋予对象更多的粗糙度：

```rust
// 对纹理进行采样
让 ibl_sample =textureSampleLevel（环境纹理，textureSampler，ibl_uv，6.0）.rgb;
```

```{figure} /images/envmap-ldr-rough.png
:align: center
:class: with-shadow
待办事项
```

事实上，我们不是像常规纹理那样让 GPU 猜测我们想要哪个 MIP 级别，而是根据每个表面元素所需的粗糙度手动指定我们想要的级别。

例如，我们可以创建交替的粗糙条纹和光泽条纹：

```rust
// 对纹理进行采样
让级别 = mix(1.0, 6.0, step(fract(20.0 * in.uv.y), 0.5));
让 ibl_sample =textureSampleLevel(environmentTexture、textureSampler、ibl_uv、level).rgb;
```

```{figure} /images/envmap-ldr-stripes.png
:align: center
:class: with-shadow
待办事项
```

结论
----------

到目前为止，我们的方法存在 3 个问题：

- 通过手动设置 MIP 级别，我们失去了 GPU 根据屏幕空间梯度进行的“智能”选择，以避免出现锯齿。为了获得手动控制和此行为，我们可以使用 `textureSampleBias`。

- MIP 级别的生成就好像纹理代表沿网格定期采样的信号，但环境贴图是沿方向采样的，这会导致 MIP 生成应考虑到的失真。

- 为了能够同时表示间接照明区域中的太阳和颜色细节，环境纹理通常使用 **高动态范围** (HDR) 图像格式，例如 .hdr 或 .exr。

我们在下一章中将看到一种更有效的方法来使用特殊类型的纹理（即**立方体贴图**）对环境进行采样。在[HDR纹理](../../advanced-techniques/hdr-textures.md)章节中，我们将看到如何加载HDR纹理。

````{tab} With webgpu.hpp
*结果代码：* [`step115`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step115)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step115-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step115-vanilla)
````
