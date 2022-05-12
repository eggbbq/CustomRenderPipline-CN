# 方向光<sub>直接照明</sub>
- *使用法线向量计算光照*
- *支持四个方向光*
- *应用BRDF*
- *创造透明光照材质*
- *创建自定义着色器GUI面板*

这是[自定义可编程渲染管线](https://catlikecoding.com/unity/tutorials/custom-srp/)系列教程的第三篇。它新增了多个直接光着色。

教程使用Unity2019.2.12f1制作。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/tutorial-image.jpg)

*四个灯光照亮的各种小球*

# 1 光照
如果我们想创造更真实的场景，我们就必须模拟光线对物体表面的影响。这需要比我们目前这个无光照着色更复杂的着色器。

# 1.1 光照着色器
复制一份UnlitPass.hlsl重命名为LitPass.hlsl。调整引入防御定义，以及顶点和片段函数名。我们在稍后加入光照计算。
```c#
#ifndef CUSTOM_LIT_PASS_INCLUDED
#define CUSTOM_LIT_PASS_INCLUDED
...
Varyings LitPassVertex(Attributes input){...}
float4 LitPassFragment(Varyings input):SV_TARGET {...}
#endif
```
同样复制一份Unlit shader并且重命名为Lit。改变它的菜单名称、引入的文件，以及使用的函数名。让我们吧默认的颜色修改成灰色，因为一个纯白的表面在光照充足的场景中会特别亮。通用管线也是默认使用灰色。
```c#
Shader "Custom RP/Lit"
{
    Properties 
    {
        _BaseMap("Texture", 2D) = "white"{}
        _BaseColor("Color", Color) = (0.5, 0.5, 0.5, 1.0)
        ...
    }
    SubShader 
    {
        Pass
        {
            ...
            #pragma vertex LitPassVertex
            #pragma fragment LitPassFragment
            #include "LitPass.hlsl"
            ENDHLSL
        }
    }
}
```
我们即将使用自定义的光照方法，我们将通过把我们的着色器光照模式设置成*CustomLit*来表示。在Pass中添加一个Tags区块，包含"LightMode" = "CustomLit"。
```c#
    Pass
    {
        Tags
        {
            "LightMode" = "CustomLit"
        }
    }
```
要渲染使用我们这个通道的物体，我们必须把这个通道驾到CameraRender中。首先，为它添加一个着色器标签标识符。
```c#
static ShaderTagId unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit");
static ShaderTagId litShaderTagId = new ShaderTagId("CustomLit");
```
然后把它添加到DrawVisiableGeometry的通道中，就像之前在DrawUnsupportedShaders那样。
```c#
var drawingSetting = new DrawingSettings(unlitShaderTagId, sortingSettings)
{
    enableDynamicBatching = useDynamicBatching,
    enableInstancing = useGPUInstancing
};

drawingSettings.SetShaderPassName(1, listShaderTagId);
```
现在我们可以创建一个新的不透明材质，通过与创建无光照材质的方式一样。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/opaque-material.png)

*默认不透明材质*

# 法线向量
一个物体光照程度，取决于很多因素，其中就包括光线和物体表面的角度。要知道物体表面的方向值，我们就需要访问表面法向量，垂直表面向外的单位向量。这个向量是顶点数据的一部分，就跟位置一样定义在模型空间。因此把它添加到*LitPas*s的*Attributes*中。
```c#
struct Attributes
{
    float4 positionOS:POSTION;
    float3 normalOS:NORMAL;
    float2 baseUV:TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```
光照是逐片段计算，所以我们也把法线向量添加到Varyings中，把它命名为normalWS。
```c#
struct Varyings
{
    float4 positionCS:SV_POSTION;
    float4 normalWS:VAR_NORMAL;
    float2 baseUV:VAR_BASE_UV;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```
我们可以在LitPassVertex中，通过*SpaceTransfroms*中的*TransformObjectToWorldNormal*方法，把法线转换到世界空间。
```c#
output.positionWS = TransformObjectToWorld(input.positionOS);
output.positionCS = TransformWorldToHClip(positionWS);
output.normalWS = TransfromObjectToWorldNormal(input.normalWS);
```
>*TransfromObjectToWorldNormal*是怎么样工作的?<br>
如果你查看代码，你会看到有两个方法，你会看到它基于是否定义UNITY_ASSUME_UNIFORM_SCALING，选中两个方法中的一个。<br>
如果UNITY_ASSUME_UNIFORM_SCALING定义了，就调用*TransformObjectToWorldDir*, 除了忽略平移，它跟*TransformObjectToWorld*做的事情一样，就像用方向向量替换位置似的。但是因为向量被统一缩放了，所以之后还要把它单位化。<br>另一种情况是非统一缩放。这个就更复杂，因为当物体通过非统一缩放变形，法线向量需要反向缩放来匹配新的平面方向。这需要乘以转置矩阵（UNITY_MATRIX_I_M),然后单位化。![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/scaling-incorrect.png)![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/scaling-correct.png)<br>*错误和正确的法线转换*<br>使用UNITY_ASSUME_UNIFORM_SCALING是个小优化，你可以通过自己的定义来开启它。然而，当使用GPU实例化时，它就有很多不同点了，因为需要把UNITY_MATRIX_I_M矩阵发送到GPU。当不需要的时候避免它时值得的。你可以通过添加#pragma instancing_option assumeuniformscaling指令到我们的着色器中来开启它，但是只有你确定渲染的物体都是统一缩放时才这么做。

