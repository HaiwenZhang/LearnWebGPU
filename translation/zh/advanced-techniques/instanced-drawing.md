实例绘图 (<span class="bullet">🔴</span>TODO)
=================

```{translation-warning} 译文可能已过时, /advanced-techniques/instanced-drawing.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

多次绘制**相同的几何图形**是很常见的。当**在地形上散布物体**（如岩石或树木）、绘制**粒子系统**等时，就会发生这种情况。

事实证明，得益于**实例化**机制，GPU 特别擅长绘制此类重复的几何图形。

```{note}
它比发出多个绘制调用**更高效**，不仅因为它避免了重复**构建绘制命令的开销**，而且还因为它使 GPU 通过同时通过渲染管道流式传输实例来更好地**管理内存**。
```

渲染管道编码器的 `draw` 和 `drawIndexed` 方法都通过第二个参数支持实例化（`wgpuRenderPipelineEncoderDraw` 和 `wgpuRenderPipelineEncoderDrawIndexed`）。

```C++
renderPipeline.draw(vertexCount, instanceCount, 0, 0);
```

但您很快就会注意到，如果更改此实例计数参数，所有实例都会绘制在**完全相同的位置**！

区分实例的第一个解决方案是在着色器中，使用顶点着色器输入中的 `@builtin(instance_id)` 属性。

```rust
待办事项
```

第二种解决方案是使用实例级顶点属性。待办事项
