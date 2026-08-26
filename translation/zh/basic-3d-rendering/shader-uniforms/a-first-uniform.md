第一身军服<span class="bullet">🟢</span>
===============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/shader-uniforms/a-first-uniform.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/039 - A first uniform - vanilla
:parent: 037 - Loading from file - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/039 - A first uniform
:parent: 037 - Loading from file
```

````{tab} With webgpu.hpp
*结果代码 :* [`step039`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step039)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step039-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step039-vanilla)
````

导言
------------

###动机

如果我们看看现在的阴影,我们可以看到一些**硬码**常数 :

```rust
百分率=640.0/480.0;
页:1
```

这不是非常满意的,当我们想要在应用程序执行过程中移动对象时会发生什么? 或因用户调整窗口大小而改变比例?

> QQ 我们可以动态更改阴影码,重建一个新的阴影模块!

它会工作,但**建立阴影模块需要时间**。很多时间,如果我们将其与预算比较,用于制作单一框架(通常为60秒)。 想象我们想**动画**我们的场景,从而改变`offset`每个框!

这就是为什么一个适当的解决方案是使用**统一变量**.

###定义

制服是阴影中的一个全局变量,其值由GPU缓冲器加载. 我们说这是**绑定**到缓冲器。

它的价值是**制服**跨越不同的顶点和碎片**给定的呼叫**改为`draw`,但可以通过**更新**缓冲器的值*绑定*改为。

要使用制服,我们必须:

1. 宣布《公约》中的制服**阴影**.
2. 创建**缓冲**这是必然的。
3. 配置**绑定**(a.k.a. 约束**版式**).
4. 创建一个**绑定组**.

###设备限制

在本章中,我们将要求为我们的装置设定以下限制:

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
// We use at most 1 bind group for now
requiredLimits.limits.maxBindGroups = 1;
// We use at most 1 uniform buffer per stage
requiredLimits.limits.maxUniformBuffersPerShaderStage = 1;
// Uniform structs have a size of maximum 16 float (more than what we need)
requiredLimits.limits.maxUniformBufferBindingSize = 16 * 4;
```

阴影侧
-----------

为了激活我们的场景,我们造了一套制服`uTime`我们按当前时间更新每个框架,以秒表示(按`glfwGetTime()`).

```{note}
我通常会**前缀**带有“ u” 的一致变量, 以方便在读取长阴影器时查找**当变量是统一时**而不是本地变量。
```

正如我所说,制服是一个全球变数,因此我们可以在我们的遮阳镜的第一行宣布。 WGSL中一个简单的变量声明外观如下:

```{lit} rust, Declare uniforms (also for tangle root "Vanilla")
// Simple variable declaration
var uTime: f32;
```

那个`var`关键字可以标签为**地址空间**,用于控制 GPU 中如何存储变量(see [Memory Model](/appendices/memory-model.md)) (中文(简体) ). 在这里,我们声明变量存储在*制服*地址空间 :

```{lit} rust, Declare uniforms (replace, also for tangle root "Vanilla")
// Variable in the *uniform* address space
var<uniform> uTime: f32;
```

现在我们需要告诉谁**缓冲**军服是**绑定**。为此,请指定**绑定索引**与`@binding(...)`WGSL属性.

```{lit} rust, Declare uniforms (replace, also for tangle root "Vanilla")
// Specify which binding index the uniform is attached to
@binding(0) var<uniform> uTime: f32;
```

然后在C++代码中定义绑定物的缓冲#0,实际上,绑定在**约束群体**,因此,制服的约束性通过同时提供`@group(...)`属性 :

```{lit} rust, Declare uniforms (replace, also for tangle root "Vanilla")
// The memory location of the uniform is given by a pair of a *bind group* and a *binding*
@group(0) @binding(0) var<uniform> uTime: f32;
```

我们现在完成了对统一变量的宣布,我们可以像其它变量一样在我们的阴影中使用:

```{lit} rust, Shader prelude (append, also for tangle root "Vanilla")
// We add the declaration of 'uTime' to the shader prelude
{{Declare uniforms}}
```

