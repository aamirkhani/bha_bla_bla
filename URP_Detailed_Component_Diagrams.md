# Unity URP - Detailed Component Diagrams

## 1. UniversalRenderer Component Breakdown

```mermaid
classDiagram
    class UniversalRenderer {
        -m_RenderingMode: RenderingMode
        -m_DepthPrimingMode: DepthPrimingMode
        -m_CopyDepthMode: CopyDepthMode
        -m_IntermediateTextureMode: IntermediateTextureMode
        
        -m_ForwardLights: ForwardLights
        -m_DeferredLights: DeferredLights
        -m_PostProcessPasses: PostProcessPasses
        -m_ColorBufferSystem: RenderTargetBufferSystem
        
        -m_DepthPrepass: DepthOnlyPass
        -m_DepthNormalPrepass: DepthNormalOnlyPass
        -m_MainLightShadowCasterPass: MainLightShadowCasterPass
        -m_AdditionalLightsShadowCasterPass: AdditionalLightsShadowCasterPass
        -m_GBufferPass: GBufferPass
        -m_DeferredPass: DeferredPass
        -m_RenderOpaqueForwardPass: DrawObjectsPass
        -m_RenderTransparentForwardPass: DrawObjectsPass
        -m_DrawSkyboxPass: DrawSkyboxPass
        -m_CopyDepthPass: CopyDepthPass
        -m_CopyColorPass: CopyColorPass
        -m_FinalBlitPass: FinalBlitPass
        
        -m_ActiveCameraColorAttachment: RTHandle
        -m_ActiveCameraDepthAttachment: RTHandle
        -m_CameraDepthAttachment: RTHandle
        -m_DepthTexture: RTHandle
        -m_NormalsTexture: RTHandle
        -m_OpaqueColor: RTHandle
        -m_MotionVectorColor: RTHandle
        
        +Setup(ScriptableRenderContext, RenderingData)
        +SetupLights(ScriptableRenderContext, RenderingData)
        +SetupCullingParameters(ScriptableCullingParameters, CameraData)
        +FinishRendering(CommandBuffer)
        -CreateCameraRenderTarget(ScriptableRenderContext, RenderTextureDescriptor)
        -EnqueueDeferred(RenderTextureDescriptor, bool, bool, bool, bool, bool)
        -GetRenderPassInputs(bool, bool, bool): RenderPassInputSummary
        -RequiresIntermediateColorTexture(UniversalCameraData, RenderPassInputSummary): bool
        -CanCopyDepth(UniversalCameraData): bool
        -IsDepthPrimingEnabled(UniversalCameraData): bool
    }
    
    class ForwardLights {
        -m_AdditionalLightPositions: Vector4[]
        -m_AdditionalLightColors: Vector4[]
        -m_AdditionalLightAttenuations: Vector4[]
        -m_AdditionalLightSpotDirections: Vector4[]
        -m_UseStructuredBuffer: bool
        -m_UseForwardPlus: bool
        -m_LightCookieManager: LightCookieManager
        -m_ReflectionProbeManager: ReflectionProbeManager
        
        +PreSetup(UniversalRenderingData, UniversalCameraData, UniversalLightData)
        +Setup(ScriptableRenderContext, RenderingData)
        +SetupLights(UnsafeCommandBuffer, UniversalRenderingData, UniversalCameraData, UniversalLightData)
        -SetupShaderLightConstants(UnsafeCommandBuffer, CullingResults, UniversalLightData)
        -SetupMainLightConstants(UnsafeCommandBuffer, UniversalLightData)
        -SetupAdditionalLightConstants(UnsafeCommandBuffer, CullingResults, UniversalLightData)
        -SetupPerObjectLightIndices(CullingResults, UniversalLightData): int
    }
    
    class DeferredLights {
        -m_StencilVisLights: NativeArray~ushort~
        -m_StencilVisLightOffsets: NativeArray~ushort~
        -m_SphereMesh: Mesh
        -m_HemisphereMesh: Mesh
        -m_FullscreenMesh: Mesh
        -m_StencilDeferredMaterial: Material
        -m_ClusterDeferredMaterial: Material
        -m_LightCookieManager: LightCookieManager
        
        +GbufferAttachments: RTHandle[]
        +GbufferFormats: GraphicsFormat[]
        +UseFramebufferFetch: bool
        +AccurateGbufferNormals: bool
        +MixedLightingSetup: MixedLightingSetup
        
        +Setup(AdditionalLightsShadowCasterPass, bool, bool, bool, RTHandle, RTHandle, RTHandle)
        +SetupLights(CommandBuffer, UniversalCameraData, Vector2Int, UniversalLightData)
        +ExecuteDeferredPass(RasterCommandBuffer, UniversalCameraData, UniversalLightData, UniversalShadowData)
        -RenderStencilLights(RasterCommandBuffer, UniversalLightData, UniversalShadowData, bool)
        -RenderClusterLights(RasterCommandBuffer, UniversalShadowData)
        -CreateGbufferResources()
        -ResolveMixedLightingMode(UniversalLightData)
    }
    
    UniversalRenderer --> ForwardLights
    UniversalRenderer --> DeferredLights
```

