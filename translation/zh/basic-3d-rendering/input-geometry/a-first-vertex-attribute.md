第一个顶点属性 <span class="bullet">🟢</span>
========================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/input-geometry/a-first-vertex-attribute.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/032 - A first Vertex Attribute - vanilla
:parent: 030 - Hello Triangle - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/032 - A first Vertex Attribute
:parent: 030 - Hello Triangle
```

````{tab} With webgpu.hpp
*结果代码：* [`step032`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step032)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step032-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step032-vanilla)
````

在 [Hello Triangle](../hello-triangle.md) 章节中，我们直接在着色器中对 3 个顶点位置进行硬编码，但这显然不能很好地扩展。在本章中，我们将看到将顶点属性作为**输入到*顶点着色器***的**正确方法**。

顶点着色器输入
-------------------

请记住，我们**不**控制 `vs_main` 函数的调用方式，而是渲染管道的**固定部分**。但是，我们可以通过使用 **WGSL 属性**标记函数的参数来**请求**一些输入数据。

```{important}
术语“属性”用于两个不同的事物。我们所说的 **WGSL 属性** 是指 WGSL 代码中 `@something` 形式的标记，而 **顶点属性** 是指顶点着色器的输入。
```

实际上，我们已经在顶点着色器输入中使用了 WGSL 属性，即 `@builtin(vertex_index)` 属性：

```rust
@顶点
fn vs_main(@builtin(vertex_index) in_vertex_index: u32) -> /* [...] */ {
// ^^^^^^^^^^^^^^^^^^^^^^^^ 这是一个 WGSL 属性
	// [...]
}
```

这意味着参数 `in_vertex_index` 必须由顶点获取阶段填充当前顶点的索引。

```{note}
**内置**的属性（例如 `vertex_index`）列在 [WGSL 文档](https://gpuweb.github.io/gpuweb/wgsl/#builtin-values) 中。
```

我们可以创建自己的输入，而不是使用内置输入。为此，我们需要：

1. 创建一个**缓冲区**来存储每个顶点的输入值；当然，这些数据必须存储在**GPU 端**。
2. 告诉渲染管道在获取每个顶点的条目时**如何解释**原始缓冲区数据。这是顶点缓冲区**布局**。
3. 在绘制调用之前在渲染通道中设置顶点缓冲区。

在着色器方面，我们用新参数替换顶点索引参数：

```rust
@顶点
fn vs_main(@location(0) in_vertex_position: vec2f) -> /* [...] */ {
// ^^^^^^^^^^^^^ 这是一个 WGSL 属性
	// [...]
}
```

`@location(0)` 属性意味着该输入变量由 `pipelineDesc.vertex.buffers` 数组中的第一个（索引“0”）**顶点属性**描述。类型 `vec2f` 必须符合我们将在 **布局** 中声明的内容。参数名称 `in_vertex_position` 由您决定，它仅是着色器代码的本地参数！

顶点着色器最终变得非常简单：

```{lit} rust, Vertex shader (also for tangle root "Vanilla")
fn vs_main(@location(0) in_vertex_position: vec2f) -> @builtin(position) vec4f {
	return vec4f(in_vertex_position, 0.0, 1.0);
}
```

```{lit} rust, Shader source literal (hidden, replace, also for tangle root "Vanilla")
const char* shaderSource = R"(
@vertex
{{Vertex shader}}

@fragment
{{Fragment shader}}
)";
```

```{lit} rust, Fragment shader (hidden, also for tangle root "Vanilla")
fn fs_main() -> @location(0) vec4f {
	return vec4f(0.0, 0.4, 1.0, 1.0);
}
```

设备能力
-------------------

### 检查能力

> 🤓 嘿，位置属性的**最大数量**是多少？

很高兴你问了！如果我们不指定任何内容，我们的设备可用的顶点属性数量可能会有所不同。我们可以按如下方式检查：

````{tab} With webgpu.hpp
```C++
SupportedLimits supportedLimits;

