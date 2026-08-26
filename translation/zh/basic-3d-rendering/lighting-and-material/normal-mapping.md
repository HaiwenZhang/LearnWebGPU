正常绘图<span class="bullet">🟡</span>
==============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/lighting-and-material/normal-mapping.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step110`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step110)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step110-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step110-vanilla)
````

我们已经看到,**常规**表面(即与其垂直的方向)在光线反弹时起着关键作用。**正常绘图**根据输入地图(文本文件)到**模拟极小几何细节的效果**.

```{note}
这些**微细节**理论上可以用将网格分化很多并且略微移动一些顶点来表示,但对于普通地图模型的尺度来说,这个方法真的不可拉动.
```

```{image} /images/normal-mapping/equiv-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-mapping/equiv-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>扰动原网格的正常是代表微几何的方法,不需要精细网格.</em></span>
</p>

普通地图
----------

提供扰动的正常情况有不同的方式:

- 指定一个扰动的正常**每个顶点**并插进脸部 这是用来**滑动**相貌之间的界限 从而创造平滑的表面 我们已经有了这个

- 提供**纹理**这使得扰动的正常。 这更重,但可以带来更多的细节。 这个纹理叫做**普通地图**,这就是本章的内容!

在普通地图上**每个顶点的颜色代表一个小矢量**1. 红、绿和蓝的数值解释为X、Y和Z和**范围中的编码值$(-1, 1)$**.

这些X、Y和Z轴的意义取决于**公约**; 重要的是在您的文件和阴影码之间保持一致 :

```{image} /images/normal-mapping/normal-map-light.png
:align: center
:class: only-light
```

```{image} /images/normal-mapping/normal-map-dark.png
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>a 每个像素的颜色<strong>普通地图</strong>编码普通矢量。 这个编码有两个常规,叫做"DirectX"和"OpenGL",因为它们与这些API过去如何要求格式有关.</em></span>
</p>

```{note}
惯例"DirectX"和"OpenGL"已不再是来自图形API的限制,因为我们会用阴影书写自己如何解释普通的地图像素. 这些名字来源于我们无法写出阴影。

