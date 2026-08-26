命令队列<span class="bullet">🟢</span>
=================

```{translation-warning} 译文可能已过时, /getting-started/the-command-queue.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/015 - The Command Queue
:parent: 010 - The Device
```

*结果代码 :* [`step015`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step015)

让我们先看看最后一个概念,然后打开一个窗口来借鉴:我们在本章中了解到一个**关键概念**WebGPU (以及大多数现代图形 API) 中:**命令队列**.

```{figure} /images/command-queue.png
:align: center
:class: with-shadow
CPU通过命令队列发送命令指示GPU怎么做.
```

不同时限
-------------------

做图形编程时要记住的一件事:**两个同时运行的处理器**其中之一是CPU,又称*主机*,另一个是GPU,或者*设备*。有两个规则:

 1. **我们在CPU上写的代码**,而其中部分会触发GPU上的操作. 唯一的例外是*阴影*,实际上运行在GPU。
2. 处理器是 "**很远**",意味着他们之间的沟通需要时间.

它们并不太远,但对于像实时图形这样的高性能应用来说,这很重要. 在高级管道中,渲染帧可能涉及数以千计或数万计的指令运行在GPU上.

因此,我们无法从CPU中逐个发送命令,并等待每个命令后的反应. 相反,打算用于GPU的命令是通过一个**命令队列**。GPU一旦准备好,就会消耗这个队列,这样处理器可以最大限度地减少其兄弟们回应所花费的时间。

您程序的CPU侧, 即您写的 C++ 代码, 生活在**内容时间表**。命令队列的另一边位于**队列时间表**运行在GPU。

```{note}
还有**设备时间表**定义[WebGPU 文档](https://www.w3.org/TR/webgpu/#programming-model-timelines). 它与GPU操作相对应,我们的代码实际上在等待即时的响应(称为"同步"呼叫),但与JavaScript API不同的是,它与我们C++案例的内容时间表大致相同.
```

队列操作
----------------

我们的WebGPU设备有一个单一的队列,用于发送两者**命令**和**数据**我们可以用它`wgpuDeviceGetQueue`.

```{lit} C++, Get Queue
WGPUQueue queue = wgpuDeviceGetQueue(device);
```

我们还必须释放队列 一旦我们不再使用它, 在程序的结尾:

```{lit} C++, Destroy things (prepend)
// At the end
wgpuQueueRelease(queue);
```

```{note}
**其他图形 API**允许一个构建**多个队列**每个设备,未来版本的WebGPU也可能. 但现在,一个排队已经足够我们玩了!
```

看着`webgpu.h`,我们找到三种不同的方式将工作提交到这个队列:

 - `wgpuQueueSubmit`
 - `wgpuQueueWriteBuffer`
 - `wgpuQueueWriteTexture`

第一个只送来**命令**(虽然可能很复杂),另外两个则发出**数据**从CPU内存(RAM)到GPU 1(VRAM). 来文的拖延可能变得特别重要。

我们还发现`wgpuQueueOnSubmittedWorkDone`程序,我们可以用来设置一个功能,待工作完成后被召回. 让我们做到这一点,以确保事情如预期的那样发生:

```{lit} C++, Add queue callback
auto onQueueWorkDone = [](WGPUQueueWorkDoneStatus status, void* /* pUserData */) {
	std::cout << "Queued work finished with status: " << status << std::endl;
};
wgpuQueueOnSubmittedWorkDone(queue, onQueueWorkDone, nullptr /* pUserData */);
```

