纹理映射<span class="bullet">🟡</span>
===============

```{translation-warning} 译文可能已过时, /basic-3d-rendering/texturing/texture-mapping.md
这是[英文原文](%original%)的**社区翻译**。如果原文在译文完成后有更新，本页内容可能不再完全同步。欢迎您[参与改进翻译](%contribute%)！
```

````{tab} With webgpu.hpp
*结果代码 :* [`step065`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step065)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step065-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step065-vanilla)
````

在前一章中,我们使用了**屏幕像素坐标** (`in.position`)来决定从图像到加载的哪个texel.

如果我们的几何学不再是一个完整的屏幕四角,这就是**不是我们想要的**。让我们回到视角投影,并使用方便`glm::lookAt`函数以获取有趣的视角:

```C++
uniforms.modelMatrix = mat4x4(1.0);
uniforms.viewMatrix = glm::lookAt(vec3(-0.5f, -2.5f, 2.0f), vec3(0.0f), vec3(0, 0, 1)); // the last argument indicates our Up direction convention
uniforms.projectionMatrix = glm::perspective(45 * PI / 180, 640.0f / 480.0f, 0.01f, 100.0f);
```

```{figure} /images/wrong-mapping.png
:align: center
:class: with-shadow
纹理不"跟随"几何.
```

Texel 坐标
-----------------

如何解决这个问题? 通过计算**Texel 坐标**在顶点的遮蔽处 让光栅对每个碎片插进去

```rust
结构输出{
	// [...]
@ location(2) texelCoords: vec2f, (中文(简体) ).
};

fn 对s_主( in: Vertex Input) - > 垂直输出{
	// [...]

/ 请检查url=值 (帮助). 在平面.obj中,顶点xycoords从−1到1不等.
/ 和我们重映射到( 0, 256) 我们的纹理大小 。
out.texelCoords = (in.position.xy + 1.* 128.0;
返回;
}

fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
让颜色=纹理Load(梯度Texture, vec2( in.texelCoords), 0; rgb;
	// [...]
}
```

```{important}
转换为整数(`vec2i`)是在片段遮蔽器而不是顶点遮蔽器中完成的,因为整数顶点输出不会被光栅喷射器内插.
```

````{note}
由于我们增加了从顶点到碎片遮蔽器的过渡属性的大小,我们需要**更新设备限制**:

```C++
requiredLimits.limits.maxInterStageShaderComponents = 8;
//                                                    ^ This was a 6
```
````

```{figure} /images/fixed-mapping.png
:align: center
:class: with-shadow
一个更好的纹理映射, 当观点改变时保持一致。
```

紫外线坐标
--------------

纹理坐标实际上用范围表示$(0,1)$因此,他们**不取决于决议**,而不是明确给出texel指数。 我们称它们为 普通坐标**紫外线坐标**.

我们可以使用`textureDimensions`函数 :

```rust
结构输出{
	// [...]
@ 位置(2) uv: vec2f,
};

fn 对s_主( in: Vertex Input) - > 垂直输出{
	// [...]
/ 请检查url=值 (帮助). 在平面.obj中,顶点xycoords从−1到1不等.
/ 而我们把它重新映射到分辨率不可知( 0, 1) 范围
输出.uv = 输入. position.xy* 0.5 + 0.5;
返回;
}

fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
/我们把紫外线coords重新映射到实际的texel坐标
让 texelCoords = vec2i( in.uv) 翻译*vec2f(纹理图));
让颜色=纹理Load(梯度Texture,texelCoords,0.rgb);
	// [...]
}
```

从文件装入
-----------------

紫外线坐标很常见**OBJ 文件中有紫外线**包括这架飞机 我们可以加一个**新属性**就像我们从OBJ文件到遮蔽器提供紫外线一样:

```C++
using glm::vec2;

struct VertexAttributes {
	vec3 position;
	vec3 normal;
	vec3 color;
	vec2 uv;
};

// [...]

requiredLimits.limits.maxVertexAttributes = 4;
//                                          ^ This was a 3

// [...]

std::vector<VertexAttribute> vertexAttribs(4);
//                                         ^ This was a 3

// [...]

// UV attribute
vertexAttribs[3].shaderLocation = 3;
vertexAttribs[3].format = VertexFormat::Float32x2;
vertexAttribs[3].offset = offsetof(VertexAttributes, uv);

// [...]
```

请注意,在从文件装入紫外线坐标时,我们需要这样做**一点点转换**在V轴上。

```{image} /images/uv-coords-light.svg
:align: center
:class: only-light
```

```{image} /images/uv-coords-dark.svg
:align: center
:class: only-dark
```

<p class="align-center">
	<span class="caption-text"><em>现代图形 API使用与OBJ文件格式不同的紫外坐标系统.</em></span>
</p>

```C++
bool loadGeometryFromObj(const fs::path& path, std::vector<VertexAttributes>& vertexData) {
	// [...]

	vertexData[offset + i].uv = {
		attrib.texcoords[2 * idx.texcoord_index + 0],
		1 - attrib.texcoords[2 * idx.texcoord_index + 1]
	};

	// [...]
}
```

并在阴间:

```rust
垂直输入{
@ 位置( 0) 位置: vec3f,
@ 位置(1) 正常: vec3f,
@ 位置(2) 颜色: vec3f,
@ location(3) uv: vec2f,
};

// [...]

@ 顶点
fn 对s_主( in: Vertex Input) - > 垂直输出{
	// [...]
输出.uv = in.uv;
返回;
}

@ flagment (英语).
fn fs (英语)._主( in: VertexOutput) - > @ location (0) vec4f{
让 texelCoords = vec2i( in.uv) 翻译*vec2f(纹理图));
让颜色=纹理Load(梯度Texture,texelCoords,0.rgb);
    // [...]
}
```

对于飞机来说,这不应该改变任何事情, 但如果你尝试[立方体. obj](../../../../data/cube.obj)比如说,它也是很好的!

```{figure} /images/textured-cube.png
:align: center
:class: with-shadow
从位置看到的纹理立方体$(-2, -3, 2)$.
```

结论
----------

我们现在可以装上纹理坐标 将纹理映射到3D meshes上 但正如你可能注意到的**很多别名**在碎片遮蔽器里收集Texel数据

因此,下一章介绍**取样的适当方式**纹理!

````{tab} With webgpu.hpp
*结果代码 :* [`step065`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step065)
````

````{tab} Vanilla webgpu.h
*结果代码 :* [`step065-vanilla`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step065-vanilla)
````
