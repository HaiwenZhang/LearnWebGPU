多个属性 <span class="bullet">🟢</span>
===================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/input-geometry/multiple-attributes.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/033 - Multiple Attributes - Option A - vanilla
:parent: 032 - A first Vertex Attribute - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/033 - Multiple Attributes - Option A
:parent: 032 - A first Vertex Attribute
```

````{tab} With webgpu.hpp
*结果代码：* [`step033`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step033)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step033-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step033-vanilla)
````

顶点可以包含多个位置属性。一个典型的例子是给每个顶点**添加颜色属性**。这还将向我们展示光栅化器如何自动在三角形之间插入顶点属性。

着色器
------

### 顶点输入结构

您可能已经猜到，我们可以简单地将 **第二个参数** 添加到顶点着色器入口点 `vs_main`，并使用不同的 `@location` WGSL 属性：

```rust
// 我们可以这样做，但是可能有很多属性
@顶点
fn vs_main(@location(0) in_position: vec2f, @location(1) in_color: vec3f) -> /* ... */ {
	// [...]
}
```

这是可行的，但是当**输入属性的数量增加**时，我们更愿意采用**单个参数**，其类型是带有位置标记的**自定义结构**：

```{lit} rust, Define VertexInput struct (also for tangle root "Vanilla")
/**
 * A structure with fields labeled with vertex attribute locations can be used
 * as input to the entry point of a shader.
 */
struct VertexInput {
	@location(0) position: vec2f,
	@location(1) color: vec3f,
};
```

因此，我们的顶点着色器仅接收一个参数，其类型为 `VertexInput`：

```{lit} rust, Vertex shader (also for tangle root "Vanilla")
fn vs_main(in: VertexInput) -> /* ... */ {
	{{Vertex shader body}}
}
```

```{lit} C++, Shader source literal (hidden, replace, also for tangle root "Vanilla")
const char* shaderSource = R"(
{{Shader source}}
)";
```

```{lit} rust, Shader source (hidden, also for tangle root "Vanilla")
{{Shader prelude}}

@vertex
{{Vertex shader}}

@fragment
{{Fragment shader}}
```

```{lit} rust, Shader prelude (hidden, also for tangle root "Vanilla")
{{Define VertexInput struct}}
```

### 将顶点属性转发到片段

> 😐但是我不需要顶点着色器中的颜色，我想要片段着色器中的颜色。那么我可以做`fn fs_main(@location(1) color: vec3f)`吗？

不。顶点属性仅提供给顶点着色器。然而，**片段着色器可以接收顶点着色器返回的任何内容！**这就是基于结构的方法变得方便的地方。

首先，我们再次更改 `vs_main` 的签名以**返回自定义结构体**（而不是 `@builtin(position) vec4f`）：

```{lit} rust, Vertex shader (replace, also for tangle root "Vanilla")
fn vs_main(in: VertexInput) -> VertexOutput {
	//                         ^^^^^^^^^^^^ We return a custom struct
	{{Vertex shader body}}
}
```

然后我们需要定义这个结构体。我们当然需要光栅化器所需的**强制** `@builtin(position)` 属性来知道在屏幕上的何处绘制几何图形。我们还添加一个**自定义顶点着色器输出**，我们将其命名为 `color` 并关联到位置 0。

```{lit} rust, Define VertexOutput struct (also for tangle root "Vanilla")
/**
 * A structure with fields labeled with builtins and locations can also be used
 * as *output* of the vertex shader, which is also the input of the fragment
 * shader.
 */
