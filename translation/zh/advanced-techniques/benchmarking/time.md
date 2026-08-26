时间<span class="bullet">🟡</span>
====

```{translation-warning} 译文可能已过时, /advanced-techniques/benchmarking/time.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step095-timestamp-queries`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-timestamp-queries)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step095-timestamp-queries-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-timestamp-queries-vanilla)
````

我们从**测量计算时间**开始，这通常是最有价值的资源。

```{warning}
截至2023年9月6日，wgpu-native尚不支持时间戳查询。我建议你暂时只关注 Dawn 的这一章。
```

异步性
--------------

重要的是，测量 GPU 时间与测量 CPU 时间**完全不同**，因为您可能还记得**我们仅通过 CPU 代码 (C++) 中发出的远程调用**与 GPU 交互。

在命令式 CPU 代码中，测量时间如下所示：

```python
# Pseudocode of a simple CPU benchmarking
start_time = get_current_time()
do_on_cpu(something)
end_time = get_current_time()
ellapsed = end_time - start_time
```

但在 GPU 上执行此操作时，我们仅**提交**在**不同时间轴**上运行的操作：

```python
# Pseudocode of a **wrong** GPU benchmarking
start_time = get_current_time()
submit_to_do_on_gpu(something)
end_time = get_current_time()
ellapsed = end_time - start_time # wrong!
```

“某件事”此时可能还没有开始。我们测量的是提交指令所需的时间，**而不是实际执行指令**！

时间戳查询
-----------------

我们必须指示 GPU 在自己的时间线上运行一些相当于 `get_current_time()` 的东西。此操作的结果存储在称为**时间戳查询**的专用对象中。

```python
# Pseudocode of a correct GPU benchmarking
start_timestamp_query = create_timestamp_query()
end_timestamp_query = create_timestamp_query()
submit_to_do_on_gpu(write_current_time, start_timestamp_query)
submit_to_do_on_gpu(something)
submit_to_do_on_gpu(write_current_time, end_timestamp_query)
```

