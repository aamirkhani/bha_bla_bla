# Unity URP - Execution Flow Diagrams

## 1. Complete Frame Rendering Flow

```mermaid
sequenceDiagram
    participant Unity as Unity Engine
    participant URP as UniversalRenderPipeline
    participant UR as UniversalRenderer
    participant FL as ForwardLights
    participant DL as DeferredLights
    participant PP as PostProcessPasses
    participant GPU as GPU
    
    Unity->>URP: Render(ScriptableRenderContext, Camera[])
    
    loop For each camera
        URP->>URP: InitializeCameraData(camera)
        URP->>URP: InitializeLightData(visibleLights)
        URP->>URP: InitializeShadowData(lightData)
        URP->>URP: InitializePostProcessingData(camera)
        
        URP->>UR: Setup(context, renderingData)
        
        alt Forward/Forward+ Mode
            UR->>FL: PreSetup(renderingData, cameraData, lightData)
            UR->>FL: Setup(context, renderingData)
        else Deferred/Deferred+ Mode
            UR->>DL: SetupLights(cmd, cameraData, lightData)
            UR->>DL: ResolveMixedLightingMode(lightData)
        end
        
        UR->>UR: EnqueueShadowPasses()
        UR->>UR: EnqueueDepthPasses()
        UR->>UR: EnqueueGeometryPasses()
        UR->>UR: EnqueueLightingPasses()
        UR->>UR: EnqueuePostProcessPasses()
        
        URP->>UR: Execute(context, renderingData)
        
        loop For each render pass
            UR->>GPU: Execute render pass
            GPU-->>UR: Pass completed
        end
        
        UR->>PP: Execute post-processing
        PP-->>UR: Post-processing completed
        
        UR-->>URP: Camera rendering completed
    end
    
    URP-->>Unity: Frame rendering completed
```

## 2. Forward Rendering Mode Flow

```mermaid
flowchart TD
    Start([Start Forward Rendering]) --> Setup[Setup Camera & Lights]
    Setup --> ShadowSetup{Shadow Setup Required?}
    
    ShadowSetup -->|Yes| MainShadow[Main Light Shadow Pass]
    ShadowSetup -->|No| DepthCheck{Depth Prepass Required?}
    MainShadow --> AddShadow[Additional Lights Shadow Pass]
    AddShadow --> DepthCheck
    
    DepthCheck -->|Yes| DepthPrepass[Depth Only Pass]
    DepthCheck -->|No| LightSetup[Setup Forward Lights]
    DepthPrepass --> DepthPriming{Depth Priming Enabled?}
    
    DepthPriming -->|Yes| PrimedCopy[Primed Depth Copy Pass]
    DepthPriming -->|No| LightSetup
    PrimedCopy --> LightSetup
    
    LightSetup --> OpaquePass[Render Opaque Objects]
    OpaquePass --> SkyboxCheck{Skybox Required?}
    
    SkyboxCheck -->|Yes| SkyboxPass[Draw Skybox Pass]
    SkyboxCheck -->|No| CopyCheck{Copy Depth/Color Required?}
    SkyboxPass --> CopyCheck
    
    CopyCheck -->|Depth| CopyDepth[Copy Depth Pass]
    CopyCheck -->|Color| CopyColor[Copy Color Pass]
    CopyCheck -->|None| TransparentPass[Render Transparent Objects]
    CopyDepth --> TransparentPass
    CopyColor --> TransparentPass
    
    TransparentPass --> PostProcess{Post-Processing Enabled?}
    PostProcess -->|Yes| PPPass[Post-Process Pass]
    PostProcess -->|No| FinalBlit[Final Blit Pass]
    PPPass --> FinalBlit
    
    FinalBlit --> End([End Forward Rendering])
```

## 3. Deferred Rendering Mode Flow

