使用缓冲区 <span class="bullet">🟢</span>
====================

```{translation-warning} 译文可能已过时, /basic-3d-rendering/input-geometry/playing-with-buffers.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/031 - Playing with buffers - vanilla
:parent: 030 - Hello Triangle - vanilla
:alias: Vanilla
```

```{lit-setup}
:tangle-root: zh/031 - Playing with buffers
:parent: 030 - Hello Triangle
```

````{tab} With webgpu.hpp
*结果代码：* [`step031`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step031)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step031-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step031-vanilla)
````

在将顶点数据输入渲染管道之前，我们需要熟悉**缓冲区**的概念。缓冲区“只是”在 **VRAM**（GPU 内存）中分配的**内存块**。将其视为 GPU 的某种 `new` 或 `malloc` 。

在本章中，我们将了解如何**创建**（即分配）、**从CPU写入**、**从GPU复制**到GPU以及**读回**到CPU。

```{note}
请注意，纹理是一种特殊的内存（因为我们通常对它们进行采样的方式），因此它们存在于不同类型的对象中。
```

由于这只是一个实验，所以我建议我们暂时将本章的全部代码写在`Initialize()`函数的末尾。我们代码的总体轮廓如下：

```{lit} C++, Playing with buffers (insert in {{Initialize}} after "InitializePipeline()", also for tangle root "Vanilla")
// Experimentation for the "Playing with buffers" chapter
{{Create a first buffer}}
{{Create a second buffer}}

{{Write input data}}

{{Encode and submit the buffer to buffer copy}}

{{Read buffer data back}}

{{Release buffers}}
```

创建缓冲区
-----------------

缓冲区创建的整体结构现在不会让任何人感到惊讶：描述符和对 `createBuffer` 的调用。

````{tab} With webgpu.hpp
```{lit} C++, Create a first buffer
BufferDescriptor bufferDesc;
bufferDesc.label = "Some GPU-side data buffer";
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::CopySrc;
bufferDesc.size = 16;
bufferDesc.mappedAtCreation = false;
Buffer buffer1 = device.createBuffer(bufferDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create a first buffer (for tangle root "Vanilla")
WGPUBufferDescriptor bufferDesc = {};
bufferDesc.nextInChain = nullptr;
bufferDesc.label = "Some GPU-side data buffer";
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_CopySrc;
bufferDesc.size = 16;
bufferDesc.mappedAtCreation = false;
WGPUBuffer buffer1 = wgpuDeviceCreateBuffer(device, &bufferDesc);
```
````

与 CPU 缓冲区的一个**显着区别**是我们必须声明一些**使用提示**，告诉我们对此内存的使用。例如，如果我们只想使用它从 CPU 写入数据，但从不读回它，则我们将其 `CopyDst` 使用标志设置为打开，但不设置 `CopySrc` 标志。这种不完全不可知的内存管理**帮助设备**找出最佳的内存布局。

```{note}
当 GPU 缓冲区连接到 CPU 端 RAM 的特定部分时，它就会被**映射**。然后，驱动程序会自动同步其内容，以进行读取或写入。我们暂时不使用此功能。
```

对于我们的小练习，让我们**创建第二个缓冲区**，名为 `buffer2`。我们将在第一个缓冲区中加载数据，发出**复制**命令，以便 GPU 将数据从一个缓冲区复制到另一个缓冲区，然后**读**回目标缓冲区。

我们可以重用描述符，现在只改变标签：

````{tab} With webgpu.hpp
```{lit} C++, Create a second buffer
bufferDesc.label = "Output buffer";
Buffer buffer2 = device.createBuffer(bufferDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create a second buffer (for tangle root "Vanilla")
bufferDesc.label = "Output buffer";
WGPUBuffer buffer2 = wgpuDeviceCreateBuffer(device, &bufferDesc);
```
````

另外，一旦不再使用缓冲区，不要忘记**释放它们：

````{tab} With webgpu.hpp
```{lit} C++, Release buffers
// In Terminate()
buffer1.release();
buffer2.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Release buffers (for tangle root "Vanilla")
// In Terminate()
wgpuBufferRelease(buffer1);
wgpuBufferRelease(buffer2);
```
````

`````{note}
缓冲区（以及纹理）还提供 **destroy** 方法：

````{tab} With webgpu.hpp
```C++
buffer1.destroy();
```
````

````{tab} Vanilla webgpu.h
```C++
wgpuBufferDestroy(buffer1);
```
````

即使仍然存在对缓冲区的引用，这也可用于**强制释放** GPU 内存。

- **销毁**释放为缓冲区分配的 GPU 内存，但位于驱动程序/后端端的缓冲区对象本身仍然存在。
- **释放** 释放驱动程序/后端对象（或者更确切地说减少其引用指针）并在没有其他人使用它时销毁它。
`````

