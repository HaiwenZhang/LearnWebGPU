索引缓冲区 <span class="bullet">🟢</span>
============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/input-geometry/index-buffer.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/034 - Index Buffer - vanilla
:parent: 033 - Multiple Attributes - Option A - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/034 - Index Buffer
:parent: 033 - Multiple Attributes - Option A
```

````{tab} With webgpu.hpp
*结果代码：* [`step034`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step034)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step034-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step034-vanilla)
````

索引缓冲区用于将**顶点属性**列表与它们**连接**的实际顺序分开。为了说明它的有趣之处，让我们画一个由 2 个三角形组成的正方形。

```{image} /images/quad-light.plain.svg
:align: center
:class: only-light
```

```{image} /images/quad-dark.plain.svg
:align: center
:class: only-dark
```

指数数据
----------

绘制此类正方形的一种直接方法是使用以下顶点属性：

```C++
std::vector<float> vertexData = {
	// Triangle #0
	-0.5, -0.5, // A
	+0.5, -0.5,
	+0.5, +0.5, // C

	// Triangle #1
	-0.5, -0.5, // A
	+0.5, +0.5, // C
	-0.5, +0.5,
};
```

但正如您所看到的，一些**数据是重复的**（点 $A$ 和 $C$）。对于具有相连三角形的较大形状，这种重复可能会更严重。

表达正方形几何形状的一种更紧凑的方法是将**位置**与**连接性**分开：

```C++
// Define point data
// The de-duplicated list of point positions
std::vector<float> pointData = {
	-0.5, -0.5, // Point #0 (A)
	+0.5, -0.5, // Point #1
	+0.5, +0.5, // Point #2 (C)
	-0.5, +0.5, // Point #3
};

// Define index data
// This is a list of indices referencing positions in the pointData
std::vector<uint16_t> indexData = {
	0, 1, 2, // Triangle #0 connects points #0, #1 and #2
	0, 2, 3  // Triangle #1 connects points #0, #2 and #3
};
```

索引数据的类型必须为 `uint16_t` 或 `uint32_t`。前者更紧凑，但仅限于 $2^{16} = 65 536$ 个顶点。

````{note}
在这个例子中我还保留了交错颜色属性，我的点数据是：

```{lit} C++, Define point data (also for tangle root "Vanilla")
std::vector<float> pointData = {
	// x,   y,     r,   g,   b
	-0.5, -0.5,   1.0, 0.0, 0.0,
	+0.5, -0.5,   0.0, 1.0, 0.0,
	+0.5, +0.5,   0.0, 0.0, 1.0,
	-0.5, +0.5,   1.0, 1.0, 0.0
};
```

```{lit} C++, Define index data (hidden, also for tangle root "Vanilla")
// This is a list of indices referencing positions in the pointData
std::vector<uint16_t> indexData = {
	0, 1, 2, // Triangle #0 connects points #0, #1 and #2
	0, 2, 3  // Triangle #1 connects points #0, #2 and #3
};
```

使用索引缓冲区会增加 `6 * sizeof(uint16_t)` = 12 字节的**开销**，但也节省** `2 * 5 * sizeof(float)` = 40 字节，因此即使在这个非常简单的示例中也值得使用。
````

这种数据分割重新组织了我们的缓冲区初始化方法：

```{lit} C++, InitializeBuffers method (replace, also for tangle root "Vanilla")
void Application::InitializeBuffers() {
	{{Define point data}}
	{{Define index data}}
	
	// We now store the index count rather than the vertex count
	indexCount = static_cast<uint32_t>(indexData.size());

	{{Create point buffer}}
	{{Create index buffer}}
}
```

````{topic} Terminology
当引用去重复属性缓冲区时，我通常将名称**顶点**数据替换为**点**数据。换句话说，`vertex[i] = points[index[i]]`。名称“顶点”用于表示三角形的**角**，即一对点和使用该点的三角形。

```{lit} C++, Create point buffer (hidden)
// Create point buffer
BufferDescriptor bufferDesc;
bufferDesc.size = pointData.size() * sizeof(float);
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Vertex; // Vertex usage here!
bufferDesc.mappedAtCreation = false;
pointBuffer = device.createBuffer(bufferDesc);

