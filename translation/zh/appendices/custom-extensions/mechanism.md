扩展机制（<span class="bullet">🟠</span>WIP）
=======================

```{translation-warning} 译文可能已过时, /appendices/custom-extensions/mechanism.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{admonition} Disclaimer
本指南的这一部分探讨了在官方 `webgpu.h` 的幕后运行的 **内部 API**。这些内容非常**可能会发生变化**，因为它们是开发人员自己的约定，并不意味着要记录下来。

我尽力不时更新本节，但示例代码**可能无法在较新的版本上按原样**工作。 **无论如何，整体想法应该仍然成立**，如果您遇到麻烦，我邀请您在 [Discord 服务器](https://discord.gg/2Tar4Kt564) 上分享您的实验！
```

描述符扩展
---------------------

您一定已经注意到所有描述符中都存在这个 `nextInChain` 指针。这是一种向描述符添加额外字段的方法，后端可能会也可能不会读取这些字段，具体取决于后端是否识别扩展名。

### 结构类型

创建新扩展时，我们首先需要选择扩展 `SType` （“结构类型”）标识符。此标识符可以是适合 32 位整数的任何值，但**某些值是保留的**。例如，第一个值对应于官方扩展：

```C++
// The SType identifiers defined in the standard webgpu.h header:
typedef enum WGPUSType {
    WGPUSType_Invalid = 0x00000000,
    WGPUSType_SurfaceDescriptorFromMetalLayer = 0x00000001,
    WGPUSType_SurfaceDescriptorFromWindowsHWND = 0x00000002,
    WGPUSType_SurfaceDescriptorFromXlibWindow = 0x00000003,
    WGPUSType_SurfaceDescriptorFromCanvasHTMLSelector = 0x00000004,
    WGPUSType_ShaderModuleSPIRVDescriptor = 0x00000005,
    WGPUSType_ShaderModuleWGSLDescriptor = 0x00000006,
    WGPUSType_PrimitiveDepthClipControl = 0x00000007,
    WGPUSType_SurfaceDescriptorFromWaylandSurface = 0x00000008,
    WGPUSType_SurfaceDescriptorFromAndroidNativeWindow = 0x00000009,
    WGPUSType_SurfaceDescriptorFromXcbWindow = 0x0000000A,
    WGPUSType_RenderPassDescriptorMaxDrawCount = 0x0000000F,
    WGPUSType_Force32 = 0x7FFFFFFF
} WGPUSType;
```

每个后端都选择自己的值范围，尽可能避免冲突：`wgpu-native` 从 `0x60000001` (`1610612737`) 开始，Dawn 从 `0x000003E8` (`1000`) 开始。

```{note}
在不久的将来，Dawn 将迁移到 `0x20000` (`131072`)，wgpu-native 迁移到 `0x30000` (`196608`)。
```

```C++
// Extensions specific to wgpu-native, defined in wgpu.h
typedef enum WGPUNativeSType {
    // Start at 6 to prevent collisions with webgpu STypes
    WGPUSType_DeviceExtras = 0x60000001,
    WGPUSType_AdapterExtras = 0x60000002,
    WGPUSType_RequiredLimitsExtras = 0x60000003,
    WGPUSType_PipelineLayoutExtras = 0x60000004,
    WGPUSType_ShaderModuleGLSLDescriptor = 0x60000005,
    WGPUSType_SupportedLimitsExtras = 0x60000003,
    WGPUSType_InstanceExtras = 0x60000006,
    WGPUSType_SwapChainDescriptorExtras = 0x60000007,
    WGPUNativeSType_Force32 = 0x7FFFFFFF
} WGPUNativeSType;
```