## 2. Data Container Relationships

```mermaid
classDiagram
    class ContextContainer {
        <<abstract>>
        +Get~T~(): T
        +Create~T~(): T
        +Reset()
    }
    
    class UniversalRenderingData {
        +commandBuffer: CommandBuffer
        +cullResults: CullingResults
        +supportsDynamicBatching: bool
        +perObjectData: PerObjectData
        +postProcessingEnabled: bool
        +Reset()
    }
    
    class UniversalCameraData {
        -m_ViewMatrix: Matrix4x4
        -m_ProjectionMatrix: Matrix4x4
        -m_JitterMatrix: Matrix4x4
        -m_HistoryManager: UniversalCameraHistory
        
        +camera: Camera
        +renderType: CameraRenderType
        +targetTexture: RenderTexture
        +cameraTargetDescriptor: RenderTextureDescriptor
        +renderScale: float
        +clearDepth: bool
        +isHdrEnabled: bool
        +requiresDepthTexture: bool
        +requiresOpaqueTexture: bool
        +xrRendering: bool
        +maxShadowDistance: float
        +postProcessEnabled: bool
        +antialiasing: AntialiasingMode
        +renderer: ScriptableRenderer
        +worldSpaceCameraPos: Vector3
        +backgroundColor: Color
        
        +GetViewMatrix(int): Matrix4x4
        +GetProjectionMatrix(int): Matrix4x4
        +GetGPUProjectionMatrix(int): Matrix4x4
        +IsHandleYFlipped(RTHandle): bool
        +IsCameraProjectionMatrixFlipped(): bool
        +IsTemporalAAEnabled(): bool
        +Reset()
    }
    
    class UniversalLightData {
        +mainLightIndex: int
        +additionalLightsCount: int
        +maxPerObjectAdditionalLightsCount: int
        +visibleLights: NativeArray~VisibleLight~
        +shadeAdditionalLightsPerVertex: bool
        +supportsMixedLighting: bool
        +reflectionProbeBoxProjection: bool
        +reflectionProbeBlending: bool
        +reflectionProbeAtlas: bool
        +supportsLightLayers: bool
        +supportsAdditionalLights: bool
        +Reset()
    }
    
    class UniversalShadowData {
        +supportsMainLightShadows: bool
        +mainLightShadowmapWidth: int
        +mainLightShadowmapHeight: int
        +mainLightShadowCascadesCount: int
        +mainLightShadowCascadesSplit: Vector3
        +mainLightShadowCascadeBorder: float
        +supportsAdditionalLightShadows: bool
        +additionalLightsShadowmapWidth: int
        +additionalLightsShadowmapHeight: int
        +supportsSoftShadows: bool
        +shadowmapDepthBufferBits: int
        +bias: List~Vector4~
        +resolution: List~int~
        +Reset()
    }
    
    ContextContainer <|-- UniversalRenderingData
    ContextContainer <|-- UniversalCameraData
    ContextContainer <|-- UniversalLightData
    ContextContainer <|-- UniversalShadowData
```

## 3. Render Pass Hierarchy and Execution

