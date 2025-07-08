# Unity URP Transparency Architecture Diagrams

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Unity Engine Core"
        UE[Unity Engine]
        GM[Graphics Module]
        RM[Rendering Module]
    end
    
    subgraph "Render Pipeline Core (v17.1.0)"
        RPC[RenderPipeline Base]
        RG[RenderGraph System]
        RTH[RTHandle System]
        SRP[Scriptable Render Pipeline]
    end
    
    subgraph "Universal Render Pipeline (v17.1.0)"
        URP[UniversalRenderPipeline]
        UR[UniversalRenderer]
        SR[ScriptableRenderer]
        
        subgraph "Transparency Passes"
            DOP[DrawObjectsPass]
            TSP[TransparentSettingsPass]
            RTP[RenderTransparentForwardPass]
        end
        
        subgraph "Supporting Systems"
            LS[Lighting System]
            SS[Shadow System]
            PS[Post-Processing]
        end
    end
    
    subgraph "Shader Graph & Shaders"
        SG[Shader Graph]
        LS_SHADER[Lit.shader]
        HLSL[HLSL Libraries]
    end
    
    subgraph "Material System"
        MS[Material Inspector]
        MP[Material Properties]
        BM[Blend Modes]
    end
    
    UE --> GM
    GM --> RM
    RM --> RPC
    RPC --> RG
    RPC --> RTH
    RPC --> SRP
    SRP --> URP
    URP --> UR
    UR --> SR
    SR --> DOP
    SR --> TSP
    SR --> RTP
    URP --> LS
    URP --> SS
    URP --> PS
    SG --> LS_SHADER
    LS_SHADER --> HLSL
    MS --> MP
    MP --> BM
    BM --> LS_SHADER
```

## 2. Transparency Rendering Pipeline Flow

```mermaid
flowchart TD
    START([Frame Start]) --> SETUP[Pipeline Setup]
    SETUP --> SHADOW[Shadow Generation]
    SHADOW --> DEPTH_PRE[Depth Prepass?]
    
    DEPTH_PRE -->|Yes| DEPTH_PASS[Depth Only Pass]
    DEPTH_PRE -->|No| GBUFFER_CHECK
    DEPTH_PASS --> GBUFFER_CHECK
    
    GBUFFER_CHECK{Deferred Mode?}
    GBUFFER_CHECK -->|Yes| GBUFFER[G-Buffer Pass]
    GBUFFER_CHECK -->|No| OPAQUE
    GBUFFER --> DEFERRED[Deferred Lighting]
    DEFERRED --> OPAQUE
    
    OPAQUE[Render Opaque Objects] --> SKYBOX[Render Skybox]
    SKYBOX --> DEPTH_COPY{Copy Depth?}
    
    DEPTH_COPY -->|Yes| COPY_DEPTH[Copy Depth Pass]
    DEPTH_COPY -->|No| TRANS_SETUP
    COPY_DEPTH --> TRANS_SETUP
    
    TRANS_SETUP[Transparent Settings Pass] --> TRANS_SORT[Sort Transparent Objects]
    TRANS_SORT --> TRANS_RENDER[Render Transparent Objects]
    
    subgraph "Transparency Rendering Detail"
        TRANS_RENDER --> SHADOW_CHECK{Receive Shadows?}
        SHADOW_CHECK -->|No| DISABLE_SHADOWS[Disable Shadow Sampling]
        SHADOW_CHECK -->|Yes| ENABLE_SHADOWS[Enable Shadow Sampling]
        DISABLE_SHADOWS --> BLEND_SETUP
        ENABLE_SHADOWS --> BLEND_SETUP
        BLEND_SETUP[Setup Blend State] --> DRAW_TRANS[Draw Transparent Geometry]
    end
    
    DRAW_TRANS --> POST[Post-Processing]
    POST --> UI[UI Rendering]
    UI --> END([Frame End])
    
    style TRANS_RENDER fill:#ff9999
    style TRANS_SORT fill:#ff9999
    style TRANS_SETUP fill:#ff9999
    style DRAW_TRANS fill:#ff9999
