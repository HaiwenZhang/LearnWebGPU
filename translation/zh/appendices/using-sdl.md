使用 SDL 进行窗口管理 <span class="bullet">🟡</span>
===============================

```{translation-warning} 译文可能已过时, /appendices/using-sdl.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码：* [`step030-sdl`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-sdl)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-sdl-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-sdl-vanilla)
````

本指南的主要部分使用 GLFW 进行窗口管理，因为它非常轻量级。然而，另一个流行的选择是使用 [SDL](https://wiki.libsdl.org/SDL2/FrontPage) (*Simple DirectMedia Layer*)。

SDL 支持 **Android 和 iOS**，并为编写视频游戏带来了其他出色的功能，例如 **声音处理**。

本附录展示了如何在 [Hello Triangle](../basic-3d-rendering/hello-triangle.md) 章的简单示例中**用 SDL2 替换 GLFW**。

依赖关系
------------

首先删除 `glfw` 和 `glfw3webgpu` 目录。我们将它们替换为 SDL 等效项：

1.下载【最新版本的SDL](https://github.com/libsdl-org/SDL/releases/latest)（使用2.28.5测试）的源代码，解压得到文件`SDL2/README-SDL.txt`。

2. 下载 SDL2 的 `glfw3webgpu` 的等效项，即 [`sdl2webgpu`](https://github.com/eliemichel/sdl2webgpu/archive/refs/heads/main.zip)。解压它，得到一个文件 `sdl2webgpu/CMakeLists.txt`。

在主`CMakeLists.txt`中，将GLFW替换为SDL2：

```diff
- add_subdirectory(glfw)
+ add_subdirectory(SDL2)
add_subdirectory(webgpu)
- add_subdirectory(glfw3webgpu)
+ add_subdirectory(sdl2webgpu)

add_executable(App
	main.cpp
)

- target_link_libraries(App PRIVATE glfw webgpu glfw3webgpu)
+ target_link_libraries(App PRIVATE SDL2::SDL2 webgpu sdl2webgpu)
```

代码
----

在 `main.cpp` 的开头，替换包含 `GLFW` 和 `glfw3webgpu`：

```C++
#include <glfw3webgpu.h>
#include <GLFW/glfw3.h>
```

变成：

```C++
#define SDL_MAIN_HANDLED
#include <sdl2webgpu.h>
#include <SDL2/SDL.h>
```

```{note}
定义 `SDL_MAIN_HANDLED` 可以防止 SDL 创建自己的 `main` 函数，这会与您的函数发生冲突。
```

在 `main` 函数中，替换对 `glfwInit()` 的调用：

```C++
// Remove
if (!glfwInit()) {
	std::cerr << "Could not initialize GLFW!" << std::endl;
	return 1;
}
```

变成：

```C++
// Add
SDL_SetMainReady();
if (SDL_Init(SDL_INIT_VIDEO) < 0) {
	std::cerr << "Could not initialize SDL! Error: " << SDL_GetError() << std::endl;
	return 1;
}
```

```{note}
使用`SDL_MAIN_HANDLED`时需要调用`SDL_SetMainReady()`，而不是让SDL自动管理main函数。
```

更改窗口的创建：

```C++
// Remove
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
GLFWwindow* window = glfwCreateWindow(640, 480, "Learn WebGPU", NULL, NULL);
```

变成：

```C++
// Add
int windowFlags = 0;
SDL_Window *window = SDL_CreateWindow("Learn WebGPU", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED, 640, 480, windowFlags);
```

更改函数名称以获取曲面：

```C++
// Remove
Surface surface = glfwGetWGPUSurface(instance, window);
```

变成：

```C++
// Add
Surface surface = SDL_GetWGPUSurface(instance, window);
```

主循环也发生了变化：

```C++
// Remove
while (!glfwWindowShouldClose(window)) {
	glfwPollEvents();
	// [...]
}
```

变成：

```C++
// Add
bool shouldClose = false;
while (!shouldClose) {

	// Poll events and handle them.
	// (contrary to GLFW, close event is not automatically managed, and there
	// is no callback mechanism by default.)
	SDL_Event event;
	while (SDL_PollEvent(&event))
	{
		switch (event.type)
		{
		case SDL_QUIT:
			shouldClose = true;
			break;

		default:
			break;
		}
	}

	// [...]
}
```

最后，清理是这样完成的：

```C++
// Remove
glfwDestroyWindow(window);
glfwTerminate();
```

变成：

```C++
// Add
SDL_DestroyWindow(window);
SDL_Quit();
```

结论
----------

您现在已经为与 SDL 合作奠定了良好的基础。如果您想移植更高级的章节，您可能需要调整事件处理，让 SDL 事件触发我们用来传递给 GLFW 的回调（例如窗口大小调整、鼠标移动、单击、键盘等）。

```{important}
让我知道 sdl2webgpu 是否无法在您的平台上工作，因为我还没有深入检查。我还没有将其适配到Android和iOS上。
```

````{tab} With webgpu.hpp
*结果代码：* [`step030-sdl`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-sdl)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step030-sdl-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step030-sdl-vanilla)
````
