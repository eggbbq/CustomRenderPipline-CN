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
//	return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
//}

//float4 TransformWorldToHClip (float3 positionWS) {
//	return mul(unity_MatrixVP, float4(positionWS, 1.0));
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