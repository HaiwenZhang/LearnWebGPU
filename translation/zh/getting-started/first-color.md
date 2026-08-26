第一个颜色<span class="bullet">🟢</span>
===========

```{translation-warning} 译文可能已过时, /getting-started/first-color.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

```{lit-setup}
:tangle-root: zh/025 - First Color
:parent: 020 - Opening a window
```

*结果代码 :* [`step025`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step025)

本章的目标是:**绘制坚实的颜色**在我们的整个窗口。 为此,我们增加了以下步骤:

1. 我们必须首先**配置**联合国*表面*我们的窗口。
2. 然后在每个框架上,我们得到**表面纹理**来吸引。
3. 最后,我们创建了**传送通道**有效绘制一些东西。

表面配置
---------------------

在前一章末尾,我们介绍了**表面**对象为OS窗口(由GLFW管理)和WebGPU实例之间的链接。

然而,这个表面需要**配置**在我们能够利用它之前。 为了了解原因,我们需要多了解一下窗户的表面是如何绘制的。

###绘图过程

第一,导轨管道**不直接绘制当前显示的纹理**,否则我们就会看到像素的改变。 一个典型的管道引向**屏幕外纹理**,它仅在完成后替换当前显示的。 然后我们说纹理是**提交**到达地面

第二,画一个**不同时间**而不是应用程序所需的帧率,所以GPU可能不得不等待到下一个帧需要. 在要呈现的队列中可能会有不止一个屏幕外的纹理等待,这样渲染时间的波动就会被摊销.

最后一个**这些外屏纹理被重用**尽可能多。 一旦呈现出新的纹理,前一个纹理就可以被重用作为下一个框架的目标. 这整个机制叫做**交换链**和处理在引擎盖下 由**表面**对象。

```{note}
记住GPU进程以自己的速度运行,我们CPU发布的命令只是同步执行的. 因此,手动实施互换链程序需要大量的锅炉板,所以我们很高兴它是由API提供的!
```

<video autoplay loop muted inline nocontrols style="width:100%;height:auto;max-width:960px">
    <source src="../_static/swapchain.mp4" type="video/mp4">
</video>

<p class="align-center">
    <span class="caption-text"><em>左侧: 渲染过程借鉴了屏幕外的纹理 。 中间:纹理在队列中等待. 对: 按常规帧速率,制成的纹理呈现在窗面.</em></span>
</p>

###配置

我们刚才描述的过程 有一些参数 我们设定了`wgpuSurfaceConfigure`,它的作用有点像一个对象的创建:

```{lit} C++, Surface Configuration
WGPUSurfaceConfiguration config = {};
config.nextInChain = nullptr;

{{Describe Surface Configuration}}

wgpuSurfaceConfigure(surface, &config);
```

一定要这样**初始化结束时**:

```{lit} C++, Initialize (append, hidden)
{{Surface Configuration}}
```

在程序结束时,我们可以解析表面:

```{lit} C++, Terminate (prepend)
wgpuSurfaceUnconfigure(surface);
```

####纹理参数

我们必须首先具体说明用于**分配纹理**基础交换链。 这当然包括a**大小**(我们设置到窗口大小),但也有一个**格式**备注a**使用量**.

```{lit} C++, Describe Surface Configuration
// Configuration of the textures created for the underlying swap chain
config.width = 640;
config.height = 480;
{{Describe Surface Usage}}
{{Describe Surface Format}}
```

```{warning}
你猜得到,我们得重新配置地面**当窗口调整大小时**同时,不要试图调整其大小。 您可以添加`glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);`在创建窗口之前指示 GLFW 禁用重新大小。
```

那个**格式**是一个组合**频道数**(红、绿、蓝、α的子集),a**每个频道大小**(8,16或32比特)和a**频道类型**(浮动,整数,签名与否),压缩方案,正常化模式等.

所有可用的组合均列于`WGPUTextureFormat`enum,但由于我们的互换链 瞄准一个现有的表面, 我们可以只是使用 任何格式的表面使用:

```{lit} C++, Describe Surface Format
WGPUTextureFormat surfaceFormat = wgpuSurfaceGetPreferredFormat(surface, adapter);
config.format = surfaceFormat;
// And we do not need any particular view format:
config.viewFormatCount = 0;
config.viewFormats = nullptr;
```

````{warning}
记得把电话移到`wgpuAdapterRelease` **之后**电话呼叫`wgpuSurfaceGetPreferredFormat`由于后者使用我们`adapter`手柄。

```{lit} C++, Initialize (hidden, replace)
{{Open window and get adapter}}

{{Request device}}
queue = wgpuDeviceGetQueue(device);

{{Add device error callback}}

