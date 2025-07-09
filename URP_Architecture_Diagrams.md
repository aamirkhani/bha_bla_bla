# Unity Universal Render Pipeline (URP) - Architecture Diagrams

## 1. Overall URP Architecture Diagram

```mermaid
graph TB
    subgraph "Unity Engine Core"
        UC[Unity Camera System]
        URP[UniversalRenderPipeline]
        SRC[ScriptableRenderContext]
    end
    
    subgraph "URP Core Components"
        UR[UniversalRenderer]
        FL[ForwardLights]
        DL[DeferredLights]
        PP[PostProcessPasses]
    end
    
    subgraph "Render Passes"
        SP[Shadow Passes]
        DP[Depth Passes]
        GP[Geometry Passes]
        LP[Lighting Passes]
        PPP[Post-Process Passes]
    end
    
    subgraph "Data Containers"
        UCD[UniversalCameraData]
        ULD[UniversalLightData]
        USD[UniversalShadowData]
        URD[UniversalRenderingData]
    end
    
    subgraph "Resource Management"
        RTH[RTHandle System]
        RTS[RenderTargetBufferSystem]
        CB[CommandBuffer Pool]
    end
    
    UC --> URP
    URP --> SRC
    SRC --> UR
    UR --> FL
    UR --> DL
    UR --> PP
    
    UR --> SP
    UR --> DP
    UR --> GP
    UR --> LP
    UR --> PPP
    
    URP --> UCD
    URP --> ULD
    URP --> USD
    URP --> URD
    
    UR --> RTH
    UR --> RTS
    UR --> CB
```

## 2. URP Class Hierarchy Diagram

```mermaid
classDiagram
    class RenderPipeline {
        <<abstract>>
        +Render(ScriptableRenderContext, Camera[])
        +ProcessRenderRequests(ScriptableRenderContext, Camera, RenderRequest[])
    }
    
    class UniversalRenderPipeline {
        -m_Renderer: ScriptableRenderer
        -m_AdditionalCameraData: UniversalAdditionalCameraData
        +Render(ScriptableRenderContext, Camera[])
        +RenderSingleCamera(ScriptableRenderContext, Camera)
        +SetupPerFrameShaderConstants()
        +InitializeCameraData(Camera, UniversalAdditionalCameraData)
        +InitializeLightData(VisibleLight[])
        +InitializeShadowData(UniversalLightData)
    }
    
    class ScriptableRenderer {
        <<abstract>>
        #activeRenderPassQueue: List~ScriptableRenderPass~
        #frameData: ContextContainer
        +Setup(ScriptableRenderContext, RenderingData)
        +Execute(ScriptableRenderContext, RenderingData)
        +EnqueuePass(ScriptableRenderPass)
        +SetupCullingParameters(ScriptableCullingParameters, CameraData)
    }
    
    class UniversalRenderer {
        -m_RenderingMode: RenderingMode
        -m_ForwardLights: ForwardLights
        -m_DeferredLights: DeferredLights
        -m_PostProcessPasses: PostProcessPasses
        -m_ColorBufferSystem: RenderTargetBufferSystem
        +Setup(ScriptableRenderContext, RenderingData)
        +SetupLights(ScriptableRenderContext, RenderingData)
        +CreateCameraRenderTarget(ScriptableRenderContext, RenderTextureDescriptor)
    }
    
    class ScriptableRenderPass {
        <<abstract>>
        +renderPassEvent: RenderPassEvent
        +colorAttachmentHandles: RTHandle[]
        +depthAttachmentHandle: RTHandle
        +Configure(CommandBuffer, RenderTextureDescriptor)
        +Execute(ScriptableRenderContext, RenderingData)
        +OnCameraCleanup(CommandBuffer)
    }
    
    class DrawObjectsPass {
        -m_FilteringSettings: FilteringSettings
        -m_RenderStateBlock: RenderStateBlock
        -m_ShaderTagIds: ShaderTagId[]
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class DeferredPass {
        -m_DeferredLights: DeferredLights
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    class PostProcessPass {
        -m_Materials: Material[]
        -m_Descriptors: RenderTextureDescriptor[]
        +Execute(ScriptableRenderContext, RenderingData)
    }
    
    RenderPipeline <|-- UniversalRenderPipeline
    ScriptableRenderer <|-- UniversalRenderer
    ScriptableRenderPass <|-- DrawObjectsPass
    ScriptableRenderPass <|-- DeferredPass
    ScriptableRenderPass <|-- PostProcessPass
    UniversalRenderPipeline --> UniversalRenderer
    UniversalRenderer --> ScriptableRenderPass
```