adapter.getLimits(&supportedLimits);
std::cout << "adapter.maxVertexAttributes: " << supportedLimits.limits.maxVertexAttributes << std::endl;

device.getLimits(&supportedLimits);
std::cout << "device.maxVertexAttributes: " << supportedLimits.limits.maxVertexAttributes << std::endl;

// Personally I get:
//   adapter.maxVertexAttributes: 32
//   device.maxVertexAttributes: 16
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUSupportedLimits supportedLimits{};
supportedLimits.nextInChain = nullptr;

wgpuAdapterGetLimits(adapter, &supportedLimits);
std::cout << "adapter.maxVertexAttributes: " << supportedLimits.limits.maxVertexAttributes << std::endl;

wgpuDeviceGetLimits(device, &supportedLimits);
std::cout << "device.maxVertexAttributes: " << supportedLimits.limits.maxVertexAttributes << std::endl;

// Personally I get:
//   adapter.maxVertexAttributes: 32
//   device.maxVertexAttributes: 16
```
````

WebGPU 提供的 [适配器 + device](../../getting-started/adapter-and-device/index.md) 抽象的精神是首先检查适配器是否具有我们需要的功能，然后我们**要求**在设备创建过程中需要的最小限制，如果创建成功，我们**保证**具有我们要求的限制。

我们得到的只是所需的内容，因此，如果我们在使用更多顶点缓冲区时忘记更新初始检查，程序就会失败。通过这个**好的实践**，我们限制了*“它对我有用”*的情况，即程序在您的设备上正确运行，但在其他人的设备上运行不正确，这很快就会成为一场噩梦。

### 需要能力

此初始检查是通过在设备描述符中指定非空 `requiredLimits` 指针来完成的。我建议我们创建一个方法 `GetRequiredLimits(Adapter adapter)` 专门用于设置我们的应用程序要求：

````{tab} With webgpu.hpp
```{lit} C++, Private methods (append)
// In Application class
private:
	RequiredLimits GetRequiredLimits(Adapter adapter) const;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Private methods (append, for tangle root "Vanilla")
// In Application class
private:
	WGPURequiredLimits GetRequiredLimits(WGPUAdapter adapter) const;