struct VertexOutput {
	@builtin(position) position: vec4f,
	// The location here does not refer to a vertex attribute, it just means
	// that this field must be handled by the rasterizer.
	// (It can also refer to another field of another struct that would be used
	// as input to the fragment shader.)
	@location(0) color: vec3f,
};
```

```{lit} rust, Shader prelude (hidden, append, also for tangle root "Vanilla")
{{Define VertexOutput struct}}
```

现在我们可以**更改片段着色器入口点`fs_main`的参数**。与顶点着色器一样，我们可以直接标记参数：

```rust
// 我们可以直接标记参数：
fn fs_main(@location(0) 颜色: vec3f) -> @location(0) vec4f {
// ^^^^^^^^^^^^^^^^^^^^^^^^^^^ 一个新参数，带有位置 WGSL 属性
{{Fragment shader body}}
}
```

或者，我们可以使用一个自定义结构，其字段被标记...就像 `VertexOutput` 本身！只要我们在 `@location` 索引方面保持一致，它就可能是不同的。

```{lit} rust, Fragment shader (replace, also for tangle root "Vanilla")
// Or we can use a custom struct whose fields are labeled
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
	//     ^^^^^^^^^^^^^^^^ Use for instance the same struct as what the vertex outputs
	{{Fragment shader body}}
}
```

现在剩下的就是**连接点**并从顶点着色器返回片段着色器所需的颜色：

```rust
@顶点
fn vs_main(in: VertexInput) -> VertexOutput {
var out：顶点输出； // 创建输出结构体
out.position = vec4f(in.position, 0.0, 1.0); // 和我们之前直接返回的一样
输出颜色=输入颜色； // 将颜色属性转发给片段着色器
返回；
}

@片段
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
返回 vec4f(in.color, 1.0); // 使用来自顶点着色器的插值颜色
}
```

```{lit} rust, Vertex shader body (hidden, also for tangle root "Vanilla")
// In vs_main()
var out: VertexOutput; // create the output struct
out.position = vec4f(in.position, 0.0, 1.0); // same as what we used to directly return
out.color = in.color; // forward the color attribute to the fragment shader
return out;
```

```{lit} rust, Fragment shader body (hidden, also for tangle root "Vanilla")
// In fs_main()
return vec4f(in.color, 1.0); // use the interpolated color coming from the vertex shader
```

### 能力

可以从顶点着色器转发到片段着色器的组件数量**有限制。在我们的例子中，我们要求 3 个（浮动）组件：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
// There is a maximum of 3 float forwarded from vertex to fragment shader
requiredLimits.limits.maxInterStageShaderComponents = 3;
```

顶点缓冲区布局
--------------------

我们在上一节中定义了如何在着色器中处理**新颜色属性**，但到目前为止，我们还没有**为该属性**提供**任何新的**数据**。

有**不同的方式**将多个属性提供给顶点获取阶段。选择通常取决于输入数据的组织方式，这随上下文而变化，因此我将介绍两种不同的方式。

````{admonition} Device limits
在进行任何操作之前，不要忘记**增加设备的顶点属性限制**：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
requiredLimits.limits.maxVertexAttributes = 2;
//                                          ^ This was 1
```
````

### 选项 A：交错属性

```{image} /images/vertex-buffer/interleaved-attributes-light.svg
:align: center
:class: only-light
```

```{image} /images/vertex-buffer/interleaved-attributes-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>两个属性都在同一个缓冲区中，同一顶点的所有属性都分组在一起。两个连续 x 值之间的 <strong>byte 距离 </strong> 称为 <strong>stride</strong>.</em></span>
</p>

#### 顶点数据

交错属性意味着我们将第一个顶点的**所有属性**的值放入**单个缓冲区**中，然后放入第二个顶点的所有值，依此类推：

```{lit} C++, Define vertex data (replace, also for tangle root "Vanilla")
std::vector<float> vertexData = {
	// x0,  y0,  r0,  g0,  b0
	-0.5, -0.5, 1.0, 0.0, 0.0,

	// x1,  y1,  r1,  g1,  b1
	+0.5, -0.5, 0.0, 1.0, 0.0,

	// ...
	+0.0,   +0.5, 0.0, 0.0, 1.0,
	-0.55f, -0.5, 1.0, 1.0, 0.0,
	-0.05f, +0.5, 1.0, 0.0, 1.0,
	-0.55f, +0.5, 0.0, 1.0, 1.0
};

// We now divide the vector size by 5 fields.
vertexCount = static_cast<uint32_t>(vertexData.size() / 5);
```

