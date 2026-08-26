设备<span class="bullet">🟢</span>
==========

```{translation-warning} 译文可能已过时, /getting-started/adapter-and-device/the-device.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/010 - The Device
:parent: 005 - The Adapter
```

*结果代码 :* [`step010`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step010)

网络GPU**设备**代表一个**上下文**使用API。 我们创建的所有对象(几何,纹理等)都归设备所有.

设备是从一个**适配器**通过指定**限制和特征子集**我们被卷入其中。 设备创建后,应不再使用适配器.**唯一重要的能力**应用程序是设备的。

设备请求
--------------

请求设备看起来很像请求适配器,所以我们将使用非常相似的功能:

```{lit} C++, Utility functions (append, hidden)
{{Request device function}}
```

```{lit} C++, Request device function
/**
 * Utility function to get a WebGPU device, so that
 *     WGPUDevice device = requestDeviceSync(adapter, options);
 * is roughly equivalent to
 *     const device = await adapter.requestDevice(descriptor);
 * It is very similar to requestAdapter
 */
WGPUDevice requestDeviceSync(WGPUAdapter adapter, WGPUDeviceDescriptor const * descriptor) {
	struct UserData {
		WGPUDevice device = nullptr;
		bool requestEnded = false;
	};
	UserData userData;

	auto onDeviceRequestEnded = [](WGPURequestDeviceStatus status, WGPUDevice device, char const * message, void * pUserData) {
		UserData& userData = *reinterpret_cast<UserData*>(pUserData);
		if (status == WGPURequestDeviceStatus_Success) {
			userData.device = device;
		} else {
			std::cout << "Could not get WebGPU device: " << message << std::endl;
		}
		userData.requestEnded = true;
	};

	wgpuAdapterRequestDevice(
		adapter,
		descriptor,
		onDeviceRequestEnded,
		(void*)&userData
	);

#ifdef __EMSCRIPTEN__
	while (!userData.requestEnded) {
		emscripten_sleep(100);
	}
#endif // __EMSCRIPTEN__

	assert(userData.requestEnded);

	return userData.device;
}
```

```{note}
在配套代码中,我把这些功能移到`webgpu-utils.cpp`
```

```{lit} C++, file: webgpu-utils.h (hidden)
#pragma once

#include <webgpu/webgpu.h>

/**
 * Utility function to get a WebGPU adapter, so that
 *     WGPUAdapter adapter = requestAdapter(options);
 * is roughly equivalent to
 *     const adapter = await navigator.gpu.requestAdapter(options);
 */
WGPUAdapter requestAdapterSync(WGPUInstance instance, WGPURequestAdapterOptions const * options);

/**
 * Utility function to get a WebGPU device, so that
 *     WGPUAdapter device = requestDevice(adapter, options);
 * is roughly equivalent to
 *     const device = await adapter.requestDevice(descriptor);
 * It is very similar to requestAdapter
 */
WGPUDevice requestDeviceSync(WGPUAdapter adapter, WGPUDeviceDescriptor const * descriptor);

/**
 * An example of how we can inspect the capabilities of the hardware through
 * the adapter object.
 */
void inspectAdapter(WGPUAdapter adapter);

/**
 * Display information about a device
 */
void inspectDevice(WGPUDevice device);
```

```{lit} C++, file: webgpu-utils.cpp (hidden)
#include "webgpu-utils.h"

#include <iostream>
#include <vector>
#include <cassert>

#ifdef __EMSCRIPTEN__
#  include <emscripten.h>
#endif // __EMSCRIPTEN__

{{Utility functions}}
```

```{lit} C++, Utility functions in main.cpp (replace, hidden)
```

```{lit} C++, Includes (prepend, hidden)
#include "webgpu-utils.h"
```

```{lit} CMake, Define app target (replace, hidden)
{{Dependency subdirectories}}

add_executable(App
	{{App source files}}
)

{{Link libraries}}
```

```{lit} CMake, App source files (hidden)
main.cpp
webgpu-utils.h
webgpu-utils.cpp
```

在主函数中,在得到适配器后,我们可以请求设备:

```{lit} C++, Request device
std::cout << "Requesting device..." << std::endl;

WGPUDeviceDescriptor deviceDesc = {};
{{Build device descriptor}}
WGPUDevice device = requestDeviceSync(adapter, &deviceDesc);

std::cout << "Got device: " << device << std::endl;
```

```{lit} C++, Create things (append, hidden)
{{Request device}}
{{Setup device callbacks}}
```

当然,当程序结束的时候,我们释放设备:

```{lit} C++, Destroy things (prepend)
wgpuDeviceRelease(device);
```

````{note}
适配器可以是**在设备之前释放**事实上,我们一拿到设备就放出来 别再使用它,这是好的做法

```{lit} C++, Create things (append, hidden)
// We no longer need to access the adapter once we have the device
{{Destroy adapter}}
```

```{lit} C++, Destroy things (replace)
wgpuAdapterRelease(adapter);
```

````

设备描述符
-----------------

我们进去看看`webgpu.h`描述者是什么样子的:

```C++
typedef struct WGPUDeviceDescriptor {
	WGPUChainedStruct const * nextInChain;
	WGPU_NULLABLE char const * label;
	size_t requiredFeatureCount;
	WGPUFeatureName const * requiredFeatures;
	WGPU_NULLABLE WGPURequiredLimits const * requiredLimits;
	WGPUQueueDescriptor defaultQueue;
	WGPUDeviceLostCallback deviceLostCallback;
	void * deviceLostUserdata;
} WGPUDeviceDescriptor;

// (this struct definition is actually above)
typedef struct WGPUQueueDescriptor {
	WGPUChainedStruct const * nextInChain;
	WGPU_NULLABLE char const * label;
} WGPUQueueDescriptor;
```

现在,我们将将它初始化为一个极小的选项,不需要特殊特征,并使用默认限制:

```{lit} C++, Build device descriptor
deviceDesc.nextInChain = nullptr;
deviceDesc.label = "My Device"; // anything works here, that's your call
deviceDesc.requiredFeatureCount = 0; // we do not require any specific feature
deviceDesc.requiredLimits = nullptr; // we do not require any specific limit
deviceDesc.defaultQueue.nextInChain = nullptr;
deviceDesc.defaultQueue.label = "The default queue";
{{Set device lost callback}}
```

```{lit} C++, Set device lost callback
// Null for now, see below
deviceDesc.deviceLostCallback = nullptr;
```

每当我们需要从设备中获得更多的能力时,我们将回到这里完善这些选择。

```{note}
那个`label`这是**错误消息中使用**帮助您在出错的地方调试,所以一旦获得多个相同类型的对象,就使用它是一个很好的做法。 目前,这只被Dawn使用.
```

检查设备
---------------------

和适配器一样,该装置也有自己的一套能力.

```{lit} C++, Utility functions (append)
// We also add an inspect device function:
void inspectDevice(WGPUDevice device) {
	std::vector<WGPUFeatureName> features;
	size_t featureCount = wgpuDeviceEnumerateFeatures(device, nullptr);
	features.resize(featureCount);
	wgpuDeviceEnumerateFeatures(device, features.data());

	std::cout << "Device features:" << std::endl;
	std::cout << std::hex;
	for (auto f : features) {
		std::cout << " - 0x" << f << std::endl;
	}
	std::cout << std::dec;

	WGPUSupportedLimits limits = {};
	limits.nextInChain = nullptr;

#ifdef WEBGPU_BACKEND_DAWN
	bool success = wgpuDeviceGetLimits(device, &limits) == WGPUStatus_Success;
#else
	bool success = wgpuDeviceGetLimits(device, &limits);
#endif

	if (success) {
		std::cout << "Device limits:" << std::endl;
		std::cout << " - maxTextureDimension1D: " << limits.limits.maxTextureDimension1D << std::endl;
		std::cout << " - maxTextureDimension2D: " << limits.limits.maxTextureDimension2D << std::endl;
		std::cout << " - maxTextureDimension3D: " << limits.limits.maxTextureDimension3D << std::endl;
		std::cout << " - maxTextureArrayLayers: " << limits.limits.maxTextureArrayLayers << std::endl;
		{{Extra device limits}}
	}
}
```

```{lit} C++, Extra device limits (hidden)
std::cout << " - maxBindGroups: " << limits.limits.maxBindGroups << std::endl;
std::cout << " - maxDynamicUniformBuffersPerPipelineLayout: " << limits.limits.maxDynamicUniformBuffersPerPipelineLayout << std::endl;
std::cout << " - maxDynamicStorageBuffersPerPipelineLayout: " << limits.limits.maxDynamicStorageBuffersPerPipelineLayout << std::endl;
std::cout << " - maxSampledTexturesPerShaderStage: " << limits.limits.maxSampledTexturesPerShaderStage << std::endl;
std::cout << " - maxSamplersPerShaderStage: " << limits.limits.maxSamplersPerShaderStage << std::endl;
std::cout << " - maxStorageBuffersPerShaderStage: " << limits.limits.maxStorageBuffersPerShaderStage << std::endl;
std::cout << " - maxStorageTexturesPerShaderStage: " << limits.limits.maxStorageTexturesPerShaderStage << std::endl;
std::cout << " - maxUniformBuffersPerShaderStage: " << limits.limits.maxUniformBuffersPerShaderStage << std::endl;
std::cout << " - maxUniformBufferBindingSize: " << limits.limits.maxUniformBufferBindingSize << std::endl;
std::cout << " - maxStorageBufferBindingSize: " << limits.limits.maxStorageBufferBindingSize << std::endl;
std::cout << " - minUniformBufferOffsetAlignment: " << limits.limits.minUniformBufferOffsetAlignment << std::endl;
std::cout << " - minStorageBufferOffsetAlignment: " << limits.limits.minStorageBufferOffsetAlignment << std::endl;
std::cout << " - maxVertexBuffers: " << limits.limits.maxVertexBuffers << std::endl;
std::cout << " - maxVertexAttributes: " << limits.limits.maxVertexAttributes << std::endl;
std::cout << " - maxVertexBufferArrayStride: " << limits.limits.maxVertexBufferArrayStride << std::endl;
std::cout << " - maxInterStageShaderComponents: " << limits.limits.maxInterStageShaderComponents << std::endl;
std::cout << " - maxComputeWorkgroupStorageSize: " << limits.limits.maxComputeWorkgroupStorageSize << std::endl;
std::cout << " - maxComputeInvocationsPerWorkgroup: " << limits.limits.maxComputeInvocationsPerWorkgroup << std::endl;
std::cout << " - maxComputeWorkgroupSizeX: " << limits.limits.maxComputeWorkgroupSizeX << std::endl;
std::cout << " - maxComputeWorkgroupSizeY: " << limits.limits.maxComputeWorkgroupSizeY << std::endl;
std::cout << " - maxComputeWorkgroupSizeZ: " << limits.limits.maxComputeWorkgroupSizeZ << std::endl;
std::cout << " - maxComputeWorkgroupsPerDimension: " << limits.limits.maxComputeWorkgroupsPerDimension << std::endl;
```

