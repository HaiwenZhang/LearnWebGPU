调试（<span class="bullet">🟠</span>WIP）
=========

```{translation-warning} 译文可能已过时, /appendices/debugging.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

尽早熟悉一些基本的调试技巧非常重要。

图形调试器
-----------------

在图形调试方面存在一个特殊的挑战：通常的调试工具**只能看到CPU上发生的情况**，因为那是它们所在的位置。

因此，我们需要一个专用工具，称为**图形调试器**。该调试器仍然在 CPU 上运行，但它被**注入到您的应用程序和 GPU 命令队列之间**，并监视通过的所有内容，以便它可以**重播**您的所有命令。如果你愿意的话，有点像[中间人](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)。

图形调试器通过使我们能够检查 GPU 内存和状态，对调试着色器、缓冲区、纹理等确实有很大帮助。它还可以给出性能提示。

我使用两个图形调试器：

- [RenderDoc](https://renderdoc.org/) 非常适合**调试**，它得到积极且非常准确的维护。

- [Nsight Graphics](https://developer.nvidia.com/nsight-graphics) 也有一个调试器，但我主要将它用于其**帧分析器**，它提供有关各种 GPU 单元所花费的时间的详细见解。

调试纹理
------------------

第一个调试示例的灵感来自读者[^iazalong]在阅读[纹理加载](../basic-3d-rendering/texturing/loading-from-file.md)章节时遇到的一个真实错误。

[^iazalong]: Thanks *Iazalong*!

显然，纹理出了问题：

```{figure} /images/debug-problem.png
:align: center
:class: with-shadow
纹理有问题。
```

在接触代码之前的第一反应是用 RenderDoc 更好地**诊断**问题。在 RenderDoc 的“启动应用程序”选项卡中，浏览到“可执行路径”中的应用程序，如果从相对路径加载资源，则可能会调整“工作目录”，并且您应该会看到程序在顶部以“调试覆盖”开始：

```{figure} /images/debug-rd.png
:align: center
:class: with-shadow
该应用程序从 RenderDoc 启动。
```

此覆盖图确认 RenderDoc 能够将自身注入到您的程序和 GPU 之间，并且正如其建议的那样，按 F12 捕获帧。这会记录传输到 GPU 的所有内容并创建 **Capture**。

```{note}
然后您可以关闭程序，捕获的内容包含重播您的帧所需的一切。
```

双击捕获将其打开。 *事件浏览器* 显示调试器拦截的所有事件（即命令），单击其中一个即可进入当时 GPU 的状态。例如找到主绘制调用：

```{figure} /images/debug-event.png
:align: center
:class: with-shadow
RenderDoc 中的事件列表。
```

您可以使用*纹理查看器*的*输出*选项卡来帮助您浏览事件。

```{note}
捕获的事件与 WebGPU 命令并不完全匹配，因为 RenderDoc 捕获隐藏在其后面的低级 API。在我的示例中，WebGPU 在 Vulkan 后端上运行，因此我们看到的是 Vulkan 事件。在不同的平台上，您可能会看到不同的 API，例如 Metal 或 DirectX，但无论如何您都应该认识整体结构。
```

由于输入纹理似乎有问题，让我们转到 *Inputs* 选项卡并查看它（确保您在事件浏览器中处于正确的绘制调用位置）：

```{figure} /images/debug-mipmap.png
:align: center
:class: with-shadow
在 RenderDoc 中检查的输入纹理。
```

到目前为止一切都很好，那么有什么问题吗？嗯，现在让我们检查一下不同的 mip 级别：

```{figure} /images/debug-wrong-mipmaps.png
:align: center
:class: with-shadow
在 RenderDoc 中检查的 mip 级别。
```

在这里，mip 级别未正确构建！现在我们知道应该将调试工作重点放在构建和上传 mip 级别的 `loadTexture` 部分。

```{note}
事实证明，问题在于 `writeTexture` 调用指向所有 mip 级别的原始像素缓冲区。
```

调试几何体
------------------

RenderDoc 的另一个典型用例是检查几何体。更一般地说，“管道状态”选项卡提供了有关绘制调用的宝贵信息：

```{figure} /images/debug-pipeline.png
:align: center
:class: with-shadow
RenderDoc 中的图形管道选项卡。
```

我们可以在这里看到整个**渲染管道**，及其固定功能和可编程阶段。在“顶点输入”阶段，有一个非常有洞察力的“网格视图”：

```{figure} /images/debug-mesh.png
:align: center
:class: with-shadow
RenderDoc 中的网格视图。
```

您可以在表格和 3D 查看器中看到组装的输入几何体和后顶点着色器几何体。

在浏览器中调试
--------------------

如果您使用 emscripten 将项目编译为网页，则应该查看 [`webgpu-devtools` Chrome 扩展](https://chrome.google.com/webstore/detail/webgpu-devtools/ckabpgjkjmbkfmichbbgcgbelkbbpopi)，它提供了高级的 WebGPU 特定调试工具！
