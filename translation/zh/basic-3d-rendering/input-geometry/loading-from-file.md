从文件 <span class="bullet">🟢</span> 加载
=================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/input-geometry/loading-from-file.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/037 - Loading from file - vanilla
:parent: 034 - Index Buffer - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/037 - Loading from file
:parent: 034 - Index Buffer
```

````{tab} With webgpu.hpp
*结果代码：* [`step037`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step037)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step037-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step037-vanilla)
````

现在我们已经熟悉了 GPU 期望的几何数据的表示，我们可以从文件加载它，而不是在源代码中硬编码它。借此机会向我们的项目介绍一些基本的**资源管理**（尽管这并非特定于 WebGPU）。

文件格式
-----------

### 示例

我这里介绍的文件格式不是标准的，但解析起来很简单。这是 `webgpu.txt` 的内容，我将其放入 `resources/` 目录中：

```{lit} C++, file: resources/webgpu.txt (also for tangle root "Vanilla")
[points]
# x   y      r   g   b

0.5   0.0    0.0 0.353 0.612
1.0   0.866  0.0 0.353 0.612
0.0   0.866  0.0 0.353 0.612

0.75  0.433  0.0 0.4   0.7
1.25  0.433  0.0 0.4   0.7
1.0   0.866  0.0 0.4   0.7

1.0   0.0    0.0 0.463 0.8
1.25  0.433  0.0 0.463 0.8
0.75  0.433  0.0 0.463 0.8

1.25  0.433  0.0 0.525 0.91
1.375 0.65   0.0 0.525 0.91
1.125 0.65   0.0 0.525 0.91

1.125 0.65   0.0 0.576 1.0
1.375 0.65   0.0 0.576 1.0
1.25  0.866  0.0 0.576 1.0

[indices]
 0  1  2
 3  4  5
 6  7  8
 9 10 11
12 13 14
```

它基本上是之前直接在我们的 C++ 代码中定义的 `pointData` 和 `indexData` 的内容（在 `Application::InitializeBuffers()` 中），其中 `[section name]` 形式的一行介绍了每个部分。空行或以 `#` 开头的行将被忽略。

````{note}
我们已经可以提高最大缓冲区大小限制：

```{lit} C++, Other device limits (append, also for tangle root "Vanilla")
// We need buffers to support up to 15 points that have 5 float attributes each (x, y, r, g, b)
requiredLimits.limits.maxBufferSize = 15 * 5 * sizeof(float);
```
````

### 解析器

我不打算详细介绍解析器。我相信这很容易理解，而且**不是本次讲座的核心主题**。

```{note}
一旦我们开始使用 **3D 数据**，我们无论如何都会切换到更**标准的格式**。
```

您可以简单地将这个 `loadGeometry` 函数复制到 `main.cpp` 文件的开头（我们稍后会将其移动到专用的 `ResourceManager.cpp`）：

```C++
#include <vector>
#include <filesystem>
#include <fstream>
#include <sstream>
#include <string>

bool loadGeometry(
	const std::filesystem::path& path,
	std::vector<float>& pointData,
	std::vector<uint16_t>& indexData
) {
	std::ifstream file(path);
	if (!file.is_open()) {
		return false;
	}

	pointData.clear();
	indexData.clear();

	enum class Section {
		None,
		Points,
		Indices,
	};
	Section currentSection = Section::None;

	float value;
	uint16_t index;
	std::string line;
	while (!file.eof()) {
		getline(file, line);
		
		// overcome the `CRLF` problem
		if (!line.empty() && line.back() == '\r') {
			line.pop_back();
		}
		
		if (line == "[points]") {
			currentSection = Section::Points;
		}
		else if (line == "[indices]") {
			currentSection = Section::Indices;
		}
		else if (line[0] == '#' || line.empty()) {
			// Do nothing, this is a comment
		}
		else if (currentSection == Section::Points) {
			std::istringstream iss(line);
			// Get x, y, r, g, b
			for (int i = 0; i < 5; ++i) {
				iss >> value;
				pointData.push_back(value);
			}
		}
		else if (currentSection == Section::Indices) {
			std::istringstream iss(line);
			// Get corners #0 #1 and #2
			for (int i = 0; i < 3; ++i) {
				iss >> index;
				indexData.push_back(index);
			}
		}
	}
	return true;
}
```

