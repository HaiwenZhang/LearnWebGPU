从文件 <span class="bullet">🟡</span> 加载
=================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/3d-meshes/loading-from-file.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step058`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step058)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step058-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step058-vanilla)
````

这次，我们准备加载**实际的 3D 文件格式**，而不是我们迄今为止一直使用的临时文件格式，以便您可以使用您想要的任何模型。

在本章中，我们从 [OBJ](https://en.wikipedia.org/wiki/Wavefront_.obj_file) 格式加载 3D 模型。这是一种非常常见的格式，用于存储具有额外属性（顶点颜色、法线，还有纹理坐标和任何其他任意数据）的 3D 网格。例如，您可以通过从 [Blender](https://www.blender.org/).导出 OBJ 文件来创建 OBJ 文件。

## TinyOBJLoader

我们不使用手动解析 OBJ 文件，而是使用 [TinyOBJLoader](https://github.com/tinyobjloader/tinyobjloader) 库。文件格式**没那么复杂**，但解析文件不是本教程系列的重点，并且该库已经过深入测试并且占用空间非常小。

与[我们的`webgpu.hpp`包装器](/getting-started/cpp-idioms.md)类似，TinyOBJLoader由单个文件[`tiny_obj_loader.h`](https://raw.githubusercontent.com/tinyobjloader/tinyobjloader/release/tiny_obj_loader.h)组成，您可以简单地将其保存在`main.cpp`旁边。您的源文件之一必须在包含它之前定义 `TINYOBJLOADER_IMPLEMENTATION` ：

```C++
#define TINYOBJLOADER_IMPLEMENTATION // add this to exactly 1 of your C++ files
#include "tiny_obj_loader.h"
```

我们创建一个新的加载函数：

```C++
bool loadGeometryFromObj(const fs::path& path, std::vector<VertexAttributes>& vertexData);
```

我们可以使用[TinyOBJLoader的README中的示例代码](https://github.com/tinyobjloader/tinyobjloader#example-code-deprecated-api)作为基础，它展示了如何调用它提供的`LoadObj`函数：

```C++
bool loadGeometryFromObj(const fs::path& path, std::vector<VertexAttributes>& vertexData) {
	tinyobj::attrib_t attrib;
	std::vector<tinyobj::shape_t> shapes;
	std::vector<tinyobj::material_t> materials;

	std::string warn;
	std::string err;

	bool ret = tinyobj::LoadObj(&attrib, &shapes, &materials, &warn, &err, path.string().c_str());

	if (!warn.empty()) {
		std::cout << warn << std::endl;
	}

	if (!err.empty()) {
		std::cerr << err << std::endl;
	}

	if (!ret) {
		return false;
	}

	// Fill in vertexData here

	return true;
}
```

一旦填充了tinyobj特定的结构（`shape_t`，`attrib_t`），我们就可以提取顶点数组数据，使代码更清晰一些：

```C++
// Filling in vertexData:
const auto& shape = shapes[0]; // look at the first shape only