本文件其余部分,**我们将遵循 OpenGL 公约。**
```

在实践中,为了使用这样的普通地图,我们需要:

 1. **装入**文件作为纹理并绑在阴影上。
 2. **样本**解码为普通矢量
 3. **变形**这对面部方向来说很正常

在飞机上
----------

让我们首先集中关注步骤1。**装入**页:1**样本**使用简单的平面(因此不需要第3步)。**变形**(现在)

###正在装入普通地图

将装入的对象切换到[`plane.obj`](../../../../data/plane.obj)网点 :

```C++
bool success = ResourceManager::loadGeometryFromObj(RESOURCE_DIR "/plane.obj", m_vertexData);
```

举个例子`cobblestone_floor_08`来自[波利哈文语Name](https://polyhaven.com/a/cobblestone_floor_08)(一种可以自由使用的材料的好来源). 我们只看[扩散](../../../../images/cobblestone_floor_08_diff_2k.jpg)和[常规](../../../../images/cobblestone_floor_08_nor_gl_2k.png)现在的地图,并把它们装入纹理。

由于我们有一个以上的纹理,我们增加了属性`Application`类 :

````{tab} With webgpu.hpp
```C++
// In Application.h, replace:
wgpu::Texture m_texture = nullptr;
wgpu::TextureView m_textureView = nullptr;
// with:
wgpu::Texture m_baseColorTexture = nullptr;
wgpu::TextureView m_baseColorTextureView = nullptr;
wgpu::Texture m_normalTexture = nullptr;
wgpu::TextureView m_normalTextureView = nullptr;
```
````

````{tab} Vanilla webgpu.h
```C++
// In Application.h, replace:
WGPUTexture m_texture = nullptr;
WGPUTextureView m_textureView = nullptr;
// with:
WGPUTexture m_baseColorTexture = nullptr;
WGPUTextureView m_baseColorTextureView = nullptr;
WGPUTexture m_normalTexture = nullptr;
WGPUTextureView m_normalTextureView = nullptr;
```
````

我们现在可以更新`initTexture()`(我重命名为)`initTextures()`因此,委员会也注意到,在《公约》第2条第1款(a)项(a)项中,《公约》第2条第1款(a)项(a)项和(b)项(c)项(c)项(c)项(c)目(c)项(c)目(c)项(c)目(c)目(c)目(c)项(c)目(c)目(c)目(c)目(c)项)和(c)项(c)目(c)项(c)目(c)目(c)项)目(c)目(c)目(c)目(c)目(c)目(c)目(c)目(c)目(c)目(c)项)。`terminateTextures()`:

```C++
m_baseColorTexture = ResourceManager::loadTexture(RESOURCE_DIR "/cobblestone_floor_08_diff_2k.jpg", m_device, &m_baseColorTextureView);
m_normalTexture = ResourceManager::loadTexture(RESOURCE_DIR "/cobblestone_floor_08_nor_gl_2k.png", m_device, &m_normalTextureView);
// [...] Check errors
```

我们把这个新的纹理添加到我们的绑定中, 无论是C++一侧还是阴影面(我选择在底色1旁边插入):

```rust
@ group( 0) @ 绑定( 0) var<uniform>uMyUniforms: MyUniforms; 互联网档案馆的存檔,存档日期2013-12-20.
@ group( 0) @ 绑定(1) var base ColorTure: 纹理_第2款d项<f32>;
@ group( 0) @ bonding(2) var 正常图案:纹理_第2款d项<f32>;
//  新装订!
@group( 0) @ bonding(3) var textureSampler:采样器;
// ^ 这是 2
@ group( 0) @ bonding(4) var<uniform>uLighting:灯光统一;
// ^ 这是3
```

更新`initBindGroup()`和`initBindGroupLayout()`相应.

````{tab} With webgpu.hpp
```C++
bool Application::initBindGroupLayout() {
	std::vector<BindGroupLayoutEntry> bindingLayoutEntries(5, Default);
	//                                                     ^ This was a 4
	// [...]
	// The normal map binding
	BindGroupLayoutEntry& normalTextureBindingLayout = bindingLayoutEntries[2];
	normalTextureBindingLayout.binding = 2;
	normalTextureBindingLayout.visibility = ShaderStage::Fragment;
	normalTextureBindingLayout.texture.sampleType = TextureSampleType::Float;
	normalTextureBindingLayout.texture.viewDimension = TextureViewDimension::_2D;
	// [...] Don't forget to offset bindings after the normal map!
}

bool Application::initBindGroup() {
	std::vector<BindGroupEntry> bindings(5);
	//                                   ^ This was a 4
	// [...]
	bindings[1].binding = 1;
	bindings[1].textureView = m_baseColorTextureView;

	bindings[2].binding = 2;
	bindings[2].textureView = m_normalTextureView;
	// [...] Don't forget to offset bindings after the normal map!
}
```
````

````{tab} Vanilla webgpu.h
```C++
bool Application::initBindGroupLayout() {
	std::vector<WGPUBindGroupLayoutEntry> bindingLayoutEntries(5, Default);
	//                                                         ^ This was a 4
	// [...]
	// The normal map binding
	WGPUBindGroupLayoutEntry& normalTextureBindingLayout = bindingLayoutEntries[2];
	normalTextureBindingLayout.binding = 2;
	normalTextureBindingLayout.visibility = WGPUShaderStage_Fragment;
	normalTextureBindingLayout.texture.sampleType = WGPUTextureSampleType_Float;
	normalTextureBindingLayout.texture.viewDimension = WGPUTextureViewDimension_2D;
	// [...] Don't forget to offset bindings after the normal map!
}

