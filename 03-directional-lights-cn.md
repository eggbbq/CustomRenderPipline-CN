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
复制一份UnlitPass.hlsl重命名为LitPass.hlsl。调整导入防御定义，以及顶点和片段函数名。我们在稍后加入光照计算。
```c#
#ifndef CUSTOM_LIT_PASS_INCLUDED
#define CUSTOM_LIT_PASS_INCLUDED
...
Varyings LitPassVertex(Attributes input){...}
float4 LitPassFragment(Varyings input):SV_TARGET {...}
#endif
```
同样复制一份Unlit shader并且重命名为Lit。改变它的菜单名称、包含的文件，以及使用的函数名。让我们吧默认的颜色修改成灰色，因为一个纯白的表面在光照充足的场景中会特别亮。通用管线也是默认使用灰色。
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

把它包含到LitPass中，放到Common之后。这样我们可以保持LitPass简短。从现在开始，我们会把特定的代码放在它自己的HLSL文件中，方便查找相关的功能。
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
把它包含到LitPass中，放在包含*Surface*之后，因为*Lighting*依赖它。
```c#
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```
>为什么不在*Lighting*中包含*Surface*?<br>
可以这么做，但是会导致多个文件交叉依赖。我选中把所有的包含语句放在一个地方，它能使依赖清晰。也方便替换文件，来改变着色器的工作方式，只要新的文件依赖同样的功能。

线我们可以在LitPassFragment中获得光照，用于片段的RGB部分。
```c#
float3 color = GetLighting(surface);
    return float4(color, surface.alpha);
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/diffuse-lighting-from-above.png)

*从上面照射的漫反射*

此时的结果是表面法线的Y值。所以，在小球的上部分，从顶端到中间Y值由1递减到0。在小球的下部分，从中间到底部，Y值从0递减到-1，但是我们看不到负数的。这个结果与法线和上方向向量的余弦夹角匹配。（*这个地方是模拟光线从小球正上方照射，上方向向量就是这个光照方向向量*）丢弃到负数部分，结果看起来就像直接光线直接向县照射到漫反射一样。最后一步是把表面颜色加入到*GetLighting*的结果中，把它认为是表面的反照率（albedo）。
```c#
float3 GetLighting(Surface surface)
{
    return surface.normal.y * surface.color;
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/albedo.png)

*应用反照率*

>反照率是什么意思?<br>
反照率在拉丁语中表示**白色度**。它是衡量表面有多少漫反射的量。如果反照率不是全白，就意味着一部分光能被表面吸收，而没有反射。

# 2 灯光
要执行恰当的光照，我们还需要制动灯光的属性。这篇教程中，我们会限制值用方向光。方向光表示光源很远，它的位置无关紧要，只有方向。这是个简化模式，但是它足够用来模拟地球上的太阳光和其他一些入射光方向几乎只有一个方向的情形。

# 2.1 灯光结构体
我们用一个结构体来存储光线数据。现在我们只需要颜色和方向就够了。把它放在一个独立的*Light*HLSL文件中。同时定义一个*GetDirectionalLight*函数返回一个配置的方向光。用一个白色和向上的矢量初始化，匹配我们目前使用光线数据。注意，光线的方向用光线来的方向定义，而不是光线离开的方向（*这句话有点绕，应该这样说。光线的方向定义正好与光线的照射方向相反，这个是图形学里面这么约定的）。
```c#
#ifndef CUSTOM_LIGHT_INCLUDE
#define CUSTOM_LIGHT_INCLUDE
struct Light
{
    float3 color;
    float3 direction;
};

Light GetDirectionLight()
{
    Light light;
    light.color = 1.0;
    light.direction = float3(0.0, 1.0, 0.0);
    return light;
}
#endif
```
在LitPass中*Lighting*之前包含这个文件。
```c#
#include "../ShaderLibrary/Light.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```
添加一个*IncomingLight*到*Lighting*文件中，用它计算给点表面的入射光线量。对于任意光线方向，我们要求表面法线的点积。我们可以用*dot*函数求点积。这个结果应该有光线的颜色调节。

>光照这部分，作者写得不是太清晰。如果不了解光照模型的，这部分应该去查阅经典光照模型(Lambert/Blinn-Phong)，以便于系统的了解光照。方向这部分很简单，内容也不多。
```c#
float3 IncomingLight (Surface surface, Light light)
{
    return dot(surface.normal, light.direction) * light.color;
}
```
>什么是**点积**?<br>
这部分不翻译。因为向量/矩阵是学习的必要前提，并且作者这部分也是简略一说。相关的知识，你需要在回顾一下线性代数。

但是只有在光线与表面朝向才正确。当点积为负数时，我们需要动工*saturate*函数把它挟持到0。
```c#
float3 IncomingLight(Surface surface, Light light)
{
    return saturate(dot(surface.normal, light.direction)) * light.color;
}
```
>saturate做了什么?<br>
它把值挟持到[0,1]这个范围。因为点积用于不会大于1，所以我们只需要约束最小值。

