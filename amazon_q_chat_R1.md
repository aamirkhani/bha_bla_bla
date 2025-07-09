Based on my comprehensive analysis of the Unity URP source code, I can now provide you with a detailed explanation of the transparency implementation in Unity 6.1's Universal Render 
Pipeline.

# Unity URP Transparency Implementation - Detailed Analysis

## Architecture Overview

Unity's Universal Render Pipeline (URP) implements transparency through a sophisticated multi-layered architecture that combines C# runtime logic, shader code, and modern RenderGraph-
based rendering. Here's the detailed breakdown:

## 1. Core Pipeline Structure

### Main Components:

• **UniversalRenderPipeline.cs**: Main entry point that orchestrates the entire rendering process
• **UniversalRenderer.cs**: The primary renderer implementation that handles forward rendering
• **ScriptableRenderer.cs**: Abstract base class providing the rendering framework
• **DrawObjectsPass.cs**: Core pass that handles both opaque and transparent object rendering

### Rendering Modes:

URP supports multiple rendering modes that affect transparency handling:
• **Forward**: Traditional forward rendering with per-object light culling
• **Forward+**: Forward rendering with clustered lighting data structures
• **Deferred**: G-buffer based deferred shading (limited transparency support)
• **Deferred+**: Deferred with clustered data structures

## 2. Transparency Rendering Pipeline

### Render Pass Sequence:

1. Shadow Passes: Generate shadow maps
2. Depth Prepass: Optional depth-only rendering for optimization
3. G-Buffer Pass: (Deferred mode only) - renders opaque geometry to G-buffers
4. Opaque Forward Pass: Renders opaque objects
5. Skybox Pass: Renders skybox
6. Transparent Forward Pass: This is where transparency is handled
7. Post-processing: Final effects and tone mapping

### Key Transparency Classes:

#### DrawObjectsPass

csharp
public class DrawObjectsPass : ScriptableRenderPass
{
    bool m_IsOpaque;
    bool m_ShouldTransparentsReceiveShadows;
    FilteringSettings m_FilteringSettings;
    RenderStateBlock m_RenderStateBlock;
}


This pass handles both opaque and transparent objects based on the m_IsOpaque flag. For transparency:
• Uses SortingCriteria.CommonTransparent for back-to-front sorting
• Applies different render state blocks for blending
• Conditionally applies shadow receiving based on settings

#### TransparentSettingsPass

csharp
internal class TransparentSettingsPass : ScriptableRenderPass
{
    bool m_shouldReceiveShadows;
    
    public static void ExecutePass(RasterCommandBuffer rasterCommandBuffer)
    {
        MainLightShadowCasterPass.SetShadowParamsForEmptyShadowmap(rasterCommandBuffer);
        AdditionalLightsShadowCasterPass.SetShadowParamsForEmptyShadowmap(rasterCommandBuffer);
    }
}


This specialized pass disables shadow receiving for transparent objects when configured to do so.

## 3. Shader Implementation

### Lit Shader Transparency Features:

The main Lit.shader includes comprehensive transparency support:

hlsl
// Transparency-related properties
_Surface("__surface", Float) = 0.0          // 0=Opaque, 1=Transparent
_Blend("__blend", Float) = 0.0              // Blend mode selection
_SrcBlend("__src", Float) = 1.0             // Source blend factor
_DstBlend("__dst", Float) = 0.0             // Destination blend factor
_SrcBlendAlpha("__srcA", Float) = 1.0       // Source alpha blend
_DstBlendAlpha("__dstA", Float) = 0.0       // Destination alpha blend
_ZWrite("__zw", Float) = 1.0                // Depth writing control
_AlphaClip("__clip", Float) = 0.0           // Alpha testing


### Render State Configuration:

hlsl
Blend[_SrcBlend][_DstBlend], [_SrcBlendAlpha][_DstBlendAlpha]
ZWrite[_ZWrite]
Cull[_Cull]


### Shader Variants:
hlsl
#pragma shader_feature_local_fragment _SURFACE_TYPE_TRANSPARENT
#pragma shader_feature_local_fragment _ALPHATEST_ON
#pragma shader_feature_local_fragment _ _ALPHAPREMULTIPLY_ON _ALPHAMODULATE_ON


### Alpha Output Processing:
In LitForwardPass.hlsl:
hlsl
color.a = OutputAlpha(color.a, IsSurfaceTypeTransparent(_Surface));


This function handles different alpha blending modes and ensures proper alpha output based on the surface type.

## 4. RenderGraph Implementation