```C++
// Dawn provides its extensions in its modified webgpu.h
typedef enum WGPUSType {
    WGPUSType_Invalid = 0x00000000,
    //[...] The standard extensions
    WGPUSType_DawnTextureInternalUsageDescriptor = 0x000003E8,
    WGPUSType_DawnTogglesDeviceDescriptor = 0x000003EA,
    WGPUSType_DawnEncoderInternalUsageDescriptor = 0x000003EB,
    WGPUSType_DawnInstanceDescriptor = 0x000003EC,
    WGPUSType_DawnCacheDeviceDescriptor = 0x000003ED,
    WGPUSType_DawnAdapterPropertiesPowerPreference = 0x000003EE,
    WGPUSType_DawnBufferDescriptorErrorInfoFromWireClient = 0x000003EF,
    WGPUSType_DawnTogglesDescriptor = 0x000003F0,
    WGPUSType_DawnShaderModuleSPIRVOptionsDescriptor = 0x000003F1,
    WGPUSType_Force32 = 0x7FFFFFFF
} WGPUSType;
```

> 😐 那么我应该如何为我的扩展选择一个值呢？

**不要**在已经存在的值之后**使用接近的值。每个后端和标准标头可能会在它们已经使用的值之后添加新值，因此如果您使用了一个值，您将遇到**冲突**。

目前还没有特别推荐的范围（我想最终会有更多指导，但目前对此扩展机制的反馈很少）。只需选择一个与其他值足够远的基值，然后依次在其中添加扩展即可。

为了示例，我使用 `0x40000` (`262144`)。在自定义 `webgpu-ext-foo.h` 中，我定义了我的扩展 `SType`。为了引入我们的新功能“Foo”，我们可能需要为多个描述符添加扩展。

```C++
// In file webgpu-ext-foo.h we define the API of our extension "Foo"

// We give our enum a name that is a variant of WGPUSType
typedef enum WGPUFooSType {
	// Our new extension, for instance to extend the Render Pipeline
	WGPUFooSType_FooRenderPipelineDescriptor = 0x00040001,

	// Force the enum value to be represented on 32-bit integers
	WGPUFooSType_Force32 = 0x7FFFFFFF
} WGPUFooSType; // (set enum name here as well)
```

```{important}
不要忘记 `WGPUNativeSType_Force32 = 0x7FFFFFFF`，它用于确保 C++ 编译器不会优化小于 32 位的枚举值的表示。
```

```{note}
在接下来的部分中，我们在这里介绍 **一个非常简单且无趣的扩展**，称为 **foo**，它将一个可选的 `foo` 整数成员添加到 `RenderPipeline` 中，并在渲染通道编码器上调用 `setPipeline` 时将其显示在调试行中。
```

### 结构字段

让我们跟踪一下 WebGPU 设备接收描述符时的逻辑：

1. 查看`nextInChain`是否不为空。它的类型为 `WGPUChainedStruct`，这意味着它的第一个字段是另一个链指针 `next`，它的第二个字段是一个名为 `sType` 的整数。
2. 查看值 `nextInChain->sType` 并检查它是否知道该值。
3. 如果设备确实知道，则设备会知道 `nextInChain` 实际上指向一个 **不仅仅是** `WGPUChainedStruct` 的结构，而是一个以相同方式开始但具有 **额外字段** 的结构。
4. 设备**将** `nextInChain` 转换为该已知结构并读取额外字段。
5. 重复步骤 1，将 `nextInChain` 替换为 `nextInChain->next`。

扩展的核心在于步骤 3。我们的扩展必须提供此结构的定义，该定义以 `WGPUChainedStruct` 开头，并且用户代码和后端代码都同意。

要像 `WGPUChainedStruct` 一样开始，我们使用 C 惯用继承机制：让结构的**第一个属性**为 `WGPUChainedStruct` 类型。这确保结构体的实例可以转换为 `WGPUChainedStruct`：

```C++
// In file webgpu-ext-foo.h
typedef struct WGPUFooRenderPipelineDescriptor {
	// This first field is conventionally called 'chain'
	WGPUChainedStruct chain;

	// Your custom fields here
	uint32_t foo;
} WGPUFooRenderPipelineDescriptor;
```