vertexData.resize(shape.mesh.indices.size());
for (size_t i = 0; i < shape.mesh.indices.size(); ++i) {
	const tinyobj::index_t& idx = shape.mesh.indices[i];

	vertexData[i].position = {
		attrib.vertices[3 * idx.vertex_index + 0],
		attrib.vertices[3 * idx.vertex_index + 1],
		attrib.vertices[3 * idx.vertex_index + 2]
	};

	vertexData[i].normal = {
		attrib.normals[3 * idx.normal_index + 0],
		attrib.normals[3 * idx.normal_index + 1],
		attrib.normals[3 * idx.normal_index + 2]
	};

	vertexData[i].color = {
		attrib.colors[3 * idx.vertex_index + 0],
		attrib.colors[3 * idx.vertex_index + 1],
		attrib.colors[3 * idx.vertex_index + 2]
	};
}
```

我们可以通过迭代 `shapes` 来推广到模型的所有部分：

```C++
vertexData.clear();
for (const auto& shape : shapes) {
	size_t offset = vertexData.size();
	vertexData.resize(offset + shape.mesh.indices.size());
	// [...]
	vertexData[offset + i].position = /* ... */;
	vertexData[offset + i].normal = /* ... */;
	vertexData[offset + i].color = /* ... */;
}
```

````{note}
> 🤔 为什么我们在这里停止使用**索引绘图**？

OBJ 文件格式比 GPU 光栅化管道可以处理的文件格式更灵活。通常，在 OBJ 文件中可能有一个顶点，其**位置 `p1` 由 2 个三角形共享**，但具有**不同的法线** `n1`/`n1b`：

```
# OBJ 文件内部
f 3 p1/n1 p2/n2 p3/n3
f 3 p1/n1b p4/n4 p5/n5
# ^^^ 正常索引
# ^^ 位置索引
# ^ 脸上的点数
```

但在**索引绘制调用**中，具有**相同索引**的两个顶点必须共享**其所有属性**，否则它们必须被视为两个不同的顶点。因此，无论哪种方式，我们都需要重新组织从文件加载的数据和存储在 GPU 缓冲区中的数据。
````

## 加载第一个对象

我们停止使用索引绘图，但切换到 `VertexAttributes` 向量，而不是顶点缓冲区数据的 `float` 盲向量：

````{tab} With webgpu.hpp
```C++
std::vector<VertexAttributes> vertexData;
bool success = loadGeometryFromObj(RESOURCE_DIR "/pyramid.obj", vertexData);
if (!success) {
	std::cerr << "Could not load geometry!" << std::endl;
	return 1;
}
// [...]
bufferDesc.size = vertexData.size() * sizeof(VertexAttributes);
// [...]
queue.writeBuffer(vertexBuffer, 0, vertexData.data(), bufferDesc.size);
// [...]
int indexCount = static_cast<int>(vertexData.size());
// (remove 'Create index buffer')
// [...]
renderPass.setVertexBuffer(0, vertexBuffer, 0, vertexData.size() * sizeof(VertexAttributes));
// (remove renderPass.setIndexBuffer)
// [...]
renderPass.draw(indexCount, 1, 0, 0);
// [...]
// (remove indexBuffer destruction)
```
````

````{tab} Vanilla webgpu.h
```C++
std::vector<VertexAttributes> vertexData;
bool success = loadGeometryFromObj(RESOURCE_DIR "/pyramid.obj", vertexData);
if (!success) {
	std::cerr << "Could not load geometry!" << std::endl;
	return 1;
}
// [...]
bufferDesc.size = vertexData.size() * sizeof(VertexAttributes);
// [...]
wgpuQueueWriteBuffer(queue, vertexBuffer, 0, vertexData.data(), bufferDesc.size);
// [...]
int indexCount = static_cast<int>(vertexData.size());
// (remove 'Create index buffer')
// [...]
wgpuRenderPassSetVertexBuffer(renderPass, 0, vertexBuffer, 0, vertexData.size() * sizeof(VertexAttributes));
// (remove wgpuRenderPassSetIndexBuffer)
// [...]
wgpuRenderPassDraw(renderPass, indexCount, 1, 0, 0);
// [...]
// (remove indexBuffer destruction)
```
````

我们需要的最后一件事是增加最大缓冲区大小以便能够加载各种网格：

```C++
// Update max buffer size to allow up to 10000 vertices in the loaded file:
requiredLimits.limits.maxBufferSize = 10000 * sizeof(VertexAttributes);
```

使用 [pyramid.obj](../../../../data/pyramid.obj) 进行测试，它是相同的金字塔，但具有斜角边缘（我还设置了 `T1 = mat4x4(1.0)` 和 `focalPoint = vec3(0.0, 0.0, -1.0)`）：

```{figure} /images/pyramid-obj-yup.png
:align: center
:class: with-shadow
3D 模型已正确加载，但未按预期定向。
```

````{note}
如果您在加载 OBJ 时看到有关缺少材质信息的警告，请不要担心：

```
在 .mtl 中找不到材料 ['None']
```

OBJ 文件可能有一个配套的 .mtl 文件，提供有关材料的信息，但我们在此不使用此信息。
````

## 纵轴约定

对于哪个轴应代表垂直方向，3D 建模和 3D 渲染工具之间**没有达成共识**。但文件格式通常会强加一个约定。在OBJ格式的情况下，规定**Y轴表示向上垂直方向**。

由于我们一直隐式遵循**Z 是代码中的垂直方向**的约定，因此我们需要更改约定（通过更改视图矩阵）或在加载时进行转换。我选择了后者：

```C++
vertexData[i].position = {
	attrib.vertices[3 * idx.vertex_index + 0],
	-attrib.vertices[3 * idx.vertex_index + 2], // Add a minus to avoid mirroring
	attrib.vertices[3 * idx.vertex_index + 1]
};

// Also apply the transform to normals!!
vertexData[i].normal = {
	attrib.normals[3 * idx.normal_index + 0],
	-attrib.normals[3 * idx.normal_index + 2],
	attrib.normals[3 * idx.normal_index + 1]
};
```

```{figure} /images/pyramid-obj.png
:align: center
:class: with-shadow
加载的 3D 模型，方向正确。
```

## 另一个例子

想要一些更有趣的东西吗？尝试[mammoth.obj](../../../../data/mammoth.obj)！并感谢史密森学会[分享模型](https://sketchfab.com/3d-models/mammuthus-primigenius-blumbach-229976b3db4646b39c44e57a7e3d8744)。

```{figure} /images/mammoth.png
:align: center
:class: with-shadow
复杂的 3D 模型。
```

结论
----------

本章总结了有关加载和渲染 3D 网格的部分。这是相当重要的一部分，恭喜您到目前为止！

从现在开始，我们将了解如何在此基础上进行改进，但请随意花一些时间**尝试一下**。

您现在可以加载自己的模型（如果您以不同的文件格式获取它们，则通过 Blender 来转换它们）、为不同对象的位置设置动画、更改照明（甚至使用制服为其设置动画）等。

````{tab} With webgpu.hpp
*结果代码：* [`step058`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step058)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step058-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step058-vanilla)
````

