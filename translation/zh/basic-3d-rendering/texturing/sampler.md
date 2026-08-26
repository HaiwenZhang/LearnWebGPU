样本<span class="bullet">🟡</span>
=======

```{translation-warning} 译文可能已过时, /basic-3d-rendering/texturing/sampler.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step070`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step070)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step070-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step070-vanilla)
````

那个`textureLoad`函数,即我们在阴影器中访问纹理数据时所使用的函数,几乎如同一个纹理是一个基本的缓冲。 这个**无法受益**从GPU能够实现的插值和MIP级选择特性.

改为**样本**我们定义了另一个资源,叫做**样本**。我们在下文中看到,为什么这种获取纹理数据的适当方式对于**避免别名文物**.

样本设置
-------------

###样本创建

我将在本章第二部分详细介绍取样器的设置, 一旦一切被连接起来。 现在只要复制一下 这样我们就可以马上开始试验了


````{tab} With webgpu.hpp
```C++
// Create a texture
// [...]

// Create a sampler
SamplerDescriptor samplerDesc;
samplerDesc.addressModeU = AddressMode::ClampToEdge;
samplerDesc.addressModeV = AddressMode::ClampToEdge;
samplerDesc.addressModeW = AddressMode::ClampToEdge;
samplerDesc.magFilter = FilterMode::Linear;
samplerDesc.minFilter = FilterMode::Linear;
samplerDesc.mipmapFilter = MipmapFilterMode::Linear;
samplerDesc.lodMinClamp = 0.0f;
samplerDesc.lodMaxClamp = 1.0f;
samplerDesc.compare = CompareFunction::Undefined;
samplerDesc.maxAnisotropy = 1;
Sampler sampler = device.createSampler(samplerDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create a texture
// [...]

// Create a sampler
WGPUSamplerDescriptor samplerDesc;
samplerDesc.addressModeU = WGPUAddressMode_ClampToEdge;
samplerDesc.addressModeV = WGPUAddressMode_ClampToEdge;
samplerDesc.addressModeW = WGPUAddressMode_ClampToEdge;
samplerDesc.magFilter = WGPUFilterMode_Linear;
samplerDesc.minFilter = WGPUFilterMode_Linear;
samplerDesc.mipmapFilter = WGPUMipmapFilterMode_Linear;
samplerDesc.lodMinClamp = 0.0f;
samplerDesc.lodMaxClamp = 1.0f;
samplerDesc.compare = WGPUCompareFunction_Undefined;
samplerDesc.maxAnisotropy = 1;
WGPUSampler sampler = wgpuDeviceCreateSampler(device, samplerDesc);
```
````

我们还需要提高以下装置限制:

```C++
requiredLimits.limits.maxSamplersPerShaderStage = 1;
```

###标本装订

增加新的约束,现在应该感到相当直截了当:

````{tab} With webgpu.hpp
```C++
std::vector<BindGroupLayoutEntry> bindingLayoutEntries(3, Default);
//                                                     ^ This was a 2

// The texture sampler binding
BindGroupLayoutEntry& samplerBindingLayout = bindingLayoutEntries[2];
samplerBindingLayout.binding = 2;
samplerBindingLayout.visibility = ShaderStage::Fragment;
samplerBindingLayout.sampler.type = SamplerBindingType::Filtering;

// [...]

std::vector<BindGroupEntry> bindings(3);
//                                   ^ This was a 2

bindings[2].binding = 2;
bindings[2].sampler = sampler;
```
````

````{tab} Vanilla webgpu.h
```C++
std::vector<WGPUBindGroupLayoutEntry> bindingLayoutEntries(3, Default);
//                                                         ^ This was a 2

// The texture sampler binding
WGPUBindGroupLayoutEntry& samplerBindingLayout = bindingLayoutEntries[2];
samplerBindingLayout.binding = 2;
samplerBindingLayout.visibility = WGPUShaderStage_Fragment;
samplerBindingLayout.sampler.type = WGPUSamplerBindingType_Filtering;

// [...]

std::vector<WGPUBindGroupEntry> bindings(3);
//                                       ^ This was a 2

bindings[2].binding = 2;
bindings[2].sampler = sampler;
```
````

###样本使用量