```mermaid
classDiagram
    class ScriptableRenderPass {
        <<abstract>>
        +renderPassEvent: RenderPassEvent
        +colorAttachmentHandles: RTHandle[]
        +depthAttachmentHandle: RTHandle
        +input: ScriptableRenderPassInput
        +clearFlag: ClearFlag
        +clearColor: Color
        +requiresIntermediateTexture: bool
        
        +ConfigureInput(ScriptableRenderPassInput)
        +ConfigureTarget(RTHandle, RTHandle)
        +ConfigureClear(ClearFlag, Color)
        +Configure(CommandBuffer, RenderTextureDescriptor)
        +Execute(ScriptableRenderContext, RenderingData)
        +OnCameraCleanup(CommandBuffer)
        +RecordRenderGraph(RenderGraph, ContextContainer)
    }
    
    class DepthOnlyPass {
        -m_FilteringSettings: FilteringSettings
        -m_ShaderTagIds: ShaderTagId[]
        
        +Setup(RenderTextureDescriptor, RTHandle)
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class DepthNormalOnlyPass {
        -m_FilteringSettings: FilteringSettings
        -m_ShaderTagIds: ShaderTagId[]
        -m_DecalLayersTexture: RTHandle
        
        +Setup(RTHandle, RTHandle)
        +Setup(RTHandle, RTHandle, RTHandle)
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class DrawObjectsPass {
        -m_FilteringSettings: FilteringSettings
        -m_RenderStateBlock: RenderStateBlock
        -m_ShaderTagIds: ShaderTagId[]
        -m_IsOpaque: bool
        
        +Execute(ScriptableRenderContext, RenderingData)
        -RenderBlock(ScriptableRenderContext, CommandBuffer, CullingResults, DrawingSettings, FilteringSettings, RenderStateBlock)
    }
    
    class GBufferPass {
        -m_FilteringSettings: FilteringSettings
        -m_RenderStateBlock: RenderStateBlock
        -m_ShaderTagIds: ShaderTagId[]
        -m_DeferredLights: DeferredLights
        
        +Execute(ScriptableRenderContext, RenderingData)
        +Dispose()
    }
    
    class DeferredPass {
        -m_DeferredLights: DeferredLights
        
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class MainLightShadowCasterPass {
        -m_MainLightShadowmapTexture: RTHandle
        -m_MaxShadowDistanceSq: float
        -m_CascadeSplitDistances: Vector4
        -m_ShadowCasterCascadesCount: int
        
        +Setup(UniversalRenderingData, UniversalCameraData, UniversalLightData, UniversalShadowData): bool
        +Execute(ScriptableRenderContext, RenderingData)
        -SetupMainLightShadowReceiverConstants(CommandBuffer, VisibleLight, bool)
        -RenderMainLightCascadeShadowmap(ScriptableRenderContext, CommandBuffer, int, Matrix4x4, float, float)
    }
    
    class AdditionalLightsShadowCasterPass {
        -m_AdditionalLightsShadowmapHandle: RTHandle
        -m_AdditionalShadowsBufferId: int
        -m_AdditionalShadowsIndicesId: int
        -m_ShadowAtlasLayout: AdditionalLightsShadowAtlasLayout
        
        +Setup(UniversalRenderingData, UniversalCameraData, UniversalLightData, UniversalShadowData): bool
        +Execute(ScriptableRenderContext, RenderingData)
        +GetShadowLightIndexFromLightIndex(int): int
        -RenderAdditionalShadowmapAtlas(ScriptableRenderContext, CommandBuffer, int, VisibleLight[], List~int~)
    }
    
    ScriptableRenderPass <|-- DepthOnlyPass
    ScriptableRenderPass <|-- DepthNormalOnlyPass
    ScriptableRenderPass <|-- DrawObjectsPass
    ScriptableRenderPass <|-- GBufferPass
    ScriptableRenderPass <|-- DeferredPass
    ScriptableRenderPass <|-- MainLightShadowCasterPass
    ScriptableRenderPass <|-- AdditionalLightsShadowCasterPass
```

## 4. Forward+ Clustering System

