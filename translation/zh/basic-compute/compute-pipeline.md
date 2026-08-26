计算管道<span class="bullet">🟡</span>
================

```{translation-warning} 译文可能已过时, /basic-compute/compute-pipeline.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step201`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step201)

渲染3D数据是GPU的原始使用,但远非现在唯一一个. 而即使是3D应用,我们有时也会使用GPU来进行非还原性的东西,如模拟,图像处理等.

当使用 GPU 时**一般用途**计算( GPGPU),我们通常**不需要调用三维特定固定部件**就像光栅化一样

此章介绍运行的骨架**计算阴影**,是指在固定功能管道外运行的荫影。

```{note}
我们并不期望你读过 整个3D 渲染部分的指南, 但至少直到结束[制服](../basic-3d-rendering/shader-uniforms/index.md)编。
```

设置
------

###简单的例子

首先让我们从非常**简单问题**: 我们有一个GPU侧缓冲器, 想要评价一个简单的函数`f`用于该缓冲器的每个元素:

```rust
/ 在工作组中
fn f(x: f32) - > f32{
返回 2.0*x+1.0; (中文(简体) ).
}
```

页:1**天真的解决办法**将这个缓冲器复制回CPU,在这里评价函数,并将结果再次上传到GPU. 但这是**效率低下**有两个原因:

- CPU - GPU - 电脑**副本**价格昂贵,特别是大型缓冲器。
- 自从`f`独立应用到每个值,问题就非常**平行**,而GPU在这种类型中比CPU要好得多.*单指令多数据*(SIMD)平行主义.

因此我们设置了一个计算遮阳器来评估`f`直接在GPU上,并将结果保存为第二个缓冲.

###建筑

为了在本章中举出例子,我们建立了一个`onCompute`函数,我们会调用一次`onInit`。您也可以删除主循环:`main.cpp`因为我们不需要互动部分。

我们重用同样的提纲 当提交我们的提交 传递`onFrame`:

```C++
void Application::onCompute() {
	// Initialize a command encoder
	Queue queue = m_device.getQueue();
	CommandEncoderDescriptor encoderDesc = Default;
	CommandEncoder encoder = m_device.createCommandEncoder(encoderDesc);

	// Create and use compute pass here!

	// Encode and submit the GPU commands
	CommandBuffer commands = encoder.finish(CommandBufferDescriptor{});
	queue.submit(commands);

	// Clean up
#if !defined(WEBGPU_BACKEND_WGPU)
	wgpuCommandBufferRelease(commands);
	wgpuCommandEncoderRelease(encoder);
	wgpuQueueRelease(queue);
#endif
}
```

在初始化方法中,我们大多保持设备初始化,并创建(私人)方法来组织不同的步骤:

```C++
bool Application::onInit() {
	if (!initDevice()) return false;
	initBindGroupLayout();
	initComputePipeline();
	initBuffers();
	initBindGroup();
	return true;
}
```

每个`initSomething`脚步与一个`terminateSomething`以反向顺序调用:

```C++
void Application::onFinish() {
	terminateBindGroup();
	terminateBuffers();
	terminateComputePipeline();
	terminateBindGroupLayout();
	terminateDevice();
}
```

计算通过
------------

记得我们画的画吗?[我们的第一个颜色](../getting-started/first-color.md)* 我们向命令队列提交了特定渲染指令序列,称为`RenderPass`。运行只计算阴影的方法类似,并使用`ComputePass`换了

计算通行证的产生是**简单得多**胜过其中的一次考验,因为我们做到了**不使用任何固定功能阶段**几乎没有什么可以配置的! 唯一的选择是时间戳写,将在基准一章中描述。

```C++
// Create compute pass
ComputePassDescriptor computePassDesc;
computePassDesc.timestampWriteCount = 0;
computePassDesc.timestampWrites = nullptr;
ComputePassEncoder computePass = encoder.beginComputePass(computePassDesc);

// Use compute pass

// Finalize compute pass
computePass.end();

// Clean up
#if !defined(WEBGPU_BACKEND_WGPU)
	wgpuComputePassEncoderRelease(computePass);
#endif
```

