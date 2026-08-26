向导过滤器<span class="bullet">🟡</span>
===================

```{translation-warning} 译文可能已过时, /basic-compute/image-processing/convolution-filters.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step215`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step215)

许多**图像处理算法**其基础是进化过滤操作 在他们的管道的某处。

概念很简单:每个像素的新值是通过查看一个像素计算出来的**滑动窗口**以像素为中心,将其乘以固定**内核**并总结.

```{image} /images/convolution/problem-light.png
:align: center
:class: only-light
```

```{image} /images/convolution/problem-dark.png
:align: center
:class: only-dark
```

<p class="align-center">
    <span class="caption-text"><em>垂直的Sobel滤波器是通过一个3乘3像素的窗口时滑动1像素,用给定的内核将其内容乘以.</em></span>
</p>

那个**内核**定义利息像素的每个邻居如何影响其输出值。 特别是,内核是$1$在它的中央和$0$其它任何地方都定义一个不起作用的过滤器。

通过改变内核,可以创建多种滤波器. 其中一些人**众所周知**比如说**索贝尔**上图中过滤器或**高斯模糊度**过滤器。 还有一些是**算法发现**,就像层[革命神经网络](https://en.wikipedia.org/wiki/Convolutional_neural_network).

索贝尔过滤器
------------

我们可以从上面图中的索贝尔过滤器开始。 索贝尔过滤器的意思是**检测边缘**,或者垂直的,或者水平的,取决于我们如何引导内核。 对于垂直边缘,内核是:

$$
K = (单位:千美元)
左转[开始( B){数组}{页:1}
-1 & 0 & +1 \\
-2 & 0 & +2 \\
-1 & 0 & +1 \\
结束( E){数组}对]
$$

让我们先写下阴影,然后连接点:

```rust
@ group( 0) @ 绑定( 0) var 输入_第2款d项<f32>;
@ group( 0) @ 绑定(1) var 输出Texture:纹理_存储_第2款d项<rgba8unorm,write>;

@ compute @ workgroup_大小( 8, 8)
fn 计算 SobelX (@ buildingin (global) )_援引_编号) 编号:vec3<u32>) {
让颜色=abs(
          1 *纹理LOAD( 输入图解, vec2)<u32>(d.x-1, id.y-1), 0. rgb.
        + 2 *纹理LOAD( 输入图解, vec2)<u32>(d.x - 1, id.y + 0). rgb.
        + 1 *纹理LOAD( 输入图解, vec2)<u32>(id.x - 1, id.y + 1), 0 尔格b.
        - 1 *纹理LOAD( 输入图解, vec2)<u32>(d.x + 1, id.y - 1), 0 rgb.
        - 2 *纹理LOAD( 输入图解, vec2)<u32>(d.x + 1, id.y + 0). rgb.
        - 1 *纹理LOAD( 输入图解, vec2)<u32>(id.x + 1, id.y + 1), 0. rgb.
    );
纹理Store( 输出图解, id. xy, vec4)<f32>(颜色,1.0);
}
```

很简单吧? 我们只需要输入和输出纹理。**我把它当作练习**因为它非常相似 设置第一部分[映射生成](mipmap-generation.md)章,但输出是不同纹理的视图,而不是同一纹理的不同MIP级别。

````{note}
别忘了改变计算管道的入口:

```C++
computePipelineDesc.compute.entryPoint = "computeSobelX";
```
````

打开[此输入图像](../../../../images/pexels-petr-ganaj-4121222.jpg),应当获得以下结果:

```{figure} /images/convolution/sobelX.jpg
:align: center
:class: with-shadow
我们垂直索贝尔过滤器的结果
```

```{note}
将这个过滤器独立应用到不同的MIP级别,在不同尺度上检测边缘,这很有趣.
```

图形界面
---

```{admonition} Optional section
如果您只对编译过滤器本身感兴趣,且制作非交互命令行工具是好的,可以跳过此段.
```

###设置

为了快速试验我们的滤波器,我们重新使用来自[简单的图形界面](../../basic-3d-rendering/some-interaction/simple-gui.md)章节。

简言之,我们必须:

- 增加一个`glfw`, `glfw3webgpu`和`imgui`库作为依赖性(不要忘记在`imgui`目录).
- 加一个`initWindow`, `initSwapChain`和`initGui`输入阶梯(和匹配的轮廓阶梯)。
- 添加一个主应用程序循环。

我还加了一个`m_shouldCompute`布尔以指示主循环仅在需要时(即输入参数发生变化时)调用计算机:

```C++
// Main frame
while (app.isRunning()) {
	app.onFrame();

	if (app.shouldCompute()) {
		app.onCompute();
	}
}
```

###显示纹理

改为**避免建造我们自己的管道**,我们可以使用 ImGui 绘制纹理视图,方法是手动添加指令到`ImDrawList`:

```C++
void Application::onGui(RenderPassEncoder renderPass) {
	// [...]

	ImDrawList* drawList = ImGui::GetBackgroundDrawList();
	// Draw a red rectangle
	drawList->AddRectFilled({ 0, 0 }, { 20, 20 }, ImColor(255, 0, 0));
	// Draw a texture view
	drawList->AddImage((ImTextureID)m_textureMipViews[0], { 20, 0 }, { 220, 200 });

	// [...]
}
```

```{figure} /images/drawList.png
:align: center
:class: with-shadow
红色的矩形和图像是由Imgui绘制的,我们不需要关心一个渲染管道!
```

最后,我们的功劳进入`onFrame`简单:

```C++
RenderPassEncoder renderPass = encoder.beginRenderPass(renderPassDesc);
onGui(renderPass);
renderPass.end();
```

```{note}
Imgui所说的`ImTextureID`取决于绘图后端。 对于基于 WebGPU 的后端,它必须对应有效的`TextureView`对象。 使用a`Texture`反而导致坠机
```

###制服

我们可以增加制服,以从UI驱动过滤器的行为.

注意所有 ImGui 函数返回布尔, 显示它们所代表的值是否已被修改, 我们可以用它来更新我们`m_shouldCompute`和**仅在需要时运行计算阴影**:

```C++
bool changed = false;
ImGui::Begin("Uniforms");
changed = ImGui::SliderFloat("Test", &m_uniforms.test, 0.0f, 1.0f) || changed;
ImGui::End();

m_shouldCompute = changed;
```

然后,您可以加入一个制服**内核**。小心点[校正规则](https://gpuweb.github.io/gpuweb/wgsl/#structure-member-layout)使用时`mat3x3`,因为需要在矩阵的列之间添加:

```rust
// WGSL 结构
钢制制服{
内核: mat3x3<f32>,
试验:f32,
}
```

```C++
// C++ struct with matching alignment
struct Uniforms {
	// The mat3x3 becomes a mat3x4 because vec3 columns are aligned as vec4
	mat3x4 kernel = mat3x4(0.0);

	float test = 0.5f;

	// Add padding at the end to round up to a multiple of 16 bytes
	float _pad[3];
};
```

```{note}
为了帮助你完成这个乏味的校对 我开始写了[一个小小的在线工具](https://eliemichel.github.io/WebGPU-AutoLayout)!
```

你现在可以动态地玩滤波器! 尝试不同轴上的 Sobel 过滤器 :

```{figure} /images/convolution/ui-uniforms.jpg
:align: center
:class: with-shadow
内核在UI中暴露为制服. 在这里我们显示一个水平的索贝尔过滤器(左:输入,右:输出).
```

模糊过滤器
------------

###框模糊

如果我们把内核的重量设置到$1/9$我们得到一个模糊的效果。

$$
K =\ frac 语句{1}{9}
左转[开始( B){数组}{页:1}
1 & 1 & 1 \\
1 & 1 & 1 \\
1 & 1 & 1 \\
结束( E){数组}对]
$$

```{figure} /images/convolution/box-blur-insets.jpg
:align: center
:class: with-shadow
页:1**框模糊**是通过与a**统一内核**.
```

```{note}
当内核全部非负核时,通常需要**常态**其权重,即以权重的总和除以. 您也许想要添加**用户界面中的复选框**这样做。
```

###分离性

如果我们想要一个**更加模糊**,我们可以简单地使用**大内核**然而,这很快会达到不合理的数字,每个新的Texel都会得到老的Texel。

幸运的是,盒子过滤器是**可分离**: 可以应用在**第一个轴**,然后在第二个轴上,使用大小的非平方内核$r \times 1$和$1 \times r$。这要求$2r$操作代替$r^2$巨大的储蓄! 比如说$r = 5$:

$$
K级_1 =\ frac{1}{5}
左转[开始( B){数组}{(c) 联合国}
1 \\
1 \\
1 \\
1 \\
1 \\
结束( E){数组}对]
宽度
文本( T){接下来}
宽度
K级_2 =\ frac{1}{5}
左转[开始( B){数组}{ccccc (英语).}
1 & 1 & 1 & 1 & 1 \\
结束( E){数组}对]
$$

盒子模糊是好的,但是如果你用大内核应用它 你会很快看到**为什么它叫"盒子"模糊**它把信号的尖端 变成非常显著的方形

###高斯模糊度

高斯模糊是最自然的模糊类型(a.k.a.低通道过滤器),它存在于许多不同的上下文中,你可能至少见过一次它的1D内核:

```{figure} /images/convolution/gaussians.png
:align: center
:class: with-shadow
标准偏差的单维高斯内核(正态分布)$\sigma = 1.0$(红色),$\sigma = 0.5$(绿色)和$\sigma = 0.25$(蓝).
```

理论上,高斯模糊度需要**无限内核**但邻居的影响力呈指数下降, 所以我们可以快速绕到0,从而**绑定大小**内核。 以下是一个2D高斯内核的简单近似:

$$
K =\ frac 语句{1}{16}
左转[开始( B){数组}{页:1}
1 & 2 & 1 \\
2 & 4 & 2 \\
1 & 2 & 1 \\
结束( E){数组}对]
$$

还有**喜讯**:高斯模糊度也是可以分离的! 这其实是唯一模糊的**既分离又循环对称** ([proof](https://www.sciencedirect.com/science/article/pii/089396599090151Z)).

要计算高斯模糊(分离)内核的系数,可以使用[此模糊系数生成器](https://lisyarus.github.io/blog/graphics/2023/02/24/blur-coefficients-generator.html)编号:`WEIGHTS`它创建的数组可直接用于`writeBuffer`把它传递给计算阴影器。

```{important}
为了保存一些纹理读数,这个生成器假设您正在使用线性**样本**从输入纹理中获取值。 换言之,使用`textureSample`而不是说`textureLoad`.
```

口腔过滤器
---------------------

口腔滤波器也基于**滑动窗口**,但内核乘以后,**而不是总结**不同的邻居,我们收集**上限**或**最低**(或者另一种非线性操作).

一个最大**扩展**明亮的地带和最低的反面**侵蚀**白色的像素。

内核一般由二进制面具定义,称为:**结构要素**的形态操作。 当结构元素是**矩形**,过滤器是**可分离**.

```{figure} /images/convolution/morpho.jpg
:align: center
:class: with-shadow
应用到 RGB 图像上的形态过滤器 。
```

```{note}
通常在黑白口罩上使用形态过滤器,例如,在轮廓检测中过滤不完美之处。
```

结论
----------

Corvolution filters是使用计算阴影器的好例子(尽管它们很容易通过绘制全屏幕三角形,然后使用碎片阴影器来模仿). 它们也是许多图像处理工具的构件.

*结果代码 :* [`step215`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step215)
