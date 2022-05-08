# 自定义渲染管线 <sub>控制渲染</sub>

1. *创建一个渲染管线资源和管线实例*
2. *渲染相机视图*
3. *执行剔除，过滤，排序*
4. *分离不透明，透明和无效通道*
5. *多相机工作*

这是[自定义可编程渲染管线](https://catlikecoding.com/unity/tutorials/custom-srp/)系列教程的第一个部分。它涵盖了我们将要扩展的渲染管线的基本结构的初始创建过程。

这个系列假定你通过了[对象管理](https://catlikecoding.com/unity/tutorials/object-management/)系列和[程序化网格](https://catlikecoding.com/unity/tutorials/procedural-grid/)系列教程。

这个教程使用Unity 2019.2.6f1制作(*实际上，你可以用以上的任意版本来做*)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/tutorial-image.jpg)

*使用自定义渲染管线渲染*

# 1 新的渲染管线
要渲染任何东西，Unity需要确定它的形状，位置，时间，以及用什么样的设置。它可能会很复杂，这取决与涉及多少效果。灯光，阴影，透明，图像特效，体积特效等等，在渲染到最后的图像之前都需要正确有序的处理。这就是渲染管线要做的。

在之前，Unity在内置管线中支持很少的渲染事情。Unity 2018 引入了可编写脚本的渲染管线（简称 RP），使我们可以做任何我们想做的事情，同时仍然能够依赖 Unity 进行基本步骤，例如剔除。Unity2018还用这个新的方法，添加了两个实验性的RP：轻量级管线和高清管线。在Unity2019中，轻量级管线不在实验性的了，它被更名为通用渲染管线。

通用渲染管线(Universal RP)注定要替换当前旧版的管线，成为默认管线。其想法是，这是个万能的管线，同时又很容易自定义。本教程不会涉及该管线，而是从头设计一个整个管线。

这个教程为，使用最小的渲染管线，用前向渲染管线绘制无光照形状，打基础。当这些都完成了，我们就可以在之后的教程中扩展我们的管线，添加光照，阴影，不同的渲染方法，和更多其高级功能。


## 1.1 项目设置

用Unity2019.2.6+创建一个新的3D项目。我们将创建自己的管线，隐藏不要选择RP项目模版。打开项目你可以通过Package Manager删除所有不需要的包。在这个教程中我们只需要Unity UI包，来实验绘制UI，所以你可以保留它。

我们将只工作在线性颜色空间(Linear Colr Space)，但是Unity2019默认还使用伽马空间(Gamma Space)。通过Editor/Project Setting，在其他设置中把颜色空间（Color Space）选择为线性（Linear）。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/color-space.png)

*颜色空间设置成线性*

创建几个物体到默认场景中，使用混合标准的，不透明的和透明材质。Unit/Transparent着色器只需要一个纹理就可以工作，这里为此准备了一个UV球面贴图。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/sphere-alpha-map.png)

*UV球面alpha贴图，黑色背景*

我放置了几个立方体在测试场中，他妈都是不透明的。红色的立方体使用标准着色器（Standard Shader），绿色和黄色的立方体使用Unlit/Color着色器。
蓝色的小球使用标准着色器（Standard Shader）并且设置为透明模式，白色小球使用Unlit/Transparent着色器。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/scene.png)

*测试场景*

## 1.2 管线资源
此时，Unity使用的是默认管线。要替换成自定义管线，我们首先要为此创建一个资源类型。我们使用和通用渲染管线相同的文件夹结构。创建一个CustomRenderPipeline的C#脚本类。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/folder-structure.png)

*文件夹结构*

这个资源类型必须继承自UnityEngine.Rendering.RenderPipelieAsset.

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderingPipelineAsset : RenderPipelineAsset {}
```
渲染管线资源（RP Asset）的主要目的是让Unity有个方法可以持有一个负责渲染的管线对象实例。资源本身只是持有和存设置的空间。我们还没有任何设置，现在要做的是给Unity一个方式来获取我们的管线对象实例。这是通过重写抽象方法CreatePipeline，它需要返回一个 RenderPipeline实例。但是我们现在还没有定义自定义RP类型，所以现在返回null。

CreatePipeline方法使用proected修饰，它意味着只有RenderPipline子类可以访问。

```c#
protected override RenderPipeline CreatePipelie() {
    return null;
}
```

现在我们要添加一个资源类型到我们的项目中。
```c#
[CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline"")]
public class CustomRenderPipelineAsset : RenderPipeline {...}
```
**CreateAssetMenu** 这个属性的用法参考官方文档，就不翻译了。

现在去项目的图像设置（Graphics）中在Scriptable Render Settings中选择我们创建的资源

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/srp-settings.png)

**选择自由定义RP资源**

替换默认的RP改变了一些东西。首先，一些选项从Graphics Settings信息面板中消失了。其次，我们禁用了默认管线同时又没有提供一个有效的替代，因此什么都不在渲染了。游戏窗口，场景窗口，材质预览都不在起作用。如果你通过FrameDebuger查看，你会发现游戏窗体中确确实实什么都没有绘制。

## 1.3 渲染管线实例

创建一个叫CustomRenderPipeline的类，放在和CustomRenderPipelineAsset同一个文件中。这将是是我们RP实例返回的类型，因此他必须继承RenderPipeline, 重写Render方法，它有两个参数，一个是ScriptableRenderContext，一个是Camera数组，现在让方法空着。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderPipeline : RenderPipeline
{
    protected override void Render (
        ScriptableRenderContext context, Camera[] cameras
    )
    {
    }
}
```