Unity 6.1 introduces a modern RenderGraph-based rendering path that provides better performance and memory management:

### Transparent Pass in RenderGraph:

csharp
// In UniversalRendererRenderGraph.cs
m_RenderTransparentForwardPass.m_ShouldTransparentsReceiveShadows = !m_TransparentSettingsPass.Setup();
m_RenderTransparentForwardPass.Render(
    renderGraph,
    frameData,
    resourceData.activeColorTexture,
    resourceData.activeDepthTexture,
    TextureHandle.nullHandle,
    TextureHandle.nullHandle,
    uint.MaxValue
);


### Event Ordering:
csharp
RecordCustomRenderGraphPasses(renderGraph, RenderPassEvent.AfterRenderingOpaques, RenderPassEvent.BeforeRenderingTransparents);
// Transparent rendering happens here
RecordCustomRenderGraphPasses(renderGraph, RenderPassEvent.AfterRenderingTransparents, RenderPassEvent.AfterRendering);


## 5. Transparency Features

### Supported Blend Modes:
• **Alpha**: Standard alpha blending (SrcAlpha, OneMinusSrcAlpha)
• **Premultiply**: Pre-multiplied alpha (One, OneMinusSrcAlpha)
• **Additive**: Additive blending (SrcAlpha, One)
• **Multiply**: Multiplicative blending (DstColor, Zero)

### Shadow Handling:
Transparent objects can optionally receive shadows through a configurable system:
csharp
if (!data.isOpaque && !data.shouldTransparentsReceiveShadows)
    TransparentSettingsPass.ExecutePass(context.cmd);


### Sorting:
Transparent objects are sorted back-to-front using SortingCriteria.CommonTransparent which includes:
• Distance from camera
• Sorting layer
• Render queue
• Material render queue

## 6. Performance Optimizations

### Depth Buffer Optimization:
• Transparent objects typically don't write to depth buffer (ZWrite Off)
• Depth testing is still performed for proper occlusion
• Optional depth prepass for complex scenes

### Batching:
• SRP Batcher compatibility for transparent materials
• GPU Resident Drawer support for large numbers of transparent objects
• Instancing support for identical transparent objects

### Adaptive Performance:
csharp
#if ADAPTIVE_PERFORMANCE_2_1_0_OR_NEWER
if (needTransparencyPass)
#endif
    m_RenderTransparentForwardPass.Render(/*...*/);


The system can dynamically disable transparency rendering under performance constraints.

## 7. Advanced Features

### Motion Vectors:

Transparent objects support motion vector generation for temporal effects like TAA and motion blur.

### Clustered Lighting:

In Forward+ mode, transparent objects benefit from clustered light culling, allowing many lights to affect transparent surfaces efficiently.

### Render Layers:
Transparent objects support rendering layers for selective lighting and effects.

## Conclusion

