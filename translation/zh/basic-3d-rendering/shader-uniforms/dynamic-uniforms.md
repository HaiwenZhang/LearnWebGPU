动态制服<span class="bullet">🟡</span>
================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/shader-uniforms/dynamic-uniforms.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{admonition} 🚧 WIP
从本章起,指南使用先前版本的配套代码(特别是,它没有定义一个`Application`班级,而是把一切 在一个单一的`main`函数)。**我现在正在刷新**逐章,这是**我现在在工作**!
```

````{tab} With webgpu.hpp
*结果代码 :* [`step044`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step044)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step044-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step044-vanilla)
````

想象我们想发行**两个电话**页:1`draw`我们输油管的方法**有不同数值**,以绘制两个不同颜色的WebGPU标志。 我们自然可以试试:

````{tab} With webgpu.hpp
```C++
// THIS WON'T WORK!

// A first color
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
queue.writeBuffer(uniformBuffer, offsetof(MyUniforms, color), &uniforms.color, sizeof(MyUniforms::color));

// First draw call
renderPass.drawIndexed(indexCount, 1, 0, 0, 0);

// Different location and different color for another draw call
uniforms.time += 1.0;
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
queue.writeBuffer(uniformBuffer, 0, &uniforms, sizeof(MyUniforms));

// Second draw call
renderPass.drawIndexed(indexCount, 1, 0, 0, 0);
```
````

````{tab} Vanilla webgpu.h
```C++
// THIS WON'T WORK!

// A first color
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
wgpuQueueWriteBuffer(queue, uniformBuffer, offsetof(MyUniforms, color), &uniforms.color, sizeof(MyUniforms::color));

// First draw call
wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);

// Different location and different color for another draw call
uniforms.time += 1.0;
uniforms.color = { 1.0f, 0.5f, 0.0f, 1.0f };
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &uniforms, sizeof(MyUniforms));

// Second draw call
wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);
```
````

这是合法的,但是**将不这样做**你所期待的,你所期待的。 记住命令已经执行**同步**当我们称之为方法`renderPass`对象,我们没有真正触发操作, 我们宁愿建立一个命令缓冲器,**立即发送**在结尾。 ┮筿杠璶`writeBuffer` **别**按我们的要求,在抽奖电话之间互换

相反,我们需要使用**动态统一缓冲器**。这是一个简单的选项,可以在绑定布局中打开,但需要小心缓冲器**脚步**(见下文)。

设备限制
-------------

和往常一样,我们先检查一下 我们使用的特性在某个地方`WGPULimits`构建并加入设备创建代码中的要求:

```C++
// Extra limit requirement
requiredLimits.limits.maxDynamicUniformBuffersPerPipelineLayout = 1;
```

另一个相关的限制是`minUniformBufferOffsetAlignment`,我们已经设定为适配器支持的最低值(见下文)。

装订布局
--------------

当宣布绑定组布局时,我们可以**将缓冲设置为动态抵消**:

````{tab} With webgpu.hpp
```C++
// Create binding layouts
BindGroupLayoutEntry bindingLayout = Default;
// [...]
// Make this binding dynamic so we can offset it between draw calls
bindingLayout.buffer.hasDynamicOffset = true;
```
````

````{tab} Vanilla webgpu.h
```C++
// Create binding layouts
WGPUBindGroupLayoutEntry bindingLayout;
setDefault(bindingLayout);
// [...]
// Make this binding dynamic so we can offset it between draw calls
bindingLayout.buffer.hasDynamicOffset = true;
```
````

这一动态抵消的价值以后将转给`renderPass.setBindGroup`.

缓冲数据
-----------

基本想法是有一个缓冲器**大小的两倍**页:1`MyUniforms`。对于第一个绘图调用,我们把动态偏移设定为0,以便它使用第一组数值,然后我们再发布第二个绘图调用,并抵消`sizeof(MyUniforms)`指向缓冲器的下半部分。

有个**记住一件事**但:抵消额的数值受限制**倍数**联合国`minUniformBufferOffsetAlignment`限制设备。

```
  ---------------------------------     ---------------------------------
 |尔|(单位:千美元)|3个P-4|(单位:千美元)|页:1| _ | _ | _ |  ...  |尔|(单位:千美元)|3个P-4|(单位:千美元)|页:1| _ | _ | _ |
  ---------------------------------     ---------------------------------
^^^^^^ My Uniform 结构的二审
MyUniform结构的初审
```

这意味着**脚步**,即第一个字节之间的字节数。`r`和第二个`r`上方,必须四舍五入到最接近的倍数`minUniformBufferOffsetAlignment`:

````{tab} With webgpu.hpp
```C++
SupportedLimits supportedLimits;
device.getLimits(&supportedLimits);
Limits deviceLimits = supportedLimits.limits;
//[...]

