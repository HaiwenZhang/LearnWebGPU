立方体贴图 (<span class="bullet">🟠</span>WIP)
=========

```{translation-warning} 译文可能已过时, /basic-3d-rendering/lighting-and-material/cubemap.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step117`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step117)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step117-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step117-vanilla)
````

由于 `acos` 和 `atan2` 操作，我们在上一章中对环境光照进行采样的 `ibl_uv` 坐标的计算成本有点高。存储环境贴图的更有效方法是作为**立方体贴图**。

待办事项

```{figure} /images/cubemap-conv/folded.svg
:align: center
立方体贴图的采样和硬件加速更加高效。
```

多层纹理
--------------------

我们将在[立方体贴图转换](../../basic-compute/image-processing/cubemap-conversion.md)章节中看到如何将等距柱状环境贴图转换为立方体贴图，反之亦然。

现在我们需要知道的是**立方体贴图是一种特殊类型的纹理**。它存储为 **2D 数组纹理**，具有 **6 层**，这意味着在创建纹理时，指定维度为 `2D` 但 `size` 有 3 个维度：

```C++
TextureDescriptor textureDesc;
textureDesc.dimension = TextureDimension::_2D;
textureDesc.size = { size, size, 6 };
// [...]
```

**按照惯例**，立方体的面按以下**顺序**存储：

|层 |立方体贴图面|  S |  T |
| :---: | :-----------: | :--: | :--: |
|   0 | `Positive X` | `-Z` | `-Y` |
|   1 | `Negative X` | `+Z` | `-Y` |
|   2 | `Positive Y` | `+X` | `+Z` |
|   3 | `Negative Y` | `+X` | `-Z` |
|   4 | `Positive Z` | `+X` | `-Y` |
|   5 | `Negative Z` | `-X` | `-Y` |

正如您所看到的，该约定还指定了局部纹理轴 `S` 和 `T` 对应的世界空间方向。

```{image} /images/cubemap-conv/stacked-light.svg
:align: center
:class: only-light
```

```{image} /images/cubemap-conv/stacked-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>CubeMaps 表示为 2D 数组纹理。</em></span>
</p>

在实践中，我们从单个文件中**一张一张地加载面孔**。 **MIP 级别**的计算也是面对面完成的。纹理采样器将负责将面适当地混合在一起。

```C++
Extent3D singleLayerSize = { size, size, 1 };
for (uint32_t layer = 0; layer < 6; ++layer) {
    destination.origin = { 0, 0, layer };
    m_queue.writeTexture(destination, pixelData[layer], (size_t)(4 * size * size), source, singleLayerSize);
}
```

```{image} /images/cubemap-conv/faces-light.svg
:align: center
:class: only-light
```

```{image} /images/cubemap-conv/faces-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>立方体贴图的每个面都是从不同的图像文件加载的。</em></span>
</p>

```{note}
图像出现颠倒是因为惯例是由使用 $Y$ 作为垂直轴的人设计的，在本指南中我们使用 $Z$ 作为垂直轴。无论如何，即使使用 $Y$-up 时，最好还是坚持上面的约定表，而不是尝试直观地猜测正确的 S 和 T 纹理轴。
```

待办事项

实施
--------------

待办事项

将 [`autumn_park_4k.zip`](../../../../data/autumn_park_4k.zip) 解压到 `resource` 目录中。

```C++
// In Application.h
bool initTexture(const std::filesystem::path& path, bool isCubemap = false);

// In onInit()
if (!initTexture(RESOURCE_DIR "/autumn_park_4k"), true /* isCubemap */) return false;

// In Application.cpp
bool Application::initTexture(const std::filesystem::path& path, bool isCubemap) {
    TextureView textureView = nullptr;
    Texture texture =
        isCubemap
        ? ResourceManager::loadCubemapTexture(path, m_device, &textureView)
        : ResourceManager::loadTexture(path, m_device, &textureView);

    // [...]

    bindingLayout.texture.viewDimension =
        isCubemap
        ? TextureViewDimension::Cube
        : TextureViewDimension::_2D;

    // [...]
}
```

```C++
// In ResourceManager.h
static wgpu::Texture loadCubemapTexture(const path& path, wgpu::Device device, wgpu::TextureView* pTextureView = nullptr);

// In ResourceManager.cpp
Texture ResourceManager::loadCubemapTexture(const path& path, Device device, TextureView* pTextureView) {
    const char* cubemapPaths[] = {
        "cubemap-posX.png",
        "cubemap-negX.png",
        "cubemap-posY.png",
        "cubemap-negY.png",
        "cubemap-posZ.png",
        "cubemap-negZ.png",
    };

    // Load image data for each of the 6 layers
    Extent3D cubemapSize = { 0, 0, 6 };
    std::array<uint8_t*, 6> pixelData;
    for (uint32_t layer = 0; layer < 6; ++layer) {
        int width, height, channels;
        auto p = path / cubemapPaths[layer];
        pixelData[layer] = stbi_load(p.string().c_str(), &width, &height, &channels, 4 /* force 4 channels */);
        if (nullptr == pixelData[layer]) throw std::runtime_error("Could not load input texture!");
        if (layer == 0) {
            cubemapSize.width = (uint32_t)width;
            cubemapSize.height = (uint32_t)height;
        }
        else {
            if (cubemapSize.width != (uint32_t)width || cubemapSize.height != (uint32_t)height)
                throw std::runtime_error("All cubemap faces must have the same size!");
        }
    }

    // [...]
    textureDesc.size = cubemapSize;

    // [...]
    Extent3D cubemapLayerSize = { cubemapSize.width , cubemapSize.height , 1 };
    for (uint32_t layer = 0; layer < 6; ++layer) {
        Extent3D origin = { 0, 0, layer };

        writeMipMaps(device, texture, cubemapLayerSize, textureDesc.mipLevelCount, pixelData[layer], origin);

        // Free CPU-side data
        stbi_image_free(pixelData[layer]);
    }

    // [...]
    textureViewDesc.arrayLayerCount = 6;
    //                                ^ This was 1
    textureViewDesc.dimension = TextureViewDimension::Cube;
    //                                                ^ This was 2D
```

请注意，我们还向 `writeMipMaps` 添加了一个新的额外参数来指定上传到哪个层：

```C++
template<typename component_t>
static void writeMipMaps(
    /* [...] */
    Origin3D origin = { 0, 0, 0 }
) {
    // [...]
    destination.origin = origin;
    // ^                 ^ This was { 0, 0, 0 }
```

```rust
// 在着色器中
@group(0) @binding(4) varcubemapTexture:texture_cube<f32>;

// [...]

让ibl_sample =textureSample（cubemapTexture，textureSampler，ibl_direction）.rgb;
// ^ 这是 ibl_uv
```

待办事项

结论
----------


````{tab} With webgpu.hpp
*结果代码：* [`step117`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step117)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step117-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step117-vanilla)
````
