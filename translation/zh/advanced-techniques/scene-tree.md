场景树 (<span class="bullet">🔴</span>TODO)
==========

```{translation-warning} 译文可能已过时, /advanced-techniques/scene-tree.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step100-gltf`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-gltf)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step100-gltf-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-gltf-vanilla)
````

多个对象
----------------

在继续讨论整个场景树之前，让我们开始组织代码来绘制**多个网格**。

待办事项

正在加载 GLTF
------------

用于表示由多个（可能是动画的）对象的层次结构组成的 3D 场景的一种非常常见的格式是 [GLTF](https://github.com/KhronosGroup/glTF).在本章中，我们将了解如何将 GLTF 文件加载到示例应用程序中。

我们使用流行的 [TinyGLTF](https://github.com/syoyo/tinygltf) 库：它是仅标头的（因此易于集成），维护良好，并且依赖于我们已经用于纹理加载的依赖项：`stb_image.h`。

```{note}
本章基于`step100`。
```

下载[`tiny_gltf.h`](https://github.com/syoyo/tinygltf/blob/release/tiny_gltf.h)、[`json.hpp`](https://github.com/syoyo/tinygltf/blob/release/json.hpp)和[`stb_image_write.h`](https://github.com/syoyo/tinygltf/blob/release/stb_image_write.h)（您应该已经有`stb_image.h`）。

与我们一直在使用的所有小型库类似，将以下内容添加到 `implementations.cpp` （或任何其他 cpp 文件，只要它**仅在其中一个**中）。

```C++
// In implementations.cpp
// TinyGLTF must be **before** stb_image and stb_image_write
#define TINYGLTF_IMPLEMENTATION
#include "tiny_gltf.h"

#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
```

为了熟悉这个新的加载库，让我们假设它只加载单个网格，就像我们的 OBJ 加载函数一样。在 `ResourceManager` 类中创建一个新的静态方法：

```C++
class ResourceManager {
	// [...]
	// Load an 3D mesh from a standard .gltf file into a vertex data buffer
	static bool loadGeometryFromGltf(const path& path, std::vector<VertexAttributes>& vertexData);
};
```

我们首先简单地检查我们是否正确加载文件：

```C++
#include "tiny_gltf.h"

bool ResourceManager::loadGeometryFromGltf(const path& path, std::vector<VertexAttributes>& vertexData) {
	using namespace tinygltf;

	Model model;
	TinyGLTF loader;
	std::string err;
	std::string warn;

	bool success = false;
	if (path.extension() == ".glb") {
		success = loader.LoadBinaryFromFile(&model, &err, &warn, path.string());
	}
	else {
		success = loader.LoadASCIIFromFile(&model, &err, &warn, path.string());
	}

	if (!warn.empty()) {
		std::cout << "Warning: " << warn << std::endl;
	}

	if (!err.empty()) {
		std::cerr << "Error: " << err << std::endl;
	}

	return success;
}
```

作为示例，我们将使用典型的科幻头盔：

<div class="sketchfab-embed-wrapper"> <iframe title="Battle Damaged Sci-fi Helmet - PBR" frameborder="0" allowfullscreen mozallowfullscreen="true" webkitallowfullscreen="true" allow="autoplay; fullscreen; xr-spatial-tracking" xr-spatial-tracking execution-while-out-of-viewport execution-while-not-rendered web-share src="https://sketchfab.com/models/b81008d513954189a063ff901f7abfe4/embed?dnt=1"> </iframe> </div>

将 [DamagedHelmet.glb](https://github.com/KhronosGroup/glTF-Sample-Models/raw/main/2.0/DamagedHelmet/glTF-Binary/DamagedHelmet.glb) 下载到您的资源目录中并尝试将其加载到应用程序中：

```C++
bool Application::initGeometry() {
	// [...]
	bool success = ResourceManager::loadGeometryFromGltf(RESOURCE_DIR "/DamagedHelmet.glb", vertexData);
	// [...]
}
```

```{note}
GLTF 文件可以具有 `.gltf` 或 `.glb` 扩展名。后者有一个“b”代表**二进制**，并且更紧凑，但人类可读性较差。它还嵌入了所有依赖项，而 .gltf
```

上传场景数据
--------------------

现在我们已经以 CPU 上的本机表示形式加载了 GLTF 数据，我们必须组织它以适合我们的渲染管道，并将我们需要的所有内容上传到 GPU。

为了**避免不需要的副本**和数据处理，我们尝试让 GLTF 场景驱动我们的渲染过程。然而，有几点是由我们的着色器的需求驱动的：

- 我们支持和期望的**输入顶点属性**列表。

初始化 CPU 模型数据的 GPU 对应部分包括：

1.上传资源（缓冲区和图像），假设它们都是需要的（否则我们最终可以添加一个清理GLTF数据的步骤）。为纹理创建 mipmap（在 GLTF 术语中称为“图像”）。
2. 创建采样器（关于 GLTF 中映射概念的一个映射）
3.为每种材质创建材质绑定组
4.为每个网格节点创建一个节点绑定组
5. 存储所有网格图元的顶点缓冲区索引

在制品大纲：
1. 重构几何体加载，以便我们有一个 Scene 和 GpuScene 对象，用于加载和绘制。
2. 将其切换为 GLFW。

```{note}
顶点属性绑定的存在必须取决于着色器使用的内容，但布局本身取决于 GLTF 数据。
```

### 调试渲染器

我们从一个简单的调试渲染器开始，它为场景树中的每个节点绘制一帧。

```C++
/**
 * A renderer that draws debug frame axes for each node of a GLTF scene.
 * 
 * You must call create() before draw(), and destroy() at the end of the
 * program. If create() is called twice, the previously created data gets
 * destroyed.
 */
class GltfDebugRenderer {
public:
	// Create from a CPU-side tinygltf model
	void create(wgpu::Device device, const tinygltf::Model& model);

	// Draw all nodes that use a given renderPipeline
	void draw(wgpu::RenderPassEncoder renderPass);

	// Destroy and release all resources
	void destroy();
};
```

让我们从我们想要画的东西开始。正如我们所说，我们希望场景树中的每个节点一帧：

```C++
// We do as if we had all the variable we need, like 'm_nodeCount',
// 'm_vertexCount', 'm_vertexBuffer', 'm_vertexBufferByteSize' and 'm_pipeline'
void GltfDebugRenderer::draw(wgpu::RenderPassEncoder renderPass) {
	// Activate our pipeline dedicated to drawing frame axes
	renderPass.setPipeline(m_pipeline);
	// Activate the vertex buffer holding frame axes data
	renderPass.setVertexBuffer(0, m_vertexBuffer, 0, m_vertexBufferByteSize);

	// Iterate over all nodes of the scene tree
	for (uint32_t i = 0 ; i < m_nodeCount ; ++i) {
		// Draw a frame, i.e., a mesh of size 'm_vertexCount'
		renderPass.draw(m_vertexCount, 1, 0, 0);
	}
}
```

```{note}
我们并不是故意在这里使用实例。确实，像我们在这里那样在每个节点上绘制完全相同的几何体，不使用实例是一种浪费，但我们将使用这个简化的示例作为在每个节点上绘制不同网格的基础。之后我将展示如何在此调试渲染器中切换到实例化。
```

现在我们知道我们要寻找什么，让我们定义所需的属性并在 `create` 方法中初始化它们：

```C++
// In GltfDebugRenderer.h
class GltfDebugRenderer {
	// [...]
private:
	wgpu::Device m_device = nullptr;
	wgpu::RenderPipeline m_pipeline = nullptr;
	wgpu::Buffer m_vertexBuffer = nullptr;
	uint64_t m_vertexBufferByteSize = 0;
	uint32_t m_nodeCount = 0;
	uint32_t m_vertexCount = 0;
};
```

请注意，我们保留对设备的引用，以便以各种方法使用它。当它不为空时，它也被用来告诉渲染器已经被初始化。

```C++
// In GltfDebugRenderer.cpp
void GltfDebugRenderer::create(wgpu::Device device, const tinygltf::Model& model) {
	if (m_device != nullptr) destroy();
	m_device = device;
	m_device.reference(); // increase reference counter to make sure the device remains valid
	// [...] Initialize things here
}

void GltfDebugRenderer::destroy() {
	// [...] Destroy things here
	m_device.release(); // decrease reference counter
	m_device = nullptr;
}
```

然后我们为渲染器的各个元素创建初始化/终止方法：

```C++
// In GltfDebugRenderer.h
class GltfDebugRenderer {
	// [...]
private:
	void initVertexBuffer();
	void terminateVertexBuffer();

	void initNodeData(const tinygltf::Model& model);
	void terminateNodeData();

	void initPipeline();
	void terminatePipeline();
	// [...]
};
```

````{tab} With webgpu.hpp
*结果代码：* [`step100-gltf`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-gltf)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step100-gltf-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-gltf-vanilla)
````
