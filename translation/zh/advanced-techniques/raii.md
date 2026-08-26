RAII <span class="bullet">🟡</span>
====

```{translation-warning} 译文可能已过时, /advanced-techniques/raii.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

*结果代码：* [`gist`](https://gist.github.com/eliemichel/2e154152f981e3f16827ba4d17d1123a)

是否厌倦了管理 `foo.release()` 以确保它与 `device.createFoo()` 匹配并且您没有悬空资源？ C API 需要这种**手动机制**，但在 C++ 中，借助**析构函数**，我们可以将其**包装**成更方便的接口。

这里介绍的**编程习惯**称为[RAII](https://en.cppreference.com/w/cpp/language/raii)，代表*“资源获取即初始化”*；它适用于许多其他非 WebGPU 相关的上下文！

这个想法是编写一个 C++ 类，我们**通过构造**强制使其代表的底层资源存在**当且仅当**该类的实例仍然存在。

励志例子
---------------------

### 功能

让我们从一些示例开始，假设我们有一个类 `raii::Buffer`，它使用 RAII 习惯用法包装 `wgpu::Buffer` 资源。

```C++
void foo(Device device) {
	BufferDescriptor bufferDesc = /* ... */;

	// Create an instance of our RAII class, which by construction allocates
	// a new wgpu::Buffer.
	raii::Buffer buffer(device, bufferDesc);

	// [...] Do stuff with `buffer`
}
```

请注意，这里不需要调用 `buffer.release()`：**一旦变量超出范围**，它所包装的缓冲区就会自动释放。

### 类

如果我们需要缓冲区寿命更长，我们通常可以将其定义为类成员：

```C++
class Foo {
private:
	// Class has a buffer attribute
	raii::Buffer m_buffer;

public:
	// Constructor must initialize m_buffer
	Foo(Device device)
		: m_buffer(device, describeBuffer())
	{}

	void doStuff() {
		// [...] Do stuff with `buffer`
	}

private:
	BufferDescriptor describeBuffer() { /* ... */ }
};
```

这次，每当 `Foo` 的实例被析构时，缓冲区就会被释放：它会自动调用 `raii::Buffer` 的析构函数，从而释放缓冲区资源。

```{important}
由于 `raii::Buffer` 不能在没有底层缓冲区的情况下存在，因此它必须由构造函数初始化。
```

### 智能指针

有时我们确实希望能够存储“空”缓冲区，因此在定义 RAII 实例后立即初始化它可能会很烦人。典型的解决方案是使用标准库提供的**智能指针**：

```C++
#include <memory> // for smart pointers

class Foo {
private:
	// Buffer attribute is a unique smart pointer
	std::unique_ptr<raii::Buffer> m_buffer;

public:
	// No need for a constructor here
	void doStuff(Device device) {
		if (!m_buffer) initBuffer(device);
		// [...] Do stuff with `buffer`
	}

private:
	void initBuffer(Device device) {
		// The make_unique<T> utility function takes the same arguments
		// than the constructor of type T.
		m_buffer = std::make_unique<raii::Buffer>(device, describeBuffer());
	}

