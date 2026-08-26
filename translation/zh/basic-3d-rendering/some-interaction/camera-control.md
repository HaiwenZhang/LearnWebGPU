相机控制<span class="bullet">🟡</span>
==============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/some-interaction/camera-control.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step090`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step090)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step090-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step090-vanilla)
````

我们通常需要与3D场景进行第一种互动**改变观点**。根据使用情况,相机控制有很多不同类型,以下是几个例子:

 - **第一人称:**相机位置是固定的,其方向遵循鼠标光标. 这就是第一人称电子游戏中像射手一样使用的东西.
 - **转盘 :**这就是模拟工具通常使用的方法:照相机**轨道**环绕着它仍以它为中心的焦点。
 - **田径球 :**与转盘不同,田径球并没有给"上"轴赋予特定意义. 这使得一个人能够绕物体绕轨道飞行,而没有任何形式的[Gimbal 锁](https://en.wikipedia.org/wiki/Gimbal_lock).

```{note}
有不同的风味*转盘*控制取决于旋转中心的选择方式:它可能被固定,或设定为每次互动开始时点击的3D表面的点。
```

我们在这里集中处理**转盘**模型,我为3D对象查看器找到最舒适的模型. Imho大多数现实生活中的物体都有一个"上"和"下"的概念,这证明视图控制器也这样做是有道理的.

事件处理器
--------------

和我们为调整规模所做的一样,我们用电线连接了3个新的GLFW窗口事件:

```C++
class Application {
	// Mouse events
	void onMouseMove(double xpos, double ypos);
	void onMouseButton(int button, int action, int mods);
	void onScroll(double xoffset, double yoffset);
	// [...]
}
```

```C++
// Add window callbacks
glfwSetWindowUserPointer(m_window, this);
glfwSetFramebufferSizeCallback(m_window, /* [...] */);
glfwSetCursorPosCallback(m_window, [](GLFWwindow* window, double xpos, double ypos) {
	auto that = reinterpret_cast<Application*>(glfwGetWindowUserPointer(window));
	if (that != nullptr) that->onMouseMove(xpos, ypos);
});
glfwSetMouseButtonCallback(m_window, [](GLFWwindow* window, int button, int action, int mods) {
	auto that = reinterpret_cast<Application*>(glfwGetWindowUserPointer(window));
	if (that != nullptr) that->onMouseButton(button, action, mods);
});
glfwSetScrollCallback(m_window, [](GLFWwindow* window, double xoffset, double yoffset) {
	auto that = reinterpret_cast<Application*>(glfwGetWindowUserPointer(window));
	if (that != nullptr) that->onScroll(xoffset, yoffset);
});
```

相机状态
------------

我们不直接操纵相机视星等作为由16个系数组成的矩阵,而是存储一个相机状态,即**更接近用户输入的影响**:

```C++
// (After the definition of struct MyUniforms)
struct CameraState {
	// angles.x is the rotation of the camera around the global vertical axis, affected by mouse.x
	// angles.y is the rotation of the camera around its local horizontal axis, affected by mouse.y
	vec2 angles = { 0.8f, 0.5f };
	// zoom is the position of the camera along its local forward axis, affected by the scroll wheel
	float zoom = -1.2f;
};
```

然后,我们加上这个结构 作为状态在我们`Application`课。

```C++
// In the declaration of class Application
private:
	CameraState m_cameraState;
```

然后我们创建一种(私人)方法,将这个状态转换成一个实际矩阵. 我们称它为任何时候 相机状态被修改:

````{tab} With webgpu.hpp
```C++
void Application::updateViewMatrix() {
	float cx = cos(m_cameraState.angles.x);
	float sx = sin(m_cameraState.angles.x);
	float cy = cos(m_cameraState.angles.y);
	float sy = sin(m_cameraState.angles.y);
	vec3 position = vec3(cx * cy, sx * cy, sy) * std::exp(-m_cameraState.zoom);
	m_uniforms.viewMatrix = glm::lookAt(position, vec3(0.0f), vec3(0, 0, 1));
	m_queue.writeBuffer(
		m_uniformBuffer,
		offsetof(MyUniforms, viewMatrix),
		&m_uniforms.viewMatrix,
		sizeof(MyUniforms::viewMatrix)
	);
}
```
````

````{tab} Vanilla webgpu.h
```C++
void Application::updateViewMatrix() {
	float cx = cos(m_cameraState.angles.x);
	float sx = sin(m_cameraState.angles.x);
	float cy = cos(m_cameraState.angles.y);
	float sy = sin(m_cameraState.angles.y);
	vec3 position = vec3(cx * cy, sx * cy, sy) * std::exp(-m_cameraState.zoom);
	m_uniforms.viewMatrix = glm::lookAt(position, vec3(0.0f), vec3(0, 0, 1));
	wgpuQueueWriteBuffer(
		m_queue,
		m_uniformBuffer,
		offsetof(MyUniforms, viewMatrix),
		&m_uniforms.viewMatrix,
		sizeof(MyUniforms::viewMatrix)
	);
}
```
````

```{note}
你可以援引`updateViewMatrix()`结束时`initUniforms()`确保原始视图矩阵与相机状态一致。
```

主计长
----------

与相机控制器的交互由下列事件的顺序组成:

- 老鼠来了**按键**.
- 老鼠来了**移动**.
- 老鼠来了**移动**.
 - [...]
- 老鼠来了**移动**.
- 老鼠来了**已发行**.

当鼠标被按下时,我们保存一些关于当前状态的信息,我们需要在以后的每次移动时更新视图. 我们称之为`DragState`.

当鼠标发布时,我们忘记了这些信息,以防止新的动作影响视点.

```C++
struct DragState {
	// Whether a drag action is ongoing (i.e., we are between mouse press and mouse release)
	bool active = false;
	// The position of the mouse at the beginning of the drag action
	vec2 startMouse;
	// The camera state at the beginning of the drag action
	CameraState startCameraState;

