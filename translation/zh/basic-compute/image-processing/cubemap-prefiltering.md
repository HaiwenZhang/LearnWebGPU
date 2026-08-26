立方体图预过滤(<span class="bullet">🟠</span>WIP)
====================

```{translation-warning} 译文可能已过时, /basic-compute/image-processing/cubemap-prefiltering.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step222`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step222)

问题
-------

在那个[物理材料](../../basic-3d-rendering/lighting-and-material/pbr.md)章节,我们装入**预过滤**立方体地图, 其中**MIP级别**对应于光度预集**不同粗糙度**而不是一个简单的平均值。

我们在本节中看到如何产生这些MIP水平。

多个中期计划层次
-------------------

开始

```C++
// In Application.h
// We generate views for each MIP level, and for each cubemap face
std::vector<std::array<wgpu::TextureView, 6>> m_cubemapTextureLayers;
// Views that represent a complete cube at a given MIP level
std::vector<wgpu::TextureView> m_cubemapTextureMips;
// Remove m_cubemapTextureView

struct Settings {
	// [...]
	int mipLevel = 0;
};
```

```C++
// In initTexturesEquirectToCubemap()
textureDesc.mipLevelCount = wgpuMaxMipLevels2D(textureDesc.size);
m_cubemapTexture = m_device.createTexture(textureDesc);
```

```C++
// In initTextureViews()
uint32_t levelCount = m_cubemapTexture.getMipLevelCount();
m_cubemapTextureLayers.resize(levelCount, {nullptr, nullptr, nullptr, nullptr, nullptr, nullptr});
for (uint32_t level = 0; level < levelCount; ++level) {
	textureViewDesc.baseMipLevel = level;
	for (uint32_t i = 0; i < 6; ++i) {
		textureViewDesc.label = outputLabels[i];
		textureViewDesc.baseArrayLayer = i;
		m_cubemapTextureLayers[level][i] = m_cubemapTexture.createView(textureViewDesc);
	}
}

textureViewDesc.baseArrayLayer = 0;
textureViewDesc.arrayLayerCount = 6;
textureViewDesc.dimension = TextureViewDimension::Cube;
textureViewDesc.dimension = m_settings.mode == Mode::EquirectToCubemap ? TextureViewDimension::_2DArray : TextureViewDimension::Cube;
m_cubemapTextureMips.resize(levelCount, nullptr);
for (uint32_t level = 0; level < levelCount; ++level) {
	textureViewDesc.baseMipLevel = level;
	m_cubemapTextureMips[level] = m_cubemapTexture.createView(textureViewDesc);
}

// In initBindGroup
entries[1].textureView = m_cubemapTextureMips[0];

// At the end of initTextures()
m_settings.mipLevel = glm::clamp(m_settings.mipLevel, 0, (int)m_cubemapTexture.getMipLevelCount() - 1);
```

```C++
// In onGui()
const auto& mipViews = m_cubemapTextureLayers[m_settings.mipLevel];

view = (ImTextureID)mipViews[(int)CubeFace::NegativeY];
drawList->AddImage(view, { of.x + 0 * s, of.y + s }, { of.x + 1 * s, of.y + 2 * s }, { 0, 0 }, {1, 1});

// [...]
ImGui::SliderInt("MIP Level", &m_settings.mipLevel, 0, m_cubemapTexture.getMipLevelCount() - 1);
```

开始

多个调度
-------------------

开始

每个 MIP 级别创建一个绑定组 。 捆绑组#0是我们以前拥有的,其他的则用于计算不同级别的MIP级别0的预过滤版本.