## 3. Data Flow Architecture

```mermaid
graph LR
    subgraph "Input Data"
        C[Camera]
        L[Lights]
        R[Renderers]
        V[Volumes]
    end
    
    subgraph "Data Processing"
        CC[Camera Culling]
        LC[Light Culling]
        SC[Shadow Culling]
        VC[Volume Culling]
    end
    
    subgraph "Data Containers"
        UCD[UniversalCameraData]
        ULD[UniversalLightData]
        USD[UniversalShadowData]
        UPD[UniversalPostProcessingData]
    end
    
    subgraph "Render Execution"
        RP[Render Passes]
        CB[Command Buffers]
        GPU[GPU Execution]
    end
    
    C --> CC
    L --> LC
    R --> CC
    V --> VC
    
    CC --> UCD
    LC --> ULD
    SC --> USD
    VC --> UPD
    
    UCD --> RP
    ULD --> RP
    USD --> RP
    UPD --> RP
    
    RP --> CB
    CB --> GPU
```

## 4. Rendering Mode Architecture

```mermaid
graph TB
    subgraph "Rendering Modes"
        FM[Forward Mode]
        FPM[Forward+ Mode]
        DM[Deferred Mode]
        DPM[Deferred+ Mode]
    end
    
    subgraph "Forward Rendering Path"
        FLC[Forward Light Culling]
        FOG[Forward Opaque Geometry]
        FTG[Forward Transparent Geometry]
    end
    
    subgraph "Forward+ Rendering Path"
        TLC[Tiled Light Culling]
        CLB[Clustered Light Buffers]
        FPOG[Forward+ Opaque Geometry]
        FPTG[Forward+ Transparent Geometry]
    end
    
    subgraph "Deferred Rendering Path"
        GB[G-Buffer Pass]
        DLP[Deferred Lighting Pass]
        FOF[Forward-Only Pass]
        DTG[Deferred Transparent Geometry]
    end
    
    subgraph "Deferred+ Rendering Path"
        DGB[Deferred+ G-Buffer]
        DCLP[Deferred+ Clustered Lighting]
        DPFOF[Deferred+ Forward-Only]
        DPTG[Deferred+ Transparent]
    end
    
    FM --> FLC
    FLC --> FOG
    FOG --> FTG
    
    FPM --> TLC
    TLC --> CLB
    CLB --> FPOG
    FPOG --> FPTG
    
    DM --> GB
    GB --> DLP
    DLP --> FOF
    FOF --> DTG
    
    DPM --> DGB
    DGB --> DCLP
    DCLP --> DPFOF
    DPFOF --> DPTG
```

## 5. Forward Lighting System Diagram

```mermaid
graph TB
    subgraph "Light Input"
        ML[Main Light]
        AL[Additional Lights]
        RP[Reflection Probes]
        LC[Light Cookies]
    end
    
    subgraph "Light Processing"
        LI[Light Initialization]
        LCull[Light Culling]
        LShadow[Light Shadow Setup]
        LData[Light Data Preparation]
    end
    
    subgraph "Forward Rendering"
        POL[Per-Object Lighting]
        SB[Structured Buffers]
        UA[Uniform Arrays]
        LS[Light Sampling]
    end
    
    subgraph "Forward+ Clustering"
        TS[Tile Setup]
        ZB[Z-Binning]
        TC[Tile Culling]
        LM[Light Masks]
    end
    
    ML --> LI
    AL --> LI
    RP --> LI
    LC --> LI
    
    LI --> LCull
    LCull --> LShadow
    LShadow --> LData
    
    LData --> POL
    POL --> SB
    POL --> UA
    SB --> LS
    UA --> LS
    
    LData --> TS
    TS --> ZB
    ZB --> TC
    TC --> LM
    LM --> LS
```

## 6. Deferred Rendering G-Buffer Layout

