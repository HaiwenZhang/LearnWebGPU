你好三角形<span class="bullet">🟢</span>
==============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/hello-triangle.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/030 - Hello Triangle - vanilla
:parent: 025 - First Color
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/030 - Hello Triangle
:parent: 028 - C++ Wrapper
```

````{tab} With webgpu.hpp
*结果代码：* [`step030`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-vanilla)
````

从整体轮廓来看，**画一个三角形**就这么简单：

````{tab} With webgpu.hpp
```{lit} C++, Draw a triangle
// Select which render pipeline to use
renderPass.setPipeline(pipeline);
// Draw 1 instance of a 3-vertices shape
renderPass.draw(3, 1, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Draw a triangle (for tangle root "Vanilla")
// Select which render pipeline to use
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);
// Draw 1 instance of a 3-vertices shape
wgpuRenderPassEncoderDraw(renderPass, 3, 1, 0, 0);
```
````

有点冗长的是**渲染管道**的配置，以及**着色器**的创建。

渲染管线
---------------

为了实现高性能实时 3D 渲染，GPU 通过预定义的管道处理形状。管道本身**始终相同**（它通常内置于硬件的物理架构中），但我们可以通过多种方式**配置**它。为此，WebGPU 提供了一个“渲染管道”对象。

```{note}
如果您熟悉 OpenGL，您可以将 WebGPU 的渲染管道视为配置渲染管道的所有有状态函数的记忆值。
```

下图说明了渲染管道执行的数据处理**阶段**的顺序。其中大多数是**固定功能**阶段，这意味着我们只能自定义一些变量，但最强大的是**可编程阶段**。

在这些可编程阶段中，称为**着色器**的真正程序以大规模并行方式执行（跨输入**顶点**，或跨光栅化**片段**）。

```{image} /images/render-pipeline-light.svg
:align: center
:class: only-light
```

```{image} /images/render-pipeline-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>WebGPU 使用的渲染管道抽象，在下面的小节中详细介绍。</em></span>
</p>

```{note}
其他图形 API 提供对更多可编程阶段（几何着色器、网格着色器、任务着色器）的访问。这些不是 Web 标准的一部分。它们将来可能会作为纯原生扩展提供，但在很多情况下，人们可以使用通用的[计算着色器](../basic-compute/index.md) 来模仿它们的行为。
```

与往常一样，我们构建一个描述符来创建渲染管道：

````{tab} With webgpu.hpp
```{lit} C++, Create Render Pipeline
RenderPipelineDescriptor pipelineDesc;
{{Describe render pipeline}}
RenderPipeline pipeline = device.createRenderPipeline(pipelineDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create Render Pipeline (for tangle root "Vanilla")
WGPURenderPipelineDescriptor pipelineDesc{};
pipelineDesc.nextInChain = nullptr;
{{Describe render pipeline}}
WGPURenderPipeline pipeline = wgpuDeviceCreateRenderPipeline(device, &pipelineDesc);
```
````

我们现在详细介绍**不同阶段**的配置。我们从一个非常**最小的设置**开始，其中许多功能未使用，它们将在接下来的章节中逐步介绍。

**管道描述**包含以下步骤，**遵循上图的顺序**：

```{lit} C++, Describe render pipeline (also for tangle root "Vanilla")
{{Describe vertex pipeline state}}
{{Describe primitive pipeline state}}
{{Describe fragment pipeline state}}
{{Describe stencil/depth pipeline state}}
{{Describe multi-sampling state}}
{{Describe pipeline layout}}
```

### 顶点管道状态

**顶点获取**和**顶点着色器**阶段都是通过**顶点状态**结构配置的，可通过`pipelineDesc.vertex`访问。

```{lit} C++, Describe vertex pipeline state (also for tangle root "Vanilla")
// Configure 'pipelineDesc.vertex'
{{Describe vertex buffers}}
{{Describe vertex shader}}
```

渲染管道首先从 GPU 内存中的一些缓冲区**获取顶点属性**。这些“属性”通常至少包括一个“顶点位置”，并且可能包括附加的每个顶点信息，例如“颜色”、“法线”、“纹理坐标”等。