回到CustomRenderPipelineAsset.CreatePipeline方法中，返回CustomRenderPipeline。它会给我一个合法且有功能的管线，然而它还是什么都不会渲染。
```c#
protected overrde RenderPipeline CreatePipeline()
{
    return new CustomRenderPipeline();
}
```

# 2 渲染

每一帧Unity都会调用RP实例的Render方法。它传递一个上下文结构体，该结构用于连接本机引擎，我们用它来渲染。它还传递一组相机，因为场景中可能有多个激活的相机。这就是RP的职责，按照相机的顺序渲染他们。

## 2.1 相机渲染器

每个相机独立渲染。因此，与其把所有的相机都交给CustomRenderPipeline渲染，我们不如更近一步，指定一个新的class专门用于渲染一个相机。我们把它叫做CamerRender，给它一个public Render方法，带有一个上下文 和相机参数。我们把这些个参数存到类的字段中，以便访问。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CameraRenerer
{
    ScriptableRenderContext context;
    Camera camera;
    public void Render(ScriptableRenderContext context, Camera camera)
    {
        this.context = context;
        this.camera = camera;
    }
}
```
让CustomRenderPipeline 创建一个实例，然后循环渲染所有的相机。
```c#
CameraRenderer renderer = new CameraRenderer();

protected override void Render (ScriptableRenderContext context, Camera[] cameras)
{
    foreach (Camera camera in cameras)
    {
        renderer.Render(context, camera);
    }
}
```
我们的相机渲染器大致等于URP的渲染器了。这个方法在将来让支持，每个相机不同的渲染方式变得简单。比如说，一个相机用于渲染第一人称视角，一个相机用于渲染3D小地图，或前向渲染延迟渲染。 但是现在，我们会让所有相机用同样的方式渲染。

## 2.2 绘制天空盒

CamerRenderer.Render的任务是渲染所有相机看见的几何体。为了结构清晰，编程习惯上是一个方法做一个事情，所以这里把绘制集合体放在单独的方法中DrawVisiableGeometry.

```c#
    public void Render (ScriptableRenderContext context, Camera camera) {
        this.context = context;
        this.camera = camera;

        DrawVisibleGeometry();
    }

    void DrawVisibleGeometry ()
    {
        context.DrawSkybox(camera);
    }
}
```
这样做之后还是不能让天空盒出现。那是因为我们上下文发去的命令被放在了缓冲区。为了执行它，我们还需要调用contet的Submit方法，提交队列中的任务。

```c#
public void Render(ScriptableRenderContext context, Camera camera)
{
    this.context = context;
    this.camera = camera;
    DrawVisibleGeometry();
    Submit();
}

void DrawVisibleGeometry()
{
    context.DrawSkybox(camera);
}

void Submit()
{
   context.Submit();
}
```
天空盒最终出现在游戏窗口和场景窗口中。你可以在frame debugger中谈到整个过程。它列出了 Camera.RenderSkybox 下又个一个Draw Mesh项，它代表真实的draw call。这和游戏窗口中的渲染相符，frame debugger不会报告其他窗口的绘制信息。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox-debugger.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox-debugger.png)

*天空盒绘制*

需要注意的是，相机的旋转值目前不能影响天空盒的渲染。我们把相机传递到了DrawSkybox中，但是那仅仅是用来决定是否天空盒需要渲染，它通过相机的clear flags控制。

要正确的渲染天空盒盒整个场景，我们还必须设置观察投影矩阵。这个变换矩阵包含相机的位置和旋转（观察矩阵），观察矩阵包含相机透视或者正交投影（投影矩阵）。它就是在Shader中的 unity_MatrixVP，在绘制几何体时shader用到的属性。你可以通过frame debugger ShaderProperties选中draw call查看这个矩阵。

在这个时候，unity_MatrixVP 矩阵都是一样的。我们要调用context SetupCameraProperties，来应用相机的属性到context。


```c#
public void Render(ScriptableRenderContext context, Camera camera)
{
    this.context = context;
    this.camera = camera;

    Setup();
    DrawVisibleGeometry();
    Submit();
}

