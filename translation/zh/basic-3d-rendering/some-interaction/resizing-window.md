调整窗口大小<span class="bullet">🟡</span>
===================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/some-interaction/resizing-window.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step085`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step085)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step085-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step085-vanilla)
````

当我们引入互换链时**我们阻止窗口调整大小**设置`GLFW_RESIZABLE`窗口“int”到“false”,因为互换的链条与特定大小绑定。

现在我们的代码组织得更好了 很容易**摆脱这个限制**:在本章中,我们恢复调整窗口大小的可能性:

```C++
glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE);
```

为此,我们增加了一个`onResize`当窗口大小调整时,我们呼叫的处理器,以及**重建互换链**与新的决议。 我们还需要调整规模**深度缓冲**顺便说一句

召回设置
--------------

让我们首先添加`onResize()`方法 :

```C++
class Application {
public:
	// A function called when the window is resized.
	void onResize();
}
```

GLFW 提供了一个机制,用于设定每次窗口大小变化时要引用的回调:[`glfwSetFramebufferSizeCallback`](https://www.glfw.org/docs/3.0/group__window.html#ga3203461a5303bf289f2e05f854b2f7cf)自然,我们可以考虑以下几点:

```C++
// DON'T (this will not work)
bool Application::initWindowAndDevice() {
	// [...]

	// Add window callbacks
	glfwSetFramebufferSizeCallback(m_window, Application::onResize);

	return /* [...] */;
}
```

**这不行**因为一种非静态的类方法`Application::onResize`需要一个值`this`当它被调用时,所以它不能用作函数指针(即`glfwSetFramebufferSizeCallback`页:1

这里通常的技巧是定义一个简单的召回功能,它除了调用`onResize()`方法。

# # # 但我如何具体说明 "这个"? 我应该使用全球变量吗?

正是为了这种模式,GLFW提供了一种**用户指针**,即可以与窗口相联系的任意值。 由于回调收到窗口作为第一个参数,我们可以调回用户指针并使用:

```C++
// The raw GLFW callback, that must always have the same signature
// even though we do not use the 'width' and 'height' arguments here.
void onWindowResize(GLFWwindow* window, int /* width */, int /* height */) {
	// We know that even though from GLFW's point of view this is
	// "just a pointer", in our case it is always a pointer to an
	// instance of the class `Application`
	auto that = reinterpret_cast<Application*>(glfwGetWindowUserPointer(window));

	// Call the actual class-member callback
	if (that != nullptr) that->onResize();
}

bool Application::initWindowAndDevice() {
	// [...]

	// Set the user pointer to be "this"
	glfwSetWindowUserPointer(m_window, this);
	// Add the raw `onWindowResize` as resize callback
	glfwSetFramebufferSizeCallback(m_window, onWindowResize);

	return /* [...] */;
}
```

如果我们想使这个更紧凑,避免产生一个专门的职能,我们可以使用一个**羊绒**呼叫时`glfwSetFramebufferSizeCallback`:

```C++
bool Application::initWindowAndDevice() {
	// [...]

	// Set the user pointer to be "this"
	glfwSetWindowUserPointer(m_window, this);
	// Use a non-capturing lambda as resize callback
	glfwSetFramebufferSizeCallback(m_window, [](GLFWwindow* window, int, int){
		auto that = reinterpret_cast<Application*>(glfwGetWindowUserPointer(window));
		if (that != nullptr) that->onResize();
	});

	return /* [...] */;
}
```

```{important}
我并没有马上展示Lambda版本,**很诱人**用于使用**抓取上下文**羊肉园里的人,`[]`在Lambda的论点之前)提供`this`呼叫回话。