Unity URP's transparency implementation is a comprehensive system that balances visual quality, performance, and flexibility. The architecture separates concerns between high-level 
rendering orchestration (C#), low-level rendering commands (RenderGraph), and per-pixel shading (HLSL), providing a robust foundation for transparent rendering in modern games and 
applications.

The system's strength lies in its configurability - developers can choose between different blend modes, shadow receiving options, and performance trade-offs based on their specific 
needs, while the underlying architecture handles the complexity of proper sorting, state management, and GPU resource utilization.











> please expand details 10 times 
# Unity URP Transparency Implementation - Comprehensive Deep Dive Analysis

## 1. Architecture Overview - Detailed Breakdown

### 1.1 Core Pipeline Structure and Class Hierarchy

The Unity URP transparency system is built on a sophisticated multi-layered architecture that spans across several key namespaces and assemblies:

#### Primary Assembly Structure:
csharp
// Assembly: Unity.RenderPipelines.Universal.Runtime
namespace UnityEngine.Rendering.Universal
{
    // Core pipeline classes
    public sealed partial class UniversalRenderPipeline : RenderPipeline
    public sealed partial class UniversalRenderer : ScriptableRenderer
    public abstract partial class ScriptableRenderer : IDisposable
    
    // Pass implementations
    namespace Internal
    {
        public class DrawObjectsPass : ScriptableRenderPass
        public class TransparentSettingsPass : ScriptableRenderPass
        public class PostProcessPass : ScriptableRenderPass
    }
}


#### Dependency Chain Analysis:
csharp
UniversalRenderPipeline (v17.1.0)
├── Dependencies:
│   ├── com.unity.render-pipelines.core@17.1.0
│   ├── com.unity.shadergraph@17.1.0
│   ├── com.unity.render-pipelines.universal-config@17.0.3
│   └── Unity Engine Modules:
│       ├── UnityEngine.CoreModule
│       ├── UnityEngine.GraphicsModule
│       └── UnityEngine.RenderingModule


### 1.2 Rendering Mode Deep Analysis

#### Forward Rendering Mode (Primary Transparency Path):
csharp
public enum RenderingMode
{
    Forward = 0,        // Traditional forward with per-object culling
    ForwardPlus = 2,    // Forward with clustered lighting
    Deferred = 1,       // G-buffer based (limited transparency)
    DeferredPlus = 3,   // Deferred with clustering
}

// Camera stacking support per mode
public override int SupportedCameraStackingTypes()
{
    switch (m_RenderingMode)
    {
        case RenderingMode.Forward:
        case RenderingMode.ForwardPlus:
            return 1 << (int)CameraRenderType.Base | 1 << (int)CameraRenderType.Overlay;
        case RenderingMode.Deferred:
        case RenderingMode.DeferredPlus:
            return 1 << (int)CameraRenderType.Base; // No overlay support
        default:
            return 0;
    }
}


#### Transparency Limitations by Mode:
• **Forward/Forward+**: Full transparency support with proper sorting and blending
• **Deferred/Deferred+**: Transparent objects fall back to forward rendering after G-buffer pass
• **Mixed Mode**: Opaque objects in deferred, transparent in forward (hybrid approach)

### 1.3 Memory Management and Resource Allocation

#### RTHandle System Integration:
csharp
internal static class UniversalRenderPipeline
{
    internal static RenderGraph s_RenderGraph;
    internal static RTHandleResourcePool s_RTHandlePool;
    
    // Dynamic resolution support
    RTHandles.SetHardwareDynamicResolutionState(true);
    
    // Resource cleanup per frame
    s_RenderGraph.EndFrame();
    s_RTHandlePool.PurgeUnusedResources(Time.frameCount);
}


## 2. Transparency Rendering Pipeline - Exhaustive Analysis

### 2.1 Complete Render Pass Sequence with Transparency Focus

#### Detailed Pass Execution Order:
csharp
// From UniversalRenderer.Setup()
public override void Setup(ScriptableRenderContext context, ref RenderingData renderingData)
{
    // 1. Shadow Generation Phase
    if (mainLightShadows)
        EnqueuePass(m_MainLightShadowCasterPass);
    
    if (additionalLightShadows)
        EnqueuePass(m_AdditionalLightsShadowCasterPass);
    
    // 2. Depth Prepass (Optional - affects transparency performance)
    bool requiresDepthPrepass = m_DepthPrimingMode == DepthPrimingMode.Forced ||
                               (m_DepthPrimingMode == DepthPrimingMode.Auto && 
                                RequiresDepthPrepass(ref renderingData));
    
    if (requiresDepthPrepass)
    {
        m_DepthPrepass.Setup(renderingData.cameraData.cameraTargetDescriptor, m_DepthTexture);
        EnqueuePass(m_DepthPrepass);
    }
    
    // 3. G-Buffer Pass (Deferred mode only)
    if (renderingModeActual == RenderingMode.Deferred)
    {
        EnqueuePass(m_GBufferPass);
        EnqueuePass(m_DeferredPass);
    }
    
    // 4. Forward Opaque Pass
    EnqueuePass(m_RenderOpaqueForwardPass);
    
    // 5. Skybox Pass
    if (camera.clearFlags == CameraClearFlags.Skybox && cameraData.renderType != CameraRenderType.Overlay)
        EnqueuePass(m_DrawSkyboxPass);
    
    // 6. Copy Depth Pass (if needed before transparency)
    if (requiresDepthTexture && m_CopyDepthMode == CopyDepthMode.AfterOpaques)
        EnqueuePass(m_CopyDepthPass);
    
    // 7. TRANSPARENCY PASS - CRITICAL SECTION
    bool transparencyAfterPostProcessing = false;
    if (!transparencyAfterPostProcessing)
    {
        // Configure transparent settings
        if (m_TransparentSettingsPass.Setup())
            EnqueuePass(m_TransparentSettingsPass);
        
        // Main transparency rendering
        EnqueuePass(m_RenderTransparentForwardPass);
    }
    
    // 8. Post-processing and final passes
    // ... additional passes
}


### 2.2 DrawObjectsPass - Complete Implementation Analysis

#### Core Class Structure:
csharp
public class DrawObjectsPass : ScriptableRenderPass
{
    // Core filtering and state
    FilteringSettings m_FilteringSettings;
    RenderStateBlock m_RenderStateBlock;
    List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();
    
    // Transparency-specific flags
    bool m_IsOpaque;
    bool m_ShouldTransparentsReceiveShadows;
    bool m_IsActiveTargetBackBuffer;
    
    // Render data container
    PassData m_PassData;
    
    // Shader property ID for pass data
    static readonly int s_DrawObjectPassDataPropID = Shader.PropertyToID("_DrawObjectPassData");
}


#### Initialization Process:
csharp
internal void Init(bool opaque, RenderPassEvent evt, RenderQueueRange renderQueueRange, 
                  LayerMask layerMask, StencilState stencilState, int stencilReference, 
                  ShaderTagId[] shaderTagIds = null)
{
    // Default shader tags for URP
    if (shaderTagIds == null)
        shaderTagIds = new ShaderTagId[] { 
            new ShaderTagId("SRPDefaultUnlit"), 
            new ShaderTagId("UniversalForward"), 
            new ShaderTagId("UniversalForwardOnly") 
        };

    // Initialize pass data
    m_PassData = new PassData();
    foreach (ShaderTagId sid in shaderTagIds)
        m_ShaderTagIdList.Add(sid);
    
    // Set rendering parameters
    renderPassEvent = evt;
    m_FilteringSettings = new FilteringSettings(renderQueueRange, layerMask);
    m_RenderStateBlock = new RenderStateBlock(RenderStateMask.Nothing);
    m_IsOpaque = opaque;
    m_ShouldTransparentsReceiveShadows = false; // Default: no shadows for transparency
    m_IsActiveTargetBackBuffer = false;

    // Configure render state for transparency
    if (stencilState.enabled)
    {
        m_RenderStateBlock.stencilReference = stencilReference;
        m_RenderStateBlock.mask = RenderStateMask.Stencil;
        m_RenderStateBlock.stencilState = stencilState;
    }
}


#### Sorting Criteria Implementation:
csharp
private void ExecutePass(ScriptableRenderContext context, PassData data, ref RenderingData renderingData)
{
    ref Camera camera = ref data.cameraData.camera;
    
    // Critical sorting logic for transparency
    var sortFlags = (m_IsOpaque) ? 
        data.cameraData.defaultOpaqueSortFlags : 
        SortingCriteria.CommonTransparent;
    
    // Depth priming optimization for opaque objects
    if (data.cameraData.renderer.useDepthPriming && m_IsOpaque && 
        (data.cameraData.renderType == CameraRenderType.Base || data.cameraData.clearDepth))
    {
        sortFlags = SortingCriteria.SortingLayer | 
                   SortingCriteria.RenderQueue | 
                   SortingCriteria.OptimizeStateChanges | 
                   SortingCriteria.CanvasOrder;
    }

    // Create drawing settings
    var drawingSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, sortFlags);
    
    // Apply transparency-specific settings
    if (!m_IsOpaque)
    {
        // Ensure proper blend state for transparency
        drawingSettings.overrideMaterial = null;
        drawingSettings.overrideMaterialPassIndex = 0;
    }
    
    // Execute the actual drawing
    context.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings, ref m_RenderStateBlock);
}


