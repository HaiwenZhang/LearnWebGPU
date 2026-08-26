照明控制<span class="bullet">🟡🟢</span>
================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/some-interaction/lighting-control.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step100`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step100-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-vanilla)
````

```{important}
**2024年11月17日 (中文(简体) ).**本章以二级标记**绿色子弹**只为了引起注意,因为有一个附带代码的预览,可供更近的版本Dawn和`wgpu-native`: [`step100-next`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-next)。本章的内容仍然依赖于旧版本。
```

现在我们有了GUI的元素, 我们可以用它们来揭露例如**照明设置**给用户。 我们希望他们能够**活在摇摆**光源的方向和颜色

基本阴影上的复选
----------------------

让我们首先回顾一下我们在[基本阴影](../3d-meshes/basic-shading.md)章节。

- 表面元素的遮蔽外观取决于其**常规**.
- 这个表情也取决于**光源**特别是在他们的方向。
- 我们测试的最基本阴影是`shading = max(0.0, dot(lightDirection, normal))`。这是(灯塔)**扩散**阴影。

```{seealso}
关于这个简单的扩散模型背后的理性细节,你可以在Scratchpixel上查阅这个很好的介绍:[迪弗斯和兰贝蒂安·沙丁](https://www.scratchapixel.com/lessons/3d-basic-rendering/introduction-to-shading/diffuse-lambertian-shading.html).
```

我们可以用纹理采样来插上这个,在应用阴影之前用纹理作为底色:

```rust
@ flagment (英语).
fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
/ 计算阴影
让正态=正态(非正态);
使光导1=vec3f(0.5、-0.9、0.1);
使光导2=vec3f(0.2、0.4、0.3);
(c) 允许轻Color1=vec3f(1.0、0.9、0.6);
让浅色Color2=vec3f(0.6、0.9、1.0);
让阴影1=最大值(0.0,点(光线1,普通));
让阴影2=最大值(0.0,点(光线2,普通));
让阴影=阴影1*浅色1 + 阴影2*浅色2;
	
// 样本纹理
(b) 以下列方式处理:

// 结合纹理和照明
让颜色=基色*阴影;

// 伽玛校正
更正_颜色=poow(颜色,vec3f(2.2));
返回 vec4f(已更正)_(a) 颜色;
}
```

```{note}
我重新命名了所谓的`gradientTexture`输入`baseColorTexture`.
```

```{figure} /images/lit-boat.png
:align: center
:class: with-shadow
船型带有一些基本的照明.
```

照明制服
-----------------

为了方便在下一个聊天室进一步测试照明和材料,让我们**使光源具有动态**把它们连接到我们的GUI。

与其用硬码编码灯光设置,不如用制服传递:

```rust
/**
 *具有照明设置的结构
 */
结构灯光统一{
方向: 数组<vec4f, 2>,
颜色: 数组<vec4f, 2>,
}

@ group( 0) @ 绑定 (3) var<uniform>uLighting:灯光统一;

@ flagment (英语).
fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
/ 计算阴影
让正态=正态(非正态);
var 阴影=vec3f(0.0);
用于( var i: i32 = 0; i < 2; i++){
让方向=常态(ULighting). 方向[(单位:千美元)].xyz) (英语).
让颜色 = uLighting。颜色[(单位:千美元)]rgb; (中文(简体) ).
显示最大值( 0.0, 点( 方向, 一般) )*颜色;
	}
	
// 样本纹理
(b) 以下列方式处理:

// 结合纹理和照明
让颜色=基色*阴影;

// 伽玛校正
更正_颜色=poow(颜色,vec3f(2.2));
返回 vec4f(已更正)_(a) 颜色;
}
```

为此,我们需要:

- 创建一个新的**统一缓冲器**.
- 创建一个新的**绑定**为这身制服而装订的团体。
- 把这捆绑起来**绑定组布局**.

###制服

首先,我们复制`LightingUniforms`在 C++ 代码中构建 :

```C++
#include <array>

