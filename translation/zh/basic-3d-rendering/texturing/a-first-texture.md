第一个纹理<span class="bullet">🟡</span>
===============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/texturing/a-first-texture.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step060`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step060)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step060-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step060-vanilla)
````

纹理播放**非常重要的作用**用于2D或3D图形的渲染管道。

**理论上**,可以使用 GPU 缓冲器存储纹理数据,而阴影器将计算出在这个缓冲器内一个给定像素所属的抵消. 但是,这将是**效率**(甚至可能在某些设备上都没有支持). 纹理访问实际上由**专用固定单位**(原始内容存档于2018-06-21). Official website

这就是为什么,尽管所有类型的**资源**生活在VRAM中,**缓冲器**, **纹理**和**存储纹理**是WebGPU API(以及所有其他图形API)中不同的对象.

纹理创建
----------------

让我们回到我们为深度缓冲创造的纹理,**抄下来**在应用程序初始化中创建新纹理:

````{tab} With webgpu.hpp
```C++
TextureDescriptor textureDesc;
// [...] setup descriptor
Texture texture = device.createTexture(textureDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUTextureDescriptor textureDesc;
textureDesc.nextInChain = nullptr;
// [...] setup descriptor
WGPUTexture texture = wgpuDeviceCreateTexture(device, &textureDesc);
```
````

并立即在应用程序末尾添加纹理破坏:

````{tab} With webgpu.hpp
```C++
texture.destroy();
texture.release();
```
````

````{tab} Vanilla webgpu.h
```C++
wgpuTextureDestroy(texture);
wgpuTextureRelease(texture);
```
````

###大小

一个简单的设置是纹理的大小(我任意设定为2的功率,因为它通常有助于GPU对齐内存):

````{tab} With webgpu.hpp
```C++
textureDesc.dimension = TextureDimension::_2D;
textureDesc.size = { 256, 256, 1 };
//                             ^ ignored because it is a 2D texture
```
````

````{tab} Vanilla webgpu.h
```C++
textureDesc.dimension = WGPUTextureDimension_2D;
textureDesc.size = { 256, 256, 1 };
//                             ^ ignored because it is a 2D texture
```
````

纹理可以有$1$, $2$或$3$维度。 意思是它要么是一个1D的颜色梯度,一个3D网格的2D图像的狐狸.

>                                                                                                                                                  

纹理允许**样本**数值**持续**。例如,即使您的1D纹理只有10特克塞尔(a)**特克塞尔**是纹理的像素,您可以在**非整数坐标** $4.5 / 10$你会自动得到一个混合 特克塞尔4和5。

纹理的另一个强力特征是 有可能取样**平均数**价值超过一个Texel的邻居,多亏了**缩写**的纹理**包含多个图像**(亦称*次级资源*) (中文(简体) ). 所以,纹理大小不仅是`size`字段, 以及 MIP 映射数`mipLevelCount`.

我们会知道为什么这样有用**后来**,现在我们把 mip 值设置到最小值$1$关闭此特性 :

```C++
textureDesc.mipLevelCount = 1;
```

纹理也可以存储**每个特克塞尔不止一个颜色**,感谢**多采样**。通常用于处理反变异性([MSAA](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)) (中文(简体) ). 同样,我们不使用这个来做我们的纹理:

```C++
textureDesc.sampleCount = 1;
```

MIP水平计数和样本计数都有**对内存大小的影响**,所以不使用这些特性时,设置为$1$(最小)保存内存.

###格式

纹理格式同时表示**频道数**在纹理(R、RG、RGB、RGBA)中,**他们的顺序**(RGBA、BGRA等)和**每个频道的编码方式**,包括位数和比例。

比如说,`RGBA8Unorm`表示4个频道(RGBA),每个频道有8个比特,代表未签名的值(U)正常化(Norm),以便它们作为实际数字在区域内被操纵$(0,1)$(而不是整数在$(0,255)$(例如,第1段)。

````{tab} With webgpu.hpp
```C++
textureDesc.format = TextureFormat::RGBA8Unorm;
```
````

````{tab} Vanilla webgpu.h
```C++
textureDesc.format = WGPUTextureFormat_RGBA8Unorm;
```
````

###杂项

像缓冲器一样,纹理必须**宣布预定用途**,这样它们就可以被GPU的内存分配器放在更合适的内存部分.

为了能够**从 C++ 复制像素数据**,纹理需要`CopyDst`用途。 然后我们再用纹理**从阴影中取样**因此,它必须与`TextureBinding`用途 :

````{tab} With webgpu.hpp
```C++
textureDesc.usage = TextureUsage::TextureBinding | TextureUsage::CopyDst;
```
````

````{tab} Vanilla webgpu.h
```C++
textureDesc.usage = WGPUTextureUsage_TextureBinding | WGPUTextureUsage_CopyDst;
```
````

最后,我们将纹理视图设置初始化为 0(稍后将更多关于这一点):

```C++
textureDesc.viewFormatCount = 0;
textureDesc.viewFormats = nullptr;
```

上传纹理数据
----------------------

我们创建了纹理,在VRAM(即GPU)中分配了内存. 但这仍然是**未初始化块内存**我们必须为此设定数据。 这通常由**上传**从CPU。

###测试数据

我们的第一个纹理将是**简单的梯度**,可以在C++代码中将其定义如下:

```C++
// Create image data
std::vector<uint8_t> pixels(4 * textureDesc.size.width * textureDesc.size.height);
for (uint32_t i = 0; i < textureDesc.size.width; ++i) {
	for (uint32_t j = 0; j < textureDesc.size.height; ++j) {
		uint8_t *p = &pixels[4 * (j * textureDesc.size.width + i)];
		p[0] = (uint8_t)i; // r
		p[1] = (uint8_t)j; // g
		p[2] = 128; // b
		p[3] = 255; // a
	}
}
```

```{figure} /images/gradient-texture.png
:align: center
:class: with-shadow
我们用C++代码生成的测试纹理.
```

###写入纹理

上传此像素数据到纹理用途`Queue::writeTexture`,一旦创建纹理及其数据,就称为。 但有一个呼吁`Queue::writeTexture`有点复杂`Queue::writeBuffer`:由于写纹理有很多论据,它们被分组为子结构:

````{tab} With webgpu.hpp
```C++
// Arguments telling which part of the texture we upload to
// (together with the last argument of writeTexture)
ImageCopyTexture destination;
// [...]

// Arguments telling how the C++ side pixel memory is laid out
TextureDataLayout source;
// [...]

queue.writeTexture(destination, pixels.data(), pixels.size(), source, textureDesc.size);
```
````

````{tab} Vanilla webgpu.h
```C++
// Arguments telling which part of the texture we upload to
// (together with the last argument of writeTexture)
WGPUImageCopyTexture destination;
// [...]

// Arguments telling how the C++ side pixel memory is laid out
WGPUTextureDataLayout source;
// [...]

wgpuQueueWriteTexture(queue, destination, pixels.data(), pixels.size(), source, textureDesc.size);
```
````

###目标

那个`writeTexture`程序写入**只有一个图像**(纹理的子资源)一次. 论点`destination.mipLevel`告诉谁我们的目标, 而就我们的情况来说,我们只有**一个米普级**.

那个`destination.origin`等同为**页:1**理由`Queue::writeBuffer`与`writeSize`参数,它告诉**图像的哪一部分**得到更新,这样我们只能更新其中的一小部分。 最后,这个方面与色彩纹理无关.

````{tab} With webgpu.hpp
```C++
destination.texture = texture;
destination.mipLevel = 0;
destination.origin = { 0, 0, 0 }; // equivalent of the offset argument of Queue::writeBuffer
destination.aspect = TextureAspect::All; // only relevant for depth/Stencil textures
```
````

````{tab} Vanilla webgpu.h
```C++
destination.texture = texture;
destination.mipLevel = 0;
destination.origin = { 0, 0, 0 }; // equivalent of the offset argument of Queue::writeBuffer
destination.aspect = WGPUTextureAspect_All; // only relevant for depth/Stencil textures
```
````

###来源

那个`source`布局说明如何从缓冲器读取。 那个`offset`说明数据在CPU数据指针之后的起始位置`writeTexture`.

那个`bytesPerRow`告诉脚步,即CPU数据中连续两行之间的字节数.

还有`rowsPerImage`是一个图像的高度, 当一次上传多个图像时( 只有在上传一个图像时才可能) , 这很重要 。*纹理阵列*,我们不使用。

就我们而言,数据是毗连的,所以我们确定如下:

```C++
source.offset = 0;
source.bytesPerRow = 4 * textureDesc.size.width;
source.rowsPerImage = textureDesc.size.height;
```

```{note}
在从图像文件中装入数据时, 行可能会与圆形字节数对齐, 因此`source.bytesPerRow`会比`4 * textureDesc.size.width`。在上传非常小的图像时也需要这样做,因为有**最小值**(单位:千美元)`bytesPerRow`由API设定为256人.
```

纹理绘图
---------------

好,我们把纹理装到GPU的内存上... 但我们能用它做什么? 我们如何检查它是否正确上传?

不幸的是,没有快速的方法将纹理复制到纹理视图中`nextTexture`由互换链返回。 相反,我们**把我们的纹理绑在导轨管道上**并把它从我们的荫影。

###装订布局

这种约束接近于统一的缓冲约束. 我们首先需要把它加进束缚组**版式**。我们也稍微重组了我们的代码,以处理多个约束:

````{tab} With webgpu.hpp
```C++
// Create binding layouts

// Since we now have 2 bindings, we use a vector to store them
std::vector<BindGroupLayoutEntry> bindingLayoutEntries(2, Default);

// The uniform buffer binding that we already had
BindGroupLayoutEntry& bindingLayout = bindingLayoutEntries[0];
bindingLayout.binding = 0;
bindingLayout.visibility = ShaderStage::Vertex | ShaderStage::Fragment;
bindingLayout.buffer.type = BufferBindingType::Uniform;
bindingLayout.buffer.minBindingSize = sizeof(MyUniforms);

// The texture binding
BindGroupLayoutEntry& textureBindingLayout = bindingLayoutEntries[1];
// [...] Setup texture binding

// Create a bind group layout
BindGroupLayoutDescriptor bindGroupLayoutDesc{};
bindGroupLayoutDesc.entryCount = (uint32_t)bindingLayoutEntries.size();
bindGroupLayoutDesc.entries = bindingLayoutEntries.data();
BindGroupLayout bindGroupLayout = device.createBindGroupLayout(bindGroupLayoutDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create binding layouts

// Since we now have 2 bindings, we use a vector to store them
std::vector<WGPUBindGroupLayoutEntry> bindingLayoutEntries(2);

// The uniform buffer binding that we already had
WGPUBindGroupLayoutEntry& bindingLayout = bindingLayoutEntries[0];
setDefaults(bindingLayout);
bindingLayout.binding = 0;
bindingLayout.visibility = WGPUShaderStage_Vertex | WGPUShaderStage_Fragment;
bindingLayout.buffer.type = WGPUBufferBindingType_Uniform;
bindingLayout.buffer.minBindingSize = sizeof(MyUniforms);

// The texture binding
WGPUBindGroupLayoutEntry& textureBindingLayout = bindingLayoutEntries[1];
setDefaults(textureBindingLayout);
// [...] Setup texture binding

// Create a bind group layout
WGPUBindGroupLayoutDescriptor bindGroupLayoutDesc{};
bindGroupLayoutDesc.nextInChain = nullptr;
bindGroupLayoutDesc.entryCount = (uint32_t)bindingLayoutEntries.size();
bindGroupLayoutDesc.entries = bindingLayoutEntries.data();
WGPUBindGroupLayout bindGroupLayout = wgpuDeviceCreateBindGroupLayout(device, bindGroupLayoutDesc);
```
````

我们现在可以特别设置纹理绑定布局:

````{tab} With webgpu.hpp
```C++
// Setup texture binding
textureBindingLayout.binding = 1;
textureBindingLayout.visibility = ShaderStage::Fragment;
textureBindingLayout.texture.sampleType = TextureSampleType::Float;
textureBindingLayout.texture.viewDimension = TextureViewDimension::_2D;
```
````

````{tab} Vanilla webgpu.h
```C++
// Setup texture binding
textureBindingLayout.binding = 1;
textureBindingLayout.visibility = WGPUShaderStage_Fragment;
textureBindingLayout.texture.sampleType = WGPUTextureSampleType_Float;
textureBindingLayout.texture.viewDimension = WGPUTextureViewDimension_2D;
```
````

可见度设定为**仅碎片阴影**,我们不会在顶点遮蔽处取样。

纹理**样本类型**告诉在采样纹理时在阴影码中返回哪个变量类型。 因为我们的纹理使用一种普通格式(`RGBAUnorm`),其值以浮点表示。

###约束

一旦管道布局和纹理本身都形成,我们可以改变绑定组,以添加纹理绑定(并再次略作重组):

````{tab} With webgpu.hpp
```C++
// Create a binding
std::vector<BindGroupEntry> bindings(2);

bindings[0].binding = 0;
bindings[0].buffer = uniformBuffer;
bindings[0].offset = 0;
bindings[0].size = sizeof(MyUniforms);

bindings[1].binding = 1;
bindings[1].textureView = ???;

BindGroupDescriptor bindGroupDesc;
bindGroupDesc.layout = bindGroupLayout;
bindGroupDesc.entryCount = (uint32_t)bindings.size();
bindGroupDesc.entries = bindings.data();
BindGroup bindGroup = device.createBindGroup(bindGroupDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
// Create a binding
std::vector<WGPUBindGroupEntry> bindings(2);

bindings[0].binding = 0;
bindings[0].buffer = uniformBuffer;
bindings[0].offset = 0;
bindings[0].size = sizeof(MyUniforms);

bindings[1].binding = 1;
bindings[1].textureView = ???;

WGPUBindGroupDescriptor bindGroupDesc{};
bindGroupDesc.nextInChain = nullptr;
bindGroupDesc.layout = bindGroupLayout;
bindGroupDesc.entryCount = (uint32_t)bindings.size();
bindGroupDesc.entries = bindings.data();
WGPUBindGroup bindGroup = wgpuDeviceCreateBindGroup(device, bindGroupDesc);
```
````

你们可以看到,我们不允许直接将纹理传递给约束者,我们更需要建立一个**纹理视图**。在纹理本身创建后立即创建:

````{tab} With webgpu.hpp
```C++
TextureViewDescriptor textureViewDesc;
textureViewDesc.aspect = TextureAspect::All;
textureViewDesc.baseArrayLayer = 0;
textureViewDesc.arrayLayerCount = 1;
textureViewDesc.baseMipLevel = 0;
textureViewDesc.mipLevelCount = 1;
textureViewDesc.dimension = TextureViewDimension::_2D;
textureViewDesc.format = textureDesc.format;
TextureView textureView = texture.createView(textureViewDesc);
```
````

````{tab} Vanilla webgpu.h
```C++
TextureViewDescriptor textureViewDesc;
textureViewDesc.aspect = WGPUTextureAspect_All;
textureViewDesc.baseArrayLayer = 0;
textureViewDesc.arrayLayerCount = 1;
textureViewDesc.baseMipLevel = 0;
textureViewDesc.mipLevelCount = 1;
textureViewDesc.dimension = WGPUTextureViewDimension_2D;
textureViewDesc.format = textureDesc.format;
TextureView textureView = texture.createView(textureViewDesc);
```
````

###阴影

我们现在可以宣布一个全球性的变量类型`texture_2d<f32>`我们的荫影, 绑在约束组0的1。 职能`textureLoad`返回给定坐标的原始 Texel 值,并返回`@builtin(position)`框架阴影器的输入是像素屏幕坐标 :

```rust
@ group( 0) @ 绑定(1) var 梯度Texture:纹理_第2款d项<f32>;

fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
让颜色=纹理Load(梯度Texture,vec2i(in.position.xy),0.rgb;
	// ...
}
```

从阴影中的纹理取样需要设定新的设备限制,每次添加纹理时都要增加:

```C++
// Add the possibility to sample a texture in a shader
requiredLimits.limits.maxSampledTexturesPerShaderStage = 1;
```

为了更好地看到结果,我们通过绘制覆盖整个屏幕的平面来确保每个像素都有碎片. 装入文件[飞机。](../../../../data/plane.obj)并设置矩阵到 :

```C++
bool success = loadGeometryFromObj(RESOURCE_DIR "/plane.obj", vertexData);

// [...]

uniforms.modelMatrix = mat4x4(1.0);
uniforms.viewMatrix = glm::scale(mat4x4(1.0), vec3(1.0f));
uniforms.projectionMatrix = glm::ortho(-1, 1, -1, 1, -1, 1);
```

```{figure} /images/gradient-texture-window.png
:align: center
:class: with-shadow
我们的第一个纹理通过绘制一个全屏幕四角。
```

随意使用其它公式创建纹理数据:

```C++
// Create image data
std::vector<uint8_t> pixels(4 * textureDesc.size.width * textureDesc.size.height);
for (uint32_t i = 0; i < textureDesc.size.width; ++i) {
	for (uint32_t j = 0; j < textureDesc.size.height; ++j) {
		uint8_t *p = &pixels[4 * (j * textureDesc.size.width + i)];
		p[0] = (i / 16) % 2 == (j / 16) % 2 ? 255 : 0; // r
		p[1] = ((i - j) / 16) % 2 == 0 ? 255 : 0; // g
		p[2] = ((i + j) / 16) % 2 == 0 ? 255 : 0; // b
		p[3] = 255; // a
	}
}
```

```{figure} /images/other-texture-window.png
:align: center
:class: with-shadow
更改像素阵列确实会改变显示的图像 。
```

结论
----------

我们看到了**如何创建**纹理和**如何获取**它的像素从阴影。 在接下来的一章中,我们将看到**如何将纹理映射到 3D 网格**我们将认识到,我们错过了一个非常重要的成分:要充分受益于一种纹理的力量,就必须通过一种**样本**.

````{tab} With webgpu.hpp
*结果代码 :* [`step060`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step060)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step060-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step060-vanilla)
````