// Subtlety
uint32_t uniformStride = ceilToNextMultiple(
	(uint32_t)sizeof(MyUniforms),
	(uint32_t)deviceLimits.minUniformBufferOffsetAlignment
);
```
````

````{tab} Vanilla webgpu.h
```C++
WGPUSupportedLimits supportedLimits;
wgpuDeviceGetLimits(device, &supportedLimits);
WGPULimits deviceLimits = supportedLimits.limits;
//[...]

// Subtlety
uint32_t uniformStride = ceilToNextMultiple(
	(uint32_t)sizeof(MyUniforms),
	(uint32_t)deviceLimits.minUniformBufferOffsetAlignment
);
```
````

如果函数由:

```C++
/**
 * Round 'value' up to the next multiplier of 'step'.
 */
uint32_t ceilToNextMultiple(uint32_t value, uint32_t step) {
	uint32_t divide_and_ceil = value / step + (value % step == 0 ? 0 : 1);
	return step * divide_and_ceil;
}
```

我们现在可以创建缓冲器并上传2组不同的值:

````{tab} With webgpu.hpp
```C++
// The buffer will contain 2 values for the uniforms plus the space in between
// (NB: stride = sizeof(MyUniforms) + spacing)
bufferDesc.size = uniformStride + sizeof(MyUniforms);

// [...]

MyUniforms uniforms;

// Upload first value
uniforms.time = 1.0f;
uniforms.color = { 0.0f, 1.0f, 0.4f, 1.0f };
queue.writeBuffer(uniformBuffer, 0, &uniforms, sizeof(MyUniforms));

// Upload second value
uniforms.time = -1.0f;
uniforms.color = { 1.0f, 1.0f, 1.0f, 0.7f };
queue.writeBuffer(uniformBuffer, uniformStride, &uniforms, sizeof(MyUniforms));
//                               ^^^^^^^^^^^^^ beware of the non-null offset!
```
````

````{tab} Vanilla webgpu.h
```C++
// The buffer will contain 2 values for the uniforms plus the space in between
// (NB: stride = sizeof(MyUniforms) + spacing)
bufferDesc.size = uniformStride + sizeof(MyUniforms);

// [...]

MyUniforms uniforms;

// Upload first value
uniforms.time = 1.0f;
uniforms.color = { 0.0f, 1.0f, 0.4f, 1.0f };
wgpuQueueWriteBuffer(queue, uniformBuffer, 0, &uniforms, sizeof(MyUniforms));

// Upload second value
uniforms.time = -1.0f;
uniforms.color = { 1.0f, 1.0f, 1.0f, 0.7f };
wgpuQueueWriteBuffer(queue, uniformBuffer, uniformStride, &uniforms, sizeof(MyUniforms));
//                                         ^^^^^^^^^^^^^ beware of the non-null offset!
```
````

绘图
-------

在前几章中,我们没有使用最后两个论点:`renderPass.setBindGroup`,即:`dynamicOffsetCount`和`dynamicOffsets`数组。 它们是提供不同呼号的抵消方式。 为了改变偏移,我们只用不同的偏移来重新组合绑定组.

如果我们有多种动态制服, 我们需要指出一个阵列, 但是,由于我们只有一个, 我们可以只是给出一个地址`dynamicOffset`变量 :

````{tab} With webgpu.hpp
```C++
renderPass.setPipeline(pipeline);

uint32_t dynamicOffset = 0;

// Set binding group
dynamicOffset = 0 * uniformStride;
renderPass.setBindGroup(0, bindGroup, 1, &dynamicOffset);
renderPass.drawIndexed(indexCount, 1, 0, 0, 0);

// Set binding group with a different uniform offset
dynamicOffset = 1 * uniformStride;
renderPass.setBindGroup(0, bindGroup, 1, &dynamicOffset);
renderPass.drawIndexed(indexCount, 1, 0, 0, 0);

renderPass.end();
```
````

````{tab} Vanilla webgpu.h
```C++
wgpuRenderPassEncoderSetPipeline(renderPass, pipeline);

uint32_t dynamicOffset = 0;

// Set binding group
dynamicOffset = 0 * uniformStride;
wgpuRenderPassEncoderSetBindGroup(renderPass, 0, bindGroup, 1, &dynamicOffset);
wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);

// Set binding group with a different uniform offset
dynamicOffset = 1 * uniformStride;
wgpuRenderPassEncoderSetBindGroup(renderPass, 0, bindGroup, 1, &dynamicOffset);
wgpuRenderPassEncoderDrawIndexed(renderPass, indexCount, 1, 0, 0, 0);

wgpuRenderPassEncoderEnd(renderPass);
```
````

```{note}
另一种解决办法可能是建立两个不同的结合组,指向两个不同的缓冲. 但动态抵消办法**比较好**在发出大量带有不同制服的抽奖电话时。
```

结论
----------

我们现在对制服感到很舒服,我们准备转向实际的3D形状!

```{figure} /images/webgpu-logo-double.png
:align: center
:class: with-shadow
我们用不同的价值画了两次场景**动态制服**.
```

````{tab} With webgpu.hpp
*结果代码 :* [`step044`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step044)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step044-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step044-vanilla)
````