void Setup()
{
    context.SetupCameraProperties(camera);
}

void DrawVisibleGeometry()
{
    context.DrawSkybox(camera);
}

void Submit()
{
   context.Submit();
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox-aligned.png)

*天空盒正确对齐*

## 2.3 命令缓冲

上下文渲染延迟的，直到我们提交时才会真正地渲染。在此之前，我们对上下文配置和添加命令，用于后续的执行。一些任务，比如说绘制天空盒，可以通过发布一个确定的方法来绘制，但是其他的一些命令需要间接地通过一个单独的命令缓冲完成。我们需要这样的一个缓冲来绘制场景中其他的几何体。

new CommandBuffer 实例即可获得一个缓冲。我们目前只需要一个缓冲，在CameraRender中new一个存在字段中。同时，你要给它一个名字，以便于在framer debugger中查看。

```c#
const string bufferName = "Render Camara"
CommandBuffer buffer = new CommandBuffer(){
    name = bufferName
};
```
我们可以用命令缓冲注入分析器采样，在profiler 和frame debugger都可以查看。在合适的位置调用BeginSample 和EndSample，在我们的实例中，我们在Setup和Submit中调用。
```c#
void Setup()
{
    buffer.ExcuteSample(buffName);
    context.SetupCameraProperties(camera);
}

void Submit()
{
    buffer.ExcuteSample(bufferName);
    context.Submit();
}
```
要执行缓冲，调用context的ExcuteCommandBuffer传人buffer。它会从缓冲区复制命令但不会清空缓冲区，我们需要明确的指明我们要重复利用它。因为执行和清理总是同时完成，因此添加一个方法同时执行很方便。
```c#
void Setup()
{
    buffer.ExcuteSample(buffName);
    ExecuteBuffer();
    context.SetupCameraProperties(camera);
}

void Submit()
{
    buffer.ExcuteSample(bufferName);
    ExecuteBuffer();
    context.Submit();
}

void ExecuteBuffer()
{
    context.ExecuteCommandBuffer(buffer);
    buffer.Clear();
}
```
Camera.RenderSkybox样本现在被嵌套在来Render Camera内部。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/render-camera-sample.png)

*渲染相机样本*

## 2.4 清理渲染目标
无论我们绘制什么，最终都会渲染到相机目标，默认是帧缓冲区，但是也可以是渲染纹理。在绘制时，由于早先绘制的东西还存在缓冲区里面，这会干扰我们正则渲染的图像。确保正确的渲染，我们需要清理渲染目标，以摆脱旧内容的干扰。通过调用buffer的ClearRenderTarget，可以做到。
```c#
void Setup()
{
    buffer.BeginSample(buffName);
    buffer.ClearRenderTarget(true,true, Color.clear);
    ExecuteBuffer();
    context.SetCameraProperties(camera);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-nested-sample.png)

*清理，嵌套在样本中*

帧调试器现在显示清除操作的 Draw GL 条目，该条目嵌套在渲染相机的附加级别中。这是因为，ClearRenderTarget将清除操作包裹在了带有名字的明明缓冲中了。我们可以在采样之前调用，去除不必要的嵌套。这会导致两个临近点Render Camera 合并。
```c#
void Setup()
{
    buffer.ClearRenderTarget(true, trure, Color.clear);
    buffer.BeginSample(buffName);
    //buffer.ClearRenderTarget(true, trure, Color.clear);
    ExecuteBuffer();
    context.SetCamerProperties(camera);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-one-sample.png)

*清理操作，没有嵌套*

Draw GL条目表示用Hidden/InternalClear着色器绘制一个全屏的四边形，它不是最游戏效率的清理方式。使用这个方法是因为我们在设置相机属性之前清理。如果我们交换两个操作的顺序，就很容易明白。