{{Surface Configuration}}

// We no longer need to access the adapter
wgpuAdapterRelease(adapter);
```
````

纹理被分配到一个**具体用途**这决定了GPU组织记忆的方式. 就我们来说,我们用交换链纹理作为目标*传送通道*因此它需要与`RenderAttachment`使用标记 :

```{lit} C++, Describe Surface Usage
config.usage = WGPUTextureUsage_RenderAttachment;
```

最后,表面需要知道用于创建纹理的设备:

```{lit} C++, Describe Surface Configuration (append)
config.device = device;
```

####演示文稿参数

在告诉如何分配纹理后,我们可以分辨出等待队列中的哪些纹理必须在每个框架上呈现. 可能值见于`WGPUPresentMode`语句 :

 - `Immediate`:不使用屏外纹理,渲染过程直接引出表面,可能导致文物(称为:*撕裂*),但有零延迟.
 - `Mailbox`:队列中只有一个插槽,当一个新框架被渲染出来时,它取代了目前等待的(它被丢弃而从未被呈现).
 - `Fifo`:支持"先入后出",意为呈现的纹理总是最古老的纹理,如常规排队. 没有使纹理浪费。

```{tip}
那个`Force32`读取源代码时可以找到的 enum 值`webgpu.h`这不是一个"合法"的值,它只是在这里强迫基础的enum类型是32位整数.
```

在我们的情况下,我们使用`Fifo`如以上视频所示。

```{lit} C++, Describe Surface Configuration (append)
config.presentMode = WGPUPresentMode_Fifo;
```

最后,我们可以具体说明如何将纹理合成到OS窗口上,该窗口可用于创建**透明**窗口。 我们还可以简单地把它留给汽车模式:

```{lit} C++, Describe Surface Configuration (append)
config.alphaMode = WGPUCompositeAlphaMode_Auto;
```

```{admonition} Troubleshooting
如果你得到错误`Uncaptured device error: type 3 (Device(OutOfMemory))`呼叫时`wgpuSurfaceConfigure`中,检查您是否指定了`GLFW_NO_API`创建窗口时值为 glfw。
```

表面纹理
---------------

水面已经布置好了 我们可以问**每个框架**联 合 国**下一个可用的纹理**在互换链中,即我们必须绘制的纹理。 总体而言,我们的主要循环内容如下:

```{lit} C++, Main loop content (replace)
// In Application::MainLoop()
{{Get the next target texture view}}
{{Draw things}}
{{Present the surface onto the window}}
```

因为我们通常需要的是**纹理视图**而不是原始的表面纹理, 我们可能创建一个专用功能`GetNextSurfaceViewData()`我们的应用课。

```{lit} C++, GetNextSurfaceViewData method
std::pair<WGPUSurfaceTexture, WGPUTextureView> Application::GetNextSurfaceViewData() {
    {{Get the next surface texture}}
    {{Create surface texture view}}
    {{Release the texture}}
    return { surfaceTexture, targetView };
}
```

```{lit} C++, Application implementation (append, hidden)
{{GetNextSurfaceViewData method}}
```

然后我们简单地在主循环开头调用此函数, 并检查它是否返回一个有效的视图 :

```{lit} C++, Get the next target texture view
// Get the next target texture view
auto [ surfaceTexture, targetView ] = GetNextSurfaceViewData();
if (!targetView) return;
```

````{note}
别忘了**声明方法**输入`Application`阶级宣言。 这是我们的软件的内部设备 所以我们就用这种方法**私营**:

```{lit} C++, Private methods (insert in {{Application class}} after "bool IsRunning();")
private:
    std::pair<WGPUSurfaceTexture, WGPUTextureView> GetNextSurfaceViewData();
