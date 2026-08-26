正在重构<span class="bullet">🟡</span>
===========

```{translation-warning} 译文可能已过时, /basic-3d-rendering/some-interaction/refactoring.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step080`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step080)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step080-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step080-vanilla)
````

本章的目标是:**添加一些交互**我们的观众。 从WebGPU的角度来看,我们知道我们需要的一切。 例如,使用户能够使用鼠标旋转对象仅涉及**更新视图矩阵**.

然而,这也是一个机会,**组织一点我们的代码基础**我们一直没讨论到现在 它不是主要话题,代码的大小仍然可以管理,因为大部分是主要功能,但这绝不适用于更大的应用.

应用程序结构
------------------------

###主要类

我会**避免过度工程设计**由于每个使用大小写都不同,所以您会根据需要定制细节。 尽管如此,它总是从一个拥有全球应用状态的类(或struct)开始.

我们在两个新文件中执行这个`Application.h`和`Application.cpp`,该应用程序的行为分布于各处**事件处理器**。要澄清逻辑,首先处理“上”,比如`onFrame`。我们已经可以想到三个事件: 输入、帧和完成。

```C++
// In Application.h
#pragma once
#include <webgpu/webgpu.hpp>

class Application {
public:
	// A function called only once at the beginning. Returns false is init failed.
	bool onInit();

	// A function called at each frame, guaranteed never to be called before `onInit`.
	void onFrame();

	// A function called only once at the very end.
	void onFinish();

private:
	// Everything that is initialized in `onInit` and needed in `onFrame`.
	wgpu::Instance m_instance = nullptr;
	wgpu::Surface m_surface = nullptr;
	// [...]
};
```

```{note}
我给私人属性名称前缀`m_`在读取代码时将它们与本地变量更好地区分开。 这是一种常见的做法(有些人还喜欢只使用`_foo`,或全摄像头`mFoo`).
```

````{important}
在使用 C++ 包装时,必须初始化 WebGPU 控件为无效(例如,`m_instance = nullptr`)因为它们没有默认的构造器。 否则你会经历这样的错误:

```
错误: “ 应用程序 : 应用程序( 无效) ” : 试图引用已删除的函数
```
````

在实际实施这些方法之前,我们已经可以尝试起草如何在我们的`main`函数 :

```C++
// In main.cpp
#include "Application.h"

int main(int, char**) {
	Application app;
	if (!app.onInit()) return 1;

	while (app.isRunning()) {
		app.onFrame();
	}

	app.onFinish();
	return 0;
}
```

因此,我们看到,我们还需要增加一个`isRunning`方法,即简单的调用`glfwWindowShouldClose`在头罩后面,但不将GLFW窗口暴露于"客户"代码(即主功能).

```C++
// In Application.h
#include <GLFW/glfw3.h>

class Application {
public:
	// A function that tells if the application is still running.
	bool isRunning();
	// [...]

private:
	GLFWwindow* m_window = nullptr;
	// [...]
};
```

```C++
// In Application.cpp
#include "Application.h"

bool Application::onInit() {
	// [...]
}

void Application::onFrame() {
	// [...]
}

void Application::onFinish() {
	// [...]
}

bool Application::isRunning() {
	return !glfwWindowShouldClose(m_window);
}
```

不要忘记将这些文件(最重要的是.cpp)添加到列表中的源文件`CMakeLists.txt`:

```CMake
add_executable(App
	Application.h  # optional, just to show it in your IDE
	Application.cpp
	main.cpp
)
```

###初始化步骤

我们的代码的一部分是**最单质**目前为止是初始化步骤。 因此,我们把它分成多种(私人)方法:

```C++
bool Application::onInit() {
	if (!initWindowAndDevice()) return false;
	if (!initSwapChain()) return false;
	if (!initDepthBuffer()) return false;
	if (!initRenderPipeline()) return false;
	if (!initTexture()) return false;
	if (!initGeometry()) return false;
	if (!initUniforms()) return false;
	if (!initBindGroup()) return false;
	return true;
}
```

每一个`initSomething`脚步与一个`terminateSomething`以反向顺序调用:

```C++
void Application::onFinish() {
	terminateBindGroup();
	terminateUniforms();
	terminateGeometry();
	terminateTexture();
	terminateRenderPipeline();
	terminateDepthBuffer();
	terminateSwapChain();
	terminateWindowAndDevice();
}
```