不过,**只捕捉不捕捉羊羔**可能被投放到GLFW期望召回的原始函数指针上。 事实上,任何消费物价指数都是如此。
```

调整事件处理器大小
--------------------

###交换链和深度缓冲

随着我们的新设计,内容`onResize()`很简单:

```C++
void Application::onResize() {
	// Terminate in reverse order
	terminateDepthBuffer();
	terminateSwapChain();

	// Re-init
	initSwapChain();
	initDepthBuffer();
}
```

```{note}
在这个简单的例子上,**管理寿命**用于WebGPU对象(如重建前销毁,在应用程序结束时释放等)的,可以手动进行. 但是,因为总的来说,**容易出错**,则[拉尔](../../advanced-techniques/raii)章节显示一个常见的 C++**设计模式**这样就容易了
```

然而,我们需要**更新** `initSwapChain()`和`initDepthBuffer()`考虑**窗口的实际大小**,而不是硬码$(640,480)$.

```C++
bool Application::initSwapChain() {
	// Get the current size of the window's framebuffer:
	int width, height;
	glfwGetFramebufferSize(m_window, &width, &height);

	// [...]
	swapChainDesc.width = static_cast<uint32_t>(width);
	swapChainDesc.height = static_cast<uint32_t>(height);
	// [...]
}

bool Application::initDepthBuffer() {
	// Get the current size of the window's framebuffer:
	int width, height;
	glfwGetFramebufferSize(m_window, &width, &height);

	// [...]
	depthTextureDesc.size = { static_cast<uint32_t>(width), static_cast<uint32_t>(height), 1 };
	// [...]
}
```

###相机投影

如果您在代码中查找"640"以确保不再依赖原始窗口大小,你会发现**我们在定义相机投影时使用大小**因此,`onResize`函数还必须更新投影矩阵统一:

```C++
void Application::onResize() {
	// [...] Rebuild swap chain and depth buffer
	updateProjectionMatrix();
}
```

何处`updateProjectionMatrix`是一种新的私人方法:

````{tab} With webgpu.hpp
```C++
void Application::updateProjectionMatrix() {
	int width, height;
	glfwGetFramebufferSize(m_window, &width, &height);
	float ratio = width / (float)height;
	m_uniforms.projectionMatrix = glm::perspective(45 * PI / 180, ratio, 0.01f, 100.0f);
	m_queue.writeBuffer(
		m_uniformBuffer,
		offsetof(MyUniforms, projectionMatrix),
		&m_uniforms.projectionMatrix,
		sizeof(MyUniforms::projectionMatrix)
	);
}
```
````

````{tab} Vanilla webgpu.h
```C++
void Application::updateProjectionMatrix() {
	int width, height;
	glfwGetFramebufferSize(m_window, &width, &height);
	float ratio = width / (float)height;
	m_uniforms.projectionMatrix = glm::perspective(45 * PI / 180, ratio, 0.01f, 100.0f);
	wgpuQueueWriteBuffer(
		m_queue,
		m_uniformBuffer,
		offsetof(MyUniforms, projectionMatrix),
		&m_uniforms.projectionMatrix,
		sizeof(MyUniforms::projectionMatrix)
	);
}
```
````

```{note}
如果你的屏幕大于我们设定的2048年限值, 你需要使用[`glfwGetMonitors`](https://www.glfw.org/docs/3.3/group__monitor.html#ga70b1156d5d24e9928f145d6c864369d2)接着[`glfwGetMonitorWorkarea`](https://www.glfw.org/docs/3.3/group__monitor.html#ga7387a3bdb64bfe8ebf2b9e54f5b6c9d0)当初始化WebGPU限制时, 设定它们至少为您最大监视器的大小 !
```

结论
----------

我们用来将原始的 GLFW 调整大小的事件与 我们的 C++ 偏移对象的应用程序骨架连接的这个模式是**常用图案**:我们将同样为我们需要的所有其他交互回调. 我们在下一章中看到鼠标按钮和鼠标移动调用来获取**相机控制器**!

````{tab} With webgpu.hpp
*结果代码 :* [`step085`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step085)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step085-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step085-vanilla)
````