```
````

###获得下一个目标纹理

为了让纹理吸引我们`wgpuSurfaceGetCurrentTexture`。“表面纹理”并不是真正的物体,而是一种**容器**联 合 国**此函数返回的多个事项**因此,我们有责任建立`WGPUSurfaceTexture`容器,我们通过这个功能将其写入:

```{lit} C++, Get the next surface texture
WGPUSurfaceTexture surfaceTexture;
wgpuSurfaceGetCurrentTexture(surface, &surfaceTexture);
```

然后,我们获得以下信息:

 - `surfaceTexture.status`告诉我们行动是不是**已经成功**,如果不给一些提示为什么。
 - `surfaceTexture.suboptimal`可能还要指出的是,尽管成功地回收了纹理,但基础表面发生了变化,我们也许应该**重新配置**这个
 - `surfaceTexture.texture`这是**纹理**我们必须利用**在此框架中**.

我们只处理明显失败的案件,现在却忽略了不理想的旗帜:

```{lit} C++, Get the next surface texture (append)
if (surfaceTexture.status != WGPUSurfaceGetCurrentTextureStatus_Success) {
    return { surfaceTexture, nullptr };
}
```

###纹理视图

下一节我们需要的不是表面纹理 而是**纹理视图**,它可能代表纹理的一个子部分,或者使用不同的格式将其曝光。 我们会回来的 纹理观点[纹理](/basic-3d-rendering/texturing/index.md)本指南部分,现可复制-粘贴以下锅炉板:

```{lit} C++, Create surface texture view
WGPUTextureViewDescriptor viewDescriptor;
viewDescriptor.nextInChain = nullptr;
viewDescriptor.label = "Surface texture view";
viewDescriptor.format = wgpuTextureGetFormat(surfaceTexture.texture);
viewDescriptor.dimension = WGPUTextureViewDimension_2D;
viewDescriptor.baseMipLevel = 0;
viewDescriptor.mipLevelCount = 1;
viewDescriptor.baseArrayLayer = 0;
viewDescriptor.arrayLayerCount = 1;
viewDescriptor.aspect = WGPUTextureAspect_All;
WGPUTextureView targetView = wgpuTextureCreateView(surfaceTexture.texture, &viewDescriptor);
```

当然,一旦我们不再需要这种观点,我们就必须在提出之前发表这种观点:

```{lit} C++, Present the surface onto the window
// At the end of the frame
wgpuTextureViewRelease(targetView);
```

纹理本身必须释放出来,实际上我们在创建纹理视图后可以做正确的事情,因为后者会保持自己对纹理的参考,以免它被过早销毁:

```{lit} C++, Release the texture
#ifndef WEBGPU_BACKEND_WGPU
    // We no longer need the texture, only its view
    // (NB: with wgpu-native, surface textures must be release after the call to wgpuSurfacePresent)
    wgpuTextureRelease(surfaceTexture.texture);
#endif // WEBGPU_BACKEND_WGPU
```

###介绍

最后,一旦纹理被填充并释放出来,我们可以告诉表面呈现其互换链的下一个纹理(这也许是也可能不是我们刚刚绘制的纹理,取决于`presentMode`):

```{lit} C++, Present the surface onto the window (append)
wgpuSurfacePresent(surface);

#ifdef WEBGPU_BACKEND_WGPU
    wgpuTextureRelease(surfaceTexture.texture);
#endif
```

````{admonition} Building for the Web
A. 涉及**网页浏览器**,我们做**没有**呈现出表面的纹理 我们宁愿依靠`emscripten_set_main_loop_arg`(中文(简体) ).`requestAnimationFrame`在 JavaScript 中调用`MainLoop()`函数正前方显示。

因此,我们**绝对不能**电话`wgpuSurfacePresent()`使用脚本构建时 :

```{lit} C++, Present the surface onto the window (replace)
// At the end of the frame
wgpuTextureViewRelease(targetView);
#ifndef __EMSCRIPTEN__
wgpuSurfacePresent(surface);
#endif
```
````

传送通道
-----------

###通过编码器

我们现在握着纹理来绘制,以便在我们的窗口中展示一些东西. 像任何GPU侧操作一样,我们触发从**命令队列**,使用命令编码器,如[命令队列](the-command-queue.md).

我们建一个`WGPUCommandEncoder`调用`encoder`,然后将其提交队列。 在两者之间,我们将增加一个命令,以统一的颜色清除屏幕。

```{lit} C++, Draw things
{{Create Command Encoder}}
{{Encode Render Pass}}
{{Finish encoding and submit}}
```

如果你往里面看的话`webgpu.h`在编码器的方法(从`wgpuCommandEncoder`),大多数与周围复制缓冲和纹理有关. 例外情况是:**两种特殊方法**: `wgpuCommandEncoderBeginComputePass`和`wgpuCommandEncoderBeginRenderPass`。这些返回**专用编码器对象**,即:`WGPUComputePassEncoder`和`WGPURenderPassEncoder`,从而可以访问分别用于**计算**和**三维渲染**.

就我们来说,我们用的是**传递**:

```{lit} C++, Encode Render Pass
WGPURenderPassDescriptor renderPassDesc = {};
renderPassDesc.nextInChain = nullptr;

{{Describe Render Pass}}

