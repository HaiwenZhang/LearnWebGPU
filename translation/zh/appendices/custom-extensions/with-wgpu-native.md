与 `wgpu-native` (<span class="bullet">🟠</span>WIP)
==================

```{translation-warning} 译文可能已过时, /appendices/custom-extensions/with-wgpu-native.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码：* [`wgpu`](https://github.com/eliemichel/wgpu/tree/eliemichel/foo)、[`wgpu-native`](https://github.com/eliemichel/wgpu-native/tree/eliemichel/foo) 和 [`step030-test-foo-extension`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-test-foo-extension)

设置
-----

在修改任何代码之前，我们需要设置`wgpu-native`所依赖的**2个存储库**。

### 建筑 `wgpu-native`

首先使用[其存储库中的说明](https://github.com/gfx-rs/wgpu-native/wiki/Getting-Started) 构建“常规”`wgpu-native`。您特别需要 [rust](https://www.rust-lang.org/) 和 [LLVM/Clang](https://rust-lang.github.io/rust-bindgen/requirements.html)。安装完这些后，建筑物看起来像这样：

```bash
git clone --recurse-submodules https://github.com/gfx-rs/wgpu-native
cd wgpu-native
set LIBCLANG_PATH=E:/Libraries/libclang/bin
cargo build --release
```

```{important}
将上面的 `LIBCLANG_PATH` 值调整为您实际安装的 LLVM/Clang。
```

然后您可以在 `target\release` 中找到二进制文件。如果您在最终项目中使用基于`wgpu-native`的[WebGPU-distribution](https://github.com/eliemichel/WebGPU-distribution/tree/wgpu)，只需替换`webgpu/bin`中的相关文件即可。

### 建筑 `wgpu`

`wgpu-native` 存储库是一个**薄层**，将实际的 `wgpu` 后端公开为 C 接口。创建自定义扩展时，我们需要更改后端，并指示 `wgpu-native` 层使用我们自定义的 `wgpu` 分支。

```bash
git clone https://github.com/gfx-rs/wgpu
cd wgpu
cargo build --release
```

要将 `wgpu-native` 指向我们的自定义 `wgpu`，我们可以修改其 `wgpu-native/Cargo.toml` 并添加：

```toml
[patch."https://github.com/gfx-rs/wgpu"]
wgpu-core = { path = "../wgpu/wgpu-core" }
wgpu-types = { path = "../wgpu/wgpu-types" }
wgpu-hal = { path = "../wgpu/wgpu-hal" }
naga = { path = "../wgpu/naga" }
```

````{note}
为了确保重现完全相同的二进制文件，请在 `wgpu` 目录中查看 `wgpu-native` 的 `Cargo.toml` 中 `rev =` 之后指定的提交。

```bash
cd wgpu
git checkout 011a4e26d04f388ef40e3baee3f19a255b9b5148
```

但由于您正在编写扩展，您可能想使用 `wgpu` 的最新版本。
````

您可能还需要更新 `[dependencies.naga]` 的版本哈希值以匹配您的 `wgpu` 版本所使用的版本。

Foo 扩展
-----------------

### 扩展类型

我们可以将 `webgpu-ext-foo.h` 标头复制到 `wgpu-native/ffi/` 目录中，位于 `wgpu.h` 旁边。为了让 Rust 的构建系统解析这些文件，我们将自定义标头添加到 `build.rs` 中：

```rust
让 mut builder = bindgen::Builder::default()
.header("ffi/webgpu-headers/webgpu.h")
.header("ffi/wgpu.h")
.header("ffi/webgpu-ext-foo.h") // <-- 这是我们的自定义文件
    // [...]