```C++
// In Application.h
std::vector<wgpu::BindGroup> m_bindGroups;
// Remove wgpu::BindGroup m_bindGroup

// In Application.cpp
void Application::initBindGroups() {
	uint32_t levelCount = m_cubemapTexture.getMipLevelCount();
	m_bindGroups.resize(m_settings.mode == Mode::EquirectToCubemap ? levelCount : 1, nullptr);

	// [...]

	m_bindGroups[0] = m_device.createBindGroup(bindGroupDesc);

	// Create bind groups for other MIP levels
	if (m_settings.mode == Mode::EquirectToCubemap) {
		// MIP level 0
		entries[0].binding = 1; // inputCubemapTexture
		entries[0].textureView = m_cubemapTextureMips[0];

		// MIP level `level`
		entries[1].binding = 3; // outputCubemapTexture

		bindGroupDesc.layout = m_prefilteringBindGroupLayout;

		for (uint32_t level = 1; level < levelCount; ++level) {
			entries[1].textureView = m_cubemapTextureMips[level];
			m_bindGroups[level] = m_device.createBindGroup(bindGroupDesc);
		}
	}
}

void Application::terminateBindGroups() {
	wgpuBindGroupRelease(m_bindGroup);
	for (BindGroup bg : m_prefilteringBindGroup) {
		wgpuBindGroupRelease(bg);
	}
}
```

我们需要定义`m_prefilteringBindGroupLayout`:

```C++
// In Application.h
wgpu::BindGroupLayout m_prefilteringBindGroupLayout = nullptr;

// At the end of Application::initBindGroupLayouts()
if (m_settings.mode == Mode::EquirectToCubemap) {
	// CubeMap as input
	bindings[0].binding = 1;
	bindings[0].texture.sampleType = TextureSampleType::Float;
	bindings[0].texture.viewDimension = TextureViewDimension::Cube;
	bindings[0].visibility = ShaderStage::Compute;

	// CubeMap as output
	bindings[1].binding = 3;
	bindings[1].storageTexture.access = StorageTextureAccess::WriteOnly;
	bindings[1].storageTexture.format = TextureFormat::RGBA8Unorm;
	bindings[1].storageTexture.viewDimension = TextureViewDimension::_2DArray;
	bindings[1].visibility = ShaderStage::Compute;

	m_prefilteringBindGroupLayout = m_device.createBindGroupLayout(bindGroupLayoutDesc);
}
```

内`onCompute`在派出第一个工作组之后:

```C++
// Prefiltering dispatches
computePass.setPipeline(m_prefilteringPipeline);
if (m_settings.mode == Mode::EquirectToCubemap) {
	uint32_t levelCount = m_cubemapTexture.getMipLevelCount();
	for (uint32_t level = 1; level < levelCount; ++level) {
		computePass.setBindGroup(0, m_bindGroups[level], 0, nullptr);

		invocationCountX = invocationCountX / 2;
		invocationCountY = invocationCountY / 2;
		workgroupCountX = (invocationCountX + workgroupSizePerDim - 1) / workgroupSizePerDim;
		workgroupCountY = (invocationCountY + workgroupSizePerDim - 1) / workgroupSizePerDim;
		computePass.dispatchWorkgroups(workgroupCountX, workgroupCountY, 1);
	}
}
```

我们还需要确定这一点`m_prefilteringPipeline`。它使用相同的阴影,但有一个不同的切入点。

```C++
// In Application.h
wgpu::ComputePipeline m_prefilteringPipeline = nullptr;

// In initComputePipelines()
switch (m_settings.mode) {
case Mode::EquirectToCubemap:
	computePipelineDesc.compute.entryPoint = "computeCubeMap";
	m_pipeline = m_device.createComputePipeline(computePipelineDesc);
	computePipelineDesc.compute.entryPoint = "prefilterCubeMap";
	m_prefilteringPipeline = m_device.createComputePipeline(computePipelineDesc);
	break;
case Mode::CubemapToEquirect:
	computePipelineDesc.compute.entryPoint = "computeEquirectangular";
	m_pipeline = m_device.createComputePipeline(computePipelineDesc);
	m_prefilteringPipeline = nullptr;
	break;
default:
	assert(false);
}
```

开始

结论
----------

*结果代码 :* [`step222`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step222)