写入缓冲区
-------------------

设备队列提供了一个 `queue.writeBuffer` 函数（或 C 风格的 `wgpuQueueWriteBuffer`），我们首先向其提供要写入的 **GPU 缓冲区**，然后是从中复制数据的 CPU 端 **内存地址** 和 **大小**：

````{tab} With webgpu.hpp
```{lit} C++, Write input data
// Create some CPU-side data buffer (of size 16 bytes)
std::vector<uint8_t> numbers(16);
for (uint8_t i = 0; i < 16; ++i) numbers[i] = i;
// `numbers` now contains [ 0, 1, 2, ... ]

// Copy this from `numbers` (RAM) to `buffer1` (VRAM)
queue.writeBuffer(buffer1, 0, numbers.data(), numbers.size());
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Write input data (for tangle root "Vanilla")
// Create some CPU-side data buffer (of size 16 bytes)
std::vector<uint8_t> numbers(16);
for (uint8_t i = 0; i < 16; ++i) numbers[i] = i;
// `numbers` now contains [ 0, 1, 2, ... ]

// Copy this from `numbers` (RAM) to `buffer1` (VRAM)
wgpuQueueWriteBuffer(queue, buffer1, 0, numbers.data(), numbers.size());
```
````

```{note}
将数据从 CPU 端内存 (RAM) 上传到 GPU 端内存 (VRAM) **需要时间**。当函数 `writeBuffer()` 返回时，数据传输可能尚未完成，但**保证**是：

- 您可以从刚刚传递的地址**释放内存**，因为后端在传输期间维护自己的 CPU 端缓冲区副本（如果您想避免这种情况，请使用映射）。

- 在`writeBuffer()`操作之后**在队列中提交的命令在数据传输完成之前不会被执行。

并且不要忘记，通过 **命令编码器** 发送的命令仅在使用 `encoder.finish()` 返回的编码命令缓冲区调用 `queue.submit()` 时才会提交。
```

复制缓冲区
----------------

我们现在可以向命令队列提交**缓冲区-缓冲区复制**操作。这不能直接从队列对象中获得，而是需要我们**创建一个命令编码器**。一旦我们有了编码器，我们可以简单地添加以下内容：

````{tab} With webgpu.hpp
```{lit} C++, Copy buffer to buffer
// After creating the command encoder
encoder.copyBufferToBuffer(buffer1, 0, buffer2, 0, 16);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Copy buffer to buffer (for tangle root "Vanilla")
// After creating the command encoder
wgpuCommandEncoderCopyBufferToBuffer(encoder, buffer1, 0, buffer2, 0, 16);
```
````

每个缓冲区后面的参数 `0` 是缓冲区内必须进行复制的**字节偏移量**。这使得可以复制缓冲区的**子部分**。

我们将其包装在类似于渲染通道的命令编码过程中：

````{tab} With webgpu.hpp
```{lit} C++, Encode and submit the buffer to buffer copy
CommandEncoder encoder = device.createCommandEncoder(Default);

{{Copy buffer to buffer}}

CommandBuffer command = encoder.finish(Default);
encoder.release();
queue.submit(1, &command);
command.release();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Encode and submit the buffer to buffer copy (for tangle root "Vanilla")
WGPUCommandEncoder encoder = wgpuDeviceCreateCommandEncoder(device, nullptr);

{{Copy buffer to buffer}}

WGPUCommandBuffer command = wgpuCommandEncoderFinish(encoder, nullptr);
wgpuCommandEncoderRelease(encoder);
wgpuQueueSubmit(queue, 1, &command);
wgpuCommandBufferRelease(command);
```
````

从缓冲区读取
---------------------

我们用来发送数据 (`writeBuffer`) 和指令 (`copyBufferToBuffer`) 的 **命令队列**，**仅以一种方式**：从 CPU 主机到 GPU 设备。因此，它是一个“即发即忘”队列：函数不会返回值，因为它们**在不同的时间轴上运行**。

那么，我们如何读取数据呢？我们使用**异步操作**，就像我们在[命令队列](../../getting-started/the-command-queue.md)章节中使用`wgpuQueueOnSubmittedWorkDone`时所做的那样。我们没有直接获取值，而是设置了一个**回调**，只要请求的数据准备就绪，就会调用该回调。然后我们**轮询设备**以检查传入事件。