添加另一个*GetLighting*函数，返回表面和光线的最终光照。现在是入射光线乘以表面颜色。在其他函数之前定义它。
```c#
float3 GetLighting (Surface surface, Light light)
{
    return IncomingLight(surface, light) * surface.color;
}
```
最后，重写一个*GetLighting*函数，只需要一个表面参数，它调用另一个同名函数，使用*GetDirectionalLight*提供光线数据。
```c#
float3 GetLighting (Surface surface)
{
    return GetLighting(surface, GetDirectionalLight());
}
```
# 2.3 将光线数据发送到GPU
我们应该使用当前场景的光线，而不是向上面那样，总是使用一个白色的光线。默认的场景中有个方向光，它表示太阳，它略带黄色(#FFF4d6), 绕X周旋转50<sup>o</sup>，绕Y轴-30<sup>o</sup>旋转。如果这个灯光不存在的话，就创建一个。

为了让着色器可以访问光线数据，我们需要创建统一变量，就像着色器属性一样。这里，我们定义两个float3向量：_DirectionalLightColor和_DirectionalLightDirection。把他们放到顶部的*Light*中的定义的_CustomLight的缓冲区中。
```c#
CBUFFER_START(_CustomLight)
    float3 _DirectionalLightColor;
    float3 _DirectionalLightDirection;
CBUFFER_END
```
用这些值，替换GetDirectionalLight中的常数。
```c#
Light GetDirectionalLight ()
{
    Light light;
    light.color = _DirectionalLightColor;
    light.direction = _DirectionalLightDirection;
    return light;
}
```
现在我们的RP必须要把光线数据发送到GPU。我们将创建一个新*Lighting*类来做这件事。它与*CameraRenderer*的工作类似。创建一个Setup方法，方法需要一个上下文参数，并且在这个方法中调用过一个独立的SetupDirectionalLight方法。在指向完成之后，给他指定一个专门的命令缓冲区，虽然这不是很有必要，但是可以方便我们调试。另一个方法是添加缓冲区参数。
```c#
using UnityEngine;
using UnityEngine.Rendering;

public class Lighting
{
    const string bufferName = "Lighting";
    CommandBuffer buffer = new CommandBuffer(){
        name = bufferName
    };

    public void Setup(ScriptableRenderContext context)
    {
        buffer.BeginSample(bufferName);
        SetupDirectionalLight();
        buffer.EndSample(bufferName);
        context.ExecuteCommnadBuffer(buffer);
        buffer.clear();
    }

    void SetupDirectionalLight(){}
}
```
保持跟踪这两个着色器标识符。
```c#
static int dirLightColorId = Shader.PropertyToID ("_DirectionalLightColor");
static int dirLightDirectionId = Shader.PropertyToID("_DirectionalLightDirection");
```
我们可以通过RenderSettings.sun,获得场景的主光源。它默认提供给我们最重要的方向光，也可以通过*Window/Rendering/Lighting Settings*显示配置。使用CommandBuffer.SetGlobalVector把灯光的数据发送到GPU。颜色是灯光的线性空间颜色，方向是光线变换的前方方向取取反值。
```c#
void SetupDirectionalLight()
{
    Light light = RenderSettings.sun;
    buffer.SetGlobalVector(dirLightColorId, light.color.linear);
    buffer.SetGlobalVerctor(dirLightDirectionId, -light.transform.forward.direction);
}
```
>SetGlobalVector不需要Vector4吗?<br>
需要的，发送到GPU的向量总是4个项，即使我们定义更少项也是如此。多余4个，则会被着色器隐藏。同时，Vector3能隐式转换为Vector4，尽管它们表示的不是同一个方向。

灯光的颜色属性是它的配置颜色，但是灯光还有一个单独的强度系数。光线最终的颜色至都必须乘上这个系数。
```c#
buffer.SetGlobalVector(dirLightColorId, light.color.linear * light.intensity);
```
给CameraRenderer一个Lighting实例，在绘制可见物体之前，用它设置光线。
```c#
Lighting lighting = new Lighting();
public void Render (ScriptableRenderContext context, Camera camera,
    bool useDynamicBatching, bool useGPUInstancing)
{
    …
    Setup();
    lighting.Setup(context);
    DrawVisibleGeometry(useDynamicBatching, useGPUInstancing);
    DrawUnsupportedShaders();
    DrawGizmos();
    Submit();
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lights/lit-by-sun.png)

*被太阳照亮*

# 2.4 可见灯光
当计算剔除时，Unity还会计算出哪些灯光影响相机可见空间。我们可以依靠这些信息，而不是全局太阳光。要这样做，*Lighting*需要访问剔除结果，隐藏给Setup添加一个参数，为了方便把它保存在字段中。然后我们能支持不止一个灯光，所以用一个新的方法SetupLights替换SetupDirectionalLight。
```c#
CullingResults cullingResults;

public void Setup (ScriptableRenderContext context, CullingResults cullingResults)
{
    this.cullingResults = cullingResults;
    buffer.BeginSample(bufferName);
    //SetupDirectionalLight();
    SetupLights();
    …
}

void SetupLights () {}
```
在调用CameraRenderer.Render的Setup时，添加剔除结果作为参数。
```c#
lighting.Setup(context, cullingResults);
```
此时，Lighting.SetupLights可以通过剔除结果的visibleLights这个属性获得需要的数据。
```c#
using Unity.Collections;
using UnityEngine;
using UnityEngine.Rendering;

public class Lighting
{
    …
    void SetupLights ()
    {
        NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
    }
}
```
>什么时NativeArray?<br>
它是类似于数组的结构，但是提供连到接本地内存缓冲区。它让托管的c#代码与本机引擎代码共享数据更加高效。

# 2.5 多个方向灯光
使用可见灯光数据让支持多个方向灯光成为可能，但是我们需要发送所有这些灯光的数据到GPU。所以，我们会使用两个Vector4数组替代一对向量，同时加上要给灯光数量的整数。我还定义一个醉倒的方向灯数量，用它来初始化了两个数据缓冲数组。让我们将最大值设置为4，对大多数场景来说应该足够了。
```c#
const int maxDirLightCount = 4;

//static int dirLightColorId = Shader.PropertyToID("_DirectionalLightColor");
//static int dirLightDirectionId = Shader.PropertyToID("_DirectionalLightDirection");
static int dirLightCountId = Shader.PropertyToID("_DirectionalLightCount");
static int dirLightColorsId = Shader.PropertyToID("_DirectionalLightColors");
static int dirLightDirectionsId = Shader.PropertyToID("_DirectionalLightDirections");

static Vector4[] dirLightColors = new Vector4[maxDirLightCount];
static Vector4[] dirLightDirections = new Vector4[maxDirLightCount];
```
添加一个索引和一个VisibleLight参数到SetupDirectionalLight。用提供的索引设置颜色和方向元素。在这种情况下，最终的颜色通过VisibleLight.finalColor属性提供。向前向量可以通过VisibleLight.localToWorldMaxix获得，它在这个矩阵的第三列，同样取反。
```c#
void SetupDirectionalLight (int index, VisibleLight visibleLight) {
    dirLightColors[index] = visibleLight.finalColor;
    dirLightDirections[index] = -visibleLight.localToWorldMatrix.GetColumn(2);
}
```
最终的颜色已经应用了灯光的强度，但是默认情况Unity不会把它转换到线性空间。我们需要设置GraphicsSettings.lightsUseLinearIntensity为true，我们可以在CustomRenderPipeline的构造函数中设置一次。
```c#
public CustomRenderPipeline(bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher)
{
    this.useDynamicBatching = useDynamicBatching;
    this.useGPUInstancing = useGPUInstancing;
    GraphicsSettings.useScriptableRenderPipelineBatching = useSRPBatcher;
    GraphicsSettings.lightsUseLinearIntensity = true;
}
```
下一步，在Lighting.SetupLights中遍历所有的可见灯光同时对每个灯光调用SetupDirectionalLight。然后调用缓冲区的SetGlobalInt和SetGlobalVectorArray把数据发送到GPU。
```c#
NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
for (int i = 0; i < visibleLights.Length; i++)
{
    VisibleLight visibleLight = visibleLights[i];
    SetupDirectionalLight(i, visibleLight);
}

buffer.SetGlobalInt(dirLightCountId, visibleLights.Length);
buffer.SetGlobalVectorArray(dirLightColorsId, dirLightColors);
buffer.SetGlobalVectorArray(dirLightDirectionsId, dirLightDirections);
```
但是我们最多支持个方向光，因此我们需要在循环到最大的时候中断。
```c#
int dirLightCount = 0;
for (int i = 0; i < visibleLights.Length; i++)
{
    VisibleLight visibleLight = visibleLights[i];
    SetupDirectionalLight(dirLightCount++, visibleLight);
    if (dirLightCount >= maxDirLightCount)
    {
        break;
    }
}
buffer.SetGlobalInt(dirLightCountId, dirLightCount);
```
因为我们只支持方向光，所以我们要忽略其他灯光类型。我们可以通过检查可见灯光的lightType属性是否等于LightType.Directional。
```c#
VisibleLight visibleLight = visibleLights[i];
if (visibleLight.lightType == LightType.Directional)
{
    SetupDirectionalLight(dirLightCount++, visibleLight);
    if (dirLightCount >= maxDirLightCount)
    {
        break;
    }
}
```
虽然这样也可以正常工作，但是VisibleLight这个结构体太大了。理想情况是，我们只需要从原生数组中获取它一次，不会以普通参数的形式传递到SetupDirectionalLight, 因为这样做会拷贝这个结构体。我们可以使用和ScriptablerRenderContext.DrawRenderer方法一样的伎俩，通过引用传递参数。
```c#
SetupDirectionalLight(dirLightCount++, ref visibleLight);
```
它需要我们把参数定义成一个引用。
```c#
void SetupDirectionalLight (int index, ref VisibleLight visibleLight) { … }
```
# 着色器循环