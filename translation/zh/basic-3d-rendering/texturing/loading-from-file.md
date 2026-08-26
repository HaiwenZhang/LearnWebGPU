从文件装入<span class="bullet">🟡</span>
=================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/texturing/loading-from-file.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step075`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step075)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step075-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step075-vanilla)
````

本章的目标是:**总结一下我们所做的一切**写入工具函数`loadTexture`从文件中创建纹理。

````{tab} With webgpu.hpp
```C++
Texture loadTexture(const fs::path& path, Device device) {
	// [...]
}
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUTexture loadTexture(const fs::path& path, WGPUDevice device) {
	// [...]
}
```
````

##钢铁_图像库

再来一次**装入标准文件格式**我们最好用现有的代码 而不是自己研究规格 我们在这里使用`stb_image`单页图书库,从非常方便[标准b](https://github.com/nothings/stb)集库。 它支持所有基本图像类型(png, jpg, bmp,...).

保存[标准b_图像.h](https://raw.githubusercontent.com/nothings/stb/master/stb_image.h)在源代码树中,并包含在主文件中:

```C++
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

这提供了两项职能:`stbi_load`和`stbi_image_free`,用于:

```C++
int width, height, channels;
unsigned char *data = stbi_load(path.string().c_str(), &width, &height, &channels, 0);
// If data is null, loading failed.

// Use the width, height, channels and data variables here
// [...]

stbi_image_free(data);
// (Do not use data after this)
```

````{note}
钢铁_图像库会触发一些警告,如果想要将警告视为错误,则需要关闭。

```CMake
if (MSVC)
	# Disable warning C4244: conversion from 'int' to 'short', possible loss of data
	target_compile_options(App PUBLIC /wd4244)
endif (MSVC)
```

注意此也禁用您的文件警告 。 仅禁用 stb 警告的解决方案_图像是将文件与`STB_IMAGE_IMPLEMENTATION`在自己的CMake目标中被隔离。
````

##装入图解

有了手头的纹理尺寸,我们就可以创建纹理对象.

````{tab} With webgpu.hpp
```C++
Texture loadTexture(const fs::path& path, Device device) {
	int width, height, channels;
	unsigned char *pixelData = stbi_load(path.string().c_str(), &width, &height, &channels, 4 /* force 4 channels */);
	if (nullptr == pixelData) return nullptr;

	TextureDescriptor textureDesc;
	textureDesc.dimension = TextureDimension::_2D;
	textureDesc.format = TextureFormat::RGBA8Unorm; // by convention for bmp, png and jpg file. Be careful with other formats.
	textureDesc.mipLevelCount = 1;
	textureDesc.sampleCount = 1;
	textureDesc.size = { (unsigned int)width, (unsigned int)height, 1 };
	textureDesc.usage = TextureUsage::TextureBinding | TextureUsage::CopyDst;
	textureDesc.viewFormatCount = 0;
	textureDesc.viewFormats = nullptr;
	Texture texture = device.createTexture(textureDesc);

	// Upload data to the GPU texture (to be implemented!)
	writeMipMaps(device, texture, textureDesc.size, textureDesc.mipLevelCount, pixelData);

	stbi_image_free(pixelData);

	return texture;
}
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUTexture loadTexture(const fs::path& path, WGPUDevice device) {
	int width, height, channels;
	unsigned char *pixelData = stbi_load(path.string().c_str(), &width, &height, &channels, 4 /* force 4 channels */);
	if (nullptr == pixelData) return nullptr;

	WGPUTextureDescriptor textureDesc = {};
	textureDesc.nextInChain = nullptr;
	textureDesc.dimension = WGPUTextureDimension_2D;
	textureDesc.format = WGPUTextureFormat_RGBA8Unorm; // by convention for bmp, png and jpg file. Be careful with other formats.
	textureDesc.mipLevelCount = 1;
	textureDesc.sampleCount = 1;
	textureDesc.size = { (unsigned int)width, (unsigned int)height, 1 };
	textureDesc.usage = WGPUTextureUsage_TextureBinding | WGPUTextureUsage_CopyDst;
	textureDesc.viewFormatCount = 0;
	textureDesc.viewFormats = nullptr;
	WGPUTexture texture = wgpuDeviceCreateTexture(device, &textureDesc);

	// Upload data to the GPU texture (to be implemented!)
	writeMipMaps(device, texture, textureDesc.size, textureDesc.mipLevelCount, pixelData);

	stbi_image_free(pixelData);

	return texture;
}
```
````

我们还需要写`writeMipMaps`辅助功能。 第一个MIP级别很简单,可以直接使用`data`矢量 :