	BufferDescriptor describeBuffer() { /* ... */ }
};
```

智能指针确保当没有任何东西指向它时（例如，当 `Foo` 被破坏时），它指向的对象被破坏。

```{note}
标准类型 [`std::optional`](https://en.cppreference.com/w/cpp/utility/optional) 也可以是存储“可能的缓冲区”的有趣选项。
```

实施
--------------

那么我们如何编写这个 `raii::Buffer` 包装器呢？看起来很简单，但有**一些需要避免的警告**！

```C++
namespace raii {

// WARNING: This RAII class is not complete.
class Buffer {
public:
	// Whenever a RAII instance is created, we create an underlying resource
	Buffer(wgpu::Device device, const wgpu::BufferDescriptor& bufferDesc)
		: m_raw(device.createBuffer(bufferDesc))
	{}

	// And whenever it gets destroyed, we free the resource
	~Buffer() {
		m_raw.release();
	}

private:
	// Raw resources that is wrapped by the RAII class
	// (Remember that `wgpu::Buffer` is actually just a pointer)
	wgpu::Buffer m_raw;
};

// namespace raii
```

```{caution}
在实现 RAII 习惯用法时，请务必记住，对于任何类，**析构函数不得引发异常**！ WebGPU API 承诺释放对象不会导致异常，所以我们在这里应该没问题。
```

这在某种程度上是有效的......直到我们遇到这种方案：

```C++
raii::Buffer buffer1;

if (something) {
	raii::Buffer buffer2 = buffer1;
}

// Can I still use buffer1 here?
```

在 `if` 块的末尾，变量 `buffer2` 超出范围，因此它的 `m_raw` 成员被释放。除了由于 `=` 赋值而导致 `m_raw` 与 `buffer1` 相同之外！

该代码片段实际上会系统性地因**双重释放错误**而崩溃，因为即使您在 `if` 之后不使用 `buffer1`，每当它超出范围时，其析构函数都会尝试第二次释放 `m_raw`。

### 三法则

C++ 中有一条准确适用于我们的情况的一般经验法则，即 [三规则](https://en.cppreference.com/w/cpp/language/rule_of_three)。简而言之：

> 如果一个类需要一个用户定义的**析构函数**（`~Foo()`）、一个用户定义的**复制赋值**运算符（`operator=(const Foo& other)`）或一个用户定义的**复制构造函数**（`Foo(const Foo& other)`），那么它很可能需要**其中三个**。

在我们的例子中，我们显然需要一个自定义析构函数来释放资源，因此规则告诉我们还应该手动定义**如何复制 RAII 缓冲区**。

```C++
// Rule of three
class Buffer {
public:
	// We define a destructor...
	~Buffer();

	// ...so we need a copy assignment operator...
	Buffer& operator=(const Buffer& other);

	// ...and a copy constructor.
	Buffer(const Buffer& other);

	// [...]
};
```

我们如何实现这些复制操作呢？我们有**三个选择**：

- **选项 A**：我们创建一个新缓冲区并复制前一个缓冲区的内容。
- **选项 B**：我们禁用复制缓冲区的可能性。
- **选项 C**：我们计算参考文献。

选项 A 的问题在于，它将像 `buffer2 = buffer1` 这样看似无害的代码行变成了耗时和内存消耗的操作。它要求缓冲区对象保存对 `Device` 对象的引用，该对象必须用于创建新缓冲区。

选项 B 通过强制使用更明确的方法（例如 `buffer.copyFrom(device, other)`），使 API 用户更清楚。在这种情况下，我们只需删除复制运算符/构造函数：

```C++
// Delete copy semantics
Buffer& operator=(const Buffer& other) = delete;
Buffer(const Buffer& other) = delete;
```

选项 C 包括使用计数器来跟踪有多少不同的 RAII 实例使用相同的 `m_raw`，以便我们仅在没有其他主体使用它时才释放它。

一种可能性是实现选项 B，但随后使用 `std::shared_pointer<raii::Buffer>`，因为共享指针正是带有引用计数器的（智能）指针。

另一种可能性是使用 `buffer.reference()` （或 `wgpuBufferReference`）过程来增加 WebGPU 后端的内部计数器。

### 五法则

定义自定义复制运算符/构造函数会停用 **移动** 运算符/构造函数的自动创建。三规则的一个变体，称为 **五规则**，指出还应该考虑这些移动语义。

这些定义了当我们执行 `buffer2 = std::move(buffer1)` 等操作时会发生什么，这意味着 `buffer1` 将永远不会再次使用。它还使人们能够创建一个返回 `raii::Buffer` 的函数，而无需执行任何复制。

在这种情况下，我们可以简单地将 `m_raw` 的值从 `buffer1` 移动到 `buffer2` ，将其从 `buffer1` 中删除，这样就无法再释放它了。

```C++
Buffer&& operator=(Buffer&& other) {
	m_raw = other.m_raw;
	other.m_raw = nullptr;
}

Buffer(Buffer&& other) {
	m_raw = other.m_raw;
	other.m_raw = nullptr;
}

// And we need to slightly change the destructor:
~Buffer() {
	if (!m_raw) return;
	m_raw.release();
}
```

```{note}
我们创建了一种使 RAII 对象不指向任何底层资源的方法，但这是通过遵循 C++ 生命周期语义的方式完成的，以便编译器可以意识到它。
```

结论
----------

我们已经了解了如何围绕 WebGPU 缓冲区创建 RAII 包装器，并且从这里实现其他类非常简单（您甚至可以将其自动化）。即使对于其他项目也请记住这种设计模式，因为它是一个非常常见且强大的习惯用法！

*结果代码：* [`gist`](https://gist.github.com/eliemichel/2e154152f981e3f16827ba4d17d1123a)