**在第一个示例中**，我们在着色器中对三角形的 3 个顶点的位置进行硬编码，因此我们甚至不需要位置缓冲区。

```{lit} C++, Describe vertex buffers (also for tangle root "Vanilla")
// We do not use any vertex buffer for this first simplistic example
pipelineDesc.vertex.bufferCount = 0;
pipelineDesc.vertex.buffers = nullptr;
```

然后每个顶点都由自定义的**顶点着色器**处理。着色器是以下各项的组合：

1. 一个**着色器模块**，其中包含着色器的实际代码。
2. **入口点**，它是着色器模块中每个顶点必须调用的函数名称。这使得给定的着色器模块能够同时包含多个渲染管道配置的入口点。特别是，我们对顶点和片段着色器使用相同的模块。
3. 着色器的**常量**的赋值数组。我们暂时不使用任何。

```{lit} C++, Describe vertex shader (also for tangle root "Vanilla")
// NB: We define the 'shaderModule' in the second part of this chapter.
// Here we tell that the programmable vertex shader stage is described
// by the function called 'vs_main' in that module.
pipelineDesc.vertex.module = shaderModule;
pipelineDesc.vertex.entryPoint = "vs_main";
pipelineDesc.vertex.constantCount = 0;
pipelineDesc.vertex.constants = nullptr;
```