// Before Application's private attributes
struct LightingUniforms {
	std::array<vec4, 2> directions;
	std::array<vec4, 2> colors;
};
static_assert(sizeof(LightingUniforms) % 16 == 0);
```

```{caution}
注意我如何改变方向和颜色`vec4`改为`vec3`。这是因为[校正规则](https://www.w3.org/TR/WGSL/#alignment-and-size)页:1`vec3`被对齐为`vec4`.
```

然后我们用这个结构来创建属性,并定义它的GPU侧对应:

````{tab} With webgpu.hpp
```C++
wgpu::Buffer m_lightingUniformBuffer = nullptr;
LightingUniforms m_lightingUniforms;
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUBuffer m_lightingUniformBuffer = nullptr;
LightingUniforms m_lightingUniforms;
```
````

我们创造了三种方法来管理这个新的统一缓冲器:

```C++
// In class Application:
bool initLightingUniforms(); // called in onInit()
void terminateLightingUniforms(); // called in onFinish()
void updateLightingUniforms(); // called when GUI is tweaked
```

这些与我们的其他制服非常相似:

````{tab} With webgpu.hpp
```C++
bool Application::initLightingUniforms() {
	// Create uniform buffer
	BufferDescriptor bufferDesc;
	bufferDesc.size = sizeof(LightingUniforms);
	bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Uniform;
	bufferDesc.mappedAtCreation = false;
	m_lightingUniformBuffer = m_device.createBuffer(bufferDesc);

	// Initial values
	m_lightingUniforms.directions[0] = { 0.5f, -0.9f, 0.1f, 0.0f };
	m_lightingUniforms.directions[1] = { 0.2f, 0.4f, 0.3f, 0.0f };
	m_lightingUniforms.colors[0] = { 1.0f, 0.9f, 0.6f, 1.0f };
	m_lightingUniforms.colors[1] = { 0.6f, 0.9f, 1.0f, 1.0f };

	updateLightingUniforms();

	return m_lightingUniformBuffer != nullptr;
}

void Application::terminateLightingUniforms() {
	m_lightingUniformBuffer.destroy();
	m_lightingUniformBuffer.release();
}

void Application::updateLightingUniforms() {
	m_queue.writeBuffer(m_lightingUniformBuffer, 0, &m_lightingUniforms, sizeof(LightingUniforms));
}
```
````

````{tab} Vanilla webgpu.h
```C++
bool Application::initLightingUniforms() {
	// Create uniform buffer
	WGPUBufferDescriptor bufferDesc = {};
	bufferDesc.size = sizeof(LightingUniforms);
	bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Uniform;
	bufferDesc.mappedAtCreation = false;
	m_lightingUniformBuffer = wgpuDeviceCreateBuffer(m_device, bufferDesc);

	// Initial values
	m_lightingUniforms.directions[0] = { 0.5f, -0.9f, 0.1f, 0.0f };
	m_lightingUniforms.directions[1] = { 0.2f, 0.4f, 0.3f, 0.0f };
	m_lightingUniforms.colors[0] = { 1.0f, 0.9f, 0.6f, 1.0f };
	m_lightingUniforms.colors[1] = { 0.6f, 0.9f, 1.0f, 1.0f };

	updateLightingUniforms();

	return m_lightingUniformBuffer != nullptr;
}

void Application::terminateLightingUniforms() {
	wgpuBufferDestroy(m_lightingUniformBuffer);
	wgpuBufferRelease(m_lightingUniformBuffer);
}

void Application::updateLightingUniforms() {
	wgpuQueueWriteBuffer(m_queue, m_lightingUniformBuffer, 0, &m_lightingUniforms, sizeof(LightingUniforms));
}
```
````

等会儿再打给你`updateLightingUniforms()`但在此之前,我们需要为这个新的统一缓冲器增加约束。

###约束

内`initBindGroup`,添加新的约束:

````{tab} With webgpu.hpp
```C++
std::vector<BindGroupEntry> bindings(4);
//                                   ^ This was a 3
// [...]

bindings[3].binding = 3;
bindings[3].buffer = m_lightingUniformBuffer;
bindings[3].offset = 0;
bindings[3].size = sizeof(LightingUniforms);
```
````

````{tab} Vanilla webgpu.h
```C++
std::vector<WGPUBindGroupEntry> bindings(4);
//                                       ^ This was a 3
// [...]

bindings[3].binding = 3;
bindings[3].buffer = m_lightingUniformBuffer;
bindings[3].offset = 0;
bindings[3].size = sizeof(LightingUniforms);
```
````

当然,我们还必须把这个加到约束群体中**版式**因为改变束缚群体及其布局是非常常见的,所以我愿意建立一个`initBindGroupLayout()`坐在旁边的方法`initBindGroup()`在代码中,通过它不 完全调用之前:

```C++
bool Application::onInit() {
	// [...]
	// Bind group layout is init before the pipeline
	if (!initBindGroupLayout()) return false;
	if (!initRenderPipeline()) return false;
	// [...]
	// Bind group is init after the resources it binds
	if (!initBindGroup()) return false;
	// [...]
}
```

从`initRenderPipeline()`,然后添加新的照明制服缓冲器:

```C++
bool Application::initBindGroupLayout() {
	std::vector<BindGroupLayoutEntry> bindingLayoutEntries(4, Default);
	//                                                     ^ This was a 3

	// [...]

	// The lighting uniform buffer binding
	BindGroupLayoutEntry& lightingUniformLayout = bindingLayoutEntries[3];
	lightingUniformLayout.binding = 3;
	lightingUniformLayout.visibility = ShaderStage::Fragment; // only Fragment is needed
	lightingUniformLayout.buffer.type = BufferBindingType::Uniform;
	lightingUniformLayout.buffer.minBindingSize = sizeof(LightingUniforms);

	// [...]
}
```

````{note}
由于我们增加了一个新的统一缓冲器,不要忘记更新所需的限制:

```C++
requiredLimits.limits.maxUniformBuffersPerShaderStage = 2;
```
````

###图形界面

现在让我们通过替换“你好,世界”面板来连接图形界面`updateGui()`:

```C++
// In Application::updateGui:
ImGui::Begin("Lighting");
ImGui::ColorEdit3("Color #0", glm::value_ptr(m_lightingUniforms.colors[0]));
ImGui::DragFloat3("Direction #0", glm::value_ptr(m_lightingUniforms.directions[0]));
ImGui::ColorEdit3("Color #1", glm::value_ptr(m_lightingUniforms.colors[1]));
ImGui::DragFloat3("Direction #1", glm::value_ptr(m_lightingUniforms.directions[1]));
ImGui::End();
```

```{note}
那个`glm::value_ptr`函数返回指针到 glm 对象的原始数据存储位置,这是 ImGui 需要操作的。 在对矢量的实践中,它相当于使用矢量对象本身的地址.
```

这将改变`m_lightingUniforms`当用户调整滑动器时。 为了反映GPU侧制服中的这些变化,我们在每个框架复制GUI操纵的值到GPU侧缓冲器.

```C++
void Application::onFrame() {
	updateLightingUniforms();
	// [...]
}
```

```{figure} /images/light-color-ui.png
:align: center
:class: with-shadow
UI与照明制服相连.
```

补充
----------

###自定义图形界面输入

这个`ImGui::DragFloat3`输入对方向并不理想。 它不是那么直观,也不保证方向矢量总有1长. 我们很容易创造出自己的投入`ImGui::DragDirection`以两个极角显示方向:

```C++
#include <glm/gtx/polar_coordinates.hpp>

namespace ImGui {
bool DragDirection(const char* label, vec4& direction) {
	vec2 angles = glm::degrees(glm::polar(vec3(direction)));
	bool changed = ImGui::DragFloat2(label, glm::value_ptr(angles));
	direction = vec4(glm::euclidean(glm::radians(angles)), direction.w);
	return changed;
}
} // namespace ImGui
```

其用途如下:

```C++
ImGui::DragDirection("Direction #0", m_lightingUniforms.directions[0]);
```

###简单优化

即使没有变化,我们也可以从ImGui那里获得字段是否改变的信息:

```C++
// In class Application
bool m_lightingUniformsChanged = false;

// In Application.cpp
void Application::updateLighting() {
	if (m_lightingUniformsChanged) {
		// [...] Update uniforms
		m_lightingUniformsChanged = false;
	}
}

// In Application::updateGui:
bool changed = false;
ImGui::Begin("Lighting");
changed = ImGui::ColorEdit3("Color #0", glm::value_ptr(m_lightingUniforms.colors[0])) || changed;
changed = ImGui::DragDirection("Direction #0", m_lightingUniforms.directions[0]) || changed;
changed = ImGui::ColorEdit3("Color #1", glm::value_ptr(m_lightingUniforms.colors[1])) || changed;
changed = ImGui::DragDirection("Direction #1", m_lightingUniforms.directions[1]) || changed;
ImGui::End();
m_lightingUniformsChanged = changed;
```

结论
----------

在本章中,我们:

- 将扩散阴影与纹理相连,
- 用GUI连接照明。
- 创建了自定义图形界面。

好了,我们现在准备好 潜入物质模型真正的!

````{tab} With webgpu.hpp
*结果代码 :* [`step100`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step100-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step100-vanilla)
````