#### 布局和属性

仍然是一个缓冲区，但 `vertexBufferLayout.attributes` 数组中有 2 个元素。因此，我们不传递单个条目的地址 `&positionAttrib`，而是使用 `std::vector`：

````{tab} With webgpu.hpp
```{lit} C++, Describe the vertex buffer layout (replace)
// We now have 2 attributes
std::vector<VertexAttribute> vertexAttribs(2);

{{Describe the position attribute}}
{{Describe the color attribute}}

vertexBufferLayout.attributeCount = static_cast<uint32_t>(vertexAttribs.size());
vertexBufferLayout.attributes = vertexAttribs.data();

{{Describe buffer stride and step mode}}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the vertex buffer layout (replace, for tangle root "Vanilla")
// We now have 2 attributes
std::vector<WGPUVertexAttribute> vertexAttribs(2);

{{Describe the position attribute}}
{{Describe the color attribute}}

vertexBufferLayout.attributeCount = static_cast<uint32_t>(vertexAttribs.size());
vertexBufferLayout.attributes = vertexAttribs.data();

{{Describe buffer stride and step mode}}
```
````

首先要注意的是，现在位置属性 $(x,y)$ 的 **字节跨度** 已从 `2 * sizeof(float)` 更改为 `5 * sizeof(float)`：

````{tab} With webgpu.hpp
```{lit} C++, Describe buffer stride and step mode (replace)
vertexBufferLayout.arrayStride = 5 * sizeof(float);
//                               ^^^^^^^^^^^^^^^^^ The new stride
vertexBufferLayout.stepMode = VertexStepMode::Vertex;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe buffer stride and step mode (replace, for tangle root "Vanilla")
vertexBufferLayout.arrayStride = 5 * sizeof(float);
//                               ^^^^^^^^^^^^^^^^^ The new stride
vertexBufferLayout.stepMode = WGPUVertexStepMode_Vertex;
```
````

````{admonition} Device limits
因此，我们需要更新**缓冲区大小和步幅限制**：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
requiredLimits.limits.maxBufferSize = 6 * 5 * sizeof(float);
requiredLimits.limits.maxVertexBufferArrayStride = 5 * sizeof(float);
```
````

这个步长对于两个属性来说都是相同的，因为从 $x_1$ 跳转到 $x_2$ 与从 $r_1$ 跳转到 $r_2$ 的距离相同。因此，在整个**缓冲区布局**的级别设置步幅并不是问题。

我们的两个属性之间的主要区别实际上是它们在缓冲区中开始的**字节偏移量**。该位置仍然从缓冲区的开头开始，即偏移量 0 处：

````{tab} With webgpu.hpp
```{lit} C++, Describe the position attribute (replace)
// Describe the position attribute
vertexAttribs[0].shaderLocation = 0; // @location(0)
vertexAttribs[0].format = VertexFormat::Float32x2;
vertexAttribs[0].offset = 0;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the position attribute (replace, for tangle root "Vanilla")
// Describe the position attribute
vertexAttribs[0].shaderLocation = 0; // @location(0)
vertexAttribs[0].format = WGPUVertexFormat_Float32x2;
vertexAttribs[0].offset = 0;
```
````

颜色在 2 个浮点数 $x$ 和 $y$ 之后开始：

````{tab} With webgpu.hpp
```{lit} C++, Describe the color attribute
// Describe the color attribute
vertexAttribs[1].shaderLocation = 1; // @location(1)
vertexAttribs[1].format = VertexFormat::Float32x3; // different type!
vertexAttribs[1].offset = 2 * sizeof(float); // non null offset!
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the color attribute (for tangle root "Vanilla")
// Describe the color attribute
vertexAttribs[1].shaderLocation = 1; // @location(1)
vertexAttribs[1].format = WGPUVertexFormat_Float32x3; // different type!
vertexAttribs[1].offset = 2 * sizeof(float); // non null offset!
```
````

### 选项 B：多个缓冲区

```{lit-setup}
:tangle-root: zh/033 - Multiple Attributes - Option B - vanilla
:parent: 033 - Multiple Attributes - Option A - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/033 - Multiple Attributes - Option B
:parent: 033 - Multiple Attributes - Option A
```

```{image} /images/vertex-buffer/multiple-buffers-light.svg
:align: center
:class: only-light
```

```{image} /images/vertex-buffer/multiple-buffers-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>每个属性都有其<strong>自己的缓冲区</strong>，因此其<strong>自己的字节步长</strong>.</em></span>
</p>

