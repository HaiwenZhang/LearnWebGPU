物理材料(<span class="bullet">🟠</span>WIP)
==========================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/lighting-and-material/pbr.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step125`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step125)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step125-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step125-vanilla)
````

开始

我们迄今看到的扩散+光谱材料模型对于教育目的的一步很重要,但它缺乏适当的物理基础。

材料有两大类,它们对应两种非常不同的方式,光线必须在表面反弹:

 - **金属**物体进行电磁波,因此基本上是**镜像**:在显微光下,光线真的在它们上反弹,沿着反射方向离去。

 - **电源**对象与字面相反:**匀支**电磁波的能量在表面下方的薄层中, 只在四面八方重新熔化, 使其大部分扩散。 其阴影的光谱部分来自由于斯内尔定律而从未被吸收的光的一部分.

从中得出的一些拇指规则:
- 金属表面没有任何扩散照明。
- 电极表面的光谱亮度不能有色。
- 双电路表面在放牧角度只有光谱亮度。
- 表面是金属的或二电的,没有中间。

因此,我们通常使用下列参数来描述一种材料:

 - `metallic`: 1.0 金属表面, 0.0 双电表面。
 - `baseColor`:对于一个二电物体来说,这就是扩散的颜色;对于一个金属物体来说,这就是谱色.
 - `reflectance`:仅仅对于二电,我们需要指定(无色)光谱贡献的强度.

不需要两种不同的颜色图,因为一个表面从不同时是金属和二电的.

我们还有一些几何信息:

 - `normal`当然,你记得前一章 我们指出当地的正常。
 - `roughness`:这表示微常数在平均数周围的标准偏差(由正常地图给出). 这在0.0到1.0的尺度上比之前使用的谱子的"硬度"更能直观地操控.

正常和粗糙度信息是"自然分布函数"(NDF)的统计表示,即概率法说明在texel内微镜面如何定向.

测试台
----------

我们将利用一个简单的球体来集中我们的测试。

```{figure} /images/pbr-test-roughness.png
:align: center
:class: with-shadow
同一个带粗糙的电线物体$0.41$改为$0.79$.
```

```{figure} /images/pbr-test-roughness-metallic.png
:align: center
:class: with-shadow
同样的金属物体 粗糙的从$0.15$改为$0.83$.
```

```{figure} /images/pbr-test-roughness-metallic-boat.png
:align: center
:class: with-shadow
第1行:介电材料从粗糙到光泽不等。 第2行:从二电到金属不等(但不应在两者之间使用)。 第3行:金属材料上从粗糙到光泽不等。
```

结论
----------

我建议你读一下[设计文件](https://google.github.io/filament/Filament.html)提供非常有洞察力和随时可用的信息。

````{tab} With webgpu.hpp
*结果代码 :* [`step125`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step125)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step125-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step125-vanilla)
````