计算管道
----------------

一旦创建,计算通行证的使用看起来与渲染通行证的使用很像. 主要的区别是`draw`改为:`dispatchWorkgroups`,它叫我们的计算遮阳器, 没有像顶点缓冲。

```C++
// Use compute pass
computePass.setPipeline(computePipeline);
computePass.setBindGroup(/* ... */);
computePass.dispatchWorkgroups(/* ... */);
```

计算管道首先定义了要使用的遮蔽器:

```C++
// In initComputePipeline():

// Load compute shader
ShaderModule computeShaderModule = ResourceManager::loadShaderModule(RESOURCE_DIR "/compute-shader.wsl", m_device);

// Create compute pipeline
ComputePipelineDescriptor computePipelineDesc = Default;
computePipelineDesc.compute.entryPoint = "computeStuff";
computePipelineDesc.compute.module = computeShaderModule;
ComputePipeline computePipeline = m_device.createComputePipeline(computePipelineDesc);
```

文件`compute-shader.wsl`定义一个以条目命名的函数`computeStuff`并显示它是一个`@compute`。它也必须表明**工作组大小**再来一次!

```rust
@ compute @ workgroup_大小(32)
fn 计算Stuff (){
// 计算内容
}
```

```{note}
我们完全可以使用 与其它遮蔽器相同的遮蔽模块和文件, 我只是避免混合 代码中无关的部分。
```

此时此刻,只要我们没有受约束的群体,就可以援引我们的阴影:

```C++
// Use compute pass
computePass.setPipeline(computePipeline);
//computePass.setBindGroup(/* ... */);
computePass.dispatchWorkgroups(1, 1, 1);
```

哟! 除了... 它几乎什么都不做, 因为没有访问任何资源, 计算阴影器不能 通信任何输出。

资源
---------

我们的遮蔽器要与输入和输出缓冲器进行实际交流,就需要设置一个**管道布局**说明阴影资源应如何约束,**绑定组**连接用于特定阴影引用的资源。

###管道布局

在计算阴影器中,我们添加两个缓冲绑定作为变量`storage`地址空间。 必须具体说明:**访问模式**,这是`read`用于输入和`read_write`用于输出( 没有“ 仅写” 模式):

```rust
@ group( 0) @ 绑定( 0) var<storage,read>输入缓冲:数组<f32,64>;
@ group( 0) @ bonding(1) var<storage,read_write>输出缓冲: 数组<f32,64>;
```

```{note}
那个[`array`](https://gpuweb.github.io/gpuweb/wgsl/#array-types)WGSL 类型与[`std::array`](https://en.cppreference.com/w/cpp/container/array)C++ 类型。
```

我们在C++侧创建一个**绑定组布局**匹配这些绑定的:

```C++
void Application::initBindGroupLayout() {
	// Create bind group layout
	std::vector<BindGroupLayoutEntry> bindings(2, Default);

	// Input buffer
	bindings[0].binding = 0;
	bindings[0].buffer.type = BufferBindingType::ReadOnlyStorage;
	bindings[0].visibility = ShaderStage::Compute;

	// Output buffer
	bindings[1].binding = 1;
	bindings[1].buffer.type = BufferBindingType::Storage;
	bindings[1].visibility = ShaderStage::Compute;

	BindGroupLayoutDescriptor bindGroupLayoutDesc;
	bindGroupLayoutDesc.entryCount = (uint32_t)bindings.size();
	bindGroupLayoutDesc.entries = bindings.data();
	m_bindGroupLayout = m_device.createBindGroupLayout(bindGroupLayoutDesc);
}
```

内`initComputePipeline()`我们只是把这个分配到 计算管道通过**管道布局**:

```C++
// Create compute pipeline layout
PipelineLayoutDescriptor pipelineLayoutDesc;
pipelineLayoutDesc.bindGroupLayoutCount = 1;
pipelineLayoutDesc.bindGroupLayouts = (WGPUBindGroupLayout*)&m_bindGroupLayout;
m_pipelineLayout = m_device.createPipelineLayout(pipelineLayoutDesc);
computePipelineDesc.layout = m_pipelineLayout;
```

