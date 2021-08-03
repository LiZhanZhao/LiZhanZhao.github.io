---
layout:     post
title:      "Unity URP 流程分析"
subtitle:   ""
date:       2021-8-3 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---


## 流程

## 入口

- 如果Unity需要进行 Scriptable Render Pipeline 的话，需要在 Edit -> Project Setting 设置 Scriptable Render Pipeline Setting, 
这个Setting 需要使用 Render Pipeline Asset

## 创建 URP 的 Render Pipeline Asset

- 去到某一个目录下，右键 -> Create -> Rendering -> Universal Render Pipeline -> Pileline Asset (Forward Renderer)

#### 代码

- 菜单的代码文件在 <br>
*Packages\src\com.unity.render-pipelines.universal@7.3.1\Runtime\Data\UniversalRenderPipelineAsset.cs*
<br><br>

- 菜单入口, 执行这个菜单，最后会生成两个Asset，一个是  UniversalRenderPipelineAsset, 一个是 ForwardRendererData，然后关联这两个Asset，UniversalRenderPipelineAsset.m_RendererDataList[0] = ForwardRendererData Asset
```C#
[MenuItem("Assets/Create/Rendering/Universal Render Pipeline/Pipeline Asset (Forward Renderer)", priority = CoreUtils.assetCreateMenuPriority1)]
static void CreateUniversalPipeline()
{
    ProjectWindowUtil.StartNameEditingIfProjectWindowExists(0, CreateInstance<CreateUniversalPipelineAsset>(),
        "UniversalRenderPipelineAsset.asset", null, null);
}
```


## UniversalRenderPipelineAsset

- UniversalRenderPipelineAsset 的代码在 *Packages\src\com.unity.render-pipelines.universal@7.3.1\Runtime\Data\UniversalRenderPipelineAsset.cs* <br>

- 按照 Unity的规则，UniversalRenderPipelineAsset肯定会有一个 CreatePipeline 方案来创建渲染管线


#### CreatePipeline

- 实例化一个 UniversalRenderPipeline 出来，把 同时 把 UniversalRenderPipelineAsset 和 UniversalRenderPipeline 关联起来
- CreatePipeline 还会调用 CreateRenderers，去初始化 UniversalRenderPipelineAsset 的 m_Renderers[i]

```C#
protected override RenderPipeline CreatePipeline()
{
    if (m_RendererDataList == null)
        m_RendererDataList = new ScriptableRendererData[1];

    // If no data we can't create pipeline instance
    if (m_RendererDataList[0] == null)
    {
        // If previous version and current version are miss-matched then we are waiting for the upgrader to kick in
        if(k_AssetPreviousVersion != k_AssetVersion)
            return null;

        Debug.LogError(
            $"Default Renderer is missing, make sure there is a Renderer assigned as the default on the current Universal RP asset:{UniversalRenderPipeline.asset.name}",
            this);
        return null;
    }

    //初始化 UniversalRenderPipelineAsset 的 m_Renderers[i]
    CreateRenderers();
    return new UniversalRenderPipeline(this);
}

```


####  CreateRenderers

- CreateRenderers 主要就是 用来初始化 UniversalRenderPipelineAsset 的 m_Renderers[i]
- m_Renderers[i] 保存的是 ForwardRenderer
- ForwardRenderer 实例还会关联 ForwardRendererData


```C#
void CreateRenderers()
{
    m_Renderers = new ScriptableRenderer[m_RendererDataList.Length];
    for (int i = 0; i < m_RendererDataList.Length; ++i)
    {
        // 这个是虚拟函数，要看继承
        if (m_RendererDataList[i] != null)
            m_Renderers[i] = m_RendererDataList[i].InternalCreateRenderer();
    }
}

internal ScriptableRenderer InternalCreateRenderer()
{
    isInvalidated = false;
    return Create();
}

// 这个是ForwardRenderData.cs里面
protected override ScriptableRenderer Create()
{
#if UNITY_EDITOR
    if (!Application.isPlaying)
    {
        ResourceReloader.TryReloadAllNullIn(this, UniversalRenderPipelineAsset.packagePath);
        ResourceReloader.TryReloadAllNullIn(postProcessData, UniversalRenderPipelineAsset.packagePath);
    }
#endif
    return new ForwardRenderer(this);
}

```