另一种可能的数据布局是为两个属性提供**两个不同的缓冲区**。

````{admonition} Device limits
确保更改设备限制以支持此操作：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
requiredLimits.limits.maxVertexBuffers = 2;
```
````

#### 顶点数据

因此我们有 2 个输入向量：

```{lit} C++, Define vertex data (replace, also for tangle root "Vanilla")
// x0, y0, x1, y1, ...
std::vector<float> positionData = {
	-0.5, -0.5,
	+0.5, -0.5,
	+0.0, +0.5,
	-0.55f, -0.5,
	-0.05f, +0.5,
	-0.55f, +0.5
};

// r0,  g0,  b0, r1,  g1,  b1, ...
std::vector<float> colorData = {
	1.0, 0.0, 0.0,
	0.0, 1.0, 0.0,
	0.0, 0.0, 1.0,
	1.0, 1.0, 0.0,
	1.0, 0.0, 1.0,
	0.0, 1.0, 1.0
};

vertexCount = static_cast<uint32_t>(positionData.size() / 2);
assert(vertexCount == static_cast<uint32_t>(colorData.size() / 3));
```

````{note}
这次，最大缓冲区大小/步长可以更低：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
requiredLimits.limits.maxBufferSize = 6 * 3 * sizeof(float);
requiredLimits.limits.maxVertexBufferArrayStride = 3 * sizeof(float);
```
````

#### 缓冲区

这导致两个创建两个 GPU 缓冲区 `positionBuffer` 和 `colorBuffer`：

````{tab} With webgpu.hpp
```{lit} C++, Create vertex buffer (replace)
// Create vertex buffers
BufferDescriptor bufferDesc;
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Vertex;
bufferDesc.mappedAtCreation = false;

bufferDesc.label = "Vertex Position";
bufferDesc.size = positionData.size() * sizeof(float);
positionBuffer = device.createBuffer(bufferDesc);
queue.writeBuffer(positionBuffer, 0, positionData.data(), bufferDesc.size);

bufferDesc.label = "Vertex Color";
bufferDesc.size = colorData.size() * sizeof(float);
colorBuffer = device.createBuffer(bufferDesc);
queue.writeBuffer(colorBuffer, 0, colorData.data(), bufferDesc.size);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create vertex buffer (replace, for tangle root "Vanilla")
// Create vertex buffers
WGPUBufferDescriptor bufferDesc;
bufferDesc.nextInChain = nullptr;
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Vertex;
bufferDesc.mappedAtCreation = false;

bufferDesc.label = "Vertex Position";
bufferDesc.size = positionData.size() * sizeof(float);
positionBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);
wgpuQueueWriteBuffer(queue, positionBuffer, 0, positionData.data(), bufferDesc.size);

bufferDesc.label = "Vertex Color";
bufferDesc.size = colorData.size() * sizeof(float);
colorBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);
wgpuQueueWriteBuffer(queue, colorBuffer, 0, colorData.data(), bufferDesc.size);
```
````