**要从缓冲区读取数据**，我们使用 `buffer.mapAsync` （或 `wgpuBufferMapAsync`）。此操作将 GPU 缓冲区映射到 CPU 内存，然后每当准备就绪时，它就会执行所提供的回调函数。完成后，我们可以**取消映射**缓冲区。

```{note}
这种**异步性**使编程工作流程比同步操作**更加复杂**，但最大限度地减少浪费的处理器空闲非常重要。通常启动映射操作，然后**在等待**数据时执行其他操作（与运行 CPU 指令相比，这需要花费大量时间）。
```

### 映射

让我们首先通过添加 `BufferUsage::MapRead` 标志来更改 **第二个缓冲区** 的 `usage` ，以便可以映射缓冲区以进行读取：

````{tab} With webgpu.hpp
```{lit} C++, Create a second buffer (replace)
bufferDesc.label = "Output buffer";
bufferDesc.usage = BufferUsage::CopyDst | BufferUsage::MapRead;
Buffer buffer2 = device.createBuffer(bufferDesc);
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Create a second buffer (replace, for tangle root "Vanilla")
bufferDesc.label = "Output buffer";
bufferDesc.usage = WGPUBufferUsage_CopyDst | WGPUBufferUsage_MapRead;
WGPUBuffer buffer2 = wgpuDeviceCreateBuffer(device, &bufferDesc);
```
````

```{note}
`BufferUsage::MapRead` 标志与 `BufferUsage::CopySrc` 标志不兼容，因此请确保不要同时拥有两者。创建专用于映射操作的**缓冲区是很常见的。
```