## UniversalRenderPipeline

- UniversalRenderPipelineAsset 的代码在 *Packages\src\com.unity.render-pipelines.universal@7.3.1\Runtime\UniversalRenderPipeline.cs*
<br><br>

- 由于 UniversalRenderPipelineAsset 的 CreatePipeline 实例化了一个 UniversalRenderPipeline，所以会调用 UniversalRenderPipeline 的构造函数, UniversalRenderPipeline 的构造函数 就做一些初始化的工作


#### Render(ScriptableRenderContext context, Camera[] cameras);

- 按照Unity的规则，UniversalRenderPipeline 继承 RenderPipeline, 需要提供 Render 函数，Render 函数主要负责整渲染一帧渲染管线的渲染流程, Defines custom rendering for this RenderPipeline.


#### Dispose(bool disposing);
- 按照Unity的规则，UniversalRenderPipeline 继承 RenderPipeline, 需要提供 Dispose 函数，渲染一帧结束之后进行清理工作



## UniversalRenderPipeline.Render

```C#
protected override void Render(ScriptableRenderContext renderContext, Camera[] cameras)
{
    BeginFrameRendering(renderContext, cameras);

    GraphicsSettings.lightsUseLinearIntensity = (QualitySettings.activeColorSpace == ColorSpace.Linear);
    GraphicsSettings.useScriptableRenderPipelineBatching = asset.useSRPBatcher;
    SetupPerFrameShaderConstants();
#if ENABLE_VR && ENABLE_XR_MODULE
    SetupXRStates();
    if(xrSkipRender)
        return;
#endif

    SortCameras(cameras);
    for (int i = 0; i < cameras.Length; ++i)
    {
        var camera = cameras[i];
        if (IsGameCamera(camera))
        {
            RenderCameraStack(renderContext, camera);
        }
        else
        {
           ...
        }
    }

    EndFrameRendering(renderContext, cameras);
}
```
- 先根据Camera.depth 进行排序 cameras
- 再进行 RenderCameraStack


<br><br>


```C#
static void RenderCameraStack(ScriptableRenderContext context, Camera baseCamera)
{
    baseCamera.TryGetComponent<UniversalAdditionalCameraData>(out var baseCameraAdditionalData);

    // Overlay cameras will be rendered stacked while rendering base cameras
    if (baseCameraAdditionalData != null && baseCameraAdditionalData.renderType == CameraRenderType.Overlay)
        return;

    // renderer contains a stack if it has additional data and the renderer supports stacking
    var renderer = baseCameraAdditionalData?.scriptableRenderer;
    bool supportsCameraStacking = renderer != null && renderer.supportedRenderingFeatures.cameraStacking;
    List<Camera> cameraStack = (supportsCameraStacking) ? baseCameraAdditionalData?.cameraStack : null;

    bool anyPostProcessingEnabled = baseCameraAdditionalData != null && baseCameraAdditionalData.renderPostProcessing;
    anyPostProcessingEnabled &= SystemInfo.graphicsDeviceType != GraphicsDeviceType.OpenGLES2;

    // We need to know the last active camera in the stack to be able to resolve
    // rendering to screen when rendering it. The last camera in the stack is not
    // necessarily the last active one as it users might disable it.
    ...


    bool isStackedRendering = lastActiveOverlayCameraIndex != -1;

    BeginCameraRendering(context, baseCamera);
#if VISUAL_EFFECT_GRAPH_0_0_1_OR_NEWER
    //It should be called before culling to prepare material. When there isn't any VisualEffect component, this method has no effect.
    VFX.VFXManager.PrepareCamera(baseCamera);
#endif
    UpdateVolumeFramework(baseCamera, baseCameraAdditionalData);
    InitializeCameraData(baseCamera, baseCameraAdditionalData, out var baseCameraData);
    RenderSingleCamera(context, baseCameraData, !isStackedRendering, anyPostProcessingEnabled);
    EndCameraRendering(context, baseCamera);

    if (!isStackedRendering)
        return;

    for (int i = 0; i < cameraStack.Count; ++i)
    {
        var currCamera = cameraStack[i];

        if (!currCamera.isActiveAndEnabled)
            continue;

        currCamera.TryGetComponent<UniversalAdditionalCameraData>(out var currCameraData);
        // Camera is overlay and enabled
        if (currCameraData != null)
        {
            // Copy base settings from base camera data and initialize initialize remaining specific settings for this camera type.
            CameraData overlayCameraData = baseCameraData;
            bool lastCamera = i == lastActiveOverlayCameraIndex;

            BeginCameraRendering(context, currCamera);
#if VISUAL_EFFECT_GRAPH_0_0_1_OR_NEWER
            //It should be called before culling to prepare material. When there isn't any VisualEffect component, this method has no effect.
            VFX.VFXManager.PrepareCamera(currCamera);
#endif
            UpdateVolumeFramework(currCamera, currCameraData);
            InitializeAdditionalCameraData(currCamera, currCameraData, ref overlayCameraData);
            RenderSingleCamera(context, overlayCameraData, lastCamera, anyPostProcessingEnabled);
            EndCameraRendering(context, currCamera);
        }
    }
}
```
- 先渲染当前的 Camera, 然后再遍历 cameraStack List，再进行渲染, 每一个camera 都调用 RenderSingleCamera



