无头上下文 <span class="bullet">🟡</span>
================

```{translation-warning} 译文可能已过时, /advanced-techniques/headless.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step030-headless`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-headless-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless-vanilla)
````

有时，需要在根本不打开窗口的情况下使用 GPU。使用 WebGPU 可以很容易地做到这一点！

```{note}
本章的代码基于[Hello Triangle](../basic-3d-rendering/hello-triangle.md)中的[`step030`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030)。
```

在我们的 *Hello Triangle* 代码中，WebGPU 仅在两个位置与窗口交互：

1. 创建适配器时。
2. 设置交换链时。

适配器创建
----------------

我们的第一个更改是将适配器选项的 `compatibleSurface` 替换为 `nullptr`：

````{tab} With webgpu.hpp
```C++
RequestAdapterOptions adapterOpts;
adapterOpts.compatibleSurface = nullptr;
//                              ^^^^^^^ This was 'surface'
Adapter adapter = instance.requestAdapter(adapterOpts);
```
````

````{tab} Vanilla webgpu.h
```C++
WGPURequestAdapterOptions adapterOpts = {};
adapterOpts.nextInChain = nullptr;
adapterOpts.compatibleSurface = nullptr;
//                              ^^^^^^^ This was 'surface'
WGPUAdapter adapter = requestAdapter(instance, &adapterOpts);
```
````

交换链
----------

交换链是一种旨在在窗口上无缝显示渲染图像的机制。当我们不再有窗口时......我们只是**不再需要交换链**！

````{tab} With webgpu.hpp
```C++
// Remove all this:
std::cout << "Creating swapchain..." << std::endl;
// [...]
std::cout << "Swapchain: " << swapChain << std::endl;

// We keep a target format though, for instance:
TextureFormat swapChainFormat = TextureFormat::RGBA8UnormSrgb;
```
````

````{tab} Vanilla webgpu.h
```C++
// Remove all this:
std::cout << "Creating swapchain..." << std::endl;
// [...]
std::cout << "Swapchain: " << swapChain << std::endl;

// We keep a target format though, for instance:
WGPUTextureFormat swapChainFormat = WGPUTextureFormat_BGRA8Unorm;
```
````

在主渲染循环中（顺便说一下，这可能不是无头用例中的循环），我们需要自己**管理目标纹理视图**，而不是使用 `swapChain.getCurrentTextureView()` 。

````{tab} With webgpu.hpp
```C++
// Remove this:
TextureView nextTexture = swapChain.getCurrentTextureView();
```
````

````{tab} Vanilla webgpu.h
```C++
// Remove this:
WGPUTextureView nextTexture = wgpuSwapChainGetCurrentTextureView(swapChain);
```
````

### 目标纹理

我们现在创建要渲染的纹理，例如我们用来创建交换链的位置。有关纹理创建的更多详细信息，请参阅章节[第一个纹理](../basic-3d-rendering/texturing/a-first-texture.md)。

````{tab} With webgpu.hpp
```C++
// During initialization
TextureDescriptor targetTextureDesc;
targetTextureDesc.label = "Render target";
targetTextureDesc.dimension = TextureDimension::_2D;
// Any size works here, this is the equivalent of the window size
targetTextureDesc.size = { 640, 480, 1 };
// Use the same format here and in the render pipeline's color target
targetTextureDesc.format = swapChainFormat;
// No need for MIP maps
targetTextureDesc.mipLevelCount = 1;
// You may set up supersampling here
targetTextureDesc.sampleCount = 1;
// At least RenderAttachment usage is needed. Also add CopySrc to be able
// to retrieve the texture afterwards.
targetTextureDesc.usage = TextureUsage::RenderAttachment | TextureUsage::CopySrc;
targetTextureDesc.viewFormats = nullptr;
targetTextureDesc.viewFormatCount = 0;
Texture targetTexture = device.createTexture(targetTextureDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// During initialization
WGPUTextureDescriptor targetTextureDesc = {};
targetTextureDesc.nextInChain = nullptr;
targetTextureDesc.label = "Render target";
targetTextureDesc.dimension = WGPUTextureDimension_2D;
// Any size works here, this is the equivalent of the window size
targetTextureDesc.size = { 640, 480, 1 };
// Use the same format here and in the render pipeline's color target
targetTextureDesc.format = swapChainFormat;
// No need for MIP maps
targetTextureDesc.mipLevelCount = 1;
// You may set up supersampling here
targetTextureDesc.sampleCount = 1;
// At least RenderAttachment usage is needed. Also add CopySrc to be able
// to retrieve the texture afterwards.
targetTextureDesc.usage = WGPUTextureUsage_RenderAttachment | WGPUTextureUsage_CopySrc;
targetTextureDesc.viewFormats = nullptr;
targetTextureDesc.viewFormatCount = 0;
Texture targetTexture = wgpuDeviceCreateTexture(device, &targetTextureDesc);
```
````