在那个**阴影**,样本者仅使用`sampler`类型。 一旦捆绑,我们可以使用`textureSample(t, s, uv)`来样本纹理`t`在紫外线上`uv`使用取样器`s`:

```rust
@ group( 0) @ bonding(2) var textureSampler: 采样器;

// [...]

@ flagment (英语).
fn fs (英语)._主( in: VertexOutput) - > @ 位置( 0) vec4<f32> {
o 颜色=纹理Sample(梯度Texture,纹理Sampler, in.uv).rgb;
	// [...]
}
```

```{figure} /images/sampled-cube.png
:align: center
:class: with-shadow
立方体,通过过滤采样器纹理.
```

```{note}
我们已经可以看到使用采样器的效果:由于我们的纹理分辨率低,采样器**插图**邻居Texels被要求在某处采样**非整数 Texel 坐标**因此,这个**少点像素**结果。
```

演讲
----------

采样器设置的第一部分相当容易理解. 地址模式( E)`addressModeU`,...)对坐标空间的每个轴线(U, V, 和 W for 3D纹理)说明如何处理属于**超出范围** $(0,1)$.

为了说明这一点,我们回到飞机的例子并编辑**顶点阴影器**以缩放和抵消紫外线。

```rust
输出.uv = in.uv* 2.0 - 0.5;
```

```{figure} /images/clamp-to-edge.png
:align: center
:class: with-shadow
使用 ClampToEdge 模式,紫外线退出$(0,1)$区域被夹到$0$或$1$.
```

注意, 如果我们切换回原始纹理加载, 出界 texels 返回一个无效的颜色 :

```rust
/ 用上一章的方法:
让 texelCoords = vec2i( in.uv) 翻译*vec2f(纹理图));
让颜色=纹理Load(梯度Texture,texelCoords,0.rgb);
```

```{figure} /images/transformed-uv.png
:align: center
:class: with-shadow
原始纹理加载返回出界时的无效颜色 。
```

回到一个抽样的纹理,让我们现在尝试一个不同的U地址模式值:

````{tab} With webgpu.hpp
```C++
samplerDesc.addressModeU = AddressMode::Repeat;
```
````

````{tab} Vanilla webgpu.h
```C++
samplerDesc.addressModeU = WGPUAddressMode_Repeat;
```
````

```{figure} /images/repeat-u.png
:align: center
:class: with-shadow
U上设定的重复模式可以无限地重复纹理.
```

最后一个地址模式,我们可以在V轴上尝试,重复一个反射效果:

````{tab} With webgpu.hpp
```C++
samplerDesc.addressModeV = AddressMode::MirrorRepeat;
```
````

````{tab} Vanilla webgpu.h
```C++
samplerDesc.addressModeV = WGPUAddressMode_MirrorRepeat;
```
````

```{figure} /images/mirror-v.png
:align: center
:class: with-shadow
V上的重复模式用镜像重复纹理.
```

Filtering
---------

下面介绍纹理过滤。

下一个采样器设置约为**过滤**,这是**最强**样本的一部分。 有一个**两种过滤**.

###放大过滤

放大滤波包括当一个碎片取样一个落下的位置时,插入(即混合)两个相邻像素的价值.**两个圆形的Texel坐标之间**.

我们可以比较两个可能的过滤器:

````{tab} With webgpu.hpp
```C++
samplerDesc.magFilter = FilterMode::Nearest;
// versus
samplerDesc.magFilter = FilterMode::Linear;
```
````

````{tab} Vanilla webgpu.h
```C++
samplerDesc.magFilter = WGPUFilterMode_Nearest;
// versus
samplerDesc.magFilter = WGPUFilterMode_Linear;
```
````

```{image} /images/mag-filter-light.svg
:align: center
:class: only-light
```

```{image} /images/mag-filter-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>不同放大滤波器.</em></span>
</p>

那个`Nearest`模式对应**四舍五入**与最接近的整数texel坐标相对应,这大致相当于我们原始的texel加载(并非因为我们在切换坐标而不是四舍五入)。

那个`Linear`模式,**常用软件**,对应混合坐标`floor(u * width)`和`floor((u + 1) * width)`带有系数`fract(u * width)`.

###缩小过滤

####异形

其它过滤器在我们目前的设置中被关闭(因为我们只有1 mip级别),我们可以强调通过将相机移动到平面上来修改过滤地址的问题:

````{tab} With webgpu.hpp
```C++
samplerDesc.addressModeU = AddressMode::Repeat;
samplerDesc.addressModeV = AddressMode::Repeat;

