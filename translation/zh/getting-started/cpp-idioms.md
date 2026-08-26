C++ 包装器<span class="bullet">🟢</span>
===========

```{translation-warning} 译文可能已过时, /getting-started/cpp-idioms.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/028 - C++ Wrapper
:parent: 025 - First Color
```

*结果代码 :* [`step028`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step028)

目前为止,我们使用**原始 WebGPU API**,这是**消费物价指数**这阻止了我们使用一些**C++ 的良好生产特征**本章首先举例说明了如何加以改进,然后将它与一个小库连接起来,在另一个库中执行所有这些功能。`webgpu.hpp`替换页眉`webgpu.h`.

指南的其余部分**总是同时提供片段**此 C++ 包装( 使用)`webgpu.hpp`)和“vanilla”CPI(使用`webgpu.h`).

```{important}
这里介绍的所有变化都只影响编码时间,但是我们浅薄的C++包装器导致同样的运行时间二进制.
```

```{note}
黎明和标语提供了自己的C++包装器. 它们遵循我在这里介绍的类似精神,只是我需要我们的包装纸对所有可能的执行有效,包括:`wgpu-native`.
```

```{caution}
本章未达到[WebGPU-C++的读取器](https://github.com/eliemichel/WebGPU-Cpp)我建议你暂时读一下
```

命名空间
---------

C接口无法使用命名空间,因为它们只在 C++ 中存在,所以您可能注意到每个函数都以`wgpu`每个结构都从`WGPU`。一个更典型的C++方法就是将所有这些功能纳入**命名空间**.

```C++
// Using Vanilla webgpu.h
WGPUInstanceDescriptor desc = {};
WGPUInstance instance = wgpuCreateInstance(&desc);
```

成为命名空间 :

```C++
// Using C++ webgpu.hpp
wgpu::InstanceDescriptor desc = {};
wgpu::Instance instance = wgpu::createInstance(desc);
```

```{note}
你也可以注意到`&`失踪: 这是因为 Cast 描述符指针成为包装符中的引用 !
```

当然,你可以启动您的源文件`using namespace wgpu;`避免拼写`wgpu::`到处都是 加上默认的描述符,这就导致:

```C++
using namespace wgpu;
Instance instance = createInstance();
```

对象
-------

除了命名空间,大多数功能也是**按其第一个参数的类型前缀**例如:

```C++
WGPUBuffer wgpuDeviceCreateBuffer(WGPUDevice device, WGPUBufferDescriptor const * descriptor);
               ^^^^^^             ^^^^^^^^^^^^^^^^^
size_t wgpuAdapterEnumerateFeatures(WGPUAdapter adapter, WGPUFeatureName * features);
           ^^^^^^^                  ^^^^^^^^^^^^^^^^^^^
void wgpuBufferDestroy(WGPUBuffer buffer);
         ^^^^^^        ^^^^^^^^^^^^^^^^^
```

这些职能的概念**方法**他们的第一个论点所构成的目标。 C再次没有对方法的内置支持,但C++有,因此我们在包装中将这些WebGPU功能暴露如下:

```C++
namespace wgpu {
	struct Device {
		// [...]
		createBuffer(BufferDescriptor const * descriptor = nullptr) const;
	};

	struct Adapter {
		// [...]
		enumerateFeatures(WGPUFeatureName * features) const;
	};

	struct Device {
		// [...]
		destroy();
	};
} // namespace wgpu
```

```{note}
那个`const`为一些方法指定了修饰符. 这是包装器为减少潜在的编程错误而提供的额外信息.
```

这在调用这些方法时极大地减少了视觉杂乱:

```C++
// Using Vanilla webgpu.h
WGPUAdapter adapter = /* [...] */
size_t n = wgpuAdapterEnumerateFeatures(adapter, nullptr);
```

成为命名空间 :

```C++
// Using C++ webgpu.hpp
Adapter adapter = /* [...] */
size_t n = adapter.enumerateFeatures(nullptr);
```

范围计数
-------------------