```c#
void Setup()
{
    context.SetupCameraProperties(camera);
    buffer.ClearRenderTarget(true, true, Color.clear);
    buffer.BeginSample(bufferName);
    ExecuteBuffer();
    //context.SetupCameraProperties(camera);
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-correct.png)

*正确的清理*

现在我们可以看到 Clear(color+Z+stencil), 它指明了颜色和深度缓冲都被清理了。Z表示深度缓冲，stencil数据表示同一个buffer的另一个部分。

## 2.5 剔除
我们现在看到了天空盒，但是看不见任何我们放在场景中的物体。我们将绘制所有相机看得见的的物体，而不是绘制所有物体。我们从场景中的渲染组建开始，然后通过相机的视锥体剔除这些物体。

搞清楚什么能被剔除，需要我们跟踪多个相机设置和矩阵，这些信息包涵在ScriptableCullingParameters 这个结构体里面。我们通过调用相机的TryGetCullingParameters的方法来获得此结构体，而不是自己设置值。这个方法返回是否参数能成功的返回，因为有可能因为相机设置而失败。这个函数需要我们提供一个out参数来获得返回值。在我们的代码中，用一个单独的函数来返回此值。(*在此，我省略了ref/out相关的篇幅, 这个太基础了，作者实在太细节了*)
```c#
bool Cull()
{
    if(camera.TryGetCullingParameters(out ScriptableCullingParameters p))
    {
        return true;
    }
    return false;
}
```
现在在我们编写的Render函数中调用Cull，并且如果返回失败时，中断后续执行。
```c#
public void Render(ScriptableRenderContext context, Camera camera)
{
    this.context = context;
    this.camera = camera;

    if(!Cull())
    {
        return;
    }

    Setup();
    DrawVisiableGeometry();
    Submit();
}
```
camera.TryGetCullingParameters只是返回剔除参数，真正的剔除操作要通过调用context的Cull方法完成，它会返回一个CullingResults的结构体。context的Cull函数要求传入剔除参数作为一个ref参数。(*ref/out部分是c#的基础，不翻译*)
```c#
CullingResults cullingResults;
...
public Cull()
{
    if(camera.TryGetParameters(out ScriptableCullingParameters p))
    {
        cullingResults = context.Cull(ref p);
        return true;
    }
    return false;
}
```
## 2.6 绘制几何图形
一旦我们知道什么东西可见，我们就可以继续渲染这些东西。调用context的DrawRenderers方法，它要求传入剔除的结果结构体(CullingResults),告诉它使用哪些渲染器。同时，还需提供绘图设置(DrawingSettings)和过滤设置(FilterSettings)作为ref参数, 它们都是结构体。在我们编写的DrawVisibleGeometry函数中，绘制天空盒之前调用。
```c#
void DrawVisibleGeometry()
{
    var drawSettings = new DrawSettings();
    var filterSettings = new FilterSettings();

    context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);

    context.DrawSkybox(camera);
}
```
我们还是没有看到任何物体，因为我们还需要指明使用哪种着色器通道。这种情况下，我们只支持无光照着色器(Unlit Shader)，我们要给SRPDefaultUnlit通道获取一个着色器标签ID，我们可以创建一个并缓存到静态字段中。
```c#
static ShaderTagId unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit");
```
把它传入到DrawSettings构造函数的第一个参数，第二个参数传入SortingSettings结构体。SortingSettings构造函数需要相机参数，来确定是基于正交或者距离排序。
```c#
void DrawVisibleGeometry()
{
    var sortingSettings = new SortingSettings(camera);
    var drawingSettings = new DrawingSettings(unlitShaderTagId, sortingSettings);
    ...
}
```
除此之外，我们还需要指明，允许哪个渲染队列。把RenderQueueRange.all传入FilteringSettings的构造函数，以包含所有的物体。

```c#
var filteringSettings = new FilteringSetings(RenderQueueRange.all);
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit-debugger.png)

*绘制无光照的几何图形*

只有可见的物体，并且使用unlit shader的，获得了绘制。在Frame Debugger中，所有的绘制都在RenderLoop Draw下罗列出来了。此时，半透明的物体绘制有点奇怪，先别在意，先看看物体的绘制顺序。在Frame Debugger中，你可以一步一步查看draw call。