主循环的 `nextTexture` 现在只是目标纹理的视图。

````{tab} With webgpu.hpp
```C++
// During initialization
TextureViewDescriptor targetTextureViewDesc;
targetTextureViewDesc.label = "Render texture view";
// Render to a single layer
targetTextureViewDesc.baseArrayLayer = 0;
targetTextureViewDesc.arrayLayerCount = 1;
// Render to a single mip level
targetTextureViewDesc.baseMipLevel = 0;
targetTextureViewDesc.mipLevelCount = 1;
// Render to all channels
targetTextureViewDesc.aspect = TextureAspect::All;
TextureView targetTextureView = targetTexture.createView(targetTextureViewDesc);

// In main loop, instead of using swapChain.getCurrentTextureView():
TextureView nextTexture = targetTextureView;
```
````

````{tab} Vanilla webgpu.h
```C++
// During initialization
WGPUTextureViewDescriptor targetTextureViewDesc = {};
targetTextureViewDesc.nextInChain = nullptr;
targetTextureViewDesc.label = "Render texture view";
// Render to a single layer
targetTextureViewDesc.baseArrayLayer = 0;
targetTextureViewDesc.arrayLayerCount = 1;
// Render to a single mip level
targetTextureViewDesc.baseMipLevel = 0;
targetTextureViewDesc.mipLevelCount = 1;
// Render to all channels
targetTextureViewDesc.aspect = WGPUTextureAspect_All;
WGPUTextureView targetTextureView = wgpuTextureCreateView(targetTexture, &targetTextureViewDesc);

// In main loop, instead of using swapChain.getCurrentTextureView():
WGPUTextureView nextTexture = targetTextureView;
```
````

```{note}
根据您的使用情况，您可以拥有**更高级的目标纹理管理**，例如可以交换多个纹理，以便您可以渲染下一帧，同时将前一帧保存在光盘上。
```

由于我们从一帧到另一帧重复使用相同的视图，**不要释放它**！：

````{tab} With webgpu.hpp
```C++
// Remove this:
nextTexture.release();
```
````

````{tab} Vanilla webgpu.h
```C++
// Remove this:
wgpuTextureViewRelease(nextTexture);
```
````

### 保存帧

我们不再需要*呈现*交换链：

```C++
// Remove this
swapChain.present();
```

但这意味着渲染的纹理没有做任何事情！为了看到它，我们可以使用[屏幕截图](screen-capture.md)章节中提供的`saveImage`函数。

将文件 [save_image.h](../../../data/save_image.h) 和 [stb_image_write.h](../../../data/stb_image_write.h) 保存在 `main.cpp` 旁边，**将交换链演示文稿**替换为以下内容：

```C++
saveTexture("output.png", device, targetTexture);
```

````{note}
还要在主文件的开头包含 save_image.h 文件：

```C++
#include "save_image.h"
```
````

主循环
---------

主循环包括渲染一个新帧，直到窗口关闭：

```C++
// Previously:
while (!glfwWindowShouldClose(window)) {
	glfwPollEvents();
	// [...]
}
```

这会变成什么完全取决于您的用例。在本示例中，我们将只保留括号，但实际上仅运行其内容**一次**，然后终止程序：

```C++
// Now:
{ // Mock main "loop"
	// [...]
}
```

清理
--------

如果您现在运行该程序，它应该已经创建了一个 `output.png` 文件（在 `build` 目录中）！您最终可以删除所有未使用的部分并完全摆脱 GLFW：

```C++
#include <glfw3webgpu.h>
#include <GLFW/glfw3.h>

// [...]

if (!glfwInit()) {
	std::cerr << "Could not initialize GLFW!" << std::endl;
	return 1;
}

glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
GLFWwindow* window = glfwCreateWindow(640, 480, "Learn WebGPU", NULL, NULL);
if (!window) {
	std::cerr << "Could not open window!" << std::endl;
	return 1;
}

// [...]

Surface surface = glfwGetWGPUSurface(instance, window);

// [...]

swapChain.release();
surface.release();
// [...]
glfwDestroyWindow(window);
glfwTerminate();
```

您还可以删除目录 `glfw` 和 `glfw3webgpu` 并将它们从 `CMakeLists.txt` 中删除：

```CMake
# Remove this
add_subdirectory(glfw)
# and that
add_subdirectory(glfw3webgpu)

# And remove glfw and glfw3webgpu from dependencies here:
target_link_libraries(App PRIVATE webgpu)
```


````{tab} With webgpu.hpp
*结果代码：* [`step030-headless`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-headless-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-headless-vanilla)
````