```{note}
对象`m_bindGroupLayout`和`m_pipelineLayout`属于`Application`类(因此`m_`前缀),这样可以用于不同的方法. 顺便说一句,不要忘记在终止功能中摧毁它们。
```

###缓冲

在捆绑缓冲器之前,我们当然必须创建缓冲器(在`initBuffers`) (中文(简体) ). 重要的一点是用`Storage`旗帜,这样我们就可以从阴影中读/写:

```C++
// We save this size in an attribute, it will be useful later on
m_bufferSize = 64 * sizeof(float);

// Create input buffers
BufferDescriptor bufferDesc;
bufferDesc.mappedAtCreation = false;
bufferDesc.size = m_bufferSize;
bufferDesc.usage = BufferUsage::Storage | BufferUsage::CopyDst;
Buffer inputBuffer = m_device.createBuffer(bufferDesc);


// Create output buffer: the only difference is the usage
bufferDesc.usage = BufferUsage::Storage;
Buffer outputBuffer = m_device.createBuffer(bufferDesc);
```

拥有`Storage`使用面临特定设备限制:

```C++
// We bind an input, and an output buffers:
requiredLimits.limits.maxStorageBuffersPerShaderStage = 2;

// Each buffer has (at most) the size m_bufferSize (which definition should be
// moved to the constructor so that it is known in initDevice):
requiredLimits.limits.maxStorageBufferBindingSize = m_bufferSize;
```

我们已经可以填写输入缓冲器中的一些值:

```C++
// Fill in input buffer
std::vector<float> input(m_bufferSize / sizeof(float));
for (int i = 0; i < input.size(); ++i) {
	input[i] = 0.1f * i;
}
queue.writeBuffer(inputBuffer, 0, input.data(), m_bufferSize);
```

###捆绑组

记住:捆绑组**版式**刚才说*怎么样*将资源绑在阴影上。 一旦我们有效地创建了这些资源(缓冲器),我们就能够确定**绑定组**告诉*什麽*约束:

```C++
void Application::initBindGroup() {
	// Create compute bind group
	std::vector<BindGroupEntry> entries(2, Default);

	// Input buffer
	entries[0].binding = 0;
	entries[0].buffer = m_inputBuffer;
	entries[0].offset = 0;
	entries[0].size = m_bufferSize;

	// Output buffer
	entries[1].binding = 1;
	entries[1].buffer = m_outputBuffer;
	entries[1].offset = 0;
	entries[1].size = m_bufferSize;

	BindGroupDescriptor bindGroupDesc;
	bindGroupDesc.layout = m_bindGroupLayout;
	bindGroupDesc.entryCount = (uint32_t)entries.size();
	bindGroupDesc.entries = (WGPUBindGroupEntry*)entries.data();
	m_bindGroup = m_device.createBindGroup(bindGroupDesc);
}
```

一旦捆绑组形成,它就可以被捆绑在管道上`onCompute()`:

```C++
computePass.setPipeline(computePipeline);
// Set the bind group:
computePass.setBindGroup(0, m_bindGroup, 0, nullptr);
computePass.dispatchWorkgroups(1, 1, 1);
```

召集
----------

###同时通话

现在`dispatchWorkgroups`我们来解释一下它的作用

计算阴影器(更一般的GPU)是**擅长做同样的事情 多次平行**,所以在这里建造*发送*操作是多次调用阴影点的可能性。

我们不是提供单一的并行电话数,而是将这一数字表示为**网格**(中文(简体) ).**发送**(单位:千美元)$x \times y \times z$ **工作组**:

```C++
computePass.dispatchWorkgroups(x, y, z);
```

小心发射$x \times y \times z$ **工作组**,即各组电话. 每个工作组本身就是一个小块$w \times h \times d$ **线程**,每个入口。 工作组大小由阴影器的切入点设定 :

```rust
@ 计算
@ 工作组_大小(w, d, h)
fn 计算Stuff (){
	// [...]
}
```

```{note}
工作组大小必须是恒定表达式.
```

###工作组大小对计数