```{lit} rust, Vertex shader (replace, also for tangle root "Vanilla")
fn vs_main(in: VertexInput) -> VertexOutput {
	var out: VertexOutput;
	let ratio = 640.0 / 480.0;

	// We now move the scene depending on the time!
	var offset = vec2f(-0.6875, -0.463);
	offset += 0.3 * vec2f(cos(uTime), sin(uTime));

	out.position = vec4f(in.position.x + offset.x, (in.position.y + offset.y) * ratio, 0.0, 1.0);
	out.color = in.color;
	return out;
}
```

```{hint}
如果你们不熟悉[三角函数](https://en.wikipedia.org/wiki/Trigonometric_functions)喜欢`cos`和`sin`,注意立场$(\cos(a), \sin(a))$是半径圆上的点$1$角度$a$(用弧度表示). 因此,这个公式使得三角形**沿着圆圈移动**随着时间的推移。 乘以数$0.3$以缩小这个圆的半径。
```

统一缓冲器
--------------

统一缓冲器和任何其他缓冲器一样被创建,但我们必须具体说明`BufferUsage::Uniform`联合国`usage`字段。 我们只需要它 包含一个浮点,但现在,**缓冲大小需要调整到 16 字节**,因此我们创建了4个浮点的缓冲器(1个浮点使用4字节).

我们首先宣布`uniformBuffer`输入`Application`类属性 :

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	Buffer uniformBuffer;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUBuffer uniformBuffer;
```
````

然后在初始化时创建缓冲器 :

````{tab} With webgpu.hpp
```{lit} C++, Create uniform buffer
// Create uniform buffer (reusing bufferDesc from other buffer creations)
// The buffer will only contain 1 float with the value of uTime
// then 3 floats left empty but needed by alignment constraints
bufferDesc.size = 4 * sizeof(float);

// Make sure to flag the buffer as BufferUsage::Uniform
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Uniform;

bufferDesc.mappedAtCreation = false;
uniformBuffer = device.createBuffer(bufferDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create uniform buffer (for tangle root "Vanilla")
// Create uniform buffer (reusing bufferDesc from other buffer creations)
// The buffer will only contain 1 float with the value of uTime
// then 3 floats left empty but needed by alignment constraints
bufferDesc.size = 4 * sizeof(float);

// Make sure to flag the buffer as BufferUsage::Uniform
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Uniform;

bufferDesc.mappedAtCreation = false;
uniformBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);
```
````

然后使用`Queue::writeBuffer`在缓冲器的第一个浮点上传一个值:

````{tab} With webgpu.hpp
```{lit} C++, Upload uniform values
float currentTime = 1.0f;
queue.writeBuffer(uniformBuffer, 0, &currentTime, sizeof(float));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Upload uniform values (for tangle root "Vanilla")
float currentTime = 1.0f;
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &currentTime, sizeof(float));
```
````

总而言之,我们把这个放在`Application::InitializeBuffers()`:

```{lit} C++, InitializeBuffers method (replace, also for tangle root "Vanilla")
void Application::InitializeBuffers() {
	// 1. Load from disk into CPU-side vectors pointData and indexData
	{{Load geometry data from file}}

	// 2. Create GPU buffers and upload data to them
	{{Create point buffer}}
	{{Create index buffer}}

	// 3. Create and fill uniform buffer <-- HERE
	{{Create uniform buffer}}
	{{Upload uniform values}}
}
```

`````{note}
不要忘记释放这些缓冲器`Terminate()`方法 :

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
uniformBuffer.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
wgpuBufferRelease(uniformBuffer);
```
````
`````

装订配置
---------------------

如果您现在尝试运行您的程序, 您将会碰到一个设备错误, 说明阴影中引用的一些绑定组是**不兼容**带有预期绑定组布局。 这是什么意思?

缓冲器与制服的实际连接有两个步骤. 一个是**在编审中申报**我们想要一个约束, 如何准确。 这是束缚**版式**。第二个是创建绑定组并启用它(见下一节)。