```mermaid
graph TB
    subgraph "G-Buffer Layout"
        GB0["G-Buffer 0<br/>Albedo (RGB) + Material Flags (A)"]
        GB1["G-Buffer 1<br/>Specular/Metallic (RGB) + Unused (A)"]
        GB2["G-Buffer 2<br/>World Normal (RGB) + Smoothness (A)"]
        GB3["G-Buffer 3<br/>Emission + Baked Lighting (RGB)"]
        GB4["G-Buffer 4 (Optional)<br/>Depth for Mobile Render Pass"]
        GB5["G-Buffer 5 (Optional)<br/>Shadow Mask for Mixed Lighting"]
        GB6["G-Buffer 6 (Optional)<br/>Rendering Layers"]
    end
    
    subgraph "Deferred Lighting"
        DL[Directional Lights]
        PL[Point Lights]
        SL[Spot Lights]
        AL[Area Lights]
    end
    
    subgraph "Light Volume Rendering"
        FS[Fullscreen Triangle]
        SP[Sphere Mesh]
        CO[Cone Mesh]
        SV[Stencil Volumes]
    end
    
    GB0 --> DL
    GB1 --> DL
    GB2 --> DL
    GB3 --> DL
    
    DL --> FS
    PL --> SP
    SL --> CO
    
    SP --> SV
    CO --> SV
    SV --> AL
```

## 7. Render Pass Execution Flow

```mermaid
sequenceDiagram
    participant URP as UniversalRenderPipeline
    participant UR as UniversalRenderer
    participant SP as Shadow Passes
    participant DP as Depth Pass
    participant GP as Geometry Pass
    participant LP as Lighting Pass
    participant PP as Post Process
    participant FB as Final Blit
    
    URP->>UR: Setup(context, renderingData)
    UR->>SP: EnqueuePass(shadowPass)
    UR->>DP: EnqueuePass(depthPass)
    UR->>GP: EnqueuePass(geometryPass)
    UR->>LP: EnqueuePass(lightingPass)
    UR->>PP: EnqueuePass(postProcessPass)
    UR->>FB: EnqueuePass(finalBlitPass)
    
    URP->>UR: Execute(context, renderingData)
    UR->>SP: Execute(context, renderingData)
    SP-->>UR: Shadow Maps Generated
    UR->>DP: Execute(context, renderingData)
    DP-->>UR: Depth Buffer Ready
    UR->>GP: Execute(context, renderingData)
    GP-->>UR: Geometry Rendered
    UR->>LP: Execute(context, renderingData)
    LP-->>UR: Lighting Applied
    UR->>PP: Execute(context, renderingData)
    PP-->>UR: Post Effects Applied
    UR->>FB: Execute(context, renderingData)
    FB-->>UR: Final Image Ready
```

## 8. Resource Management System

```mermaid
graph TB
    subgraph "RTHandle System"
        RTH[RTHandle Manager]
        RTA[RTHandle Allocation]
        RTS[RTHandle Scaling]
        RTR[RTHandle Reuse]
    end
    
    subgraph "Render Target Buffer System"
        RTBS[RenderTargetBufferSystem]
        BB[Back Buffer]
        FB[Front Buffer]
        BS[Buffer Swapping]
    end
    
    subgraph "Command Buffer Management"
        CBP[CommandBuffer Pool]
        CBC[CommandBuffer Creation]
        CBR[CommandBuffer Reuse]
        CBE[CommandBuffer Execution]
    end
    
    subgraph "Memory Optimization"
        DP[Dynamic Resolution]
        MSAA[MSAA Handling]
        TC[Texture Compression]
        GC[Garbage Collection]
    end
    
    RTH --> RTA
    RTA --> RTS
    RTS --> RTR
    
    RTBS --> BB
    RTBS --> FB
    BB --> BS
    FB --> BS
    
    CBP --> CBC
    CBC --> CBR
    CBR --> CBE
    
    RTH --> DP
    RTH --> MSAA
    RTBS --> TC
    CBP --> GC
```

## 9. Post-Processing Pipeline