````{tab} With webgpu.hpp
```C++
// Auxiliary function for loadTexture
static void writeMipMaps(
	Device device,
	Texture texture,
	Extent3D textureSize,
	[[maybe_unused]] uint32_t mipLevelCount, // not used yet
	const unsigned char* pixelData)
{
	ImageCopyTexture destination;
	destination.texture = texture;
	destination.mipLevel = 0;
	destination.origin = { 0, 0, 0 };
	destination.aspect = TextureAspect::All;

	TextureDataLayout source;
	source.offset = 0;
	source.bytesPerRow = 4 * textureSize.width;
	source.rowsPerImage = textureSize.height;

	Queue queue = device.getQueue();
	queue.writeTexture(destination, pixelData, 4 * textureSize.width * textureSize.height, source, textureSize);
	queue.release();
}
```
````

````{tab} Vanilla webgpu.h
```C++
// Auxiliary function for loadTexture
static void writeMipMaps(
	WGPUDevice device,
	WGPUTexture texture,
	WGPUExtent3D textureSize,
	[[maybe_unused]] uint32_t mipLevelCount, // not used yet
	const unsigned char* pixelData)
{
	WGPUImageCopyTexture destination = {};
	destination.texture = texture;
	destination.mipLevel = 0;
	destination.origin = { 0, 0, 0 };
	destination.aspect = WGPUTextureAspect_All;

	WGPUTextureDataLayout source = {};
	source.offset = 0;
	source.bytesPerRow = 4 * textureSize.width;
	source.rowsPerImage = textureSize.height;

	WGPUQueue queue = wgpuDeviceGetQueue(device);
	wgpuQueueWriteTexture(queue, destination, pixelData, 4 * textureSize.width * textureSize.height, source, textureSize);
	wgpuQueueRelease(queue);
}
```
````

##纹理视图

在处理MIP地图之前,我们想测试一下`loadTexture`0级米普 但为此,我们仍然错过了一部分:**纹理视图**.

要创建采样者使用的纹理视图,我们需要**MIP 级别计数**和**格式**我们可以修改`loadTexture`或返回这种信息,或象这个例子那样,是否创建适当的纹理视图并返回。

这是制作的**选项**用指针将返回的视图传递,如果无效,则忽略。

````{tab} With webgpu.hpp
```C++
Texture loadTexture(const fs::path& path, Device device, TextureView* pTextureView = nullptr) {
	// [...]

	if (pTextureView) {
		TextureViewDescriptor textureViewDesc;
		textureViewDesc.aspect = TextureAspect::All;
		textureViewDesc.baseArrayLayer = 0;
		textureViewDesc.arrayLayerCount = 1;
		textureViewDesc.baseMipLevel = 0;
		textureViewDesc.mipLevelCount = textureDesc.mipLevelCount;
		textureViewDesc.dimension = TextureViewDimension::_2D;
		textureViewDesc.format = textureDesc.format;
		*pTextureView = texture.createView(textureViewDesc);
	}

	return texture;
}
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUTexture loadTexture(const fs::path& path, WGPUDevice device, WGPUTextureView* pTextureView = nullptr) {
	// [...]

	if (pTextureView) {
		TextureViewDescriptor textureViewDesc;
		textureViewDesc.aspect = WGPUTextureAspect_All;
		textureViewDesc.baseArrayLayer = 0;
		textureViewDesc.arrayLayerCount = 1;
		textureViewDesc.baseMipLevel = 0;
		textureViewDesc.mipLevelCount = textureDesc.mipLevelCount;
		textureViewDesc.dimension = WGPUTextureViewDimension_2D;
		textureViewDesc.format = textureDesc.format;
		*pTextureView = wgpuTextureCreateView(texture, &textureViewDesc);
	}

	return texture;
}
```
````

因此,我们可以将我们的纹理装入如下:

````{tab} With webgpu.hpp
```C++
// Create a texture
TextureView textureView = nullptr;
Texture texture = loadTexture(RESOURCE_DIR "/texture.jpg", device, &textureView);
if (!texture) {
	std::cerr << "Could not load texture!" << std::endl;
	return 1;
}
// (remove the "Create and upload data" section from the init)
```
````

````{tab} Vanilla webgpu.h
```C++
// Create a texture
WGPUTextureView textureView = nullptr;
WGPUTexture texture = loadTexture(RESOURCE_DIR "/texture.jpg", device, &textureView);
if (!texture) {
	std::cerr << "Could not load texture!" << std::endl;
	return 1;
}
// (remove the "Create and upload data" section from the init)
```
````