	// Constant settings
	float sensitivity = 0.01f;
	float scrollSensitivity = 0.1f;
};

// In the declaration of class Application
DragState m_drag;
```

```C++
void Application::onMouseMove(double xpos, double ypos) {
	if (m_drag.active) {
		vec2 currentMouse = vec2(-(float)xpos, (float)ypos);
		vec2 delta = (currentMouse - m_drag.startMouse) * m_drag.sensitivity;
		m_cameraState.angles = m_drag.startCameraState.angles + delta;
		// Clamp to avoid going too far when orbitting up/down
		m_cameraState.angles.y = glm::clamp(m_cameraState.angles.y, -PI / 2 + 1e-5f, PI / 2 - 1e-5f);
		updateViewMatrix();
	}
}

void Application::onMouseButton(int button, int action, int /* modifiers */) {
	if (button == GLFW_MOUSE_BUTTON_LEFT) {
		switch(action) {
		case GLFW_PRESS:
			m_drag.active = true;
			double xpos, ypos;
			glfwGetCursorPos(m_window, &xpos, &ypos);
			m_drag.startMouse = vec2(-(float)xpos, (float)ypos);
			m_drag.startCameraState = m_cameraState;
			break;
		case GLFW_RELEASE:
			m_drag.active = false;
			break;
		}
	}
}
```

在使用滚动时, 我们还添加一个简单的交互功能, 以缩放 :

```C++
void Application::onScroll(double /* xoffset */, double yoffset) {
	m_cameraState.zoom += m_drag.scrollSensitivity * static_cast<float>(yoffset);
	m_cameraState.zoom = glm::clamp(m_cameraState.zoom, -2.0f, 2.0f);
	updateViewMatrix();
}
```

奖金:因诺蒂亚
--------------

不错的添加**外观和感觉( F)**您的观看者将添加一些**势头**以淡化用户的手势。

为此,我们在拖动状态中添加当前**速度**,然后在下一个框架的旋转中加入一点。

```C++
struct DragState {
	// [...]
	
	// Inertia
	vec2 velocity = {0.0, 0.0};
	vec2 previousDelta;
	float inertia = 0.9f;
};
```

```C++
void Application::onMouseMove(double xpos, double ypos) {
	if (m_drag.active) {
		// [...]

		// Inertia
		m_drag.velocity = delta - m_drag.previousDelta;
		m_drag.previousDelta = delta;
	}
}
```

我们需要定义一个`updateDragInertia()`而不是仅仅当用户移动鼠标时 :

```C++
// In Application.h
class Application {
private:
	void updateDragInertia();
	// [...]
};
```

```C++
// In Application.cpp
void Application::onFrame() {
	updateDragInertia();
	// [...]
}

void Application::updateDragInertia() {
	constexpr float eps = 1e-4f;
	// Apply inertia only when the user released the click.
	if (!m_drag.active) {
		// Avoid updating the matrix when the velocity is no longer noticeable
		if (std::abs(m_drag.velocity.x) < eps && std::abs(m_drag.velocity.y) < eps) {
			return;
		}
		m_cameraState.angles += m_drag.velocity;
		m_cameraState.angles.y = glm::clamp(m_cameraState.angles.y, -PI / 2 + 1e-5f, PI / 2 - 1e-5f);
		// Dampen the velocity so that it decreases exponentially and stops
		// after a few frames.
		m_drag.velocity *= m_drag.inertia;
		updateViewMatrix();
	}
}
```

结论
----------

相机控制器是进入照明前需要的重要一步,因为我们需要详细检查我们的模型.

当然,你可以自由地 适应自己的相机模型。 以此为例,您已经可以做很多工作,而且您应该能够轻松地自行添加一些键盘交互[`glfwSetKeyCallback`](https://www.glfw.org/docs/3.0/group__input.html#ga7e496507126f35ea72f01b2e6ef6d155).

接下来我们再看最后一点通用代码 以获得一个基础**用户界面**,然后我们搬回3D特定的东西!

````{tab} With webgpu.hpp
*结果代码 :* [`step090`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step090)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step090-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step090-vanilla)
````