我们可以用一个颜色来验证，LitPassFragment是否获得了正确的法线向量。
```c#
base.rgb = input.normalWS;
return base;
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/world-normals.png)

*世界空间的法线向量*

负数值不能够被显示，因为会被夹持到0。

# 1.3 法线插值
尽管法线在顶点程序中是单位长度，三角形线性插值会影响他们的长度。我们可以通过渲染，10倍（1-向量的长度）来让这个误差更明显。
```c#
base.rgb = abs(length(input.normalWS) - 1.0) * 10.0;
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/interpolated-normal-error.png)

*夸张的，法线向量线性插值错误*

我们可在LitPassFragment中单位化法线向量来平滑误差。如果只是观察法线向量，它不是很明显，但是当它被用于光照时就会更明显。
```c#
base.rgb = normalize(input.normalWS);
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/normalization-after-interpolation.png)

*插值之后单位化*

# 1.4 表面属性
着色器中的光照就是模拟光线和它照射物体的相互作用，它意味着我们必须持续跟踪表面的属性。现在我们有法线和基本颜色。我们可以把后者分割成两个部分：RGB颜色和alpha值。我们绘制多个地方使用它，因此让我们定义一个方便的*Surface*结构体来包含所有相关的数据。把这个*Surface*HLSL文件放入一个独立放在*ShaderLibrayry*文件夹中。
```c#
#ifndef CUSTOM_SURFACE_INCLUDED
#define CUSTOM_SURFACE_INCLUDED
struct Surface
{
    float3 normal;
    float3 color;
    float alpha;
}
#endif
```
>我们不应该把法线定义为normalWS吗?<br>
可以这么做，但是表面并不管线法线是在哪个空间中定义的。光照计算可以在任何适当的3D空间中执行。所以我们就去掉了空间相关的定义。当要填充数据的时候，我们必须在任何地方都用同个空间。我们将使用世界空间，之后我们也可以切换到另一个空间，一切还是同样起作用。

把它引入到LitPass中，放到Common之后。这样我们可以保持LitPass简短。从现在开始，我们会把特定的代码放在它自己的HLSL文件中，方便查找相关的功能。
```c#
#include "../ShaderLibrary/Common.hlsl"
#include "../ShaderLibrary/Surface.hlsl"
```
在LitPassFragment中定义一个*surface*变量，然后填充它。
```c#
Surface surface;
surface.normal = normalize(input.normalWS);
surface.color  = base.rgb;
surface.alpha  = base.a;

return float4(surface.color, surface.alpha);
```
>这样的代码不会效率低下吗?<br>
不会有什么区别的，因为着色器编译器会生成高度优化的代码，完全重写我们的代码。结构体完全是为了我们方便。你可以通过着色器的查看面板下面的查看代码按钮，查看编译结果。

# 1.5 光照计算
要计算实际的光照，我们创建一个*GetLighting*方法，需要传入一个*Surface*参数。一开始让它返回表面法线的Y值。因为我们会把它光照功能放到一个独立的*Lighting HLSL*文件中。
```c#
#ifndef CUSTOM_LIGHTING_INCLUDED
#define CUSTOM_LIGHTING_INCLUDED

float3 GetLighting (Surface surface) {
    return surface.normal.y;
}
#endif
```
把它引入到LitPass中，放在引入*Surface*之后，因为*Lighting*依赖它。
```c#
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```
>为什么不在*Lighting*中引入*Surface*?<br>
可以这么做，但是会导致多个文件交叉依赖。我选中把所有的引入语句放在一个地方，它能使依赖清晰。也方便替换文件，来改变着色器的工作方式，只要新的文件依赖同样的功能。

线我们可以在LitPassFragment中获得光照，用于片段的RGB部分。
```c#
float3 color = GetLighting(surface);
    return float4(color, surface.alpha);
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/diffuse-lighting-from-above.png)

*从上面照射的漫反射*