```mermaid
graph LR
    subgraph "Input"
        CC[Camera Color]
        CD[Camera Depth]
        MV[Motion Vectors]
    end
    
    subgraph "Volume System"
        VS[Volume Stack]
        VB[Volume Blending]
        VP[Volume Parameters]
    end
    
    subgraph "Post-Process Effects"
        TAA[Temporal AA]
        SSAO[Screen Space AO]
        DOF[Depth of Field]
        MB[Motion Blur]
        B[Bloom]
        CG[Color Grading]
        TM[Tone Mapping]
    end
    
    subgraph "Final Output"
        FXAA[Fast AA]
        US[Upscaling]
        HDR[HDR Output]
        FB[Final Blit]
    end
    
    CC --> TAA
    CD --> TAA
    MV --> TAA
    
    VS --> VB
    VB --> VP
    VP --> SSAO
    VP --> DOF
    VP --> MB
    VP --> B
    VP --> CG
    VP --> TM
    
    TAA --> SSAO
    SSAO --> DOF
    DOF --> MB
    MB --> B
    B --> CG
    CG --> TM
    
    TM --> FXAA
    FXAA --> US
    US --> HDR
    HDR --> FB
```

## 10. XR/VR Rendering Architecture

```mermaid
graph TB
    subgraph "XR Input"
        XRC[XR Camera]
        XRD[XR Display]
        XRS[XR Settings]
    end
    
    subgraph "Stereo Rendering Modes"
        SPI[Single Pass Instanced]
        MP[Multi-Pass]
        SPM[Single Pass Multiview]
    end
    
    subgraph "XR Passes"
        XOM[XR Occlusion Mesh]
        XDM[XR Depth Motion]
        XCD[XR Copy Depth]
    end
    
    subgraph "XR Optimizations"
        FR[Foveated Rendering]
        VRS[Variable Rate Shading]
        FFR[Fixed Foveated Rendering]
    end
    
    XRC --> SPI
    XRC --> MP
    XRC --> SPM
    
    XRD --> XOM
    XRS --> XDM
    XRS --> XCD
    
    SPI --> FR
    MP --> VRS
    SPM --> FFR
```

## 11. Shader Integration Architecture

```mermaid
graph TB
    subgraph "Shader Libraries"
        CL[Core Library]
        LL[Lighting Library]
        SL[Shadow Library]
        PL[Post-Process Library]
    end
    
    subgraph "Shader Variants"
        KM[Keyword Management]
        VS[Variant Stripping]
        CC[Conditional Compilation]
        PA[Platform Abstraction]
    end
    
    subgraph "Shader Passes"
        SP[Shadow Pass]
        DP[Depth Pass]
        FP[Forward Pass]
        GP[GBuffer Pass]
        LP[Lighting Pass]
        PP[Post-Process Pass]
    end
    
    subgraph "Platform Support"
        D3D[DirectX]
        VK[Vulkan]
        GL[OpenGL]
        MT[Metal]
        GLES[OpenGL ES]
    end
    
    CL --> KM
    LL --> KM
    SL --> KM
    PL --> KM
    
    KM --> VS
    VS --> CC
    CC --> PA
    
    PA --> SP
    PA --> DP
    PA --> FP
    PA --> GP
    PA --> LP
    PA --> PP
    
    PA --> D3D
    PA --> VK
    PA --> GL
    PA --> MT
    PA --> GLES
```

## 12. Performance Optimization Systems

```mermaid
graph TB
    subgraph "CPU Optimizations"
        SB[SRP Batcher]
        GDR[GPU Driven Rendering]
        JS[Job System]
        MT[Multi-Threading]
    end
    
    subgraph "GPU Optimizations"
        DP[Depth Priming]
        EZT[Early Z Testing]
        NRP[Native Render Pass]
        TBR[Tile-Based Rendering]
    end
    
    subgraph "Memory Optimizations"
        DR[Dynamic Resolution]
        TC[Texture Compression]
        LOD[Level of Detail]
        OC[Occlusion Culling]
    end
    
    subgraph "Platform Specific"
        MO[Mobile Optimizations]
        CO[Console Optimizations]
        PCO[PC Optimizations]
        VRO[VR Optimizations]
    end
    
    SB --> GDR
    GDR --> JS
    JS --> MT
    
    DP --> EZT
    EZT --> NRP
    NRP --> TBR
    
    DR --> TC
    TC --> LOD
    LOD --> OC
    
    MT --> MO
    TBR --> MO
    GDR --> CO
    NRP --> CO
    SB --> PCO
    EZT --> PCO
    DR --> VRO
    NRP --> VRO
```

These diagrams provide a comprehensive visual representation of Unity's URP architecture, showing the relationships between components, data flow, and execution patterns. Each diagram focuses on a specific aspect of the pipeline while maintaining connections to the overall system architecture.