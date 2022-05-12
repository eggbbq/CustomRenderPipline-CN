# Draw Calls <sub>着色器和批次</sub>
- *编写HLSL着色器*
- *支持SRP批处理，GPU实例化，动态合批*
- *配置每个对象的材质属性，随机大量绘制*
- *创建透明镂空材质*

这是[自定义可编程渲染管线]系列教程的第二篇。它包含编写着色器和高效绘制多个物体。

这个教程使用Unity 2019.2.9f1制作(*实际上，你可以用更高的任意版本来实践*)

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/tutorial-image.jpg)

*大量球体但是Draw Call很少*

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
我们要写的着色器代码叫做High-Level Shading Language,简称HLSL。我们必须把它放在Pass区块中，用HLSLPROGRAM和ENDHLSL关键词包起来。之所以这样做，是因为Pass区块中还可能放放七天不是HLSL的代码。
```c#
Pass {
    HLSLPROGRAM
    ENDHLSL
}
```
要绘制一个网格，GPU需要把它的三角形光栅化，转换成像素数据。其过程大致上是，把顶点坐标从3D空间到2D可视化空间的顶点坐标转换，然后用像素填充最终的三角形。这个两个步骤分别有两个单独的着色器程序控制，二者我们都需要定义。第一个被称为顶点着色器，第二个被称为片段着色器。片段着色器对应一个显示像素绘制纹理纹素，当有其他东西在它之后绘制到它的上面，时，它可能会被覆盖，所以它并不一定代表最终的结果。

我们需要通过pragma指令定义顶点和片段程序名称。这是个单行语句，以#pragma开始，跟着是vertex或者fragment，加上相应的名字。这里我们用UnlitPassVertex和UnlitPassFragment。
```c#
    HLSPROGRAM
    #pragma vertex UnlitPassVertex
    #pragma fragment UnlitPassFragment
    ENDHLSL
```
现在着色器编译器会抱怨找不到声明的着色器内核。我们还需要编写同名的HLSL代码的具体实现。我们可以直接在#pragma指令之后编写代码，但是我们会把所以的HLSL代码放大一个单独的文件中。我们用这个UnlitPass.hlsl文件，把它放在shader同一个文件夹内，还要添加一个#include指令后面带上相关的文件路径，让着色器编译器插入这个文件的内容。
```c#
HLSLPROGRAM
#pragma vertex UnlitPassVertex
#pragma fragment UnlitPassFragment
#include "UnlitPass.hlsl"
ENDHLSL
```
Unity还没有菜单来创建HLSL文件，你要自己创建它。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-pass-asset.png)

*UnlitPass HLSL资源文件*

# 1.3 Include防御
*Include这个单词的中文，暂时不好找简单明了的对应词语。对于程序员来说，都知道它是什么，所以就不用翻译了。*

HLSL文件向c#类文件一样组织代码，虽然HLSL没有类的概念。除了代码块的本地作用域之外，它只有一个全局作用域。因此，任何东西都可以在任何地方访问。引入文件也没有命名空间这个一说。它就是通过include这个指令，插入整个文件。如果你多次include同一个文件，代码就重复了，会导致编译器错误。为了防止这个事情，我们就要添加一个include的防御代码到UnitlPass.hlsl中。

用#define指令可以定义任何标识符，通常为大写。我在文件顶部定义CUSTOM_UNLIT_PASS_INCLUDED。
```c#
#define CUSTOM_UNLIT_PASS_INCLUDED
```
这个简单的宏定义例子。如果它存在，即表示我们的文件已经被包含。因此我们不需要靠再次包含。换个说法，我们只想插入还没有定义的代码。我们可以使用#ifndef指令检查。检查放在，在宏定义之前。
```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED
```
如果我们已经定义了宏，所有#ifndef之后的代码都会被跳过，因此不会被编译。我们需要用#endif指令指定宏定义的范围。
```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED
#endif
```
现在我们可以确保所有相关的代码，即便多次导入也不会插入多次。

# 1.4 着色器函数
我们在include防御中，定义我们的着色器函数。跟没有访问修饰符的c#方法一样。以一个简单的空函数开始吧。
```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

void UnlitPassVertex () {}

void UnlitPassFragment () {}

#endif
```
这足以使我们的着色器通过编译了。如果有东西显示的话，其结果可能是个默认的青色着色。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/cyan-sphere.png)

*青色球体*

要产生有效的输出，我们还需要让片段函数返回一个颜色。颜色由4个分量构成的float4矢量，包含红色，绿色，蓝色，和透明。通过float4(0.0,0.0,0.0,0.0)我们可以定义纯黑色，但是我们也可以只用一个数字0来定义，因为它会自动展开成一个完成的矢量。透明值不起作用，因为我们创建的是不透明着色器，因此返回0是可以的。
```c#
float4 UnlitPassFragment () {
    return 0.0;
}
```
>为什么用0.0替代0?<br>
它指示给我们指明它是个浮点数而不是整数，对编译器来说没有区别。

>我们应该使用float或者half精度吗?<br>
大多数的移动设备GPU都支持精度类型，half会更高效。所以，如果你正在优化移动设备性能，可以使用half精度。大多数经验是，位置和纹理坐标使用float 其他使用half，就足以产生足够好的结果了。<br>
如果不是移动设备平台，精度就没有意义，因为GPU总是用float，即便我们写half。这个系列教程，我会始终使用float。<br>
还有一个叫fixed的类型，但是只有老的移动硬件平台支持。它通常和half等效。

