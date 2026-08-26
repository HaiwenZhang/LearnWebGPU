立方体转换(<span class="bullet">🟠</span>WIP)
==================

```{translation-warning} 译文可能已过时, /basic-compute/image-processing/cubemap-conversion.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step220`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step220)

问题
-------

记住从[基于图像的照明](../../basic-3d-rendering/lighting-and-material/ibl.md)章节? 我们实际上可以从平方环境图中构建它们,例如:[波利哈文语Name](https://polyhaven.com/hdris)或[环境CG](https://ambientcg.com/list?type=HDRI).

**输入 :**

```{figure} /images/autumn_park.webp
:align: center
:class: with-shadow
环境图是高动力范围内的360°图像,我们作为全向光源使用.
```

**参数化:**

```{figure} /images/ibl-coords.png
:align: center
正方形地图以纬度和经度毕业为参数,与地球一样.
```

**输出 :**

```{figure} /images/cubemap-conv/cubemap.svg
:align: center
立方体由6平方纹理制成,或者说1个纹理有6层. 每层都对应一个方块的一面来包裹场景.
```

由于比较容易找到等效的图像,但立方体的查询速度更快(因为硬件加速了),我们的目标是将等效的图像转换成立方体.

执行情况
--------------

开始

```C++
// The output is a square texture with 6 layers, one per face of the cube
textureDesc.size = { (uint32_t)height, (uint32_t)height, 6 };
m_outputTexture = m_device.createTexture(textureDesc);
```

```C++
// We create 1 view per face of the cube (i.e., per layer of the output texture)
// NB: This is only used when drawing the GUI
std::array<wgpu::TextureView, 6> m_outputTextureLayers = {nullptr, nullptr, nullptr, nullptr, nullptr, nullptr};
```

```C++
const char* outputLabels[] = {
	"Output Positive X",
	"Output Negative X",
	"Output Positive Y",
	"Output Negative Y",
	"Output Positive Z",
	"Output Negative Z",
};

for (uint32_t i = 0; i < 6; ++i) {
	textureViewDesc.label = outputLabels[i];
	textureViewDesc.baseArrayLayer = i;
	m_outputTextureLayers[i] = m_outputTexture.createView(textureViewDesc);
}

// We still use the view that covers all layers, because we will draw all of
// them in a single dispatch.
textureViewDesc.baseArrayLayer = 0;
textureViewDesc.arrayLayerCount = 6;
textureViewDesc.dimension = TextureViewDimension::_2DArray;
m_outputTextureView = m_outputTexture.createView(textureViewDesc);
```

在绑定组布局中,我们将纹理维度切换到`2DArray`:

```C++
bindings[1].storageTexture.viewDimension = TextureViewDimension::_2DArray;
//                                                               ^ This was _2D
```

在阴影中,我们必须改变类型或`outputTexture`:

```rust
@ group( 0) @ 绑定(1) var 输出Texture:纹理_存储_第2款d项_数组<rgba8unorm,write>;
// ^ 这是纹理_存储_第2款d项
```

呼叫`textureStore`然后用一个新的参数,即我们所写的图层:

```rust
让图层=id.z;
纹理Store(输出Texture, id.xy, 层, 颜色);
```

我们发射`4 * 4 * 6 = 96 = 3 * 32`线程 :

```rust
@ compute @ workgroup_大小(4、4、6)
```

```C++
uint32_t invocationCountX = m_outputTexture.getWidth();
uint32_t invocationCountY = m_outputTexture.getHeight();
uint32_t workgroupSizePerDim = 4;
```

开始

结论
----------

*结果代码 :* [`step220`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step220)
