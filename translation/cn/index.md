WebGPU 教程
============

*For native graphics in C++.*

本文档将引导您使用 WebGPU 图形 API 从头开始在 C++ 中创建**原生的 3D 应用程序**，并且程序适用于 Windows、Linux 和 macOS。

`````{admonition} 快速开始! (点击我)
:class: foldable quickstart

*你希望理解你写的每一行 GPU 代码吗？*

````{admonition} 是的，**从头开始**编写 WebGPU 代码！!
:class: foldable yes

太好了！您可以直接前往[介绍](introduction.md)，然后按顺序**阅读所有章节**。
````

````{admonition} 不，我想**跳过初始的样板代码**部分.
:class: foldable no

这完全有道理, 你可直接从 [**入门**](getting-started/index.md)部分开始.

您可能希望查看每篇教程的开头和结尾部分 _**Resulting code**_ 链接, 例如:

```{image} /images/intro/resulting-code-light.png
:class: only-light with-shadow
```

```{image} /images/intro/resulting-code-dark.png
:class: only-dark with-shadow
```

*您是否同意使用浅层包装以便更容易阅读？

```{admonition} 是的，我更喜欢**C++ 风格**的代码。
:class: foldable yes

请使用 "**With webgpu.hpp**" 标签页。
```

```{admonition} 不，给我看**原始的 C WebGPU API**！
:class: foldable no

请使用 "**Vanilla webgpu.h**" 标签页。虽然纯 WebGPU 的 *Resulting code* 更新较慢，但本标签页中指南内的**所有代码块**都是**最新的**。
```

为了**构建这个基础代码**，请参考项目设置章节的[构建](getting-started/project-setup.md#building) 部分。您可以在 `cmake -B build` 命令行中添加 `-DWEBGPU_BACKEND=WGPU`（默认）或 `-DWEBGPU_BACKEND=DAWN`，以分别选择使用  [`wgpu-native`](https://github.com/gfx-rs/wgpu-native) 或 [Dawn](https://dawn.googlesource.com/dawn/) 作为后端。

*你希望基础代码做到什么程度？*

```{admonition} 一个简单的三角形
:class: foldable quickstart

请阅读 [你好三角形](basic-3d-rendering/hello-triangle.md) 章节.
```

```{admonition} 一个带有基本交互的 3D Mesh 查看器
:class: foldable quickstart

我建议从[光照控制](basic-3d-rendering/some-interaction/lighting-control.md) 章节的结尾开始阅读。
```

````

```{admonition} 我希望这些代码能在**Web上运行**。
:class: foldable warning

主体指南中缺少一些额外的内容，请参考[构建Web应用](appendices/building-for-the-web.md) 附录来**调整示例**，使其在Web上运行！
```

`````

目录
--------

```{toctree}
:titlesonly:

introduction
getting-started/index
```
