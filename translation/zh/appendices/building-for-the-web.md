为网络构建<span class="bullet">🟡</span>
====================

```{translation-warning} 译文可能已过时, /appendices/building-for-the-web.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{warning}
本指南的**第一次迭代**没有**包含 Web 构建的说明，因此包含此附录。在最新的**持续更新浪潮**中，**指南的主要部分**直接支持 Web 构建，因此此附录很快就会过时并最终被删除。
```

*结果代码：* [`step095-emscripten`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-emscripten)

即使本指南重点关注本机应用程序开发，这是使用 WebGPU 的一个很好的副产品，它能够将我们的代码编译为网页。

在这种情况下，我们不再需要发行版（wgpu-native 或 Dawn），而是让编译器将对 WebGPU 符号的调用映射到对实际 JavaScript WebGPU API 的调用。

构建系统
------------

### Emscripten 工具链

将 C++ 代码构建到网页中需要一个**特定的编译器**，该编译器可以针对 **WebAssembly** 而不是本机二进制文件。典型的选择是*Emcripten*，我邀请您**遵循[安装说明](https://emscripten.org/docs/getting_started/downloads.html)**。

打开终端并激活`emsdk`（请参阅安装说明），以便**命令`emcmake`在`PATH`中可用**（您可以在Windows上使用`where emcmake`在其他操作系统上使用`which emcmake`进行检查）。

### 依赖关系

尽管我们下面使用的 `emcmake` 命令在很大程度上使我们能够无缝过渡到 Web 构建，但我们应该在运行任何内容之前稍微改变**我们的 `CMakeLists.txt` ，以便添加一些特定于 Web 的选项。

Emscripten 提供**其自己的 GLFW 版本**，因为在网页上绘图与在本机窗口上绘图有很大不同。因此，我们告诉 CMake 仅当**不**使用 Emscripten 时才包含我们自己的 `glfw` 目录：

```CMake
if (NOT EMSCRIPTEN)
	# Do not include this with emscripten, it provides its own version.
	add_subdirectory(glfw)
endif()
```

其他依赖项（`webgpu`、`glfw3webgpu` 和 `imgui`）不受影响或已在内部处理 Emscripten。

但是，为了让 Emscripten 在 **链接** 应用程序时使用自己的 GLFW，我们必须告诉它使用 `-sUSE_GLFW=3` 参数。我们还使用 `-sUSE_WEBGPU` 告诉链接器它必须**处理 WebGPU 符号**（并将它们替换为对 JavaScript API 的调用）：

```CMake
# At the end of the CMakeLists.txt
if (EMSCRIPTEN)
	# Add Emscripten-specific link options
	target_link_options(App PRIVATE
		-sUSE_GLFW=3 # Use Emscripten-provided GLFW
		-sUSE_WEBGPU # Handle WebGPU symbols
		-sASYNCIFY # Required by WebGPU-C++
	)
endif()
```

```{note}
在包装器中使用 `Instance::requestAdapter` 或 `Instance::requestDevice` 时，需要 `-sASYNCIFY` 选项。它使这些函数作为**同步**操作工作，而在 JavaScript 中它们是异步的（这需要在回调中编写我们的整个应用程序）。
```

在构建之前对 `CMakeLists.txt` 进行最后一次更改：**默认情况下** Emscripten 生成 **WebAssembly 模块**，但不是网页。为了获得默认的网页，我们必须将 `App` 目标的扩展名更改为 `.html`：

```CMake
# in the 'if (EMSCRIPTEN)' block:
# Generate a full web page rather than a simple WebAssembly module
set_target_properties(App PROPERTIES SUFFIX ".html")
```

```{note}
我们在下面看到如何自定义该网页的 HTML 部分（又名 *shell*）。
```

### 配置

我们现在准备使用 `emcmake` 配置我们的项目。这是 Emcripten 提供的命令，用于简化基于 CMake 的 WebAssembly 编译。它必须简单地用作 `cmake` 配置调用的前缀**：

```bash
# Create a build-web directory in which we configured the project to be built
# with emscripten. You may use any regular cmake command line option here.
emcmake cmake -B build-web
```

```{note}
**仅在第一次调用** CMake 时需要使用 `emcmake` 前缀。相关信息将被正确存储在 `CMakeCache.txt` 中，以便之后您可以照常使用 CMake。
```

但请注意，它不会按原样正确运行，我们需要更改 CMakeLists 中的一些内容。

### 构建

与 CMake 一样，构建项目的过程如下：

```bash
cmake --build build-web
```

### 运行

构建准备就绪后，它会创建一个 `App.html` 页面。为了规避浏览器安全规则，您**不能**直接打开它，而是运行**本地服务器**，例如使用Python：

```bash
python -m http.server -d build-web
```

您现在可以浏览到 [`http://localhost:8000/App.html`](http://localhost:8000/App.html)！请注意，目前**仅 Chromium/Google Chrome** 启用了 WebGPU 支持。

```{important}
在此阶段，项目应该成功构建，但网页**将无法正确运行**。
```

代码变更
------------

### 获取限制

我们面临的第一个错误（截至 2023 年 9 月 4 日）是 Chromium 缺失的功能：

```
已中止（TODO：wgpuAdapterGetLimits 未实现）
```

这里别无选择，我们必须硬编码一些值。我们仅对两个“最小”限制使用支持的限制。事实证明，根据 [web3dsurvey](https://web3dsurvey.com/webgpu)，将它们设置为 256 使我们能够**支持 99.95% 的用户**！

````{tab} With webgpu.hpp
```C++
SupportedLimits supportedLimits;
#ifdef __EMSCRIPTEN__
// Error in Chrome so we hardcode values:
supportedLimits.limits.minStorageBufferOffsetAlignment = 256;
supportedLimits.limits.minUniformBufferOffsetAlignment = 256;
#else
m_adapter.getLimits(&supportedLimits);
#endif
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUSupportedLimits supportedLimits;
#ifdef __EMSCRIPTEN__
// Error in Chrome so we hardcode values:
supportedLimits.limits.minStorageBufferOffsetAlignment = 256;
supportedLimits.limits.minUniformBufferOffsetAlignment = 256;
#else
wgpuAdapterGetLimits(m_adapter, &supportedLimits);
#endif
```
````

```{note}
WebAssembly 模块可能会被浏览器缓存，因此重新加载页面时请使用 Ctrl/Cmd+F5，而不仅仅是 F5。
```

### 资源

当尝试创建渲染管道时，我们面临的下一个问题发生：

```
无法在“GPUDevice”上执行“createRenderPipeline”
```

如果您注意上面的日志行，您可能会看到着色器模块被设置为空值：`<wgpu::ShaderModule 0>`。事实上，该程序**无法访问本地文件系统上的着色器**！

幸运的是，有一种方法可以告诉 Emscripten **将哪些数据**与 WebAssembly 模块一起打包。因此，我们向 CMakeLists 的 `target_link_options` 行添加一个新选项：

```CMake
target_link_options(App PRIVATE
	# [...]
	--preload-file "${CMAKE_CURRENT_SOURCE_DIR}/resources"
)
```

这使得 `resource` 目录的内容可供网页使用。

```{warning}
`resource` 目录的全部内容将由您的最终用户下载。确保**仅包含需要的内容**，这样您的网页就不会太重！您也可以单独枚举所需的文件。
```

### 最大内存

我们现在面临**内存不足**（OOM）错误：

```
已中止（无法将内存数组扩大到 16953344 字节 (OOM)）
```

正如错误消息中详细说明的那样，WebAssembly 模块默认情况下仅获得有限的内存量。我们可以增加这个默认数量，或者允许浏览器根据需要增量分配更多内存。我们在这里选择第二个选项，因为我们没有满足于特定的用例。

再次，通过额外的链接器选项解决了这个问题：

```CMake
target_link_options(App PRIVATE
	# [...]
	-sALLOW_MEMORY_GROWTH
)
```

### 主循环

现在应用程序已正确初始化，但在稍微停顿并显示第一帧后，它再次失败：

```
已中止（不支持 wgpuSwapChainPresent）
```

这实际上隐藏了 Emscripten 构建的应用程序的一个更普遍的问题：**不可能**有 **显式主循环**！

Web 应用程序**不能停止**其运行所在的浏览器，因此它不能永远循环。相反，在 JavaScript 中，通常使用 [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) 让浏览器在每一帧调用主循环的**主体**。

当使用 Emscripten 构建 C++ 代码时，后者在 [`emscripten/html5.h`](https://emscripten.org/docs/api_reference/html5.h.html) 标头中提供了一些与浏览器动画帧交互的实用函数。

我们特别使用[`emscripten_set_main_loop_arg`](https://emscripten.org/docs/api_reference/emscripten.h.html#c.emscripten_set_main_loop_arg)，它的作用类似于主循环，除了循环的**主体**由**函数指针**给出：

```C++
// Signature provided by html5.h:
void emscripten_set_main_loop_arg(
	em_arg_callback_func func,
	void *arg,
	int fps,
	int simulate_infinite_loop
)
```

类型 `em_arg_callback_func` 是一个函数指针，以 `void*` 作为参数，不返回任何内容。与原始 WebGPU 回调一样，这个 void 指针只是 `void *arg` 参数的 **盲目转发**，我们可以使用它来将任何数据传递到主体。

在我们的例子中，我们使用 `arg` 用户指针将指针传递给应用程序：

```C++
emscripten_set_main_loop_arg(
	[](void *userData) {
		// Cast the blind user data into the Application object it actually is
		Application& app = *reinterpret_cast<Application*>(userData);
		app.onFrame();
	},
	(void*)&app, // value sent to the 'userData' arg of the callback
	0, true
);
```

```{note}
`func` 参数可以作为 [C++ lambda](https://en.cppreference.com/w/cpp/language/lambda) **仅当**它不捕获任何变量时给出。这就是为什么我们需要使用`arg`。
```

最后，我们的 `main.cpp` 文件仍然相当简单：

```C++
// In main.cpp
#include "Application.h"

#ifdef __EMSCRIPTEN__
#include <emscripten/html5.h>
#endif

int main(int, char**) {
	Application app;
	app.onInit();

#ifdef __EMSCRIPTEN__

	emscripten_set_main_loop_arg(
		[](void *userData) {
			Application& app = *reinterpret_cast<Application*>(userData);
			app.onFrame();
		},
		(void*)&app,
		0, true
	);

#else // __EMSCRIPTEN__

	while (app.isRunning()) {
		app.onFrame();
	}
	app.onFinish();

#endif // __EMSCRIPTEN__

	return 0;
}
```

至于交换链的初始问题，我们可以简单地忽略 emscripten 版本中对 `wgpuSwapChainPresent` 的调用：

```C++
// In Application::onFrame()
#ifndef __EMSCRIPTEN__
	m_swapChain.present();
#endif
```

```{figure} /images/emscripten/result.jpg
:align: center
:class: with-shadow
我们的交互式应用程序最终在浏览器中运行。
```

奖励：贝壳
------------

如果您想更改 Emscripten 包装应用程序的 HTML 模板，您可以指定 **另一个链接选项** 来设置 **shell 文件**：`--shell-file`。

例如，从 Emscripten 的存储库下载 [`shell_minimal.html`](https://github.com/emscripten-core/emscripten/blob/main/src/shell_minimal.html)。

我还在下面的代码片段中添加了 `LINK_DEPENDS` 属性到 `App` 目标，以确保**每当编辑 shell 文件**时，构建系统都知道它必须重新链接应用程序（即使代码中没有任何更改）。

```CMake
# (In 'if (EMSCRIPTEN)')
set(SHELL_FILE shell_minimal.html)

target_link_options(App PRIVATE
	# [...]
	--shell-file "${CMAKE_CURRENT_SOURCE_DIR}/${SHELL_FILE}"
)

# Make sure to re-link when the shell file changes
set_property(
	TARGET App
	PROPERTY LINK_DEPENDS
	"${CMAKE_CURRENT_SOURCE_DIR}/${SHELL_FILE}"
)
```

结论
----------

您现在几乎可以移植本指南的任何步骤！ Emscripten 还有许多高级选项可供您探索，但我在这里不详细介绍它们，因为它们不是特定于 WebGPU 的。

*结果代码：* [`step095-emscripten`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-emscripten)