```{lit} C++, Create vertex buffer (hidden, append, also for tangle root "Vanilla")
// It is not easy with the auto-generation of code to remove the previously
// defined `vertexBuffer` attribute, but at the same time some compilers
// (rightfully) complain if we do not use it. This is a hack to mark the
// variable as used and have automated build tests pass.
(void)vertexBuffer;
```

我们将 `positionBuffer` 和 `colorBuffer` 声明为 `Application` 类的成员，以便我们可以在 `MainLoop()` 中访问它们：

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	Buffer positionBuffer;
	Buffer colorBuffer;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUBuffer positionBuffer;
	WGPUBuffer colorBuffer;
```
````

并且不要忘记在 `Terminate()` 中释放它们：

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
// At the beginning of Terminate()
positionBuffer.release();
colorBuffer.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
// At the beginning of Terminate()
wgpuBufferRelease(positionBuffer);
wgpuBufferRelease(colorBuffer);
```
````

#### 布局和属性

这次不是 `VertexAttribute` 结构体，而是 `VertexBufferLayout` 被向量替换：

````{tab} With webgpu.hpp
```{lit} C++, Describe vertex buffers (replace)
// We now have 2 attributes
std::vector<VertexBufferLayout> vertexBufferLayouts(2);

// Position attribute
{{Describe the position attribute and buffer layout}}

// Color attribute
{{Describe the color attribute and buffer layout}}

pipelineDesc.vertex.bufferCount = static_cast<uint32_t>(vertexBufferLayouts.size());
pipelineDesc.vertex.buffers = vertexBufferLayouts.data();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe vertex buffers (replace, for tangle root "Vanilla")
// We now have 2 attributes
std::vector<WGPUVertexBufferLayout> vertexBufferLayouts(2);

// Position attribute
{{Describe the position attribute and buffer layout}}

// Color attribute
{{Describe the color attribute and buffer layout}}

pipelineDesc.vertex.bufferCount = static_cast<uint32_t>(vertexBufferLayouts.size());
pipelineDesc.vertex.buffers = vertexBufferLayouts.data();
```
````

位置属性本身仍然像上一章中的那样，当时它是唯一的属性：

````{tab} With webgpu.hpp
```{lit} C++, Describe the position attribute and buffer layout
// Position attribute remains untouched
VertexAttribute positionAttrib;
positionAttrib.shaderLocation = 0; // @location(0)
positionAttrib.format = VertexFormat::Float32x2; // size of position
positionAttrib.offset = 0;

vertexBufferLayouts[0].attributeCount = 1;
vertexBufferLayouts[0].attributes = &positionAttrib;
vertexBufferLayouts[0].arrayStride = 2 * sizeof(float); // stride = size of position
vertexBufferLayouts[0].stepMode = VertexStepMode::Vertex;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the position attribute and buffer layout (for tangle root "Vanilla")
// Position attribute remains untouched
WGPUVertexAttribute positionAttrib;
positionAttrib.shaderLocation = 0; // @location(0)
positionAttrib.format = WGPUVertexFormat_Float32x2; // size of position
positionAttrib.offset = 0;

vertexBufferLayouts[0].attributeCount = 1;
vertexBufferLayouts[0].attributes = &positionAttrib;
vertexBufferLayouts[0].arrayStride = 2 * sizeof(float); // stride = size of position
vertexBufferLayouts[0].stepMode = WGPUVertexStepMode_Vertex;
```
````

新的颜色属性这次也有一个**字节偏移** 0（在它自己的缓冲区中），但这次有**不同的字节跨度**：