```

## 3. Class Hierarchy Diagram

```mermaid
classDiagram
    class RenderPipeline {
        <<abstract>>
        +Render(ScriptableRenderContext, List~Camera~)
        +InternalRender(ScriptableRenderContext, List~Camera~)
        +Dispose()
    }
    
    class UniversalRenderPipeline {
        -UniversalRenderPipelineGlobalSettings m_GlobalSettings
        -UniversalRenderPipelineRuntimeTextures runtimeTextures
        +static RenderGraph s_RenderGraph
        +static RTHandleResourcePool s_RTHandlePool
        +Render(ScriptableRenderContext, List~Camera~)
        +RenderSingleCameraInternal()
        +RenderCameraStack()
    }
    
    class ScriptableRenderer {
        <<abstract>>
        #List~ScriptableRenderPass~ m_ActiveRenderPassQueue
        #List~ScriptableRendererFeature~ m_RendererFeatures
        +Setup(ScriptableRenderContext, RenderingData)
        +Execute(ScriptableRenderContext, RenderingData)
        +EnqueuePass(ScriptableRenderPass)
        +SupportedCameraStackingTypes() int
    }
    
    class UniversalRenderer {
        -RenderingMode m_RenderingMode
        -DrawObjectsPass m_RenderOpaqueForwardPass
        -DrawObjectsPass m_RenderTransparentForwardPass
        -TransparentSettingsPass m_TransparentSettingsPass
        -DeferredLights m_DeferredLights
        +Setup(ScriptableRenderContext, RenderingData)
        +SupportedCameraStackingTypes() int
        +SupportsMotionVectors() bool
    }
    
    class ScriptableRenderPass {
        <<abstract>>
        +RenderPassEvent renderPassEvent
        +ProfilingSampler profilingSampler
        +Execute(ScriptableRenderContext, RenderingData)
        +Configure(CommandBuffer, RenderTextureDescriptor)
        +OnCameraSetup(CommandBuffer, RenderingData)
    }
    
    class DrawObjectsPass {
        -FilteringSettings m_FilteringSettings
        -RenderStateBlock m_RenderStateBlock
        -List~ShaderTagId~ m_ShaderTagIdList
        -bool m_IsOpaque
        -bool m_ShouldTransparentsReceiveShadows
        -bool m_IsActiveTargetBackBuffer
        +Execute(ScriptableRenderContext, RenderingData)
        +Render(RenderGraph, FrameData, TextureHandle, TextureHandle)
    }
    
    class TransparentSettingsPass {
        -bool m_shouldReceiveShadows
        +Setup() bool
        +ExecutePass(RasterCommandBuffer)
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class RenderingData {
        +CameraData cameraData
        +LightData lightData
        +ShadowData shadowData
        +PostProcessingData postProcessingData
        +CullingResults cullResults
        +bool supportsDynamicBatching
        +PerObjectData perObjectData
    }
    
    class CameraData {
        +Camera camera
        +CameraRenderType renderType
        +RenderTexture targetTexture
        +RenderTextureDescriptor cameraTargetDescriptor
        +bool isSceneViewCamera
        +bool isPreviewCamera
        +bool requiresDepthTexture
        +bool requiresOpaqueTexture
    }
    
    RenderPipeline <|-- UniversalRenderPipeline
    ScriptableRenderer <|-- UniversalRenderer
    ScriptableRenderPass <|-- DrawObjectsPass
    ScriptableRenderPass <|-- TransparentSettingsPass
    
    UniversalRenderPipeline --> UniversalRenderer : creates
    UniversalRenderer --> DrawObjectsPass : contains
    UniversalRenderer --> TransparentSettingsPass : contains
    DrawObjectsPass --> RenderingData : uses
    DrawObjectsPass --> CameraData : uses
```

## 4. Transparency Pass Data Flow

```mermaid
sequenceDiagram
    participant URP as UniversalRenderPipeline
    participant UR as UniversalRenderer
    participant DOP as DrawObjectsPass
    participant TSP as TransparentSettingsPass
    participant GPU as GPU/CommandBuffer
    participant Shader as Lit.shader
    
    URP->>UR: Setup(context, renderingData)
    UR->>UR: Configure render passes
    
    Note over UR: Opaque rendering completed
    
    UR->>TSP: Setup()
    TSP-->>UR: shouldReceiveShadows = false
    
    alt Transparent objects should NOT receive shadows
        UR->>TSP: Execute(context, renderingData)
        TSP->>GPU: SetShadowParamsForEmptyShadowmap()
        GPU->>Shader: _MainLightShadowParams = (0,0,0,-1)
    end
    
    UR->>DOP: Execute(context, renderingData)
    DOP->>DOP: Setup sorting (CommonTransparent)
    DOP->>GPU: DrawRenderers(cullResults, drawingSettings, filteringSettings)
    
    loop For each transparent object
        GPU->>Shader: Vertex Shader
        Shader->>Shader: Transform vertices
        Shader->>GPU: Rasterization
        GPU->>Shader: Fragment Shader
        Shader->>Shader: Sample textures
        Shader->>Shader: Calculate lighting
        Shader->>Shader: Apply transparency
        Note over Shader: color.a = OutputAlpha(color.a, IsSurfaceTypeTransparent(_Surface))
        Shader->>GPU: Blend with framebuffer
        Note over GPU: Blend: [_SrcBlend][_DstBlend], [_SrcBlendAlpha][_DstBlendAlpha]
    end
    
    DOP-->>UR: Transparent rendering complete
    UR-->>URP: All passes complete
```

## 5. Shader Variant System Architecture

```mermaid
graph TB
    subgraph "Shader Compilation System"
        SC[Shader Compiler]
        SV[Shader Variants]
        SK[Shader Keywords]
    end
    
    subgraph "Transparency Variants"
        ST[_SURFACE_TYPE_TRANSPARENT]
        AT[_ALPHATEST_ON]
        AP[_ALPHAPREMULTIPLY_ON]
        AM[_ALPHAMODULATE_ON]
    end
    
    subgraph "Lighting Variants"
        MLS[_MAIN_LIGHT_SHADOWS]
        MLSC[_MAIN_LIGHT_SHADOWS_CASCADE]
        ALS[_ADDITIONAL_LIGHT_SHADOWS]
        ALV[_ADDITIONAL_LIGHTS_VERTEX]
        AL[_ADDITIONAL_LIGHTS]
    end
    
    subgraph "Feature Variants"
        RPB[_REFLECTION_PROBE_BLENDING]
        SS[_SHADOWS_SOFT]
        SSAO[_SCREEN_SPACE_OCCLUSION]
        FOG[FOG_LINEAR/FOG_EXP/FOG_EXP2]
    end
    
    subgraph "Platform Variants"
        INST[INSTANCING_ON]
        DOTS[DOTS_INSTANCING_ON]
        XR[XR_RENDERING]
        FP[_FORWARD_PLUS]
    end
    
    SC --> SV
    SV --> SK
    
    SK --> ST
    SK --> AT
    SK --> AP
    SK --> AM
    
    SK --> MLS
    SK --> MLSC
    SK --> ALS
    SK --> ALV
    SK --> AL
    
    SK --> RPB
    SK --> SS
    SK --> SSAO
    SK --> FOG
    
    SK --> INST
    SK --> DOTS
    SK --> XR
    SK --> FP
    
    subgraph "Runtime Selection"
        MS[Material Settings]
        QS[Quality Settings]
        PS[Platform Settings]
        CS[Camera Settings]
    end
    
    MS --> ST
    MS --> AT
    MS --> AP
    MS --> AM
    
    QS --> SS
    QS --> SSAO
    
    PS --> INST
    PS --> DOTS
    PS --> XR
    
    CS --> MLS
    CS --> ALS
```

## 6. Blend Mode State Machine

```mermaid
stateDiagram-v2
    [*] --> Opaque : _Surface = 0
    [*] --> Transparent : _Surface = 1
    
    state Opaque {
        [*] --> OpaqueState
        OpaqueState : _SrcBlend = One
        OpaqueState : _DstBlend = Zero
        OpaqueState : _ZWrite = On
        OpaqueState : Queue = Geometry
    }
    
    state Transparent {
        [*] --> BlendModeSelection
        
        state BlendModeSelection {
            [*] --> Alpha : _Blend = 0
            [*] --> Premultiply : _Blend = 1
            [*] --> Additive : _Blend = 2
            [*] --> Multiply : _Blend = 3
        }
        
        state Alpha {
            AlphaState : _SrcBlend = SrcAlpha
            AlphaState : _DstBlend = OneMinusSrcAlpha
            AlphaState : _SrcBlendAlpha = One
            AlphaState : _DstBlendAlpha = OneMinusSrcAlpha
            AlphaState : _ZWrite = Off
            AlphaState : Queue = Transparent
        }
        
        state Premultiply {
            PremultiplyState : _SrcBlend = One
            PremultiplyState : _DstBlend = OneMinusSrcAlpha
            PremultiplyState : _SrcBlendAlpha = One
            PremultiplyState : _DstBlendAlpha = OneMinusSrcAlpha
            PremultiplyState : _ZWrite = Off
            PremultiplyState : Queue = Transparent
        }
        
        state Additive {
            AdditiveState : _SrcBlend = SrcAlpha
            AdditiveState : _DstBlend = One
            AdditiveState : _SrcBlendAlpha = One
            AdditiveState : _DstBlendAlpha = One
            AdditiveState : _ZWrite = Off
            AdditiveState : Queue = Transparent
        }
        
        state Multiply {
            MultiplyState : _SrcBlend = DstColor
            MultiplyState : _DstBlend = Zero
            MultiplyState : _SrcBlendAlpha = DstAlpha
            MultiplyState : _DstBlendAlpha = Zero
            MultiplyState : _ZWrite = Off
            MultiplyState : Queue = Transparent
        }
    }
    
    Transparent --> AlphaClipping : _AlphaClip = 1
    
    state AlphaClipping {
        ClipState : Alpha Test Enabled
        ClipState : _Cutoff threshold
        ClipState : discard if alpha < _Cutoff
    }
```

## 7. RenderGraph Integration Architecture

```mermaid
graph TB
    subgraph "RenderGraph System"
        RG[RenderGraph]
        RGB[RenderGraphBuilder]
        RGP[RenderGraphPass]
        RGR[RenderGraphResources]
    end
    
    subgraph "Resource Management"
        RTH[RTHandle System]
        RTHP[RTHandleResourcePool]
        TH[TextureHandle]
        BH[BufferHandle]
    end
    
    subgraph "URP RenderGraph Integration"
        URRG[UniversalRendererRenderGraph]
        RGFP[RenderGraphForwardPass]
        RGTP[RenderGraphTransparentPass]
    end
    
    subgraph "Pass Data Structures"
        PD[PassData]
        FD[FrameData]
        URD[UniversalResourceData]
        UCD[UniversalCameraData]
    end
    
    subgraph "Execution"
        CMD[CommandBuffer]
        RCMD[RasterCommandBuffer]
        GPU[GPU Execution]
    end
    
    RG --> RGB
    RGB --> RGP
    RGP --> RGR
    
    RTH --> RTHP
    RTHP --> TH
    RTHP --> BH
    
    URRG --> RG
    URRG --> RGFP
    URRG --> RGTP
    
    RGFP --> PD
    RGTP --> PD
    PD --> FD
    FD --> URD
    URD --> UCD
    
    RGP --> CMD
    CMD --> RCMD
    RCMD --> GPU
    
    TH --> RGR
    BH --> RGR
    RGR --> RCMD
```

## 8. Memory Layout and Resource Flow

```mermaid
graph LR
    subgraph "CPU Memory"
        subgraph "Managed Heap"
            URP_OBJ[URP Objects]
            PASS_OBJ[Pass Objects]
            DATA_OBJ[Data Structures]
        end
        
        subgraph "Native Memory"
            CMD_BUF[Command Buffers]
            CULL_RES[Culling Results]
            RENDER_LIST[Renderer Lists]
        end
    end
    
    subgraph "GPU Memory"
        subgraph "Textures"
            COLOR_TEX[Color Targets]
            DEPTH_TEX[Depth Targets]
            SHADOW_TEX[Shadow Maps]
        end
        
        subgraph "Buffers"
            VTX_BUF[Vertex Buffers]
            IDX_BUF[Index Buffers]
            CONST_BUF[Constant Buffers]
        end
        
        subgraph "Shaders"
            VS[Vertex Shaders]
            FS[Fragment Shaders]
            VARIANTS[Shader Variants]
        end
    end
    
    URP_OBJ --> CMD_BUF
    PASS_OBJ --> CMD_BUF
    DATA_OBJ --> CONST_BUF
    
    CMD_BUF --> COLOR_TEX
    CMD_BUF --> DEPTH_TEX
    CMD_BUF --> SHADOW_TEX
    
    CULL_RES --> RENDER_LIST
    RENDER_LIST --> VTX_BUF
    RENDER_LIST --> IDX_BUF
    
    VARIANTS --> VS
    VARIANTS --> FS
    
    subgraph "Transparency Specific"
        TRANS_SORT[Transparent Sorting]
        BLEND_STATE[Blend State]
        ALPHA_TEX[Alpha Textures]
    end
    
    TRANS_SORT --> RENDER_LIST
    BLEND_STATE --> COLOR_TEX
    ALPHA_TEX --> FS
```