// [...]

// In the main loop
float viewZ = glm::mix(0.0f, 0.25f, cos(2 * PI * uniforms.time / 4) * 0.5 + 0.5);
uniforms.viewMatrix = glm::lookAt(vec3(-0.5f, -1.5f, viewZ + 0.25f), vec3(0.0f), vec3(0, 0, 1));
queue.writeBuffer(uniformBuffer, offsetof(MyUniforms, viewMatrix), &uniforms.viewMatrix, sizeof(MyUniforms::viewMatrix));
```
````

````{tab} Vanilla webgpu.h
```C++
samplerDesc.addressModeU = WGPUAddressMode_Repeat;
samplerDesc.addressModeV = WGPUAddressMode_Repeat;

// [...]

// In the main loop
float viewZ = glm::mix(0.0f, 0.25f, cos(2 * PI * uniforms.time / 4) * 0.5 + 0.5);
uniforms.viewMatrix = glm::lookAt(vec3(-0.5f, -1.5f, viewZ + 0.25f), vec3(0.0f), vec3(0, 0, 1));
wgpuQueueWriteBuffer(queue, uniformBuffer, offsetof(MyUniforms, viewMatrix), &uniforms.viewMatrix, sizeof(MyUniforms::viewMatrix));
```
````

```rust
// 沿着每个轴线重复6 次纹理
输出.uv = in.uv* 6.0;
```

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/aliasing.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">当texels变得比像素小时,很多<strong>别名</strong>文物出现.</span></p>
	</figcaption>
</figure>

这是**惨了**,而且无论何时,对一个**显示小对象**(例如,因为它远离观众).

关键是**当屏幕像素覆盖了很多特克塞尔时**,一个天真的采样程序只取其中一种特克塞尔的颜色,而像素应该与色素配色。**平均数**它覆盖的所有特克塞尔的颜色.

```{image} /images/min-filter-light.svg
:align: center
:class: only-light
```

```{image} /images/min-filter-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>异名发生于<strong>脚印</strong>一个屏幕-空间像素覆盖多个纹理-空间的texel。</em></span>
</p>


####地图绘制

这需要**时间太长了**以测量像素足迹中所有texels的平均值(当对象有4K映射但出现在100像素区域时图)。

我们该怎么办? 我们**预算**许多可能的平均值,将其存储到额外的图像中,称为**MIP 地图**。因此,更常见的称为精密过滤**缩略图**。这需要更多的内存,但与它加速事物的速度相比,需要的并不多(两次初始纹理内存).

我们已经可以改变我们的纹理的描述 以及向取样员提供的纹理视图 来分配空间**存储额外的 mip 级别**:

````{tab} With webgpu.hpp
```C++
textureDesc.mipLevelCount = 8;

// [...]

textureViewDesc.mipLevelCount = textureDesc.mipLevelCount;

// [...]

// Also setup the sampler to use these mip levels
samplerDesc.minFilter = FilterMode::Linear;
samplerDesc.mipmapFilter = MipmapFilterMode::Linear;
samplerDesc.lodMinClamp = 0.0f;
samplerDesc.lodMaxClamp = 8.0f;
```
````

````{tab} Vanilla webgpu.h
```C++
textureDesc.mipLevelCount = 8;

// [...]

textureViewDesc.mipLevelCount = textureDesc.mipLevelCount;

// [...]

// Also setup the sampler to use these mip levels
samplerDesc.minFilter = WGPUFilterMode_Linear;
samplerDesc.mipmapFilter = WGPUMipmapFilterMode_Linear;
samplerDesc.lodMinClamp = 0.0f;
samplerDesc.lodMaxClamp = 8.0f;
```
````

在分配这些米普电位后,我们可以看到采样器只使用我们最接近平面的纹理数据. 在这之外,它样本 黑色的颜色,因为我们离开了**未初始化的额外 mip 级别**:

```{figure} /images/mip0.png
:align: center
:class: with-shadow
最接近的点从米普级0进行采样,包含我们的纹理. 其它的MIP水平则充满黑色像素.
```