此时，着色器会变一失败，因为我们的函数没有语义。我们必须指明我们返回的值是什么，因为我们可能产生很多同意义的数据。这个时候，我们提供为渲染目标提供系统默认值，通过在函数参数列表后面写个:SV_TARGET。
```c#
float4 UnlitPassFragment () : SV_TARGET {
    return 0.0;
}
```
UnlitPassVertex的职责是顶点位置变换，所以需要返回一个位置信息。它也是个flaot4矢量，因为它必须定义为齐次裁剪空间位置，我们之后说这个。我们在此返回0，用SV_POSITION指明语义。
```c#
float4 UnlitPassVertex () : SV_POSITION {
    return 0.0;
}
```
# 1.5 空间变换
当所有的顶点都设置为0时，网格坍缩成一个点，什么都不会绘制。顶点函数的主要任务就是把原始的顶点位置转换到正确的空间。当它被调用时，如果我们要求的话，可用的顶点数据会传入。我们可以在UnlitPassVertex中添加一个参数来完成这个操作。我们需要顶点位置信息，它定义在模型空间，所以我们把它命名为positionOS，使用和Unity新渲染管线一样的约定位。位置的类型是一个float3，因为它是个3D点。通过float4(positionOS,1), 并返回。
```c#
float4 UnlitPassVertex (float3 positionOS) : SV_POSITION {
    return float4(positionOS, 1.0);
}
```
>顶点位置不是flaot4?<br>
3D空间中的点，通常用4D向量定义，第四个项为1，如果表示方向时，则为0。作者这部分讲得比较简略。请查阅【齐次坐标】，以获得更全面的知识。这一小段就不翻译了。

我们也同样添加语义到输入参数中，因为点点数据不仅仅包含位置信息。这个例子中我们需要POSITION，如下
```c#
float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
    return float4(positionOS, 1.0);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/object-space-position.png)

*使用对象空间位置*

网格在此显示了，但是不正确，因为我们输出的位置信息用错了空间。空间转换需要一些矩阵，他们在物体绘制时被传染到GPU。我们需要添加这些矩阵到我们的着色器中，但是由于他们重视一样的，我们就把它放到一个单独的HLSL文件中当成一个标准的输入，即能保持代码结构化，又能其他的shader文件引入。添加一个UnityInout.hlsl文件，放到Custom RP下的ShaderLibrary文件夹中，就跟Unity渲染管线的文件结构一样。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/shader-library.png)

*ShaderLibray文件夹中UnityInput文件*

文件开始写入CUSTOM_UNITY_INPUT_INCLUDED这个include防御，然后在全局作用域中，定义一个float4x4矩阵，命名为unity_ObjectToWorld。在c#中，这就可能是一个字段，但是这里称之为uniform值（?统一值）。它在每次绘制时由GPU设置，在顶点和片段绘制期间的所有调用都不变。
```c#
#ifndef CUSTOM_UNITY_INPUT_INCLUDED
#define CUSTOM_UNITY_INPUT_INCLUDED

float4x4 unity_ObjectToWorld;

#endif
```
我们用矩阵把对象空间转换到世界空间。由于这是一个很常用的功能，让我们为它创建一个函数，放在另一个文件中，这次我们叫它Common.hlsl，同样放到ShaderLibary文件中。我们引入UnityInput，然后声明一个方法叫TransformObjectToWorld，输入一个float3参数，返回一个float3。
```c#
#ifndef CUSTOM_COMMON_INCLUDED
#define CUSTOM_COMMON_INCLUDED

#include "UnityInput.hlsl"

float3 TransformObjectToWorld (float3 positionOS) {
    return 0.0;
}

