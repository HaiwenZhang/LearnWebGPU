再来点制服<span class="bullet">🟢</span>
=============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/shader-uniforms/multiple-uniforms.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/043 - More uniforms - vanilla
:parent: 039 - A first uniform - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/043 - More uniforms
:parent: 039 - A first uniform
```

````{tab} With webgpu.hpp
*结果代码 :* [`step043`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step043)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step043-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step043-vanilla)
````

为了说明统一约束进程的灵活性,让我们增加第二个统一变量,这次控制我们场景的整体颜色。

增加第二件制服有多种方法:

- 在一个不同的束缚组。
- 在同一捆绑的团体中,但不同的捆绑。
- 在同一装订中,将制服的型号改为定制式。

使用**不同的绑定组**能够调用多套制服的制导管道。 例如,一个渲染引擎通常使用不同的绑定组来存储相机和照明信息(仅在帧之间改变),并存储对象信息(位置,方向等),对于同一帧内每个绘图调用都不同.

使用**不同绑定**(同一组)是设定不同的`visibility`取决于约束。 就我们而言,时间只在`Vertex`阴影器,而颜色仅由`Fragment`遮蔽器,这样可以有好处。 然而,我们可以决定使用时间在`Fragment`阴影器最终, 所以我们会使用相同的绑定。

```{note}
另一个原因是不同的绑定物应当指向不同的缓冲物,或者指向同一个缓冲物,即:**冲抵,至少** `deviceLimits.minUniformBufferOffsetAlignment`默认情况下,这个值被设定为我256字节,由我的适配器支持的最小值为64. 加上这些垫子会有点浪费
```

阴影侧
-----------

我们换上制服吧`uTime`类型`f32`以结构为例,我们要求`uMyUniforms`,带有自定义结构类型`MyUniforms`:

```{lit} rust, Declare uniforms (replace, also for tangle root "Vanilla")
/**
 * A structure holding the value of our uniforms
 */
struct MyUniforms {
	time: f32,
	color: vec4f,
};

// Instead of the simple uTime variable, our uniform variable is a struct
@group(0) @binding(0) var<uniform> uMyUniforms: MyUniforms;
```

内`vs_main`,我们替换`uTime`与`uMyUniforms.time`,然后我们使用`uMyUniforms.color`在碎片颜色。

```{lit} rust, Vertex shader (hidden, replace, also for tangle root "Vanilla")
fn vs_main(in: VertexInput) -> VertexOutput {
	var out: VertexOutput;
	let ratio = 640.0 / 480.0;

	// We now move the scene depending on the time!
	var offset = vec2f(-0.6875, -0.463);
	let time = uMyUniforms.time;
	offset += 0.3 * vec2f(cos(time), sin(time));

	out.position = vec4f(in.position.x + offset.x, (in.position.y + offset.y) * ratio, 0.0, 1.0);
	out.color = in.color;
	return out;
}
```

```{lit} rust, Fragment shader (replace, also for tangle root "Vanilla")
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
	// We multiply the scene's color with our global uniform (this is one
	// possible use of the color uniform, among many others).
	let color = in.color * uMyUniforms.color.rgb;

	// Gamma-correction
	let linear_color = pow(color, vec3f(2.2));
	return vec4f(linear_color, 1.0);
}
```

当然,根据你的用法,你会发现一个名字 比"我的Uniforms"更相关, 但让我们坚持下去。

缓冲
------

在CPU方面,我们定义了同样的结构:

```{lit} C++, Define uniform struct (also for tangle root "Vanilla")
/**
 * The same structure as in the shader, replicated in C++
 */
struct MyUniforms {
	float time;
	std::array<float, 4> color;  // or float color[4]
};
```

````{note}
我们用`std::array`类型,需要包含其标题:

```{lit} C++, Includes (append, also for tangle root "Vanilla")
#include <array>
```
````

我们把这个坚实的定义放在`Application`类定义,在私有部分的开头,我们将放置所有内部结构:

```{lit} C++, Application private structs (insert in {{Application class}} after "bool IsRunning();", also for tangle root "Vanilla")
// After public methods, before private things
private:
	// Internal structs
	{{Define uniform struct}}