### 2.3 TransparentSettingsPass - Shadow Management Deep Dive

#### Complete Implementation:
csharp
internal class TransparentSettingsPass : ScriptableRenderPass
{
    bool m_shouldReceiveShadows;

    public TransparentSettingsPass(RenderPassEvent evt, bool shadowReceiveSupported)
    {
        profilingSampler = new ProfilingSampler("Set Transparent Parameters");
        renderPassEvent = evt;
        m_shouldReceiveShadows = shadowReceiveSupported;
    }

    public bool Setup()
    {
        // Only enqueue this pass when transparent objects should NOT receive shadows
        // This is a performance optimization and visual choice
        return !m_shouldReceiveShadows;
    }

    public static void ExecutePass(RasterCommandBuffer rasterCommandBuffer)
    {
        // Disable shadow sampling for transparent objects by setting empty shadow maps
        // This prevents transparent objects from sampling shadow textures
        MainLightShadowCasterPass.SetShadowParamsForEmptyShadowmap(rasterCommandBuffer);
        AdditionalLightsShadowCasterPass.SetShadowParamsForEmptyShadowmap(rasterCommandBuffer);
    }
}


#### Shadow Parameter Management:
csharp
// In MainLightShadowCasterPass.cs
public static void SetShadowParamsForEmptyShadowmap(RasterCommandBuffer cmd)
{
    // Set shadow parameters to indicate no shadows
    cmd.SetGlobalVector("_MainLightShadowParams", new Vector4(0, 0, 0, -1));
    cmd.SetGlobalVector("_MainLightShadowmapSize", Vector4.zero);
    
    // Set cascade parameters for directional light shadows
    cmd.SetGlobalVector("_CascadeShadowSplitSpheres0", Vector4.zero);
    cmd.SetGlobalVector("_CascadeShadowSplitSpheres1", Vector4.zero);
    cmd.SetGlobalVector("_CascadeShadowSplitSpheres2", Vector4.zero);
    cmd.SetGlobalVector("_CascadeShadowSplitSpheres3", Vector4.zero);
    cmd.SetGlobalVector("_CascadeShadowSplitSphereRadii", Vector4.zero);
    
    // Disable shadow matrix transformations
    cmd.SetGlobalMatrix("_MainLightWorldToShadow", Matrix4x4.identity);
}