```mermaid
flowchart TD
    Start([Start Deferred Rendering]) --> Setup[Setup Camera & G-Buffer]
    Setup --> ShadowSetup{Shadow Setup Required?}
    
    ShadowSetup -->|Yes| MainShadow[Main Light Shadow Pass]
    ShadowSetup -->|No| GBufferSetup[Setup G-Buffer Targets]
    MainShadow --> AddShadow[Additional Lights Shadow Pass]
    AddShadow --> GBufferSetup
    
    GBufferSetup --> DepthNormal{Depth-Normal Prepass?}
    DepthNormal -->|Yes| DepthNormalPass[Depth-Normal Prepass]
    DepthNormal -->|No| GBufferPass[G-Buffer Pass]
    DepthNormalPass --> GBufferPass
    
    GBufferPass --> CopyGDepth[Copy G-Buffer Depth]
    CopyGDepth --> DeferredLighting[Deferred Lighting Pass]
    
    DeferredLighting --> StencilLights{Stencil or Clustered?}
    StencilLights -->|Stencil| DirectionalLights[Render Directional Lights]
    StencilLights -->|Clustered| ClusteredLights[Render Clustered Lights]
    
    DirectionalLights --> PointLights[Render Point Lights]
    PointLights --> SpotLights[Render Spot Lights]
    SpotLights --> ForwardOnly[Forward-Only Pass]
    
    ClusteredLights --> ForwardOnly
    
    ForwardOnly --> SkyboxCheck{Skybox Required?}
    SkyboxCheck -->|Yes| SkyboxPass[Draw Skybox Pass]
    SkyboxCheck -->|No| TransparentPass[Render Transparent Objects]
    SkyboxPass --> TransparentPass
    
    TransparentPass --> PostProcess{Post-Processing Enabled?}
    PostProcess -->|Yes| PPPass[Post-Process Pass]
    PostProcess -->|No| FinalBlit[Final Blit Pass]
    PPPass --> FinalBlit
    
    FinalBlit --> End([End Deferred Rendering])
```

## 4. Forward+ Clustering Setup Flow

```mermaid
sequenceDiagram
    participant FL as ForwardLights
    participant JS as Job System
    participant LMZ as LightMinMaxZJob
    RPM as ReflectionProbeMinMaxZJob
    participant ZB as ZBinningJob
    participant TJ as TilingJob
    participant TRE as TileRangeExpansionJob
    participant GPU as GPU Buffers
    
    FL->>FL: PreSetup(renderingData, cameraData, lightData)
    FL->>FL: Calculate tile resolution and bin count
    FL->>FL: Setup screen space parameters
    
    FL->>JS: Schedule LightMinMaxZJob
    JS->>LMZ: Execute parallel light bounds calculation
    LMZ-->>JS: Light min/max Z values
    
    FL->>JS: Schedule ReflectionProbeMinMaxZJob
    JS->>RPM: Execute parallel probe bounds calculation
    RPM-->>JS: Probe min/max Z values
    
    FL->>JS: Schedule ZBinningJob
    JS->>ZB: Execute parallel Z-binning
    ZB-->>JS: Z-bin assignments
    
    FL->>JS: Schedule TilingJob
    JS->>TJ: Execute parallel tile range calculation
    TJ-->>JS: Tile ranges per item
    
    FL->>JS: Schedule TileRangeExpansionJob
    JS->>TRE: Execute parallel tile mask generation
    TRE-->>JS: Final tile masks
    
    FL->>FL: Complete all jobs
    FL->>GPU: Upload Z-bins buffer
    FL->>GPU: Upload tile masks buffer
    FL->>GPU: Set Forward+ parameters
```

## 5. Shadow Rendering Flow

```mermaid
flowchart TD
    Start([Start Shadow Rendering]) --> LightCull[Cull Shadow Casting Lights]
    LightCull --> MainLightCheck{Main Light Shadows?}
    
    MainLightCheck -->|Yes| MainLightSetup[Setup Main Light Shadows]
    MainLightCheck -->|No| AddLightCheck{Additional Light Shadows?}
    
    MainLightSetup --> CascadeSetup[Setup Shadow Cascades]
    CascadeSetup --> CascadeLoop{For Each Cascade}
    
    CascadeLoop --> CascadeCull[Cull Shadow Casters for Cascade]
    CascadeCull --> CascadeRender[Render Shadow Map for Cascade]
    CascadeRender --> CascadeNext{More Cascades?}
    CascadeNext -->|Yes| CascadeLoop
    CascadeNext -->|No| MainLightConstants[Setup Main Light Shadow Constants]
    
    MainLightConstants --> AddLightCheck
    
    AddLightCheck -->|Yes| AddLightSetup[Setup Additional Light Shadows]
    AddLightCheck -->|No| ShadowEnd([End Shadow Rendering])
    
    AddLightSetup --> AtlasSetup[Setup Shadow Atlas Layout]
    AtlasSetup --> AtlasLoop{For Each Shadow Light}
    
    AtlasLoop --> AtlasCull[Cull Shadow Casters for Light]
    AtlasCull --> AtlasRender[Render to Atlas Tile]
    AtlasRender --> AtlasNext{More Shadow Lights?}
    AtlasNext -->|Yes| AtlasLoop
    AtlasNext -->|No| AddLightConstants[Setup Additional Light Shadow Constants]
    
    AddLightConstants --> ShadowEnd
```

## 6. Post-Processing Execution Flow