WGPURenderPassEncoder renderPass = wgpuCommandEncoderBeginRenderPass(encoder, &renderPassDesc);
{{Use Render Pass}}
wgpuRenderPassEncoderEnd(renderPass);
wgpuRenderPassEncoderRelease(renderPass);
```

请注意,我们直接结束 通过**未印发**任何其他命令。 这是因为 渲染通道有内在机制**清除屏幕**当它开始的时候,我们将通过文字描述来建立它。

```{lit} C++, Use Render Pass (hidden)
// Use the render pass here (we do nothing with the render pass for now)
```

###颜色附件

渲染通过会利用GPU的3D渲染电路将内容引向一个或多个纹理. 所以,一个重要的事情 设置是告诉**哪些纹理是目标**这一进程。 这些是**附件**代表的通行证。

附件的数量是可变的,所以描述符通过两个字段:`colorAttachmentCount`附录和地址`colorAttachments`颜色附件数组。 自从我们**只用一个**这里,数组的地址只是单个的地址`WGPURenderPassColorAttachment`变量。

```{lit} C++, Describe Render Pass
WGPURenderPassColorAttachment renderPassColorAttachment = {};

{{Describe the attachment}}

renderPassDesc.colorAttachmentCount = 1;
renderPassDesc.colorAttachments = &renderPassColorAttachment;
```

该附件的第一个重要设置是:**纹理视图**它必须引来。

就我们来说,这只是`targetView`我们从表面得到的,因为我们想**直接绘制屏幕**但是,在先进的管道中,利用**中间纹理**,然后输入后处理通行证。

```{lit} C++, Describe the attachment
renderPassColorAttachment.view = targetView;
```

有第二个目标纹理视图叫做`resolveTarget`,但这与这里无关,因为我们不使用**多采样**(稍后再谈这个).

```{lit} C++, Describe the attachment (append)
renderPassColorAttachment.resolveTarget = nullptr;
```

`loadOp` 设置表示在**执行渲染通道之前**对纹理视图执行的加载操作。它可以从视图中读取现有内容，也可以用默认的统一颜色（即清除值）填充视图。**如果现有内容并不重要**，请选择 `WGPULoadOp_Clear`，因为这样通常更高效。

`storeOp` 表示在**执行渲染通道之后**对纹理视图执行的操作。结果既可以保存，也可以丢弃（后者只有在渲染通道本身具有其他副作用时才有意义）。

那个`clearValue`值为**清除屏幕**随你便 随便你 这四个值是**红色**, **绿色**, **蓝色**和**α**频道,规模**从0.0到1.0**.

```{lit} C++, Describe the attachment (append)
renderPassColorAttachment.loadOp = WGPULoadOp_Clear;
renderPassColorAttachment.storeOp = WGPUStoreOp_Store;
renderPassColorAttachment.clearValue = WGPUColor{ 0.9, 0.1, 0.2, 1.0 };
```

有最后一个成员`depthSlice`要在附件中设定,我们必须明确设定其未定义的值,因为我们没有使用深度缓冲。 此选项不支持`wgpu-native`现在,所以我们把这个 在一个`#ifdef`:

```{lit} C++, Describe the attachment (append)
#ifndef WEBGPU_BACKEND_WGPU
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
#endif // NOT WEBGPU_BACKEND_WGPU
```

###杂项

还有一个特殊的附着类型,即:**深度**和**调色板**附件(它是一个可能包含两个信道的单一附件)。 我们以后再来讨论,因为现在我们不使用它 所以我们把它设定为无效:

```{lit} C++, Describe Render Pass (append)
renderPassDesc.depthStencilAttachment = nullptr;
```

何时**衡量业绩**由于命令没有同步执行,因此无法使用 CPU 边计时函数。 相反,渲染证可以接收一组时间戳查询. 我们不用这个例子(see advanced chapter about [Benchmarking Time](/advanced-techniques/benchmarking/time.md)更多信息)。

```{lit} C++, Describe Render Pass (append)
renderPassDesc.timestampWrites = nullptr;
```

结论
----------

在这个阶段,你应该能够 得到一个彩色的窗口。 这似乎很简单,但它使我们遇到了许多重要的概念。

 *而不是直接画到窗口表面 我们画到屏幕外的纹理和**互换链**负责管理纹理翻转。
 *GPU的3D渲染管道通过**传递**,这是通过命令编码器可以访问的命令的特殊范围。
 *径向一个或多个**附件**,这是纹理视图。


```{figure} /images/first-color.png
:align: center
:class: with-shadow
我们的第一个颜色!
```

```{note}
在使用Dawn时,显示的颜色可能有所不同,因为表面的颜色格式使用了另一个颜色空间. 更多关于这个[后来](../basic-3d-rendering/input-geometry/loading-from-file.md)!
```

我们现在是了**配备基本的 WebGPU 设置**,还可以更深入地潜入3D渲染管道! 下一个章节是**奖金**引入**更舒适的 API**这得益于 C++ idioms。

*结果代码 :* [`step025`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step025)