```
````

然后我们在构建**设备描述符**时调用此方法：

````{tab} With webgpu.hpp
```{lit} C++, Build device descriptor (append)
// Before adapter.requestDevice(deviceDesc)
RequiredLimits requiredLimits = GetRequiredLimits(adapter);
deviceDesc.requiredLimits = &requiredLimits;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Build device descriptor (append, for tangle root "Vanilla")
// Before requestDeviceSync(adapter, &deviceDesc)
WGPURequiredLimits requiredLimits = GetRequiredLimits(adapter);
deviceDesc.requiredLimits = &requiredLimits;
```
````

所需的限制遵循与支持的限制**相同的结构**，我们可以为我们的顶点属性自定义其中一些限制：

````{tab} With webgpu.hpp
```{lit} C++, GetRequiredLimits method
RequiredLimits Application::GetRequiredLimits(Adapter adapter) const {
	// Get adapter supported limits, in case we need them
	SupportedLimits supportedLimits;
	adapter.getLimits(&supportedLimits);

	// Don't forget to = Default
	RequiredLimits requiredLimits = Default;

	// We use at most 1 vertex attribute for now
	requiredLimits.limits.maxVertexAttributes = 1;
	// We should also tell that we use 1 vertex buffers
	requiredLimits.limits.maxVertexBuffers = 1;
	// Maximum size of a buffer is 6 vertices of 2 float each
	requiredLimits.limits.maxBufferSize = 6 * 2 * sizeof(float);
	// Maximum stride between 2 consecutive vertices in the vertex buffer
	requiredLimits.limits.maxVertexBufferArrayStride = 2 * sizeof(float);

	{{Other device limits}}

	return requiredLimits;
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, GetRequiredLimits method (for tangle root "Vanilla")
// If you do not use webgpu.hpp, I suggest you create a function to init the
// WGPULimits structure:
void setDefault(WGPULimits &limits) {
	limits.maxTextureDimension1D = WGPU_LIMIT_U32_UNDEFINED;
	limits.maxTextureDimension2D = WGPU_LIMIT_U32_UNDEFINED;
	limits.maxTextureDimension3D = WGPU_LIMIT_U32_UNDEFINED;
	{{Set everything to WGPU_LIMIT_U32_UNDEFINED or WGPU_LIMIT_U64_UNDEFINED to mean no limit}}
}

WGPURequiredLimits Application::GetRequiredLimits(WGPUAdapter adapter) const {
	// Get adapter supported limits, in case we need them
	WGPUSupportedLimits supportedLimits;
	supportedLimits.nextInChain = nullptr;
	wgpuAdapterGetLimits(adapter, &supportedLimits);

	WGPURequiredLimits requiredLimits{};
	setDefault(requiredLimits.limits);

	// We use at most 1 vertex attribute for now
	requiredLimits.limits.maxVertexAttributes = 1;
	// We should also tell that we use 1 vertex buffers
	requiredLimits.limits.maxVertexBuffers = 1;
	// Maximum size of a buffer is 6 vertices of 2 float each
	requiredLimits.limits.maxBufferSize = 6 * 2 * sizeof(float);
	// Maximum stride between 2 consecutive vertices in the vertex buffer
	requiredLimits.limits.maxVertexBufferArrayStride = 2 * sizeof(float);

	{{Other device limits}}

	return requiredLimits;
}
```

```{lit} C++, Set everything to WGPU_LIMIT_U32_UNDEFINED or WGPU_LIMIT_U64_UNDEFINED to mean no limit (hidden, for tangle root "Vanilla")
limits.maxTextureArrayLayers = WGPU_LIMIT_U32_UNDEFINED;
limits.maxBindGroups = WGPU_LIMIT_U32_UNDEFINED;
limits.maxBindGroupsPlusVertexBuffers = WGPU_LIMIT_U32_UNDEFINED;
limits.maxBindingsPerBindGroup = WGPU_LIMIT_U32_UNDEFINED;
limits.maxDynamicUniformBuffersPerPipelineLayout = WGPU_LIMIT_U32_UNDEFINED;
limits.maxDynamicStorageBuffersPerPipelineLayout = WGPU_LIMIT_U32_UNDEFINED;
limits.maxSampledTexturesPerShaderStage = WGPU_LIMIT_U32_UNDEFINED;
limits.maxSamplersPerShaderStage = WGPU_LIMIT_U32_UNDEFINED;
limits.maxStorageBuffersPerShaderStage = WGPU_LIMIT_U32_UNDEFINED;
limits.maxStorageTexturesPerShaderStage = WGPU_LIMIT_U32_UNDEFINED;
limits.maxUniformBuffersPerShaderStage = WGPU_LIMIT_U32_UNDEFINED;
limits.maxUniformBufferBindingSize = WGPU_LIMIT_U64_UNDEFINED;
limits.maxStorageBufferBindingSize = WGPU_LIMIT_U64_UNDEFINED;
limits.minUniformBufferOffsetAlignment = WGPU_LIMIT_U32_UNDEFINED;
limits.minStorageBufferOffsetAlignment = WGPU_LIMIT_U32_UNDEFINED;
limits.maxVertexBuffers = WGPU_LIMIT_U32_UNDEFINED;
limits.maxBufferSize = WGPU_LIMIT_U64_UNDEFINED;
limits.maxVertexAttributes = WGPU_LIMIT_U32_UNDEFINED;
limits.maxVertexBufferArrayStride = WGPU_LIMIT_U32_UNDEFINED;
limits.maxInterStageShaderComponents = WGPU_LIMIT_U32_UNDEFINED;
limits.maxInterStageShaderVariables = WGPU_LIMIT_U32_UNDEFINED;
limits.maxColorAttachments = WGPU_LIMIT_U32_UNDEFINED;
limits.maxColorAttachmentBytesPerSample = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeWorkgroupStorageSize = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeInvocationsPerWorkgroup = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeWorkgroupSizeX = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeWorkgroupSizeY = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeWorkgroupSizeZ = WGPU_LIMIT_U32_UNDEFINED;
limits.maxComputeWorkgroupsPerDimension = WGPU_LIMIT_U32_UNDEFINED;
```
````

```{lit} C++, Application implementation (hidden, append, also for tangle root "Vanilla")
{{GetRequiredLimits method}}
```

```{important}
请注意我如何使用上面的 `= Default` **初始化所需的限制**对象。这是 `webgpu.hpp` 包装器为所有结构提供的语法帮助器，以防止我们手动设置默认值。在这种情况下，它将所有限制设置为 `WGPU_LIMIT_U32_UNDEFINED` 或 `WGPU_LIMIT_U64_UNDEFINED` ，表示没有要求。
```

我现在得到了这些更安全的支持限制：

```
适配器.maxVertexAttributes：32
设备.maxVertexBuffers：1
```

我建议您查看 `webgpu.h` 中 `WGPULimits` 结构的所有字段，以便您知道何时向所需的限制添加某些内容。

````{note}
即使设置为 `WGPU_LIMIT_U32_UNDEFINED`，也有两个限制可能会导致问题，即设置**最小值**而不是最大值的限制。适配器可能不支持默认的“未定义”值，因此**仅在这种情况下**我建议我们转发**支持的限制**中的值：

```{lit} C++, Other device limits (also for tangle root "Vanilla")
// These two limits are different because they are "minimum" limits,
// they are the only ones we may forward from the adapter's supported
// limits.
requiredLimits.limits.minUniformBufferOffsetAlignment = supportedLimits.limits.minUniformBufferOffsetAlignment;
requiredLimits.limits.minStorageBufferOffsetAlignment = supportedLimits.limits.minStorageBufferOffsetAlignment;
```
````

### 默认功能

官方默认功能可以在【规范](https://www.w3.org/TR/webgpu/#limit-default)中找到。这还提到：

> 每个适配器都保证支持默认值或更好的值。 （[来源](https://www.w3.org/TR/webgpu/#limit-default)）

**在本指南中**，我会更严格地指定所需的功能，原因有两个：

- 确保我们在介绍各种概念时展示与它们相关的功能。
- 因为可能存在“**兼容性**”适配器，这些适配器在网络中不被视为有效，但仍以本机模式公开。

顶点缓冲区
-------------

回到我们的顶点位置属性：现在我们在 C++ 源代码中对顶点缓冲区的**值进行硬编码：

```{lit} C++, Define vertex data (also for tangle root "Vanilla")
// Vertex buffer data
// There are 2 floats per vertex, one for x and one for y.
// But in the end this is just a bunch of floats to the eyes of the GPU,
// the *layout* will tell how to interpret this.
std::vector<float> vertexData = {
	// x0, y0
	-0.5, -0.5,

	// x1, y1
	+0.5, -0.5,

	// x2, y2
	+0.0, +0.5
};

// We will declare vertexCount as a member of the Application class
vertexCount = static_cast<uint32_t>(vertexData.size() / 2);
```

GPU 端顶点缓冲区的创建**像任何其他缓冲区**一样，如上一章所述。主要区别在于我们必须在其 `usage` 字段中指定 `BufferUsage::Vertex` 。

````{tab} With webgpu.hpp
```{lit} C++, Create vertex buffer
// Create vertex buffer
BufferDescriptor bufferDesc;
bufferDesc.size = vertexData.size() * sizeof(float);
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Vertex; // Vertex usage here!
bufferDesc.mappedAtCreation = false;
vertexBuffer = device.createBuffer(bufferDesc);

// Upload geometry data to the buffer
queue.writeBuffer(vertexBuffer, 0, vertexData.data(), bufferDesc.size);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create vertex buffer (for tangle root "Vanilla")
// Create vertex buffer
WGPUBufferDescriptor bufferDesc{};
bufferDesc.nextInChain = nullptr;
bufferDesc.size = vertexData.size() * sizeof(float);
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Vertex; // Vertex usage here!
bufferDesc.mappedAtCreation = false;
vertexBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);