好吧,那就有很多变数 只是为了设置一些工作 而那些只是他们最后的产物,不是吗?

问题是:**所有组合不等于**即使它们乘以相同的线程数。

他们的工作是**并不是全部同时发射**:在罩子下有一个调度员组织单个工作组的执行. 我们能知道的是,来自**同一工作组**一起发射,但两个**不同工作组**可能会在不同的时间被处决

适合工作组的规模**取决于任务**线条运行。 这里有一些关于拇指的规则**工作组大小对工作组数**:

- 号码$w \times h \times d$每个工作组的线程应为32的倍数,因为一个工作组的线程是由**曲线**32个线程的(通常是多个).

- 工作组的总资源使用额应保持在**最低**,以便排程者在组织事物方面有更多的自由.

- 当线程**共享内存**如果它们同属一个工作组(如果它们同属一个曲面,则更便宜)。

- 可能具有**相同的分支路径**。来自同一曲面的线条共享同一个指示指针,所以当其邻居跟随一个不同的分支时,线条会边延。`if`或循环条件。

- 尝试有工作组大小 是两个权力。

```{note}
这些规则有些矛盾。 只有具体使用案例的基准才能告诉你什么是最佳的取舍.
```

###工作组层面

OK,我现在看得更清楚了, 但是不同的斧头呢?$w$, $h$和$d$? 。 。 。 是一个工作组的规模$2 \times 2 \times 4$与$16 \times 1 \times 1$?

这确实不同,因为这个尺寸**给硬件提示**关于潜力**内存访问的一致性**穿过线条