````{note}
每个米普级的大小是前一个级的大小的一半,直到其中一个维度达到1,不再可分化. 以下代码片段定义了**最大米普水平数** (as specified [here](https://www.w3.org/TR/webgpu/#abstract-opdef-maximum-miplevel-count)):

```C++
// Equivalent of std::bit_width that is available from C++20 onward
uint32_t bit_width(uint32_t m) {
	if (m == 0) return 0;
	else { uint32_t w = 0; while (m >>= 1) ++w; return w; }
}

uint32_t maxMipLevelCount = bit_width(std::max(textureDesc.size.width, textureDesc.size.height));
```
````

####Mip 级数据

现在我们需要**计算数据**这些其他的米普水平。 然后每一级,我们发布一个`queue.writeTexture`调用以加载该级别的数据。

```{note}
在后期的教程中,我们将使用**计算阴影**直接在 GPU 上填写 MIP 级别, 因为这样效率更高。
```

让我们附上上传的纹理数据(我们呼叫`queue.writeTexture`在每米普级的循环中:

````{tab} With webgpu.hpp
```C++
// Create and upload texture data, one mip level at a time
ImageCopyTexture destination;
destination.texture = texture;
destination.origin = { 0, 0, 0 };
destination.aspect = TextureAspect::All;

TextureDataLayout source;
source.offset = 0;

Extent3D mipLevelSize = textureDesc.size;
for (uint32_t level = 0; level < textureDesc.mipLevelCount; ++level) {
	// Create image data for this mip level
	std::vector<uint8_t> pixels(4 * mipLevelSize.width * mipLevelSize.height);
	// [...]

	// Change this to the current level
	destination.mipLevel = level;

	// Compute from the mip level size
	source.bytesPerRow = 4 * mipLevelSize.width;
	source.rowsPerImage = mipLevelSize.height;

	queue.writeTexture(destination, pixels.data(), pixels.size(), source, mipLevelSize);

	// The size of the next mip level:
	// (see https://www.w3.org/TR/webgpu/#logical-miplevel-specific-texture-extent)
	mipLevelSize.width /= 2;
	mipLevelSize.height /= 2;
}
```
````

````{tab} Vanilla webgpu.h
```C++
// Create and upload texture data, one mip level at a time
WGPUImageCopyTexture destination;
destination.texture = texture;
destination.origin = { 0, 0, 0 };
destination.aspect = WGPUTextureAspect_All;

TextureDataLayout source;
source.offset = 0;

Extent3D mipLevelSize = textureDesc.size;
for (uint32_t level = 0; level < textureDesc.mipLevelCount; ++level) {
	// Create image data for this mip level
	std::vector<uint8_t> pixels(4 * mipLevelSize.width * mipLevelSize.height);
	// [...]

	// Change this to the current level
	destination.mipLevel = level;

	// Compute from the mip level size
	source.bytesPerRow = 4 * mipLevelSize.width;
	source.rowsPerImage = mipLevelSize.height;

	wgpuQueueWriteTexture(queue, destination, pixels.data(), pixels.size(), source, mipLevelSize);

	// The size of the next mip level:
	// (see https://www.w3.org/TR/webgpu/#logical-miplevel-specific-texture-extent)
	mipLevelSize.width /= 2;
	mipLevelSize.height /= 2;
}
```
````

如果关卡是0,`pixels`与以前一样填充。 对于额外的关卡,让我们从一些用于调试的纯色开始:

```C++
// Create image data
std::vector<uint8_t> pixels(4 * mipLevelSize.width * mipLevelSize.height);
for (uint32_t i = 0; i < mipLevelSize.width; ++i) {
	for (uint32_t j = 0; j < mipLevelSize.height; ++j) {
		uint8_t* p = &pixels[4 * (j * mipLevelSize.width + i)];
		if (level == 0) {
			// Our initial texture formula
			p[0] = (i / 16) % 2 == (j / 16) % 2 ? 255 : 0; // r
			p[1] = ((i - j) / 16) % 2 == 0 ? 255 : 0; // g
			p[2] = ((i + j) / 16) % 2 == 0 ? 255 : 0; // b
		} else {
			// Some debug value for visualizing mip levels
			p[0] = level % 2 == 0 ? 255 : 0;
			p[1] = (level / 2) % 2 == 0 ? 255 : 0;
			p[2] = (level / 4) % 2 == 0 ? 255 : 0;
		}
		p[3] = 255; // a
	}
}
```

你现在应该看看**渐变**视相机点的距离而定。 梯度的每个颜色都对应从不同的 mip 级别取样的texels.

再说一遍,**样本自动解析**样本的等级。 它这样做是基于两个相邻像素之间的紫外线坐标差异.

```{image} /images/min-pyramid-light.svg
:align: center
:class: only-light
```

```{image} /images/min-pyramid-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>那个<strong>MIP 金字塔</strong>一个纹理的通常方式 是可视化 其不同的米普水平。 每个关卡都包含上一个关卡的过滤和降级版本.</em></span>
</p>

注意,采样器实际上从多个米普级混合,以进行更连续的视觉反应. 这可以通过改变`mipmapFilter`:

````{tab} With webgpu.hpp
```C++
samplerDesc.mipmapFilter = MipmapFilterMode::Nearest; // instead of Linear
```
````

````{tab} Vanilla webgpu.h
```C++
samplerDesc.mipmapFilter = WGPUMipmapFilterMode_Nearest; // instead of Linear
```
````

```{figure} /images/mip-nearest.png
:align: center
:class: with-shadow
带有最接近 mip-map过滤模式的调试 mip 级别 。
```

我们现在可以用**实际过滤数据**: 级别中的每个特克塞尔$i$平均为4特克塞尔$i - 1$.

```C++
std::vector<uint8_t> previousLevelPixels;
for (uint32_t level = 0; level < textureDesc.mipLevelCount; ++level) {
	// [...] In the loop over pixels
	if (level == 0) {
		// [...]
	}
	else {
		// Get the corresponding 4 pixels from the previous level
		uint8_t* p00 = &previousLevelPixels[4 * ((2 * j + 0) * (2 * mipLevelSize.width) + (2 * i + 0))];
		uint8_t* p01 = &previousLevelPixels[4 * ((2 * j + 0) * (2 * mipLevelSize.width) + (2 * i + 1))];
		uint8_t* p10 = &previousLevelPixels[4 * ((2 * j + 1) * (2 * mipLevelSize.width) + (2 * i + 0))];
		uint8_t* p11 = &previousLevelPixels[4 * ((2 * j + 1) * (2 * mipLevelSize.width) + (2 * i + 1))];
		// Average
		p[0] = (p00[0] + p01[0] + p10[0] + p11[0]) / 4;
		p[1] = (p00[1] + p01[1] + p10[1] + p11[1]) / 4;
		p[2] = (p00[2] + p01[2] + p10[2] + p11[2]) / 4;
	}
	// [...]

	previousLevelPixels = std::move(pixels);
}
```

```{caution}
为了简洁起见,我假设我们是在使用一种纹理,其尺寸是:**权力为 2**这样,总有可能将大小除以2。 当情况并非如此时,必须注意边界。
```

因此,我们的取样员能够提供 纹理样品生产**更别名文物**别忘了把车开回去`mipmapFilter`改为`Linear`):

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/anti-aliasing.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">我们的棋盘纹理, 适当的样本与 mip映射。</span></p>
	</figcaption>
</figure>

在放牧角度上仍然可以看到一些别名. 这是因为MIP金字塔预算**异热带**平均,这个足迹是普通的方形,但在放牧角度上,像素的足迹是一条长长的陷阱。

这个**无异构**采样者(通过`maxAnisotropy`选项),但并非完美. 然而,我们最初的别名更能令人接受!

结论
----------

我们现在可以了**正确使用纹理**在我们的场景!

我们已经看到如何**填充的米普地图**,可以任意计算。 虽然它们常常包含平均值,**过滤是一个复杂的话题**,并可以使用其他操作。 对于MIP-映射深度缓冲器,将使用最大(和`compare`选择,我没有详细说明。 普通和粗略数据(which we'll discover in the [Lighting and Material](../lighting-and-material/index.md)由于平均值在物理上不正确,必须找到其他技术。

````{tab} With webgpu.hpp
*结果代码 :* [`step070`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step070)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step070-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step070-vanilla)
````
