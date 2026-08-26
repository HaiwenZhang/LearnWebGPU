自定义扩展
=================

```{translation-warning} 译文可能已过时, /appendices/custom-extensions/index.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

在非 Web 环境中使用 WebGPU 时，没有理由受到 Web 要求的限制。本章概述了如何通过WebGPU的扩展机制添加对新设备功能的支持。

```{note}
我在这里不使用 [`webgpu.hpp`](https://github.com/eliemichel/WebGPU-Cpp) 帮助程序，因为扩展文件必须定义 C API。然后，可以使用 [`generator.py`](https://github.com/eliemichel/WebGPU-Cpp#custom-generation) 轻松地将 `webgpu.hpp` 包装器推广到您自己的自定义扩展。
```

内容
--------

```{toctree}
:titlesonly:

mechanism
with-wgpu-native
with-dawn
```