```

这在 Rust 的 `native` 命名空间符号中定义了与 C 头文件公开的符号等效的符号。我们还必须修改 `wgpu-types/src/lib.rs` 中的一些现有类型。至少要添加我们的新适配器功能：

```rust
位标志::位标志！ {
	// [...]
pub 结构特点： u64 {
		// [...]
常量 SHADER_EARLY_DEPTH_TEST = 1 << 62;

// 我们的特征作为位标志
常量 FOO = 1 << 63；
	}
}
```

在我们运行的示例中，我们想要将字段 `foo` 添加到渲染管道。因此我们修改`wgpu/wgpu-core/src/pipeline.rs`中的`RenderPipelineDescriptor`：

```rust
pub 结构 RenderPipelineDescriptor<'a> {
	// [...]
/// 我们的自定义新字段
酒吧foo：选项<u32>，
}
```

````{note}
当尝试从 `wgpu` 根目录构建时，不要尝试构建整个项目，我们仅使用 `wgpu-core` 和 `wgpu-types` （前者依赖于前者），因此请尝试使用以下命令进行构建：

```bash
wgpu$ cargo build --package wgpu-core
```

但更有可能的是，您将从 `wgpu-native` 项目构建：

```bash
wgpu-native$ cargo build
```

如果您希望您的扩展也可供 Rust 用户使用，您还必须调整 `wgpu` 包，但我不会在这里介绍它。
````

### 扩展原生包装器

用于将 `wgpu/wgpu-types` 中定义的 Rust 端类型与 C 标头后面的 `native` 定义链接起来的转换实用程序。将您的功能名称添加到 `wgpu-native/src/conv.rs` 中的 `features_to_native` 和 `map_feature` 中：

```rust
pub fn features_to_native(features: wgt::Features) -> Vec<native::WGPUFeatureName> {
    // [...]
if features.contains(wgt::Features::FOO) {
temp.push(native::WGPUFeatureName_Foo);
    }
温度
}

pub fn map_feature(feature: native::WGPUFeatureName) -> Option<wgt::Features> {
使用 wgt:: 功能；
匹配特征{
        // [...]
本机::WGPUFeatureName_Foo => 一些(功能::FOO),
        // [...]
    }
}
```

为了在 `RenderPipelineDescriptor` 的扩展链中传递时识别我们的 `SType`，我们修改 `wgpu-native/src/lib.rs` 中的 `wgpuDeviceCreateRenderPipeline` 过程：

```rust
pub 不安全 extern "C" fn wgpuDeviceCreateRenderPipeline(
设备：本机::WGPUDevice，
描述符：选项<&native::WGPURenderPipelineDescriptor>，
) -> 本机::WGPURenderPipeline {
let (设备, 上下文) = device.unwrap_handle();
让描述符=描述符.expect(“无效的描述符”);

让 desc = wgc::pipeline::RenderPipelineDescriptor {
        // [...]

// 迭代探索扩展链，直到找到扩展
// 我们的类型，转换为 WGPUFooRenderPipelineDescriptor 并检索
// foo 的值。
foo: follow_chain!(
地图渲染管道描述符（
描述符，
WGPUFooSType_FooRenderPipelineDescriptor => 本机::WGPUFooRenderPipelineDescriptor)
        ),
    };
}
```

这调用了我们在 `conv.rs` 中创建的 `map_render_pipeline_descriptor`：

```rust
pub 不安全 fn map_render_pipeline_descriptor<'a>(
_: &native::WGPURenderPipelineDescriptor,
foo：选项<&native::WGPUFooRenderPipelineDescriptor>，
) -> 选项<u32> {
foo.map(|x| x.foo)
}
```

```{note}
不要忘记将 `map_render_pipeline_descriptor` 添加到 `use conv::{...}` 行的 `lib.rs` 开头。
```

### 扩展核心

如果您的扩展足够浅，不会影响后端，则只需修改 `wgpu/wgpu-core` 即可。但如果您花时间编写自定义扩展，则可能需要修改一个或多个后端。

### 扩展后端

```{warning}
TODO：我仍然需要更好地了解代码是如何组织的。到目前为止我注意到：