bool Application::initBindGroup() {
	std::vector<WGPUBindGroupEntry> bindings(5);
	//                                   ^ This was a 4
	// [...]
	bindings[1].binding = 1;
	bindings[1].textureView = m_baseColorTextureView;

	bindings[2].binding = 2;
	bindings[2].textureView = m_normalTextureView;
	// [...] Don't forget to offset bindings after the normal map!
}
```
````

最后,这一新纹理需要更新设备限制:

```C++
requiredLimits.limits.maxSampledTexturesPerShaderStage = 2;
//                                                       ^ This was 1
```

增加光谱硬度, 并播放视图点, 这样您就可以突出此纹理的问题 :

```{figure} /images/no-normal-mapping.png
:align: center
:class: with-shadow
没有正常的映射,卵石纹理感觉完全平坦,这对这种材料来说是奇特的.
```

###取样

正常绘图干预的关键时刻是评估正常时`N`输入**碎片阴影器**.

```rust
使 N = 正常化(非正常);
```

我们不使用正常输入,而是从正常地图中取样:

```rust
// 样本正常
(c) 让编码N = 纹理Sample(正常Texture,纹理Sampler, in.uv).rgb;
让 N = 正常化(编码N - 0.5);
```

每个频道$r$, $g$和$b$,样本值位于范围$(0,1)$,我们需要将其映射到范围$(-x,x)$改为**解码**这个 价值$x$从那时起就不再重要了。

```{note}
如果你正在使用**直接X**公约,你还需要**翻转**联合国$Y$值(在解码后乘以 -1)。
```

```{figure} /images/with-normal-mapping.png
:align: center
:class: with-shadow
随着正常的测绘,卵石材料看起来有更现实的解脱.
```

正常映射为**一个强大的诡计**当然,它并不完全等同于精炼几何。 特别是,**无法更改轮廓**因此,在放牧角度上的幻觉失败:

```{figure} /images/with-normal-mapping-grazing-angle.png
:align: center
:class: with-shadow
时**放牧角度**,正常的绘图不足以给材料带来解脱的感觉. 比较复杂的方法**救灾制图**需要在这里。
```

````{note}
如果要淡化普通地图的影响,可以将其与原普通地图混合:

```rust
让普通MapStrength = 0.5;//可以是制服
让 N = 常态(混合(正常,编码N - 0.5,普通MapStrength));
```
````

在网点上
---------

###问题

普通地图中所包含的方向**相对**到全球正常的面部。

就飞机而言 我们可以完全忽略原来的正常`in.normal`因为它符合普通地图的Z轴。 但对于任何非横向面孔,我们需要**旋转样本正常**.

你可以用这个来测试[`cylinder.obj`](../../../../data/cylinder.obj)模式 :

```C++
bool success = ResourceManager::loadGeometryFromObj(RESOURCE_DIR "/cylinder.obj", vertexData);
```

```{figure} /images/normal-mapping/wrong-normals.png
:align: center
:class: with-shadow
错误的正常:在从纹理(右)取样正常时,整体照明关闭,与输入正常(左)的效果有很大不同.
```

当我们显示计算正常时,问题变得更加明显:

```rust
/ 要调试正常情况, 我们从( 1, 1) 到( 0, 1) 。
让颜色 = N* 0.5 + 0.5;
```

```{figure} /images/normal-mapping/wrong-normals-debug.png
:align: center
:class: with-shadow
错误的常态:从正常地图(右)中抽取的方向应当旋转,以围绕原来的表面常态(左)居中.
```

###当地框架

为了解决这个问题,我们需要适当界定**本地边框**(即当地人)$X$, $Y$和$Z$轴),其中普通地图表示正常扰动.

####在2D时

这个问题在2D中可能比较容易想象:

```{image} /images/normal-mapping/wrong-mapping-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-mapping/wrong-mapping-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>相对于全局框架, 普通纹理包含正常向量<strong>(左边)</strong>。仅仅用这些抽样的正常情况来代替网状正常情况会导致错误的阴影<strong>(中间)</strong>我们必须把面部正常和样本正常结合起来<strong>(权利)</strong>.</em></span>
</p>