#endif
```
空间转换是通过矩阵和向量，调用mul函数实现的。这里我们需要一个4D向量，但是由于它第四项总是1，我们可以通过float4(positionOS,1)自己构造。返回结果也是一个4D向量，第四个项也是1。我们可以通过访问xyz属性解开，千三个项，这称之为swizzle操作。（*swizzle暂时不知道怎么翻译*）
```c#
float3 TransfromObjectToWorld(float positionOS)
{
    return mul(unity_ObjectToWorld, float4(positionOS,1).xyz;
}
```
现在我们在UnitPassVertex中，转换到世界空间。首先，直接在这个函数上面引入Common.hlsl。由于它是在另一个文件夹，我们可以动工相对路径*../ShaderLibrary/Common.hlsl*访问到。然后使用TransfromObjectToWorld计算positionWS变量，然后用它替换对象空间位置，作为函数返回。
```c#
#include "../ShaderLibrary/Common.hlsl"
float4 UnlitPassVertex (float3 positionOS:POSITION):SV_POSITION
{
    float3 positionWS = TransformObjectToWorld(positionOS.xyz);
    return float4(positionWS,1);
}
```
这个结果还是错的，因为我们需要齐次裁剪空间的坐标。这个空间定义了一个立方体，包含了任何相机观察空间的物体，在透视相机的情况下扭曲成一个梯形。从世界空间转换到这个空间，可以通过乘以视图投影矩阵完成，它记录了相机的位置，旋转，投影，视野(FOV)，远近裁剪平面。把它添加到UnityInput.hls中。
```c#
float4x4 unity_ObjectToWorld;
float4x4 unity_MatrixVP;
```
添加一个叫TransformWorldToHClip到Common.hlsl中，就像添加TransfromObjectToWorld一样的操作，不同是输入变成世界空间，使用另一个矩阵，产生一个float4。
```c#
float3 TransformObjectToWorld (float3 positionOS) {
    return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
}

float4 TransformWorldToHClip (float3 positionWS) {
    return mul(unity_MatrixVP, float4(positionWS, 1.0));
}
```
让UnlitPassVertex使用这个函数返回了正确的空间的位置。
```c#
float4 UnlitPassVertex(float3 positionOS:POSITION):SV_POSITION
{
    float3 positionWS = TransformObjectToWorld(positionOS.xyz);
    return TransformWorldToHClip(positionWS);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/black-sphere.png)

*正确的黑色球形*

# 1.6 核心库
我们定义的这两个函数，太常用了，Core RP Pipeline包中也包含了它们。核心库定义了许多有用且基本东东西，因此让我们安装它，删掉我们的定义导入相关的文件。这个实例，使用*Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransform.hlsl*
```c#
//float3 TransformObjectToWorld (float3 positionOS) {
//    return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
//}

//float4 TransformWorldToHClip (float3 positionWS) {
//    return mul(unity_MatrixVP, float4(positionWS, 1.0));
//}

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```
这里无法编译，因为SpaceTransform.hlsl不会认为unity_ObjectToWorld存在。相反，它期望相关的矩阵使用UNITY_MATRIX_M这个宏定义，所以让我们在引入文件之前，单独一行写#define UNITY_MATRIX_M unity_ObjectToWorld。在这之后，所有的UNITY_MATRIX_M都会被替换成unity_ObjectToWorld。这是有原因的，我们之后在讲。
```c#
#define UNITY_MATRIX_M unity_ObjectToWorld

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```
逆矩阵unity_WorldToObject也是一个道理，它应该用UNITY_MATRIX_I_M定义，unity_MatrixV对应UNITY_MATRIX_V，unity_MatrixVP对应UNITY_MATRIX_VP。最后投影矩阵通过UNITY_MATRIX_P定义，它由glstate_matrix_projection提供。我们目前不需要这些额外的矩阵，但是如果不引入它们就没有办法通过编译。
```c#
#define UNITY_MATRIX_M unity_ObjectToWorld
#define UNITY_MATRIX_I_M unity_WorldToObject
#define UNITY_MATRIX_V unity_MatrixV
#define UNITY_MATRIX_VP unity_MatrixVP
#define UNITY_MATRIX_P glstate_matrix_projection
```
把这些额外的矩阵也添加到UnityInput中。
```c#
float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;

float4x4 unity_MatrixVP;
float4x4 unity_MatrixV;
float4x4 glstate_matrxi_projection;
```
最后个缺少的东西不是矩阵。是unity_WorldTransformParams, 它包含一些转换信息，也是目前我们不需要的。它是一个向量，使用real4定义，他不是一个有效的类型，而是一个float4或者half4的别名，具体取决与平台。
```c#
float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;
real4 unity_WorldTransformParams;
```
这个别名和一些其他基本的宏是根据图形API定义的，我们通过引入*Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl*。在我们的Common.hlsl里面在引入UnityInput.hlsl之前这么做。如果你对这些内容好奇，你可以查看导入包中的这些文件。
```c#
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "UnityInput.hlsl"
```

# 1.7 颜色
通过调整UnityUnlitFragment，可以改变渲染的颜色。举例子，我们可以返回float4(1.0,1.0,0,1.0)替代0，让它变黄。

```c#
float4 UnlitPassFragment () : SV_TARGET {
    return float4(1.0, 1.0, 0.0, 1.0);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/yellow-sphere.png)

*黄色球形*

为了让每个材质能够配置这个颜色，我们需要用定义一个uniform值。在include指令之后，在UnlitPassVertex之前定义。我们需要一个float4，把它命名为*_BaseColor*。前置下划线命名是一个标准的方式，用于指明它是个着色器属性。返回这个值，替换掉UnlitPassFragment的硬编码。

```c#
#include "../ShaderLibrary/Common.hlsl"

float4 _BaseColor;

float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
    float3 positionWS = TransformObjectToWorld(positionOS);
    return TransformWorldToHClip(positionWS);
}

float4 UnlitPassFragment () : SV_TARGET {
    return _BaseColor;
}
```
我们又回到了黑色，因为默认值为0。要把这个属性连接到材质，我们要在Properties添加 _BaseColor。
```c#
Properties {
        _BaseColor("Color", Color) = （1.0,1.0,1.0,1.0,）
    }
```
属性名字必须是个字符，用来在查看面包中显示，类型标识符为**Color**，

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-material-color.png)

*红色的无光照材质*

现在，可以用我们的着色器创建多个材质，每个都一个定义不同的颜色。

# 批处理
每次Draw Call都需要CPU和GPU通信。如果发送大量数据到CPU，可能会因为等待白白浪费时间。同时如果CPU忙于发送数据，它就干不了别的事情。两个问题都会让帧率变慢。此时，我们的方法直接了当：每个对象一个Draw Call。这是最浪费的方法，尽管我们同一时刻发送的数据量很小。

举个例子，我做了个场景，放六6个球体，它们分别用4中材质中的一个（红，绿，黄，蓝）。需要78个Draw Call来渲染，76个用于小球，一个用于天空盒，一个用于清理渲染目标。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/76-spheres.png)

*76个球体，78个Draw Call*

如果你打开i游戏窗口的状态面板，你可以看到渲染帧的预览。有趣的是，这里有77个批次，清理操作被忽略了，节约了0个批次。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/statistics.png)

*游戏窗体统计数据*

# 2.1 SRP批处理器
批处理是合并Draw Call的程序，减少CPU和GPU通信时间。批处理最简单的方法就是开启SRP批处理器。然而，它只有在着色器兼容时才会起作用，我们的Unlit着色器不兼容。你可以通过选中它，查看面板来验证。那里有一个SRP批处理器行，指明了不兼容，在下面给出了一个不兼容的原因。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/not-compatible.png)

*不兼容*

比起减少Draw Call数量，SRP批处理器更精简。它缓存材质属性到GPU，不必每次Draw Call都发送这些数据。这个同时减少了CPU每帧通信的数据和工作。但只有着色器坚持使用一个严格的统计的数据结构才起作用。

所有的材质属性都被定义在了一个具体的内存缓冲里，而不是是全局级别的。这是通过把_BaseColor声明包装在一个cbuffer区块中，并且cbuffer使用UnityPerMaterial名字。它的工作类似于结构体声明。它把_BaseColor房子一个特殊的常量缓冲区来隔离，尽管还是可以通过全局访问。
```c#
cbuffer UnityPerMaterial{
   float _BaseColor;
};
```
如果需要在UnityPerMaterial中的话，我们还必须对unity_ObjectToWorld, unity_WorldToObject, unity_WorldTransformParams这样做，
```c#
CBUFFER_START(UnityPerDraw)
    float4x4 unity_ObjectToWorld;
    float4x4 unity_WorldToObject;
    real4 unity_WorldTransformParams;