````{note}
职能`onQueueWorkDone`这里定义为[羊肉表达式](https://en.cppreference.com/w/cpp/language/lambda)但它也可能是一个正常的功能 之前宣布`main()`,但需有相同的签名:

```C++
void onQueueWorkDone(WGPUQueueWorkDoneStatus status, void* /*pUserData*/) {
	std::cout << "Queued work finished with status: " << status << std::endl;
}
```
````

````{important}
仅**未抓获**羊肉(即与`[]`空)被允许作为回话传递到`wgpuQueueOnSubmittedWorkDone`其他同步操作。 我们必须利用`pUserData`指针(见后几章).
````

提交命令
-------------------

我们使用下列程序提交命令:

```C++
wgpuQueueSubmit(queue, /* number of commands */, /* pointer to the command array */);
```

原来如此**一个典型的偏差**在此 : WebGPU 是 CPI , 所以每当它需要接收一系列东西时, 我们首先提供阵列大小, 然后给第一个元素指针 。

如果我们有一个单一的要素,它只是:

```C++
// With a single command:
WGPUCommandBuffer command = /* [...] */;
wgpuQueueSubmit(queue, 1, &command);
wgpuCommandBufferRelease(command); // release command buffer once submitted
```

如果我们知道**编译时间**("Statistical") 命令的数量,我们可能使用 C 阵列(尽管`std::array`安全吗?

```C++
// With a statically know number of commands:
WGPUCommandBuffer commands[3];
commands[0] = /* [...] */;
commands[1] = /* [...] */;
commands[2] = /* [...] */;
wgpuQueueSubmit(queue, 3, commands);

// or, safer and avoid repeating the array size:
std::array<WGPUCommandBuffer, 3> commands;
commands[0] = /* [...] */;
commands[1] = /* [...] */;
commands[2] = /* [...] */;
wgpuQueueSubmit(queue, commands.size(), commands.data());
```

无论如何,不要忘记**释放**命令缓冲器提交后:

```C++
// Release:
for (auto cmd : commands) {
	wgpuCommandBufferRelease(cmd);
}
```

如果我们需要**动态**改变大小,我们使用`std::vector`:

```C++
std::vector<WGPUCommandBuffer> commands;
// [...] (Allocate and fill in command buffers)
wgpuQueueSubmit(queue, commands.size(), commands.data());
```

然而,我们**无法手动创建**(单位:千美元)`WGPUCommandBuffer`对象。 此缓冲器使用特殊格式, 由您的驱动程序/ 硬件自行决定 。 为了建造这个缓冲器,我们使用**命令编码器**.

命令编码器
---------------

命令编码器按照 WebGPU 的通常对象创建 idiom 创建 :

```{lit} C++, Create Command Encoder
WGPUCommandEncoderDescriptor encoderDesc = {};
encoderDesc.nextInChain = nullptr;
encoderDesc.label = "My command encoder";
WGPUCommandEncoder encoder = wgpuDeviceCreateCommandEncoder(device, &encoderDesc);
```

我们现在可以用编码器来写指令. 由于我们没有任何可操纵的物体,我们现在只能使用简单的调试占位符:

```{lit} C++, Add commands
wgpuCommandEncoderInsertDebugMarker(encoder, "Do one thing");
wgpuCommandEncoderInsertDebugMarker(encoder, "Do another thing");
```

最后从编码器中生成命令也需要一个额外的描述器:

```{lit} C++, Finish encoding and submit
WGPUCommandBufferDescriptor cmdBufferDescriptor = {};
cmdBufferDescriptor.nextInChain = nullptr;
cmdBufferDescriptor.label = "Command buffer";
WGPUCommandBuffer command = wgpuCommandEncoderFinish(encoder, &cmdBufferDescriptor);
wgpuCommandEncoderRelease(encoder); // release encoder after it's finished

// Finally submit the command queue
std::cout << "Submitting command..." << std::endl;
wgpuQueueSubmit(queue, 1, &command);
wgpuCommandBufferRelease(command);
std::cout << "Command submitted." << std::endl;
```

```{lit} C++, Test command encoding (hidden)
{{Get Queue}}
{{Add queue callback}}
{{Create Command Encoder}}
{{Add commands}}
{{Finish encoding and submit}}
{{Poll device}}
```

```{lit} C++, Create things (append, hidden)
{{Test command encoding}}
```

Device polling
--------------

下面介绍设备轮询。

上述代码实际上**失败**与 Dawn 一起使用时 :

```
正在提交命令...
命令提交。
已完成的排队工作状况: 4
```

```{note}
目前的例子很简单,`wgpu-native`实际在设备释放前完成提交的工作。
```

从中可以看出[`webgpu.h`](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h),该值`4`对应于`WGPUQueueWorkDoneStatus_DeviceLost`事实上,我们的计划**终止**在提交命令之后,没有等待命令完成,所以**设备在被销毁之前**工作已经完成了!

所以,我们需要等待一点点,和**重要的是**我们必须打电话**勾选**/**民意调查**以更新其等待的任务。 这是API的一部分**尚未达到标准**因此,我们必须调整我们的实施工作,使之适应后端:

```{lit} C++, Poll device
for (int i = 0 ; i < 5 ; ++i) {
	std::cout << "Tick/Poll device..." << std::endl;
#if defined(WEBGPU_BACKEND_DAWN)
	wgpuDeviceTick(device);
#elif defined(WEBGPU_BACKEND_WGPU)
	wgpuDevicePoll(device, false, nullptr);
#elif defined(WEBGPU_BACKEND_EMSCRIPTEN)
	emscripten_sleep(100);
#endif
}
```

````{important}
从`wgpu-native`拥有非标准功能`wgpu.h`让他们与标准分开`webgpu.h`

```{lit} C++, Includes (append)
#ifdef WEBGPU_BACKEND_WGPU
#  include <webgpu/wgpu.h>
#endif // WEBGPU_BACKEND_WGPU
```
````

我们的程序现在输出类似的东西:

```
正在提交命令...
命令提交。
选中/ Poll 设备...
状态完成的排队工作: 0
选中/ Poll 设备...
选中/ Poll 设备...
选中/ Poll 设备...
选中/ Poll 设备...
```

为了避免使用任意数量的叮当,我们可以设置一个**上下文布尔**改为`true`输入`onQueueWorkDone`并打破循环 一旦它是真的。 不过我们很快会把这叫做主应用循环

结论
----------

我们在本章中看到了一些重要概念:

- CPU和GPU住在这里**不同时限**.
- 命令从CPU流到GPU通过一个**命令队列**.
- 排队命令缓冲器必须使用**命令编码器**.
- 我们必须定期**勾选**/**民意调查**用于更新其等待任务的设备。

这有点抽象,因为尽管我们现在可以排队操作,但他们还没有让我们看到任何东西。 在接下来的章节中,我们会打开一个图形窗口,然后使用我们的队列到**最后显示一些东西**!

```{note}
如果你只对**计算阴影**不需要打开窗户,你可以离开*开始*区域并立即移动到[*基本计算*](../basic-compute/index.md)尽管一些关键概念仍然只引入到[*基本三维渲染*](../basic-3d-rendering/index.md)这部分,像[*使用缓冲器游戏*](../basic-3d-rendering/input-geometry/playing-with-buffers.md)章节。
```

*结果代码 :* [`step015`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step015)