[https://gfycat.com/@catlikecoding](https://gfycat.com/@catlikecoding)
*Frame Debugger一步一步绘制*

这个绘制顺序有些随意。我们可以通过修改SortingSetttings的criteria属性，强制使用一个特殊的绘制顺序。让我们试试SortingCriteria.CommonOpaque。
```c#
var sortingSettings = new SortingSettings(camera){
    criteria = SortingCritera.CommonOpaque
};
```

[https://gfycat.com/@catlikecoding](https://gfycat.com/@catlikecoding)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/sorting.png)

*通用的不透明排序*

现在物体或多或少有些从前向后(front-to-back)绘制，这是不透明物体理想的选择。如果一个物体在另一个物体背后，它被遮挡的部分偏移着色就被跳过，这可以提高渲染速度。常见的不透明排序还考虑其他一些标准，比如渲染队列和材质。

## 2.7 分开绘制不透明和透明的几何图形
Frame Debugger给我们展示了透明物体绘制了出来，但是天空盒绘制在了不透明物体上面。天空盒在不透明几何图形之后绘制，它被不透明物体遮挡的部分片元可跳过，但是它覆盖了透明的几何图形。产生这个问题的是因为，透明着色器不会写到深度缓冲区，它不会遮挡后面的物体，因为我们是可以透过透明物体看到后面的物体的。解决这个问题的办法是，先绘制不透明物体，在绘制天空盒，然后绘制透明物体。

我们可以在调用DrawRenders时，使用RenderQueueRange.opaque，消除透明对象。
```c#
var filteringSettings = new FilteringSettings(RenderQueueRange.opaque);
```
然后在绘制天空盒之后，再次调用DrawRenderers。但是这次传入RenderQueueRange.transparent，同时把排序标准设置成SortingCritera.CommonTransparent。这就颠倒了透明物体的绘制顺序。
```c#
context.DrawSkybox(camera);

sortingSettings.criteria = SortingCriteria.CommonTransparent;
drawingSettings.sortingSettings = sortingSettings;
filteringSettings.renderQueueRange = RenderQueueRange.transparent;

context.DrawRenderers(cullingResults, ref drawingSettings, ref filteringSettings);
```

![https://thumbs.gfycat.com/DisloyalRemarkableAvocet-mobile.mp4](https://thumbs.gfycat.com/DisloyalRemarkableAvocet-mobile.mp4)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/opaque-skybox-transparent.png)

*不透明，然后天空盒，在然后透明*

>**为什么说绘制顺序颠倒了?**<br>
由于透明对象不写深度缓冲区，因此，从前向后排序对他们来说，没有性能优势。但是当透明物体在视觉上处于其他物体后面时，只有【从后向前绘制】才能得到正确地图像混合。

不幸的是，【从后向前排序】不能保证得到正确的混合，因为排序争对每个物体，基于它的位置。相交的两个大的透明物体还是会产生不正确的结果。这个不正确的结果，有时候可以通过把大的几何图形切割成更小的部分来解决。

# 3 编辑器渲染
我们的RP正确地绘制了无光照物体，但是这里还可以做些事情来提升Unity编辑器的工作体验。

## 3.1 绘制旧版着色器
因为我们的管线只支持unlit着色器通道，使用其他通道将不会被渲染，使得透明不可见。虽然这是正确的，但是它却隐藏了一个事实：场景中有些物体使用了错误的着色器。因此，让我们以某种方式渲染它们，不过要分开处理。

如果有人从默认的Unity项目开始，之后在切换到我们的RP，他的场景中很可能有很多使用错误着色器的物体。要覆盖所有Unity默认的着色器，我们需要使用到这些着色器ID
- Alway
- ForwardBase
- PrepassBase
- Vertex
- VertexLMRGBM
- VertextLM
```c#
static ShaderTagId[] legacyShaderTagIds = {
    new ShaderTagId("Always"),
    new ShaderTagId("ForwardBase"),
    new ShaderTagId("PrepassBase"),
    new ShaderTagId("Vertex"),
    new ShaderTagId("VertexLMRGBM"),
    new ShaderTagId("VertexLM")
}
```
在绘制可见的几何图形之后，在一个单独的方法中绘制这些不支持的着色器，只绘制第一个通道。因为这些都是无效的通道，它们的结果无论如何都是错误的，所以我们不关心其他的设置。我们用默认的过滤（FilteringSettings.defaultValue）设置就好。
```c#
public void Render (ScriptableRenderContext context, Camera camera) {
        …
        Setup();
        DrawVisibleGeometry();
        DrawUnsupportedShaders();
        Submit();
    }
    …
    void DrawUnsupportedShaders () {
        var drawingSettings = new DrawingSettings(
            legacyShaderTagIds[0], new SortingSettings(camera)
        );
        var filteringSettings = FilteringSettings.defaultValue;
        context.DrawRenderers(
            cullingResults, ref drawingSettings, ref filteringSettings
        );
    }
```
我们可以通过调用绘图设置的SetShaderPassName来绘制多个通道，该函数需要一个绘制顺序和标签作为参数。我们把数组中的所有对象都
```c#
var drawingSettings = new DrawingSettings(
    legacyShaderTagIds[0], new SortingSettings(camera)
);
for (int i = 1; i < legacyShaderTagIds.Length; i++) {
    drawingSettings.SetShaderPassName(i, legacyShaderTagIds[i]);
}
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/standard-black.png)

*标准着色器渲染成黑色*

使用标准着色器的对象也渲染出来了，但是它们它们现在是纯黑色的，因为我们的RP没有设置着色器需要的属性。<br>
**实际上我在2022实践的时候，它们不是黑色**

## 3.2 错误材质
为了更清楚的指明物体使用了不支持的着色器，我们将要使用Unity默认的错误着色器。使用Unity默认错误着色器，构造一个新的材质，错误材质可以通过调用Shader.Find("Hidden/InternelErrorShader")获得。把这个材质缓存在静态字段中，因为我们不想要每帧都创建以。然后把它赋给绘图设置的overrideMaterail这个属性。
```c#
static Material errrorMaterial;
...
void DrawUnsupportedShaders()
{
    if(errorMaterial == null)
    {
        errorMaterial = new Material(Shader.Find("Hidden/InternalErrorShader"));
    }

    var drawingSettings = new DrawingSettings(legacyShaderTagIds[0], new SortingSettings(camera))
    {
        overrideMaterial = errorMaterial
    };
    ...
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/standard-magenta.png)

*使用洋红色的错误着色器渲染*

现在所有的无效的对象都看得见，并且错误显而易见。
## 3.3 部分类
绘制无效的物体在开发阶段非常有用，但是它不适用于发布版本的app。因此，让我们把这些只在编辑器可用的代码放到一个单独的部分类文件中。拷贝CamerRenderer重命名为CameraRenderer.Editor。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/two-assets.png)

*一个类，两个脚本文件*

然后在两个CamerRenderer的类前面添加partial修饰符，把原始的CamerRednerer文件中errorMaterial，legacyShaderTagIds 和DrawUnsupportedShaders这些东西统统删除。
```c#
public partial class CameraRenderer {...}
```
>关于什么是c#部分类，请自行查阅资料，非常简单，这里不翻译

在另一个部分类文件中(CameraRender.Editor), 只保留刚刚我们在上一个文件中删掉的那一个部分，如下
```c#
using UnityEngine;
using UnityEngine.Rendering;

partial class CameraRenderer {
    static ShaderTagId[] legacyShaderTagIds = {    … };
    static Material errorMaterial;
    void DrawUnsupportedShaders () { … }
}
```
编辑器部分只在编辑器环境下起作用，因此我们用 UNITY_EDITOR这个条件包起来。
```c#
partial class CameraRenderer {
#if UNITY_EDITOR
    static ShaderTagId[] legacyShaderTagIds = { … }
    };
    static Material errorMaterial;
    void DrawUnsupportedShaders () { … }
#endif
}
```
这个时候，无论如何都会编译失败，因为部分类的另一部分总是包含一个DrawUnsupportedShaders的调用，而这个函数现在只在编辑器可用。要解决这个问题，我们把这个函数也用partial修饰一下。我们可用在部分类的任意一方干这件事，因此让我们把它放到编辑器这部分。完整的方法申明必须用partial修饰。
```c#
    partial void DrawUnsupportedShaders ();
#if UNITY_EDITOR
    …
    partial void DrawUnsupportedShaders () { … }
#endif
```
现在构建编译成功了。编译器会剥离掉所有的partial修饰的不是完整定义的方法。

## 3.4 绘制小工具(Gizmos)
*Gizmo这个单词中文不好找贴切的词语，在下文中就不翻译了*
目前我们的渲染管线(RP)，在场景窗口和游戏窗口都没有绘制Gizmos，即便是它们都被启用的情况下。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/without-gizmos.png)

*没有Gizmo的场景*

我们可用调用UnityEditor.Handles.ShouldRenderGizmos来检查是否需要绘制Gizmos。如果需要，我们就调用context.DrawGizmos方法，把相机参数传入第一个参数，第二个参数需要指定Gizmo子集。Gizmo有个两个子集，分别用于图像特效之前和之后。由于我们此时还不支持图像特效，我们先将两者都调用。在只在编辑器生效的那个类中，新增一个DrawGizmos方法。
```c#
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;

partial class CameraRenderer {
    partial void DrawGizmos ();
    partial void DrawUnsupportedShaders ();
#if UNITY_EDITOR
    …
    partial void DrawGizmos () {
        if (Handles.ShouldRenderGizmos()) {
            context.DrawGizmos(camera, GizmoSubset.PreImageEffects);
            context.DrawGizmos(camera, GizmoSubset.PostImageEffects);
        }
    }
    partial void DrawUnsupportedShaders () { … }
#endif
}
```
Gizmo应该在其他所有东西绘制完成之后。
```c#
public void Render (ScriptableRenderContext context, Camera camera) {
    …

    Setup();
    DrawVisibleGeometry();
    DrawUnsupportedShaders();
    DrawGizmos();
    Submit();
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/with-gizmos.png)

*场景中的Gizmo*

# 3.5 绘制Unity UI
另一个需要我们注意的是Unity游戏内的用户界面。举个例子，通过GameObject/UI/Button创建一个简单的按钮，它们会显示在游戏窗体中，但是不会显示在场景窗口中。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-button.png)

*游戏窗口中的UI按钮*

>如果你不能创建UI按钮，你需要看看你是不是添加了Unity UI这个包。

Frame Debugger显示UI被单独渲染了，但是不在我们的渲染管线里面。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-debugger.png)

*frame debugger中的UI*

这是Canvas组件的渲染模式设置成Screen Space-Overlay, 这是默认的。修改成Screen Space - Camera 并且使用主相机作为Canvas的相机，这就会让它成为透明集合图形。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-camera-debugger.png)

*frame debugger中Screen-space-camera模式的UI*

UI在场景窗口中渲染时，总是使用世界空间模式，这就是为什么它看起来总是非常大。但是，当我们通过场景窗口编辑它时，它又不被绘制了。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/invisible-ui-scene.png)

*UI在场景窗口中不见了*

当我们在场景窗口中渲染时，需要明确的把UI添加到世界几何图形中，方法是调用ScriptableRenderContext.EmitWorldGeometryForSceneview, 传入一个相机作为参数。继续把这些放到编辑器类中，在相机类型为CameraTyoe.SceneView时渲染。
```c#
partial void PrepareForSceneWindow ();
#if UNITY_EDITOR
    …
    partial void PrepareForSceneWindow () {
        if (camera.cameraType == CameraType.SceneView) {
            ScriptableRenderContext.EmitWorldGeometryForSceneView(camera);
        }
    }
```
由于可能会在场景中天机几何图形，所以应该在剔除操作之前调用。
```c#
    PrepareForSceneWindow();
    if (!Cull()) {
        return;
    }
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/visible-ui-scene.png)