```{note}
这遵循了与区分*顶点缓冲*页:1*顶点缓冲布局*.
```

###管道布局

绑定布局通过`PipelineLayout`管道描述符的一部分 管道布局如何描述**资源**制导管道必须捆绑。

一个资源要么是一个纹理,要么是一个缓冲,其布局指定了它与哪个索引绑定,以及像它作为只读或只写访问一样的属性等.

迄今为止,我们用自动布局来设置`pipelineDesc.layout`但是从现在起,我们将明确宣布我们的管道所期望的资源:

```{lit} C++, Describe pipeline layout (replace, also for tangle root "Vanilla")
{{Create pipeline layout}}

// Assign the PipelineLayout to the RenderPipelineDescriptor's layout field
pipelineDesc.layout = layout;
```

当创建此管道布局时, 我们可以定义多个**约束群体**,而绑定组包含多个**约束**:

````{tab} With webgpu.hpp
```{lit} C++, Create pipeline layout
{{Define bindingLayout}}

// Create a bind group layout
BindGroupLayoutDescriptor bindGroupLayoutDesc{};
bindGroupLayoutDesc.entryCount = 1;
bindGroupLayoutDesc.entries = &bindingLayout;
bindGroupLayout = device.createBindGroupLayout(bindGroupLayoutDesc);

// Create the pipeline layout
PipelineLayoutDescriptor layoutDesc{};
layoutDesc.bindGroupLayoutCount = 1;
layoutDesc.bindGroupLayouts = (WGPUBindGroupLayout*)&bindGroupLayout;
layout = device.createPipelineLayout(layoutDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create pipeline layout (for tangle root "Vanilla")
{{Define bindingLayout}}

// Create a bind group layout
WGPUBindGroupLayoutDescriptor bindGroupLayoutDesc{};
bindGroupLayoutDesc.nextInChain = nullptr;
bindGroupLayoutDesc.entryCount = 1;
bindGroupLayoutDesc.entries = &bindingLayout;
bindGroupLayout = wgpuDeviceCreateBindGroupLayout(device, &bindGroupLayoutDesc);

// Create the pipeline layout
WGPUPipelineLayoutDescriptor layoutDesc{};
layoutDesc.nextInChain = nullptr;
layoutDesc.bindGroupLayoutCount = 1;
layoutDesc.bindGroupLayouts = &bindGroupLayout;
layout = wgpuDeviceCreatePipelineLayout(device, &layoutDesc);
```
````

重要的是,像任何用`createSomething`程序、布局**必须释放**一旦我们不再使用它们。 因此,我们把它们定义为阶级属性。

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	PipelineLayout layout;
	BindGroupLayout bindGroupLayout;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUPipelineLayout layout;
	WGPUBindGroupLayout bindGroupLayout;
```
````

然后我们就可以释放他们`Application::Terminate()`:

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
layout.release();
bindGroupLayout.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
wgpuPipelineLayoutRelease(layout);
wgpuBindGroupLayoutRelease(bindGroupLayout);
```
````

保留`bindGroupLayout`当我们建立约束性团体时,也需要这样做。

###装订布局

那个`BindGroupLayoutEntry`本来可以叫`BindingLayout`。第一个设置是绑定索引,用于阴影`@binding`属性。 接下来`visibility`字段说明哪个阶段需要访问此资源,以便不无必要地向所有阶段提供。

````{tab} With webgpu.hpp
```{lit} C++, Define bindingLayout
// Define binding layout (don't forget to = Default)
BindGroupLayoutEntry bindingLayout = Default;

// The binding index as used in the @binding attribute in the shader
bindingLayout.binding = 0;

// The stage that needs to access this resource
bindingLayout.visibility = ShaderStage::Vertex;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Define bindingLayout (for tangle root "Vanilla")
// Define binding layout
WGPUBindGroupLayoutEntry bindingLayout{};
setDefault(bindingLayout);

// The binding index as used in the @binding attribute in the shader
bindingLayout.binding = 0;

// The stage that needs to access this resource
bindingLayout.visibility = WGPUShaderStage_Vertex;
```