// Upload geometry data to the buffer
queue.writeBuffer(pointBuffer, 0, pointData.data(), bufferDesc.size);
```

```{lit} C++, Create point buffer (hidden, for tangle root "Vanilla")
// Create point buffer
WGPUBufferDescriptor bufferDesc{};
bufferDesc.nextInChain = nullptr;
bufferDesc.size = pointData.size() * sizeof(float);
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Vertex; // Vertex usage here!
bufferDesc.mappedAtCreation = false;
pointBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);

// Upload geometry data to the buffer
wgpuQueueWriteBuffer(queue, pointBuffer, 0, pointData.data(), bufferDesc.size);
```
````

在**应用程序属性**列表中，我们将 `vertexBuffer` 替换为 `pointBuffer` 和 `indexBuffer`，并将 `vertexCount` 替换为 `indexCount`。

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	Buffer pointBuffer;
	Buffer indexBuffer;
	uint32_t indexCount;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUBuffer pointBuffer;
	WGPUBuffer indexBuffer;
	uint32_t indexCount;
```
````

像往常一样，我们释放 `Terminate()` 中的缓冲区

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
pointBuffer.release();
indexBuffer.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
wgpuBufferRelease(pointBuffer);
wgpuBufferRelease(indexBuffer);
```
````

```{lit} C++, Create point buffer (hidden, append, also for tangle root "Vanilla")
// It is not easy with the auto-generation of code to remove the previously
// defined `vertexBuffer` attribute, but at the same time some compilers
// (rightfully) complain if we do not use it. This is a hack to mark the
// variable as used and have automated build tests pass.
(void)vertexBuffer;
(void)vertexCount;
```

缓冲区创建
---------------

当然，**索引数据**必须存储在**GPU端缓冲区**中。该缓冲区需要使用 `BufferUsage::Index`。

````{tab} With webgpu.hpp
```{lit} C++, Create index buffer
// Create index buffer
// (we reuse the bufferDesc initialized for the vertexBuffer)
bufferDesc.size = indexData.size() * sizeof(uint16_t);
{{Fix buffer size}}
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Index;
indexBuffer = device.createBuffer(bufferDesc);

queue.writeBuffer(indexBuffer, 0, indexData.data(), bufferDesc.size);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create index buffer (for tangle root "Vanilla")
// Create index buffer
// (we reuse the bufferDesc initialized for the vertexBuffer)
bufferDesc.size = indexData.size() * sizeof(uint16_t);
{{Fix buffer size}}
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Index;;
indexBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);

wgpuQueueWriteBuffer(queue, indexBuffer, 0, indexData.data(), bufferDesc.size);
```
````

````{important}
`writeBuffer` 操作必须复制 **4 的倍数** 的字节数。为了确保这一点，我们必须在创建缓冲区之前**将缓冲区大小**提高到下一个 4 的倍数：

```{lit} C++, Fix buffer size (also for tangle root "Vanilla")
bufferDesc.size = (bufferDesc.size + 3) & ~3; // round up to the next multiple of 4
```

这意味着我们还必须确保 `indexData.size()` 是 2 的倍数（因为 `sizeof(uint16_t)` 是 2）：

```{lit} C++, Fix buffer size (append, also for tangle root "Vanilla")
indexData.resize((indexData.size() + 1) & ~1); // round up to the next multiple of 2
```
````

渲染通道
-----------

要使用索引缓冲区进行绘制，渲染通道编码中有**两个变化**：

1. 使用 `renderPass.setIndexBuffer` 设置活动索引缓冲区。
2. 将 `draw()` 替换为 `drawIndexed()`。

````{tab} With webgpu.hpp
```{lit} C++, Use Render Pass (replace)
renderPass.setPipeline(pipeline);

