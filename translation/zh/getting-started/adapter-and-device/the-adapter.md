适应者<span class="bullet">🟢</span>
===========

```{translation-warning} 译文可能已过时, /getting-started/adapter-and-device/the-adapter.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/005 - The Adapter
:parent: 001 - Hello WebGPU
```

*结果代码 :* [`step005`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step005)

之前,我们得到的手**设备**,我们需要选择一个**适配器**。同一主机系统可能暴露**多个适配器**如果它可以访问多个物理GPU. 它还可能有一个代表模拟/虚拟设备的适配器。

```{note}
高端笔记本电脑很常见**两个物理GPU**页:1**业绩好**一个和一个**低能耗**消费 1(通常在CPU芯片内集成).
```

每个适配器广告列表可选**特性**和**支持的限制**它可以处理。 这些工具用于确定系统的总体能力,然后用于:**请求设备**.

我们为什么都有**适配器**之后为 a**设备**抽象吗?

想法是限制"它工作在我的机器上"问题, 当你尝试你的程序在另一个机器上。 那个**适配器**用于**获取能力**,用于在非常不同的代码路径中选择应用程序的行为。 一旦选择了代码路径, a**设备**创建于**我们选择的能力**.

然后在应用程序的其余部分只允许选择用于此设备的能力. 这边**无法无意中依赖自己机器特有的能力**.

```{themed-figure} /images/the-adapter/limit-tiers_{theme}.svg
:align: center
在高级使用适配器/装置的双重性中,我们可以设置多个限制预设,并根据适配器选择一个. 就我们而言,如果没有支持,我们有一个单一的预设和提前中止。
```


请求适配器
----------------------

适配器不是我们*创建*但其实我们*请求*使用函数`wgpuInstanceRequestAdapter`.

````{note}
A. 程序名称`webgpu.h`始终遵循相同的构造:

```C
wgpuSomethingSomeAction(something, ...)
             ^^^^^^^^^^ // What to do...
    ^^^^^^^^^ // ...on what type of object
^^^^ // (Common prefix to avoid naming collisions)
```

函数的第一个参数总是代表类型"某物"对象的"手"(一个盲指).
````

因此,按照名字的建议,第一个论点是:`WGPUInstance`那是我们在前一章中创造的。 其他人怎么办?

```C++
// Signature of the wgpuInstanceRequestAdapter function as defined in webgpu.h
void wgpuInstanceRequestAdapter(
	WGPUInstance instance,
	WGPU_NULLABLE WGPURequestAdapterOptions const * options,
	WGPURequestAdapterCallback callback,
	void * userdata
);
```

```{note}
查看一个函数如何定义在`webgpu.h`!
```

第二个论点是:**选项**这有点像**描述符**我们发现在`wgpuCreateSomething`函数,我们详述如下。 那个`WGPU_NULLABLE`旗帜是一个空定义,只是在这里告诉读者(即我们),它被允许将参数留给`nullptr`用于**默认选项**.

###同步函数

后两个论点合在一起, 并揭示另一个**WebGPU 缩写**事实上,职能`wgpuInstanceRequestAdapter`这是**一个同步**。这意味着不直接返回`WGPUAdapter`对象,此请求函数记得**回调**,即当请求结束时调用函数。

```{note}
同步函数在WebGPU API的多个位置使用,只要操作需要时间. 其实**WebGPU 函数**回来需要时间 这样,我们正在写的CPU程序就永远不会被长时间的操作所阻断!
```

为了更清楚地了解这个召回机制,这里是定义:`WGPURequestAdapterCallback`函数类型 :

```C++
// Definition of the WGPURequestAdapterCallback function type as defined in webgpu.h
typedef void (*WGPURequestAdapterCallback)(
	WGPURequestAdapterStatus status,
	WGPUAdapter adapter,
	char const * message,
	void * userdata
);
```

回电是**函数**接收**请求的适配器**作为辩论,**状态**信息(说明请求是否失败和原因)以及这个神秘的`userdata` **指针**.

这个`userdata`指针可以是任何东西,它不是由WebGPU解释的,而只是**已转发**从初始呼叫到`wgpuInstanceRequestAdapter`来回呼唤,作为一个手段**共享一些上下文信息**:

```C++
void onAdapterRequestEnded(
	WGPURequestAdapterStatus status, // a success status
	WGPUAdapter adapter, // the returned adapter
	char const* message, // error message, or nullptr
	void* userdata // custom user data, as provided when requesting the adapter
) {
	// [...] Do something with the adapter

	// Manipulate user data
	bool* pRequestEnded = reinterpret_cast<bool*>(userdata);
	*pRequestEnded = true;
}

// [...]

// In main():
bool requestEnded = false;
wgpuInstanceRequestAdapter(
	instance /* equivalent of navigator.gpu */,
	&options,
	onAdapterRequestEnded,
	&requestEnded // custom user data is simply a pointer to a boolean in this case
);
```

在下一节中,我们看到更先进的上下文,以便在请求完成后检索适配器。

````{admonition} Note - JavaScript API
:class: foldable note

在那个**Java脚本 API**WebGPU 中,使用内置函数同步[答应](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)机制:

```js
cnst适配器 Promise = navigator.gpu. requestAdapter(选项);
/ / "承诺" 还没有价值, 相当是一个手柄, 我们可以连接回调 :
Promise.when(on Adapter Request Ended). catchs(on Adapter Request Faulted) 页面存档备份,存于互联网档案馆 页面存档备份,存于互联网档案馆 互联网档案馆.

// [...]

// 代替“ 状态” 参数, 我们有多个回调 :
函数在 Adapter Requested (adapter) 上{
// 做一个适配器
}
函数在 Adapter 请求失败( 错误) 上{
// 显示错误消息
}
```

JavaScript 语言后来引入了一个机制[`async`函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function),使**"等待着"**用于同步函数,而不明确创建回调:

```js
// (从一个同步函数内)
cons 适配器 = 等待 navigator.gpu. requestAdapter(选项);
// 做一个适配器
```

