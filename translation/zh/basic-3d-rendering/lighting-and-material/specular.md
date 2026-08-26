光谱<span class="bullet">🟡</span>
===========

```{translation-warning} 译文可能已过时, /basic-3d-rendering/lighting-and-material/specular.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step105`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step105)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step105-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step105-vanilla)
````

我们的物质模型首先缺乏的是**光谱突出显示**。这是可以在任何闪亮物体,即任何非100%粗糙物体上看到的视觉效果:

```{figure} /images/pexels-othmar-vigl-792051.jpg
:align: center
:class: with-shadow
卒身上的白色亮点是一种典型的光谱效应.
```

扩散灯和光谱灯的关键区别在于后者**取决于视图点**!

环顾四周,定位任何光泽/反射物体,移动你的头部. 你应该注意,光谱亮度 移动的同时**对象和光源都没有改变**.

物理上,这是因为表面的单点会发出**不同方向的不同亮度**.

```{image} /images/specular-light.svg
:align: center
:class: only-light
```

```{image} /images/specular-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>从两个不同的视点V和V'得到的光线是不同的,尽管光线方向L和表面正常N仍然不变.</em></span>
</p>

Phong型号
-----------

在理论上,任何表面元素都在一定程度上反映了它从所有方向得到的光. 但是,当做实时渲染, 我们负担不起计算 准确这一点,我们需要一个**近似模型**可有效计算。

那个[Phong型号](https://en.wikipedia.org/wiki/Phong_reflection_model)只考虑光线从光线方向发出的光(它不考虑物体间的反射). 它是最古老的反射模型之一,但它仍然是一个良好的开端。

至今为止,我们使用的叫做`diffuse`术语,不取决于观点。 我们现在添加一个`specular`术语:

```rust
// 在碎片遮蔽器中光源的循环中
让扩散=最大(0.0,点(方向,正常))*颜色; // 我们的模型至今
让 specular = vec3f( 0.0); // 待完成
阴影 + 扩散 + 光谱;
```

```{note}
Phong模型还包括一个环境术语,这是一个来自环境的恒定值. 我们忽略了它,因为我们很快会把Phong模型完全替换为一个更准确的模型.
```

在本章的其余部分,我们着重讨论`specular`术语,所以我们设置`diffuse`部分为0。

###反映方向

我们从一个极端的例子开始:**如果表面是一个完美的镜子**然后是位于方向的光源的影响$L$只在表面可见**反映方向** $R$(见下图)。 表面看起来是黑色的 从其他任何地方!

```{image} /images/reflected-light.svg
:align: center
:class: only-light
```

```{image} /images/reflected-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>V'比V看到更多的反射光,因为它更接近反射方向R.</em></span>
</p>

这个反映的方向是:**光线方向的对称** $L$财务报告和审定财务报表$N$。由于这是一个非常常见的操作,它得到了WGSL(和其他阴影语言)的本土支持:

```rust
让 L = 方向;
让 N = 正常;
让 R = 反射(-L, N); // 等于 2.0*点( N, L)*无 - L
```

```{note}
那个`reflect`函数假设我们给出的方向是未来的方向**从**光线,但其余的代码 我们定义*L级*作为方向**目标**灯光
```

为了一个完美的镜子,我们会有类似的东西`specular = (R == V)`但是这从来就没有完全平等过**反正没有什么是完美的镜子**。相反,我们使用视图方向V与反射方向R之间的角度,或者使用其余弦,因为比较容易计算:

```rust
让 cosAngle = 点(R, V);
/ 余弦越接近1.0, V越接近R
var 谱数 = 0.0;
如果( cosAngle > 0. 99){
光谱=1.0;
}
```

###查看方向

# # 我如何获得方向$V$是吗?

事实并不明显。 首先,我们要计算一下**顶点阴影器**让光栅喷射出来,

```rust
结构输出{
	// [...]
@ 位置(3) 查看方向: vec3f,
}

/ 在顶点阴影中
。视图方向=/* ... */;

// 在碎片阴影中
让 V = 正常化( in.viewDirection);
```

````{note}
同往常一样,在从顶点向碎片遮蔽器中添加更多转动值时,我们需要将以下限制更新到新的尺寸。`VertexOutput`在阴影中:
```C++
requiredLimits.limits.maxInterStageShaderComponents = 11;
//                                                    ^ This was 8
```
````

```{caution}
我们正常`V` **在碎片阴影中**因为即使所有`out.viewDirection`数值正常化,光栅化后的插值一般不再完全正常化。
```

那么,我们如何计算`viewDirection`在顶点? 我们可以分界线`out.position`为了获得**世界空间坐标**当前顶点,在投影之前:

```rust
让 WorldPositions = uMyUniforms. modelMatrix (英语).*vec4f(在位,1.0);
出局。 位置 = uMyUniforms. projection Matrix*uMyUniforms.viewMatrix 页面存档备份,存于互联网档案馆*世界前景;

/ 然后,我们只需要相机位置来获得视图方向:
让相机WorldPosition = / 显示* ... */
Out.viewDirection = 相机WorldPosition - worldPosition.xyz; 互联网档案馆的存檔,存档日期2013-03-04.
```

```{important}
我们在工作**世界空间坐标**因为这就是我们表达光线方向的空间。 我们可以采取不同的做法,并且只能在闭路空间操纵价值,例如,重要的是**一致**.
```

相机位置的信息不知何故包含在`viewMatrix`,但提取它需要计算矩阵反向,成本高昂. 因此,不建议在阴影处执行:每个顶点的计算是:**浪费**因为它是相同的 所有顶点。

