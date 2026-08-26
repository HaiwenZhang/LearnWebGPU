高动态范围纹理 (<span class="bullet">🟠</span>WIP)
===========================

```{translation-warning} 译文可能已过时, /advanced-techniques/hdr-textures.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step120`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step120)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step120-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step120-vanilla)
````

*（注意：这曾经放置在[基于图像的照明](../basic-3d-rendering/lighting-and-material/ibl.md)之后）*

高动态范围
------------------

加载 [`autumn_park_4k.exr`](../../../data/autumn_park_4k.exr):

```C++
if (!initTexture(RESOURCE_DIR "/autumn_park_4k.exr")) return false;
```

由于我们用来加载 PNG 的库 stb_image 不识别 EXR 文件，因此我们使用 [tinyexr](https://github.com/syoyo/tinyexr)，它以类似的方式集成到我们的其他依赖项中。

下载 [`tinyexr.h`](https://github.com/syoyo/tinyexr/raw/02310c77e5156c36fedf6cf810c4071e3f83906f/tinyexr.h)、[`miniz.c`](https://raw.githubusercontent.com/syoyo/tinyexr/02310c77e5156c36fedf6cf810c4071e3f83906f/deps/miniz/miniz.c) 和 [`miniz.h`](https://raw.githubusercontent.com/syoyo/tinyexr/02310c77e5156c36fedf6cf810c4071e3f83906f/deps/miniz/miniz.h) 到源代码树中，并将以下内容添加到 `implementations.cpp`：

```C++
#define TINYEXR_IMPLEMENTATION
#include "tinyexr.h"
```

还将 `miniz.c` 和（可选）`miniz.h`/`tinyexr.h` 添加到 CMakeLists：

```CMake
add_executable(App
	# [...]
	miniz.h
	miniz.c
	tinyexr.h
)
```

我们还需要对 CMakeLists 进行更多调整：

```CMake
# Same as adding #define NOMINMAX in all files
target_compile_definitions(App PRIVATE NOMINMAX)

if(MSVC)
	# /wd4706 and /wd4127 are required by tinyexr/miniz
	target_compile_options(App PRIVATE /wd4201 /wd4706 /wd4127)
endif(MSVC)
```

在资源管理器中，我们创建一个名为 `loadExrTexture` 的 `loadTexture` 副本，其中我们使用 TinyEXR 而不是 stb_image 并将数据加载为浮点而不是 8 位整数，这会影响 mipmap 的创建：

```C++
Texture ResourceManager::loadExrTexture(const path& path, Device device, TextureView* pTextureView) {
	int width, height;
	float* pixelData; // width * height * RGBA
	const char* err = nullptr;

	int ret = LoadEXR(&pixelData, &width, &height, path.string().c_str(), &err);
	if (ret != TINYEXR_SUCCESS) {
		if (err) {
			std::cerr << "Could not load EXR file '" << path << "': " << err << std::endl;
			FreeEXRErrorMessage(err); // release memory of error message.
		}
		return nullptr;
	}

	TextureDescriptor textureDesc;
	textureDesc.dimension = TextureDimension::_2D;
	textureDesc.format = TextureFormat::RGBA16Float;
	textureDesc.size = { (unsigned int)width, (unsigned int)height, 1 };
	textureDesc.mipLevelCount = bit_width(std::max(textureDesc.size.width, textureDesc.size.height));
	textureDesc.sampleCount = 1;
	textureDesc.usage = TextureUsage::TextureBinding | TextureUsage::CopyDst;
	textureDesc.viewFormatCount = 0;
	textureDesc.viewFormats = nullptr;
	Texture texture = device.createTexture(textureDesc);

	// Convert to 16-bit floats because it is enough for a HDR
	// and 32-bit would require to enable a particular device feature to be filterable
	// (see https://www.w3.org/TR/webgpu/#texture-format-caps)
	std::vector<float16_t> halfPixels(4 * width * height);
	for (int i = 0; i < halfPixels.size(); ++i) {
		halfPixels[i] = pixelData[i];
	}
	free(pixelData);

	writeMipMaps(device, texture, textureDesc.size, textureDesc.mipLevelCount, halfPixels.data());

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

请注意，为了避免重复 mip-map 创建部分，我隔离了这个模板化实用函数：

```C++
template<typename component_t>
static void writeMipMaps(
	Device device,
	Texture texture,
	Extent3D textureSize,
	uint32_t mipLevelCount,
	const component_t* pixelData
) {
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
	std::vector<component_t> previousLevelPixels;
	Extent3D previousMipLevelSize;
	for (uint32_t level = 0; level < mipLevelCount; ++level) {
		std::vector<component_t> pixels(4 * mipLevelSize.width * mipLevelSize.height);
		if (level == 0) {
			// We cannot really avoid this copy since we need this
			// in previousLevelPixels at the next iteration
			memcpy(pixels.data(), pixelData, pixels.size() * sizeof(component_t));
		}
		else {
			// Create mip level data
			for (uint32_t i = 0; i < mipLevelSize.width; ++i) {
				for (uint32_t j = 0; j < mipLevelSize.height; ++j) {
					component_t* p = &pixels[4 * (j * mipLevelSize.width + i)];
					// Get the corresponding 4 pixels from the previous level
					component_t* p00 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 0))];
					component_t* p01 = &previousLevelPixels[4 * ((2 * j + 0) * previousMipLevelSize.width + (2 * i + 1))];
					component_t* p10 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 0))];
					component_t* p11 = &previousLevelPixels[4 * ((2 * j + 1) * previousMipLevelSize.width + (2 * i + 1))];
					// Average
					p[0] = (p00[0] + p01[0] + p10[0] + p11[0]) / (component_t)4;
					p[1] = (p00[1] + p01[1] + p10[1] + p11[1]) / (component_t)4;
					p[2] = (p00[2] + p01[2] + p10[2] + p11[2]) / (component_t)4;
					p[3] = (p00[3] + p01[3] + p10[3] + p11[3]) / (component_t)4;
				}
			}
		}

		// Upload data to the GPU texture
		destination.mipLevel = level;
		source.bytesPerRow = 4 * mipLevelSize.width * sizeof(component_t);
		source.rowsPerImage = mipLevelSize.height;
		queue.writeTexture(destination, pixels.data(), pixels.size() * sizeof(component_t), source, mipLevelSize);

		previousLevelPixels = std::move(pixels);
		previousMipLevelSize = mipLevelSize;
		mipLevelSize.width /= 2;
		mipLevelSize.height /= 2;
	}
}
```

现在要做的就是根据文件扩展名调用 `initTexture` 中的一个或另一个加载函数：

```C++
bool Application::initTexture(const std::filesystem::path &path) {
	// Create a texture
	TextureView textureView = nullptr;
	Texture texture =
		path.extension() == ".exr"
		? ResourceManager::loadExrTexture(path, m_device, &textureView)
		: ResourceManager::loadTexture(path, m_device, &textureView);
	// [...]
}
```

```{important}
[纹理格式功能表](https://www.w3.org/TR/webgpu/#texture-format-caps)显示，为了允许过滤float32纹理，我们需要在创建设备时启用`float32-filterable` **功能**。
```

我们使用 [`float16_t.hpp`](../../../data/float16_t.hpp) 因为 C++ 没有现成的 16 位浮点类型。

结论
----------

````{tab} With webgpu.hpp
*结果代码：* [`step120`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step120)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step120-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step120-vanilla)
````