我们现在可以通过简单的回调来调用缓冲区映射。 `wgpuBufferMapAsync` 过程将**映射模式**（读、写或两者）、要映射的缓冲区数据的**切片**作为参数，由偏移量 (0) 和字节数 (16) 给出，然后是 **回调**，最后是一些 **“用户数据”** 指针。我们在下面展示](#mapping-context) 后者的用途。

````{tab} With webgpu.hpp
```C++
auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* /* pUserData */) {
	std::cout << "Buffer 2 mapped with status " << status << std::endl;
};
wgpuBufferMapAsync(buffer2, MapMode::Read, 0, 16, onBuffer2Mapped, nullptr /*pUserData*/);
```
````

````{tab} Vanilla webgpu.h
```C++
auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* /* pUserData */) {
	std::cout << "Buffer 2 mapped with status " << status << std::endl;
};
wgpuBufferMapAsync(buffer2, WGPUMapMode_Read, 0, 16, onBuffer2Mapped, nullptr /*pUserData*/);
```
````

```{important}
我现在有意使用 **C 风格的过程**，它有助于理解使用 C++ 包装器提供的更简单的 API 时幕后发生的情况。
```

### 异步轮询

如果您此时运行该程序，您可能会惊讶（并且失望）地发现回调**从未执行**！我们在命令队列章节的[设备轮询](../../getting-started/the-command-queue.md#device-polling)]部分看到了这一点：WebGPU 库没有执行任何隐藏进程来**检查异步操作是否已准备好**，因此我们必须自己执行此操作。

不幸的是，这个机制还没有标准的解决方案，所以我们对 Dawn、`wgpu-native` 和 emscripten 的写法不同，对后者有一点微妙：

````{tab} With webgpu.hpp
```{lit} C++, Define the wgpuPollEvents function
// We define a function that hides implementation-specific variants of device polling:
void wgpuPollEvents([[maybe_unused]] Device device, [[maybe_unused]] bool yieldToWebBrowser) {
#if defined(WEBGPU_BACKEND_DAWN)
	device.tick();
#elif defined(WEBGPU_BACKEND_WGPU)
	device.poll(false);
#elif defined(WEBGPU_BACKEND_EMSCRIPTEN)
	if (yieldToWebBrowser) {
		emscripten_sleep(100);
	}
#endif
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Define the wgpuPollEvents function (for tangle root "Vanilla")
// We define a function that hides implementation-specific variants of device polling:
void wgpuPollEvents([[maybe_unused]] WGPUDevice device, [[maybe_unused]] bool yieldToWebBrowser) {
#if defined(WEBGPU_BACKEND_DAWN)
	wgpuDeviceTick(device);
#elif defined(WEBGPU_BACKEND_WGPU)
	wgpuDevicePoll(device, false, nullptr);
#elif defined(WEBGPU_BACKEND_EMSCRIPTEN)
	if (yieldToWebBrowser) {
		emscripten_sleep(100);
	}
#endif
}
```
````

```{lit} C++, Global declarations (hidden, append, also for tangle root "Vanilla")
{{Define the wgpuPollEvents function}}
```

```{admonition} Emscripten subtlety
当我们的 C++ 代码在 **Web 浏览器** 中运行时（通过 emscripten 编译为 WebAssembly 之后），**没有明确的方式**来勾选/轮询 WebGPU 设备。这是因为 **设备由 Web 浏览器** 本身管理，它决定轮询应该发生的速度。结果：

- 设备**不会在 WebAssembly 模块的两个连续行之间滴答**，它只能在执行流离开模块时滴答。
- 设备**总是在两次调用我们的 `MainLoop()` 函数之间滴答作响**，因为如果您还记得打开窗口一章的 [Emscripten](../../getting-started/opening-a-window.md#emscripten) 部分，我们将主循环管理留给浏览器，只提供在每个帧运行的回调。

由于第二点，在主循环开始或结束时调用 `wgpuPollEvents` 时，我们不需要执行任何操作（因此我们将 `yieldToWebBrowser` 设置为 false）。

然而，如果我们真正的目的是**等到**发生某些事情（例如，调用回调），那么第一点要求我们确保**将执行流返回**到Web浏览器，以便它可以不时地勾选其设备。我们这样做要归功于 `emscripten_sleep` 函数，但代价是在 100 毫秒内有效睡眠（我们处于无论如何都想等待的情况）。

请注意，使用 `emscripten_sleep` 需要将 `-SASYNCIFY` 链接选项传递给 emscripten，就像我们添加的 [already](../../getting-started/adapter-and-device/the-adapter.md#waiting-for-the-request-to-end).
```

在我们的示例中，我们希望在主循环迭代期间等待读回，因此我们指定返回给浏览器：

```C++
bool ready = false;

{{Define callback and start mapping buffer}}

while (!ready) {
	wgpuPollEvents(device, true /* yieldToBrowser */);
}
```

现在，在运行程序时，您可以看到 `Buffer 2 mapped with status 1` （使用 Dawn 时 `BufferMapAsyncStatus::Success` 的值为 1，WGPU 的值为 0）。 **但是**，我们永远不会将 `ready` 变量更改为 `true`！所以程序就会**永远挂起**...不太好。这就是为什么下一节将展示如何将一些上下文传递给回调。

### Mapping context

下面介绍如何传递映射上下文。

因此，我们需要回调来**访问和改变** `ready` 变量。但是，既然 `onBuffer2Mapped` 是一个单独的函数，其签名无法更改，我们该如何做到这一点呢？我们可以使用**用户指针**，就像我们在[适配器请求](../../getting-started/adapter-and-device/the-adapter.md)或[设备请求](../../getting-started/adapter-and-device/the-device.md)中所做的那样。

```{note}
当将 `onBuffer2Mapped` 定义为常规函数时，很明显 `ready` 是不可访问的。当像我们上面那样使用 lambda 表达式时，人们可能会想在 **捕获列表** （函数参数之前的括号）中添加 `ready` 。但这**不起作用**，因为捕获 lambda 具有**不同的类型**，不能用作常规回调。我们在下面看到 C++ 包装器修复了这个限制。
```

**用户指针**是在设置回调时提供给 `wgpuBufferMapAsync` 的参数，然后在映射操作准备就绪时**按原样**提供给回调 `onBuffer2Mapped` 。缓冲区仅转发此指针，但从不使用它：**只有您**（API 的用户）解释它。

````{tab} With webgpu.hpp
```C++
// A first use of the 'pUserData' argument.
bool ready = false;

auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* pUserData) {
	// We know by convention with ourselves that the user data is a pointer to 'ready':
	bool* pReady = reinterpret_cast<bool*>(pUserData);
	// We set ready to 'true'
	*pReady = true;

	std::cout << "Buffer 2 mapped with status " << status << std::endl;
};

wgpuBufferMapAsync(buffer2, MapMode::Read, 0, 16, onBuffer2Mapped, (void*)&ready);
//                                Pass the address of 'ready' here: ^^^^^^^^^^^^
```
````

````{tab} Vanilla webgpu.h
```C++
// A first use of the 'pUserData' argument.
bool ready = false;

auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* pUserData) {
	// We know by convention with ourselves that the user data is a pointer to 'ready':
	bool* pReady = reinterpret_cast<bool*>(pUserData);
	// We set ready to 'true'
	*pReady = true;

	std::cout << "Buffer 2 mapped with status " << status << std::endl;
};

wgpuBufferMapAsync(buffer2, WGPUMapMode_Read, 0, 16, onBuffer2Mapped, (void*)&ready);
//                                   Pass the address of 'ready' here: ^^^^^^^^^^^^
````

Now, we need to **access to more than a status** when running this callback, since we need to access the buffer's content. But the `onBuffer2Mapped` function cannot have a second user pointer! Not a problem: we can define **a context structure**, that holds **all fields that we want to share** with the callback, then pass the address of an instance of this context.

````{tab} With webgpu.hpp
```{lit} C++, Read buffer data back
// The context shared between this main function and the callback.
struct Context {
	bool ready;
	Buffer buffer;
};

auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* pUserData) {
	Context* context = reinterpret_cast<Context*>(pUserData);
	context->ready = true;
	std::cout << "Buffer 2 mapped with status " << status << std::endl;
	if (status != BufferMapAsyncStatus::Success) return;

	{{Use context->buffer here}}
};

// Create the Context instance
Context context = { false, buffer2 };

wgpuBufferMapAsync(buffer2, MapMode::Read, 0, 16, onBuffer2Mapped, (void*)&context);
//                   Pass the address of the Context instance here: ^^^^^^^^^^^^^^

while (!context.ready) {
	//  ^^^^^^^^^^^^^ Use context.ready here instead of ready
	wgpuPollEvents(device, true /* yieldToBrowser */);
}
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Read buffer data back (for tangle root "Vanilla")
// The context shared between this main function and the callback.
struct Context {
	bool ready;
	WGPUBuffer buffer;
};

auto onBuffer2Mapped = [](WGPUBufferMapAsyncStatus status, void* pUserData) {
	Context* context = reinterpret_cast<Context*>(pUserData);
	context->ready = true;
	std::cout << "Buffer 2 mapped with status " << status << std::endl;
	if (status != WGPUBufferMapAsyncStatus_Success) return;

	{{Use context->buffer here}}
};

// Create the Context instance
Context context = { false, buffer2 };

wgpuBufferMapAsync(buffer2, WGPUMapMode_Read, 0, 16, onBuffer2Mapped, (void*)&context);
//                      Pass the address of the Context instance here: ^^^^^^^^^^^^^^

while (!context.ready) {
	//  ^^^^^^^^^^^^^ Use context.ready here instead of ready
	wgpuPollEvents(device, true /* yieldToBrowser */);
}
```
````

```{tip}
当整个操作位于像 `Application` 这样的类中时，只需 **使用 `this` 作为用户指针**，然后在回调中检索整个应用程序对象就可以很方便。
```

### 使用映射缓冲区

一旦缓冲区被映射（直接在回调中或在确保其准备就绪的轮询循环之后），我们使用 `Buffer::getConstMappedRange` （又名 `wgpuBufferGetConstMappedRange`）来**获取指向 CPU 端数据的指针**。一旦我们处理完 CPU 端数据，我们必须**取消映射**缓冲区：

````{tab} With webgpu.hpp
```{lit} C++, Use context->buffer here
// Get a pointer to wherever the driver mapped the GPU memory to the RAM
uint8_t* bufferData = (uint8_t*)context->buffer.getConstMappedRange(0, 16);

{{Do stuff with bufferData}}

// Then do not forget to unmap the memory
context->buffer.unmap();
```
````

````{tab} Vanilla webgpu.h
```{lit} C++, Use context->buffer here (for tangle root "Vanilla")
// Get a pointer to wherever the driver mapped the GPU memory to the RAM
uint8_t* bufferData = (uint8_t*)wgpuBufferGetConstMappedRange(context->buffer, 0, 16);

{{Do stuff with bufferData}}

// Then do not forget to unmap the memory
wgpuBufferUnmap(context->buffer);
```
````

```{note}
在**写入模式**下映射缓冲区时，请使用 `Buffer::getMappedRange` （又名 `wgpuBufferGetMappedRange`）而不是“const”版本。
```

例如，我们可以**显示缓冲区的内容**并检查它是否对应于我们最初输入的缓冲区数据：

```{lit} C++, Do stuff with bufferData (also for tangle root "Vanilla")
std::cout << "bufferData = [";
for (int i = 0; i < 16; ++i) {
	if (i > 0) std::cout << ", ";
	std::cout << (int)bufferData[i];
}
std::cout << "]" << std::endl;
```

结论
----------

恭喜！我们能够创建一个 GPU 端内存缓冲区，将数据上传到其中，使用命令队列远程复制它（操作由 CPU 触发，但在 GPU 上执行），然后从 GPU 下载数据。我们现在可以使用缓冲区来指定顶点属性，特别是顶点位置！

````{tab} With webgpu.hpp
*结果代码：* [`step031`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step031)
````

````{tab} Vanilla webgpu.h
*结果代码：* [`step031-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step031-vanilla)
````
