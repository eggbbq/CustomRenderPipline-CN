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
调整*Light*中_CustomLighth缓冲区以匹配我们新的数据格式。这里我们将明确使用float4数组类型。在着色器中，数组大小是固定的，不能重设大小。确保使用最大长度与我们*Lighting*中定义的一致。
```c#
#define MAX_DIRECTIONAL_LIGHT_COUNT 4

CBUFFER_START(_CustomLight)
    //float4 _DirectionalLightColor;
    //float4 _DirectionalLightDirection;
    int _DirectionalLightCount;
    float4 _DirectionalLightColors[MAX_DIRECTIONAL_LIGHT_COUNT];
    float4 _DirectionalLightDirections[MAX_DIRECTIONAL_LIGHT_COUNT];
CBUFFER_END
```
添加一个函数来获取方向灯光的数量，调整GetDirectionalLight以便获取对应索引的数据。
```c#
int GetDirectionalLightCount()
{
    return _DirectionalLightCount;
}

Light GetDirectionalLight(int index)
{
    Light light;
    light.color = _DirectionalLightColors[index].rgb;
    light.direction = _DirectionalLightDirections[index].xyz;
    return light;
}
```
>rgb和xyz之间有什么区别?<br>
他们都是语义别名。rgba xyzw是一样的。

然后调整GetLight，以便使用一个for循环累加所有方向光的贡献。
```c#
float3 GetLight(Surface surface)
{
    float3 color = 0.0;
    for(int i=0; i < GetDirectionalLightCount();i++)
    {
        color += GetLighting(surface, GetDirectionalLight(i));
    }
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lights/four-directional-lights.png)

*4个方向灯*

此时我们的着色器支持到了4个方向灯光。通常只有一个方向灯光需要用来表示太阳或月亮，但是可能又个星球上存在多个太阳也说不准。方向灯也可以用来近似多个大型的灯光装置，比如说体育场的灯光。

如果你的游戏一直只有一个方向灯，你就可以摆脱循环或多个着色器变体。但是在这个教程中我们会保持简单，坚持使用一个通用的循环。最好的性能总是需要剔除任何你不需要的东西，尽管它不明显。

# 2.7 着色器目标级别
对着色器来说，可变长度的循环曾经是一个问题，但是现代的GPU能毫无问题的处理好他们，特别一个draw call的所有片段以同样的方式迭代同一个数据是。然而，OpenGL2.0和WebGL1.0图形API默认不支持。我们可以通过硬编码求最大值来实现，比如说GetDirectinalLight 返回 min(_DirectionalLightCount, MAX_DIRECTIONAL_LIGHT_COUNT)。这样可以把循环展开成一个条件代码序列。不幸的这个，生成的代码乱七八糟，性能迅速下降。在老式硬件中，所有的代码块都会被执行，通过条件赋值来控制使用哪个区块的执行结果。虽然我们可以完成这个任务，但是它会使代码变得复杂，因为我们还要调整其他的一些东西。因此我选择无视这些限制，为了简单起见，在构建时不支持WebGL1.0和OpenGL ES2.0。它们无论如何也支持不了线性光照。我们可以通过#pragma target 3.5指令提升着色器通道的目标级别到3.5来避免编译OpenGL ES2.0的着色器变体。
```c#
HLSLPROGRAM
#pragma target 3.5
…
ENDHLSL
```

# 3 BRDF
我们目前使用的是非常简单的光照模型，仅适用于完全漫反射表面。我们可以我们可以通过**双向反射率分布函数**(简称BRDF)，实现更真实和多变的光照。BRDF有很多实现方式。我们使用和URP管线一样的方法，此方法牺牲一部分真实感来换取性能。

# 入射光线
当一束光线从上方垂直照射一个表面片段时，所有的能量都会影响这个片段。为了简单，我们假设这一束光线的宽度和片段的宽度一致。这种情况光线的方向和法线方向对齐，因此 N.L = 1。当它们不对齐时，至少有一部分光线不能命中片段，所以影响片段的能量会更少。影响片段的能量就是 N.L。如果这个值为负数，就意味着它是背光面，所以不受光线影响。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/incoming-light.png)

*入射光部分*

# 出射光线
我们不能直接看到到达表面的光线，我们只能看到表面反弹且到达相机或者我们的眼睛的那部分光线。如果表面是一个完美的平坦镜面，光线反射时，入射角等于出射角。这被称之为镜面反射。这是一个简化的光线和表面互动的模型，但是他对我们的目的来说够用了。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/specular-reflection.png)

*完美镜面反射*

但是，如果表面不是完美平坦，光线就会散射，因为一个片段实际上被考虑为是多个更小的多个不同朝向的片段组成。这就把一束光线分割成多个更小的不同朝向的光束，它实际上使镜面反射变得模糊。最终我们看到一些光线散射，即使没有完美对齐。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/scattered-reflection.png)

*散射的镜面反射*

除此之外，光线还会穿透表面，弹射，以不同的角度射出，当然还有些其他因素我们暂时不用考虑。极端情况下，我们最终会得到一个完美的漫反射，在所有的方向均匀散射光线。这就是目前在我们着色器中计算的光照。

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/diffuse-reflection.png)

*完美漫反射*

无论G相机在哪儿，表面接受到的漫反射都是一样的。这就意味着，我们观察到到光的能量远远小于到达表面片段的能量（参考上图）。这表明我们需要按某个因子来缩小入射光。因为这个因子总是相同，我们可以把它烘焙成光的颜色和强度。因此，我们用最终光的颜色表示，从正面照射完美白漫反射表面反射时的观察量。这实际上是射出光的一小部分。还有其他灯光的配置方法，如流明度和勒克斯，这样可以更简单的配置真实光源，但是我们还是用到当前的方法。

# 3.3 表面属性
表面可以是完美的漫反射，完美镜面反射，或者介于二者之间，我有多种方式来控制它。我们将使用**金属工作流**，它需要我们在*Lit*着色器中添加两个表面属性。

第一个属性，是表面是否是金属和非金属，也称为电解质。因为一个表面同时混合二者属性，我们给它添加一个0-1的滑动条，1表示表面全金属，0表示全电解质。

第二个属性控制表面平滑度。同样给它一个0-1的滑动条，0表示绝对粗糙，1表示绝对平滑，默认值给到0.5。

```C#
_Metallic("Metallic", Range(0, 1)) = 0
_Smoothness("Smoothness", Range(0, 1))=0.5
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/metallic-smoothness.png)