```{lit} C++, Create things (append, hidden)
inspectDevice(device);
```

```{admonition} Implementation divergences
喜欢`wgpuAdapterGetLimits`,该程序`wgpuDeviceGetLimits`返回布尔值`wgpu-native`不过`WGPUStatus`在黎明时。
```

我们可以看到,默认情况下,设备的限制与适配器所支持的不一样. 设置`deviceDesc.requiredLimits`改为`nullptr`上文要求的最低限度:

```
设备限制 :
- 最大剂量
- 最大剂量
- 最大剂量
- 最大箭头:256架
```

设备调用
----------------

为了**获得通知**当设备发生错误或者它不再可用时,我们可以设置**回调**函数。 这些真的**帮助调试**因此,我鼓励你这样做 尽管它是可选的。

###设备丢失回调

正如我们在上面简要看到的,丢失的设备回调是通过设备描述器提供的`deviceLostCallback`字段 :

```{lit} C++, Set device lost callback (replace)
// A function that is invoked whenever the device stops being available.
deviceDesc.deviceLostCallback = [](WGPUDeviceLostReason reason, char const* message, void* /* pUserData */) {
	std::cout << "Device lost: reason " << reason;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
};
```

```{note}
我们用一个[C++ 羊肉酱](https://en.cppreference.com/w/cpp/language/lambda)这里,但是`deviceDesc.deviceLostCallback`也可以被指定一个正则函数的名称。
```

设备总是"丢失" 当它被最终召唤摧毁`wgpuDeviceRelease`。它也可能由于其他原因丢失,主要意味着后端执行惊慌失措并坠毁。

```{important}
那个`deviceLostCallback`必须超过该设备,这样当后者被摧毁时,回调仍然有效。
```

###未抓取错误召回

当我们滥用API时,会引用未抓取的错误回调,并给出关于出错的非常翔实的反馈. 设定在设备创建之后,通过调用`wgpuDeviceSetUncapturedErrorCallback`:

```{lit} C++, Setup device callbacks
auto onDeviceError = [](WGPUErrorType type, char const* message, void* /* pUserData */) {
	std::cout << "Uncaptured device error: type " << type;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
};
wgpuDeviceSetUncapturedErrorCallback(device, onDeviceError, nullptr /* pUserData */);
```

如果你使用调试器(我推荐),比如`gdb`或你的IDE, 我建议你**设置断点**在此调用中, 以便程序暂停, 当WebGPU遇到出乎意料的错误时, 为您提供一个调用堆栈 。

````{admonition} Dawn
**默认**Dawn 只在设备“ ticks” 时运行回调, 所以错误回调在 a**不同的调用堆栈**而不是错误发生地,使断点信息较少。 要迫使 Dawn 在出现错误时立即引用错误调用, 您可以启用实例**切换**:

```C++
WGPUInstanceDescriptor desc = {};
desc.nextInChain = nullptr;

#ifdef WEBGPU_BACKEND_DAWN
// Make sure the uncaptured error callback is called as soon as an error
// occurs rather than at the next call to "wgpuDeviceTick".
WGPUDawnTogglesDescriptor toggles;
toggles.chain.next = nullptr;
toggles.chain.sType = WGPUSType_DawnTogglesDescriptor;
toggles.disabledToggleCount = 0;
toggles.enabledToggleCount = 1;
const char* toggleName = "enable_immediate_error_handling";
toggles.enabledToggles = &toggleName;

desc.nextInChain = &toggles.chain;
#endif // WEBGPU_BACKEND_DAWN

WGPUInstance instance = wgpuCreateInstance(&desc);
```

Toggles是Dawn在整个WebGPU实例中实现/失效特性的特殊方式. 见整个列表[`Toggle.cpp`](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/dawn/native/Toggles.cpp#33).
````

结论
----------

- 我们现在有了**设备**,从中我们可以创建所有其他 WebGPU 对象。
 - **重要内容 :**一旦设备被创建,一般不应再使用适配器. 与应用相关的唯一能力是设备之一.
- 默认限度是最小限度,而不是适配器支持的限度。 这有助于确保各种设备的一致性。

*结果代码 :* [`step010`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step010)