````{tab} With webgpu.hpp
```{lit} C++, Describe the color attribute and buffer layout
// Color attribute
VertexAttribute colorAttrib;
colorAttrib.shaderLocation = 1; // @location(1)
colorAttrib.format = VertexFormat::Float32x3; // size of color
colorAttrib.offset = 0;

vertexBufferLayouts[1].attributeCount = 1;
vertexBufferLayouts[1].attributes = &colorAttrib;
vertexBufferLayouts[1].arrayStride = 3 * sizeof(float); // stride = size of color
vertexBufferLayouts[1].stepMode = VertexStepMode::Vertex;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe the color attribute and buffer layout (for tangle root "Vanilla")
// Color attribute
WGPUVertexAttribute colorAttrib;
colorAttrib.shaderLocation = 1; // @location(1)
colorAttrib.format = WGPUVertexFormat_Float32x3; // size of color
colorAttrib.offset = 0;

vertexBufferLayouts[1].attributeCount = 1;
vertexBufferLayouts[1].attributes = &colorAttrib;
vertexBufferLayouts[1].arrayStride = 3 * sizeof(float); // stride = size of color
vertexBufferLayouts[1].stepMode = WGPUVertexStepMode_Vertex;
```
````

#### 渲染通道

最后，在渲染过程中，我们必须通过调用 `renderPass.setVertexBuffer` 两次来**设置两个顶点缓冲区**。第一个参数 (`slot`) 对应于 `pipelineDesc.vertex.buffers` 数组中缓冲区布局的索引。

````{tab} With webgpu.hpp
```{lit} C++, Draw a triangle (replace)
renderPass.setPipeline(pipeline);

// Set vertex buffers while encoding the render pass
renderPass.setVertexBuffer(0, positionBuffer, 0, positionBuffer.getSize());
renderPass.setVertexBuffer(1, colorBuffer, 0, colorBuffer.getSize());
//                         ^ Add a second call to set the second vertex buffer

// We use the `vertexCount` variable instead of hard-coding the vertex count
renderPass.draw(vertexCount, 1, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Draw a triangle (replace, for tangle root "Vanilla")
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);

// Set vertex buffers while encoding the render pass
wgpuRenderPassEncoderSetVertexBuffer(renderPass, 0, positionBuffer, 0, wgpuBufferGetSize(positionBuffer));
wgpuRenderPassEncoderSetVertexBuffer(renderPass, 1, colorBuffer, 0, wgpuBufferGetSize(colorBuffer));
//                                               ^ Add a second call to set the second vertex buffer

// We use the `vertexCount` variable instead of hard-coding the vertex count
wgpuRenderPassEncoderDraw(renderPass, vertexCount, 1, 0, 0);
```
````

结论
----------

```{figure} /images/color-attribute.png
:align: center
:class: with-shadow
具有颜色属性的三角形（两个选项的结果相同）。
```

````{tip}
我将背景颜色 (`clearValue`) 更改为 `Color{ 0.05, 0.05, 0.05, 1.0 }` 以更好地欣赏三角形的颜色。

```{lit} C++, Describe the attachment (hidden, replace)
renderPassColorAttachment.view = targetView;
renderPassColorAttachment.resolveTarget = nullptr;
renderPassColorAttachment.loadOp = LoadOp::Clear;
renderPassColorAttachment.storeOp = StoreOp::Store;
renderPassColorAttachment.clearValue = Color{ 0.05, 0.05, 0.05, 1.0 };
#ifndef WEBGPU_BACKEND_WGPU
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
#endif // NOT WEBGPU_BACKEND_WGPU
```

```{lit} C++, Describe the attachment (hidden, replace, for tangle root "Vanilla")
renderPassColorAttachment.view = targetView;
renderPassColorAttachment.resolveTarget = nullptr;
renderPassColorAttachment.loadOp = WGPULoadOp_Clear;
renderPassColorAttachment.storeOp = WGPUStoreOp_Store;
renderPassColorAttachment.clearValue = WGPUColor{ 0.05, 0.05, 0.05, 1.0 };
#ifndef WEBGPU_BACKEND_WGPU
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
#endif // NOT WEBGPU_BACKEND_WGPU
```
````

````{tab} With webgpu.hpp
*结果代码：* [`step033`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step033)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step033-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step033-vanilla)
````