- `wgpu-native` 将 C 入口点映射到 `wgpu-core` 的 Rust API
- `wgpu-core` 维护通用用户 API，即基于 wgpu 使用的应用程序，通过本机包装器或通过 `wgpu-rs` （也称为 `wgpu`）。在幕后，它将指令映射到 `wgpu-hal`
- `wgpu-hal` 是后端/硬件抽象层，它定义了每个后端（Vulkan、Metal、DX12 等）必须实现的内部 API
- `wgpu-hal/vulkan` 是 Vulkan 后端，实现 HAL 的所有要求
```

文件 `wgpu/wgpu-hal/src/lib.rs` 定义每个后端必须实现的接口。后端是 `wgpu/wgpu-hal/src` 的子目录以及 `empty.rs` 文件，定义不执行任何操作的默认行为。

我们一一实现后端，也许只针对我们在实践中感兴趣的后端。因此，我们必须确保适配器仅针对我们实现的后端通告 `Foo` 功能。

让我们从 Vulkan 后端开始。我们首先宣传适配器（Vulkan 中的“物理设备”）支持我们的功能。

```{note}
当然，仅当您想要实现的机制确实受支持时，您才可以检查实际的物理设备属性以有条件地列出 `FOO` 功能。
```

```rust
// 在 wgpu/wgpu-hal/src/vulkan/adapter.rs 中
impl PhysicalDeviceFeatures {
    // [...]
fn 到_wgpu(
        /* [...] */
) -> (wgt::功能, wgt::DownlevelFlags) {
        // [...]
让 mut features = F::empty()
| F::SPIRV_SHADER_PASSTHROUGH
| F::MAPPABLE_PRIMARY_BUFFERS
            // [...]
| F：：FOO； // 我们的扩展在这里！
    }
    // [...]
}
```

现在在 DirectX 12 后端：

```rust
// 在 wgpu/wgpu-hal/src/dx12/adapter.rs 中
实现超级::适配器{
    // [...]
pub(超级) fn 暴露(
        /* [...] */
) -> 选项<crate::ExposedAdapter<super::Api>> {
        // [...]
让 mut features = wgt::Features::empty()
| wgt::功能::DEPTH_CLIP_CONTROL
| wgt::功能::DEPTH32FLOAT_STENCIL8
            // [...]
| wgt::功能::FOO; // 我们的扩展在这里！
    }
    // [...]
}
```

```{note}
在本节的其余部分中，我们仅关注 Vulkan 后端。
```

现在，我们可以检查该功能是否在我们的应用程序代码中正确可用。我从 [`step030-headless`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless) 分支开始。

```C++
if (wgpuAdapterHasFeature(adapter, (WGPUFeatureName)WGPUFeatureName_Foo)) {
    std::cout << "Feature 'Foo' supported by adapter" << std::endl;
}
else {
    std::cout << "Feature 'Foo' NOT supported by adapter" << std::endl;
}
```

复制自定义 `wgpu-native` 的 DLL 和标头后，您应该会看到支持的 Foo 功能。

````{note}
您可以通过在实例描述符中使用 `wgpu-native` 的 **instance extras** 扩展来强制使用特定后端：

```C++
WGPUInstanceExtras instanceExtras = {};
instanceExtras.chain.next = nullptr;
instanceExtras.chain.sType = (WGPUSType)WGPUSType_InstanceExtras;
instanceExtras.backends = WGPUInstanceBackend_Vulkan; // We only accept Vulkan adapters here!
WGPUInstanceDescriptor instanceDesc = {};
instanceDesc.nextInChain = &instanceExtras.chain;

Instance instance = createInstance(instanceDesc);
```
````

不要忘记在创建设备时请求该功能：

```C++
WGPUFeatureName featureFoo = (WGPUFeatureName)WGPUFeatureName_Foo;
deviceDesc.requiredFeaturesCount = 1;
deviceDesc.requiredFeatures = &featureFoo;
// [...]

