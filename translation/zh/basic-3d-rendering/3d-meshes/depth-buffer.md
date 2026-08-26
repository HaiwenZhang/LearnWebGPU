深度缓冲区 <span class="bullet">🟡</span>
============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/3d-meshes/depth-buffer.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step052`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step052)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step052-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step052-vanilla)
````

```{figure} /images/pyramid-zissue.png
:align: center
:class: with-shadow
深度有问题。
```

Z 缓冲区算法
----------------------

我们在这个基本示例中面临的问题来自**可见性**的问题。尽管很容易想象，但“这一点是否看到那个点”（即它们之间的线是否与任何几何图形相交）这个问题很难有效回答。

特别是，在生成**片段**时，我们必须弄清楚它对应的3D点是否被视点看到，以便决定是否必须将其混合到输出纹理中。

**Z-Buffer 算法** 是 GPU 渲染管道用来解决可见性问题的算法：

1. **对于每个像素**，它存储已混合到该像素的最后一个片段的深度，或默认值（表示可能的最远深度）。
2. 每次产生新片段时，都会将其**深度**与该值进行比较。如果片段深度大于当前存储的深度，则将其**丢弃**而不进行混合。否则，它会正常混合，并且存储的值会更新为这个新的最接近片段的深度。

因此，在生成的图像中仅可见深度最低的片段。每个像素的深度值存储在称为 **Z 缓冲区** 的特殊 **纹理** 中。这是 Z 缓冲区算法所需的唯一内存开销，使其非常适合实时渲染。

```{topic} About transparency
当片段的**不透明度值既不是 0 也不是 1**（并且使用 alpha 混合）时，**不能保证**只有具有最低深度的片段可见。更糟糕的是；片段的发出顺序会对结果产生影响（因为混合片段 **A** 然后片段 **B** 与混合 **B** 然后 **A** 不同）。