CBUFFER_END
```
如果需要，我们定义一组特殊的只。对于转换组来说，我们还需要引入一个float4 unity_LODFace的向量，即使现在我们不需要它。变量的位置不重要，但是Unity把它直接放到unity_WorldToObject后面，我们也这样做。
```c#
CBUFFER_START(UnityPerDraw)
    float4x4 unity_ObjectToWorld;
    float4x4 unity_WorldToObject;
    float4 unity_LODFade;
    real4 unity_WorldTransformParams;
CBUFFER_END
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/compatible.png)

*兼容SRP批处理器*

随着我们的着色器兼容了，下一步就是开启SRP批处理，GraphicsSettings.useScriptableRenderPipleBatching为true即可。我们只需要设置一次这个值，因此让我们在我们的管线实例的构造函数中设置它。
```c#
public CustomRenderPipeline () {
        GraphicsSettings.useScriptableRenderPipelineBatching = true;
    }
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/statistics-srp-batching.png)

*节约负数个批次*

状态面包显示节约了76个批次，虽然它是个负数。Frame Debugger此时在RenderLoopNewBatcher.Draw下，显示个SRP批次条目，但是请你记住，这不是一个Draw Call，而是Draw Call的优化序列。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/srp-batch.png)

*一个SRP批次*

# 很多颜色
即便使用了4个材质，也只有一个批次。之所以可行，是因为他们的数据都被缓存到了GPU，每次绘制只需要包含一个偏移量就得到了正确的内存地址。唯一的限制是每个材质的内存布局必须一样。我们的材质都是用同一个着色器，只包含一个颜色属性，符合这种情况。Unity不会比较材质的内存布局，它简单的把使用相同的着色器变体批量绘制。

如果我们只需要很少的不同颜色话，这样做是没有问题的。但是如果我们想要每个小球，都有自己的颜色，我们就不得不创建很多材质。如果我们可以给每个小球设置颜色，就更方便了。默认情况下，这是不可能的，但是我们可以创建一个自定义的组建来支持。把它就叫做PerObjectMaterialProperties。由于它是个示例，我们把它放到Custom RP下的Examples文件夹。

这个想法是，一个游戏对象有一个PerObjectMaterialProperties组件，可以配置Base Color，用它设置材质属性_BaseColor。它还需要知道着色器属性标识符，我可以通过Shader.PropertyToID检索并把它存储到一个静态变量中，就如之前在CameraRender存储着色器标识符一样，只不过这次是要给整数。
```c#
using UnityEngine;

[DisallowMultipleComponent]
public class PerObjectMaterialProperties : MonoBehaviour {

    static int baseColorId = Shader.PropertyToID("_BaseColor");

    [SerializeField]
    Color baseColor = Color.white;
}
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/per-object-material-properties.png)

*PerObjectMaterailProperties组件*

通过MaterialPropertyBlock设置每个材质的属性。所有的PerObjectMaterialProperties可以通用一个MaterialPropertyBlock实例, 所以把它声明为静态字段。
```c#
static MaterialPropertyBlock block;
```
如果block不存在，就创建一个，然后传入属性标识和颜色，调用SetColor方法，最后通过SetPropertyBlock，把这个block应用到游戏对象的Render组件, 该组件会复制其设置。在OnValidate这个事件函数中编写这些代码，以便于在编辑器中立即看到结果。
```c#
void OnValidate () {
        if (block == null) {
            block = new MaterialPropertyBlock();
        }
        block.SetColor(baseColorId, baseColor);
        GetComponent<Renderer>().SetPropertyBlock(block);
    }
```
>OnValidate何时被调用?<br>
OnValidate在Unity编辑器中，当组件被加载或被修改是调用。因此，各个颜色会立即响应编辑。

我把这个组件添加到了24个小球并赋予不同的颜色。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/many-colors.png)

*很多颜色*

不幸的是，SRP批处理器不能处理每个物体材质属性。因此，24个小球都退回到普通的Draw Call，由于排序，还可能会将其他小球分割成多个批次。


![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/24-non-batched.png)

*24个没有合批的Draw Call*

与此同时，OnValidate方法在构建版本中不会被调用。要让颜色出现，我们还要在Awake中调用它，这里我们简单调用一下OnValidate方法。
```c#
void Awake () {
        OnValidate();
    }
```

# 2.3 GPU实例化
还有另一个方法来合并Draw Call, 它对每个材质属性都有效。这就是众所周知的GPU实例化，它把多个使用同一个网格的物体一次绘制。CPU搜集每个物体的变换和材质属性，把他们放到数组中发送到GPU。GPU遍历所有条目，用指定的顺序渲染它们。