这一机制现在以其他语言存在,例如:[Py](https://docs.python.org/3/library/asyncio-task.html),甚至已被引入到 C++20 中[皮层](https://en.cppreference.com/w/cpp/language/coroutines).

不过,我尽量**避免堆叠过多层次的抽象**在此指南中, 所以我们不会使用这些( 也坚持 C++17), 但高级读者可能想要创建自己的依赖皮层的 WebGPU 包装器 。
````

###请求

我们可以将整个适配器请求包在下面`requestAdapterSync()`函数,我提供这个功能是为了我们不要花太多时间**锅炉板**代码(这里的重要部分是,你得到**同步调用**说明:

```{lit} C++, Includes (append)
#include <cassert>
```

```{lit} C++, Request adapter function
/**
 * Utility function to get a WebGPU adapter, so that
 *     WGPUAdapter adapter = requestAdapterSync(options);
 * is roughly equivalent to
 *     const adapter = await navigator.gpu.requestAdapter(options);
 */
WGPUAdapter requestAdapterSync(WGPUInstance instance, WGPURequestAdapterOptions const * options) {
	// A simple structure holding the local information shared with the
	// onAdapterRequestEnded callback.
	struct UserData {
		WGPUAdapter adapter = nullptr;
		bool requestEnded = false;
	};
	UserData userData;

	// Callback called by wgpuInstanceRequestAdapter when the request returns
	// This is a C++ lambda function, but could be any function defined in the
	// global scope. It must be non-capturing (the brackets [] are empty) so
	// that it behaves like a regular C function pointer, which is what
	// wgpuInstanceRequestAdapter expects (WebGPU being a C API). The workaround
	// is to convey what we want to capture through the pUserData pointer,
	// provided as the last argument of wgpuInstanceRequestAdapter and received
	// by the callback as its last argument.
	auto onAdapterRequestEnded = [](WGPURequestAdapterStatus status, WGPUAdapter adapter, char const * message, void * pUserData) {
		UserData& userData = *reinterpret_cast<UserData*>(pUserData);
		if (status == WGPURequestAdapterStatus_Success) {
			userData.adapter = adapter;
		} else {
			std::cout << "Could not get WebGPU adapter: " << message << std::endl;
		}
		userData.requestEnded = true;
	};

	// Call to the WebGPU request adapter procedure
	wgpuInstanceRequestAdapter(
		instance /* equivalent of navigator.gpu */,
		options,
		onAdapterRequestEnded,
		(void*)&userData
	);

	// We wait until userData.requestEnded gets true
	{{Wait for request to end}}

	assert(userData.requestEnded);

	return userData.adapter;
}
```

```{lit} C++, Utility functions (hidden)
// All utility functions are regrouped here
{{Request adapter function}}
```

在主要功能中,在创建WebGPU实例后,我们可以获得适配器:

```{lit} C++, Request adapter
std::cout << "Requesting adapter..." << std::endl;

WGPURequestAdapterOptions adapterOpts = {};
adapterOpts.nextInChain = nullptr;
WGPUAdapter adapter = requestAdapterSync(instance, &adapterOpts);

std::cout << "Got adapter: " << adapter << std::endl;
```

#### Waiting for the request to end

下面说明如何等待请求结束。

你可能注意到上面的话**我们需要等待**要求结束,即在返回之前援引回调。

使用时**本地语言**API( 黎明或`wgpu-native`),它实际上**无需**我们知道,当`wgpuInstanceRequestAdapter`函数返回其调用已调用。

然而,在使用时**标记**我们需要交出控制器**返回浏览器**直到适配器准备好 在 JavaScript 中, 这将使用`await`关键词 相反,标记提供了`emscripten_sleep`函数中断 C++ 模块数毫秒:

```{lit} C++, Wait for request to end
#ifdef __EMSCRIPTEN__
	while (!userData.requestEnded) {
		emscripten_sleep(100);
	}
#endif // __EMSCRIPTEN__
```

为了使用这个,我们必须增加一个**自定义链接选项**输入`CMakeLists.txt`时,`if (EMSCRIPTEN)`块 :

```{lit} CMake, Emscripten-specific options (append)
# Enable the use of emscripten_sleep()
target_link_options(App PRIVATE -sASYNCIFY)
```

也不要忘记包括`emscripten.h`为了使用`emscripten_sleep`:

```{lit} C++, Includes (append)
#ifdef __EMSCRIPTEN__
#  include <emscripten.h>
#endif // __EMSCRIPTEN__
```

###销毁

类似WebGPU实例,我们必须释放适配器:

```{lit} C++, Destroy adapter
wgpuAdapterRelease(adapter);
```

````{note}
我们不再需要使用`instance`一旦我们选择**适配器**,所以我们可以打电话`wgpuInstanceRelease(instance)`在适配器请求之后**而不是在结尾**编辑**基本实例**对象会一直活到适配器释放,但我们不需要管理它。

```{lit} C++, Create things (hidden)
{{Create WebGPU instance}}
{{Check WebGPU instance}}
{{Request adapter}}
// We no longer need to use the instance once we have the adapter
{{Destroy WebGPU instance}}
```
````

```{lit} C++, file: main.cpp (replace, hidden)
{{Includes}}

{{Utility functions in main.cpp}}

int main() {
	{{Create things}}

	{{Main body}}

	{{Destroy things}}

	return 0;
}
```

```{lit} C++, Utility functions in main.cpp (hidden)
{{Utility functions}}
```

```{lit} C++, Main body (hidden)
```

```{lit} C++, Destroy things (hidden)
{{Destroy adapter}}
```

检查适配器
----------------------

适配对象提供**关于基本执行情况的信息**和硬件, 关于它能做什么或不做什么。 其广告信息如下:

 - **限制**重新组合所有**最高和最低数额**可能限制 GPU 及其驱动程序的行为的数值。 一个典型的例子就是最大纹理大小. 使用`wgpuAdapterGetLimits`.
 - **特征**非强制性**扩展**的WebGPU,适配器可能支持也可能不支持。 它们可以使用`wgpuAdapterEnumerateFeatures`单独试验或`wgpuAdapterHasFeature`.
 - **属性**是关于适配器的额外信息,如其名称、销售商等。 使用`wgpuAdapterGetProperties`.

```{note}
在所附代码中,适配器能力检查载于`inspectAdapter()`函数。
```

```{lit} C++, Utility functions (append, hidden)
void inspectAdapter(WGPUAdapter adapter) {
	{{Inspect adapter}}
}
```

```{lit} C++, Request adapter (append, hidden)
inspectAdapter(adapter);
```

###限制

我们首先可以列出适配器支持的限度。`wgpuAdapterGetLimits`此函数作为参数 a`WGPUSupportedLimits`在它写限制时的对象 :

```{lit} C++, Inspect adapter
#ifndef __EMSCRIPTEN__
WGPUSupportedLimits supportedLimits = {};
supportedLimits.nextInChain = nullptr;

#ifdef WEBGPU_BACKEND_DAWN
bool success = wgpuAdapterGetLimits(adapter, &supportedLimits) == WGPUStatus_Success;
#else
bool success = wgpuAdapterGetLimits(adapter, &supportedLimits);
#endif

if (success) {
	std::cout << "Adapter limits:" << std::endl;
	std::cout << " - maxTextureDimension1D: " << supportedLimits.limits.maxTextureDimension1D << std::endl;
	std::cout << " - maxTextureDimension2D: " << supportedLimits.limits.maxTextureDimension2D << std::endl;
	std::cout << " - maxTextureDimension3D: " << supportedLimits.limits.maxTextureDimension3D << std::endl;
	std::cout << " - maxTextureArrayLayers: " << supportedLimits.limits.maxTextureArrayLayers << std::endl;
}
#endif // NOT __EMSCRIPTEN__
```

```{admonition} Implementation divergences
程序`wgpuAdapterGetLimits`返回布尔值`wgpu-native`不过`WGPUStatus`在黎明时。

同时,截至2024年4月1日,`wgpuAdapterGetLimits`尚未在 Google Chrome 上执行,因此`#ifndef __EMSCRIPTEN__`见上文。
```

以下是你能看到的一个例子:

```
适配器限制 :
- 最大剂量
- 最大剂量
- 最大剂量
- 最大箭头:2048
```

这意味着例如我的GPU可以处理高达32k的2D纹理,高达16k的3D纹理以及高达2k层的纹理阵列.

```{note}
有一个**更多限制**我们将在下几章逐步介绍。 那个**完整列表**这是[类型](https://www.w3.org/TR/webgpu/#limits),连同他们的**默认值**,它预计也是适配器要求支持WebGPU的最低限度。
```

###特征

现在让我们集中讨论`wgpuAdapterEnumerateFeatures`函数,它列举了WebGPU执行的特性,因为它的使用非常典型地来自WebGPU本土.

我们称之为功能**两次**编辑**初次**,我们作为返回提供了无指针,因此函数只返回**特性数**,但不是特征本身。

然后我们充满活力**分配内存**用于存储这些结果项,并将同一函数称为**第二次**,这次要用一个指针指向结果应存储在哪里。

```{lit} C++, Includes (append)
#include <vector>
```

```{lit} C++, Inspect adapter (append)
std::vector<WGPUFeatureName> features;

// Call the function a first time with a null return address, just to get
// the entry count.
size_t featureCount = wgpuAdapterEnumerateFeatures(adapter, nullptr);

// Allocate memory (could be a new, or a malloc() if this were a C program)
features.resize(featureCount);

// Call the function a second time, with a non-null return address
wgpuAdapterEnumerateFeatures(adapter, features.data());

std::cout << "Adapter features:" << std::endl;
std::cout << std::hex; // Write integers as hexadecimal to ease comparison with webgpu.h literals
for (auto f : features) {
	std::cout << " - 0x" << f << std::endl;
}
std::cout << std::dec; // Restore decimal numbers
```

特征是对应 enum 的数字`WGPUFeatureName`定义`webgpu.h`我们使用`std::hex`把它们显示为十六进制值,因为这样它们就会被列在`webgpu.h`.

您可能注意到非常高的数字显然在本列表中没有定义。 这些是**扩展**由我们本地执行(例如,定义如下:`wgpu.h`改为`webgpu.h`情况`wgpu-native`).

###属性

最后,我们可以看看适配器的属性,其中包含着我们可能想要向最终用户展示的信息:

```{lit} C++, Inspect adapter (append)
WGPUAdapterProperties properties = {};
properties.nextInChain = nullptr;
wgpuAdapterGetProperties(adapter, &properties);
std::cout << "Adapter properties:" << std::endl;
std::cout << " - vendorID: " << properties.vendorID << std::endl;
if (properties.vendorName) {
	std::cout << " - vendorName: " << properties.vendorName << std::endl;
}
if (properties.architecture) {
	std::cout << " - architecture: " << properties.architecture << std::endl;
}
std::cout << " - deviceID: " << properties.deviceID << std::endl;
if (properties.name) {
	std::cout << " - name: " << properties.name << std::endl;
}
if (properties.driverDescription) {
	std::cout << " - driverDescription: " << properties.driverDescription << std::endl;
}
std::cout << std::hex;
std::cout << " - adapterType: 0x" << properties.adapterType << std::endl;
std::cout << " - backendType: 0x" << properties.backendType << std::endl;
std::cout << std::dec; // Restore decimal numbers
```

我用我的Titan RTX做样本:

```
适配器属性 :
- 供应商身份证:4318
- 销售商名称: NVIDIA
- 建筑:
- 设备ID:7682
- 名称:NVIDIA TITAN RTX
- 司机 说明: 536.23
- 适配器Type:0x0
- 后端类型: 0x5
```

结论
----------

- 与WebGPU有关的第一件事就是获得**适配器**.
- 一旦我们有一个适配器, 我们可以检查它的**能力**(限制、特征)和属性。
- 我们学会了使用**同步函数**和**双打**计数函数 。

*结果代码 :* [`step005`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step005)