这样,就更容易追踪申请结束时必须公布的内容。 您也可以在声明属性时逐个分组`Application.h`.

我让你**自己移动最大的代码块**,否则本章将看起来像一个大列表。 大部分旧的`main.cpp`最终会进入`Application.cpp`; 您可以检查它与[参考代码](https://github.com/eliemichel/LearnWebGPU-Code/tree/step080)从本章当然。

下一节介绍**其他设计选择**我在重构密码的过程中做的 你可能跟不上他们

设计选择
--------------

###资源管理员

我们装入外部资源的三个程序可以移动到一个单独的命名空间或类,只有静态成员.

我们创建一个`ResourceManager.h`和`ResourceManager.cpp`文件,将其添加到CMakeLists中,并在那里移动资源加载器。

```C++
// In ResourceManager.h
#pragma once
// [...] Includes

class ResourceManager {
public:
	// (Just aliases to make notations lighter)
	using path = std::filesystem::path;
	using vec3 = glm::vec3;
	using vec2 = glm::vec2;

	/**
	 * A structure that describes the data layout in the vertex buffer,
	 * used by loadGeometryFromObj and used it in `sizeof` and `offsetof`
	 * when uploading data to the GPU.
	 */
	struct VertexAttributes {
		vec3 position;
		vec3 normal;
		vec3 color;
		vec2 uv;
	};

	// Load a shader from a WGSL file into a new shader module
	static wgpu::ShaderModule loadShaderModule(const path& path, wgpu::Device device);

	// Load an 3D mesh from a standard .obj file into a vertex data buffer
	static bool loadGeometryFromObj(const path& path, std::vector<VertexAttributes>& vertexData);

	// Load an image from a standard image file into a new texture object
	// NB: The texture must be destroyed after use
	static wgpu::Texture loadTexture(const path& path, wgpu::Device device, wgpu::TextureView* pTextureView = nullptr);
};
```

###图书馆执行情况

当我们开始将图书馆纳入多个文件时,我们必须记住:`#define FOO_IMPLEMENTATION`其中一些要求**必须只出现在一个**C++文件,且内容必须放在任何可能递归包含的内容之前.

为了避免出乎意料的复杂情况,我建议建立一个文件`implementations.cpp`成为这里,只有这里:

```C++
// In implementations.cpp
#define TINYOBJLOADER_IMPLEMENTATION
#include "tiny_obj_loader.h"

#define WEBGPU_CPP_IMPLEMENTATION
#include <webgpu/webgpu.hpp>

#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

###回调手柄

```{important}
本节为**特定 WebGPU 的 C++ 包装器**我提供。 在使用原始的CPI时,回调不能成为lambda函数,相反必须在全球范围下定义.
```

为了防止未抓取的错误回调过早解禁,我们需要存储返回的控点`device.setUncapturedErrorCallback`。到目前为止,这只是通过定义变量`h`用于整个主要功能:

```C++
// Until now, we just stored the handle in a variable local to the main function
auto h = device.setUncapturedErrorCallback([](ErrorType type, char const* message) {
	std::cout << "Device error: type " << type;
	if (message) std::cout << " (message: " << message << ")";
	std::cout << std::endl;
});
```

由于我们不再直接在主要职能中界定这一点,而是在`Application`在它,我们必须储存`h`作为类成员处理 :

```C++
// In Application.h
class Application {
private:
	// Keep the error callback alive
	std::unique_ptr<wgpu::ErrorCallback> m_errorCallbackHandle;
}

// In Application.cpp, in onInit()
m_errorCallbackHandle = device.setUncapturedErrorCallback([](ErrorType type, char const* message) {
	std::cout << "Device error: type " << type;
	if (message) std::cout << " (message: " << message << ")";
	std::cout << std::endl;
});
```

这样,只有在应用程序对象被破坏时才会释放回调.

结论
----------

我们现在有了一个更成熟的代码基础,随着我们在接下来的章节中引入新的特征,这将更容易扩展!

记住,每当你添加的东西 在一个`init`步骤,您应该在匹配时释放它`terminate`方法。

````{tab} With webgpu.hpp
*结果代码 :* [`step080`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step080)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step080-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step080-vanilla)
````