```mermaid
graph TB
    subgraph "Screen Space Tiling"
        ST[Screen Tiles]
        TW[Tile Width: 32px]
        TH[Tile Height: 32px]
        TR[Tile Resolution]
    end
    
    subgraph "Depth Binning"
        ZB[Z-Bins]
        ZS[Z-Bin Scale]
        ZO[Z-Bin Offset]
        BC[Bin Count]
    end
    
    subgraph "Light Culling Jobs"
        LMZJ[Light MinMax Z Job]
        RPMZJ[ReflectionProbe MinMax Z Job]
        ZBJ[Z-Binning Job]
        TJ[Tiling Job]
        TREJ[Tile Range Expansion Job]
    end
    
    subgraph "Data Structures"
        ZBB[Z-Bins Buffer]
        TMB[Tile Masks Buffer]
        LDB[Light Data Buffer]
        LIB[Light Indices Buffer]
    end
    
    subgraph "GPU Resources"
        CB[Constant Buffers]
        SB[Structured Buffers]
        FP[Forward+ Parameters]
    end
    
    ST --> TR
    TW --> TR
    TH --> TR
    
    ZB --> ZS
    ZS --> ZO
    ZO --> BC
    
    TR --> LMZJ
    BC --> LMZJ
    LMZJ --> RPMZJ
    RPMZJ --> ZBJ
    ZBJ --> TJ
    TJ --> TREJ
    
    ZBJ --> ZBB
    TREJ --> TMB
    LMZJ --> LDB
    RPMZJ --> LIB
    
    ZBB --> CB
    TMB --> CB
    LDB --> SB
    LIB --> SB
    CB --> FP
    SB --> FP
```

## 5. Post-Processing System Architecture

```mermaid
classDiagram
    class PostProcessPasses {
        -m_ColorGradingLut: RTHandle
        -m_AfterPostProcessColor: RTHandle
        -m_ColorGradingLutPass: ColorGradingLutPass
        -m_PostProcessPass: PostProcessPass
        -m_FinalPostProcessPass: PostProcessPass
        
        +colorGradingLut: RTHandle
        +colorGradingLutPass: ColorGradingLutPass
        +postProcessPass: PostProcessPass
        +finalPostProcessPass: PostProcessPass
        +isCreated: bool
        
        +Dispose()
        +ReleaseRenderTargets()
    }
    
    class ColorGradingLutPass {
        -m_LutMaterial: Material
        -m_InternalLut: RTHandle
        
        +ConfigureDescriptor(UniversalPostProcessingData, RenderTextureDescriptor, FilterMode)
        +Setup(RTHandle)
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class PostProcessPass {
        -m_Materials: PostProcessMaterials
        -m_Source: RTHandle
        -m_Destination: RTHandle
        -m_Depth: RTHandle
        -m_InternalLut: RTHandle
        -m_MotionVectors: RTHandle
        
        +Setup(RenderTextureDescriptor, RTHandle, bool, RTHandle, RTHandle, RTHandle, bool, bool)
        +SetupFinalPass(RTHandle, bool, bool)
        +Execute(ScriptableRenderContext, RenderingData)
        -DoTemporalAntialiasing(CommandBuffer, RenderTextureDescriptor, RTHandle, RTHandle)
        -DoSpatialTemporalPostProcessing(CommandBuffer, RenderTextureDescriptor, RTHandle, RTHandle)
        -DoScreenSpaceAmbientOcclusion(CommandBuffer, RTHandle, RTHandle)
        -DoDepthOfField(CommandBuffer, RTHandle, RTHandle, RTHandle)
        -DoMotionBlur(CommandBuffer, RTHandle, RTHandle, RTHandle)
        -DoBloom(CommandBuffer, RTHandle, RTHandle)
        -DoColorGrading(CommandBuffer, RTHandle, RTHandle, RTHandle)
        -DoFinalPost(CommandBuffer, RTHandle, RTHandle)
    }
    
    class VolumeManager {
        -m_ComponentsDefaultState: Dictionary~Type, VolumeComponent~
        -m_Volumes: List~Volume~
        -m_SortedVolumes: List~Volume~
        -m_TempColliders: List~Collider~
        
        +stack: VolumeStack
        +instance: VolumeManager
        
        +Update(Transform, LayerMask)
        +GetVolumeProfile(GameObject): VolumeProfile
        +CreateStack(): VolumeStack
    }
    
    class VolumeStack {
        -m_Components: Dictionary~Type, VolumeComponent~
        
        +GetComponent~T~(): T
        +Reload(IEnumerable~VolumeComponent~)
    }
    
    PostProcessPasses --> ColorGradingLutPass
    PostProcessPasses --> PostProcessPass
    PostProcessPass --> VolumeManager
    VolumeManager --> VolumeStack
```

## 6. RTHandle and Resource Management