if (wgpuDeviceHasFeature(device, (WGPUFeatureName)WGPUFeatureName_Foo)) {
    std::cout << "Feature 'Foo' supported by device" << std::endl;
}
else {
    std::cout << "Feature 'Foo' NOT supported by device" << std::endl;
}
```

然后我们可以尝试扩展渲染管道：

```C++
RenderPipelineDescriptor pipelineDesc;
// [...]

WGPUFooRenderPipelineDescriptor fooRenderPipelineDesc;
fooRenderPipelineDesc.chain.next = nullptr;
fooRenderPipelineDesc.chain.sType = (WGPUSType)WGPUFooSType_FooRenderPipelineDescriptor;
fooRenderPipelineDesc.foo = 42; // some test value here.
pipelineDesc.nextInChain = &fooRenderPipelineDesc.chain;

RenderPipeline pipeline = device.createRenderPipeline(pipelineDesc);
```

为了测试该值是否正确传播，只要为此管道调用 `setPipeline` ，我们就打印一条日志行：

```rust
// 在 wgpu/wgpu-hal/src/vulkan/command.rs 中
不安全 fn set_render_pipeline(&mut self, pipeline: &super::RenderPipeline) {
// 调试打印
if let Some(foo) = pipeline.foo {
println!("调试 FOO 功能：foo={:?}", foo);
    }

    // [...]
}
```

我们需要向 RenderPipeline 添加一个 `foo` 字段（我们现在只将其添加到描述符中）：

```rust
// 在 wgpu/wgpu-hal/src/vulkan/mod.rs 中
#[导出（调试）]
pub 结构 RenderPipeline {
原始：vk::管道，
foo: Option<u32>, // 添加这个
}
```

我们在创建渲染管道时从描述符进行传播：

```rust
// 在 wgpu/wgpu-hal/src/vulkan/device.rs 中
不安全 fn create_render_pipeline(
&自我，
desc: &crate::RenderPipelineDescriptor<super::Api>,
) -> 结果<super::RenderPipeline, crate::PipelineError> {
    // [...]
让 foo = desc.foo;

好的（超级::RenderPipeline { raw, foo }）
}
```

但是，您可能会注意到，这是 `RenderPipelineDescriptor` 类型（在 `wgpu-hal/lib.rs` 中定义）的 **又一** 风格。由于这变得相当混乱，让我用一张图来总结一下：

```{image} /images/extension/wgpu-architecture-light.svg
:align: center
:class: only-light
```

```{image} /images/extension/wgpu-architecture-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
<span class="caption-text"><em>wgpu-native项目的架构.</em></span>
</p>

```rust
// 在 wgpu/wgpu-hal/lib.rs 中
#[派生（克隆，调试）]
pub 结构 RenderPipelineDescriptor<'a, A: Api> {
    // [...]
/// 我们的自定义 FOO 扩展
酒吧foo：选项<u32>，
}

// 在 wgpu/wgpu-core/src/device/resource.rs 中
pub(超级) fn create_render_pipeline<G: GlobalIdentityHandlerFactory>(
    /* [...] */
) -> Result<pipeline::RenderPipeline<A>, pipeline::CreateRenderPipelineError> {
    // [...]
让 pipeline_desc = hal::RenderPipelineDescriptor {
        // [...]
多视图：desc.multiview，
foo: desc.foo, // 在此处转发我们的自定义成员
    };
    // [...]
}
```

在此阶段，如果您构建 `wgpu-native`，更新 C++ 应用程序中的 DLL 并运行该应用程序，它最终应该显示测试日志行：

```
FOO 功能的调试：foo=42
```

**恭喜**，您拥有了第一个 `wgpu-native` 分机！当然它做的并不多，而且只在Vulkan后端做这样的事情。但我们已经探索了该项目的架构。现在剩下的内容很大程度上取决于您想要实施的具体扩展！

*结果代码：* [`wgpu`](https://github.com/eliemichel/wgpu/tree/eliemichel/foo)、[`wgpu-native`](https://github.com/eliemichel/wgpu-native/tree/eliemichel/foo) 和 [`step030-test-foo-extension`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-test-foo-extension)