*带有金属和平滑度滑动条的材质*

把这两个属性添加到UnityPerMaterial缓冲区。
```c#
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaeColor)
    UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
    UNITY_DEFINE_INSTANCED_PROP(float, _Metallic)
    UNITY_DEFINE_INSTANCED_PROP(float, _Smoothness)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```
同时也把他们添加到Surface结构体中。

```c#
struct Surface
{
    float3 normal;
    float3 color;
    float alpha;
    float metallic;
    float smoothness;
}
```

在*LitPassFragment*中拷贝他们到Surface中.
```c#
Surface surface;
surface.normal = normalize(input.normalWS);
surface.color = base.rgb;
surface.alpha = base.a;
sufrace.metallic = UNITY_ACCESS_INSTANCED_PROP(UniryPerMaterial, _Metallic);
surface.smoothness = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Smoothness);
```
同时也在PerObjectMaterialProperties中支持它们。
```c#
static int baseColorId = Shader.PropertyToID("_BaseColor");
static int cutoffId = Shader.PropertyToID("_Cutoff");
static int metallicId = Shader.PropertyToID("_Metallic");
static int smoothnessId = Shader.PropertyToID("_Smoothness");
...

[SerializedField, Range(0,1)]
float alphaCutoff -= 0.5f, metallic =0f, smoothness = 0.5f;
...

void OnValidate()
{
    ...
    block.SetFloat(metallicId, metallic);
    block.SetFloat(smoothnessId, smoothness);
    GetComponent<Render>().SetPropertyBlock(block);
}
```

# 3.4 BRDF 属性
我们用表面属性计算BRDF方程。BRDF告诉我们最终看到多少光从表面反射，它是由漫反射和镜面反射组成。我们需要把表面颜色的漫反射和镜面反射部分分开，同时我们需要知道表面的粗糙度。我们把这三个值定义到一个BRDF结构体中，放在一个单独的BRDF HLSL文件中。

```c#
#define CUSTOM_BRDF_INCLUDED
#define CUSTOM_BRDF_INCLUDED
struct BRDF
{
    float3 diffuse;
    float3 specular;
    float roughness;
}
#endif
```
添加一个函数来获得给定表面的BRDF数据。我们从一个完全漫反射表面开始，所以漫反射部分等于表面颜色，镜面反射为黑色，粗糙度为1。
```c#
BRDF GetBRDF(Surface surface)
{
    BRDF brdf;
    brdf.diffuse = surface.color;
    brdf.specular = 0.0;
    brdf.roughness = 1.0;
    return brdf;
}
```
在Light之后Lighting之前添加BEDF。
```c#
#include "../ShaderLibrary/Common.hlsl"
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Light.hlsl"
#include "../ShaderLibrary/BRDF.hsls"
#include "../ShaderLibrary/Lighting"
```
在两个GetLighting函数中都加入一个BRDF参数，然后用它的漫反射部分乘以入射光。
```c#
float3 GetLighting(Surface surface, BRDF brdf, Light light){
    return IncomingLight(surface, light) * brdf.diffuse;
}

float3 GetLighting(Surface surface, BRDF brdf)
{
    float color = 0.0;
    for (int i=0;i< GetDirectionalLightCount();i++)
    {
        color += GetLighting(surface, brdf, GetDirectionalLight(i));
    }
}
```
最后，在LitPassFragment中获取BRDF数据，把它传递到GetLighting中。
```c#
BRDF brdf = GetBRDF(surface);
float3 color = GetLighting(surface, brdf);
```
# 3.5 反射率
各种表面的反射率是不同的，通常金属的光照反射都是通过镜面反射，没有漫反射。因此，我们会用表面金属度属性来等效描述反射率。反射的光不会漫反射，因此漫反射的颜色应该乘以（1- 反射率）。
```c#
float oneMinusReflectivity = 1.0 - surface.metallic;
brdf.diffuse = surface.color * oneMinusReflectivity;
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/reflectivity.png)

*金属度为0，0.25, 0.5, 0.75, 1的白球*

事实上，还是有一部分光会在电介质表面弹射，使它获得高光。非金属的反射率不同，但是平均在0.04左右。我们来定义一个最小反射率和一个OneMinusReflectivity函数，把原理的0-1调整到0-0.96。这个方法跟URP的方法一样。

```c#
#define MIN_REFLECTIVITY 0.04

float OneMinusReflectivity(float metallic)
{
    float range = 1.0 - MIN_REFLECTIVITY;
    return range - metallic * range;
}
```
在GetBRDF函数中，使用这个函数来强制约束最小值。仅仅渲染漫反射时，*上面的做法*不会产生太多差异，但是如果加上镜面反射，它就重要得多。没有它非金属不会获得镜面高光。
```c#
float oneMinusReflectivity = OneMinusReflectivity(surface.metallic);
```
# 3.6 镜面反射颜色