// Upload geometry data to the buffer
wgpuQueueWriteBuffer(queue, vertexBuffer, 0, vertexData.data(), bufferDesc.size);
```
````

我们将 `vertexBuffer` 和 `vertexCount` 声明为 `Application` 类的成员，并将此初始化放在专用的 `InitializeBuffers()` 方法中：

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	Buffer vertexBuffer;
	uint32_t vertexCount;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUBuffer vertexBuffer;
	uint32_t vertexCount;
```
````

```{lit} C++, Private methods (append, also for tangle root "Vanilla")
private: // Application methods
	void InitializeBuffers();
```

该方法定义CPU端顶点数据，然后创建GPU端缓冲区，如上所述：

```{lit} C++, InitializeBuffers method (also for tangle root "Vanilla")
void Application::InitializeBuffers() {
	{{Define vertex data}}
	{{Create vertex buffer}}
}
```

```{lit} C++, Application implementation (hidden, append, also for tangle root "Vanilla")
{{InitializeBuffers method}}
```

````{note}
不要忘记在 `Initialize()` 末尾调用它：

```{lit} C++, Initialize (append, also for tangle root "Vanilla")
// At the end of Initialize()
InitializeBuffers();
```
````

当然，当应用程序终止时，我们会释放该缓冲区：

```{lit} C++, Terminate (prepend)
// At the beginning of Terminate()
vertexBuffer.release();
```