CPU 和 GPU 都在一般情况下尝试在连续和/或同时操作使用内存的方式中猜测模式,例如为了[预选](https://en.wikipedia.org/wiki/Cache_prefetching)存储在缓存中或组合(a.k.a."coalesce")同时读/写为单个内存访问.

由于GPU的一个非常常见的任务是处理**作为二维或三维网格排列的数据**,一个图形 API 提供网格数据存储(**纹理**)和基于网格的货币模型。 当邻居线程以类似的方式访问邻居像素/voxels时,硬件可以更好地预测正在发生的事情.

所以这里主要的拇指规则是,虽然$x$, $y$和$z$斧头乍一看就是抽象的值,它们"只是"相乘,你应该真的把它们当作$x$, $y$和$z$您的数据网轴 。

###示例

以我们简单的例子,我们处理1D缓冲中设定的数据,所以我们的发送也是一维系列的工作组:$(x, y, z) = (x, 1, 1)$和$(w, h, d) = (w, 1, 1)$.

工作组大小$w$应该至少是32岁,而且没有明显的理由可以超越这一点。 最终我们派出工作组`32 * 1 * 1`线程 :

```rust
@ compute @ workgroup_大小( 32, 1, 1) / 或只是@ workgroup_大小(32)
fn 计算Stuff (){
    // [...]
}
```

我们从预期的援引呼吁中推断出工作组的数目:

```C++
uint32_t invocationCount = m_bufferSize / sizeof(float);
uint32_t workgroupSize = 32;
// This ceils invocationCount / workgroupSize
uint32_t workgroupCount = (invocationCount + workgroupSize - 1) / workgroupSize;
computePass.dispatchWorkgroups(workgroupCount, 1, 1);
```

```{note}
当心天花板`invocationCount / workgroupSize`而不是放在地上,否则`workgroupSize`不完全区分`invocationCount`最后的线索将丢失。
```

我们现在需要的就是知道我们属于哪个工作组, 找出我们需要处理的缓冲指数。 这是由[内置阴影输入](https://gpuweb.github.io/gpuweb/wgsl/#built-in-values-global_invocation_id),特别是**援引 id**原文照发`global_invocation_id`内置 :

```rust
@ compute @ workgroup_大小(32)
fn 计算Stuff (@ buildingin (global))_援引_编号) 编号:vec3<u32>) {
/ 将函数 f 应用到索引 id.x 中的缓冲元素:
输出缓冲[页:1]=f(输入压力)[页:1]);
}
```

###设备限制

与选择工作组大小/计算相关的设备限制有:

```C++
// The maximum value for respectively w, h and d
requiredLimits.limits.maxComputeWorkgroupSizeX = 32;
requiredLimits.limits.maxComputeWorkgroupSizeY = 1;
requiredLimits.limits.maxComputeWorkgroupSizeZ = 1;

// The maximum value of the product w * h * d
requiredLimits.limits.maxComputeInvocationsPerWorkgroup = 32;

// And the maximum value of max(x, y, z)
// (It is 2 because workgroupCount = 64 / 32 = 2)
requiredLimits.limits.maxComputeWorkgroupsPerDimension = 2;
```

读回
---------

发送所有平行计算线程后,输出缓冲器会随结果而存在. 所以现在我们自然想要**读取此输出缓冲器**回来。

```{note}
在GPU上计算事物的一个点是避免CPU-GPU复制,因为也许输出缓冲器只用于GPU上的后续操作. 但是,就我们的例子而言,我们仍然想检查一下计算是否顺利。
```

###地图缓冲

我们已经看到如何使用缓冲器`mapAsync`读取缓冲器回移的方法,但这不会直接起作用:

```C++
// DON'T
m_outputBuffer.mapAsync(MapMode::Read, /* ... */);
```

为什么不呢? 这需要使用此程序创建输出缓冲器`MapRead`使用标记。 不过很不幸**此旗帜不兼容**与`Storage`,用于允许阴影器在输出中写入。

解决办法是**创建第三个缓冲器**负责CPU的运输 在那个`initBuffers()`我们创建新的“地图缓冲”并添加`CopySrc`输出的用法 :

```C++
// Add the CopySrc usage here, so that we can copy to the map buffer
bufferDesc.usage = BufferUsage::Storage | BufferUsage::CopySrc;
m_outputBuffer = m_device.createBuffer(bufferDesc);

// Create an intermediary buffer to which we copy the output and that can be
// used for reading into the CPU memory.
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::MapRead;
m_mapBuffer = m_device.createBuffer(bufferDesc);
```

之后`computePass.end()`,并在此之前`encoder.finish(...)`,我们添加一个复制命令:

```C++
// Copy the memory from the output buffer that lies in the storage part of the
// memory to the map buffer, which is in the "mappable" part of the memory.
encoder.copyBufferToBuffer(m_outputBuffer, 0, m_mapBuffer, 0, m_bufferSize);
```

###回电

我们现在准备通过给`mapAsync`:

```C++
// Print output
bool done = false;
auto handle = m_mapBuffer.mapAsync(MapMode::Read, 0, m_bufferSize, [&](BufferMapAsyncStatus status) {
	if (status == BufferMapAsyncStatus::Success) {
		const float* output = (const float*)m_mapBuffer.getConstMappedRange(0, m_bufferSize);
		for (int i = 0; i < input.size(); ++i) {
			std::cout << "input " << input[i] << " became " << output[i] << std::endl;
		}
		m_mapBuffer.unmap();
	}
	done = true;
});
```

别忘了打电话`Instance::processEvents`在等待地图完成的循环中:

```C++
while (!done) {
	// Checks for ongoing asynchronous operations and call their callbacks if needed
	m_instance.processEvents();
}
```

````{caution}
截止2023年4月23日,`wgpu-native`未执行`processEvent`但它的行为可以通过提交空队列来模仿:

```C++
#ifdef WEBGPU_BACKEND_WGPU
		queue.submit(0, nullptr);
#else
		m_instance.processEvents();
#endif
```
````

你最终应该在输出控制台上看到这样的东西:

```
输入 0 改为 1
投入0.1改为1.2
投入0.2改为1.4
投入0.3改为1.6
投入0.4改为1.8
[...]
```

结论
----------

本章的一些部分提醒人们注意,在提交通行证方面做了哪些工作。**最重要的新东西**这里是调度/ 工作组/ 线程等级 。 确保定期回到规则清单中,以检查工作组规模的选择是否相关(并尽可能制定基准)。

我们现在准备专注于计算阴影器本身的内容,以及它可以操纵资源和记忆的不同方式!

*结果代码 :* [`step201`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step201)
