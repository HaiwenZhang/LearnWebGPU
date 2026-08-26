项目设置<span class="bullet">🟢</span>
=============

```{translation-warning} 译文可能已过时, /getting-started/project-setup.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/000 - Project setup
```

*结果代码 :* [`step000`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step000)

在我们运行的例子,我们使用[计算](https://cmake.org/)组织编纂守则。 这是处理跨平台建筑的一个非常标准的方法,我们遵循的是[现代制作](https://cliutils.gitlab.io/modern-cmake/).

所需资源
------------

我们需要的只是CMake和一个C++编译器,下面每个OS都有详细的指令.

```{hint}
安装后,您可以输入`which cmake` (linux/macOS)或`where cmake`(Windows)来查看您的命令行是否找到通往`cmake`命令。 如果不是,请确保你`PATH`环境变量包含安装 CMake 的目录。
```

###链接

如果您处于Ubuntu/Debian分布下,请安装以下软件包:

```bash
sudo apt install cmake build-essential
```

其他发行版本有等效的软件包,确保您有命令`cmake`, `make`和`g++`工作时

###窗口

下载并安装 CMake 从[下载页面](https://cmake.org/download/)。您可以使用其中之一[视觉工作室](https://visualstudio.microsoft.com/downloads/)或[民运](https://www.mingw-w64.org/)作为编译工具箱。

###麦克OS

您可以使用`brew install cmake`,以及[XCode 代码](https://developer.apple.com/xcode/)以建设项目。

最小项目
---------------

最低限度的项目包括:`main.cpp`源文件,以及 a`CMakeLists.txt`构建文件。

让我们从经典的问候世界开始`main.cpp`:

```{lit} C++, file: main.cpp
#include <iostream>

int main (int, char**) {
	std::cout << "Hello, world!" << std::endl;
	return 0;
}
```

内`CMakeLists.txt`,我们指定我们要创建一个*目标*类型*可执行文件*,称为“App”(这也将成为可执行文件的名称),其源代码是`main.cpp`:

```{lit} CMake, Define app target
add_executable(App main.cpp)
```

CMake 也期望在开始时`CMakeLists.txt`要知道 CMake 文件的版本是为( 最少支持的... 您的版本) 撰写的, 以及有关项目的一些信息 :

```{lit} CMake, file: CMakeLists.txt
cmake_minimum_required(VERSION 3.0...3.25)
project(
	LearnWebGPU # name of the project, which will also be the name of the visual studio solution if you use it
	VERSION 0.1.0 # any version number
	LANGUAGES CXX C # programming languages used by the project
)

{{Define app target}}

{{Recommended extras}}
```

Building
--------

下面开始构建项目。

我们现在准备建立我们的最低项目。 打开终端并转到您拥有的目录`CMakeLists.txt`和`main.cpp`文件 :

```bash
cd your/project/directory
```

```{hint}
从显示项目目录的 Windows 探索器窗口,按 Ctrl+L,然后键入`cmd`并击回。 这将在当前目录中打开一个终端 。
```

让我们现在要求CMake为我们的项目创建构建文件. 我们要求它从源代码中分离构建文件,把它们放进*构建/*带有目录`-B build`选项。 这是非常推荐的,为了能够方便地区分这些生成的文件与我们手动撰写的文件(a.k.a.源文件):

```bash
cmake . -B build
```

创建任意一个的构建文件`make`,视系统而定的Visual Studio或XCode(您可以使用`-G`选项以强制特定构建系统,参见`cmake -h`以获得更多信息。 最终构建程序并生成`App`(或`App.exe`)可执行文件,可以打开生成的Visual Studio或XCode解决方案,也可以在终端中键入:

```bash
cmake --build build
```

然后运行由此产生的程序 :

```bash
build/App  # linux/macOS
build\Debug\App.exe  # Windows
```

建议额外
------------------

我们设定了一些目标属性`App`之后打给某个地方`add_executable`联合国`set_target_properties`命令。

```{lit} CMake, Recommended extras
set_target_properties(App PROPERTIES
	CXX_STANDARD 17
	CXX_STANDARD_REQUIRED ON
	CXX_EXTENSIONS OFF
	COMPILE_WARNING_AS_ERROR ON
)
```

那个`CXX_STANDARD`属性被设定为17,这意味着我们需要C++17(这将使我们能够在稍后使用一些合成技巧,但本身并不是强制性的)。 那个`CXX_STANDARD_REQUIRED`属性确保不支持 C++17 配置失败。

那个`CXX_EXTENSIONS`属性设定为`OFF`以禁用编译器特定扩展( 例如, 在 GCC 上, 这将让 CMake 使用)`-std=c++17`而不是说`-std=gnu++17`在编译旗帜列表中).

那个`COMPILE_WARNING_AS_ERROR`作为良好做法,确保不忽略任何警告。 警告实际上很重要,特别是在学习新语言/图书馆时。 为了尽可能多地发出警告,

```{lit} CMake, Recommended extras (append)
if (MSVC)
	target_compile_options(App PRIVATE /W4)
else()
	target_compile_options(App PRIVATE -Wall -Wextra -pedantic)
endif()
```

```{note}
在所附代码中,我把这些细节隐藏在`target_treat_all_warnings_as_errors()`函数定义`utils.cmake`并列入`CMakeLists.txt`.
```

在MacOS上,CMake可以生成XCode项目文件. 但是,默认情况下,没有*计划*已经
创建, XCode 本身为每个 CMake 目标生成一个方案 -- -- 通常我们只有
想要一个计划 我们的主要目标。 因此,我们设定`XCODE_GENERATE_SCHEME`属性。
我们还会启用 GPU 调试的帧捕获 。

```{lit} CMake, Recommended extras (append)
if (XCODE)
	set_target_properties(App PROPERTIES
		XCODE_GENERATE_SCHEME ON
		XCODE_SCHEME_ENABLE_GPU_FRAME_CAPTURE_MODE "Metal"
	)
endif()
```

结论
----------

我们现在有了好东西**基本项目配置**我们将在下几章中再接再厉。 在接下来的章节中,我们将看到如何[整合 WebGPU](hello-webgpu.md)我们的项目,如何[初始化它](adapter-and-device/index.md),如何进行[打开窗口](opening-a-window.md)进入其中。

*结果代码 :* [`step000`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step000)
