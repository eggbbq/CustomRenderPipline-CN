# Draw Calls <sub>着色器和批次</sub>
- *编写HLSL着色器*
- *支持SRP批处理，GPU实例化，动态合批*
- *配置每个对象的材质属性，随机大量绘制*
- *创建透明镂空材质*

这是[自定义可编程渲染管线]系列教程的第二篇。它包含编写着色器和高效绘制多个物体。

这个教程使用Unity 2019.2.9f1制作(*实际上，你可以用更高的任意版本来实践*)

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/tutorial-image.jpg)

*大量球体但是draw call很少*

# 着色器(Shader)
*Shader属于术语，在后面的篇幅中，将不在翻译这个单词。*

要用GPU绘制一些东西，就必须告诉GPU画什么，怎么画。画什么，通常是网格。怎么画，则是Shader定义的一系列GPU指令。除了网格，Shader还需要额外的信息才能工作，如物体的变换矩阵和材质属性。

Unity的通用管线和高清管线允许你使用**Shader Graph**来设计着色器，**Shader Graph**是个可视化的图形程序，可以帮你生成Shader代码。但是我们的自定义管线不支持这个，所以我们要手写着色器代码，这可以让我们完全控制并理解着色器的工作原理。

## 1.1 无光照着色器(Unlit Shader)
*Unlit Shader这也是个术语，后续也可能不翻译*

我们的第一个着色器将简单简单地绘制一个纯色网格，没有任何光照。通过软件菜单Assets/Create/Shader，可以创建一个着色器文件。删掉Unit Shader模板中的默认代码，我们从新开始。把我们创建的Shader资源文件，命名为Unlit，放到Custom RP下的Shaders文件夹内。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/shader-asset.png)

*Unlit着色器资源文件*

着色器代码大部分看起来，很像混合了一对古老语法的c#。着色器代码定义就像一个class, 只不过它以Shader关键字开始，接下来是材质的属性清单。让我们使用CustomRp/Unlit, 大致看看代码结构。

```c#
Shader "Custom RP/Unlit" {

    Properties {}

    SubShader {

        Pass {}
    }
}
```
这就定义了一个最小的着色器，它可以通过编译，也允许我们使用它创建材质。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-material.png)

*自定义unlit材质*

默认着色器渲染为一个纯白色的网格。材质属性显示有，一个默认的渲染队列属性(render queue), 它由着色器自带且值为2000(这个值是默认的，用于不透明几何体渲染队列)。下面还有个开关按钮，用于开启双面全局光照，不过这个现在与我们无关。

# 1.2 HLSL程序