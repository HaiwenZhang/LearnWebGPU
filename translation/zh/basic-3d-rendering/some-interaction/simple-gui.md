简单的图形界面<span class="bullet">🟡</span>
==========

```{translation-warning} 译文可能已过时, /basic-3d-rendering/some-interaction/simple-gui.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step095`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step095-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-vanilla)
````

````{tab} With Dawn
*结果代码 :* [`step095-dawn`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-vanilla)
````

写入的多个解决方案*图形用户界面*(GUI),即按钮,文本输入,数值滑动器等.

对于主要与这些输入有关的应用程序,通常使用**整个*框架***作为基地,像[数量](https://www.qt.io/), [GTK 游戏](https://www.gtk.org/), [wx 部件](https://www.wxwidgets.org/), [维尼](https://microsoft.github.io/microsoft-ui-xaml/)等 类. 这些通常由自己来管理主应用循环,是沉重的依赖性.

但对于电子游戏或我们的原型来说 通常会转向**更多轻量级解决方案**其中,[亲爱的英桂](https://github.com/ocornut/imgui)这是一个非常流行的选择。

```{note}
Imgui并没有尝试给您的应用程序一个OS-native的外观. 相反,它侧重于非常**易于整合**用于任何现有项目,以及**易于编程**.

这也是为了每个框架从零开始重新绘制整个图形界面,而框架通常只更新需要的内容。
```

设立 Imgui
----------------

ImGui完全支持使用WebGPU作为Wgpu-native和Dawn的后端,因为WebGPU是Wgpu-native和Dawn的后端.[1.89.8号版本](https://github.com/ocornut/imgui/archive/refs/tags/v1.89.8.zip)打开它作为一个`imgui/`目录, 删除`examples`, `doc`和`.github`(或留着,但我们不需要它们)

ImGui没有提供`CMakeLists.txt`但是我们自己写出来很直接(still in the `imgui/` directory):

```CMake
# Define an ImGui target that fits our use case
add_library(imgui STATIC
	# Among the different backends available, we are interested in connecting
	# the GUI to GLFW andWebGPU:
	backends/imgui_impl_wgpu.h
	backends/imgui_impl_wgpu.cpp
	backends/imgui_impl_glfw.h
	backends/imgui_impl_glfw.cpp

	# Bonus to add some C++ specific features (the core ImGUi is a C library)
	misc/cpp/imgui_stdlib.h
	misc/cpp/imgui_stdlib.cpp

	# The core ImGui files
	imconfig.h
	imgui.h
	imgui.cpp
	imgui_draw.cpp
	imgui_internal.h
	imgui_tables.cpp
	imgui_widgets.cpp
	imstb_rectpack.h
	imstb_textedit.h
	imstb_truetype.h
)

target_include_directories(imgui PUBLIC .)
target_link_libraries(imgui PUBLIC webgpu glfw)
```

在你的根头`CMakeLists.txt`如往常一样:

```CMake
add_subdirectory(imgui)

# [...]

target_link_libraries(App PRIVATE glfw webgpu glfw3webgpu imgui)
```

使用量
-----

###概览

ImGui 界面在每个帧重画,遵循非常必要的样式:


```C++
#include <imgui.h>

void Application::onFrame() {
	// [...]

	// [...] Init ImGui frame

	ImGui::Begin("Hello, world!");
	ImGui::Text("This is some useful text.");
	if (ImGui::Button("Click me")) {
		// do something
	}
	ImGui::End();

	// [...] Draw ImGui frame
}
```

可用的函数可见于[`imgui.h`](https://github.com/ocornut/imgui/blob/master/imgui.h),并在[他们的维基语录](https://github.com/ocornut/imgui/wiki).

###设置

不过**需要设置一些锅炉板**,在启动应用程序时以及在定义每个框架的图形用户界面之前/之后。

为使情况更加明确,我们将与GUI有关的代码隔离成具体的方法(注意我们需要进入 Compare ass in`updateGui`):

```C++
// In Application.h
class Application {
private:
	bool initGui(); // called in onInit
	void terminateGui(); // called in onFinish
	void updateGui(wgpu::RenderPassEncoder renderPass); // called in onFrame
};

// In Application.cpp
void Application::onInit() {
	// [...]
	if (!initGui()) return false;
	return true;
}

void Application::onFinish() {
	terminateGui();
	// [...]
}

void Application::onFrame() {
	// [...]

	renderPass.draw(m_indexCount, 1, 0, 0);

	// We add the GUI drawing commands to the render pass
	updateGui(renderPass);

	renderPass.end();

	// [...]
}
```

锅盘本身来了 对于每个步骤(global init,frame init,frame compare),通常都有"纯"的ImGui函数以及后端函数. 由于ImGui可以和GLFW和WebGPU之外的其他库一起使用,所以事情被这样解开.

```C++
// In Application.cpp
#include <imgui.h>
#include <backends/imgui_impl_wgpu.h>
#include <backends/imgui_impl_glfw.h>

bool Application::initGui() {
	// Setup Dear ImGui context
	IMGUI_CHECKVERSION();
	ImGui::CreateContext();
	ImGui::GetIO();

	// Setup Platform/Renderer backends
	ImGui_ImplGlfw_InitForOther(m_window, true);
	ImGui_ImplWGPU_Init(m_device, 3, m_swapChainFormat, m_depthTextureFormat);
	return true;
}

void Application::terminateGui() {
	ImGui_ImplGlfw_Shutdown();
	ImGui_ImplWGPU_Shutdown();
}

void Application::updateGui(RenderPassEncoder renderPass) {
	// Start the Dear ImGui frame
	ImGui_ImplWGPU_NewFrame();
	ImGui_ImplGlfw_NewFrame();
	ImGui::NewFrame();

	// [...] Build our UI

	// Draw the UI
	ImGui::EndFrame();
	// Convert the UI defined above into low-level drawing commands
	ImGui::Render();
	// Execute the low-level drawing commands on the WebGPU backend
	ImGui_ImplWGPU_RenderDrawData(ImGui::GetDrawData(), renderPass);
}
```

###图形界面示例

这个ImGui的基本例子,显示了一些**典型使用案例**.

注意变量的定义是:**静态**在这里,所以它们只被初始化一次**"记得"**跨边框。

```C++
// Build our UI
static float f = 0.0f;
static int counter = 0;
static bool show_demo_window = true;
static bool show_another_window = false;
static ImVec4 clear_color = ImVec4(0.45f, 0.55f, 0.60f, 1.00f);

ImGui::Begin("Hello, world!");                                // Create a window called "Hello, world!" and append into it.

ImGui::Text("This is some useful text.");                     // Display some text (you can use a format strings too)
ImGui::Checkbox("Demo Window", &show_demo_window);            // Edit bools storing our window open/close state
ImGui::Checkbox("Another Window", &show_another_window);

ImGui::SliderFloat("float", &f, 0.0f, 1.0f);                  // Edit 1 float using a slider from 0.0f to 1.0f
ImGui::ColorEdit3("clear color", (float*)&clear_color);       // Edit 3 floats representing a color

if (ImGui::Button("Button"))                                  // Buttons return true when clicked (most widgets return true when edited/activated)
	counter++;
ImGui::SameLine();
ImGui::Text("counter = %d", counter);

ImGuiIO& io = ImGui::GetIO();
ImGui::Text("Application average %.3f ms/frame (%.1f FPS)", 1000.0f / io.Framerate, io.Framerate);
ImGui::End();
```

###能力

使用 ImGUI 需要`maxBindGroups`至少为2岁。

```C++
requiredLimits.limits.maxBindGroups = 2;
//                                    ^ This was a 1
```

杂项
----

在玩滑动器时你可能注意到 当Imgui对你的鼠标反应时**相机控制器也接收事件**这有点烦人。

为了防止这种情况,我们可以使用`io.WantCaptureMouse`当 ImGui 检测到用户与部件交互时,该变量变为真实。 当这样的时候,我们忽略了鼠标点击相机控制器:

```C++
void Application::onMouseButton(int button, int action, int mods) {
	ImGuiIO& io = ImGui::GetIO();
	if (io.WantCaptureMouse) {
		// Don't rotate the camera if the mouse is already captured by an ImGui
		// interaction at this frame.
		return;
	}

	// [...]
}
```

```{figure} /images/first-imgui.png
:align: center
:class: with-shadow
带有 ImGUI 的基本图形界面
```

结论
----------

恭喜,我们现在有了所有的工具,可以方便地测试各种参数,并与3D场景进行互动,然后可以转到照明和材料上!

````{tab} With webgpu.hpp
*结果代码 :* [`step095`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step095-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-vanilla)
````

````{tab} With Dawn
*结果代码 :* [`step095-dawn`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step095-vanilla)
````