`shaderModule` 将在[下一节](#shaders) 中定义。

### 原始管道状态

在 `pipelineDesc.primitive` 处找到的**原始状态**结构配置**原始装配**和**光栅化**阶段。

**光栅化**是 GPU 实现的 3D 渲染算法的核心❤️。它将**图元**（点、线或三角形）转换为一系列**片段**，这些**片段**对应于图元覆盖的像素。它**插入**顶点着色器输出的任何额外属性，以便每个片段接收所有属性的值。

**原始装配**配置包括说明先前获取的顶点数组必须如何**连接**到点云、线框或三角形汤。我们将其设置为默认配置：

````{tab} With webgpu.hpp
```{lit} C++, Describe primitive pipeline state
// Each sequence of 3 vertices is considered as a triangle
pipelineDesc.primitive.topology = PrimitiveTopology::TriangleList;

// We'll see later how to specify the order in which vertices should be
// connected. When not specified, vertices are considered sequentially.
pipelineDesc.primitive.stripIndexFormat = IndexFormat::Undefined;

// The face orientation is defined by assuming that when looking
// from the front of the face, its corner vertices are enumerated
// in the counter-clockwise (CCW) order.
pipelineDesc.primitive.frontFace = FrontFace::CCW;

// But the face orientation does not matter much because we do not
// cull (i.e. "hide") the faces pointing away from us (which is often
// used for optimization).
pipelineDesc.primitive.cullMode = CullMode::None;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe primitive pipeline state (for tangle root "Vanilla")
// Each sequence of 3 vertices is considered as a triangle
pipelineDesc.primitive.topology = WGPUPrimitiveTopology_TriangleList;

// We'll see later how to specify the order in which vertices should be
// connected. When not specified, vertices are considered sequentially.
pipelineDesc.primitive.stripIndexFormat = WGPUIndexFormat_Undefined;

// The face orientation is defined by assuming that when looking
// from the front of the face, its corner vertices are enumerated
// in the counter-clockwise (CCW) order.
pipelineDesc.primitive.frontFace = WGPUFrontFace_CCW;

// But the face orientation does not matter much because we do not
// cull (i.e. "hide") the faces pointing away from us (which is often
// used for optimization).
pipelineDesc.primitive.cullMode = WGPUCullMode_None;
```
````

```{note}
通常我们将**剔除模式**设置为`Front`，以避免在渲染对象内部时浪费资源。但对于初学者来说，几个小时在屏幕上看不到任何东西却发现三角形只是面向错误的方向可能会非常令人沮丧，因此我建议您在**开发**时将其设置为 `None` 。
```

### 片段着色器

一旦光栅化器将图元分割成许多小片段，就会为每个片段调用 **片段着色器** 阶段。该着色器接收顶点着色器生成的插值，并且必须依次输出片段的**最终颜色**。

```{note}
请记住，所有这些阶段都发生在**非常并行**和**异步**环境中。当渲染大网格时，可以在最后一个图元被光栅化之前调用第一个图元的片段着色器。
```

该配置与顶点着色器的配置非常相似：

````{tab} With webgpu.hpp
```{lit} C++, Describe fragment pipeline state
// We tell that the programmable fragment shader stage is described
// by the function called 'fs_main' in the shader module.
FragmentState fragmentState;
fragmentState.module = shaderModule;
fragmentState.entryPoint = "fs_main";
fragmentState.constantCount = 0;
fragmentState.constants = nullptr;
{{We'll configure the blending stage here}}
pipelineDesc.fragment = &fragmentState;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe fragment pipeline state (for tangle root "Vanilla")
// We tell that the programmable fragment shader stage is described
// by the function called 'fs_main' in the shader module.
WGPUFragmentState fragmentState{};
fragmentState.module = shaderModule;
fragmentState.entryPoint = "fs_main";
fragmentState.constantCount = 0;
fragmentState.constants = nullptr;
{{We'll configure the blending stage here}}
pipelineDesc.fragment = &fragmentState;
```
````

请注意，片段阶段是**可选的**，因此 `pipelineDesc.fragment` 是一个（可能为空）指针，而不是直接保存片段状态结构。

### 模板/深度状态

**深度测试**用于丢弃与*相同像素*关联的其他片段**后面的片段。请记住，片段是给定基元在给定像素上的投影，因此**当基元彼此重叠时**，将为同一像素发出多个片段。片段具有**深度**信息，供深度测试使用。

**模板测试**是另一种片段丢弃机制，用于隐藏基于先前渲染的图元的片段。让我们**暂时忽略**深度和模板机制**，我们将在[深度缓冲区](3d-meshes/depth-buffer.md)章节中介绍它们。

```{lit} C++, Describe stencil/depth pipeline state (also for tangle root "Vanilla")
// We do not use stencil/depth testing for now
pipelineDesc.depthStencil = nullptr;
```

### 混合

混合阶段获取每个片段的颜色并将其“绘制”到目标颜色附件上。如果 **destination** 像素中的原始颜色是 $(r_d, g_d, b_d, a_d)$ 并且要绘制的 **source** 片段的颜色是 $(r_s, g_s, b_s, a_s)$，那么最终像素的最终颜色 $(r, g, b, a)$ 应该是什么？这就是 **混合状态** 所指定的。

我们还必须指定颜色预计存储在最终附件中的**格式**（即如何将值表示为零和一）。

````{tab} With webgpu.hpp
```{lit} C++, We'll configure the blending stage here
BlendState blendState;
{{Configure color blending equation}}
{{Configure alpha blending equation}}

ColorTargetState colorTarget;
colorTarget.format = surfaceFormat;
colorTarget.blend = &blendState;
colorTarget.writeMask = ColorWriteMask::All; // We could write to only some of the color channels.

// We have only one target because our render pass has only one output color
// attachment.
fragmentState.targetCount = 1;
fragmentState.targets = &colorTarget;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, We'll configure the blending stage here (for tangle root "Vanilla")
WGPUBlendState blendState{};
{{Configure color blending equation}}
{{Configure alpha blending equation}}

WGPUColorTargetState colorTarget{};
colorTarget.format = surfaceFormat;
colorTarget.blend = &blendState;
colorTarget.writeMask = WGPUColorWriteMask_All; // We could write to only some of the color channels.

// We have only one target because our render pass has only one output color
// attachment.
fragmentState.targetCount = 1;
fragmentState.targets = &colorTarget;
```
````

**混合方程**可以为rgb通道和alpha通道独立设置，一般情况下，它采用以下形式：

$$
rgb = \texttt{srcFactor} \times rgb_s ~~[\texttt{操作}]~~ \texttt{dstFactor} \times rgb_d
$$

**通常的混合**方程配置为 $rgb = a_s \times rgb_s + (1 - a_s) \times rgb_d$。这对应于在现有像素值上渲染片段的**分层的直觉：

````{tab} With webgpu.hpp
```{lit} C++, Configure color blending equation
blendState.color.srcFactor = BlendFactor::SrcAlpha;
blendState.color.dstFactor = BlendFactor::OneMinusSrcAlpha;
blendState.color.operation = BlendOperation::Add;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Configure color blending equation (for tangle root "Vanilla")
blendState.color.srcFactor = WGPUBlendFactor_SrcAlpha;
blendState.color.dstFactor = WGPUBlendFactor_OneMinusSrcAlpha;
blendState.color.operation = WGPUBlendOperation_Add;
```
````

Alpha 通道也有一个类似的混合方程：

$$
a = \texttt{srcFactor} \times a_s ~~[\texttt{操作}]~~ \texttt{dstFactor} \times a_d
$$

例如，我们可以保持目标 alpha 不变： $a = a_d = 0 \times a_s + 1 \times a_d$

````{tab} With webgpu.hpp
```{lit} C++, Configure alpha blending equation
blendState.alpha.srcFactor = BlendFactor::Zero;
blendState.alpha.dstFactor = BlendFactor::One;
blendState.alpha.operation = BlendOperation::Add;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Configure alpha blending equation (for tangle root "Vanilla")
blendState.alpha.srcFactor = WGPUBlendFactor_Zero;
blendState.alpha.dstFactor = WGPUBlendFactor_One;
blendState.alpha.operation = WGPUBlendOperation_Add;
```
````

### 多重采样

我之前说过，片段是基元投影到特定像素上的部分。实际上，我们可以将像素分割成子元素，称为**样本**，并且片段与样本相关联。像素的值是通过对其关联样本进行平均来计算的。

这种机制称为“多重采样”，用于“抗锯齿”，但我们暂时将其保留，将每个像素的样本数设置为 1。

```{lit} C++, Describe multi-sampling state (also for tangle root "Vanilla")
// Samples per pixel
pipelineDesc.multisample.count = 1;
// Default value for the mask, meaning "all bits on"
pipelineDesc.multisample.mask = ~0u;
// Default value as well (irrelevant for count = 1 anyways)
pipelineDesc.multisample.alphaToCoverageEnabled = false;
```

好的，我们终于**配置了渲染管道的所有阶段**。现在剩下的就是指定两个**可编程阶段**的行为，即**顶点**和**片段着色器**。

Shaders
-------

下面开始介绍着色器。

顶点和片段可编程阶段都使用相同的**着色器模块**，我们必须首先创建该模块。该模块是一种动态库（如 .dll、.so 或 .dylib 文件），不同之处在于它使用的是 GPU 的**二进制语言，而不是 CPU 的二进制语言。

与任何编译程序一样，着色器首先用**人类可写的编程语言**编写，然后**编译**为低级机器代码。然而，低级代码高度依赖于硬件，并且通常没有公开记录。因此，应用程序随着色器的**源代码**一起分发，该源代码**在初始化应用程序时**动态编译**。

### 着色器代码

WebGPU官方使用的着色器语言称为[WGSL](https://gpuweb.github.io/gpuweb/wgsl/)，即*WebGPU Shading Language*。 WebGPU 的任何实现都必须支持它，并且 API 的 JavaScript 版本仅支持 WGSL，但本机标头 `webgpu.h` 还提供了提供以 [SPIR-V](https://www.khronos.org/spir) 或 [GLSL](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL))（仅限 `wgpu-native`）编写的着色器的可能性。

```{tip}
**SPIR-V** 更像是一种中间表示，即 **字节码**，而不是手动编写的东西。它是 Vulkan API 使用的着色器语言，并且存在许多工具可以从其他常见着色器语言（HLSL、GLSL）**交叉编译**代码，因此当需要重用**现有着色器代码库**时，这是一个有趣的选择。
```

```{note}
另请注意，WGSL 最初被设计为 SPIR-V 编程模型的人类可编辑版本，因此从 SPIR-V 到 WGSL 的 **转译** 理论上是高效且无损的（为此使用 [Naga](https://github.com/gfx-rs/naga) 或 [Tint](https://dawn.googlesource.com/tint)]）。
```

在本指南中，我们将仅使用 WGSL。如果您习惯使用 C/C++，它的语法可能看起来有点陌生，所以让我们从一个非常简单的函数及其在 C++ 中的等效函数开始：

```rust
// 在 WGSL 中
fn my_addition(x: f32, y: f32) -> f32 {
让 z = x + y；
返回z；
}
```

```C++
// In C++
float my_addition(float x, float y) {
	auto z = x + y;
	return z;
}
```

请注意，**类型**是如何在参数名称的**之后**而不是**之前**指定的。 `z` 的类型是自动推断的，尽管我们可以使用 `let z: f32 = ...` 强制它。返回类型在参数之后使用箭头 `->` 指定。

```{note}
关键字 `let` 定义一个常量，即不能重新分配的变量。常规变量使用 `var` 关键字定义。
```

现在这是我们三角形的着色器：

```{lit} rust, The WGSL shader source code (also for tangle root "Vanilla")
@vertex
fn vs_main(@builtin(vertex_index) in_vertex_index: u32) -> @builtin(position) vec4f {
	var p = vec2f(0.0, 0.0);
	if (in_vertex_index == 0u) {
		p = vec2f(-0.5, -0.5);
	} else if (in_vertex_index == 1u) {
		p = vec2f(0.5, -0.5);
	} else {
		p = vec2f(0.0, 0.5);
	}
	return vec4f(p, 0.0, 1.0);
}

@fragment
fn fs_main() -> @location(0) vec4f {
	return vec4f(0.0, 0.4, 1.0, 1.0);
}
```

函数名称 `vs_main` （或 `fs_main`）必须与在顶点（或片段）状态中指定为 `entryPoint` 的名称完全相同。

```{note}
着色器语言本身支持最大尺寸为 4 的**向量和矩阵类型**。类型 `vec2f` 是 2 个浮点数的向量，以及*模板化*类型 `vec2<f32>` 的**别名**。作为另一个示例，类型 `vec4<u32>` 是 4 个无符号整数的向量，并且具有别名 `vec4u`。
```

以 `@` 开头的标记称为**属性**，并用各种信息装饰随后出现的对象。例如， `@builtin(vertex_index)` 告诉参数 `in_vertex_index` 将由**内置**输入顶点属性（即**顶点索引**）填充。请注意，参数 `in_vertex_index` 可以有任何其他名称：只要它用相同的内置属性修饰，它就具有相同的**语义**。

我们使用这个顶点索引来设置着色器的输出值。此输出标有 `@builtin(position)` 属性，表示它必须由光栅化器**解释为顶点位置。

为了获得更大的灵活性，应该从文件加载着色器代码，但现在我们只需将其存储在 `main.cpp` 中的多行字符串文字中：

```{lit} C++, Shader source literal (also for tangle root "Vanilla")
const char* shaderSource = R"(
{{The WGSL shader source code}}
)";
```

```{lit} C++, file: main.cpp (replace, hidden, also for tangle root "Vanilla")
{{Includes}}
{{Global declarations}}
{{Application class}}
{{Main function}}
{{Application implementation}}
```

```{lit} C++, Global declarations (hidden, also for tangle root "Vanilla")
{{Shader source literal}}
```

### 创建着色器模块

着色器模块的创建照常开始：

````{tab} With webgpu.hpp
```{lit} C++, Create Shader Module
ShaderModuleDescriptor shaderDesc;
{{Describe shader module}}
ShaderModule shaderModule = device.createShaderModule(shaderDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create Shader Module (for tangle root "Vanilla")
WGPUShaderModuleDescriptor shaderDesc{};
{{Describe shader module}}
WGPUShaderModule shaderModule = wgpuDeviceCreateShaderModule(device, &shaderDesc);
```
````

乍一看，这个描述符似乎只有一个编译“提示”数组需要填充，我们将其留空（甚至在使用 Dawn 时什么也没有）：

```{lit} C++, Describe shader module (also for tangle root "Vanilla")
#ifdef WEBGPU_BACKEND_WGPU
shaderDesc.hintCount = 0;
shaderDesc.hints = nullptr;
#endif
```

但这一次，我们**没有**将 `nextInChain` 设置为 `nullptr`！

`nextInChain` 指针是WebGPU **扩展机制**的入口点。它要么为 null，要么指向 `WGPUChainedStruct` 类型的结构。这个结构非常简单。首先，它可能递归地具有 `next` 元素（同样，要么为 null，要么指向某个 `WGPUChainedStruct`）。其次，它有一个**结构类型** `sType`，它是一个枚举，告诉链元素可以在哪个结构中进行转换。每个定义以字段 `WGPUChainedStruct chain` 开头的结构都有一个关联的 SType。

要从 WGSL 代码创建着色器模块，我们使用 `ShaderModuleWGSLDescriptor` SType。类似地，可以使用 `WGPUShaderModuleSPIRVDescriptor` 创建 SPIR-V 着色器。

当转换为简单的 `WGPUChainedStruct` 时，字段 `shaderCodeDesc.chain` 对应于链式结构，必须将其设置为相应的 SType 枚举值：

````{tab} With webgpu.hpp
```{lit} C++, Describe shader module (append)
ShaderModuleWGSLDescriptor shaderCodeDesc;
// Set the chained struct's header
shaderCodeDesc.chain.next = nullptr;
shaderCodeDesc.chain.sType = SType::ShaderModuleWGSLDescriptor;
// Connect the chain
shaderDesc.nextInChain = &shaderCodeDesc.chain;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Describe shader module (append, for tangle root "Vanilla")
WGPUShaderModuleWGSLDescriptor shaderCodeDesc{};
// Set the chained struct's header
shaderCodeDesc.chain.next = nullptr;
shaderCodeDesc.chain.sType = WGPUSType_ShaderModuleWGSLDescriptor;
// Connect the chain
shaderDesc.nextInChain = &shaderCodeDesc.chain;
```
````

最后我们可以设置着色器代码描述符的实际负载：

```{lit} C++, Describe shader module (append, also for tangle root "Vanilla")
shaderCodeDesc.code = shaderSource;
```

```{admonition} Dawn
WebGPU 的 Dawn 实现在着色器模块描述符中不包含 `hints`/`hintCount`。
```

### 管道布局

在运行代码之前，最后一件事是：着色器可能需要**访问输入和输出资源**（缓冲区和/或纹理）。通过配置内存**布局**，这些资源可供管道使用。我们的第一个示例不使用任何资源：

```{lit} C++, Describe pipeline layout (also for tangle root "Vanilla")
pipelineDesc.layout = nullptr;
```

```{note}
实际上，将管道布局设置为`nullptr`并不意味着没有输入/输出资源。它而是要求后端通过检查着色器自行**找出布局**（在这种情况下是等效的）。
```

总结
-----------

我们现在可以把这些点连起来了！我建议我们用 **专用方法** 初始化渲染管道：

```{lit} C++, Application implementation (hidden, append, also for tangle root "Vanilla")
{{InitializePipeline method}}
```

````{tab} With webgpu.hpp
```{lit} C++, InitializePipeline method
void Application::InitializePipeline() {
	{{Create Shader Module}}
	{{Create Render Pipeline}}

	// We no longer need to access the shader module
	shaderModule.release();
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, InitializePipeline method (for tangle root "Vanilla")
void Application::InitializePipeline() {
	{{Create Shader Module}}
	{{Create Render Pipeline}}

	// We no longer need to access the shader module
	wgpuShaderModuleRelease(shaderModule);
}
```
````

因此，我们**将 `pipeline` 的声明**移至**类属性**。我们还对在表面配置期间定义的 `surfaceFormat` 执行相同的操作：

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
// In Application class
private:
	RenderPipeline pipeline;
	TextureFormat surfaceFormat = TextureFormat::Undefined;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
// In Application class
private:
	WGPURenderPipeline pipeline;
	WGPUTextureFormat surfaceFormat = WGPUTextureFormat_Undefined;
```
````

````{warning}
确保在分别调用 `createRenderPipeline` 和 `getPreferredFormat` 时，不要通过重新声明来在本地**隐藏** `pipeline` 和 `surfaceFormat` 的类级声明。

```{lit} C++, Create Render Pipeline (hidden, replace)
RenderPipelineDescriptor pipelineDesc;
{{Describe render pipeline}}
pipeline = device.createRenderPipeline(pipelineDesc);
```

```{lit} C++, Create Render Pipeline (hidden, replace, for tangle root "Vanilla")
WGPURenderPipelineDescriptor pipelineDesc{};
pipelineDesc.nextInChain = nullptr;
{{Describe render pipeline}}
pipeline = wgpuDeviceCreateRenderPipeline(device, &pipelineDesc);
```

```{lit} C++, Describe Surface Format (hidden, replace)
surfaceFormat = surface.getPreferredFormat(adapter);
config.format = surfaceFormat;
// And we do not need any particular view format:
config.viewFormatCount = 0;
config.viewFormats = nullptr;
```

```{lit} C++, Describe Surface Format (hidden, replace, for tangle root "Vanilla")
surfaceFormat = wgpuSurfaceGetPreferredFormat(surface, adapter);
config.format = surfaceFormat;
// And we do not need any particular view format:
config.viewFormatCount = 0;
config.viewFormats = nullptr;
```
````

我们还在应用程序声明中声明我们的初始化方法：

```{lit} C++, Private methods (append, also for tangle root "Vanilla")
// In Application class
private:
	void InitializePipeline();
```

我们在 `Initialize()` 方法的末尾调用它：

```{lit} C++, Initialize (append, also for tangle root "Vanilla")
// At the end of Initialize()
InitializePipeline();
```

**相反**我们在 `Terminate()` 方法的开头释放管道：

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
// At the beginning of Terminate()
pipeline.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
// At the beginning of Terminate()
wgpuRenderPipelineRelease(pipeline);
```
````

最后，在创建 `renderPass` 对象之后，我们在主循环中插入本章开头的两行：

````{tab} With webgpu.hpp
```{lit} C++, Draw a triangle (replace)
// Select which render pipeline to use
renderPass.setPipeline(pipeline);
// Draw 1 instance of a 3-vertices shape
renderPass.draw(3, 1, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Draw a triangle (replace, for tangle root "Vanilla")
// Select which render pipeline to use
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);
// Draw 1 instance of a 3-vertices shape
wgpuRenderPassEncoderDraw(renderPass, 3, 1, 0, 0);
```
````

```{lit} C++, Use Render Pass (hidden, replace, also for tangle root "Vanilla")
{{Draw a triangle}}
```

还有塔达姆！你终于应该看到**一个三角形**！

```{figure} /images/hello-triangle.png
:align: center
:class: with-shadow
我们的第一个三角形是使用 WebGPU 渲染的。
```

```{admonition} Dawn
使用 Dawn 时，您可能会看到**不同的颜色**（更饱和），因为目标表面使用**不同的色彩空间**。更多相关内容[稍后](input-geometry/loading-from-file.md#color-issue)。
```

结论
----------

本章介绍了在 GPU 上渲染基于三角形的形状的**核心骨架**。目前这些都是 2D 图形，但一旦一切就绪，切换到 3D 将很简单。我们看到了两个非常重要的概念：

- **渲染管道**，它基于硬件实际工作的方式，其中一些部分是固定的，以提高效率，而其他部分是可编程的。
- **着色器**，它们是驱动管道可编程阶段的 GPU 端程序。

### 接下来是什么？

用于 3D 渲染的计算机图形学的关键算法和技术大部分是在着色器代码中实现的。不过，我们目前仍然缺少的是 C++ 代码 (CPU) 和着色器 (GPU) 之间**通信**的方法。

接下来的两章重点介绍两种向渲染管道提供输入的方法：顶点属性（每个顶点有一个值）和统一变量（定义给定调用的所有顶点和片段通用的变量）。

然后我们从管道中休息一下并切换到 **3D 网格**，这最终与代码无关，而与数学有关。我们还介绍了一些与基本**相机控制器**的**交互**。然后我们介绍第三种提供输入资源的方法，即**纹理**，以及如何将它们映射到网格上。

存储纹理以相反的方式使用，从渲染管道中获取数据，将仅在高级章节中介绍。相反，本节的最后一章完全致力于**照明**和**材质建模**的计算机图形问题。

````{tab} With webgpu.hpp
*结果代码：* [`step030`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-vanilla)
````