然后，我们必须通过映射缓冲区将时间戳值**取**回CPU，就像我们在[使用缓冲区](../../basic-3d-rendering/input-geometry/playing-with-buffers.md#mapping-context)中看到的那样。

> 🫡 好的，明白了，那么实际的 C++ 代码呢？

无论测量时间戳还是其他内容，GPU 查询都存储在 `QuerySet` 中。我们通常将开始时间和结束时间存储在同一组中：

````{tab} With webgpu.hpp
```C++
// Create timestamp queries
QuerySetDescriptor querySetDesc;
querySetDesc.type = QueryType::Timestamp;
querySetDesc.count = 2; // start and end
QuerySet timestampQueries = device.createQuerySet(querySetDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create timestamp queries
WGPUQuerySetDescriptor querySetDesc;
querySetDesc.nextInChain = nullptr;
querySetDesc.type = WGPUQueryType_Timestamp;
querySetDesc.count = 2; // start and end
WGPUQuerySet timestampQueries = wgpuDeviceCreateQuerySet(device, &querySetDesc);
```
````

```{note}
我将此示例基于 [`step095`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095)，来自章节 [简单 GUI](../../basic-3d-rendering/some-interaction/simple-gui.md)]
```

```{note}
我创建了一个 `initBenchmark()` 方法（在 `onInit` 中调用）来初始化与基准测试相关的对象，例如查询集。我还创建了一个 `terminateBenchmark()` 来释放这些资源。
```

但是，如果您尝试将上面的代码块添加到您的应用程序中，您将面临错误：

> **设备错误：**
> *(Dawn)* 时间戳查询是不允许的，因为它们可能会暴露精确的计时信息。
> *(wgpu-native)* 功能(TIMESTAMP_QUERY) 是必需的，但设备上未启用。

启用时间戳功能
--------------------------

### 黎明切换

让我们从 Dawn 开始：出于**隐私原因**，Dawn 禁用计时信息。这在 Web 上运行时是相关的，但在我们的本机应用程序上下文中则无关。幸运的是，**此保护措施可以轻松禁用**。

Dawn 有一个所谓的“切换”列表，可以在整个 WebGPU 实例的范围内打开或关闭：该列表可在 [`Toggles.cpp`](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/dawn/native/Toggles.cpp#33).

为了启用切换，我们使用 Dawn 特定的 `DawnTogglesDescriptor` **扩展**，它可以链接到实例描述符：

````{tab} With webgpu.hpp
```C++
// At the very beginning of onInit()
InstanceDescriptor instanceDesc;
#ifdef WEBGPU_BACKEND_DAWN
// Dawn-specific extension to enable/disable toggles
DawnTogglesDescriptor dawnToggles;
// [...]
instanceDesc.nextInChain = &dawnToggles.chain;
#endif
m_instance = createInstance(instanceDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// At the very beginning of onInit()
WGPUInstanceDescriptor instanceDesc;
#ifdef WEBGPU_BACKEND_DAWN
// Dawn-specific extension to enable/disable toggles
WGPUDawnTogglesDescriptor dawnToggles;
// [...]
instanceDesc.nextInChain = &dawnToggles.chain;
#endif
m_instance = wgpuCreateInstance(&instanceDesc);
```
````

然后我们指定要启用 `allow_unsafe_apis` 功能：

````{tab} With webgpu.hpp
```C++
#ifdef WEBGPU_BACKEND_DAWN
DawnTogglesDescriptor dawnToggles;
dawnToggles.chain.next = nullptr;
dawnToggles.chain.sType = SType::DawnTogglesDescriptor;

std::vector<const char*> enabledToggles = {
    "allow_unsafe_apis",
};
dawnToggles.enabledToggles = enabledToggles.data();
dawnToggles.enabledTogglesCount = enabledToggles.size();
dawnToggles.disabledTogglesCount = 0;

instanceDesc.nextInChain = &dawnToggles.chain;
#endif
```
````

````{tab} Vanilla webgpu.h
```C++
#ifdef WEBGPU_BACKEND_DAWN
WGPUDawnTogglesDescriptor dawnToggles;
dawnToggles.chain.next = nullptr;
dawnToggles.chain.sType = WGPUSType_DawnTogglesDescriptor;

std::vector<const char*> enabledToggles = {
    "allow_unsafe_apis",
};
dawnToggles.enabledToggles = enabledToggles.data();
dawnToggles.enabledTogglesCount = enabledToggles.size();
dawnToggles.disabledTogglesCount = 0;

instanceDesc.nextInChain = &dawnToggles.chain;
#endif
```
````

```{note}
切换描述符还可以用作适配器或设备请求选项的扩展。在这种情况下，设备切换将取代适配器切换，适配器切换将取代实例切换。
```

我们收到的错误消息现在略有不同：

> **设备错误：**
> *(Dawn)* 在未启用该功能的情况下创建的时间戳查询集。

这本质上与上面 `wgpu-native` 报告的错误相同，我们将在下一节中处理这两个错误。

### 功能请求

在创建 WebGPU 设备时，我们已经提到我们可以设置特定的限制。但我们也可以从 `WGPUFeatureName` 枚举中**请求特定功能**。特别是，我们需要启用 `FeatureName::TimestampQuery`。

````{tab} With webgpu.hpp
```C++
std::vector<FeatureName> requiredFeatures = {
	FeatureName::TimestampQuery,
};
deviceDesc.requiredFeatures = (const WGPUFeatureName*)requiredFeatures.data();
deviceDesc.requiredFeaturesCount = (uint32_t)requiredFeatures.size();
```
````

````{tab} Vanilla webgpu.h
```C++
std::vector<WGPUFeatureName> requiredFeatures = {
	WGPUFeatureName_TimestampQuery,
};
deviceDesc.requiredFeatures = requiredFeatures.data();
deviceDesc.requiredFeaturesCount = (uint32_t)requiredFeatures.size();
```
````

现在应该修复错误消息了！您可能还需要检查适配器和设备是否支持此功能：

````{tab} With webgpu.hpp
```C++
std::vector<FeatureName> requiredFeatures;
if (m_adapter.hasFeature(FeatureName::TimestampQuery)) {
	requiredFeatures.push_back(FeatureName::TimestampQuery);
}
// [...] Create device
if (!m_device.hasFeature(FeatureName::TimestampQuery)) {
	std::cout << "Timestamp queries are not supported!" << std::endl;
}
```
````

````{tab} Vanilla webgpu.h
```C++
std::vector<WGPUFeatureName> requiredFeatures;
if (wgpuAdapterHasFeature(m_adapter, WGPUFeatureName_TimestampQuery)) {
	requiredFeatures.push_back(WGPUFeatureName_TimestampQuery);
}
// [...] Create device
if (!wgpuDeviceHasFeature(m_device, WGPUFeatureName_TimestampQuery)) {
	std::cout << "Timestamp queries are not supported!" << std::endl;
}
```
````

```{note}
时间戳查询被指定为显式功能，因为某些设备/适配器可能不支持它们。
```

写入时间戳
------------------

将时间戳写入查询的方法有多种。与上面的伪代码最接近的是 `commandEncoder.writeTimestamp()`，每当在 **GPU 时间轴**中执行命令时，它都会将 GPU 端时间写入查询中。

如果您想更具体地测量**渲染或计算通道**所花费的时间，您还可以将时间戳查询传递给通道描述符：

````{tab} With webgpu.hpp
```C++
// In Application::onFrame()
std::vector<RenderPassTimestampWrite> timestampWrites(2);
timestampWrites[0].location = RenderPassTimestampLocation::Beginning;
timestampWrites[0].querySet = m_timestampQueries;
timestampWrites[0].queryIndex = 0; // first query = start time
timestampWrites[1].location = RenderPassTimestampLocation::End;
timestampWrites[1].querySet = m_timestampQueries;
timestampWrites[1].queryIndex = 1; // second query = end time

renderPassDesc.timestampWriteCount = (uint32_t)timestampWrites.size();
renderPassDesc.timestampWrites = timestampWrites.data();
RenderPassEncoder renderPass = encoder.beginRenderPass(renderPassDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// In Application::onFrame()
std::vector<WGPURenderPassTimestampWrite> timestampWrites(2);
timestampWrites[0].location = WGPURenderPassTimestampLocation_Beginning;
timestampWrites[0].querySet = m_timestampQueries;
timestampWrites[0].queryIndex = 0; // first query = start time
timestampWrites[1].location = WGPURenderPassTimestampLocation_End;
timestampWrites[1].querySet = m_timestampQueries;
timestampWrites[1].queryIndex = 1; // second query = end time

renderPassDesc.timestampWriteCount = (uint32_t)timestampWrites.size();
renderPassDesc.timestampWrites = timestampWrites.data();
WGPURenderPassEncoder renderPass = wgpuCommandEncoderBeginRenderPass(encoder, &renderPassDesc);
```
````

```{note}
我仅初始化查询集一次，并将其存储到 `Application` 类的属性 `m_timestampQueries` 中。
```

读取时间戳
------------------

### 解析时间戳

好的，渲染通道在开始时写入我们的第一个查询，并在结束时写入第二个查询。我们现在只需要计算差异，对吗？但时间戳仍然**存在于 GPU 内存中**，因此我们首先需要**将它们取回**到 CPU。

第一步是**解决**查询。这可以从 WebGPU 实现用于存储查询集的任何内部表示形式获取时间戳值，并将它们写入 **GPU 缓冲区**。

````{tab} With webgpu.hpp
```C++
// We create a method dedicated to fetching timestamps
void Application::resolveTimestamps(CommandEncoder encoder) {
	// Resolve the timestamp queries (write their result to the resolve buffer)
	encoder.resolveQuerySet(
		m_timestampQueries,
		0, 2, // get queries 0 to 0+2
		m_timestampResolveBuffer,
		0 // offset in resolve buffer
	);
}
```
````

````{tab} Vanilla webgpu.h
```C++
// We create a method dedicated to fetching timestamps
void Application::resolveTimestamps(WGPUCommandEncoder encoder) {
	// Resolve the timestamp queries (write their result to the resolve buffer)
	wgpuCommandEncoderResolveQuerySet(
		encoder,
		m_timestampQueries,
		0, 2, // get queries 0 to 0+2
		m_timestampResolveBuffer,
		0 // offset in resolve buffer
	);
}
```
````

正如您所注意到的，我们需要使用 `QueryResolve` 用法创建一个专用缓冲区 `m_timestampResolveBuffer` 。该缓冲区需要为每个时间戳保留一个 **64 位无符号整数**：

````{tab} With webgpu.hpp
```C++
// In Application.h
wgpu::Buffer m_timestampBuffer = nullptr;

// In Application.cpp
void Application::initBenchmark() {
	// [...]

	// Create buffer to store timestamps
	BufferDescriptor bufferDesc;
	bufferDesc.label = "timestamp resolve buffer";
	bufferDesc.size = 2 * sizeof(uint64_t);
	bufferDesc.usage = BufferUsage::QueryResolve; // important!
	m_timestampResolveBuffer = m_device.createBuffer(bufferDesc);
}
```
````

````{tab} Vanilla webgpu.h
```C++
// In Application.h
WGPUBuffer m_timestampBuffer = nullptr;

// In Application.cpp
void Application::initBenchmark() {
	// [...]

	// Create buffer to store timestamps
	WGPUBufferDescriptor bufferDesc;
	bufferDesc.label = "timestamp resolve buffer";
	bufferDesc.size = 2 * sizeof(uint64_t);
	bufferDesc.usage = WGPUBufferUsage_QueryResolve; // important!
	bufferDesc.mappedAtCreation = false;
	m_timestampResolveBuffer = wgpuDeviceCreateBuffer(m_device, bufferDesc);
}
```
````

```{warning}
在网络中，时间戳分辨率可能包括对值进行舍入，以避免访问可能导致定时攻击的精确信息。
```

我们可以在主循环中调用 `resolveTimestamps`，在 `renderPass.end()` 之后和 `encoder.finish()` 之前：

````{tab} With webgpu.hpp
```C++
// in Application::onFrame()
renderPass.end();

resolveTimestamps(encoder);
```
````

````{tab} Vanilla webgpu.h
```C++
// in Application::onFrame()
wgpuRenderPassEncoderEnd(renderPass);

resolveTimestamps(encoder);
```
````

在这个阶段，如果我们只在GPU上处理时间戳，这就是我们所需要的。例如，我们可以将它们作为着色器中的制服提供。

### 获取时间戳

但通常我们需要读取 CPU 上的时间戳（例如用 ImGui 显示它们）。

我们不能直接映射解析缓冲区，因为具有 `MapRead` 用法的缓冲区只能用于映射。因此，我们创建另一个缓冲区，即 `m_timestampMapBuffer`：

````{tab} With webgpu.hpp
```C++
// In Application.h
wgpu::Buffer m_timestampMapBuffer = nullptr;

// In Application.cpp
void Application::initBenchmark() {
	// [...]
	// add CopySrc usage here:
	bufferDesc.usage = BufferUsage::QueryResolve | BufferUsage::CopySrc;
	m_timestampResolveBuffer = m_device.createBuffer(bufferDesc);

	bufferDesc.label = "timestamp map buffer";
	bufferDesc.size = 2 * sizeof(uint64_t);
	bufferDesc.usage = BufferUsage::MapRead | BufferUsage::CopyDst;
	m_timestampMapBuffer = m_device.createBuffer(bufferDesc);
}
```
````

````{tab} Vanilla webgpu.h
```C++
// In Application.h
WGPUBuffer m_timestampMapBuffer = nullptr;

// In Application.cpp
void Application::initBenchmark() {
	// [...]
	// add CopySrc usage here:
	bufferDesc.usage = WGPUBufferUsage_QueryResolve | WGPUBufferUsage_CopySrc;
	m_timestampResolveBuffer = wgpuDeviceCreateBuffer(m_device, bufferDesc);

	bufferDesc.label = "timestamp map buffer";
	bufferDesc.size = 2 * sizeof(uint64_t);
	bufferDesc.usage = WGPUBufferUsage_MapRead | WGPUBufferUsage_CopyDst;
	m_timestampMapBuffer = wgpuDeviceCreateBuffer(m_device, bufferDesc);
}
```
````

然后我们在解析后立即复制到该缓冲区：

````{tab} With webgpu.hpp
```C++
void Application::resolveTimestamps(CommandEncoder encoder) {
	// [...] Resolve the timestamp queries (write their result to the resolve buffer)
	
	// Copy to the map buffer
	encoder.copyBufferToBuffer(
		m_timestampResolveBuffer, 0,
		m_timestampMapBuffer, 0,
		2 * sizeof(uint64_t)
	);
}
```
````

````{tab} Vanilla webgpu.h
```C++
void Application::resolveTimestamps(WGPUCommandEncoder encoder) {
	// [...] Resolve the timestamp queries (write their result to the resolve buffer)
	
	// Copy to the map buffer
	wgpuCommandEncoderCopyBufferToBuffer(
		encoder,
		m_timestampResolveBuffer, 0,
		m_timestampMapBuffer, 0,
		2 * sizeof(uint64_t)
	);
}
```
````

最后我们映射这个缓冲区。但是我们必须在**提交编码器之后**注意执行此操作，因为在映射时不允许将其复制到缓冲区。因此，我们创建一个新的 `fetchTimestamps()` 方法，在 `queue.submit()` 之后调用：

````{tab} With webgpu.hpp
```C++
m_queue.submit(command);

fetchTimestamps();

m_swapChain.present();
```
````

````{tab} Vanilla webgpu.h
```C++
wgpuQueueSubmit(m_queue, 1, &command);

fetchTimestamps();

wgpuSwapChainPresent(m_swapChain);
```
````

另外，请记住**缓冲区映射是异步的**，因此在每帧调用此 `fetchTimestamps()` 函数时我们必须小心。

```{note}
这部分在架构上略有不同，具体取决于您使用的是 C++ 包装器还是普通 API，我邀请您选择下面的正确选项卡：
```

````{tab} With webgpu.hpp
总体而言，映射操作类似于我们在[使用缓冲区](../../basic-3d-rendering/input-geometry/playing-with-buffers.md) 章节中所做的操作：

```C++
void Application::fetchTimestamps() {
	m_timestampMapBuffer.mapAsync(MapMode::Read, 0, 2 * sizeof(uint64_t), [this](BufferMapAsyncStatus status) {
		if (status != BufferMapAsyncStatus::Success) {
			std::cerr << "Could not map buffer! status = " << status << std::endl;
		}
		else {
			uint64_t* timestampData = (uint64_t*)m_timestampMapBuffer.getConstMappedRange(0, 2 * sizeof(uint64_t));
			// [...] Use timestampData here.
			m_timestampMapBuffer.unmap();
		}
	});
}
```

然而，为了确保回调**超出它在这里定义的范围**，我们必须在 `Application` 类中维护它的句柄：

```C++
// In Application.h
std::unique_ptr<wgpu::BufferMapCallback> m_timestampMapHandle;

// In Application::fetchTimestamps()
m_timestampMapHandle = m_timestampMapBuffer.mapAsync(/* [...] */);
```

最后，如果已经有一个映射操作正在进行，我们**不想触发新的映射操作**！为了检查这一点，我们可以简单地使用句柄并在它不为空时**提前返回**：

```C++
void Application::fetchTimestamps(WGPUCommandEncoder encoder) {
	// If we are already in the middle of a mapping operation,
	// no need to trigger a new one.
	if (m_timestampMapHandle) return;

	m_timestampMapHandle = m_timestampMapBuffer.mapAsync(/* [...] */ {
		// [...] Use timestamp data if mapping is successful

		// Release the callback and signal that there is no longer an
		// ongoing mapping operation.
		m_timestampMapHandle.reset();
	});
}
```
````

````{tab} Vanilla webgpu.h
待办事项：
- 定义一个静态方法用作回调
- 添加一个布尔值来检查是否有正在进行的映射操作
````

总体而言，映射操作类似于我们在[使用缓冲区](../../basic-3d-rendering/input-geometry/playing-with-buffers.md) 章节中所做的操作：

```C++
void Application::fetchTimestamps() {
	wgpuBufferMapAsync(
		m_timestampMapBuffer,
		WGPUMapMode_Read,
		0, 2 * sizeof(uint64_t),
		[](WGPUBufferMapAsyncStatus status, void *that){
			reinterpret_cast<Application*>(that)->onTimestampBufferMapped(status);
		},
		(void*)this
	);
}

// Create a new callback method in the Application class:
void Application::onTimestampBufferMapped(WGPUBufferMapAsyncStatus status) {
	if (status != WGPUBufferMapAsyncStatus_Success) {
		std::cerr << "Could not map buffer! status = " << status << std::endl;
	}
	else {
		uint64_t* timestampData = (uint64_t*)wgpuBufferGetConstMappedRange(m_timestampMapBuffer, 0, 2 * sizeof(uint64_t));
		// [...] Use timestampData here.
		wgpuBufferUnmap(m_timestampMapBuffer);
	}
}
```

然而，如果已经有一个映射操作正在进行，我们**不想触发新的映射操作**！为了检查这一点，我们添加一个简单的布尔属性，并在映射操作已在进行时**提前返回**：

```C++
// In Application.h
bool m_timestampMapOngoing = false;

// In Application::fetchTimestamps()
m_timestampMapHandle = m_timestampMapBuffer.mapAsync(/* [...] */);

void Application::fetchTimestamps(WGPUCommandEncoder encoder) {
	// If we are already in the middle of a mapping operation,
	// no need to trigger a new one.
	if (m_timestampMapOngoing) return;

	m_timestampMapOngoing = true;
	wgpuBufferMapAsync(m_timestampMapBuffer, /* [...] */);
}

void Application::onTimestampBufferMapped(WGPUBufferMapAsyncStatus status) {
	m_timestampMapOngoing = false;
	// [...]
}
```

因此，我们现在可以在提交命令缓冲区后立即在每一帧安全地调用 `fetchTimestamps()` 。

使用时间戳值
----------------------

### 显示

我们终于可以在 CPU 上操作时间戳值了！首先我们可以在终端中**显示它们**：在地图回调中，当映射成功时：

```C++
// Use timestampData
uint64_t begin = timestampData[0];
uint64_t end = timestampData[1];
uint64_t nanoseconds = (end - begin);
float milliseconds = (float)nanoseconds * 1e-6;
std::cout << "Render pass took " << milliseconds << "ms" << std::endl;
```

最终每帧得到的日志行略少于 1 行：

```
渲染通道花费了 0.484128ms
渲染通道耗时 0.538624ms
渲染通道耗时 0.49056ms
渲染通道耗时 0.490912ms
渲染通道花费了 0.504864ms
渲染通道耗时 0.491872ms
渲染通道耗时 0.487808ms
渲染通道花费了 0.587872ms
渲染通道耗时 0.493504ms
渲染通道耗时 0.498112ms
渲染通道耗时 0.547136ms
渲染通道耗时 0.452928ms
```

### 统计

通常，我对每帧一行不感兴趣，而是对在 UI 中显示我的测量值的**平均值**和**标准差**感兴趣。我为此使用我的 [`TinyTimer.h`](https://gist.github.com/eliemichel/54912bdafb8d16b21b0e7d9fce73a845):

```C++
// In Application.h
#include "TinyTimer.h"

class Application {
	// [...]
	TinyTimer::PerformanceCounter m_perf;
};

// In fetch timestamp callback
m_perf.add_sample(milliseconds * 1e-3);

// In Application::updateGui()
ImGui::Text("Application average [...]", /* [...] */);
ImGui::Text("Render pass duration on GPU: %s", m_perf.summary().c_str());
ImGui::End();
```

```{figure} /images/benchmark/render-pass-timer.png
:align: center
:class: with-shadow
我们的 GPU 计时器显示在应用程序中。
```

```{note}
在此示例中，我们可以看到渲染通道花费的时间比一帧少得多。这是因为这里的限制因素是 **VSync**，它将每秒帧数限制为 60（我的显示器的最大刷新率）。
```

```{important}
在报告和比较基准值以及一般统计数据时，重要的是要查看**标准偏差**，而且还要查看估计该标准值的**样本数量**。
```

结论
----------

您现在可以使用精确的 GPU 端计时器，这对于评估应用程序的性能和识别瓶颈至关重要。请记住：

- GPU 计时器与 CPU 计时器不在同一**时间线**上。
- 您需要创建时间戳查询，然后**写入**它们，**解析**它们，最后异步**取**它们。
- 必须注意，在解析/复制操作不仅通过**提交**编码到GPU之前，不要获取。

我建议创建一个小类，专门负责管理应用程序中的计时器，以便样板与应用程序的逻辑隔离。

```{note}
如果您想测量**不是在每一帧**发生的事件的性能，您应该为每个此类计数器保留一个**布尔值**，告诉计数器是否已更新，以便仅在时间戳**实际更新**时才在获取回调时`add_sample`。
```

````{tab} With webgpu.hpp
*结果代码：* [`step095-timestamp-queries`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-timestamp-queries)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step095-timestamp-queries-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-timestamp-queries-vanilla)
````