我建议你创建一个功能 初始化绑定组布局条目 某处,就像我们所做的`WGPULimits`:

```{lit} C++, setDefault for WGPUBindGroupLayoutEntry (for tangle root "Vanilla")
void setDefault(WGPUBindGroupLayoutEntry &bindingLayout) {
	bindingLayout.buffer.nextInChain = nullptr;
	bindingLayout.buffer.type = WGPUBufferBindingType_Undefined;
	bindingLayout.buffer.hasDynamicOffset = false;

	bindingLayout.sampler.nextInChain = nullptr;
	bindingLayout.sampler.type = WGPUSamplerBindingType_Undefined;

	bindingLayout.storageTexture.nextInChain = nullptr;
	bindingLayout.storageTexture.access = WGPUStorageTextureAccess_Undefined;
	bindingLayout.storageTexture.format = WGPUTextureFormat_Undefined;
	bindingLayout.storageTexture.viewDimension = WGPUTextureViewDimension_Undefined;

	bindingLayout.texture.nextInChain = nullptr;
	bindingLayout.texture.multisampled = false;
	bindingLayout.texture.sampleType = WGPUTextureSampleType_Undefined;
	bindingLayout.texture.viewDimension = WGPUTextureViewDimension_Undefined;
}
```

```{lit} C++, InitializePipeline method (prepend, hidden, for tangle root "Vanilla")
// We place this before InitializePipeline
{{setDefault for WGPUBindGroupLayoutEntry}}
```
````

绑定布局的剩余部分取决于资源类型. 我们要么填补`buffer`字段或`sampler`和`texture`字段或`storageTexture`字段。 就我们而言,这是一个缓冲:

````{tab} With webgpu.hpp
```{lit} C++, Define bindingLayout (append)
bindingLayout.buffer.type = BufferBindingType::Uniform;
bindingLayout.buffer.minBindingSize = 4 * sizeof(float);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Define bindingLayout (append, for tangle root "Vanilla")
bindingLayout.buffer.type = WGPUBufferBindingType_Uniform;
bindingLayout.buffer.minBindingSize = 4 * sizeof(float);
```
````

```{important}
注意我如何初始化绑定的布局对象`= Default`见上文。 这一点很重要,因为它特别规定了:`buffer`, `sampler`, `texture`和`storageTexture`用于`Undefined`因此只使用我们设置的资源类型。
```

捆绑组
----------

###创建

约束组包含实际约束. 绑定组的结构与渲染管中所规定的绑定组布局相映射,并实际将其与资源连接. 这样,同样的管道可以重复使用,如同根据资源的不同变体在多次抽调时一样。

绑定组再次成为我们初始化后想要保存的对象(我们在主循环中使用),因此在**类级**:

````{tab} With webgpu.hpp
```{lit} C++, Application attributes (append)
private: // Application attributes
	BindGroup bindGroup;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Application attributes (append, for tangle root "Vanilla")
private: // Application attributes
	WGPUBindGroup bindGroup;
```
````

绑定组必须遵循与其布局相同的结构:

````{tab} With webgpu.hpp
```{lit} C++, Create bind group
// Create a binding
BindGroupEntry binding{};
{{Setup binding}}

// A bind group contains one or multiple bindings
BindGroupDescriptor bindGroupDesc{};
bindGroupDesc.layout = bindGroupLayout;
// There must be as many bindings as declared in the layout!
bindGroupDesc.entryCount = 1;
bindGroupDesc.entries = &binding;
bindGroup = device.createBindGroup(bindGroupDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create bind group (for tangle root "Vanilla")
// Create a binding
WGPUBindGroupEntry binding{};
binding.nextInChain = nullptr;
{{Setup binding}}

// A bind group contains one or multiple bindings
WGPUBindGroupDescriptor bindGroupDesc{};
bindGroupDesc.nextInChain = nullptr;
bindGroupDesc.layout = bindGroupLayout;
// There must be as many bindings as declared in the layout!
bindGroupDesc.entryCount = 1;
bindGroupDesc.entries = &binding;
bindGroup = wgpuDeviceCreateBindGroup(device, &bindGroupDesc);
```
````