```

我们还在创建缓冲器时更新其大小:

````{tab} With webgpu.hpp
```{lit} C++, Create uniform buffer (replace)
bufferDesc.size = sizeof(MyUniforms);
//                ^^^^^^^^^^^^^^^^^^ This was 4 * sizeof(float)

bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::Uniform;
bufferDesc.mappedAtCreation = false;
uniformBuffer = device.createBuffer(bufferDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create uniform buffer (replace, for tangle root "Vanilla")
bufferDesc.size = sizeof(MyUniforms);
//                ^^^^^^^^^^^^^^^^^^ This was 4 * sizeof(float)

bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_Uniform;
bufferDesc.mappedAtCreation = false;
uniformBuffer = wgpuDeviceCreateBuffer(device, &bufferDesc);
```
````

因此,初始缓冲上传变为:

````{tab} With webgpu.hpp
```{lit} C++, Upload uniform values (replace)
// Upload the initial value of the uniforms
MyUniforms uniforms;
uniforms.time = 1.0f;
uniforms.color = { 0.0f, 1.0f, 0.4f, 1.0f };
queue.writeBuffer(uniformBuffer, 0, &uniforms, sizeof(MyUniforms));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Upload uniform values (replace, for tangle root "Vanilla")
// Upload the initial value of the uniforms
MyUniforms uniforms;
uniforms.time = 1.0f;
uniforms.color = { 0.0f, 1.0f, 0.4f, 1.0f };
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &uniforms, sizeof(MyUniforms));
```
````

更新缓冲器的值现在看起来是这样的:

````{tab} With webgpu.hpp
```{lit} C++, Update uniform buffer (replace)
// Update uniform buffer
MyUniforms uniforms;
uniforms.time = static_cast<float>(glfwGetTime());
queue.writeBuffer(uniformBuffer, 0, &uniforms, sizeof(MyUniforms));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Update uniform buffer (replace, for tangle root "Vanilla")
// Update uniform buffer
MyUniforms uniforms;
uniforms.time = static_cast<float>(glfwGetTime());
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &uniforms, sizeof(MyUniforms));
```
````

事实上,我们可以更微妙, 只能上传与`time`字段 :

````{tab} With webgpu.hpp
```{lit} C++, Update uniform buffer (replace)
float time = static_cast<float>(glfwGetTime());
// Only update the 1-st float of the buffer
queue.writeBuffer(uniformBuffer, 0, &time, sizeof(float));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Update uniform buffer (replace, for tangle root "Vanilla")
float time = static_cast<float>(glfwGetTime());
// Only update the 1-st float of the buffer
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &time, sizeof(float));
```
````

同样,我们只能更新颜色字节:

````{tab} With webgpu.hpp
```C++
// Update uniform buffer
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
queue.writeBuffer(uniformBuffer, sizeof(float), &uniforms.color, sizeof(Color));
//                               ^^^^^^^^^^^^^ offset of `color` in the uniform struct
```
````

````{tab} Vanilla webgpu.h
```C++
// Update uniform buffer
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
wgpuQueueWriteBuffer(queue, uniformBuffer, sizeof(float), &uniforms.color, sizeof(Color));
//                                         ^^^^^^^^^^^^^ offset of `color` in the uniform struct
```
````

更好的是,如果我们忘记了抵消,或者想灵活地增加新的字段,我们可以使用内置的`offsetof`宏 :

````{tab} With webgpu.hpp
```{lit} C++, Update uniform buffer (replace)
float time = static_cast<float>(glfwGetTime());
// Upload only the time, whichever its order in the struct
queue.writeBuffer(uniformBuffer, offsetof(MyUniforms, time), &time, sizeof(float));
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Update uniform buffer (replace, for tangle root "Vanilla")
float time = static_cast<float>(glfwGetTime());
// Upload only the time, whichever its order in the struct
wgpuQueueWriteBuffer(queue, uniformBuffer, offsetof(MyUniforms, time), &time, sizeof(float));
```
````

如果我们能更新颜色:

````{tab} With webgpu.hpp
```C++
// Upload only the color, whichever its order in the struct
queue.writeBuffer(uniformBuffer, offsetof(MyUniforms, color), &uniforms.color, sizeof(MyUniforms::color));
```
````

````{tab} Vanilla webgpu.h
```C++
// Upload only the color, whichever its order in the struct
wgpuQueueWriteBuffer(queue, uniformBuffer, offsetof(MyUniforms, color), &uniforms.color, sizeof(MyUniforms::color));
```
````


装订布局
--------------


我们增加缓冲器的预期尺寸,首先在**版式**:

```{lit} C++, Define bindingLayout (append, also for tangle root "Vanilla")
bindingLayout.buffer.minBindingSize = sizeof(MyUniforms);
```

还有在**绑定**本身:

```{lit} C++, Setup binding (append, also for tangle root "Vanilla")
binding.size = sizeof(MyUniforms);
```


我们还需要改变绑定的布局 可见度,以便两者`Vertex`和`Fragment`遮蔽器可以访问制服:

````{tab} With webgpu.hpp
```{lit} C++, Define bindingLayout (append)
bindingLayout.visibility = ShaderStage::Vertex | ShaderStage::Fragment;
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Define bindingLayout (append, for tangle root "Vanilla")
bindingLayout.visibility = WGPUShaderStage_Vertex | WGPUShaderStage_Fragment;
```
````


内存布局限制
-------------------------

###对齐

直到现在,我忽略了一件事:GPU的架构对我们在统一的缓冲中组织球场的方式施加了一些限制.

如果我们看看[统一布局限制](https://gpuweb.github.io/gpuweb/wgsl/#address-space-layout-constraints)我们可以看到**页:1**(由`offsetof`类型字段`vec4f` **必须是多个**大小`vec4f`,即16字节. 我们说球场**对齐**至 16 字节。

在现在`MyUniforms`Struct, 这个财产是**未核实**因为`color`页:1`sizeof(float)`这显然不是16字节的倍数! 很简单的解决就是交换`color`和`time`字段 :

```C++
// Don't
struct MyUniforms {
	// offset = 0 * sizeof(f32) -> OK
	float time;