因为Enums是*未覆盖*默认情况下, CPI 被迫**前缀所有值**名字可以指: 名字,可以指: 名字,可以指: 名字,可以指: 名字,可以指:

```C
typedef enum WGPURequestAdapterStatus {
    WGPURequestAdapterStatus_Success = 0x00000000,
    WGPURequestAdapterStatus_Unavailable = 0x00000001,
    WGPURequestAdapterStatus_Error = 0x00000002,
    WGPURequestAdapterStatus_Unknown = 0x00000003,
    WGPURequestAdapterStatus_Force32 = 0x7FFFFFFF
} WGPURequestAdapterStatus;
```

C++ 可以定义**范围**enums,其输入力强,只能通过名称访问,例如此范围enum:

```C++
enum class RequestAdapterStatus {
    Success = 0x00000000,
    Unavailable = 0x00000001,
    Error = 0x00000002,
    Unknown = 0x00000003,
    Force32 = 0x7FFFFFFF
};
```

可用如下:

```C++
wgpu::RequestAdapterStatus::Success;
```

```{note}
实际执行使用一点诡计,使得enum名称被范围化,但含蓄地转换成和从原WebGPU enum值.
```

默认描述值
-------------------------

有时我们只需要默认建造一个描述符. 更一般地说,我们很少需要描述符的所有字段都偏离默认,这样我们就可以从为描述符设置默认构建符的可能性中获益.

控制关闭
------------------

许多同步操作使用回调. 为了给召回的身体提供一些背景,总是有`void *userdata`争吵时有发生 在C++中可以通过捕捉关闭来缓解这种情况.

```{important}
这只缓解了标记,但在创建捕获羊肉时,与用户数据指针非常相似的技术机制会自动执行.
```

```C++
// C style
struct Context {
	WGPUBuffer buffer;
};
auto onBufferMapped = [](WGPUBufferMapAsyncStatus status, void* pUserData) {
	Context* context = reinterpret_cast<Context*>(pUserData);
	std::cout << "Buffer mapped with status " << status << std::endl;
	unsigned char* bufferData = (unsigned char*)wgpuBufferGetMappedRange(context->buffer, 0, 16);
	std::cout << "bufferData[0] = " << (int)bufferData[0] << std::endl;
	wgpuBufferUnmap(context->buffer);
};
Context context;
wgpuBufferMapAsync(buffer, WGPUMapMode_Read, 0, 16, onBufferMapped, (void*)&context);
```

变成

```C++
// C++ style
buffer.mapAsync(buffer, [&context](wgpu::BufferMapAsyncStatus status) {
	std::cout << "Buffer mapped with status " << status << std::endl;
	unsigned char* bufferData = (unsigned char*)context.buffer.getMappedRange(0, 16);
	std::cout << "bufferData[0] = " << (int)bufferData[0] << std::endl;
	context.buffer.unmap();
});
```

图书馆
-------

提供这些 C++ idioms 的库是[`webgpu.hpp`](../../../data/webgpu.hpp)。它是由单个标题文件组成的,只需要复制到源代码树上即可。 您的其中一个源文件必须定义`WEBGPU_CPP_IMPLEMENTATION`在此之前`#include "webgpu.hpp"`:

```C++
#define WEBGPU_CPP_IMPLEMENTATION
#include <webgpu/webgpu.hpp>
```

```{note}
此信头实际上包含在 WebGPU zip 中, 我先前提供给您的, 在包含路径`<webgpu/webgpu.hpp>`.
```

