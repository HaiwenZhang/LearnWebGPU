介绍
============

什么是图形API?
-----------------------

一台个人电脑或者智能手机通常包含两个计算单元：**CPU**（*中央处理器*）和**GPU**（*图形处理器*）。对于大多数编程语言来说，在编写应用程序时，**主要就是为了CPU编写指令**。

```{figure} /images/architecture-notes.png
:align: center
CPU 和 GPU 是两种不同的处理器。我们编程时，让 CPU 通过图形 API 和驱动程序指示 GPU 要做什么。
```

如果希望应用程序在 GPU 上执行指令（例如渲染 3D 图像），CPU 代码必须向 GPU 的驱动程序**发送指令**。图形 API 是 CPU 代码与 GPU 对话所使用的编程接口。

有许多这样的图形 API，例如你可能听说过 OpenGL、DirectX、Vulkan 或 Metal。

```{tip}
理论上，任何人都可以发明自己的图形 API。每个 GPU 厂商都有自己的低级协议，用于驱动程序与硬件进行对话，常见的图形 API 常常是在这些低级协议之上实现的（通常随驱动程序一同提供）。
```

在这份文档中，我们学习一个名为 [WebGPU](https://www.w3.org/TR/webgpu/) 的图形 API。这个 API 的设计目的是为应用程序提供**统一访问** GPU 的方式，无论应用程序运行在哪种 GPU 厂商或操作系统上。

```{figure} /images/rhi.png
:align: center
:class: with-shadow
WebGPU 是建立在驱动程序/操作系统提供的各种 API 之上的渲染硬件接口。这种重复的开发工作由 web 浏览器完成，并通过它们提供的 webgpu.h 头文件向我们提供。
```

为什么是 WebGPU?
-----------

> 🤔 确实，为什么我会使用一个**Web API**来开发**桌面应用程序**呢？

很高兴你问了，简短的答案是：

- 合理的抽象层级
- 良好的性能
- 跨平台支持
- 足够的标准化
- 未来性

事实上，它是**唯一一个**同时具备所有这些特性的图形 API！

是的，WebGPU API 主要是**为Web设计**的，作为 JavaScript 和 GPU 之间的接口。这**并非是一个缺点**，因为如今网页性能的要求实际上与原生应用程序的要求相同。你可以阅读更多关于[我为什么认为在2023年学习WebGPU是最佳选择](appendices/teaching-native-graphics-in-2023.md)的理由。

```{note}
在为 Web 设计 API 时，两个关键限制是**可移植性**和**隐私性**。在这里，我们**受益于**为实现可移植性而开发的工作，而由于隐私考虑而导致的 API 的限制，在将 WebGPU 作为原生 API 使用时可以**禁用**。
```

那为什么要用 C++ 呢?
-------------

我们不应该使用 **JavaScript** 吗？毕竟它是 WebGPU 的初始目标。或者使用 **C**，因为它是我们将要使用的 `webgpu.h` 头文件的语言？又或者使用 **Rust**，因为其中一个 WebGPU 后端就是用这种语言编写的？所有这些语言都可以有效地使用 WebGPU，但我选择了 C++，因为：

- C++ 仍然是高性能图形应用程序（如视频游戏、渲染引擎、建模工具等）的主要语言。
- C++ 提供的抽象层级和控制能力非常适合与图形 API 进行交互。
- 图形编程是学习 C++ 的绝佳机会。在开始阶段，我只假设读者对这门语言有非常基础的了解。

```{seealso}
对于 Rust 版本的类似文档，我建议您查看 Sotrh 的 [Learn WGPU](https://sotrh.github.io/learn-wgpu)。
```

如何使用这篇文档
------------------------------

### 阅读指南

本文档的前两部分被设计为顺序阅读，如同完整的讲座，但其不同页面也可以用作特定主题的提醒。

[入门部分](getting-started/index.md) 处理初始化 WebGPU 和窗口管理（使用 GLFW）所需的样板代码，并介绍了 API 的关键概念和惯用法。在这一部分中，我们操作原始的 C API，并最终介绍了在本文档其余部分中使用的 C++ 封装器。

您可以直接跳到第二部分[基本3D渲染](basic-3d-rendering/index.md)，并使用第1部分生成的样板代码作为起步工具包。你可以随时回来详细了解入门部分的细节。

如今，渲染远非 GPU 的唯一用途；第三部分介绍了[基本计算](basic-compute/index.md)，即 WebGPU 的非渲染应用。

第四部分[高级技术](advanced-techniques/index.md) 着重介绍各种计算机图形技术的焦点，可以更独立地阅读各章节。

### 文学编程

```{warning}
这个指南处于早期阶段，目前只有前几章可用。
```

这个指南遵循**文学编程**的原则：
您阅读的文档已注释，以便可以**自动将代码块组合成**完全可运行的代码。
这是一种方法，可以确保指南包含了您**复现结果**所需的所有代码内容。

在支持的章节的右侧侧边栏，您可以启用/禁用显示这些信息：

```{image} /images/literate-light.png
:align: center
:class: only-light
```

```{image} /images/literate-dark.png
:align: center
:class: only-dark
```

默认情况下，所有内容都被关闭，以避免视觉混乱，但如果您觉得不确定特定代码片段应该放在哪里，您可以将它们打开。

### 如何给本教程贡献

如果您发现任何拼写错误或更重要的问题，请随时点击每页顶部的编辑按钮进行修正！

```{image} images/edit-light.png
:alt: Use the edit button present on top of each page!
:class: only-light
```

```{image} images/edit-dark.png
:alt: Use the edit button present on top of each page!
:class: only-dark
```

更广泛地说，您可以通过仓库的[问题页面](https://github.com/eliemichel/LearnWebGPU/issues)讨论任何技术或组织选择。
欢迎任何建设性和/或善意的反馈！

### 进行中的工作

这个指南仍在建设中，而 WebGPU API 本身也在发展中。我尽量紧密地跟踪变化，但在 API 稳定之前，这不可避免地会导致轻微的不一致性。

请始终注意页面和相关代码的最后修改日期（使用 [git](https://github.com/eliemichel/LearnWebGPU)）。它们可能不会完全同步；通常我会先更新代码，然后再更新指南内容。

