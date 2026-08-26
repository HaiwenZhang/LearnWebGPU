与黎明 (<span class="bullet">🟠</span>WIP)
=========

```{translation-warning} 译文可能已过时, /appendices/custom-extensions/with-dawn.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码：* [`dawn:eliemichel/foo`](https://github.com/eliemichel/Dawn/tree/eliemichel/foo) 和 [`step030-test-foo-extension-dawn`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-test-foo-extension-dawn)

```{note}
如果您还没有，请不要忘记阅读【扩展机制](mechanism.md)的介绍。
```

设置
-----

我们不是在配置时获取 Dawn 源代码并将其丢失在 `build/_deps` 中，而是将 Dawn 克隆为源代码树中的 git 子模块（或只是复制它）。

```{note}
我从 [`step030-headless`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless) 分支开始，获得测试应用程序的简约版本（并剥离 `save_image` 部分）。
```

```bash
# Setup Dawn in a dawn/ subdirectory
git submodule add https://dawn.googlesource.com/dawn dawn

cd dawn

# Fetch the release tag of your choice (in a shallow way)
git fetch origin chromium/5869
git reset --hard FETCH_HEAD

# Create your new custom branch from this tag
git checkout -b eliemichel/foo
```

更新 `webgpu/webgpu.cmake` 以使用本地 Dawn 子模块（只需从 [此处](https://github.com/eliemichel/LearnWebGPU-Code/blob/step030-test-foo-extension-dawn/webgpu/webgpu.cmake) 复制）。

```{note}
我还将 [`webgpu.hpp`](https://github.com/eliemichel/WebGPU-Cpp/blob/main/dawn/webgpu.hpp) 复制到 `webgpu/include/webgpu/` 中。
```

新功能
-----------

### 公共API

我们首先更新描述符结构来处理新的 Foo 扩展。与`wgpu-native`相反，我们**不直接写入扩展头**（我们称之为`webgpu-ext-foo.h`），因为Dawn的`webgpu.h`是**自动生成的**。

这一代的来源是 Dawn 存储库根目录下的大型 `dawn.json` 文件。例如找到 `"feature name": {` 添加我们的 Foo 功能：

```json
"feature name": {
    "category": "enum",
    "values": [
        {"value": 0, "name": "undefined", "jsrepr": "undefined"},
        {"value": 1, "name": "depth clip control"},
        /* [...] */
        {"value": 1010, "name": "MSAA render to single sampled", "tags": ["dawn"]},
        /* We add our custom extension here: */
        {"value": 4097, "name": "Foo", "tags": ["native"]}
    ]
},
```

```{note}
我们添加标签 `native` 表示此功能必须仅为本机构建生成，而不是为基于 Web 的设置生成。
```

```{caution}
不要忘记在上一行的末尾、我们的自定义行之前添加一个逗号 (`,`)。
```

我们现在可以检查该功能是否确实受支持：

```C++
if (wgpuAdapterHasFeature(adapter, WGPUFeatureName_Foo)) {
	std::cout << "Feature 'Foo' supported by adapter" << std::endl;
}
else {
	std::cout << "Feature 'Foo' NOT supported by adapter" << std::endl;
}
```

> 😡 这里**不**支持！

从上面的 json 文件自动生成的代码描述了 Dawn 库的 **公共 API**。我们现在需要手动修改用于与不同后端（Vulkan、DirectX、Metal 等）进行对话的 **内部 `dawn::native` API**。

### 内部API

我们将我们的功能添加到内部 `Feature` 枚举中：

```C++
// In dawn/src/dawn/native/Features.h
enum class Feature {
	// [...]
	MSAARenderToSingleSampled,

    // Our custom feature here
    Foo,

    EnumCount,
    InvalidEnum = EnumCount,
    FeatureMin = TextureCompressionBC,
};
```

```{admonition} Update
我在 Dawn 的早期版本上写了这篇文章，最近它已移至自动生成的版本，因此无需担心！
```

然后，我们在 `Features.cpp` 中指定如何在公共 API 和内部 API 之间来回转换：

```C++
// In dawn/src/dawn/native/Features.cpp
Feature FromAPIFeature(wgpu::FeatureName feature) {
	switch (feature) {
		// [...]
		// Add a case for our new feature
		case wgpu::FeatureName::Foo:
			return Feature::Foo;
	}
	return Feature::InvalidEnum;
}

wgpu::FeatureName ToAPIFeature(Feature feature) {
	switch (feature) {
		// [...]
		// Add a case for our new feature
		case Feature::Foo:
			return wgpu::FeatureName::Foo;

		case Feature::EnumCount:
			break;
	}
	UNREACHABLE();
}
```

最后，我们添加一些有关此功能的信息：

```C++
// In dawn/src/dawn/native/Features.cpp
static constexpr FeatureEnumAndInfoList kFeatureNameAndInfoList = {{
    // [...]
    // We add info about our new feature
    {Feature::Foo,
     {"foo",
      "Support our custom FOO feature on render pipelines.",
      "https://eliemichel.github.io/LearnWebGPU/appendices/custom-extensions.html", FeatureInfo::FeatureState::Stable}},
}};
```

```{note}
为了简单起见，我将功能状态保留为 `Stable`。如果要将其设置为 `Experimental`，则必须确保在应用程序代码中启用 `allow_unsafe_apis` 切换。
```

### 后端更改（Vulkan）

好的，现在我们的功能已在内部 API 中正确连接，但到目前为止 **没有后端支持它**！在这个阶段，我们必须专注于**一次一个**。

我们从 **Vulkan** 开始，查看 `dawn/src/dawn/native/vulkan` 的内部。因此，我们首先在应用程序中强制使用 Vulkan 后端：

```C++
RequestAdapterOptions adapterOpts = Default;
// Force Vulkan backend
adapterOpts.backendType = BackendType::Vulkan;
Adapter adapter = instance.requestAdapter(adapterOpts);
```

在 Vulkan 措辞中（以及 Dawn 的内部），可用的功能集由 `PhysicalDevice` 提供。

在 `vulkan/PhysicalDeviceVk.h` 中，我们可以看到特定于该后端的 `PhysicalDevice` 类继承自 `PhysicalDeviceBase` 类，该类由与后端无关的内部 `dawn::native` API 定义。

该基类包含一个受保护的方法 `void EnableFeature(Feature feature)`，子类可以调用该方法来启用特定功能。 **实际上**这是在 `InitializeSupportedFeaturesImpl()` 中完成的，我们在其中添加了我们的功能：

```C++
// In native/vulkan/PhysicalDeviceVk.cpp
void PhysicalDevice::InitializeSupportedFeaturesImpl() {
	// [...]
	// Our feature is always enabled on Vulkan backend:
	EnableFeature(Feature::Foo);
}
```

您现在应该看到支持的功能：

```
适配器支持的功能“Foo”
```

将其添加到**设备描述符**的`requiredFeatures`列表中，然后您可以检查`wgpuDeviceHasFeature(device, WGPUFeatureName_Foo)`是否为真！

渲染管线
---------------

### 公共API

现在让我们实际向此扩展添加一些行为。我们创建了 `RenderPipelineDescriptor` 的扩展，因此我们创建了一个可以链接到该描述符的类型。

**公共 API** 在 `dawn.json` 中处理：我们通过在根范围中的任意位置（例如末尾）添加以下条目来定义扩展链式结构：

```json
"foo render pipeline descriptor": {
    "category": "structure",
    "chained": "in",
    "chain roots": ["render pipeline descriptor"],
    "tags": ["native"],
    "members": [
        {"name": "foo", "type": "uint32_t"}
    ]
},
```

我们还必须将**该结构体的名称**添加到 SType 枚举中：

```json
"s type": {
    "category": "enum",
    "emscripten_no_enum_table": true,
    "values": [
        {"value": 0, "name": "invalid", "valid": false},
        {"value": 1, "name": "surface descriptor from metal layer", "tags": ["native"]},
        /* [...] */
        {"value": 1013, "name": "dawn render pass color attachment render to single sampled", "tags": ["dawn"]},
        /* We add our custom extension here: */
        {"value": 4097, "name": "foo render pipeline descriptor", "tags": ["native"]}
    ]
},
```

我们现在可以尝试在我们的应用程序中使用这个新结构：

```C++
RenderPipelineDescriptor pipelineDesc;
// [...]

// Our custom extension in use in our main.cpp
WGPUFooRenderPipelineDescriptor fooRenderPipelineDesc;
fooRenderPipelineDesc.chain.next = nullptr;
fooRenderPipelineDesc.chain.sType = WGPUSType_FooRenderPipelineDescriptor;
fooRenderPipelineDesc.foo = 42; // some arbitrary value here
pipelineDesc.nextInChain = &fooRenderPipelineDesc.chain;

RenderPipeline pipeline = device.createRenderPipeline(pipelineDesc);
```

````{important}
由于到目前为止，渲染管道描述符尚未通过任何功能进行扩展，因此 `ValidateRenderPipelineDescriptor` 中的健全性检查可确保描述符中没有链接数据。我们必须删除它并检查链的有效性：

```C++
// In dawn/src/dawn/native/RenderPipeline.cpp
MaybeError ValidateRenderPipelineDescriptor(DeviceBase* device,
                                            const RenderPipelineDescriptor* descriptor) {
    // Commend that out:
    //DAWN_INVALID_IF(descriptor->nextInChain != nullptr, "nextInChain must be nullptr.");
    // And add this check:
    DAWN_TRY(ValidateSTypes(descriptor->nextInChain, {{wgpu::SType::FooRenderPipelineDescriptor}}));
    // [...]
}
```
````

我们的应用程序现在可以正确运行，但没有做任何特别的事情。

### 内部API

这次，`RenderPipelineDescriptor` 的内部版本与公共版本相同（它具有不同的类型名称，但为 `reinterpret_cast`-ed）。这样我们就可以直接进行后端的修改了。

### 后端更改（不可知）

我们的示例功能**非常简单**，实际上可以在 `RenderPipelineBase` 类中实现，这**与后端无关**！

我们在结构中添加**两个私有属性**，存储由我们的描述符扩展提供的“foo”的值，以及一个告诉是否提供它的布尔值：

```C++
// In dawn/src/dawn/native/RenderPipeline.h
class RenderPipelineBase : public PipelineBase {
	// [...]

	// Foo extension
    bool mUseFoo = false;
    uint32_t mFoo;
};
```

然后我们**修改构造函数**以从描述符中读取这些值。方便的 `FindInChain` 实用函数**递归地检查扩展链**，寻找正确的 SType：

```C++
// In dawn/src/dawn/native/RenderPipeline.cpp
RenderPipelineBase::RenderPipelineBase(/* [...] */) : /* [...] */ {
	// [...]

	// Handle foo info, if provided
    const FooRenderPipelineDescriptor* fooDesc = nullptr;
    FindInChain(descriptor->nextInChain, &fooDesc);
    if (fooDesc != nullptr) {
        mUseFoo = true;
        mFoo = fooDesc->foo;
    }
}
```

我们还定义了一个 `DoTestFoo` 方法来发出我们的测试日志行：

```C++
// In RenderPipeline.h, in RenderPipelineBase class definition
public:
	// Display a debug log line with the value of foo, is enabled
	void DoTestFoo() const;

// In RenderPipeline.cpp
#include <iostream>
// [...]
void RenderPipelineBase::DoTestFoo() const {
    if (mUseFoo) {
        std::cout << "DEBUG of FOO feature: foo=" << mFoo << std::endl;
    }
}
```

为了在调用 `setPipeline` 时触发该行，我们转到 `RenderEncoderBase` 类的定义：

```C++
// In dawn/src/dawn/native/RenderEncoderBase.cpp
void RenderEncoderBase::APISetPipeline(RenderPipelineBase* pipeline) {
    pipeline->DoTestFoo();
    // [...]
}
```

```{note}
以 `API` 开头的方法名称直接对应于对公共 API 的调用。映射是自动生成的。
```

还有塔达姆！我们能够创建一个自定义扩展，并将额外的“foo”信息从我们的应用程序代码一直传播到 Dawn 的内部。

```
FOO 功能的调试：foo=42
```

当然，这只是一个基本示例，但从那里开始的更改很大程度上取决于您想要实现的实际扩展！

*结果代码：* [`dawn:eliemichel/foo`](https://github.com/eliemichel/Dawn/tree/eliemichel/foo) 和 [`step030-test-foo-extension-dawn`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-test-foo-extension-dawn)