从光盘加载资源
---------------------------

### 基本方法

我们可以通过调用 `Application::InitializeBuffers()` 中的新 `loadGeometry` 函数来替换 `pointData` 和 `indexData` 向量的定义。

```{lit} C++, InitializeBuffers method (replace, also for tangle root "Vanilla")
void Application::InitializeBuffers() {
	// 1. Load from disk into CPU-side vectors pointData and indexData
	{{Load geometry data from file}}

	// 2. Create GPU buffers and upload data to them
	{{Create point buffer}}
	{{Create index buffer}}
}
```

```{lit} C++, Load geometry data from file (also for tangle root "Vanilla")
// Define data vectors, but without filling them in
std::vector<float> pointData;
std::vector<uint16_t> indexData;

// Here we use the new 'loadGeometry' function:
bool success = loadGeometry("resources/webgpu.txt", pointData, indexData);

// Check for errors
if (!success) {
	std::cerr << "Could not load geometry!" << std::endl;
	exit(1);
}

// We now store the index count rather than the vertex count
indexCount = static_cast<uint32_t>(indexData.size());
```

这个硬编码相对路径的一个**问题**是它的解释取决于您运行可执行文件的目录：

```output
我的项目> ./build/App
（工作正常）
my_project> cd 构建
my_project/build> ./应用程序
无法加载几何图形！
```

在第二种情况下，您的程序尝试打开不存在的`my_project/build/resources/webgpu.txt`。有几个选项可以解决这个问题：

- **选项 A** 不在乎，只需从正确的目录调用您的程序即可。这可能很烦人，问题是 IDE 通常从 `build/` 甚至 `build/` 的子目录运行可执行文件。
- **选项 B** 使用绝对路径。这只适用于您的机器，这是一个很大的限制。
- **选项 C** 使用 CMake 自动生成的绝对路径。这就是我们要做的。
- **选项 D** 使用命令行参数告诉程序在哪里找到资源目录。这是一个有趣的选项，可以与*选项 C* 结合使用，但需要更多的工作。
- **选项 E** 自动将资源复制到 IDE 启动程序的目录中。一旦我们尝试在程序运行时修改资源（这在编写着色器时非常方便），这将是一个问题。

我建议我们选择 **选项 C** 进行开发，同时当您想要分发程序时可以轻松切换到 **选项 A**。

### 资源路径解析

我们将做这样的事情：

```C++
#define RESOURCE_DIR "/home/me/src/my_project/resources"
loadGeometry(RESOURCE_DIR "/webgpu.txt", pointData, indexData);
```

除了 `#define RESOURCE_DIR` 将由 CMake 添加，而不是显式写入我们的源代码中！