*UI在场景窗口中可见了*

# 4 多相机
一个场景中可以有多个活动的相机。因此，我们要确保它们一起工作。

## 4.1 两个相机
每个相机都有一个深度值，主相机默认为-1。它们按照深度递增渲染。要看这个，请复制一个主相机，重命名为Secondary Camera, 设置它的深度为0。给它一个新的tag，因为MainCamera这个tag只能被一个相机使用。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/two-cameras-sample-sample.png)

*相机都分组到了一个样本范围内*

现在场景被渲染了两次。结果图像仍然相同，因为渲染目标在两次渲染之间被清理了。frame debugger显示了这一点，但是因为相邻的两个样本使用了相同的名字，这就导致它们被合并到了一个的Render Camera范围里面了。

如果每个相机拥有自己的范围，会更清晰一点。为此，请添加一个只给编辑器调用的方法PrepareBuffer, 它把缓冲区的名字修改成相机的名字。
```c#
partial void PrepareBuffer();
#if UNITY_EDITOR
partial void PrepareBuffer()
{
    buffer.name = camera.name;
}
#endif
```
在准备场景窗口之前调用它。
```c#
PrepareBuffer();
PrepareForSceneWindow();
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/separate-samples.png)

*每个相机样本分离*

## 4.2 解决缓冲区名字修改的问题
尽管frame debugger现在每个相机独立显示样本层次结构，当我们进入游戏模式时，Unity控制台输出警告信息显示：BeginSample和EndSample次数必须匹配。因为我们样本和缓冲区使用了不同的名称，引擎就搞不清楚了。除此之外，每次访问相机的name属性还会导致内存分配，这是我们构建时不想要的。

要处理这些问题，我们添加一个SampleName的字符串属性。如果我们在编辑器中，我们就在PrepareBuffer中修改缓冲区的名字，否则就用字符常量作为Render Camera的别名。
```c#
#if UNITY_EDITOR
...
string SampeName {get;set;}
...
partial void PrepareBuffer()
{
    buffer.name = SampleName = camera.name;
}
#else
const string SampleName = bufferName;
#endif
```
在Setup和Submit函数中使用SampleName.
```c#
void Setup()
{
    context.SetupCameraProperties(camera);
    buffer.ClearRenderTarget(true, true, Color.clear);
    buffer.BeginSample(SampleName);
    ExecuteBuffer();
}
void Submit()
{
    buffer.EndSample(SampleName);
    ExecuteBuffer();
    context.Submit();
}
```
我们可以打开profiler(Window/Analysis/Profile)并在编辑器运行，查看此二者不同。切换到层次结构模式，以GC Alloc列排序。你会看到一个GC Alloc条目，总共分配了100字节，它是由检索名字引起的。在往下，你会看到显示样本：Main Camera和Secondary Camera.

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/profiler-cg-alloc.png)

*分析器显示单独的样本和100B内存分配*

接下来，构建一个Develop Build版本，并且启用自动连接调试器。运行构建并且确保分析器连接并且录制。这个情况下，没有100字节的内存分配，同时相机样本又变成了个Render Camera样本。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/profiler-build.png)

*分析器构建*

通过包装编辑器名称在分析器中，我们可以清晰的知道，它只在编辑器中分配内存而是在构建版本中。在这个例子中，我们需要调用Profiler.BeginSample和Profiler.EndSample，只有BeginSample需要传入名称。
```c#
using UnityEditor;
using UnityEngine;
using UnityEngine.Profiling;
using UnityEngine.Rendering;

