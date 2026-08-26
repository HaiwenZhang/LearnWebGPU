打开窗口<span class="bullet">🟢</span>
================

```{translation-warning} 译文可能已过时, /getting-started/opening-a-window.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/020 - Opening a window
:parent: 015 - The Command Queue
:fetch-files: ../data/glfw-3.4.0-light.zip, ../data/glfw3webgpu-v1.2.0.zip
```

*结果代码 :* [`step020`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step020)

在能够将任何东西在屏幕上渲染之前,我们需要要求操作系统(OS)给我们一些地方来绘制东西,这种东西通常被称为**窗口**.

打开窗口的过程**取决于操作系统**,所以我们使用 一个小图书馆叫[两性平等](https://www.glfw.org/)它统一了不同的窗口管理 API 并使我们的代码成为**不可知论**在操作系统。

```{note}
我尽量少用图书馆 但需要这个**让我们的代码跨平台**对我来说 比从头写代码更重要 GLFW是另一个**一个非常常见的选择**和相当小的设计。
```

```{admonition} Headless mode
网络GPU**不需要窗口**要画东西,它可能运行 无头和画到**屏幕外纹理**由于这不是像在窗口中画画那样常见的用法,我将这个选项的细节留给[专门一章](../advanced-techniques/headless.md)高级部分。
```

```{admonition} SDL
**另一个流行的选择**用于窗口管理**SDL 库**它没有GLFW那么轻,而是提供了更多的特性,比如支持声音和Android/iOS目标.[专门附录](../appendices/using-sdl.md)显示在使用 SDL 时对主向导的修改。
```

GLFW的一体化
-------------------

没错**不需要安装**它,我们只需要增加 GLFW的代码 在我们的项目目录。 下载文件[glfw.zip (英语).](../../../data/glfw-3.4.0-light.zip)(780 KB)和 (中文(简体) ).**调色调**在你的项目。 我删除了文件、例子和测试,**轻型**.

改为**整合 GLFW**在我们的项目中,我们把它的目录加入我们的根`CMakeLists.txt`与`add_subdirectory(glfw)`我们添加一个**标记的例外**因为它已经**内置**支持GLFW;在这种情况下,我们需要的只是确定`-sUSE_GLFW=3`链接选项。

```{lit} CMake, Dependency subdirectories (prepend)
if (NOT EMSCRIPTEN)
	# Add the 'glfw' directory, which contains the definition of a 'glfw' target
	add_subdirectory(glfw)
else()
	# Create a mock 'glfw' target that just sets the `-sUSE_GLFW=3` link option:
	add_library(glfw INTERFACE)
	target_link_options(glfw INTERFACE -sUSE_GLFW=3)
endif()
# In both cases, we can now link to the 'glfw' target
```

```{note}
使用时**黎明**,确保添加`glfw`目录**在此之前**您添加时`webgpu`,否则Dawn会提供自己的版本(有时可能很好,但你无法选择该版本).
```

那么,我们必须告诉CMake**将我们的应用程序链接到此库**就像我们对Webgpu所做的:

```{lit} CMake, Link libraries (replace)
# Add the 'glfw' target as a dependency of our App
target_link_libraries(App PRIVATE webgpu glfw)
```

您现在应该能够构建应用程序并添加`#include <GLFW/glfw3.h>`在主文件开头处。

```{lit} C++, Includes (append)
#include <GLFW/glfw3.h>
```

````{important}
如果你在一个**链接**系统,确保安装[GLFW 所需依赖性](https://www.glfw.org/docs/3.3/compile.html#compile_deps)默认情况下,它试图为两者建造**十进制**和**韦兰**所以,你需要两组依赖。 如果你只想使用/安装其中之一,请转过身来`GLFW_BUILD_X11`或`GLFW_BUILD_WAYLAND`调用cmake时关闭,例如:

```
#仅使用 X11 支持构建
cmake - B 构建 - DGLFW_比尔_韦兰德=OFF
```
````

基本用途
-----------

在本节中,我们熟悉GLFW,而没有任何WebGPU特定的考虑.

###初始化

首先,任何给GLFW库的电话必须在初始化和终止之间:

```{lit} C++, Main Content
glfwInit();
{{Use GLFW}}
glfwTerminate();
```

init 函数返回**虚假**当它不能设局的时候,

```{lit} C++, Use GLFW
if (!glfwInit()) {
	std::cerr << "Could not initialize GLFW!" << std::endl;
	return 1;
}
{{Create and destroy window (hidden)}}
```

图书馆一经初始化,我们就可以**创建窗口**:

```{lit} C++, Create and destroy window
// Create the window
GLFWwindow* window = glfwCreateWindow(640, 480, "Learn WebGPU", nullptr, nullptr);

{{Use the window}}

// At the end of the program, destroy the window
glfwDestroyWindow(window);
```

再来一点**错误管理**:

```{lit} C++, Use the window
if (!window) {
	std::cerr << "Could not open window!" << std::endl;
	glfwTerminate();
	return 1;
}
```

###窗口提示

那个`glfwCreateWindow`函数具有一些可选性**其他参数**电话传来`glfwWindowHint` **在此之前**引用`glfwCreateWindow`。我们增加了两个提示:

- 设定`GLFW_CLIENT_API`改为`GLFW_NO_API`告诉GLFW**不关心图形 API**,因为它不知道WebGPU,而且我们不会使用它默认为其他API设置的东西.
- 设定`GLFW_RESIZABLE`改为`GLFW_FALSE`防止用户**调整窗口大小**我们稍后会解开这个限制, 但现在它避免了一些不方便的坠机。

```{lit} C++, Create window
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API); // <-- extra info for glfwCreateWindow
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
GLFWwindow* window = glfwCreateWindow(640, 480, "Learn WebGPU", nullptr, nullptr);
```

```{tip}
我邀请你看看GLFW的文件 了解更多关于[`glfwCreateWindow`](https://www.glfw.org/docs/latest/group__window.html#ga3555a418df92ad53f917597fe2f64aeb)和其他相关职能。
```

###主循环

此时,窗口打开**立即关闭**后结. 为了解决这个问题,我们加入了应用程序的**主循环**就在所有释放/销毁/终止电话之前:

```{lit} C++, Main loop
while (!glfwWindowShouldClose(window)) {
	// Check whether the user clicked on the close button (and any other
	// mouse/key event, which we don't use so far)
	glfwPollEvents();
}
```

```{note}
这个主循环是应用程序逻辑大部分发生的地方. 我们将反复澄清和重新绘制整个图像,并检查新的用户输入。
```

```{figure} /images/glfw-boilerplate.png
:align: center
:class: with-shadow
我们的第一个窗口,使用 GLFW 库。
```

```{warning}
此主循环将**没有工作**与**标记**。在网页中,主循环由浏览器处理,我们只是告诉它**每个边框的调用**我们**修复这个**在下面重构我们的程序`Application`课。
```

申请类
-----------------

###内容

让我们**重新组织我们的项目**因此,它更便于使用,更清楚地了解**初始化**对**主循环**分别.

我们创建函数`Initialize()`, `MainLoop()`和`Terminate()`将我们计划的三个关键部分分开 我们还把**所有变量**我们要求这些职能在共同的类别/结构中共享,例如,`Application`为了更好的可读性,我们可能有`Initialize()`, `MainLoop()`和`Terminate()`成为这一类的成员:

```{lit} C++, Application class
class Application {
public:
	// Initialize everything and return true if it went all right
	bool Initialize();

	// Uninitialize everything that was initialized
	void Terminate();

	// Draw a frame and handle events
	void MainLoop();

	// Return true as long as the main loop should keep on running
	bool IsRunning();

private:
	// We put here all the variables that are shared between init and main loop
	{{Application attributes}}
};
```

```{lit} C++, file: main.cpp (replace, hidden)
{{Includes}}

{{Application class}}

{{Main function}}

{{Application implementation}}
```

我们的主要功能变得如此简单:

```{lit} C++, Main function
int main() {
	Application app;

	if (!app.Initialize()) {
		return 1;
	}

	// Warning: this is still not Emscripten-friendly, see below
	while (app.IsRunning()) {
		app.MainLoop();
	}

	app.Terminate();

	return 0;
}
```

现在我们可以把几乎所有的代码移动到`Initialize()`唯一属于`MainLoop()`目前是GLFW事件投票和WebGPU设备:

```{lit} C++, Application implementation
bool Application::Initialize() {
	// Move the whole initialization here
	{{Initialize}}
	return true;
}

void Application::Terminate() {
	// Move all the release/destroy/terminate calls here
	{{Terminate}}
}

void Application::MainLoop() {
	glfwPollEvents();

	{{Main loop content}}

	// Also move here the tick/poll but NOT the emscripten sleep
	{{Poll WebGPU Events}}
}

bool Application::IsRunning() {
	return !glfwWindowShouldClose(window);
}
```

```{lit} C++, Poll WebGPU Events (hidden)
#if defined(WEBGPU_BACKEND_DAWN)
	wgpuDeviceTick(device);
#elif defined(WEBGPU_BACKEND_WGPU)
	wgpuDevicePoll(device, false, nullptr);
#endif
```

```{lit} C++, Main loop content (hidden)
```

```{lit} C++, Initialize (hidden)
{{Open window and get adapter}}

{{Request device}}

// We no longer need to access the adapter
wgpuAdapterRelease(adapter);

{{Add device error callback}}

queue = wgpuDeviceGetQueue(device);
```

```{lit} C++, Open window and get adapter (hidden)
// Open window
glfwInit();
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API); // <-- extra info for glfwCreateWindow
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
window = glfwCreateWindow(640, 480, "Learn WebGPU", nullptr, nullptr);

// Create instance
WGPUInstance instance = wgpuCreateInstance(nullptr);

// Get adapter
std::cout << "Requesting adapter..." << std::endl;
{{Request adapter}}
std::cout << "Got adapter: " << adapter << std::endl;

// We no longer need to access the instance
wgpuInstanceRelease(instance);
```

```{lit} C++, Add device error callback (hidden)
// Device error callback
auto onDeviceError = [](WGPUErrorType type, char const* message, void* /* pUserData */) {
	std::cout << "Uncaptured device error: type " << type;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
};
wgpuDeviceSetUncapturedErrorCallback(device, onDeviceError, nullptr /* pUserData */);
```

```{lit} C++, Request device (hidden)
// Get device
std::cout << "Requesting device..." << std::endl;
WGPUDeviceDescriptor deviceDesc = {};
deviceDesc.nextInChain = nullptr;
{{Build device descriptor}}
device = requestDeviceSync(adapter, &deviceDesc);
std::cout << "Got device: " << device << std::endl;
```

```{lit} C++, Build device descriptor (hidden)
deviceDesc.label = "My Device"; // anything works here, that's your call
deviceDesc.requiredFeatureCount = 0; // we do not require any specific feature
deviceDesc.requiredLimits = nullptr; // we do not require any specific limit
deviceDesc.defaultQueue.nextInChain = nullptr;
deviceDesc.defaultQueue.label = "The default queue";
// A function that is invoked whenever the device stops being available.
deviceDesc.deviceLostCallback = [](WGPUDeviceLostReason reason, char const* message, void* /* pUserData */) {
	std::cout << "Device lost: reason " << reason;
	if (message) std::cout << " (" << message << ")";
	std::cout << std::endl;
};
```

```{lit} C++, Terminate (hidden)
wgpuQueueRelease(queue);
{{Destroy surface}}
wgpuDeviceRelease(device);
glfwDestroyWindow(window);
glfwTerminate();
```

```{important}
这么说**别**移动`emscripten_sleep(100)`直线到`MainLoop()`。一旦我们**让浏览器处理**主循环,因为浏览器会勾选其WebGPU后端本身.
```

一旦您移动了所有的东西, 您应该最终拥有以下类属性 。

```{lit} C++, Application attributes
GLFWwindow *window;
WGPUDevice device;
WGPUQueue queue;
```

```{note}
那个`WGPUInstance`和`WGPUAdapter`是获得在初始化中可能释放的设备的中间步骤。
```

### Emscripten

正如上文多次提到的那样,明确写入`while`在构建时无法循环**网页**因为它**冲突**与网页浏览器自己的环路。 在这样的情况下,我们用不同的方式写出主要循环:

```{lit} C++, Main loop (replace)
#ifdef __EMSCRIPTEN__
	{{Emscripten main loop}}
#else // __EMSCRIPTEN__
	while (app.IsRunning()) {
		app.MainLoop();
	}
#endif // __EMSCRIPTEN__
```

```{lit} C++, Main function (replace, hidden)
int main() {
	Application app;

	if (!app.Initialize()) {
		return 1;
	}

	{{Main loop}}

	app.Terminate();

	return 0;
}
```

在这里,我们使用功能[`emscripten_set_main_loop_arg()`](https://emscripten.org/docs/api_reference/emscripten.h.html#c.emscripten_set_main_loop_arg),这恰恰是**专门讨论这一问题**此设置为**回调**浏览器每次运行主渲染循环时都会调用。

```C++
// Callback type takes one argument of type 'void*' and returns nothing
typedef void (*em_arg_callback_func)(void*);

// Signature of 'emscripten_set_main_loop_arg' as provided in emscripten.h
void emscripten_set_main_loop_arg(
	em_arg_callback_func func,
	void *arg,
	int fps,
	int simulate_infinite_loop
)
```

我们可以认出来**回调模式**用于请求适配器和设备,或者设置错误调用。 所谓的`arg`这是WebGPU所说的`userdata`: 这是一个指针是**盲目传递到召回**函数。

```{lit} C++, Emscripten main loop
// Equivalent of the main loop when using Emscripten:
auto callback = [](void *arg) {
    //                   ^^^ 2. We get the address of the app in the callback.
    Application* pApp = reinterpret_cast<Application*>(arg);
    //                  ^^^^^^^^^^^^^^^^ 3. We force this address to be interpreted
    //                                      as a pointer to an Application object.
    pApp->MainLoop(); // 4. We can use the application object
};
emscripten_set_main_loop_arg(callback, &app, 0, true);
//                                     ^^^^ 1. We pass the address of our application object.
```

建议额外论据为:`0`和`true`:

 - `fps`这是**帧率**函数在其中被调用。 为:**业绩较好**,建议设定为 0 ,将其留给浏览器(相当于使用`requestAnimationFrame`在 Java 脚本中)
 - `simulate_infinite_loop`必须是`true`预防`app`从被解放。 否则,`main`函数**在调用回话前返回**,因此应用程序已不复存在,`arg`指针是*摇摆*(即指无实.

表面
-----------

现在是时候**将 GLFW 窗口连接到 WebGPU**。这发生在**请求适配器**,通过指定**缩写**对象以 :

```{lit} C++, Request adapter (replace)
{{Get the surface}}

WGPURequestAdapterOptions adapterOpts = {};
adapterOpts.nextInChain = nullptr;
adapterOpts.compatibleSurface = surface;
//                              ^^^^^^^ Use the surface here

WGPUAdapter adapter = requestAdapterSync(instance, &adapterOpts);
```

**如何获得水面?**这取决于OS,GLFW不为我们处理,因为它不知道WebGPU([yet?](https://github.com/glfw/glfw/pull/2333)) (中文(简体) ). 因此,我为您提供这个功能, 在GLFW3的小扩展中,[`glfw3webgpu`](https://github.com/eliemichel/glfw3webgpu).

###GLFW3 WebGPU 扩展

下载和解析[glfw3webgpu.zip (英语).](https://github.com/eliemichel/glfw3webgpu/releases/download/v1.2.0/glfw3webgpu-v1.2.0.zip)在您的项目目录中。 现在应该有一个目录`glfw3webgpu`坐在你身边`main.cpp`。和以前一样,我们可以添加这个目录,并将其创建的目标链接到我们的App上:

```{lit} CMake, glfw3webgpu subdirectory (insert in {{Dependency subdirectories}} after "add_subdirectory(webgpu)")
add_subdirectory(glfw3webgpu)
```

```{lit} CMake, Link libraries (replace)
target_link_libraries(App PRIVATE glfw webgpu glfw3webgpu)
target_copy_webgpu_binaries(App)
```

```{note}
那个`glfw3webgpu`库是**很简单**,它只是两个文件,所以我们几乎可以把它们直接列入我们的项目的源树. 然而,它需要一些特殊编译旗帜 在macOS中,我们不得不处理(你可以看到它们在`CMakeLists.txt`).
```

你现在可以通过简单的行动获得表面:

```{lit} C++, Includes (append)
#include <glfw3webgpu.h>
```

```{lit} C++, Get the surface
surface = glfwGetWGPUSurface(instance, window);
```

也不要忘记在结尾释放表面:

```{lit} C++, Destroy surface
wgpuSurfaceRelease(surface);
```

````{important}
那个**表面**生活独立于适配器和装置,所以**绝对不能**释放**在结束之前**就像我们对适配器和实例所做的那样 因此,它被定义为:`Application`:

```{lit} C++, Application attributes (append)
WGPUSurface surface;
```
````

结论
----------

在本章中,我们规定如下:

- 用那个[两性平等](https://www.glfw.org/)要处理的库**窗口**(以及用户输入,见后).
- 调整我们的代码**初始化**从**主循环**.
 - **连接 WebGPU**使用[glfw3webgpu (英语).](https://github.com/eliemichel/glfw3webgpu)扩展名。

我们现在准备好**显示一些东西**这窗户上!

*结果代码 :* [`step020`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step020)