顶点缓冲区布局
--------------------

为了让 *vertex fetch* 阶段将来自顶点缓冲区的 **原始数据** 转换为顶点着色器期望的**，我们需要将 `VertexBufferLayout` 指定为 `pipelineDesc.vertex.buffers`：

````{tab} With webgpu.hpp
```{lit} C++, Describe vertex buffers (replace)
// Vertex fetch
VertexBufferLayout vertexBufferLayout;
{{Describe the vertex buffer layout}}

pipelineDesc.vertex.bufferCount = 1;
pipelineDesc.vertex.buffers = &vertexBufferLayout;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe vertex buffers (replace, for tangle root "Vanilla")
// Vertex fetch
WGPUVertexBufferLayout vertexBufferLayout{};
{{Describe the vertex buffer layout}}

pipelineDesc.vertex.bufferCount = 1;
pipelineDesc.vertex.buffers = &vertexBufferLayout;
```
````

请注意，给定的渲染管道**可能使用多个**顶点缓冲区。另一方面，**相同的顶点缓冲区**可以包含**多个顶点属性**。

```{note}
这就是为什么 `maxVertexAttributes` 和 `maxVertexBuffers` 限制是不同的概念。
```

因此，在我们的顶点缓冲区布局中，我们指定它包含**多少个属性**并**详细说明每个属性**。在我们的例子中，只有位置属性：


````{tab} With webgpu.hpp
```{lit} C++, Describe the vertex buffer layout
VertexAttribute positionAttrib;
{{Describe the position attribute}}

vertexBufferLayout.attributeCount = 1;
vertexBufferLayout.attributes = &positionAttrib;

{{Describe buffer stride and step mode}}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the vertex buffer layout (for tangle root "Vanilla")
WGPUVertexAttribute positionAttrib;
{{Describe the position attribute}}

vertexBufferLayout.attributeCount = 1;
vertexBufferLayout.attributes = &positionAttrib;

{{Describe buffer stride and step mode}}
```
````

### 对于每个属性

我们现在可以配置顶点属性：

- `shaderLocation` 的值必须与顶点着色器中指定 **WGSL 属性** `@location(...)` 的值**相同**。
- **格式** `Float32x2` 同时对应于着色器中的类型 `vec2f` 以及顶点缓冲区数据中的 2 个浮点序列。
- **offset** 告诉位置值序列在原始顶点缓冲区中的起始位置。在我们的例子中，它从头开始（偏移量 0）。当同一个顶点缓冲区**包含多个属性**时，它是**非空**。

