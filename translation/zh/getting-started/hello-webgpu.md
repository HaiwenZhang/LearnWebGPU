你好,WebGPU,你好吗?<span class="bullet">🟢</span>
============

```{translation-warning} 译文可能已过时, /getting-started/hello-webgpu.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/001 - Hello WebGPU
:parent: 000 - Project setup
:fetch-files: ../data/webgpu-distribution-v0.2.0-beta2.zip
```

*结果代码 :* [`step001`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step001)

WebGPU 是一个*硬件接口*(RHI),意思是它是一个编程库,意在提供一个**统一接口**用于多个基础图形硬件和操作系统设置。

对于您的 C++ 代码, WebGPU 只不过是**单个头文件**,其中列出了所有可用的程序和数据结构:[`webgpu.h`](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h).

然而,在构建程序时,您的编译器必须最终知道(最后)*链接*步骤)**在哪里寻找**实际履行这些职能。 与本地API相反,WebGPU的实现不是由驱动程序提供的,所以我们必须明确提供.

```{figure} /images/rhi-vs-opengl.png
:align: center
页:1*硬件接口*就像WebGPU一样**司机不直接提供**:我们需要链接到一个在系统所支持的低层之上执行API的库.
```

正在安装 WebGPU
-----------------

WebGPU本土头头主要有两个执行:

 - [wgpu-内在](https://github.com/gfx-rs/wgpu-native),将本地界面暴露于[`wgpu`](https://github.com/gfx-rs/wgpu)为Firefox开发的 Rust 库.
- 谷歌的音乐[黎明](https://dawn.googlesource.com/dawn),为Chrome开发.

```{figure} /images/different-backend.png
:align: center
(至少)WebGPU有两款执行,为两个主要的Web引擎开发.
```

这两次执行仍然**一些差异**,但这些会随着WebGPU规格稳定而消失. 我试图写这个指南,以至于它**两人的工作**他们。

为了方便将其中任何一个纳入CMake项目,我分享一个[WebGPU 分发](https://github.com/eliemichel/WebGPU-distribution)允许您选择以下选项之一的仓库 :

`````{admonition} Too many options? (Click Me)
:class: foldable quickstart

*您倾向于快速构建而不是细节错误信息吗 ?*

````{admonition} Yes please, **fast build** and **no need** for an Internet connection the first time I build
:class: foldable yes

继续[**备选案文A**](#选项- a- the- lightness - of- wgpu-native) (wgpu-native) !
````

````{admonition} No, I'd rather have **detailed error messages**.
:class: foldable no

继续[**备选案文B**](#选项- b- the- complete- of- dawn) (Dawn)!

````

```{admonition} I don't want to chose.
:class: foldable warning

继续[**备选案文C**](#选项- c- 两者均灵活, 这使得您可以在任何时候从一个后端切换到另一个后端 !
```

`````

###备选案文A:wgpu-native的轻度

从`wgpu-native`我们无法从零开始建造,

```{important}
**妇女:**使用“任何平台”的链接, 而不是特定平台的链接, 我还没有自动化他们的世代,所以他们通常在主平台后面。
```

 - [任意平台的 wgpu 本地化](https://github.com/eliemichel/WebGPU-distribution/archive/refs/tags/wgpu-v24.0.0.2.zip)(因为它是所有可能的平台的合并)
 - [用于 Linux 的 wgpu 本地化](#)
 - [Wgpu 窗口本地化](#)
 - [用于 MacOS 的 wgpu 本地化](#)

```{note}
预编的二进制由`wgpu-native`这样你就可以相信他们 我分发的唯一内容是`CMakeLists.txt`这使得它容易整合。
```

**专业**
- 这是最轻的建筑。

**计数**
- 你不能从源头建起来
 - `wgpu-native`没有像Dawn那样提供信息丰富的调试信息。

###备选案文B:黎明的舒适

Dawn给出了更好的错误信息,由于它用C++写成,我们可以从源头构建它,从而在崩溃时更深入地检查堆栈跟踪:

 - [任何平台的黎明](https://github.com/eliemichel/WebGPU-distribution/archive/refs/tags/dawn-6536.zip)

```{note}
我在这里提供的基于 Dawn 的分发从它的原始寄存器中获取 Dawn 的源代码,但以尽可能浅的方式获取,并预设一些选项以避免构建我们不使用的部分.
```

**专业**

- Dawn更适合开发,因为它给出了更详细的错误信息.
- 一般来说,它领先于`wgpu-native`执行进展情况报告(但`wgpu-native`最终会赶上的).

**计数**
- 虽然我减少了对额外依赖的需要,你仍然需要[安装 Python](https://www.python.org/)和[gi](https://git-scm.com/download).
- 发行会获取 Dawn 的源代码和它的依赖性,所以你第一次建立的时候需要**互联网连接**.
- 最初的建造需要更长的时间,总体占用更多的磁盘空间。

````{note}
在 Linux 上检查[Dawn的建设文件](https://dawn.googlesource.com/dawn/+/HEAD/docs/building.md)用于安装软件包列表。 截止2024年4月7日,名单如下(为乌本图):

```bash
sudo apt-get install libxrandr-dev libxinerama-dev libxcursor-dev mesa-common-dev libx11-xcb-dev pkg-config nodejs npm
```
````

###备选案文C:两者的灵活性

在这个选项中, 我们只包含几个 CMake 文件 在我们的项目中, 然后动态获取`wgpu-native`或 Dawn 取决于配置选项 :

```
cmake -B 构建 -DWEBGPU_后端=工作组
#或
cmake -B 构建 -DWEBGPU_(原始内容存档于2018-09-21) (中文(简体) ). Refend=Down
```

```{note}
那个**附带代码**使用本选项C。
```

这是由`main`我的发行仓库分支 :

 - [WebGPU 任意发行](https://github.com/eliemichel/WebGPU-distribution/archive/refs/tags/main-v0.2.0.zip)

```{tip}
该寄存器的 README 有如何使用 FackContent 添加到您的工程中的指示_宣说. 如果你这样做,你可能会使用 更新版本的黎明或wgpu -native 比这个写反对。 因此,本书中的例子可能不会为您编译. 关于如何下载这本书的版本,见下文。
```

**专业**
- 你可以吃两个`build`同时,一个使用黎明,一个使用`wgpu-native`

**计数**
- 这是一个“元分配”,它获取一个在配置时想要的(即调用时)`cmake`所以,你需要一个**互联网连接**和**gi**那时

当然,取决于你的选择 利弊*备选案文A*和*备选案文B*适用。

###一体化

无论你选择哪一种分布,整合是相同的:

1. 下载您选择的拉链。
2. 将它解开作为项目的基础,应该有一个`webgpu/`包含一个`CMakeLists.txt`文档和其他一些文件(.dll或.so)。
3. 添加`add_subdirectory(webgpu)`在你身边`CMakeLists.txt`.

```{lit} CMake, Dependency subdirectories (insert in {{Define app target}} before "add_executable")
# Include webgpu directory, to define the 'webgpu' target
add_subdirectory(webgpu)
```

```{important}
这里的“ webgpu” 名称指定了 Webgpu 所在的目录, 因此应该有一个文件`webgpu/CMakeLists.txt`否则,这意味着`webgpu.zip`未在正确的目录中解压缩;您可以移动该目录或修改该目录。`add_subdirectory`指令。
```

4. 增加:`webgpu`目标作为我们的应用程序的依赖性,使用`target_link_libraries`命令( 之后)`add_executable(App main.cpp)`).

```{lit} CMake, Link libraries (insert in {{Define app target}} after "add_executable")
# Add the 'webgpu' target as a dependency of our App
target_link_libraries(App PRIVATE webgpu)
```

```{tip}
这一次,“webgpu”这个名字是*目标*定义`webgpu/CMakeLists.txt`通过呼叫`add_library(webgpu ...)`,与目录名称无关。
```

使用预编译二进制时的附加步骤:调用函数`target_copy_webgpu_binaries(App)`结束时`CMakeLists.txt`,这可以确保您运行时所依赖的 .dll/. so 文件被复制到旁边。 当您分发您的应用程序时, 请确保同时分发此动态库文件 。

```{lit} CMake, Link libraries (append)
# The application's binary must find wgpu.dll or libwgpu.so at runtime,
# so we automatically copy it (it's called WGPU_RUNTIME_LIB in general)
# next to the binary.
target_copy_webgpu_binaries(App)
```

```{note}
就Dawn而言,没有预编的二进制可以复制,但我定义了`target_copy_webgpu_binaries`无论如何,函数(它什么都不做),这样您就可以真正使用同样的 CMakeLists 和两个分布式。
```

测试安装
------------------------

为了测试执行,我们只是创建 WebGPU**实例**,即相当于`navigator.gpu`我们可以进入JavaScript。 然后我们检查并摧毁它。

```{important}
确保包含`<webgpu/webgpu.h>`在使用任何 WebGPU 函数或类型之前 !
```

```{lit} C++, Includes
// Includes
#include <webgpu/webgpu.h>
#include <iostream>
```

```{lit} C++, file: main.cpp
{{Includes}}

int main (int, char**) {
    {{Create WebGPU instance}}

    {{Check WebGPU instance}}

    {{Destroy WebGPU instance}}

    return 0;
}
```

###描述和创建

实例是使用`wgpuCreateInstance`函数。 和所有WebGPU的功能一样**创建**以一个实体为论据a**描述符**,用于指定如何设置此对象的选项。

```{lit} C++, Create WebGPU instance
// We create a descriptor
WGPUInstanceDescriptor desc = {};
desc.nextInChain = nullptr;

// We create the instance using this descriptor
WGPUInstance instance = wgpuCreateInstance(&desc);
```

```{note}
描述器是一种方法**装入许多函数参数**因为有些描述符真的有很多领域 也可以用来写出注意充电参数的效用函数,以方便程序的架构.
```

我们遇见另一个WebGPU**平话**输入`WGPUInstanceDescriptor`结构:描述符的第一个字段总是一个指针,叫做`nextInChain`。这是 API 启用**自定义扩展**,或返回多个数据条目。 在很多情况下,我们规定`nullptr`.


###检查

用 WebGPU 创建的 WebGPU 实体`wgpuCreateSomething`函数在技术上**只是个指针**。它是一个盲柄,可以识别实际物体,它生活在后端,我们从不需要直接进入。

为了检查一个对象是否有效,我们可以把它和`nullptr`,或使用布尔运算符:

```{lit} C++, Check WebGPU instance
// We can check whether there is actually an instance created
if (!instance) {
    std::cerr << "Could not initialize WebGPU!" << std::endl;
    return 1;
}

// Display the object (WGPUInstance is a simple pointer, it may be
// copied around without worrying about its size).
std::cout << "WGPU instance: " << instance << std::endl;
```

这应该显示一些类似的东西`WGPU instance: 000001C0D2637720`启动时。

###销毁和寿命管理

所有实体可以**创建**使用 WebGPU 最终必须是**已发行**。创建一个对象的程序看起来总是`wgpuCreateSomething`,其等效释放它就是`wgpuSomethingRelease`.

请注意,每个对象在内部都持有一个参考计数器,如果您的代码中没有其他部分仍然引用它,则只释放相关的内存(即计数器降为0):

```C++
WGPUSomething sth = wgpuCreateSomething(/* descriptor */);

// This means "increase the ref counter of the object sth by 1"
wgpuSomethingReference(sth);
// Now the reference is 2 (it is set to 1 at creation)

// This means "decrease the ref counter of the object sth by 1
// and if it gets down to 0 then destroy the object"
wgpuSomethingRelease(sth);
// Now the reference is back to 1, the object can still be used

// Release again
wgpuSomethingRelease(sth);
// Now the reference is down to 0, the object is destroyed and
// should no longer be used!
```

我们尤其需要发布全球WebGPU实例:

```{lit} C++, Destroy WebGPU instance
// We clean up the WebGPU instance
wgpuInstanceRelease(instance);
```

###具体实施行为

为了处理执行之间的微小差异,我提供的分布图还定义了下列预处理器变量:

```C++
// If using Dawn
#define WEBGPU_BACKEND_DAWN

// If using wgpu-native
#define WEBGPU_BACKEND_WGPU

// If using emscripten
#define WEBGPU_BACKEND_EMSCRIPTEN
```

###网站大楼

上面列出的WebGPU发行版本很容易兼容[标记](https://emscripten.org/docs/getting_started/downloads.html)如果您在构建您的网络应用程序时有困难, 您可以咨询[专门附录](../appendices/building-for-the-web.md).

由于我们将不时为网络建设增加一些特定的选项,我们可以在网络建设结束时增加一节。`CMakeLists.txt`:

```{lit} CMake, file: CMakeLists.txt (append)
# Options that are specific to Emscripten
if (EMSCRIPTEN)
    {{Emscripten-specific options}}
endif()
```

目前我们只更改输出扩展,使其是一个HTML网页(而不是WebAssembly模块或JavaScript库):

```{lit} CMake, Emscripten-specific options
# Generate a full web page rather than a simple WebAssembly module
set_target_properties(App PROPERTIES SUFFIX ".html")
```

由于某种原因,例描述符**必须是无效的**(意思是"使用默认")在使用Emscripten时,我们就可以使用我们的`WEBGPU_BACKEND_EMSCRIPTEN`宏 :

```{lit} C++, Create WebGPU instance (replace)
// We create a descriptor
WGPUInstanceDescriptor desc = {};
desc.nextInChain = nullptr;

// We create the instance using this descriptor
#ifdef WEBGPU_BACKEND_EMSCRIPTEN
WGPUInstance instance = wgpuCreateInstance(nullptr);
#else //  WEBGPU_BACKEND_EMSCRIPTEN
WGPUInstance instance = wgpuCreateInstance(&desc);
#endif //  WEBGPU_BACKEND_EMSCRIPTEN
```

结论
----------

在本章中,我们建立了WebGPU,并得知有**多个后端**可用。 我们还看到了基本规范**对象创建和销毁**这将在 WebGPU API 中全部使用 !

*结果代码 :* [`step001`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step001)