在这个2D的例子中,正确的映射将简单地包括:**总结方向角度**(显微面和显微的正常纹理元素). 但3D比较狡猾

####在三维

3D中的问题也是将正常的纹理产生的普通解释为一种**当地扰动**脸部正常。 只有在这种情况下,取向的概念是由三轴给出的$X$, $Y$和$Z$而不是一个单一角度。

```{image} /images/normal-mapping/local-frame-light.png
:align: center
:class: only-light
```

```{image} /images/normal-mapping/local-frame-dark.png
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>和2D一样,评估扰动正常的正确方法是<strong>组合</strong>从正常的纹理中取样的显微镜正常的,带有局部面部方向。 在3D中,这个面部定向不仅仅是一个角度,而是<strong>整个 XYZ 框架</strong>.</em></span>
</p>

如果从纹理中提取的正常样本是$n = (n_x, n_y, n_z)$,端端普通向量$N$以$n_x X + n_y Y + n_z Z$(我们可以写成矩阵矢量产品)$M n$栏数$M$已经$XYZ$).

**如何构建这个本地正常框架 XYZ ?**我们轮换的关键要求是 当样本正常时$(0,0,1)$,它应该意味着"不扰"。 换句话说,结束的正常矢量$N$必须对应`in.normal`在这样的情况下。 因此,我们的结论是:$Z$正常空间的轴是面部的常态。

我们仍然需要定义一个$X$备注a$Y$完全旋转。 这些斧头分别叫做**正切**和**比特剂**向导

```{image} /images/normal-mapping/bitangent-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-mapping/bitangent-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>如X和Y的这些多重可能性所示,设置Z成为表面正常N并不足以充分定义局部框架.</em></span>
</p>

定义切值的方法很多$X$和比特剂$Y$但重要的是,**定义它与编写工具相同**为了获得对正常纹理的相同解释.

因此,我们使用的定义是:**公约**而不是一个数学需要。

```{note}
此本地框架定义了通常称为“ 常规” 的**正切空间**在阴影点的表面。
```

###色素和比特剂

####计算

正常绘图的本地框架叫做$TBN$(丹增、宾增、普通)和**常规**使用**3D和紫外线坐标**三角面的角,我们绘制一个正常的纹理:

```{image} /images/normal-mapping/local-frame-convention-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-mapping/local-frame-convention-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>以公约形式列出的方向<span class="math notranslate nohighlight">\(T)</span>和<span class="math notranslate nohighlight">\(B\)</span>被定义为此等式。</em></span>
</p>

特别是如果边缘$\bar e_1$在紫外线空间是水平的, 那么$e_1$用作正切方向$T$。这一公式还确保$T$和$B$永远是正交的。

实际上,我们需要计算紫外空间矩阵的反转:

```{image} /images/normal-mapping/local-frame-computation-light.svg
:align: center
:class: only-light
```

```{image} /images/normal-mapping/local-frame-computation-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>紫外空间矩阵像任何<span class="math notranslate nohighlight">\(2\时2\)</span>矩阵通过计算其决定因素<span class="math notranslate nohighlight">\ (\ 德尔塔\ )</span>.</em></span>
</p>

在实践中,我们并不真正需要关注全球多重因素$\Delta$因为我们只是把方向正常化 这导致我们得到以下代码:

```C++
// Compute the TBN local to a triangle face from its corners and return it as
// a matrix whose columns are the T, B and N vectors.
mat3x3 ResourceManager::computeTBN(const VertexAttributes corners[3]) {
	// What we call e in the figure
	vec3 ePos1 = corners[1].position - corners[0].position;
	vec3 ePos2 = corners[2].position - corners[0].position;

	// What we call \bar e in the figure
	vec2 eUV1 = corners[1].uv - corners[0].uv;
	vec2 eUV2 = corners[2].uv - corners[0].uv;

	vec3 T = normalize(ePos1 * eUV2.y - ePos2 * eUV1.y);
	vec3 B = normalize(ePos2 * eUV1.x - ePos1 * eUV2.x);
	vec3 N = cross(T, B);

	return mat3x3(T, B, N);
}
```