````{tab} With webgpu.hpp
```{lit} C++, Describe the position attribute
// == For each attribute, describe its layout, i.e., how to interpret the raw data ==
// Corresponds to @location(...)
positionAttrib.shaderLocation = 0;
// Means vec2f in the shader
positionAttrib.format = VertexFormat::Float32x2;
// Index of the first element
positionAttrib.offset = 0;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the position attribute (for tangle root "Vanilla")
// == For each attribute, describe its layout, i.e., how to interpret the raw data ==
// Corresponds to @location(...)
positionAttrib.shaderLocation = 0;
// Means vec2f in the shader
positionAttrib.format = WGPUVertexFormat_Float32x2;
// Index of the first element
positionAttrib.offset = 0;
```
````

### 对于整个顶点缓冲区

来自同一顶点缓冲区的属性**共享一些属性**：

- **跨度**是缓冲区操作中的常见概念：它指定两个连续元素之间的字节数。在我们的例子中，位置是**连续的**，因此步幅等于 `vec2f` 的大小，但是如果属性是**交错的**，那么当在同一缓冲区中添加更多属性时，这会发生变化。

- 最后，`stepMode` 设置为 `Vertex`，表示缓冲区中的**每个值**对应于**不同的顶点**。当每个值被形状的同一实例（即副本）的所有顶点共享时，步骤模式设置为 `Instance` （我们稍后将看到实例化）。

````{tab} With webgpu.hpp
```{lit} C++, Describe buffer stride and step mode
// == Common to attributes from the same buffer ==
vertexBufferLayout.arrayStride = 2 * sizeof(float);
vertexBufferLayout.stepMode = VertexStepMode::Vertex;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe buffer stride and step mode (for tangle root "Vanilla")
// == Common to attributes from the same buffer ==
vertexBufferLayout.arrayStride = 2 * sizeof(float);
vertexBufferLayout.stepMode = WGPUVertexStepMode_Vertex;
```
````

渲染通道
-----------

我们需要应用的最后一个更改是在对渲染通道进行编码时将顶点缓冲区“连接”到管道的顶点缓冲区布局：

````{tab} With webgpu.hpp
```{lit} C++, Draw a triangle (replace)
renderPass.setPipeline(pipeline);

// Set vertex buffer while encoding the render pass
renderPass.setVertexBuffer(0, vertexBuffer, 0, vertexBuffer.getSize());

// We use the `vertexCount` variable instead of hard-coding the vertex count
renderPass.draw(vertexCount, 1, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Draw a triangle (replace, for tangle root "Vanilla")
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);

// Set vertex buffer while encoding the render pass
wgpuRenderPassEncoderSetVertexBuffer(renderPass, 0, vertexBuffer, 0, wgpuBufferGetSize(vertexBuffer));

// We use the `vertexCount` variable instead of hard-coding the vertex count
wgpuRenderPassEncoderDraw(renderPass, vertexCount, 1, 0, 0);
```
````

我们得到...与之前完全相同的三角形。但现在我们可以非常**轻松地添加一些几何图形**：

```{lit} C++, Define vertex data (replace, also for tangle root "Vanilla")
// Vertex buffer data
// There are 2 floats per vertex, one for x and one for y.
std::vector<float> vertexData = {
	// Define a first triangle:
	-0.5, -0.5,
	+0.5, -0.5,
	+0.0, +0.5,

	// Add a second triangle:
	-0.55f, -0.5,
	-0.05f, +0.5,
	-0.55f, +0.5
};
```

结论
----------

```{figure} /images/two-triangles.png
:align: center
:class: with-shadow
使用顶点属性渲染的三角形
```

我们在本章中了解了如何使用**GPU缓冲区**将数据作为**输入**提供给顶点着色器，从而提供给整个光栅化管道。我们将在下一章中通过添加**附加属性**来完善这一点。

````{tab} With webgpu.hpp
*结果代码：* [`step032`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step032)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step032-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step032-vanilla)
````