```mermaid
sequenceDiagram
    participant UR as UniversalRenderer
    participant PP as PostProcessPasses
    participant CGL as ColorGradingLutPass
    participant PPP as PostProcessPass
    participant FPP as FinalPostProcessPass
    participant VM as VolumeManager
    participant GPU as GPU
    
    UR->>PP: Setup post-processing
    PP->>VM: Update volume stack
    VM-->>PP: Volume parameters
    
    alt Color Grading LUT Required
        UR->>CGL: Setup(colorGradingLut)
        UR->>CGL: Execute(context, renderingData)
        CGL->>GPU: Generate LUT texture
        GPU-->>CGL: LUT ready
    end
    
    UR->>PPP: Setup(descriptor, source, destination, depth, lut, motionVectors)
    UR->>PPP: Execute(context, renderingData)
    
    PPP->>PPP: Check TAA enabled
    alt TAA Enabled
        PPP->>GPU: Execute Temporal Anti-Aliasing
        GPU-->>PPP: TAA result
    end
    
    PPP->>PPP: Check STP enabled
    alt STP Enabled
        PPP->>GPU: Execute Spatial Temporal Post-processing
        GPU-->>PPP: STP result
    end
    
    PPP->>PPP: Execute effect chain
    loop For each effect
        PPP->>GPU: Execute post-process effect
        GPU-->>PPP: Effect result
    end
    
    alt Final Post-Processing Required
        UR->>FPP: SetupFinalPass(source, resolveToScreen, needsColorEncoding)
        UR->>FPP: Execute(context, renderingData)
        
        FPP->>FPP: Check FXAA enabled
        alt FXAA Enabled
            FPP->>GPU: Execute FXAA
            GPU-->>FPP: FXAA result
        end
        
        FPP->>FPP: Check upscaling required
        alt Upscaling Required
            FPP->>GPU: Execute upscaling filter
            GPU-->>FPP: Upscaled result
        end
    end
```

## 7. XR/VR Rendering Flow

```mermaid
flowchart TD
    Start([Start XR Rendering]) --> XRCheck{XR Enabled?}
    XRCheck -->|No| RegularFlow[Regular Rendering Flow]
    XRCheck -->|Yes| XRSetup[Setup XR Parameters]
    
    XRSetup --> StereoMode{Stereo Mode?}
    StereoMode -->|Single Pass Instanced| SPISetup[Setup SPI Rendering]
    StereoMode -->|Multi-Pass| MPSetup[Setup Multi-Pass Rendering]
    StereoMode -->|Single Pass Multiview| SPMSetup[Setup SPM Rendering]
    
    SPISetup --> XROcclusion{Occlusion Mesh?}
    MPSetup --> XROcclusion
    SPMSetup --> XROcclusion
    
    XROcclusion -->|Yes| OcclusionPass[XR Occlusion Mesh Pass]
    XROcclusion -->|No| XRDepthMotion{Depth Motion?}
    OcclusionPass --> XRDepthMotion
    
    XRDepthMotion -->|Yes| DepthMotionPass[XR Depth Motion Pass]
    XRDepthMotion -->|No| XRRender[XR Render Loop]
    DepthMotionPass --> XRRender
    
    XRRender --> ViewLoop{For Each Eye/View}
    ViewLoop --> SetupView[Setup View Matrices]
    SetupView --> RenderView[Render View Content]
    RenderView --> NextView{More Views?}
    NextView -->|Yes| ViewLoop
    NextView -->|No| XRCopyDepth{Copy Depth Required?}
    
    XRCopyDepth -->|Yes| CopyDepthPass[XR Copy Depth Pass]
    XRCopyDepth -->|No| XREnd([End XR Rendering])
    CopyDepthPass --> XREnd
    
    RegularFlow --> RegularEnd([End Regular Rendering])
```

## 8. Resource Allocation and Management Flow

```mermaid
sequenceDiagram
    participant UR as UniversalRenderer
    participant RTH as RTHandles
    participant RTBS as RenderTargetBufferSystem
    participant GPU as GPU Memory
    
    UR->>UR: Setup(context, renderingData)
    UR->>UR: Determine render target requirements
    
    alt Intermediate Color Texture Required
        UR->>RTBS: GetBackBuffer(cmd)
        RTBS->>RTH: Alloc color buffer
        RTH->>GPU: Allocate GPU memory
        GPU-->>RTH: Memory allocated
        RTH-->>RTBS: RTHandle created
        RTBS-->>UR: Color buffer ready
    end
    
    alt Depth Texture Required
        UR->>RTH: ReAllocateHandleIfNeeded(depthTexture, descriptor)
        RTH->>GPU: Check existing allocation
        alt Reallocation needed
            RTH->>GPU: Release old memory
            RTH->>GPU: Allocate new memory
            GPU-->>RTH: New memory allocated
        end
        RTH-->>UR: Depth texture ready
    end
    
    alt Normal Texture Required
        UR->>RTH: ReAllocateHandleIfNeeded(normalTexture, descriptor)
        RTH->>GPU: Allocate normal buffer
        GPU-->>RTH: Normal buffer allocated
        RTH-->>UR: Normal texture ready
    end
    
    alt Motion Vector Texture Required
        UR->>RTH: ReAllocateHandleIfNeeded(motionVectorTexture, descriptor)
        RTH->>GPU: Allocate motion vector buffer
        GPU-->>RTH: Motion vector buffer allocated
        RTH-->>UR: Motion vector texture ready
    end
    
    UR->>UR: Execute render passes
    
    UR->>UR: FinishRendering(cmd)
    UR->>RTBS: Clear()
    UR->>RTH: Release temporary resources
    RTH->>GPU: Free temporary memory
```