长话短说：**在 Z 缓冲区管道中处理透明对象总是有点棘手**。一个简单的解决方案是限制透明对象的数量，并对它们进行动态排序。他们到视点的距离。存在更高级的方案，例如[与顺序无关的透明度](https://en.wikipedia.org/wiki/Order-independent_transparency)技术。
```

管道状态
--------------

由于此 Z 缓冲区算法是 3D 光栅化管道的关键步骤，因此它被实现为**固定功能**阶段。我们通过 `pipelineDesc.depthStencil` 字段进行配置，到目前为止我们已将其保留为空。

````{tab} With webgpu.hpp
```C++
DepthStencilState depthStencilState = Default;
// Setup depth state
pipelineDesc.depthStencil = &depthStencilState;
```
````

````{tab} Vanilla webgpu.h
```C++
void setDefault(WGPUStencilFaceState &stencilFaceState); {
	stencilFaceState.compare = WGPUCompareFunction_Always;
	stencilFaceState.failOp = WGPUStencilOperation_Keep;
	stencilFaceState.depthFailOp = WGPUStencilOperation_Keep;
	stencilFaceState.passOp = WGPUStencilOperation_Keep;
}

void setDefault(WGPUDepthStencilState &depthStencilState); {
	depthStencilState.format = WGPUTextureFormat_Undefined;
	depthStencilState.depthWriteEnabled = false;
	depthStencilState.depthCompare = WGPUCompareFunction_Always;
	depthStencilState.stencilReadMask = 0xFFFFFFFF;
	depthStencilState.stencilWriteMask = 0xFFFFFFFF;
	depthStencilState.depthBias = 0;
	depthStencilState.depthBiasSlopeScale = 0;
	depthStencilState.depthBiasClamp = 0;
	setDefault(depthStencilState.stencilFront);
	setDefault(depthStencilState.stencilBack);
}

// [...]

WGPUDepthStencilState depthStencilState;
setDefault(depthStencilState);
// Setup depth state
pipelineDesc.depthStencil = &depthStencilState;
```
````

我们可以配置的 Z-Buffer 算法的第一个方面是**比较函数**，用于决定是否应该保留新片段。它默认为 `Always`，这基本上禁用深度测试（片段始终是混合的）。

**最常见的选择**是将其设置为 `Less` ，表示仅当片段的深度**小于** Z 缓冲区的当前值时才会混合片段。

````{tab} With webgpu.hpp
```C++
depthStencilState.depthCompare = CompareFunction::Less;
```
````

````{tab} Vanilla webgpu.h
```C++
depthStencilState.depthCompare = WGPUCompareFunction_Less;
```
````

我们的第二个选择是，一旦片段通过测试，我们是否要**更新 Z 缓冲区的值**。例如，在渲染用户界面元素时或处理透明对象时，禁用此功能可能很有用，但**对于常规用例**，我们确实希望在每次混合片段时​​写入新的深度。

```C++
depthStencilState.depthWriteEnabled = true;
```

最后，我们必须告诉管道 Z 缓冲区的深度值如何在内存中**编码**：

````{tab} With webgpu.hpp
```C++
// Store the format in a variable as later parts of the code depend on it
TextureFormat depthTextureFormat = TextureFormat::Depth24Plus;
depthStencilState.format = depthTextureFormat;
```
````

````{tab} Vanilla webgpu.h
```C++
// Store the format in a variable as later parts of the code depend on it
WGPUTextureFormat depthTextureFormat = WGPUTextureFormat_Depth24Plus;
depthStencilState.format = depthTextureFormat;
```
````

```{important}
深度纹理 **不使用与颜色纹理相同的格式**，它们有自己的一组可能值（全部以 `Depth` 开头）。启用后，相同的纹理用于表示深度和“模板”值，并且总预算为每像素 32 位，因此通常使用 24 位编码的深度并将最后 8 位留给潜在的模板缓冲区。
```

最后，**我们通过告知它不应读取或写入模板缓冲区的任何字节来停用模板**。

```C++
// Deactivate the stencil alltogether
depthStencilState.stencilReadMask = 0;
depthStencilState.stencilWriteMask = 0;
```

深度纹理
-------------

我们必须自己分配 GPU 存储 Z 缓冲区的纹理。我会快速讨论这一部分，因为**我们稍后会回来讨论纹理**。

我们首先创建一个具有交换链纹理大小的纹理，使用 `RenderAttachment` 以及与 `depthStencilState.format` 中声明的格式相匹配的格式。

````{tab} With webgpu.hpp
```C++
// Create the depth texture
TextureDescriptor depthTextureDesc;
depthTextureDesc.dimension = TextureDimension::_2D;
depthTextureDesc.format = depthTextureFormat;
depthTextureDesc.mipLevelCount = 1;
depthTextureDesc.sampleCount = 1;
depthTextureDesc.size = {640, 480, 1};
depthTextureDesc.usage = TextureUsage::RenderAttachment;
depthTextureDesc.viewFormatCount = 1;
depthTextureDesc.viewFormats = (WGPUTextureFormat*)&depthTextureFormat;
Texture depthTexture = device.createTexture(depthTextureDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create the depth texture
WGPUTextureDescriptor depthTextureDesc;
depthTextureDesc.dimension = WGPUTextureDimension_2D;
depthTextureDesc.format = depthTextureFormat;
depthTextureDesc.mipLevelCount = 1;
depthTextureDesc.sampleCount = 1;
depthTextureDesc.size = {640, 480, 1};
depthTextureDesc.usage = WGPUTextureUsage_RenderAttachment;
depthTextureDesc.viewFormatCount = 1;
depthTextureDesc.viewFormats = &depthTextureFormat;
Texture depthTexture = wgpuDeviceCreateTexture(device, depthTextureDesc);
```
````

我们还创建一个**纹理视图**，这是渲染管道所期望的。一般来说，纹理视图表示纹理的子部分，可能以不同的格式公开，但这里我们有一个简单的纹理，视图主要表示整个纹理。只有将 `aspect` 设置为 `DepthOnly` 才会限制视图的范围。

````{tab} With webgpu.hpp
```C++
// Create the view of the depth texture manipulated by the rasterizer
TextureViewDescriptor depthTextureViewDesc;
depthTextureViewDesc.aspect = TextureAspect::DepthOnly;
depthTextureViewDesc.baseArrayLayer = 0;
depthTextureViewDesc.arrayLayerCount = 1;
depthTextureViewDesc.baseMipLevel = 0;
depthTextureViewDesc.mipLevelCount = 1;
depthTextureViewDesc.dimension = TextureViewDimension::_2D;
depthTextureViewDesc.format = depthTextureFormat;
TextureView depthTextureView = depthTexture.createView(depthTextureViewDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create the view of the depth texture manipulated by the rasterizer
WGPUTextureViewDescriptor depthTextureViewDesc;
depthTextureViewDesc.aspect = WGPUTextureAspect_DepthOnly;
depthTextureViewDesc.baseArrayLayer = 0;
depthTextureViewDesc.arrayLayerCount = 1;
depthTextureViewDesc.baseMipLevel = 0;
depthTextureViewDesc.mipLevelCount = 1;
depthTextureViewDesc.dimension = WGPUTextureViewDimension_2D;
depthTextureViewDesc.format = depthTextureFormat;
WGPUTextureView depthTextureView = wgpuTextureCreateView(depthTexture, depthTextureViewDesc);
```
````

与缓冲区一样，纹理在使用后必须被销毁，并且视图和纹理都必须被释放：

````{tab} With webgpu.hpp
```C++
// Destroy the depth texture and its view
depthTextureView.release();
depthTexture.destroy();
depthTexture.release();
```
````

````{tab} Vanilla webgpu.h
```C++
// Destroy the depth texture and its view
wgpuTextureViewRelease(depthTextureView);
wgpuTextureDestroy(depthTexture);
wgpuTextureRelease(depthTexture);
```
````

最后，我们需要更新所需的限制以规定最大纹理大小：

```C++
// For the depth buffer, we enable textures (up to the size of the window):
requiredLimits.limits.maxTextureDimension1D = 480;
requiredLimits.limits.maxTextureDimension2D = 640;
requiredLimits.limits.maxTextureArrayLayers = 1;
```

```{note}
再次强调，稍后将详细介绍纹理和纹理视图！
```

深度附件
----------------

就像附加颜色目标或绑定统一缓冲区时一样，我们定义一个对象来将深度纹理“连接”到渲染管道。这是 `RenderPassDepthStencilAttachment`：

````{tab} With webgpu.hpp
```C++
// We already had a color attachment:
renderPassDesc.colorAttachments = &colorAttachment;

// We now add a depth/stencil attachment:
RenderPassDepthStencilAttachment depthStencilAttachment;
// [...] // Setup depth/stencil attachment
renderPassDesc.depthStencilAttachment = &depthStencilAttachment;
```
````

````{tab} Vanilla webgpu.h
```C++
// We already had a color attachment:
renderPassDesc.colorAttachments = &colorAttachment;

// We now add a depth/stencil attachment:
WGPURenderPassDepthStencilAttachment depthStencilAttachment;
// [...] // Setup depth/stencil attachment
renderPassDesc.depthStencilAttachment = &depthStencilAttachment;
```
````

即使我们不使用模板部分，我们也必须设置清除/存储操作：

````{tab} With webgpu.hpp
```C++
// Setup depth/stencil attachment

// The view of the depth texture
depthStencilAttachment.view = depthTextureView;

// The initial value of the depth buffer, meaning "far"
depthStencilAttachment.depthClearValue = 1.0f;
// Operation settings comparable to the color attachment
depthStencilAttachment.depthLoadOp = LoadOp::Clear;
depthStencilAttachment.depthStoreOp = StoreOp::Store;
// we could turn off writing to the depth buffer globally here
depthStencilAttachment.depthReadOnly = false;

// Stencil setup, mandatory but unused
depthStencilAttachment.stencilClearValue = 0;
depthStencilAttachment.stencilLoadOp = LoadOp::Clear;
depthStencilAttachment.stencilStoreOp = StoreOp::Store;
depthStencilAttachment.stencilReadOnly = true;
```
````

````{tab} Vanilla webgpu.h
```C++
// Setup depth/stencil attachment

// The view of the depth texture
depthStencilAttachment.view = depthTextureView;

// The initial value of the depth buffer, meaning "far"
depthStencilAttachment.depthClearValue = 1.0f;
// Operation settings comparable to the color attachment
depthStencilAttachment.depthLoadOp = WGPULoadOp_Clear;
depthStencilAttachment.depthStoreOp = WGPUStoreOp_Store;
// we could turn off writing to the depth buffer globally here
depthStencilAttachment.depthReadOnly = false;

// Stencil setup, mandatory but unused
depthStencilAttachment.stencilClearValue = 0;
depthStencilAttachment.stencilLoadOp = WGPULoadOp_Clear;
depthStencilAttachment.stencilStoreOp = WGPUStoreOp_Store;
depthStencilAttachment.stencilReadOnly = true;
```
````

````{admonition} Dawn
当使用 WebGPU 的 Dawn 实现时，`stencilLoadOp` 和 `stencilStoreOp` 必须分别设置为 `LoadOp::Undefined` 和 `StoreOp::Undefined` 。

此外， `depthStencilAttachment` 的 `clearDepth` 属性必须转换为 NaN （这是向后兼容的事情）：

```C++
constexpr auto NaNf = std::numeric_limits<float>::quiet_NaN();
depthStencilAttachment.clearDepth = NaNf;
```
````

着色器
------

我们需要做的最后一件事是为每个片段设置深度，我们可以通过**顶点着色器**来完成（并且光栅化器将为每个片段插入它）：

```rust
out.position = vec4f(position.x,position.y *ratio, /* 在此设置深度 */ 1.0);
```

深度值必须在 $(0,1)$ 范围内。我们将在下一章中建立一个正确的方法来定义它，但现在让我们简单地将 `position.z` 从其范围 $(-1,1)$ 重新映射到 $(0,1)$：

```rust
out.position = vec4f(position.x,position.y * 比例,position.z * 0.5 + 0.5, 1.0);
```

结论
----------

我们现在解决了深度问题，并设置了 3D 渲染管道的重要部分，我们无需进行太多编辑。

```{figure} /images/pyramid-zissue-fixed.png
:align: center
:class: with-shadow
深度排序问题消失了！
```

````{tab} With webgpu.hpp
*结果代码：* [`step052`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step052)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step052-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step052-vanilla)
````