然后我们把绑起来的队伍放进去`Application::Terminate()`:

````{tab} With webgpu.hpp
```{lit} C++, Terminate (prepend)
bindGroup.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Terminate (prepend, for tangle root "Vanilla")
wgpuBindGroupRelease(bindGroup);
```
````

约束本身的内容相当直截了当:

```{lit} C++, Setup binding (also for tangle root "Vanilla")
// The index of the binding (the entries in bindGroupDesc can be in any order)
binding.binding = 0;
// The buffer it is actually bound to
binding.buffer = uniformBuffer;
// We can specify an offset within the buffer, so that a single buffer can hold
// multiple uniform blocks.
binding.offset = 0;
// And we specify again the size of the buffer.
binding.size = 4 * sizeof(float);
```

```{note}
田园`binding.sampler`和`binding.textureView`只有在绑定布局使用时才需要。
```

请注意,与建立约束集团不同**版式**,必须建立约束团体**资源创建后**把它绑起来! 因此,我建议我们采用一种新的初始化方法:

```{lit} C++, InitializeBindGroups method (also for tangle root "Vanilla")
void Application::InitializeBindGroups() {
	{{Create bind group}}
}
```

```{lit} C++, Application implementation (append, hidden, also for tangle root "Vanilla")
// Add this in the main file
{{InitializeBindGroups method}}
```

同往常一样,我们在申请声明中宣布了这种方法:

```{lit} C++, Private methods (append, also for tangle root "Vanilla")
private: // Application methods
	void InitializeBindGroups();
```

然后我们开始组装**之后**其他东西:

```{lit} C++, Initialize (append, also for tangle root "Vanilla")
// At the end of Initialize()
InitializeBindGroups();
```

###使用量

好了,我们现在准备连接点! 它与设定绑定组使用一样简单**在绘图调用前**时`Application::MainLoop()`:

````{tab} With webgpu.hpp
```{lit} C++, Use Render Pass (replace)
renderPass.setPipeline(pipeline);
renderPass.setVertexBuffer(0, pointBuffer, 0, pointBuffer.getSize());
renderPass.setIndexBuffer(indexBuffer, IndexFormat::Uint16, 0, indexBuffer.getSize());

// Set binding group here!
renderPass.setBindGroup(0, bindGroup, 0, nullptr);

renderPass.drawIndexed(indexCount, 1, 0, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Use Render Pass (replace, for tangle root "Vanilla")
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);
wgpuRenderPassEncoderSetVertexBuffer(renderPass, 0, pointBuffer, 0, wgpuBufferGetSize(pointBuffer));
wgpuRenderPassEncoderSetIndexBuffer(renderPass, indexBuffer, WGPUIndexFormat_Uint16, 0, wgpuBufferGetSize(indexBuffer));

// Set binding group here!
wgpuRenderPassEncoderSetBindGroup(renderPass, 0, bindGroup, 0, nullptr);

wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);
```
````

它应该已经起作用了,但不能移动 因为统一缓冲器的内容**尚未更改**。我们需要在主要循环中更新它们:

````{tab} With webgpu.hpp
```{lit} C++, Update uniform buffer
// Update uniform buffer
float t = static_cast<float>(glfwGetTime()); // glfwGetTime returns a double
queue.writeBuffer(uniformBuffer, 0, &t, sizeof(float));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Update uniform buffer (for tangle root "Vanilla")
// Update uniform buffer
float t = static_cast<float>(glfwGetTime()); // glfwGetTime returns a double
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &t, sizeof(float));
```
````

比如说,我们把它放在开始的时候`Application::MainLoop()`:

```{lit} C++, Main loop content (prepend, also for tangle root "Vanilla")
// Right after glfwPollEvents():
{{Update uniform buffer}}
```

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../../_static/turning-webgpu-logo.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">我们的第一个动态场景!</span></p>
	</figcaption>
</figure>

````{tab} With webgpu.hpp
*结果代码 :* [`step039`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step039)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step039-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step039-vanilla)
````