```mermaid
classDiagram
    class RTHandles {
        <<static>>
        -s_RTHandleSystem: RTHandleSystem
        +rtHandleProperties: RTHandleProperties
        
        +Initialize(int, int)
        +Shutdown()
        +Alloc(RenderTargetIdentifier): RTHandle
        +Alloc(int, int, int, DepthBits, GraphicsFormat): RTHandle
        +Release(RTHandle)
        +ResetReferenceSize(int, int)
        +SetHardwareDynamicResolutionState(bool)
    }
    
    class RTHandle {
        -m_RT: RenderTexture
        -m_NameID: int
        -m_EnableMSAA: bool
        -m_EnableRandomWrite: bool
        -m_ScaleFunc: Vector2
        
        +rt: RenderTexture
        +nameID: int
        +name: string
        +useScaling: bool
        +scaleFactor: Vector2
        
        +GetScaledSize(Vector2Int): Vector2Int
        +Release()
        +SwitchToFastMemory(CommandBuffer, float, FastMemoryFlags)
        +SwitchOutFastMemory(CommandBuffer)
    }
    
    class RenderTargetBufferSystem {
        -m_A: RTHandle
        -m_B: RTHandle
        -m_AliasIds: int[]
        -m_Descriptor: RenderTextureDescriptor
        -m_FilterMode: FilterMode
        -m_AllowMSAA: bool
        
        +PeekBackBuffer(): RTHandle
        +GetBackBuffer(CommandBuffer): RTHandle
        +GetFrontBuffer(CommandBuffer): RTHandle
        +Swap()
        +Clear()
        +SetCameraSettings(RenderTextureDescriptor, FilterMode)
        +EnableMSAA(bool)
        +Dispose()
    }
    
    class RenderingUtils {
        <<static>>
        +ReAllocateHandleIfNeeded(RTHandle, RenderTextureDescriptor, FilterMode, TextureWrapMode, int, string)
        +CreateRenderGraphTexture(RenderGraph, RenderTextureDescriptor, string, bool): TextureHandle
        +SupportsRenderTextureFormat(RenderTextureFormat): bool
        +MultisampleDepthResolveSupported(): bool
        +useStructuredBuffer: bool
    }
    
    RTHandles --> RTHandle
    RenderTargetBufferSystem --> RTHandle
    RenderingUtils --> RTHandle
```

## 7. Shadow System Architecture

```mermaid
graph TB
    subgraph "Shadow Input"
        ML[Main Light]
        AL[Additional Lights]
        SC[Shadow Cascades]
        SA[Shadow Atlas]
    end
    
    subgraph "Shadow Culling"
        SCC[Shadow Caster Culling]
        SRC[Shadow Receiver Culling]
        CSM[Cascade Shadow Maps]
        ASM[Atlas Shadow Maps]
    end
    
    subgraph "Shadow Rendering"
        MLSCP[Main Light Shadow Caster Pass]
        ALSCP[Additional Lights Shadow Caster Pass]
        SMR[Shadow Map Rendering]
        SB[Shadow Bias]
    end
    
    subgraph "Shadow Sampling"
        PCF[Percentage Closer Filtering]
        SS[Soft Shadows]
        CSF[Cascade Shadow Filtering]
        ASF[Atlas Shadow Filtering]
    end
    
    subgraph "Shadow Data"
        SMT[Shadow Map Textures]
        SMM[Shadow Matrices]
        SMP[Shadow Parameters]
        SSC[Shadow Shader Constants]
    end
    
    ML --> SCC
    AL --> SCC
    SC --> SCC
    SA --> SCC
    
    SCC --> SRC
    SRC --> CSM
    SRC --> ASM
    
    CSM --> MLSCP
    ASM --> ALSCP
    MLSCP --> SMR
    ALSCP --> SMR
    SMR --> SB
    
    SB --> PCF
    PCF --> SS
    SS --> CSF
    SS --> ASF
    
    SMR --> SMT
    SMR --> SMM
    SMR --> SMP
    SMR --> SSC
```

## 8. XR/VR Rendering Pipeline