<br><br>



```C#
/// <summary>
/// Renders a single camera. This method will do culling, setup and execution of the renderer.
/// </summary>
/// <param name="context">Render context used to record commands during execution.</param>
/// <param name="cameraData">Camera rendering data. This might contain data inherited from a base camera.</param>
/// <param name="requiresBlitToBackbuffer">True if this is the last camera in the stack rendering, false otherwise.</param>
/// <param name="anyPostProcessingEnabled">True if at least one camera has post-processing enabled in the stack, false otherwise.</param>
static void RenderSingleCamera(ScriptableRenderContext context, CameraData cameraData, bool requiresBlitToBackbuffer, bool anyPostProcessingEnabled)
{
    Camera camera = cameraData.camera;
    var renderer = cameraData.renderer;
    if (renderer == null)
    {
        Debug.LogWarning(string.Format("Trying to render {0} with an invalid renderer. Camera rendering will be skipped.", camera.name));
        return;
    }

    if (!camera.TryGetCullingParameters(IsStereoEnabled(camera), out var cullingParameters))
        return;

    SetupPerCameraShaderConstants(cameraData);

    ProfilingSampler sampler = (asset.debugLevel >= PipelineDebugLevel.Profiling) ? new ProfilingSampler(camera.name): _CameraProfilingSampler;
    CommandBuffer cmd = CommandBufferPool.Get(sampler.name);
    using (new ProfilingScope(cmd, sampler))
    {
        renderer.Clear(cameraData.renderType);
        renderer.SetupCullingParameters(ref cullingParameters, ref cameraData);

        context.ExecuteCommandBuffer(cmd);
        cmd.Clear();

#if UNITY_EDITOR
        // Emit scene view UI
        if (cameraData.isSceneViewCamera)
        {
            ScriptableRenderContext.EmitWorldGeometryForSceneView(camera);
        }
#endif

        var cullResults = context.Cull(ref cullingParameters);
        InitializeRenderingData(asset, ref cameraData, ref cullResults, requiresBlitToBackbuffer, anyPostProcessingEnabled, out var renderingData);

        renderer.Setup(context, ref renderingData);
        renderer.Execute(context, ref renderingData);
    }

    context.ExecuteCommandBuffer(cmd);
    CommandBufferPool.Release(cmd);
    context.Submit();
}
```
- 这个函数就需要留意 renderer 的方法，上面就可以知道，renderer 其实就是 ForwardRenderer.cs，ForwardRenderer.cs 继承 ScriptableRenderer.cs，所有的渲染的相关的东西，就放在 这两个脚本里面了