因为GPU实例化需要提供数组数据，我们的着色器不支持它。要支持它的第一步，是在我们的着色器二Pass中，在顶点和片段程序之前，添加#pragma multi_compile_instancing指令。

```c#
#pragma multi_compile_instancing
#pragma vertex UnlitPassVertex
#pragma fragment UnlitPassFragment
```
这会使Unity生成两个我们的着色器的变体，一个支持GPU实例化，一个不支持。材质查看面板会出现一个开关选项，允许我们选择那个版本的变体应用到材质。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/gpu-instancing-material.png)

*开启GPU实例化的材质*

支持GPU实例化需要改变方法，为此我们要添加核心库的UnityInstancing.hlsl文件。它必须在UNITY_MATRIX_M之后在添加SpaceTransforms.hlsl之前。
```c#
#define UNITY_MATRIX_P glstate_matrix_projection

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```
UnityInstancing.hlsl做的事情就是重新定义宏来访问实例化数组数据。但是要使它起作用，它需要知道当前渲染物体的索引。缩影有顶点数据提供，因此我们必须使它可用。UnityInstancing.hlsl定义了一些宏来简化这个操作，但是它假设我们的顶点函数有个结构体参数。

可以声明一个结构体，就像cbuffer一样，然后把它当成函数的参数。我们还要定义结构内部的语义。这个方法的好处是比长长的数据参数列表更容易阅读。因此将positionOS参数放到Attributes结构体中，表示顶点的输入数据。
```c#
struct Attributes {
    float3 positionOS : POSITION;
};

float4 UnlitPassVertex (Attributes input) : SV_POSITION {
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    return TransformWorldToHClip(positionWS);
}
```
GPU实例化时，对象索引也可当成一个顶点属性。我们可以简单地在Attributes中放一个UNITY_VERTEXT_INPUT_ID。
```c#
struct Attributes {
    float3 positionOS : POSITION;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
````
接下来，添加UNITY_SETUP_INSTANCE_ID到UnlitPassVertex的开始位置。它会从输入中提取索引，并存储到其他实例化宏所以依赖的全局静态变量中。
```c#
float4 UnlitPassVertex (Attributes input) : SV_POSITION {
    UNITY_SETUP_INSTANCE_ID(input);
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    return TransformWorldToHClip(positionWS);
}
```
这已经足以使GPU实例化工作了，然而SRP批处理器优先，所以我们得到的结果没有什么不同。但是我们还没有支持每个实例材质不同数据。我们需要把_BaseColor替换成数组。用UNITY_INSTANCING_BUFFER_START替换CBUFFER_START， 用UNITY_INSTANCING_BUFFER_END替换CBUFFER_END, UNITY_INSTANCING_BUFFER_END需要一个参数，没有强制和UNITY_INSTANCING_BUFFER_START一样，但也没有什么理由让它们不同。

```c#
//CBUFFER_START(UnityPerMaterial)
//    float4 _BaseColor;
//CBUFFER_END
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    float4 _BaseColor;
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```
然后，用UNITY_DEFINE_INSTANCINF_PROP(float4, _BaseColor)替换_BaseColor。
```c#
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    //float4 _BaseColor;
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```
在实例化时，我们现在还需要让实例索引在UnitPassFragment中可用。为了简化这个操作，我们会用一个结构体持有UnlitPassVertex输出的位置和索引数据，使用UNITY_TRANSFER_INSTANCE_ID(input, output)来复制复制存在的索引。跟Unity一样，我们把这个结构体命名为Varying，隐喻它包含的数据能在同个三角形的片段程序中发生变化的。
```c#
struct Varyings {
    float4 positionCS : SV_POSITION;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex (Attributes input) { //: SV_POSITION {
    Varyings output;
    UNITY_SETUP_INSTANCE_ID(input);
    UNITY_TRANSFER_INSTANCE_ID(input, output);
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    output.positionCS = TransformWorldToHClip(positionWS);
    return output;
}
```
把这个结构体作为UnlitPassFragment的参数。然后像之前那样用UNITY_SETUP_INSTANCE_ID让所以可用。材质属性现在必须通过UNITY_ACCESS_INSTANCE_PROP(UnityPerMaterial, _BaseColor)来访问。
```c#
float4 UnlitPassFragment (Varyings input) : SV_TARGET {
    UNITY_SETUP_INSTANCE_ID(input);
    return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/instanced-draw-calls.png)

*实例化需要的Draw Call*

现在Unity可以把24个不同颜色的小球合并, 减少Draw Call数量。最后我们得到4个Draw Call，是因为这小球使用了4个不同的材质。GPU实例化只对使用同一个材质的物体起作用。虽然小球覆写了材质的颜色但是它们仍然使用相同的材质，所以它们可以在一个批次中绘制。
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/one-instanced-material.png)

*实例化一个材质*

需要注意，批次的大小是有限制的，它取决于据目标平台和每个实例需要的数据量。如果你超过了一个批次的限制，就会有多个批次。同时，如果使用多个材质，排序也可能造成批次分割成多个。

# 2.4 绘制多个实例化的网格
当上百个物体可以合并成一次Draw Call时, GPU实例化成为一个显著的优势。在场景中手动编辑这么多物体不切实际，因此我们用脚本随机生成一堆。创建一个MashBall的示例组件，在Awake时复制大量的物体。让我们把_BaseColor缓存起来，同时给网格和材质一个配置选项。
```c#
using UnityEngine;

public class MeshBall : MonoBehaviour {

    static int baseColorId = Shader.PropertyToID("_BaseColor");

    [SerializeField]
    Mesh mesh = default;

    [SerializeField]
    Material material = default;
}
```
新建个GameObject挂上这个组件。我给它默认小球来绘制。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/mesh-ball-component.png)