我们可以把这个功能在`ResourceManager`因为装药的时候我们会用它

```C++
// In ResourceManager.h
	using mat3x3 = glm::mat3x3;
	// [...]

private:
	// Compute the TBN local to a triangle face from its corners and return it as
	// a matrix whose columns are the T, B and N vectors.
	static mat3x3 computeTBN(const VertexAttributes corners[3]);
```

####Vertex 属性

为了向碎片遮蔽器提供这些切合物和比当物属性,我们把它们储存起来。**额外的顶点属性**.

```C++
// In ResourceManager.h
struct VertexAttributes {
	vec3 position;

	// Texture mapping attributes represent the local frame in which
	// normals sampled from the normal map are expressed.
	vec3 tangent; // T = local X axis
	vec3 bitangent; // B = local Y axis
	vec3 normal; // N = local Z axis

	vec3 color;
	vec2 uv;
};
```

这是**有点多余**因为同一个三角形的所有三个角都会有非常相同的值,但是很容易设置并打开将正常的绘图与每个顶点正常的扰动相结合的可能性.

和往常一样,在添加新的顶点属性时,我们必须更新顶点属性列表`initRenderPipeline()`:

```C++
// In Application.cpp
std::vector<VertexAttribute> vertexAttribs(6);
//                                         ^ This was a 4

// [...]

// Tangent attribute
vertexAttribs[4].shaderLocation = 4;
vertexAttribs[4].format = VertexFormat::Float32x3;
vertexAttribs[4].offset = offsetof(VertexAttributes, tangent);

// Bitangent attribute
vertexAttribs[5].shaderLocation = 5;
vertexAttribs[5].format = VertexFormat::Float32x3;
vertexAttribs[5].offset = offsetof(VertexAttributes, bitangent);
```

为此,我们还需要增加限制:

```C++
requiredLimits.limits.maxVertexAttributes = 6;
//                                          ^ This was a 4
```

最后我们更新了阴影顶点输入:

```rust
垂直输入{
	// [...]
@ 位置(4) 切值: vec3f,
@ 位置(5) 位元: vec3f,
}
```

我们把这些新属性填充在`loadGeometryFromObj`。我选择将它隔离在一个专门的私人职能中:

```C++
bool ResourceManager::loadGeometryFromObj(const path& path, std::vector<VertexAttributes>& vertexData) {
	// [...]
	populateTextureFrameAttributes(vertexData);
	return true;
}
```


这个新的`populateTextureFrameAttributes`函数是一个私有的静态方法`ResourceManager`:

```C++
// In ResourceManager.h
private:
	// Compute Tangent and Bitangent attributes from the normal and UVs.
	static void populateTextureFrameAttributes(std::vector<VertexAttributes>& vertexData);
```

它只是通过呼叫`computeTBN`每个三角形 :

```C++
void ResourceManager::populateTextureFrameAttributes(std::vector<VertexAttributes>& vertexData) {
	size_t triangleCount = vertexData.size() / 3;
	// We compute the local texture frame per triangle
	for (int t = 0; t < triangleCount; ++t) {
		VertexAttributes* v = &vertexData[3 * t];

		mat3x3 TBN = computeTBN(v);

		// We assign these to the 3 corners of the triangle
		for (int k = 0; k < 3; ++k) {
			v[k].tangent = TBN[0];
			v[k].bitangent = TBN[1];
			v[k].normal = TBN[2];
		}
	}
}
```

####使用量

我们现在可以在阴影中访问这个当地TBN框架。 第一步是把它从顶点推进到碎片遮蔽器:

```rust
// 添加正切和位值作为顶点阶段的输出
结构输出{
	// [...]
@ 位置(4) 切值: vec3f,
@ 位置(5) 位元: vec3f,
}
```

在顶点阴影器中,我们只对这些属性应用模型矩阵,就像我们对正常的一样,因为光线随后被计算在世界空间中:

```rust
off.tangent = (uMyUniforms.modelMatrix) (英语).*vec4f(in.tangent, 0.0)). xyz; 2.
ext.bitangent=(uMyUniforms.modelMatrix) 互联网档案馆的存檔,存档日期2013-12-02.*vec4f(in.bitangent, 0.0)). xyz; 维基百科中的相关条目: 维基语录链接:名人名言 - 文学作品 - 谚语 - 谚语 - 谚语 - 谚语
出局。 正常 = (uMyUniforms. modelMatrix) (英语).*vec4f(非正常,0.0))xyz; 2.
```

请注意, 这里还有一个更新的限度, 在阴影组件间计数中添加2乘3浮点数 :

```C++
requiredLimits.limits.maxInterStageShaderComponents = 17;
//                                                    ^^ This was a 11
```

最后,在碎片的阴影中, 我们可以修复我们取样的方式 正常的$N$用于阴影 :

```rust
// 样本正常
(c) 让编码N = 纹理Sample(正常Texture,纹理Sampler, in.uv).rgb;
让本地N = 编码N* 2.0 - 1.0;
// TBN 矩阵将方向从本地空间转换为世界空间
让本地TOWorld = mat3x3f ()
常态( in.tangent),
规范化( in.bitangent),
正常( 正常) ,
);
让 WorldN = 本地ToWorld*当地N;
让 N = 混合(正常、世界N、正常的MapStrength);
```

####结合顶点正常

如果我们现在尝试运行我们的程序, 我们可以有点失望:

```{figure} /images/normal-mapping/flat-mapping.png
:align: center
:class: with-shadow
仅仅从面角计算正切空间不足以获得完全正确的正常映射.
```

我们在这里可以看到两个问题:

- 硬的边缘,我们希望 气瓶是平滑的。
- 面朝此示例中对象的内侧(因为给出三角角的顺序).

这两个问题都是因为一个共同的问题:我们完全忽略了OBJ原始文件提供的顶点正常!

为了利用这些珍贵的信息,我们为各国提供了预期的正常信息。`computeTBN`函数 :

```C++
glm::mat3x3 ResourceManager::computeTBN(const VertexAttributes corners[3], const vec3& expectedN) {
	// [...] Compute T, B, N like before

	// Fix overall orientation
	if (dot(N, expectedN) < 0.0) {
		T = -T;
		B = -B;
		N = -N;
	}

	// Ortho-normalize the (T, B, expectedN) frame
	// a. "Remove" the part of T that is along expected N
	N = expectedN;
	T = normalize(T - dot(T, N) * N);
	// b. Recompute B from N and T
	B = cross(N, T);

	return mat3x3(T, B, N);
}
```

这次我们计算出一个 TBN 框架,每个角都不同:

```C++
// In populateTextureFrameAttributes
for (int k = 0; k < 3; ++k) {
	mat3x3 TBN = computeTBN(v, v[k].normal);
	v[k].tangent = TBN[0];
	v[k].bitangent = TBN[1];
}
```

这次我们得到一个更好的结果!

```{figure} /images/fixed-normal-map.png
:align: center
:class: with-shadow
正常的测绘工作现已全面展开。
```

看正常情况 证实我们是对的:

```{figure} /images/normal-mapping/fixed-normals.png
:align: center
:class: with-shadow
固定的普通地图。
```

你可以试试用四星飞船[`fourareen2K_normals.png`](../../../../data/fourareen2K_normals.png):

```{figure} /images/fourareen-normal-mapping.png
:align: center
:class: with-shadow
Fourareen号的船 和正常的绘图。
```

结论
----------

当正常映射涉及:

- 我们**扰动**常数改为**模拟微细节**没有支付更精细的梅谢斯的费用。
- 我们**样本**这种扰动来自**普通地图**.
- 我们需要**组合**通过旋转抽样值与原常态发生这种扰动。

````{tab} With webgpu.hpp
*结果代码 :* [`step110`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step110)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step110-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step110-vanilla)
````