partial class CameraRenderer {
    …
#if UNITY_EDITOR
    …
    partial void PrepareBuffer () {
        Profiler.BeginSample("Editor Only");
        buffer.name = SampleName = camera.name;
        Profiler.EndSample();
    }
#else
    string SampleName => bufferName;
#endif
}
```
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/editor-only-allocations.png)

*Editor-only分配很明显*

## 4.3 层级
通过调整相机的Culling Mask，可以配置为只观察确定层级的物体。要看到这个行为，让我们把使用standard shader到Ignore Raycast层。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/ignore-raycast-layer.png)

*层级切换到Ignore Raycast*

在Main Camera中通过Culling Mask排除这个层。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/culling-ignore-raycast.png)

*剔除Ignore Raycast层*

同时让Secondary Camera只看这一层。
![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/only-ignore-raycast.png)

*剔除除Ignore Raycast之外的所有层*

因为Secondary Camera最后渲染，最终我们只看到无效的物体。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/only-ignore-raycast-game.png)

*只有Ignore Raycast层在游戏窗口中可见*

## 4.4 清除标记
我们可以调整第二个相机的清除标记，来合并两个相机的渲染结果。清楚标记被定义在CameraClearFlags这个枚举中，我们可以通过相机的clearFlags获得。在clear操作之前访问。
```c#
void Setup()
{
    context.SetCameraProperties(camera);
    CameraClearFlags flags = camera.clearFlags;
    buffer.ClearRenderTarget(true, true, Color.clear);
    buffer.BeginSample(SampleName);
    ExecuteBuffer();
}
```
CameraClearFlags定义了四个值。从1到4分别为，Skybox, Color, Depth, Noting。这些并不是真的独立标记值，而是表示一个清除减少量。
```c#
buffer.ClearRenderTarget(
    flags <= CameraClearFlags.Depth, true, Color,clear
);
```
当清除标记设置为Color时，我们只需要清除颜色缓冲区，因为在这个例子中的天空盒最终会替换之前的颜色。
```c#
buffer.ClearRenderTarget(
    flags <= CameraClearFlags.Depth,
    flags == CameraClearFlags.Color,
    Color.clear
);
```
如果我们要清除为纯色，我们必须用相机的背景色。因为我们使用的线性颜色空间，我们还需要把颜色转换到线性空间，所以最后我们需要这样做camera.backgroundColor.linear。
```c#
buffer.ClearRenderTarget(
    flags <= CameraClearFlags.Depth,
    flags == CameraClearFlags.Color,
    flags == CameraClearFlags.Color ?
             camera.backgroundColor.linear : Color.clear
);
```
因为Main Camera是第一个渲染的，它的Clear Flags应当设置成Skybox或则Color。~~当启用frame debugger时，我们总是从一个清理缓冲区开始，但这个通常不能保证。~~

Secondary Camera的清除标记决定如何合并渲染。在这个例子中，之前的渲染结果中的天空盒或颜色都被替换了。当第二个相机只清除深度时，相机渲染为普通模式，它不会绘制天空盒，因此上一次的结果最终显示成了背景。当不清除任何东西时，深度缓冲被保留，所以无光照物体最后遮挡了无效的物体，仿佛它们使用同一个相机绘制一样。然而，上一个相机绘制的透明物体没有深度信息，所以它们会绘制覆盖，看起来就像天空盒在之前绘制一样。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-color.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-depth.png)

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-nothing.png)

*清除颜色，清除深度，不清除*

通过调整相机的Viewport Rect(视口矩形)，还可以减少渲染的区域到渲染区域的一部分。其他部分不受影响。在这种情况下，清理操作用Hidden/InternalClear着色器完成。模板缓冲被使用来限制渲染到视口区域。

![](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/reduced-viewport.png)

*减少secondary camera的视口，清除颜色*

注意，多相机渲染意味着每帧剔除，设置，排序，等等，都会被多次执行。使用一个相机多角度拍摄对比多相机，肯定是最性能最优的方法。

下个教程是Draw Calls.