```{note}
当在 C 或 C++ 源代码中将两个“字符串文字”并排放置时（例如 `loadGeometry("resource" "/webgpu.txt"`，...），它们会自动连接。这正是为了让我们的用例发挥作用！
```

要在 `CMakeLists.txt` 中定义 `RESOURCE_DIR`，您可以在创建 `App` 目标后添加以下内容：

```{lit} CMake, Set the RESOURCE_DIR define (also for tangle root "Vanilla")
target_compile_definitions(App PRIVATE
	RESOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}/resources"
)
```

表达式 `${CMAKE_CURRENT_SOURCE_DIR}` 被 CMake 变量 [`CMAKE_CURRENT_SOURCE_DIR`](https://cmake.org/cmake/help/latest/variable/CMAKE_CURRENT_SOURCE_DIR.html) 的内容替换，该变量是一个内置变量，包含您正在编辑的 `CMakeLists.txt` 文件的父目录的完整路径。

```{note}
编写 CMake *函数* 时，`CMAKE_CURRENT_SOURCE_DIR` 变量包含当前调用该函数的 `CMakeLists.txt` 的目录。如果要引用定义该函数的`CMakeLists.txt`的目录，请使用[`CMAKE_CURRENT_LIST_DIR`](https://cmake.org/cmake/help/latest/variable/CMAKE_CURRENT_LIST_DIR.html)代替。
```

```{figure} /images/loaded-webgpu-logo-transform-issue.png
:align: center
:class: with-shadow
现在应该加载数据文件并显示类似这样的内容。
```

几乎认不出 WebGPU 标志？别担心，我们很快就会重新集中它！

### 便携性

> 😒 嘿，但最终我们的可执行文件使用了**绝对路径**，所以我们在尝试共享它时遇到了**可移植性问题**，对吧？

确实是的，但是在构建我们希望能够分发的版本时，我们可以轻松添加一个选项来全局更改资源目录：

```{lit} CMake, Set the RESOURCE_DIR define (replace, also for tangle root "Vanilla")
# We add an option to enable different settings when developing the app than
# when distributing it.
option(DEV_MODE "Set up development helper settings" ON)

if(DEV_MODE)
	# In dev mode, we load resources from the source tree, so that when we
	# dynamically edit resources (like shaders), these are correctly
	# versionned.
	target_compile_definitions(App PRIVATE
		RESOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}/resources"
	)
else()
	# In release mode, we just load resources relatively to wherever the
	# executable is launched from, so that the binary is portable
	target_compile_definitions(App PRIVATE
		RESOURCE_DIR="./resources"
	)
endif()
```

然后，您可以在两个不同的目录中拥有项目的 2 个不同版本：

```
cmake -B build-dev -DDEV_MODE=ON -DCMAKE_BUILD_TYPE=调试
cmake -B build-release -DDEV_MODE=OFF -DCMAKE_BUILD_TYPE=Release
```

第一个是为了开发的舒适性，第二个是为了发布的可移植性。

```{tip}
`CMAKE_BUILD_TYPE` 选项是 CMake 的内置选项，非常常用。将其设置为 `Debug` 以使用 **调试符号** 编译程序（请参阅 [debugging](/appendices/debugging.md)），但代价是可执行文件更慢且更重。将其设置为 `Release` 以获得**快速且轻量级**的可执行文件，无需调试保护。

当使用某些 CMake 生成器（例如 Visual Studio 生成器）时，这一点会被忽略，因为生成的解决方案可以直接在 IDE 中从 `Debug` 切换到 `Release` 模式，而不需要询问 CMake。
```

### 构建网络

> 😒 当我尝试使用 emscripten 构建时，它仍然无法工作！

事实上，网页无法访问一个人的文件系统（出于良好的安全原因），而且它甚至没有意义，因为客户端不会拥有这些文件。

在这种情况下，我们需要做的是**要求 emscripten 将文件捆绑在应用程序中**。这可以通过 [`--preload-file`](https://emscripten.org/docs/porting/files/packaging_files.html) 链接选项轻松完成，该选项将我们的内容预加载到**虚拟文件系统**中。

```{lit} CMake, Emscripten-specific options (append, also for tangle root "Vanilla")
target_link_options(App PRIVATE
	--preload-file "${CMAKE_CURRENT_SOURCE_DIR}/resources"
)
```

这种方法使我们的 C++ 代码的行为**就像磁盘上有文件一样**，而实际上它只是应用程序附带的数据的一部分。

````{note}
我们可以在 `--preload-file` 选项末尾精确指定 `@resources` ，表示该目录必须安装在虚拟文件系统中的路径 `/resources` 处。默认情况下，使用预加载文件相对于 `CMakeLists.txt` 的路径。

```CMake
target_link_options(App PRIVATE
	-sASYNCIFY
	--preload-file "${CMAKE_CURRENT_SOURCE_DIR}/resources@resources"
	#                                                    ^^^^^^^^^^ here
)
```
````

资源管理器
----------------

为了使我们的项目保持井井有条，我建议我们创建一个新的 `ResourceManager` 类，专用于我们所有的资源**输入/输出过程**，特别是我们的新 `loadGeometry`。

根据添加 C++ 类时的建议，我们添加头文件 `ResourceManager.h` 和实现文件 `ResourceManager.cpp`。我们已经可以将它们列在我们的 `CMakeLists.txt` 中：

```{lit} CMake, Define app target (replace, hidden, also for tangle root "Vanilla")
{{Dependency subdirectories}}

add_executable(App
	{{App source files}}
)

{{Link libraries}}

{{Set the RESOURCE_DIR define}}
```

```{lit} CMake, App source files (append, also for tangle root "Vanilla")
# In the list of files of add_executable(App ...):
ResourceManager.h
ResourceManager.cpp
```

### 头文件

类头文件的结构总是如下所示：

```{lit} C++, file: ResourceManager.h (also for tangle root "Vanilla")
// ResourceManager.h
#pragma once
{{ResourceManager.h includes}}

class ResourceManager {
public:
	{{Public ResourceManager members}}

private:
	{{Private ResourceManager members}}
};
```

我们可以在 `public` 成员中声明我们的 `loadGeometry` 函数：

```{lit} C++, Public ResourceManager members (also for tangle root "Vanilla")
/**
 * Load a file from `path` using our ad-hoc format and populate the `pointData`
 * and `indexData` vectors.
 */
static bool loadGeometry(
	const std::filesystem::path& path,
	std::vector<float>& pointData,
	std::vector<uint16_t>& indexData
);
```

```{note}
此方法（与本指南的其他加载方法一样）定义为 `static` ，以便 **我们不需要 `ResourceManager` 类的实例**。 `ResourceManager` 类实际上仅用作**某种命名空间**。无论如何，我将其保留为一个类，因为在更高级的场景中，您可能希望将其转换为可实例化的类。
```

请注意，我们需要添加这些内容：

```{lit} C++, ResourceManager.h includes (also for tangle root "Vanilla")
// Add to ResourceManager.h includes
#include <vector>
#include <filesystem>
```

````{note}
我们目前没有任何 `private` 成员。

```{lit} C++, Private ResourceManager members (hidden, also for tangle root "Vanilla")
```
````

### 实施文件

实现文件仅包含 `loadGeometry` 函数的定义。 **不要忘记**为其添加前缀 `ResourceManager::` 并包含 `"ResourceManager.h"`：

```C++
// In ResourceManager.cpp
#include "ResourceManager.h"

#include <fstream>
#include <sstream>
#include <string>

bool ResourceManager::loadGeometry(
	const std::filesystem::path& path,
	std::vector<float>& pointData,
	std::vector<uint16_t>& indexData
) {
	// [...] Copy the content of the `loadGeometry()` defined above.
}
```

```{lit} C++, file: ResourceManager.cpp (hidden, also for tangle root "Vanilla")
// ResourceManager.cpp
#include "ResourceManager.h"
{{Other ResourceManager.cpp includes}}

{{ResourceManager member definitions}}
```

```{lit} C++, Other ResourceManager.cpp includes (hidden, also for tangle root "Vanilla")
// Add to ResourceManager.cpp includes
#include <fstream>
#include <sstream>
#include <string>
```

```{lit} C++, ResourceManager member definitions (hidden, also for tangle root "Vanilla")
bool ResourceManager::loadGeometry(
	const std::filesystem::path& path,
	std::vector<float>& pointData,
	std::vector<uint16_t>& indexData
) {
	std::ifstream file(path);
	if (!file.is_open()) {
		return false;
	}

	pointData.clear();
	indexData.clear();

	enum class Section {
		None,
		Points,
		Indices,
	};
	Section currentSection = Section::None;

	float value;
	uint16_t index;
	std::string line;
	while (!file.eof()) {
		getline(file, line);
		
		// overcome the `CRLF` problem
		if (!line.empty() && line.back() == '\r') {
			line.pop_back();
		}
		
		if (line == "[points]") {
			currentSection = Section::Points;
		}
		else if (line == "[indices]") {
			currentSection = Section::Indices;
		}
		else if (line[0] == '#' || line.empty()) {
			// Do nothing, this is a comment
		}
		else if (currentSection == Section::Points) {
			std::istringstream iss(line);
			// Get x, y, r, g, b
			for (int i = 0; i < 5; ++i) {
				iss >> value;
				pointData.push_back(value);
			}
		}
		else if (currentSection == Section::Indices) {
			std::istringstream iss(line);
			// Get corners #0 #1 and #2
			for (int i = 0; i < 3; ++i) {
				iss >> index;
				indexData.push_back(index);
			}
		}
	}
	return true;
}
```

### 用法

要在 `main.cpp` 文件中使用这个新类，我们必须注意包含它的头文件：

```{lit} C++, Includes (append, also for tangle root "Vanilla")
// In main.cpp
#include "ResourceManager.h"
```

在调用静态方法时，我们必须在 `loadGeometry` 前面加上 `ResourceManager::` 前缀：

```C++
bool success = ResourceManager::loadGeometry(RESOURCE_DIR "/webgpu.txt", pointData, indexData);
//             ^^^^^^^^^^^^^^^^^ don't forget this
```

```{lit} C++, Load geometry data from file (hidden, replace, also for tangle root "Vanilla")
// Define data vectors, but without filling them in
std::vector<float> pointData;
std::vector<uint16_t> indexData;

// Here we use the new 'ResourceManager::loadGeometry' function:
bool success = ResourceManager::loadGeometry(RESOURCE_DIR "/webgpu.txt", pointData, indexData);

// Check for errors
if (!success) {
	std::cerr << "Could not load geometry!" << std::endl;
	exit(1);
}

// We now store the index count rather than the vertex count
indexCount = static_cast<uint32_t>(indexData.size());
```

### 加载着色器

现在我们有了基本的资源管理机制，我强烈建议我们也使用它来 **加载我们的着色器代码**，而不是像我们从一开始就将其硬编码在 C++ 源代码中。我们甚至可以将整个着色器模块创建样板包含在新的 `ResourceManager::loadShaderModule()` 中。

让我们首先在 `ResourceManager.h` 中声明它：

````{tab} With webgpu.hpp
```{lit} C++, Public ResourceManager members (append)
/**
 * Create a shader module for a given WebGPU `device` from a WGSL shader source
 * loaded from file `path`.
 */
static wgpu::ShaderModule loadShaderModule(
	const std::filesystem::path& path,
	wgpu::Device device
);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Public ResourceManager members (append, for tangle root "Vanilla")
/**
 * Create a shader module for a given WebGPU `device` from a WGSL shader source
 * loaded from file `path`.
 */
static WGPUShaderModule loadShaderModule(
	const std::filesystem::path& path,
	WGPUDevice device
);
```
````

我们一定不要忘记在 `ResourceManager.h` 中包含 webgpu 标头：

`````{tab} With webgpu.hpp
```{lit} C++, ResourceManager.h includes (append)
#include <webgpu/webgpu.hpp>
```

````{caution}
**不要**在**头文件**中调用 `using namespace wgpu` ，否则它会强制包含此头文件的所有文件接受使用 `wgpu` 命名空间，这可能会产生冲突。这就是为什么我们在标头中添加 `wgpu::` 前缀，但当然我们可以在实现（.cpp）文件中调用它：

```{lit} C++, Other ResourceManager.cpp includes (append)
using namespace wgpu;
```
````
`````

````{tab} Vanilla webgpu.h
```{lit} C++, ResourceManager.h includes (append, for tangle root "Vanilla")
#include <webgpu/webgpu.h>
```
````

我们现在可以在 `ResourceManager.cpp` 中定义它：

````{tab} With webgpu.hpp
```{lit} C++, ResourceManager member definitions (append)
ShaderModule ResourceManager::loadShaderModule(const std::filesystem::path& path, Device device) {
	std::ifstream file(path);
	if (!file.is_open()) {
		return nullptr;
	}
	file.seekg(0, std::ios::end);
	size_t size = file.tellg();
	std::string shaderSource(size, ' ');
	file.seekg(0);
	file.read(shaderSource.data(), size);

	ShaderModuleWGSLDescriptor shaderCodeDesc{};
	shaderCodeDesc.chain.next = nullptr;
	shaderCodeDesc.chain.sType = SType::ShaderModuleWGSLDescriptor;
	shaderCodeDesc.code = shaderSource.c_str();

	ShaderModuleDescriptor shaderDesc{};
#ifdef WEBGPU_BACKEND_WGPU
	shaderDesc.hintCount = 0;
	shaderDesc.hints = nullptr;
#endif
	shaderDesc.nextInChain = &shaderCodeDesc.chain;
	return device.createShaderModule(shaderDesc);
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, ResourceManager member definitions (append, for tangle root "Vanilla")
WGPUShaderModule ResourceManager::loadShaderModule(const std::filesystem::path& path, WGPUDevice device) {
	std::ifstream file(path);
	if (!file.is_open()) {
		return nullptr;
	}
	file.seekg(0, std::ios::end);
	size_t size = file.tellg();
	std::string shaderSource(size, ' ');
	file.seekg(0);
	file.read(shaderSource.data(), size);

	WGPUShaderModuleWGSLDescriptor shaderCodeDesc{};
	shaderCodeDesc.chain.next = nullptr;
	shaderCodeDesc.chain.sType = WGPUSType_ShaderModuleWGSLDescriptor;
	shaderCodeDesc.code = shaderSource.c_str();

	WGPUShaderModuleDescriptor shaderDesc{};
	shaderDesc.nextInChain = nullptr;
#ifdef WEBGPU_BACKEND_WGPU
	shaderDesc.hintCount = 0;
	shaderDesc.hints = nullptr;
#endif
	shaderDesc.nextInChain = &shaderCodeDesc.chain;
	return wgpuDeviceCreateShaderModule(device, &shaderDesc);
}
```
````

然后我们将 ShaderSource 变量的原始内容移动到 `resources/shader.wgsl` 中：

```{lit} rust, file: resources/shader.wgsl (also for tangle root "Vanilla")
// In a new file 'resources/shader.wgsl'
// Move the content of the global `shaderSource` variable (and remove that variable from main.cpp)
{{Shader source}}
```

```{lit} C++, Shader source literal (hidden, replace, also for tangle root "Vanilla")
```

我们将 `Application::InitializePipeline()` 中的模块创建步骤替换为：

````{tab} With webgpu.hpp
```{lit} C++, Create Shader Module (replace)
std::cout << "Creating shader module..." << std::endl;
ShaderModule shaderModule = ResourceManager::loadShaderModule(RESOURCE_DIR "/shader.wgsl", device);
std::cout << "Shader module: " << shaderModule << std::endl;

// Check for errors
if (shaderModule == nullptr) {
	std::cerr << "Could not load shader!" << std::endl;
	exit(1);
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create Shader Module (replace, for tangle root "Vanilla")
std::cout << "Creating shader module..." << std::endl;
WGPUShaderModule shaderModule = ResourceManager::loadShaderModule(RESOURCE_DIR "/shader.wgsl", device);
std::cout << "Shader module: " << shaderModule << std::endl;

// Check for errors
if (shaderModule == nullptr) {
	std::cerr << "Could not load shader!" << std::endl;
	exit(1);
}
```
````

这样，当您只想更改着色器时，您**不再需要重建**应用程序！

调整
-----------

<!--
### Alignment

indexData is aligned to 4 bytes, so if we use u32 its fine, if we use u16 we need some precaution. To be moved in index buffer section actually.
-->

### 变换

因此，正如我们之前注意到的，我们的形状没有很好地居中：

```{figure} /images/loaded-webgpu-logo-transform-issue.png
:align: center
:class: with-shadow
我们加载的形状有点偏离，我们应该移动它以更好地居中！
```

那么我们如何“移动”物体呢？与我们在上一章中解决比率问题的方式类似，我们可以在**顶点着色器**中通过向 `x` 和 `y` 坐标添加一些内容来完成此操作：

```{lit} rust, Vertex shader position( replace, also for tangle root "Vanilla")
let ratio = 640.0 / 480.0; // The width and height of the target surface
let offset = vec2f(-0.6875, -0.463); // The offset that we want to apply to the position
out.position = vec4f(in.position.x + offset.x, (in.position.y + offset.y) * ratio, 0.0, 1.0);
```

```{note}
在视口变换（比率）之前应用场景变换非常重要。当添加[绘制 3D 网格](/basic-3d-rendering/3d-meshes/index.md)! 所需的 3D 到 2D 投影变换时，我们将更详细地讨论这一点！
```

```{figure} /images/loaded-webgpu-logo-colorspace-issue.png
:align: center
:class: with-shadow
从文件加载的 WebGPU 徽标，颜色错误。
```

### Color issue

下面说明颜色显示问题。

你不只是挑剔，颜色确实有**问题**！与左侧面板中的徽标相比，窗口中的颜色显得更浅，甚至具有不同的色调。

```{note}
此行为取决于您的设备，因此您实际上可能会看到正确的颜色。无论如何，我建议您阅读以下内容！
```

> 🙄 嗯，也许您在编写您提供的文件时犯了一个错误。

不错的尝试，但不行。为了让您相信，让我们看一下文件前 3 行的颜色，它们对应于最大的三角形：

```
0.0 0.353 0.612
```

这些是在 $(0,1)$ 范围内表示的*红色*、*绿色*和*蓝色*值，但让我们将它们**重新映射**到整数范围$[0,255]$（每个通道 8 位），这是您的屏幕最有可能显示的内容（以及通常的图像文件格式存储的内容）：

```
0 90 156
```

现在我们可以在屏幕截图上检查大三角形的颜色：

```{figure} /images/color-pick.png
:align: center
:class: with-shadow
在我们的窗口屏幕截图中选择大三角形的颜色显示颜色为 $(0, 160, 205)$。
```

哦哦，不匹配。怎么了？我们有一个**颜色空间**问题，这意味着我们在给定的空间中表达颜色，但它们最终会被不同地解释。这种情况可能会在很多情况下发生，因此了解基础知识非常有用（尽管颜色科学通常不是一件小事）。

我们这里的问题来自`surfaceFormat`。让我们打印它：

```C++
std::cout << "Surface format: " << surfaceFormat << std::endl;
```

这给出*表面格式：24*。 “24”必须与 `webgpu.h` 中的 `WGPUTextureFormat` 枚举值进行比较。请注意，其中的值以 16 为基数表示（数字文字以 `0x` 开头），因此我们正在寻找 `24 = 1 * 16 + 8 = 0x18`。

````{note}
为了避免需要手动处理枚举值，我建议查看 [magic_enum](https://github.com/Neargye/magic_enum/blob/master/include/magic_enum.hpp).将此文件复制到源代码树后，您只需执行以下操作：

```C++
#include "magic_enum.hpp"

// [...]

std::cout << "Surface format: " << magic_enum::enum_name<WGPUTextureFormat>(surfaceFormat) << std::endl;
```

得益于先进的 C++ 模板机制，该库能够输出 *Surface 格式：WGPUTextureFormat_BGRA8UnormSrgb*！
````

```{admonition} Dawn
由于 Dawn 实现仅支持表面格式 `BGRA8Unorm`，因此在这种情况下您应该直接看到正确的颜色。
```

在我的设置中，首选格式是 `BGRA8UnormSrgb`：

- `BGRA` 部分表示颜色首先使用蓝色通道进行编码，然后是绿色通道，然后是红色和 Alpha。
- `8` 表示每个通道均使用 8 位（256 个可能的值）进行编码。
- `Unorm` 部分意味着它作为无符号标准化值公开，因此我们在 $(0,1)$ 范围内操作浮点数（实际上是定点实数，而不是浮点），即使底层表示仅使用 8 位。 `Snorm` 的范围为 $(-1,1)$，`Uint` 的范围为 $[0,255]$，等等。
- 最后， `Srgb` 部分告诉值使用 [sRGB](https://en.wikipedia.org/wiki/SRGB) 比例。

### sRGB 色彩空间

颜色空间的想法是回答以下问题：我们有 256 个可能值的预算来表示颜色通道，这 256 个**离散**值（索引 $i$）应该如何沿着光强度 $x$ 的**连续**范围分布？

```{image} /images/colorspace-light.plain.svg
:align: center
:class: only-light
```

```{image} /images/colorspace-dark.plain.svg
:align: center
:class: only-dark
```

最直观的方法是**线性**方法，包括在强度范围内定期分布 256 个指数。但我们可能在范围的某些部分需要更高的精度，而在其他部分则需要更低的精度。此外，屏幕的**物理响应**通常不是线性的！ **即使是你的眼睛**在将物理刺激转化为心理**感知**时也没有线性响应（并且这取决于周围的照明）。

```{note}
sRGB 色彩空间专为解决显示器的非线性问题而设计。在 [CRT](https://en.wikipedia.org/wiki/Cathode-ray_tube) 显示器上，这与物理设备的自发响应行为一致。现在我们已经切换到 LCD 或 OLED 显示器，因此物理设备具有不同的行为，但屏幕制造商人为地重现了 CRT 响应曲线以确保向后兼容性。
```

```{important}
sRGB 色彩空间是一种**标准**，所有常见图像文件格式（例如 PNG 和 JPG）都使用它。因此，当不进行任何颜色转换时，我们所做的一切（包括颜色拾取工具）都是 sRGB 格式的。

**但是**，WebGPU 假定片段着色器输出的颜色是线性的，因此当将表面格式设置为 `BGRA8UnormSrgb` 时，它会执行*线性到 sRGB* 转换。 **这就是导致我们的颜色错误的原因！**
```

### 伽马校正

一个简单的修复方法是强制使用非 sRGB 纹理格式：

```C++
TextureFormat surfaceFormat = TextureFormat::BGRA8Unorm;
```

但忽略目标表面的首选格式可能会导致性能问题（驱动程序需要始终转换格式）。相反，我们将在片段着色器中处理**颜色空间转换**。 rRGB 转换的一个很好的近似是 $R_{\text{线性}} = R_{\text{sRGB}}^{2.2}$：

```{lit} rust, Fragment shader body (replace, also for tangle root "Vanilla")
// We apply a gamma-correction to the color
// We need to convert our input sRGB color into linear before the target
// surface converts it back to sRGB.
let linear_color = pow(in.color, vec3f(2.2));
return vec4f(linear_color, 1.0);
```

```{figure} /images/loaded-webgpu-logo.png
:align: center
:class: with-shadow
具有伽马校正颜色的 WebGPU 徽标。
```

完美！我们解决了这个问题，我们甚至可以使用颜色选择器进行检查：

```{figure} /images/color-pick-corrected.png
:align: center
:class: with-shadow
现在，颜色选择显示了正确的值（几乎，我们的伽玛曲线是近似值）。
```

这种从线性到非线性色阶的转换（或相反）称为**伽玛校正**或**色调映射**。这里是出于相当技术性的考虑，但在 3D 渲染管道的末尾添加艺术驱动的色调映射操作是很常见的。片段着色器是执行此操作的正确位置。

```{note}
一般来说，色彩空间的特征是**色域**和**伽玛**。伽玛是离散值的非线性，色域是我们想要覆盖的强度范围（上面的垂直轴，概括为 3 种颜色）。色域通常由 3 个*原色*给出。
```

```{tip}
一般来说，关于色彩空间有很多东西值得迷失，不要试图一次性学习所有内容，但我个人觉得它很有趣！
```

结论
----------

从文件加载几何数据显然是一个简单的更改，但它实际上是引入多个问题的好方法，如果我们不注意它们，这些问题很容易成为噩梦：

- 资源路径解析
- 文件格式
- 色彩空间和更一般的数据编码
- 变换（比例、位置）

我们将不时地回顾这些内容并加以完善。我们现在准备转向一种避免着色器中硬编码值并增加很大灵活性的方法，即**uniforms**。

````{tab} With webgpu.hpp
*结果代码：* [`step037`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step037)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step037-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step037-vanilla)
````