## 9. Debug and Profiling Flow

```mermaid
flowchart TD
    Start([Start Debug/Profiling]) --> DebugCheck{Debug Handler Active?}
    DebugCheck -->|No| NormalRender[Normal Rendering]
    DebugCheck -->|Yes| DebugSetup[Setup Debug Parameters]
    
    DebugSetup --> DebugMode{Debug Mode?}
    DebugMode -->|Fullscreen Debug| FullscreenDebug[Setup Fullscreen Debug]
    DebugMode -->|HDR Debug| HDRDebug[Setup HDR Debug View]
    DebugMode -->|Lighting Debug| LightingDebug[Setup Lighting Debug]
    DebugMode -->|Material Debug| MaterialDebug[Setup Material Debug]
    
    FullscreenDebug --> DebugTexture{Debug Texture Type?}
    DebugTexture -->|Depth| DepthDebug[Setup Depth Visualization]
    DebugTexture -->|Motion Vector| MotionDebug[Setup Motion Vector Visualization]
    DebugTexture -->|Shadow Map| ShadowDebug[Setup Shadow Map Visualization]
    DebugTexture -->|G-Buffer| GBufferDebug[Setup G-Buffer Visualization]
    
    HDRDebug --> HDRPass[HDR Debug View Pass]
    LightingDebug --> LightComplexity[Light Complexity Visualization]
    MaterialDebug --> MaterialViz[Material Property Visualization]
    
    DepthDebug --> DebugRender[Render with Debug Overlay]
    MotionDebug --> DebugRender
    ShadowDebug --> DebugRender
    GBufferDebug --> DebugRender
    HDRPass --> DebugRender
    LightComplexity --> DebugRender
    MaterialViz --> DebugRender
    
    DebugRender --> ProfilingData{Collect Profiling Data?}
    ProfilingData -->|Yes| CollectStats[Collect Performance Statistics]
    ProfilingData -->|No| DebugEnd([End Debug Rendering])
    
    CollectStats --> GPUTiming[Collect GPU Timing]
    GPUTiming --> CPUTiming[Collect CPU Timing]
    CPUTiming --> MemoryStats[Collect Memory Statistics]
    MemoryStats --> DrawCallStats[Collect Draw Call Statistics]
    DrawCallStats --> DebugEnd
    
    NormalRender --> NormalEnd([End Normal Rendering])
```

## 10. Camera Stacking Flow

```mermaid
sequenceDiagram
    participant Unity as Unity Engine
    participant URP as UniversalRenderPipeline
    participant BaseR as Base Renderer
    participant OverlayR as Overlay Renderer
    participant GPU as GPU
    
    Unity->>URP: Render camera stack
    URP->>URP: Sort cameras by depth
    URP->>URP: Identify base and overlay cameras
    
    loop For each base camera
        URP->>BaseR: Setup base camera
        BaseR->>BaseR: Create intermediate textures
        BaseR->>BaseR: Setup color/depth targets
        BaseR->>GPU: Render base camera content
        GPU-->>BaseR: Base content rendered
        
        loop For each overlay camera in stack
            URP->>OverlayR: Setup overlay camera
            OverlayR->>BaseR: Share color/depth buffers
            BaseR-->>OverlayR: Buffers shared
            OverlayR->>GPU: Render overlay content
            GPU-->>OverlayR: Overlay content rendered
        end
        
        BaseR->>BaseR: Apply post-processing to stack
        BaseR->>GPU: Final blit to target
        GPU-->>BaseR: Stack rendering complete
    end
    
    URP-->>Unity: All camera stacks rendered
```

These execution flow diagrams provide detailed insights into how Unity URP processes rendering operations, from high-level frame orchestration down to specific subsystem execution patterns. Each diagram shows the temporal relationships and decision points that drive the rendering pipeline's behavior.