更多信息,请访问[网络gpu-cpp 仓库](https://github.com/eliemichel/WebGPU-Cpp).

*结果代码 :* [`step028`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step028)

```{lit} C++, Includes (replace, hidden)
#define WEBGPU_CPP_IMPLEMENTATION
#include <webgpu/webgpu.hpp>

#include <GLFW/glfw3.h>
#include <glfw3webgpu.h>

#ifdef __EMSCRIPTEN__
#  include <emscripten.h>
#endif // __EMSCRIPTEN__

#include <iostream>
#include <cassert>
#include <vector>

using namespace wgpu;
```

```{lit} C++, Application attributes (replace, hidden)
GLFWwindow *window;
Device device;
Queue queue;
Surface surface;
std::unique_ptr<ErrorCallback> uncapturedErrorCallbackHandle;
```

```{lit} C++, Initialize (replace, hidden)
{{Open window and get adapter}}

{{Request device}}
queue = device.getQueue();

{{Add device error callback}}

{{Surface Configuration}}

// We no longer need to access the adapter
adapter.release();
```

```{lit} C++, Open window and get adapter (replace, hidden)
// Open window
glfwInit();
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
window = glfwCreateWindow(640, 480, "Learn WebGPU", nullptr, nullptr);

// Create instance
Instance instance = wgpuCreateInstance(nullptr);

// Get adapter
std::cout << "Requesting adapter..." << std::endl;
{{Request adapter}}
std::cout << "Got adapter: " << adapter << std::endl;

// We no longer need to access the instance
instance.release();
```

```{lit} C++, Add device error callback (replace, hidden)
// Device error callback
uncapturedErrorCallbackHandle = device.setUncapturedErrorCallback([](ErrorType type, char const* message) {
	std::cout << "Uncaptured device error: type " << type;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
});
```

```{lit} C++, Request device (replace, hidden)
// Get device
std::cout << "Requesting device..." << std::endl;
DeviceDescriptor deviceDesc = {};
{{Build device descriptor}}
device = adapter.requestDevice(deviceDesc);
std::cout << "Got device: " << device << std::endl;
```

```{lit} C++, Build device descriptor (replace, hidden)
deviceDesc.label = "My Device";
deviceDesc.requiredFeatureCount = 0;
deviceDesc.requiredLimits = nullptr;
deviceDesc.defaultQueue.nextInChain = nullptr;
deviceDesc.defaultQueue.label = "The default queue";
deviceDesc.deviceLostCallback = [](WGPUDeviceLostReason reason, char const* message, void* /* pUserData */) {
	std::cout << "Device lost: reason " << reason;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
};
```

```{lit} C++, Request adapter (replace, hidden)
{{Get the surface}}

RequestAdapterOptions adapterOpts = {};
adapterOpts.compatibleSurface = surface;

Adapter adapter = instance.requestAdapter(adapterOpts);
```

```{lit} C++, Surface Configuration (replace, hidden)
SurfaceConfiguration config = {};

{{Describe Surface Configuration}}

surface.configure(config);
```

```{lit} C++, Describe Surface Configuration (replace, hidden)
// Configuration of the textures created for the underlying swap chain
config.width = 640;
config.height = 480;
{{Describe Surface Usage}}
{{Describe Surface Format}}
config.device = device;
config.presentMode = PresentMode::Fifo;
config.alphaMode = CompositeAlphaMode::Auto;
```

```{lit} C++, Describe Surface Format (replace, hidden)
TextureFormat surfaceFormat = surface.getPreferredFormat(adapter);
config.format = surfaceFormat;
// And we do not need any particular view format:
config.viewFormatCount = 0;
config.viewFormats = nullptr;
```

```{lit} C++, Describe Surface Usage (replace, hidden)
config.usage = TextureUsage::RenderAttachment;
config.device = device;
```

```{lit} C++, Terminate (replace, hidden)
surface.unconfigure();
queue.release();
surface.release();
device.release();
glfwDestroyWindow(window);
glfwTerminate();
```

```{lit} C++, Get the next target texture view (replace, hidden)
// Get the next target texture view
auto [ surfaceTexture, targetView ] = GetNextSurfaceViewData();
if (!targetView) return;
```

```{lit} C++, GetNextSurfaceViewData method (replace, hidden)
std::pair<SurfaceTexture, TextureView> Application::GetNextSurfaceViewData() {
    {{Get the next surface texture}}
    {{Create surface texture view}}
    {{Release the texture}}
    return { surfaceTexture, targetView };
}
```

```{lit} C++, Private methods (replace, hidden)
private:
    std::pair<SurfaceTexture, TextureView> GetNextSurfaceViewData();
```

```{lit} C++, Get the next surface texture (replace, hidden)
SurfaceTexture surfaceTexture;
surface.getCurrentTexture(&surfaceTexture);
Texture texture = surfaceTexture.texture;
if (surfaceTexture.status != SurfaceGetCurrentTextureStatus::Success) {
    return { surfaceTexture, nullptr };
}
```

```{lit} C++, Create surface texture view (replace, hidden)
TextureViewDescriptor viewDescriptor;
viewDescriptor.label = "Surface texture view";
viewDescriptor.format = texture.getFormat();
viewDescriptor.dimension = TextureViewDimension::_2D;
viewDescriptor.baseMipLevel = 0;
viewDescriptor.mipLevelCount = 1;
viewDescriptor.baseArrayLayer = 0;
viewDescriptor.arrayLayerCount = 1;
viewDescriptor.aspect = TextureAspect::All;
TextureView targetView = texture.createView(viewDescriptor);
```

```{lit} C++, Release the texture (replace, hidden)
#ifndef WEBGPU_BACKEND_WGPU
    // We no longer need the texture, only its view
    // (NB: with wgpu-native, surface textures must not be manually released)
    Texture(surfaceTexture.texture).release();
#endif // WEBGPU_BACKEND_WGPU
```

```{lit} C++, Create Command Encoder (replace, hidden)
CommandEncoderDescriptor encoderDesc = {};
encoderDesc.label = "My command encoder";
CommandEncoder encoder = device.createCommandEncoder(encoderDesc);
```

```{lit} C++, Encode Render Pass (replace, hidden)
RenderPassDescriptor renderPassDesc = {};

{{Describe Render Pass}}

RenderPassEncoder renderPass = encoder.beginRenderPass(renderPassDesc);
{{Use Render Pass}}
renderPass.end();
renderPass.release();
```

```{lit} C++, Describe Render Pass (replace, hidden)
RenderPassColorAttachment renderPassColorAttachment = {};

{{Describe the attachment}}

renderPassDesc.colorAttachmentCount = 1;
renderPassDesc.colorAttachments = &renderPassColorAttachment;
renderPassDesc.depthStencilAttachment = nullptr;
renderPassDesc.timestampWrites = nullptr;
```

```{lit} C++, Describe the attachment (replace, hidden)
renderPassColorAttachment.view = targetView;
renderPassColorAttachment.resolveTarget = nullptr;
renderPassColorAttachment.loadOp = LoadOp::Clear;
renderPassColorAttachment.storeOp = StoreOp::Store;
renderPassColorAttachment.clearValue = Color{ 0.9, 0.1, 0.2, 1.0 };
#ifndef WEBGPU_BACKEND_WGPU
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
#endif // NOT WEBGPU_BACKEND_WGPU
```

```{lit} C++, Finish encoding and submit (replace, hidden)
// Finally encode and submit the render pass
CommandBufferDescriptor cmdBufferDescriptor = {};
cmdBufferDescriptor.label = "Command buffer";
CommandBuffer command = encoder.finish(cmdBufferDescriptor);
encoder.release();

std::cout << "Submitting command..." << std::endl;
queue.submit(1, &command);
command.release();
std::cout << "Command submitted." << std::endl;
```

```{lit} C++, Present the surface onto the window (replace, hidden)
targetView.release();
#ifndef __EMSCRIPTEN__
surface.present();
#endif
```

```{lit} C++, Poll WebGPU Events (replace, hidden)
#if defined(WEBGPU_BACKEND_DAWN)
	device.tick();
#elif defined(WEBGPU_BACKEND_WGPU)
	device.poll(false);
#endif
```

```{lit} CMake, Define app target (replace, hidden)
{{Dependency subdirectories}}

add_executable(App
	{{App source files}}
)

{{Link libraries}}
```

```{lit} CMake, App source files (replace, hidden)
main.cpp
```

```{lit} C++, file: webgpu-utils.h (replace, hidden)
```

```{lit} C++, file: webgpu-utils.cpp (replace, hidden)
```