```{note}
在进行GPU编程时,可能发生重计算数倍相同数量比存储在内存中效率更高的情况:一些管道确实是**内存绑定**,意味着GPU花费更多的时间等待内存访问,而不是计算. 见[调试](../../appendices/debugging.md)详见第一章。
```

就我们而言,显然最好以制服提供摄像机位置:

```rust
让相机WorldPosition = u MyUniforms.camera WorldPosition; 允许相机世界(英语:WorldPosition = u My Uniforms.camera WorldPosition), 允许相机世界(英语:camera WorldPosition), 允许相机世界(英语:camera World Position) = u My Uniforms.
```

```rust
结构化 MyUniforms{
投影Matrix: mat4x4f,
视图Matrix: mat4x4f,
型号Matrix: mat4x4f,
颜色: vec4f,
相机 World Postition: vec3f, // 新字段 !
时间:f32,
};
```

注意将字段从最大的字段排序到最小的字段(地图为第一,浮在尾端),以及**还在 C++ 对应中添加此内容**这个结构。

现在,我们每次更新视图矩阵,我们也可以更新这个新的统一字段:

````{tab} With webgpu.hpp
```C++
void Application::updateViewMatrix() {
	// [...]
	m_uniforms.cameraWorldPosition = position;
	m_device.getQueue().writeBuffer(
		m_uniformBuffer,
		offsetof(MyUniforms, cameraWorldPosition),
		&m_uniforms.cameraWorldPosition,
		sizeof(MyUniforms::cameraWorldPosition)
	);
}
```
````

````{tab} Vanilla webgpu.h
```C++
void Application::updateViewMatrix() {
	// [...]
	m_uniforms.cameraWorldPosition = position;
	WGPUQueue  = wgpuDeviceGetQueue(m_device);
	wgpuQueueWriteBuffer(
		queue,
		m_uniformBuffer,
		offsetof(MyUniforms, cameraWorldPosition),
		&m_uniforms.cameraWorldPosition,
		sizeof(MyUniforms::cameraWorldPosition)
	);
}
```
````

````{note}
或者,你可以做整个阴影在**视图空间**。在视图空间中的相机位置总是`vec3f(0.0)`你只需要改变光线方向 就像我们对顶点正常一样 除了使用视图矩阵

```rust
让 L_视图空间 = (uMyUniforms.viewMatrix)*vec4f (中文(简体) ). L级_世界空间,0.0)xyz;
```
````

###红光谱

现在我们可以计算`R`和`V`,让我们测试设置`shading += specular;`在阴影中光源的循环中:

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/specular-strong.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">反射一词创造了一个突出点,它像一个真实的光谱突出点一样移动,尽管它有点苛刻.</span></p>
	</figcaption>
</figure>

在余弦角度上使用硬阈值(0.99)有点过强. 更好的模型包括应用`pow`切换到余弦**重新绘制地图**它变成仍然凝结在1.0左右的东西, 但是当我们离开时,它会更顺利地减少。

能量激发器越大,我们离光照镜的行为越近。 Phong模式就叫这种力量**硬度**:

```rust
/我们把点产品夹到 0 当它是负的
让ROV=最大值(0.0,点(R,V));
让硬度=2.0;
(b) 让光谱=pow(RV,硬度);
```

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/specular-hardness2.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">平滑的光谱突出,硬度=2.0.</span></p>
	</figcaption>
</figure>

硬度参数控制了光谱亮度的大小:其高度是亮度最小的.

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/specular-hardness32.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">硬度的光谱突出值=32.0.</span></p>
	</figcaption>
</figure>

合并
-------------

我们现在可以重新整理照明的分散和光谱贡献。 我们可以稍稍重组代码**只有散开词**乘以基准颜色。 这是因为对于非金属来说,光谱亮度总是白色的(我们将在关于[物理材料](pbr.md)) (中文(简体) ). 我们还增加常数,以平衡扩散和光谱效应:

```rust
(b) 以下列方式处理:
让 kd = 1.0; // 扩散效果的强度
让 ks = 0.5; // 光谱效果的强度

var 颜色 = vec3f(0.0);
用于( var i: i32 = 0; i < 2; i++){
	// [...]
让扩散=/* [...] */;
让光谱=/* [...] */;
颜色 QQ 基色*页:1*扩散 + ks*光谱;
}
```

<figure class="align-center">
	<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:642px">
		<source src="../../_static/phong-hardness16-ks0.5.mp4" type="video/mp4">
	</video>
	<figcaption>
		<p><span class="caption-text">diffuse和光谱贡献综合,硬度=16.0,kd=1.0,ks=0.5.</span></p>
	</figcaption>
</figure>

常数`kd`和`ks`是材料的属性 来说明它是多少光泽。 我建议你在GUI中曝光这些,这样你就可以玩了!

```C++
changed = ImGui::SliderFloat("Hardness", &m_lightingUniforms.hardness, 1.0f, 100.0f) || changed;
changed = ImGui::SliderFloat("K Diffuse", &m_lightingUniforms.kd, 0.0f, 1.0f) || changed;
changed = ImGui::SliderFloat("K Specular", &m_lightingUniforms.ks, 0.0f, 1.0f) || changed;
```

```{figure} /images/specular-ui.png
:align: center
:class: with-shadow
在照明图形界面中曝光的材料属性 。
```

结论
----------

在本章中我们得到了一个**好直觉**如何模拟材料的光谱亮度。 下几章通过首先修改**当地正常**灯光反弹,第二,引入更多**实体**材料模型。

````{tab} With webgpu.hpp
*结果代码 :* [`step105`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step105)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step105-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step105-vanilla)
````
