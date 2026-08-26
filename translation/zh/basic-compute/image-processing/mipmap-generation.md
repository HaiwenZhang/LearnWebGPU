映射生成<span class="bullet">🟡</span>
=================

```{translation-warning} 译文可能已过时, /basic-compute/image-processing/mipmap-generation.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step211`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step211)

我建议我们新学习的 计算阴影器的第一个应用是**生成缩略图**在关于[纹理取样](../../basic-3d-rendering/texturing/sampler.md#filtering)在3D网格上应用纹理之前, 我们预先计算出不同的下采样版本。

```{image} /images/mipmap-generation/problem-light.png
:align: center
:class: only-light
```

```{image} /images/mipmap-generation/problem-dark.png
:align: center
:class: only-dark
```

<p class="align-center">
    <span class="caption-text"><em>产出一级比较#1和投入#0. 每桶#1是平均4特克塞尔英寸#0.</em></span>
</p>

当时,**我们在CPU上建金字塔**,在将纹理数据上传到GPU纹理对象之前。 但问题是**非常适合计算阴影**!

一个非常平行的问题
-----------------------

MIP级别的生成问题概括如下:$n - 1$我们计算一个特克塞尔$(i,j)$员额数$n$以MIP水平的平均值表示$n - 1$带坐标$(2 i + u,2j + v)$时,$u$和$v$间距$\{0,1\}$.

页:1**优质财产**问题是 处理是**非常当地**: MIP 级的特克塞尔$n$只依赖上一个级别的少量固定的特克塞尔$n - 1$,并且只为下一级别贡献一个特克塞尔$n + 1$.

似乎有两种方式将这一问题作为平行工作的派遣:

 - **备选案文A**每一级像素一线程$n$,每个特克塞尔$(i,j)$从 MIP 一级获取 4 个特克塞尔$n - 1$并平均他们。

 - **备选案文B**每一级像素一线程$n - 1$,每个特克塞尔$(i',j')$该级别除以$4$(平均特克塞尔的总数)和累积在特克塞尔的总数$(i'/2, j'/2)$员额数$n$.

选项A有4倍小的线程,但每个线程会做4条纹理读取,而不是选项B的1条纹理读取. 两种选项都做1个写取.

```{note}
在这样一个简单的数学操作中,限制因素是内存访问,我们并不真正关心平均本身的计算.
```

**破坏器警报 :**选项 由于多种原因,A是本案中最好的:

- 备选案文B要求**初始化的额外通行证**输出 0 的所有 texels 。
- 备选案文B导致**种族条件**因为累积值需要读取当前平均值, 然后写入更新的平均值, 如果多个线程都这样做**并行写入操作将发生冲突**.
- 其实是的**甚至无法从相同的纹理读写**在一个单一的阴影中。

出于所有这些原因,我们将执行**备选案文A**.

Input/Output
------------

我们首先需要加载输入纹理,并保存输出,以检查这一过程是否运作良好. 为了测试,我们用**具有2 MIP 级的单一纹理**:MIP级别0是输入图像,MIP级别1是计算阴影器的输出.

###正在装入

举个例子,我们要装上以下[`input.jpg`](../../../../images/mipmap-generation/input.jpg):

```{figure} /images/mipmap-generation/input.jpg
:align: center
:class: with-shadow
我们的例子输入图像。
```

添加新的内置步骤和纹理相关属性到`Application`类 :

```C++
// In Application.h
void initTexture();
void terminateTexture();

void initTextureViews();
void terminateTextureViews();

// [...]

wgpu::Extent3D m_textureSize;
wgpu::Texture m_texture = nullptr;
wgpu::TextureView m_inputTextureView = nullptr;
wgpu::TextureView m_outputTextureView = nullptr;
```

那个`initTexture()`方法开始像[我们的纹理加载程序](../../basic-3d-rendering/texturing/loading-from-file.md),只有我们不包括 mipmap 生成:

```C++
void Application::initTexture() {
    // Load image data
    int width, height, channels;
    uint8_t* pixelData = stbi_load(RESOURCE_DIR "/input.jpg", &width, &height, &channels, 4 /* force 4 channels */);
    if (nullptr == pixelData) throw std::runtime_error("Could not load input texture!");
    m_textureSize = { (uint32_t)width, (uint32_t)height, 1 };

    // Create texture
    TextureDescriptor textureDesc;
    textureDesc.dimension = TextureDimension::_2D;
    textureDesc.format = TextureFormat::RGBA8Unorm;
    textureDesc.size = m_textureSize;
    textureDesc.sampleCount = 1;
    textureDesc.viewFormatCount = 0;
    textureDesc.viewFormats = nullptr;

    textureDesc.usage = (
        TextureUsage::TextureBinding | // to read the texture in a shader
        TextureUsage::StorageBinding | // to write the texture in a shader
        TextureUsage::CopyDst | // to upload the input data
        TextureUsage::CopySrc   // to save the output data
    );

    // We start with 2 MIP levels:
    //  - level 0 is given by the input.jpg file
    //  - level 1 is filled by the compute shader
    textureDesc.mipLevelCount = 2;

    m_texture = m_device.createTexture(textureDesc);

    // [...] Upload texture data for MIP level 0 to the GPU

    // Free CPU-side data
    stbi_image_free(pixelData);
}
```

````{note}
我们用[`stb_image.h`](https://raw.githubusercontent.com/nothings/stb/master/stb_image.h)要装入图像的文件。 不要忘记包含并添加到`implementations.cpp`目标如下:

```C++
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```
````

一旦我们有了纹理,我们只输入MIP级别0:

```C++
Queue queue = m_device.getQueue();

// Upload texture data for MIP level 0 to the GPU
ImageCopyTexture destination;
destination.texture = m_texture;
destination.origin = { 0, 0, 0 };
destination.aspect = TextureAspect::All;
destination.mipLevel = 0;
TextureDataLayout source;
source.offset = 0;
source.bytesPerRow = 4 * m_textureSize.width;
source.rowsPerImage = m_textureSize.height;
queue.writeTexture(destination, pixelData, (size_t)(4 * width * height), source, m_textureSize);

#if !defined(WEBGPU_BACKEND_WGPU)
    wgpuQueueRelease(queue);
#endif
```

那个`initTextureViews()`输入和输出视图的唯一区别在于 MIP 级别:

```C++
void Application::initTextureViews() {
    TextureViewDescriptor textureViewDesc;
    textureViewDesc.aspect = TextureAspect::All;
    textureViewDesc.baseArrayLayer = 0;
    textureViewDesc.arrayLayerCount = 1;
    textureViewDesc.dimension = TextureViewDimension::_2D;
    textureViewDesc.format = TextureFormat::RGBA8Unorm;

    // Each view must correspond to only 1 MIP level at a time
    textureViewDesc.mipLevelCount = 1;

    textureViewDesc.baseMipLevel = 0;
    textureViewDesc.label = "Input View";
    m_inputTextureView = m_texture.createView(textureViewDesc);

    textureViewDesc.baseMipLevel = 1;
    textureViewDesc.label = "Output View";
    m_outputTextureView = m_texture.createView(textureViewDesc);
}
```

###保存

为了保存输出图像, 我们使用[`stb_image_write.h`](https://raw.githubusercontent.com/nothings/stb/master/stb_image_write.h),的伴侣,`stb_image.h`用于写入文件,并添加到`implementations.cpp`目标如下:

```C++
#define STB_IMAGE_WRITE_IMPLEMENTATION
#define __STDC_LIB_EXT1__
#include "stb_image_write.h"
```

将纹理读回CPU的高级过程如下:

1. 创建一个GPU缓冲器,其字节大小与您想要保存的MIP级别相同.
2. 使用`encoder.copyTextureToBuffer(...)`.
3. 绘制缓冲器,就像我们用`mapBuffer`在前一章中。
4. 在地图回调中,使用`stbi_write_png`将图像写入磁盘。

我把这些细节留作练习,你可以只包括这个[`save_texture.h `](https://gist.github.com/eliemichel/0a94203fd518c70f3c528f3b2c7f73c8)文件输入您的工程,并在末尾`onCompute()`:

```C++
saveTexture(RESOURCE_DIR "/output.png", m_device, m_texture, 1 /* output MIP level */);
```

```{note}
与前一章缓冲相关的一切内容都可以删除.
```

###约束

内`initBindGroupLayout`和`initBindGroup`我们用纹理观点的束缚代替缓冲器

用于**输入绑定**与3D渲染中使用的纹理绑定类似,唯一的区别是`visibility`:

```C++
// In initBindGroupLayout():
// Input image: MIP level 0 of the texture
bindings[0].binding = 0;
bindings[0].texture.sampleType = TextureSampleType::Float;
bindings[0].texture.viewDimension = TextureViewDimension::_2D;
bindings[0].visibility = ShaderStage::Compute;
```

在**阴影**:

```rust
@ group( 0) @ 绑定( 0) var 上一个 MipLevel: 纹理_第2款d项<f32>;
```

那个**输出绑定**不一样 因为`texture`约束总是随时随地。 我们需要的是**存储纹理**绑定 :

```C++
// In initBindGroupLayout():
// Output image: MIP level 1 of the texture
bindings[1].binding = 1;
bindings[1].storageTexture.access = StorageTextureAccess::WriteOnly;
bindings[1].storageTexture.format = TextureFormat::RGBA8Unorm;
bindings[1].storageTexture.viewDimension = TextureViewDimension::_2D;
bindings[1].visibility = ShaderStage::Compute;
```

此处对应**阴影**改为:

```rust
@ group( 0) @ bonding(1) var nextMipLevel: 纹理_存储_第2款d项<rgba8unorm,write>;
```

纹理存储器**更详细的格式信息**而非纹理:`rgba8unorm`表示纹理的基本格式是4个通道的8位元,我们操纵阴影中的texels作为"未签名的正态"值,即范围中的浮点$(0, 1)$.

```{note}
只有`write`允许访问`texture_storage_2d`。其他访问可能引入WebGPU的后期版本。
```

内`initBindGroup`,条目非常简单:

```C++
// Input buffer
entries[0].binding = 0;
entries[0].textureView = m_inputTextureView;

// Output buffer
entries[1].binding = 1;
entries[1].textureView = m_outputTextureView;
```

很好,一切都准备好了,我们现在可以专注于 实际的计算阴影器。

计算
-----------

###调度

我建议我们用一个工作组的规模$8 \times 8$: 两者都这样$X$和$Y$轴对称并概括到64个线程,这是典型曲面尺寸的合理倍数.

```rust
@ 工作组_大小( 8, 8)
```

然后,我们需要计算派出工作组的数目。 这取决于线程的预期数量. 在我们所谓的选择A中,我们发起**每特克塞尔一线程**会 议 日 程 和 议 程**产出**中期计划一级:

```C++
uint32_t invocationCountX = m_textureSize.width / 2;
uint32_t invocationCountY = m_textureSize.height / 2;
```

```C++
uint32_t workgroupSizePerDim = 8;
// This ceils invocationCountX / workgroupSizePerDim
uint32_t workgroupCountX = (invocationCountX + workgroupSizePerDim - 1) / workgroupSizePerDim;
uint32_t workgroupCountY = (invocationCountY + workgroupSizePerDim - 1) / workgroupSizePerDim;
computePass.dispatchWorkgroups(workgroupCountX, workgroupCountY, 1);
```

```{note}
我在上面提供的输入图像的大小是**2 权力**总是让事情变得容易 因为没有浪费的线。
```

###阴影

每一个特克塞尔,我们使用`textureLoad`4倍于上一个MIP水平的平均值`textureStore`在新的 MIP 级别中写入:

```rust
@ compute @ workgroup_大小( 8, 8)
fn 计算MipMap (@ buildingin( 全球)_援引_编号) 编号:vec3<u32>) {
页:1<u32>(0, 1);
让颜色=(
纹理LOAD( 先前的MipLevel, 2)*页:1
纹理LOAD( 先前的MipLevel, 2)*页:1
纹理LOAD( 先前的MipLevel, 2)*页:1
纹理LOAD( 先前的MipLevel, 2)*页:1
    ) * 0.25;
纹理Store( 下一个MipLevel, id. xy, 颜色);
}
```

```{important}
最后的论点是:`textureLoad`是一个 MIP 级别**相对于纹理视图**我们的束缚。 因此,在这两种情况中,我们审视了第一个(而且只有)MIP级别。
```

就是这样! 大部分工作都是关于绑定输入/输出,计算阴影本身非常简单.

###结果

我们可以检查结果 检查它是否匹配一个[参考文献](../../../../images/mipmap-generation/reference.png),并使用像 GIMP 或 Photoshop 这样的常规图像编辑工具进行下映:

```{image} /images/mipmap-generation/compare-light.png
:align: center
:class: only-light
```

```{image} /images/mipmap-generation/compare-dark.png
:align: center
:class: only-dark
```

<p class="align-center">
    <span class="caption-text"><em>产出一级比较#1 有参考文献. 错误的地图都是黑色的,意味着这是完美的匹配.</em></span>
</p>

*中间产生的代码 :* [`step210`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step210)

适用于所有中期计划级别
-----------------------------

###跨 MIP 级循环

我们知道如何计算MIP级别$1$,考虑到中期计划一级$0$将这一点概括为循环是相当直截了当的。 我们把我们的发送 进入一个循环 在所有MIP级别:

```C++
ComputePassEncoder computePass = encoder.beginComputePass(computePassDesc);
computePass.setPipeline(m_pipeline);

uint32_t levelCount = getMaxMipLevelCount(m_textureSize);
for (uint32_t nextLevel = 1; nextLevel < levelCount; ++nextLevel) {
    computePass.setBindGroup(0, m_bindGroup, 0, nullptr);
    // [...] Compute workgroup counts
    computePass.dispatchWorkgroups(workgroupCountX, workgroupCountY, 1);
}

computePass.end();
```

```{note}
循环从1级开始,因为更一般地它负责计算级别`nextLevel`给定级别`nextLevel - 1`.
```

那个`getMaxMipLevels()`在计算CPU上的 mipmaps 时, 用于纹理加载的公用功能是相同的 :

```C++
// Equivalent of std::bit_width that is available from C++20 onward
uint32_t bit_width(uint32_t m) {
    if (m == 0) return 0;
    else { uint32_t w = 0; while (m >>= 1) ++w; return w; }
}

uint32_t getMaxMipLevelCount(const Extent3D& textureSize) {
    return bit_width(std::max(textureSize.width, textureSize.height));
}
```

我们在创建纹理时也使用此功能:

```C++
// In initTexture():
textureDesc.mipLevelCount = getMaxMipLevelCount(m_textureSize);
```

###每MIP一级一个视图

两种情况因发送的迭代而不同:

- 线程数(即下一个MIP级别的Texel数)
- 捆绑组

为了在每个迭代建立不同的结合组,我们创建于`initTextureViews`不仅有2个观点,而且每个任务执行计划都有一个观点。 我们还同时计算每一级别的规模。

```C++
// In Application.h, replacing input/output views and m_textureSize
std::vector<wgpu::TextureView> m_textureMipViews;
std::vector<wgpu::Extent3D> m_textureMipSizes;
```

```C++
// In initTextureViews()
m_textureMipViews.reserve(m_textureMipSizes.size());
for (uint32_t level = 0; level < m_textureMipSizes.size(); ++level) {
    std::string label = "MIP level #" + std::to_string(level);
    textureViewDesc.label = label.c_str();
    textureViewDesc.baseMipLevel = level;
    m_textureMipViews.push_back(m_texture.createView(textureViewDesc));

    if (level > 0) {
        Extent3D previousSize = m_textureMipSizes[level - 1];
        m_textureMipSizes[level] = {
            previousSize.width / 2,
            previousSize.height / 2,
            previousSize.depthOrArrayLayers / 2
        };
    }
}
```

```{important}
作为可写存储和可读纹理绑定的视图不得共享任何 MIP 级别 !
```

我们还确保初始化大小`m_textureMipSizes[0]`第一级:

```C++
// In initTexture()
Extent3D textureSize = { (uint32_t)width, (uint32_t)height, 1 };
// [...]
textureDesc.mipLevelCount = getMaxMipLevelCount(textureSize);
m_textureMipSizes.resize(textureDesc.mipLevelCount);
m_textureMipSizes[0] = textureSize;
```

###捆绑组

最后,我们可以增加一个 MIP 级别参数`initBindGroup()`方法并称之为计算循环。 别忘了打电话`terminateBindGroup()`每个迭代的结尾。

```C++
void Application::initBindGroup(uint32_t nextMipLevel) {
    std::vector<BindGroupEntry> entries(2, Default);
    // Input buffer
    entries[0].binding = 0;
    entries[0].textureView = m_textureMipViews[nextMipLevel - 1];
    // Output buffer
    entries[1].binding = 1;
    entries[1].textureView = m_textureMipViews[nextMipLevel];

    // [...]
}
```

因此,主要发送循环如下:

```C++
for (uint32_t nextLevel = 1; nextLevel < m_textureMipSizes.size(); ++nextLevel) {
    // Create the bind group specific to this iteration
    initBindGroup(nextLevel);
    computePass.setBindGroup(0, m_bindGroup, 0, nullptr);

    // Get size from the precomputed vector
    uint32_t invocationCountX = m_textureMipSizes[nextLevel].width;
    uint32_t invocationCountY = m_textureMipSizes[nextLevel].height;
    uint32_t workgroupSizePerDim = 8;
    // This ceils invocationCountX / workgroupSizePerDim
    uint32_t workgroupCountX = (invocationCountX + workgroupSizePerDim - 1) / workgroupSizePerDim;
    uint32_t workgroupCountY = (invocationCountY + workgroupSizePerDim - 1) / workgroupSizePerDim;
    computePass.dispatchWorkgroups(workgroupCountX, workgroupCountY, 1);

    // Destroy the bind group
    terminateBindGroup();
}
```

在命令提交后,我们可以将纹理的 MIP 级别保存在单个文件中:

```C++
for (uint32_t nextLevel = 1; nextLevel < m_textureMipSizes.size(); ++nextLevel) {
    // Save the MIP level
    std::filesystem::path path = RESOURCE_DIR "/output.mip" + std::to_string(nextLevel) + ".png";
    saveTexture(path, m_device, m_texture, nextLevel);
}
```

```{figure} /images/mipmap-generation/pyramid.png
:align: center
:class: with-shadow
直接通过GPU计算得出的MIP水平。
```

结论
----------

在本章中,我们看到如何在计算阴影器中玩弄纹理,特别是读写需要不同的束缚,**不得重叠**我们将在下一章继续讨论这个问题。

还记得我们一开始有两个选择A和B吗? 常见的是**多种平行方案似乎是可能的**但实际上其中之一 是一个更好的主意! 这么说**仔细想想**在实施你想到的第一个想法之前

*结果代码 :* [`step211`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step211)