```mermaid
graph TB
    subgraph "XR System"
        XRS[XR System]
        XRD[XR Display]
        XRC[XR Camera]
        XRT[XR Tracking]
    end
    
    subgraph "Stereo Modes"
        SPI[Single Pass Instanced]
        SPM[Single Pass Multiview]
        MP[Multi-Pass]
    end
    
    subgraph "XR Passes"
        XOM[XR Occlusion Mesh Pass]
        XDM[XR Depth Motion Pass]
        XCD[XR Copy Depth Pass]
    end
    
    subgraph "XR Data"
        XP[XR Pass]
        XV[XR View]
        XM[XR Matrices]
        XRT_TEX[XR Render Target]
    end
    
    subgraph "XR Optimizations"
        FR[Foveated Rendering]
        VRS[Variable Rate Shading]
        FFR[Fixed Foveated Rendering]
        LMS[Late Motion Synchronization]
    end
    
    XRS --> XRD
    XRD --> XRC
    XRC --> XRT
    
    XRT --> SPI
    XRT --> SPM
    XRT --> MP
    
    SPI --> XOM
    SPM --> XDM
    MP --> XCD
    
    XOM --> XP
    XDM --> XV
    XCD --> XM
    XP --> XRT_TEX
    
    XV --> FR
    XM --> VRS
    XRT_TEX --> FFR
    FR --> LMS
```

## 9. Shader Variant System

```mermaid
graph TB
    subgraph "Shader Sources"
        SS[Shader Sources]
        SI[Shader Includes]
        SL[Shader Libraries]
        SK[Shader Keywords]
    end
    
    subgraph "Variant Generation"
        KM[Keyword Matrix]
        VG[Variant Generation]
        VS[Variant Stripping]
        VC[Variant Compilation]
    end
    
    subgraph "Platform Variants"
        D3D[DirectX Variants]
        VK[Vulkan Variants]
        GL[OpenGL Variants]
        MT[Metal Variants]
        GLES[OpenGL ES Variants]
    end
    
    subgraph "Feature Variants"
        LV[Lighting Variants]
        SV[Shadow Variants]
        PPV[Post-Process Variants]
        XRV[XR Variants]
    end
    
    subgraph "Runtime Management"
        KS[Keyword State]
        VL[Variant Loading]
        SC[Shader Caching]
        SO[Shader Optimization]
    end
    
    SS --> KM
    SI --> KM
    SL --> KM
    SK --> KM
    
    KM --> VG
    VG --> VS
    VS --> VC
    
    VC --> D3D
    VC --> VK
    VC --> GL
    VC --> MT
    VC --> GLES
    
    VC --> LV
    VC --> SV
    VC --> PPV
    VC --> XRV
    
    D3D --> KS
    VK --> KS
    GL --> KS
    MT --> KS
    GLES --> KS
    
    KS --> VL
    VL --> SC
    SC --> SO
```

## 10. Performance Profiling and Debug System

```mermaid
graph TB
    subgraph "Debug Input"
        DH[Debug Handler]
        DS[Debug Settings]
        DM[Debug Modes]
        DP[Debug Parameters]
    end
    
    subgraph "Profiling Data"
        PT[Performance Timers]
        MC[Memory Counters]
        DC[Draw Call Counters]
        GS[GPU Statistics]
    end
    
    subgraph "Visualization"
        DV[Debug Visualization]
        GB[G-Buffer Debug]
        SM[Shadow Map Debug]
        LC[Light Complexity]
        OD[Overdraw Debug]
    end
    
    subgraph "Performance Analysis"
        FPS[Frame Rate Analysis]
        GT[GPU Timing]
        CT[CPU Timing]
        MA[Memory Analysis]
    end
    
    subgraph "Debug Output"
        DO[Debug Overlay]
        DT[Debug Textures]
        DR[Debug Render Targets]
        DL[Debug Logs]
    end
    
    DH --> DS
    DS --> DM
    DM --> DP
    
    DP --> PT
    DP --> MC
    DP --> DC
    DP --> GS
    
    PT --> DV
    MC --> GB
    DC --> SM
    GS --> LC
    LC --> OD
    
    DV --> FPS
    GB --> GT
    SM --> CT
    OD --> MA
    
    FPS --> DO
    GT --> DT
    CT --> DR
    MA --> DL
```

These detailed component diagrams provide an in-depth view of Unity URP's internal architecture, showing the relationships between classes, data flow patterns, and system interactions at a granular level. Each diagram focuses on specific subsystems while maintaining connections to the broader architecture.