*小球的MeshBall组件*

我们可以复制很多游戏五i他，但是我们不这么干。我们设置一个变换矩阵数组和颜色数组，然后让GPU用它们来渲染。这是就是GPU实例化最有用的地方。我们一次提供1023个实例，所以把数组长度设置为它，同时我们需要传递颜色数据和MaterPropertyBlock。颜色数组类型时Vector4。
```c#
Matrix4x4[] matrices = new Matrix4x4[1023];
Vector4[] baseColors = new Vector4[1023];

MaterialPropertyBlock block;
```
创建一个Awake方法，用随机位置和颜色填充数组。

```c#
void Awake () {
    for (int i = 0; i < matrices.Length; i++) {
        matrices[i] = Matrix4x4.TRS(
            Random.insideUnitSphere * 10f, Quaternion.identity, Vector3.one
        );
        baseColors[i] =
            new Vector4(Random.value, Random.value, Random.value, 1f);
    }
}
```
在Update中，调用block.SetVectorArray来设置颜色。之后调用Graphics.DrawMeshInstanced，参数依次为网格，子网格索引(此时用0)，材质，矩阵数组，元素数量，属性block。
```c#
void Update () {
    if (block == null) {
        block = new MaterialPropertyBlock();
        block.SetVectorArray(baseColorId, baseColors);
    }
    Graphics.DrawMeshInstanced(mesh, 0, material, matrices, 1023, block);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/mesh-ball.png)

*1023个小球，3个Draw Call*

进入运行模式，限制会生成密集的小球。多少个Draw Call依赖于平台，因为每个Draw Call的最大缓存大小不同。在我的这个例子中，渲染要3个Draw Call。

注意，各个网格的绘制顺序跟我们提供的数据顺序相同。这里不会排序不会遮挡剔除，尽管一旦离开视锥体，整个批次都会消失。

# 2.5 动态合批
还有第三个方法来减少Draw Call, 被称之为 动态合批。这时个古老的技术，把多个共享材质的细小网格合并成一个大的网格来绘制。每个物体使用不同的材质属性时，它也不会生效。

大的网格生成困难，所以只适用于小的网格。球体太大了，但立方体可以。为了观察动态合批的行为，在CameraRender中，关闭GPU实例化，把enableDynamicBatching设置为true。
```c#
var drawingSettings = new DrawingSettings(
    unlitShaderTagId, sortingSettings
) {
    enableDynamicBatching = true,
    enableInstancing = false
};
```
同时禁用SRP批处理器，因为它的优先级更高。
```c#
GraphicsSettings.useScriptableRenderPipelineBatching = false;
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/cubes.png)

*绘制立方体*

一般情况下，GPU实例化比动态合批更好。动态合批也有一些注意事项，例如当涉及不同的尺度时，不能保证较大网格的法线向量是单位化。同时，绘制顺序会改变，因为它们时一个整体网格了。

静态合批的工作方式类似，但它是通过标记batching-static来预先生成。除了需要更多的内存和存储空间之外，没有什么注意的。渲染管线不会察觉，所以我们不必担心。

# 2.6 配置批处理
采用哪种方式最好可能会有所有不同，所以我们动态合批让可配置。首先添加一个变量控制，DrawVisibleGeomety是否使用动态和批或者GPU实例化。
```c#
void DrawVisibleGeometry (bool useDynamicBatching, bool useGPUInstancing) {
   var sortingSettings = new SortingSettings(camera) {
     criteria = SortingCriteria.CommonOpaque
   };
   var drawingSettings = new DrawingSettings(
        unlitShaderTagId, sortingSettings
   ) {
       enableDynamicBatching = useDynamicBatching,
       enableInstancing = useGPUInstancing
       };
       …
}
```
渲染器现在必须提供这个配置，然后以来RP来提供它。
```c#
public void Render(ScriptableRenderContext context, Camera camera,
    bool userDynamicBatching, bool userGUPInstancing)
{
    ...
    DrawVisibleGeometry(useDyanmicBatching, useGPUIntancing);
    ...
}```
CustomRenderPipeline通过构造方法接收这两个参数，并存储在字段里面，然后在把他们传递到渲染器中。
```c#
bool useDynamicBatching, useGPUInstncing;

public CustomRenderPipeline(bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher)
{
    this.useDynamicBatching = useDynamicBatching;
    this.useGPUInstancing = useGPUInstancing;
    GraphicsSetting.useScriptableRenderPipelineBatching = useSRPBatcher;
}

protected override void Render(ScriptableRenderContext context, Cameras[] cameras)
{
    foreach(Camera camera in cameras)
    {
        renderer.Render(
            context, camera, useDynamicBatching, useGPUInstancing
        );
    }
}
```
最后，添加三个选项作为配置字段到CustomRenderPipelineAsset中，通过调用CreatePipeline把他们传递到构造函数中。
```c#
[SerializeField]
bool useDynamicBatching = true, useGPUInstancing = true, useSRPBatcher = true;

protected override RenderPipeline CreatePipeline()
{
    return new CustomRenderPipeline(useDynamicBatching, useGPUInstancing, useSRPBatcher);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/rp-configuration.png)

*RP配置*

现在可以通过我们的RP改变这些方法。切换选项会立即生效，因为Unity编辑器检测到资源变化时，会立即创建一个新的RP实例。

# 透明度
我们的着色器可用于创建不透明材质。改变颜色的alpha值，通常表示透明，但是目前不会有效果。我们可以把渲染队列设置到*Transparent*，但是它是改变绘制顺序，而不会改变绘制方法。

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/reduced-alpha.png)

*降低透明，使用透明渲染队列*

我们不需要编写一个单独的着色器来之支持透明材质。只需在我们的Unlit着色器的基础上做一点工资，就能同时支持不透明和透明渲染。

# 混合模式
不透明和透明渲染的主要不同是，是否替换之前的绘制或是否与上一次绘制合成产生透视效果。我们可以通过设置源和目标的混合模式来控制。这里源指的是当前绘制的结果，目标指的是之前绘制的结果以及最终结果要绘制的地方。添加两个着色器属性，_SrcBlend 和_DstBlend。它们是混合模式的枚举值，但是我们能用的最合适的类型是Float，把源的默认值设置为1，目标的默认值设置为0。
```c#
Properties {
    _BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
    _SrcBlend ("Src Blend", Float) = 1
    _DstBlend ("Dst Blend", Float) = 0
}
```
为了让编辑更容易，我们添加一个Enum属性到属性，用全namespace定义的UnityEngine.Rendering.BlendMode枚举类型作为参数。
```c#
[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend ("Src Blend", Float) = 1
[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend ("Dst Blend", Float) = 0
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/opaque-blend-modes.png)

*不透明混合模式*

默认值表示不透明混合配置。源设置为1，表示把它完全叠加，而目标设置为0，表示它被忽略。

标准透明源的混合模式是SrcAlpha，它意思是渲染的颜色的RGB项乘以它的alpha项。所以，alpha越小，它就越弱。混合的目标源与之设置相反；OneMinusSrcAlpha(1-alpha), 总权重为1.

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/transparent-blend-modes.png)