// Set both vertex and index buffers
renderPass.setVertexBuffer(0, pointBuffer, 0, pointBuffer.getSize());
// The second argument must correspond to the choice of uint16_t or uint32_t
// we've done when creating the index buffer.
renderPass.setIndexBuffer(indexBuffer, IndexFormat::Uint16, 0, indexBuffer.getSize());

// Replace `draw()` with `drawIndexed()` and `vertexCount` with `indexCount`
// The extra argument is an offset within the index buffer.
renderPass.drawIndexed(indexCount, 1, 0, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Use Render Pass (replace, for tangle root "Vanilla")
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);

// Set both vertex and index buffers
wgpuRenderPassEncoderSetVertexBuffer(renderPass, 0, pointBuffer, 0, wgpuBufferGetSize(pointBuffer));
// The second argument must correspond to the choice of uint16_t or uint32_t
// we've done when creating the index buffer.
wgpuRenderPassEncoderSetIndexBuffer(renderPass, indexBuffer, WGPUIndexFormat_Uint16, 0, wgpuBufferGetSize(indexBuffer));

// Replace `draw()` with `drawIndexed()` and `vertexCount` with `indexCount`
// The extra argument is an offset within the index buffer.
wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);
```
````

现在我们看到了“正方形”，其中颜色突出显示了两个三角形共享红点和蓝点（$A$ 和 $C$）的方式：

```{figure} /images/deformed-quad.png
:align: center
:class: with-shadow
由于窗口的纵横比，正方形发生了变形。
```

比率校正
----------------

我们获得的正方形是变形的，因为它的坐标是相对于窗口尺寸**表示的。这可以通过将坐标之一乘以窗口的比率（在我们的例子中为 $640/480$）来解决。

我们可以在初始顶点数据向量中执行此操作，但这需要在窗口尺寸发生变化时更新这些值。一个更有趣的选择是使用**顶点着色器的强大功能**：

```{lit} rust, Vertex shader position (also for tangle root "Vanilla")
// In vs_main():
let ratio = 640.0 / 480.0; // The width and height of the target surface
out.position = vec4f(in.position.x, in.position.y * ratio, 0.0, 1.0);
```

```{lit} rust, Vertex shader body (replace, hidden, also for tangle root "Vanilla")
var out: VertexOutput; // create the output struct
{{Vertex shader position}}
out.color = in.color; // forward the color attribute to the fragment shader
return out;
```

虽然很基础，但这是引入 **3D 变换** 时顶点着色器的关键用途的第一步。

```{note}
像这样在着色器中硬编码窗口分辨率可能会让人感觉有点不满意，但我们很快就会看到如何通过制服使其更加灵活。
```

```{figure} /images/quad.png
:align: center
:class: with-shadow
预期的正方形
```

结论
----------

使用索引缓冲区最终是一个相当**简单的概念**，并且可以节省大量 VRAM（GPU 内存）。

此外，它对应于传统格式通常编码 3D 网格的方式，以便**保留连接信息**，这对于**创作**很重要（这样当用户移动一个点时，共享该点的所有三角形都会受到影响）。

```{lit} C++, Describe the attachment (hidden, replace)
// Change background color
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
// Change background color
renderPassColorAttachment.view = targetView;
renderPassColorAttachment.resolveTarget = nullptr;
renderPassColorAttachment.loadOp = WGPULoadOp_Clear;
renderPassColorAttachment.storeOp = WGPUStoreOp_Store;
renderPassColorAttachment.clearValue = WGPUColor{ 0.05, 0.05, 0.05, 1.0 };
#ifndef WEBGPU_BACKEND_WGPU
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
#endif // NOT WEBGPU_BACKEND_WGPU
```

````{tab} With webgpu.hpp
*结果代码：* [`step034`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step034)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step034-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step034-vanilla)
````