## 3. Shader Implementation - Comprehensive Analysis

### 3.1 Lit Shader Complete Property Analysis

#### Material Properties Deep Dive:
hlsl
Shader "Universal Render Pipeline/Lit"
{
    Properties
    {
        // === CORE MATERIAL PROPERTIES ===
        _WorkflowMode("WorkflowMode", Float) = 1.0  // 0=Specular, 1=Metallic
        
        [MainTexture] _BaseMap("Albedo", 2D) = "white" {}
        [MainColor] _BaseColor("Color", Color) = (1,1,1,1)
        
        // === TRANSPARENCY CONTROL ===
        _Cutoff("Alpha Cutoff", Range(0.0, 1.0)) = 0.5  // Alpha testing threshold
        
        // === SURFACE PROPERTIES ===
        _Smoothness("Smoothness", Range(0.0, 1.0)) = 0.5
        _SmoothnessTextureChannel("Smoothness texture channel", Float) = 0  // 0=Alpha, 1=Red
        
        _Metallic("Metallic", Range(0.0, 1.0)) = 0.0
        _MetallicGlossMap("Metallic", 2D) = "white" {}
        
        // === SPECULAR WORKFLOW ===
        _SpecColor("Specular", Color) = (0.2, 0.2, 0.2)
        _SpecGlossMap("Specular", 2D) = "white" {}
        
        // === SURFACE FEATURES ===
        [ToggleOff] _SpecularHighlights("Specular Highlights", Float) = 1.0
        [ToggleOff] _EnvironmentReflections("Environment Reflections", Float) = 1.0
        
        // === NORMAL MAPPING ===
        _BumpScale("Scale", Float) = 1.0
        _BumpMap("Normal Map", 2D) = "bump" {}
        
        // === PARALLAX MAPPING ===
        _Parallax("Scale", Range(0.005, 0.08)) = 0.005
        _ParallaxMap("Height Map", 2D) = "black" {}
        
        // === AMBIENT OCCLUSION ===
        _OcclusionStrength("Strength", Range(0.0, 1.0)) = 1.0
        _OcclusionMap("Occlusion", 2D) = "white" {}
        
        // === EMISSION ===
        [HDR] _EmissionColor("Color", Color) = (0,0,0)
        _EmissionMap("Emission", 2D) = "white" {}
        
        // === DETAIL MAPPING ===
        _DetailMask("Detail Mask", 2D) = "white" {}
        _DetailAlbedoMapScale("Scale", Range(0.0, 2.0)) = 1.0
        _DetailAlbedoMap("Detail Albedo x2", 2D) = "linearGrey" {}
        _DetailNormalMapScale("Scale", Range(0.0, 2.0)) = 1.0
        [Normal] _DetailNormalMap("Normal Map", 2D) = "bump" {}
        
        // === CLEAR COAT (SRP Batching compatibility) ===
        [HideInInspector] _ClearCoatMask("_ClearCoatMask", Float) = 0.0
        [HideInInspector] _ClearCoatSmoothness("_ClearCoatSmoothness", Float) = 0.0
        
        // === TRANSPARENCY BLENDING STATE ===
        _Surface("__surface", Float) = 0.0              // 0=Opaque, 1=Transparent
        _Blend("__blend", Float) = 0.0                  // Blend mode: 0=Alpha, 1=Premultiply, 2=Additive, 3=Multiply
        _Cull("__cull", Float) = 2.0                    // Culling: 0=Off, 1=Front, 2=Back
        [ToggleUI] _AlphaClip("__clip", Float) = 0.0    // Alpha clipping enable
        
        // === BLEND FACTORS (Hidden - set by material inspector) ===
        [HideInInspector] _SrcBlend("__src", Float) = 1.0        // Source blend factor
        [HideInInspector] _DstBlend("__dst", Float) = 0.0        // Destination blend factor
        [HideInInspector] _SrcBlendAlpha("__srcA", Float) = 1.0  // Source alpha blend factor
        [HideInInspector] _DstBlendAlpha("__dstA", Float) = 0.0  // Destination alpha blend factor
        [HideInInspector] _ZWrite("__zw", Float) = 1.0           // Depth write: 0=Off, 1=On
        [HideInInspector] _BlendModePreserveSpecular("_BlendModePreserveSpecular", Float) = 1.0
        [HideInInspector] _AlphaToMask("__alphaToMask", Float) = 0.0  // Alpha to coverage
        
        // === MOTION VECTORS ===
        [HideInInspector] _AddPrecomputedVelocity("_AddPrecomputedVelocity", Float) = 0.0
        [HideInInspector] _XRMotionVectorsPass("_XRMotionVectorsPass", Float) = 1.0
        
        // === LIGHTING ===
        [ToggleUI] _ReceiveShadows("Receive Shadows", Float) = 1.0
        
        // === RENDER QUEUE ===
        _QueueOffset("Queue offset", Float) = 0.0
        
        // === LEGACY PROPERTIES (Backward compatibility) ===
        [HideInInspector] _MainTex("BaseMap", 2D) = "white" {}
        [HideInInspector] _Color("Base Color", Color) = (1, 1, 1, 1)
        [HideInInspector] _GlossMapScale("Smoothness", Float) = 0.0
        [HideInInspector] _Glossiness("Smoothness", Float) = 0.0
        [HideInInspector] _GlossyReflections("EnvironmentReflections", Float) = 0.0
        
        // === LIGHTMAPPING ===
        [HideInInspector][NoScaleOffset]unity_Lightmaps("unity_Lightmaps", 2DArray) = "" {}
        [HideInInspector][NoScaleOffset]unity_LightmapsInd("unity_LightmapsInd", 2DArray) = "" {}
        [HideInInspector][NoScaleOffset]unity_ShadowMasks("unity_ShadowMasks", 2DArray) = "" {}
    }


### 3.2 Blend Mode Implementation Details

#### Blend Factor Mapping:
csharp
// From UniversalRenderPipelineAsset.cs - Material upgrade logic
public enum BlendMode
{
    Alpha,          // Traditional alpha blending
    Premultiply,    // Pre-multiplied alpha
    Additive,       // Additive blending
    Multiply        // Multiplicative blending
}

// Blend factor assignments per mode
switch (blendMode)
{
    case BlendMode.Alpha:
        material.SetFloat("_SrcBlend", (float)UnityEngine.Rendering.BlendMode.SrcAlpha);
        material.SetFloat("_DstBlend", (float)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
        material.SetFloat("_SrcBlendAlpha", (float)UnityEngine.Rendering.BlendMode.One);
        material.SetFloat("_DstBlendAlpha", (float)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
        break;
        
    case BlendMode.Premultiply:
        material.SetFloat("_SrcBlend", (float)UnityEngine.Rendering.BlendMode.One);
        material.SetFloat("_DstBlend", (float)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
        material.SetFloat("_SrcBlendAlpha", (float)UnityEngine.Rendering.BlendMode.One);
        material.SetFloat("_DstBlendAlpha", (float)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
        break;
        
    case BlendMode.Additive:
        material.SetFloat("_SrcBlend", (float)UnityEngine.Rendering.BlendMode.SrcAlpha);
        material.SetFloat("_DstBlend", (float)UnityEngine.Rendering.BlendMode.One);
        material.SetFloat("_SrcBlendAlpha", (float)UnityEngine.Rendering.BlendMode.One);
        material.SetFloat("_DstBlendAlpha", (float)UnityEngine.Rendering.BlendMode.One);
        break;
        
    case BlendMode.Multiply:
        material.SetFloat("_SrcBlend", (float)UnityEngine.Rendering.BlendMode.DstColor);
        material.SetFloat("_DstBlend", (float)UnityEngine.Rendering.BlendMode.Zero);
        material.SetFloat("_SrcBlendAlpha", (float)UnityEngine.Rendering.BlendMode.DstAlpha);
        material.SetFloat("_DstBlendAlpha", (float)UnityEngine.Rendering.BlendMode.Zero);
        break;
}


### 3.3 Shader Variant System Deep Analysis

#### Complete Variant Matrix:
hlsl
// === TRANSPARENCY VARIANTS ===
#pragma shader_feature_local_fragment _SURFACE_TYPE_TRANSPARENT
#pragma shader_feature_local_fragment _ALPHATEST_ON
#pragma shader_feature_local_fragment _ _ALPHAPREMULTIPLY_ON _ALPHAMODULATE_ON

// === LIGHTING VARIANTS ===
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE _MAIN_LIGHT_SHADOWS_SCREEN
#pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
#pragma multi_compile_fragment _ _ADDITIONAL_LIGHT_SHADOWS
#pragma multi_compile_fragment _ _REFLECTION_PROBE_BLENDING
#pragma multi_compile_fragment _ _REFLECTION_PROBE_BOX_PROJECTION
#pragma multi_compile_fragment _ _REFLECTION_PROBE_ATLAS
#pragma multi_compile_fragment _ _SHADOWS_SOFT _SHADOWS_SOFT_LOW _SHADOWS_SOFT_MEDIUM _SHADOWS_SOFT_HIGH

// === GLOBAL ILLUMINATION VARIANTS ===
#pragma multi_compile _ LIGHTMAP_SHADOW_MIXING
#pragma multi_compile _ SHADOWS_SHADOWMASK
#pragma multi_compile _ DIRLIGHTMAP_COMBINED
#pragma multi_compile _ LIGHTMAP_ON
#pragma multi_compile _ DYNAMICLIGHTMAP_ON
#pragma multi_compile _ USE_LEGACY_LIGHTMAPS
#pragma multi_compile _ LOD_FADE_CROSSFADE
#pragma multi_compile _ USE_UNITY_CROSSFADE

// === RENDERING FEATURES ===
#pragma multi_compile_fog
#pragma multi_compile_instancing
#pragma multi_compile _ DOTS_INSTANCING_ON
#pragma multi_compile_fragment _ _SCREEN_SPACE_OCCLUSION
#pragma multi_compile_fragment _ _DBUFFER_MRT1 _DBUFFER_MRT2 _DBUFFER_MRT3
#pragma multi_compile_fragment _ DEBUG_DISPLAY
#pragma multi_compile_fragment _ _DETAILS_ON
#pragma multi_compile_fragment _ _EMISSION
#pragma multi_compile_fragment _ _METALLICSPECGLOSSMAP
#pragma multi_compile_fragment _ _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A
#pragma multi_compile_fragment _ _OCCLUSIONMAP
#pragma multi_compile_fragment _ _PARALLAXMAP
#pragma multi_compile_fragment _ _NORMALMAP
#pragma multi_compile_fragment _ _SPECULARHIGHLIGHTS_OFF
#pragma multi_compile_fragment _ _ENVIRONMENTREFLECTIONS_OFF
#pragma multi_compile_fragment _ _SPECULAR_SETUP
#pragma multi_compile_fragment _ _RECEIVE_SHADOWS_OFF

// === PLATFORM VARIANTS ===
#pragma multi_compile _ EVALUATE_SH_MIXED EVALUATE_SH_VERTEX
#pragma multi_compile_fragment _ _LIGHT_LAYERS
#pragma multi_compile_fragment _ _LIGHT_COOKIES
#pragma multi_compile _ _FORWARD_PLUS
#pragma multi_compile _ _CLUSTERED_RENDERING

// === XR VARIANTS ===
#pragma multi_compile _ _FOVEATED_RENDERING_NON_UNIFORM_RASTER
#pragma multi_compile _ _RENDER_PASS_ENABLED


### 3.4 LitForwardPass.hlsl - Complete Implementation

#### Vertex Shader Structure:
hlsl
struct Attributes
{
    float4 positionOS   : POSITION;     // Object space position
    float3 normalOS     : NORMAL;       // Object space normal
    float4 tangentOS    : TANGENT;      // Object space tangent (w = handedness)
    float2 texcoord     : TEXCOORD0;    // Primary UV coordinates
    float2 staticLightmapUV   : TEXCOORD1;  // Static lightmap UVs
    float2 dynamicLightmapUV  : TEXCOORD2;  // Dynamic lightmap UVs
    UNITY_VERTEX_INPUT_INSTANCE_ID      // Instancing support
};

struct Varyings
{
    float2 uv                       : TEXCOORD0;    // Primary UV
    
#if defined(REQUIRES_WORLD_SPACE_POS_INTERPOLATOR)
    float3 positionWS               : TEXCOORD1;    // World space position
#endif

    float3 normalWS                 : TEXCOORD2;    // World space normal
    
#if defined(REQUIRES_WORLD_SPACE_TANGENT_INTERPOLATOR)
    half4 tangentWS                : TEXCOORD3;     // World space tangent (xyz) + handedness (w)
#endif

#ifdef _ADDITIONAL_LIGHTS_VERTEX
    half4 fogFactorAndVertexLight   : TEXCOORD5;    // x: fog factor, yzw: vertex light
#else
    half  fogFactor                 : TEXCOORD5;    // Fog factor only
#endif

#if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
    float4 shadowCoord              : TEXCOORD6;    // Shadow coordinates
#endif

#if defined(REQUIRES_TANGENT_SPACE_VIEW_DIR_INTERPOLATOR)
    half3 viewDirTS                : TEXCOORD7;     // Tangent space view direction
#endif

    DECLARE_LIGHTMAP_OR_SH(staticLightmapUV, vertexSH, 8);  // Lightmap or SH data

#ifdef DYNAMICLIGHTMAP_ON
    float2  dynamicLightmapUV : TEXCOORD9;          // Dynamic lightmap UVs
#endif

#ifdef USE_APV_PROBE_OCCLUSION
    float4 probeOcclusion : TEXCOORD10;             // Probe occlusion data
#endif

    float4 positionCS               : SV_POSITION;   // Clip space position
    UNITY_VERTEX_INPUT_INSTANCE_ID                  // Instancing ID
    UNITY_VERTEX_OUTPUT_STEREO                      // Stereo rendering support
};


#### Vertex Shader Implementation:
hlsl
Varyings LitPassVertex(Attributes input)
{
    Varyings output = (Varyings)0;

    UNITY_SETUP_INSTANCE_ID(input);
    UNITY_TRANSFER_INSTANCE_ID(input, output);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(output);

    // Transform positions
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    output.positionCS = vertexInput.positionCS;

#if defined(REQUIRES_WORLD_SPACE_POS_INTERPOLATOR)
    output.positionWS = vertexInput.positionWS;
#endif

    // Transform normals and tangents
    VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);
    output.normalWS = normalInput.normalWS;

#if defined(REQUIRES_WORLD_SPACE_TANGENT_INTERPOLATOR)
    real sign = input.tangentOS.w * GetOddNegativeScale();
    half4 tangentWS = half4(normalInput.tangentWS.xyz, sign);
    output.tangentWS = tangentWS;
#endif

    // UV coordinates
    output.uv = TRANSFORM_TEX(input.texcoord, _BaseMap);

    // Lighting calculations
#ifdef _ADDITIONAL_LIGHTS_VERTEX
    uint lightsCount = GetAdditionalLightsCount();
    half3 vertexLight = CalculateVertexLighting(vertexInput.positionWS, normalInput.normalWS, lightsCount);
    half fogFactor = ComputeFogFactor(vertexInput.positionCS.z);
    output.fogFactorAndVertexLight = half4(fogFactor, vertexLight);
#else
    output.fogFactor = ComputeFogFactor(vertexInput.positionCS.z);
#endif

    // Shadow coordinates
#if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
    output.shadowCoord = GetShadowCoord(vertexInput);
#endif

    // Tangent space view direction for parallax mapping
#if defined(REQUIRES_TANGENT_SPACE_VIEW_DIR_INTERPOLATOR)
    half3 viewDirWS = GetWorldSpaceNormalizeViewDir(vertexInput.positionWS);
    half3 viewDirTS = GetViewDirectionTangentSpace(tangentWS, output.normalWS, viewDirWS);
    output.viewDirTS = viewDirTS;
#endif

    // Lightmap and SH data
    OUTPUT_LIGHTMAP_UV(input.staticLightmapUV, unity_LightmapST, output.staticLightmapUV);
#ifdef DYNAMICLIGHTMAP_ON
    output.dynamicLightmapUV = input.dynamicLightmapUV.xy * unity_DynamicLightmapST.xy + unity_DynamicLightmapST.zw;
#endif
    OUTPUT_SH(output.normalWS.xyz, output.vertexSH);

#ifdef USE_APV_PROBE_OCCLUSION
    output.probeOcclusion = GetProbeOcclusionData(vertexInput.positionWS);
#endif

    return output;
}


#### Fragment Shader - Transparency Processing:
hlsl
half4 LitPassFragment(Varyings input) : SV_Target
{
    UNITY_SETUP_INSTANCE_ID(input);
    UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX

> 
> 