在最终用户代码中，将按如下方式使用：

```C++
WGPUFooRenderPipelineDescriptor fooDesc;
// Set next to NULL or whatever else if you use other extensions as well
fooDesc.chain.next = NULL;

// Always set this to WGPUSType_FooRenderPipelineDescriptor when the struct that
// contains the "chain" field has actually type WGPUFooRenderPipelineDescriptor.
fooDesc.chain.sType = WGPUSType_FooRenderPipelineDescriptor;

// Set Foo description fields
fooDesc.foo = 0;

// Use in the nextInChain of the standard descriptor it targets
WGPURenderPipelineDescriptor desc;
desc.nextInChain = &fooDesc.chain;
```

适配器特点
----------------

引入新扩展时，我们必须通过在适配器中添加新的**功能**来向用户代码通告其可用性。因此，在启动时检查适配器功能时，最终用户代码可以确定是否允许使用 not 扩展。

其逻辑与 `SType` 非常相似：有标准的逻辑，以及后端特定的逻辑，它们的索引与 SType 相同。

```C++
// Standard adapter features
typedef enum WGPUFeatureName {
    WGPUFeatureName_Undefined = 0x00000000,
    WGPUFeatureName_DepthClipControl = 0x00000001,
    WGPUFeatureName_Depth32FloatStencil8 = 0x00000002,
    WGPUFeatureName_TimestampQuery = 0x00000003,
    // [...]
    WGPUFeatureName_Force32 = 0x7FFFFFFF
} WGPUFeatureName;
```

```C++
// Here is wgpu-native's wgpu.h
typedef enum WGPUNativeFeature {
    WGPUNativeFeature_PushConstants = 0x60000001,
    WGPUNativeFeature_TextureAdapterSpecificFormatFeatures = 0x60000002,
    WGPUNativeFeature_MultiDrawIndirect = 0x60000003,
    WGPUNativeFeature_MultiDrawIndirectCount = 0x60000004,
    WGPUNativeFeature_VertexWritableStorage = 0x60000005,
    WGPUNativeFeature_Force32 = 0x7FFFFFFF
} WGPUNativeFeature;
```

```C++
// And here are Dawn features.
typedef enum WGPUFeatureName {
    WGPUFeatureName_Undefined = 0x00000000,
    // [...] The standard feature names
    WGPUFeatureName_DawnShaderFloat16 = 0x000003E9,
    WGPUFeatureName_DawnInternalUsages = 0x000003EA,
    WGPUFeatureName_DawnMultiPlanarFormats = 0x000003EB,
    WGPUFeatureName_DawnNative = 0x000003EC,
    WGPUFeatureName_ChromiumExperimentalDp4a = 0x000003ED,
    WGPUFeatureName_TimestampQueryInsidePasses = 0x000003EE,
    WGPUFeatureName_ImplicitDeviceSynchronization = 0x000003EF,
    WGPUFeatureName_Force32 = 0x7FFFFFFF
} WGPUFeatureName;
```

我们可以添加自己的“Foo”功能，仍在我们的 `webgpu-ext-foo.h` 文件中：

```C++
// We give our enum a name that is a variant of WGPUFeatureName
typedef enum WGPUFooFeatureName {
	// Our new feature name
	WGPUFooFeatureName_Foo = 0x00040001,

	// Force the enum value to be represented on 32-bit integers
	WGPUFooFeatureName_Force32 = 0x7FFFFFFF
} WGPUFooFeatureName; // (set enum name here as well)
```

我们所拥有的 `webgpu-ext-foo.h` 文件就是我们所需要的作为用户代码和修改后的后端之间的接口。为了实现此标头，我们需要选择要编辑的后端。

接下来的两章分别关注[`wgpu-native`](with-wgpu-native.md)，然后是[Dawn](with-dawn.md)，以展示如何实现这个基本的Foo扩展的更多**内部细节**。