	// offset = 4 -> WRONG, not a multiple of sizeof(vec4f)
	std::array<float,4> color;
};

// Do
struct MyUniforms {
	// offset = 0 * sizeof(vec4f) -> OK
	std::array<float,4> color;

	// offset = 16 = 4 * sizeof(f32) -> OK
	float time;
};
```

```{warning}
如果你用这个`offsetof`宏可以部分更新统一缓冲器,您可以走了。 但是,如果你没有, 一定要反映 重新排序的字段`MyUniforms`你所依赖的任何地方!
```

还有**别忘了**以对阴影码中定义的构造应用相同的更改 !

```{lit} rust, Declare uniforms (replace, also for tangle root "Vanilla")
struct MyUniforms {
	color: vec4f, // <-- this is now first!
	time: f32,
};

@group(0) @binding(0) var<uniform> uMyUniforms: MyUniforms;
```

###铺垫

对制服类型的另一个限制是它们必须是:[可共享主机](https://gpuweb.github.io/gpuweb/wgsl/#host-shareable),这与[对总结构规模的限制](https://gpuweb.github.io/gpuweb/wgsl/#alignment-and-size).

基本上,总尺寸必须是**最大字段对齐大小的倍数**。在我们的情况下,这意味着它必须是16字节的倍数(大小为`vec4f`).

因此,我们添加**粘贴**到我们的结构,即结尾处一个未使用的属性,以额外的字节填充:

```{lit} C++, Define uniform struct (replace, also for tangle root "Vanilla")
struct MyUniforms {
	std::array<float,4> color;
	float time;
	float _pad[3];
};
// Have the compiler check byte alignment
static_assert(sizeof(MyUniforms) % 16 == 0);
```

终于成功了!

```{figure} /images/webgpu-logo-tinted.png
:align: center
:class: with-shadow
WebGPU的标志,用我们新的制服颜色涂上。
```

结论
----------

我们在这里看到,提供多种制服通常是通过提供一种由多个领域组成的结构的单一制服来实现的。 重要的是,这些领域有内存协调方面的限制。

```{seealso}
我开始写了[在线工具](https://eliemichel.github.io/WebGPU-AutoLayout)以自动生成匹配 WGSL 结构的 C++ 结构。 注意它使用该类型`vec3`从 GLM 库取而代之`std::array<float,3>`但它很容易被替换。
```

````{tab} With webgpu.hpp
*结果代码 :* [`step043`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step043)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step043-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step043-vanilla)
````
