变形(<span class="bullet">🟠</span>WIP)
===========

```{translation-warning} 译文可能已过时, /basic-compute/procedural-geometry/deformation.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码 :* [`step240`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step240)

**程序几何**游戏在3D建模工具中扮演着非常重要的角色. 它们通常采取一种形式:**后遗症堆栈**和(或)**节点图**程序操作相互连接。 例如Blender分别调用这些*修改器*和*几何节点*:

```{figure} /images/procgen/geometry-nodes.jpg
:align: center
:class: with-shadow
修饰器和几何节点是Blender中程序(a.k.a.无损)网状效应的两个实例.
```

动机
----------

这些效果是强大的,但只能用于3D模型的创建,**未在运行时间**在游戏或应用程序中。 因此,我们需要为自己规划我们打算进行直播的程序效果。

```{seealso}
无法交换程序效应(也称"无损效应"),是一段时间以来我一直试图处理的问题。[打开Mfx](https://openmfx.org/)这个"标准"还没有被任何人真正采纳,但我仍然认为我们需要这样一个API,并且如果你有兴趣,会很乐意讨论这个问题!
```

我们可以复制对CPU的网格效应,但既然这个指南是关于GPU编程的,**试着用阴影写出来**并非所有效果都适合GPU编程,但很多效果都是. 此外,有些操作比其他操作更容易在GPU上平行。

那个**最简单的**程序几何效应包括:**变形已存在的网格**,通过移动现有的顶点, 不影响连接。

首先,如果你需要的话**移动顶点**你也许可以做到**在顶点阴影中**但可能有很多原因,你应该使用一个计算阴影:

- 如果变形过程是**重任务**你不希望在每一帧都运行。
- 如果你想的话**回读变形网格**给CPU
- 如果你需要的话**接入连接**信息,如处理网格时顶点的邻居的顶点/面.
- 如果你想的话**共享顶点间的信息**.

实例
------------

我们的第一个例子是"平面化"效应,它以四层网格作为输入,并尝试通过迭代移动的顶点使其面孔尽可能平面.

```{figure} /images/procgen/quad-input.png
:align: center
我们的输入网是完全用四面体制成的,但它们相当扭曲。
```

输入文件 :[`quad-input.obj`](../../../../data/procgen/quad-input.obj)

开始

*结果代码 :* [`step240`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step240)