复制任何图像([example](../../../../images/cobblestone_floor_08_diff_2k.jpg)改为`resources/texture.jpg`你应该看到它装在3D飞机上! 您可能必须增加最大图像大小**设备限制**:

```C++
// Allow textures up to 2K
requiredLimits.limits.maxTextureDimension1D = 2048;
requiredLimits.limits.maxTextureDimension2D = 2048;
```

```{note}
这里的纹理视图指向`loadTexture`这是`&textureView`,即变量的地址`textureView`。即使变量本身被初始化为无效,**委员会的地址**,调用`pTextureView`在职能中,**不为空**,从而创建并返回视图。
```

##生成缩略图

让我们回到现在`writeMipMaps`函数。 我们可以在这里导入循环`writeTexture`在上一章中,我们已执行装入 mip 级别 :

````{tab} With webgpu.hpp
```C++
// Auxiliary function for loadTexture
static void writeMipMaps(
	Device device,
	Texture texture,
	Extent3D textureSize,
	uint32_t mipLevelCount,
	const unsigned char* pixelData)
{
	Queue queue = device.getQueue();

	// Arguments telling which part of the texture we upload to
	ImageCopyTexture destination;
	destination.texture = texture;
	destination.origin = { 0, 0, 0 };
	destination.aspect = TextureAspect::All;

	// Arguments telling how the C++ side pixel memory is laid out
	TextureDataLayout source;
	source.offset = 0;

	// Create image data
	Extent3D mipLevelSize = textureSize;
	std::vector<unsigned char> previousLevelPixels;
	Extent3D previousMipLevelSize;
	for (uint32_t level = 0; level < mipLevelCount; ++level) {
		// Pixel data for the current level
		std::vector<unsigned char> pixels(4 * mipLevelSize.width * mipLevelSize.height);
		if (level == 0) {
			// We cannot really avoid this copy since we need this
			// in previousLevelPixels at the next iteration
			memcpy(pixels.data(), pixelData, pixels.size());
		}
		else {
			// Create mip level data
			for (uint32_t i = 0; i < mipLevelSize.width; ++i) {
				for (uint32_t j = 0; j < mipLevelSize.height; ++j) {
					unsigned char* p = &pixels[4 * (j * mipLevelSize.width + i)];
					// Get the corresponding 4 pixels from the previous level
					unsigned char* p00 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 0))];
					unsigned char* p01 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 1))];
					unsigned char* p10 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 0))];
					unsigned char* p11 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 1))];
					// Average
					p[0] = (p00[0] + p01[0] + p10[0] + p11[0]) / 4;
					p[1] = (p00[1] + p01[1] + p10[1] + p11[1]) / 4;
					p[2] = (p00[2] + p01[2] + p10[2] + p11[2]) / 4;
					p[3] = (p00[3] + p01[3] + p10[3] + p11[3]) / 4;
				}
			}
		}

		// Upload data to the GPU texture
		destination.mipLevel = level;
		source.bytesPerRow = 4 * mipLevelSize.width;
		source.rowsPerImage = mipLevelSize.height;
		queue.writeTexture(destination, pixels.data(), pixels.size(), source, mipLevelSize);

		previousLevelPixels = std::move(pixels);
		previousMipLevelSize = mipLevelSize;
		mipLevelSize.width /= 2;
		mipLevelSize.height /= 2;
	}

	queue.release();
}
```
````

````{tab} Vanilla webgpu.h
```C++
// Auxiliary function for loadTexture
static void writeMipMaps(
	WGPUDevice device,
	WGPUTexture texture,
	WGPUExtent3D textureSize,
	uint32_t mipLevelCount,
	const unsigned char* pixelData)
{
	WGPUQueue queue = wgpuDeviceGetQueue(device);

	// Arguments telling which part of the texture we upload to
	WGPUImageCopyTexture destination = {};
	destination.texture = texture;
	destination.origin = { 0, 0, 0 };
	destination.aspect = WGPUTextureAspect_All;

	// Arguments telling how the C++ side pixel memory is laid out
	WGPUTextureDataLayout source = {};
	source.offset = 0;

	// Create image data
	WGPUExtent3D mipLevelSize = textureSize;
	std::vector<unsigned char> previousLevelPixels;
	WGPUExtent3D previousMipLevelSize;
	for (uint32_t level = 0; level < mipLevelCount; ++level) {
		// Pixel data for the current level
		std::vector<unsigned char> pixels(4 * mipLevelSize.width * mipLevelSize.height);
		if (level == 0) {
			// We cannot really avoid this copy since we need this
			// in previousLevelPixels at the next iteration
			memcpy(pixels.data(), pixelData, pixels.size());
		}
		else {
			// Create mip level data
			for (uint32_t i = 0; i < mipLevelSize.width; ++i) {
				for (uint32_t j = 0; j < mipLevelSize.height; ++j) {
					unsigned char* p = &pixels[4 * (j * mipLevelSize.width + i)];
					// Get the corresponding 4 pixels from the previous level
					unsigned char* p00 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 0))];
					unsigned char* p01 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 1))];
					unsigned char* p10 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 0))];
					unsigned char* p11 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 1))];
					// Average
					p[0] = (p00[0] + p01[0] + p10[0] + p11[0]) / 4;
					p[1] = (p00[1] + p01[1] + p10[1] + p11[1]) / 4;
					p[2] = (p00[2] + p01[2] + p10[2] + p11[2]) / 4;
					p[3] = (p00[3] + p01[3] + p10[3] + p11[3]) / 4;
				}
			}
		}

		// Upload data to the GPU texture
		destination.mipLevel = level;
		source.bytesPerRow = 4 * mipLevelSize.width;
		source.rowsPerImage = mipLevelSize.height;
		wgpuQueueWriteTexture(queue, destination, pixels.data(), pixels.size(), source, mipLevelSize);

		previousLevelPixels = std::move(pixels);
		previousMipLevelSize = mipLevelSize;
		mipLevelSize.width /= 2;
		mipLevelSize.height /= 2;
	}

	wgpuQueueRelease(queue);
}
```
````

最后,我们**自动推断 mip 级别计数**如前一章所述,从纹理大小:

````{tab} With webgpu.hpp
```C++
// Equivalent of std::bit_width that is available from C++20 onward
uint32_t bit_width(uint32_t m) {
	if (m == 0) return 0;
	else { uint32_t w = 0; while (m >>= 1) ++w; return w; }
}

Texture loadTexture(const fs::path& path, Device device) {
	// [...]

	textureDesc.size = { (unsigned int)width, (unsigned int)height, 1 };
	textureDesc.mipLevelCount = bit_width(std::max(textureDesc.size.width, textureDesc.size.height));

	// [...]
}
```
````

````{tab} Vanilla webgpu.h
```C++
// Equivalent of std::bit_width that is available from C++20 onward
uint32_t bit_width(uint32_t m) {
	if (m == 0) return 0;
	else { uint32_t w = 0; while (m >>= 1) ++w; return w; }
}

WGPUTexture loadTexture(const fs::path& path, WGPUDevice device) {
	// [...]

	textureDesc.size = { (unsigned int)width, (unsigned int)height, 1 };
	textureDesc.mipLevelCount = bit_width(std::max(textureDesc.size.width, textureDesc.size.height));

	// [...]
}
```
````

```{figure} /images/load-texture-from-file.jpg
:align: center
:class: with-shadow
从文件中装入的纹理 带有适当的缩写
```

##纹理模型

让我们用一个很好的纹理3D模型来完成这一章. 解析[4areen.zip (英语).](../../../../data/fourareen.zip)在资源目录中(special thanks to Scottish Maritime Museum for [sharing this model](https://sketchfab.com/3d-models/venus-a-shetland-fourareen-ce4d6915e1d041459e08f2d8da521e86)并更改两条装载线:

````{tab} With webgpu.hpp
```C++
bool success = loadGeometryFromObj(RESOURCE_DIR "/fourareen.obj", vertexData);
// [...]
Texture texture = loadTexture(RESOURCE_DIR "/fourareen2K_albedo.jpg", device, &textureView);
```
````

````{tab} Vanilla webgpu.h
```C++
bool success = loadGeometryFromObj(RESOURCE_DIR "/fourareen.obj", vertexData);
// [...]
WGPUTexture texture = loadTexture(RESOURCE_DIR "/fourareen2K_albedo.jpg", device, &textureView);
```
````

如果你尝试这个,你会看到纹理没有正确投射到几何上. 首先要确保您移除我们在前一章中添加的紫外线乘法,以探索采样器的效果:

```rust
输出.uv = in.uv; // 而不是 in.uv* 6.0
```

我们还需要再次增加顶点缓冲大小限制,因为这个网格有近150k顶点:

```C++
requiredLimits.limits.maxBufferSize = 150000 * sizeof(VertexAttributes);
```

最后有一个观点:

```C++
// Matrices
uniforms.modelMatrix = mat4x4(1.0);
uniforms.viewMatrix = glm::lookAt(vec3(-2.0f, -3.0f, 2.0f), vec3(0.0f), vec3(0, 0, 1));
uniforms.projectionMatrix = glm::perspective(45 * PI / 180, 640.0f / 480.0f, 0.01f, 100.0f);
```

```{figure} /images/fourareen.png
:align: center
:class: with-shadow
我们的 WebGPU 查看器中渲染的 3D 模型 。
```

结论
----------

即使灯光模型相当基本,我们现在可以装上和显示**有纹理的复杂三维模型**!

接下来的部分会帮助我们**组织一点我们的代码基础**因为时间太长了 无法独占主机 完成后,我们将着手进行最后部分 建造一个基本的实时 3D 渲染器,即**照明**.

````{tab} With webgpu.hpp
*结果代码 :* [`step075`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step075)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step075-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step075-vanilla)
````