*透明混合模式*

混合模式可以在Pass块中定义，使用Blend语句，后面中括号中跟着我们访问的属性。 这是着色器编程出现之前的旧语法。
```c#
Pass {
    Blend [_SrcBlend] [_DstBlend]

    HLSLPROGRAM
    …
    ENDHLSL
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/semitransparent-spheres.png)

*半透明的黄色球形*

# 3.2 不写深度值
透明渲染通常不会写入深度缓冲，因为它并不会因此而受益，反而有可能产生不希望的结果。我们可以通过ZWrite语句，控制是否写入深度。我们继续使用一个着色器属性，这次用_ZWrite。
```c#
Blend [_SrcBlend] [_DstBlend]
ZWrite [_ZWrite]
```
用自定义枚举值Enum(Off, 0, On, 1)定义这个着色器属性。Enum(Off, 0, On, 1)创造了0 1开关，默认值为1。
```c#
[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend ("Src Blend", Float) = 1
[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend ("Dst Blend", Float) = 0
[Enum(Off, 0, On, 1)] _ZWrite ("Z Write", Float) = 1
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/z-write-off.png)

*关闭深度写入*

# 纹理
以前我们使用一张alpha贴图来生成不均匀的半透明材质。让我们在着色器中添加_BaseMap纹理属性，也支持它。这时，属性类型是2D，我们采用Unity标准的白色材质作为默认值(*通过使用white字符串指定*)。同时纹理属性还需要以{}结尾。它在很早以前机用于控制纹理设置了，直到今天它任用用于防止一些清晰下防止一些奇怪的错误。
```c#
    _BaseMap("Texture", 2D) = "white" {}
    _BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/material-with-texture.png)

*有纹理的材质*

纹理需要上传到GPU内存中，Unity会帮我做这事。着色器需要一个相关纹理的句柄，就像我们定义uniform值一样，不同之处是我们使用TEXTURE2D这个宏传入名字作为参数。我们还需要给纹理定义个采样器状态，它控制如何采、考虑包装和过滤模式。它是通过SAMPLER这个宏来定义，做法类似TEXTURE2D不过名字要以sampler前缀。与Unity自动提供的采样器名字匹配。

纹理和采样器状态时着色器资源。它们不能为每个实例提供，必须声明为全局的。在UnitPass.hlsl的着色器属性之前，加上这些。
```c#
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```
除此之外，Unity还通过一个float4来实现纹理的平铺和偏移，变量名需要用纹理属性的名字加上_ST后缀。这个属性需要时UnityPerMaterial的一部分，因此它可以在每个实例中设置。
```c#
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```
采样纹理需要纹理坐标，它时顶点属性的一部分。确切的说，我们需要第一对纹理坐标，总共有8对。它是通过在Attributes中添加一个float2的字段，语义采样TEXTURE0。纹理空间尺寸通常用UV命名，因为它用于base map，所以我们会把它名为baseUV。
```c#
struct Attributes {
    float3 positionOS : POSITION;
    float2 baseUV : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```
我们需要把纹理坐标传到片段函数中，因为纹理采样在那里。所以也在Varyings中连接一个float2的baseUV。这次我们不需要添加特殊函数，它就是我们传递的数据，不需要GPU特殊注意。但是，我们还是要给他赋予一些意义。我们可以使用任何无用的标识符，就简单地用VAR_BASE_UV吧。
```c#
struct Varyings {
    float4 positionCS : SV_POSITION;
    float2 baseUV : VAR_BASE_UV;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```
在UnityPassVertex复制纹理坐标的时候，我们要应用_BaseMap_ST中的缩放和偏移。我在逐顶点中做，而不是在逐像素中。XY存储缩放，ZW存储偏移，我们可以通过swizzle访问属性。

```c#
Varyings UnlitPassVertex (Attributes input) {
    …

    float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseMap_ST);
    output.baseUV = input.baseUV * baseST.xy + baseST.zw;
    return output;
}
```
UV坐标现在通过三角形插值，可用于UnlitPassFragment了。通过SAMPLE_TEXTURE2D宏采用纹理，需要传入纹理、采样器状态、纹理坐标，作为参数。最终的颜色会把纹理采样的颜色和baseColor相乘。

```c#
float4 UnlitPassFragment (Varyings input) : SV_TARGET {
    UNITY_SETUP_INSTANCE_ID(input);
    float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
    float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
    return baseMap * baseColor;
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/textured.png)

*纹理映射的黄色球体*

因为纹理的RGB数据是均匀统一的白色，所以RGB颜色值不会受到影响(1*x=x)。但是alpha通道不一样，所以透明部分不在均匀。

# Alpha裁剪
另一重透视表面是通过镂空。通过丢弃一些渲染的片元，着色器也可以做到这一点。这个方法会产生硬边，而不是平滑的过度。这个技术就是alpha裁剪。通常的做法是定义个镂空的阈值。alpha在阈值之下的被丢弃，其他则保留。

添加一个_Cutoff属性，默认值设置为0.5。因为alpha值介于0~1,我们用Range(0.0, 1.0)作为它的类型。
```c#
    _BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
    _Cutoff ("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
```

把它也添加到UnlitPass.hlsl的材质属性中。
```c#
UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
```
我们可以在UnlitPassFragment中调用clip来丢弃片段。如果我们传入9或则更小，它会中并且丢弃片段。把最终颜色的透明度减去阈值，传入clip

```c#
    float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
    float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
    float4 base = baseMap * baseColor;
    clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
    return base;
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-inspector.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-scene.png)

*Alpha镂空设置为0.2*

材质通过用透明混合或则alpha裁剪，它们不会同时使用。一个典型的剪切材质是，完全不透明的，除了被丢弃的片段，并写入深度缓冲区。它使用AlphaTest渲染队列，这意味这它在所有不透明物体渲染之后渲染。这样做是因为丢弃片段使用一些GPU优化变得不可能，因为三角形不在认为是全球遮挡背后的物体。通过绘制一个完全不透明的物体，首先它可能覆盖透明裁剪物体的一部分，这样就不需要处理隐藏部分的片段。
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/clipped-inspector.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/clipped-scene.png)

*Alpha裁剪材质*

要优化这项工作，我们需要确保只有在需要的时候才使用clip。我们可以添加着色器功能属性开关。属性开关值默认为0，添加一个Toggle属性，用来控制着色器字段。它的名字不重要，用_Clipping即可。

```c#
_Cutoff ("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
[Toggle(_CLIPPING)] _Clipping ("Alpha Clipping", Float) = 0
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/alpha-clipping-off.png)

*透明裁剪关闭*

# 3.5 着色器功能
启用开关会添加_CLIIPING关键字到材质的激活的关键字列表中，关闭开关会把它移除。但是，这些不是自动的。我们需要告诉Unity，基于关键字是否开启，为我们的着色器编译不同的版本。通过添加#pragma shader_feature _CLIPPING指令到Pass中可以做到。

```c#
    #pragma shader_feature _CLIPPING
    #pragma multi_compile_instancing
```
现在Unity会编译一个带有和一个不带_CLIPPING的版本。它依赖我们的的材质配置，成一个或多个变体。因此我们让代码有前提条件，就像我们做include防御一样。我们可以用#ifdef _CLIPPING, 但我更喜欢 #if defined(_CLIPPING).

```c#
#if defined(_CLIPPING)
    clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
#endif
```

# 3.6 每个物体镂空
因为镂空是UnityPerMaterial缓冲的一部分，它可以每个实例单独配置。因此，让我们把它这个功能添加到PerObjectMaterialProperties中。与添之前添加颜色类似，除了我们SetFloat而不是SetColor。
```c#
static int baseColorId = Shader.PropertyToID("_BaseColor");
    static int cutoffId = Shader.PropertyToID("_Cutoff");

    static MaterialPropertyBlock block;

    [SerializeField]
    Color baseColor = Color.white;

    [SerializeField, Range(0f, 1f)]
    float cutoff = 0.5f;

    …

    void OnValidate () {
        …
        block.SetColor(baseColorId, baseColor);
        block.SetFloat(cutoffId, cutoff);
        GetComponent<Renderer>().SetPropertyBlock(block);
    }
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-per-object-inspector.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-per-object-scene.png)

*每个实例化对象单独alpha镂空*

# 3.7 Alpha裁剪小球
MeshBall也是一样的。刚才我们用一个裁剪材质，但是所有的实例最终具有同样的孔洞。
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/alpha-clipped-ball.png)

*密集的alpha裁剪的小球*

让我们添加一些变化，给每个实例一线随机旋转和0.5-1.5的随机所发。我们将改变它们颜色的alpha为0.5-1之间，而不是为每个实例设最cutoff阈值。这样控制不那么精确，但它只是一个随机的例子而已。
```c#
matrices[i] = Matrix4x4.TRS(
    Random.insideUnitSphere * 10f,
    Quaternion.Euler(
        Random.value * 360f, Random.value * 360f, Random.value * 360f
    ),
    Vector3.one * Random.Range(0.5f, 1.5f)
);
baseColors[i] =
    new Vector4(
        Random.value, Random.value, Random.value,
        Random.Range(0.5f, 1f)
    );
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/more-varied-ball.png)

*小球更加多变*

注意，Unity任然发送一个每个实例的cutoff数组到GPU，即便它们一模一样。它是材质属性的副本，所以改变材质的属性值，可以一次改变所有小球的孔洞。

到此为止，我们的无光照着色器，为下个教程的复杂着色器打好了基础。
