# Unity URP Resource and Memory Diagrams

## 1. Shader Variant System Architecture

```mermaid
graph TB
    subgraph "Shader Compilation System"
        SC[Shader Compiler]
        SV[Shader Variants]
        SK[Shader Keywords]
        SDB[Shader Database]
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
        LIGHTMAP[LIGHTMAP_ON]
        DIRLIGHTMAP[DIRLIGHTMAP_COMBINED]
    end
    
    subgraph "Platform Variants"
        INST[INSTANCING_ON]
        DOTS[DOTS_INSTANCING_ON]
        XR[XR_RENDERING]
        FP[_FORWARD_PLUS]
        CLUSTERED[_CLUSTERED_RENDERING]
    end
    
    SC --> SV
    SV --> SK
    SK --> SDB
    
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
    SK --> LIGHTMAP
    SK --> DIRLIGHTMAP
    
    SK --> INST
    SK --> DOTS
    SK --> XR
    SK --> FP
    SK --> CLUSTERED
    
    subgraph "Runtime Selection"
        MS[Material Settings]
        QS[Quality Settings]
        PS[Platform Settings]
        CS[Camera Settings]
        LS_SET[Lighting Settings]
    end
    
    MS --> ST
    MS --> AT
    MS --> AP
    MS --> AM
    
    QS --> SS
    QS --> SSAO
    QS --> FOG
    
    PS --> INST
    PS --> DOTS
    PS --> XR
    
    CS --> MLS
    CS --> ALS
    
    LS_SET --> LIGHTMAP
    LS_SET --> DIRLIGHTMAP
```

## 2. RenderGraph Integration Architecture

```mermaid
graph TB
    subgraph "RenderGraph System"
        RG[RenderGraph]
        RGB[RenderGraphBuilder]
        RGP[RenderGraphPass]
        RGR[RenderGraphResources]
        RGC[RenderGraphCompiler]
    end
    
    subgraph "Resource Management"
        RTH[RTHandle System]
        RTHP[RTHandleResourcePool]
        TH[TextureHandle]
        BH[BufferHandle]
        RLH[RendererListHandle]
    end
    
    subgraph "URP RenderGraph Integration"
        URRG[UniversalRendererRenderGraph]
        RGFP[RenderGraphForwardPass]
        RGTP[RenderGraphTransparentPass]
        RGPP[RenderGraphPostProcessPass]
    end
    
    subgraph "Pass Data Structures"
        PD[PassData]
        FD[FrameData]
        URD[UniversalResourceData]
        UCD[UniversalCameraData]
        ULD[UniversalLightData]
        USD[UniversalShadowData]
    end
    
    subgraph "Execution Context"
        CMD[CommandBuffer]
        RCMD[RasterCommandBuffer]
        CCMD[ComputeCommandBuffer]
        UCMD[UnsafeCommandBuffer]
        GPU[GPU Execution]
    end
    
    RG --> RGB
    RGB --> RGP
    RGP --> RGR
    RG --> RGC
    RGC --> RGP
    
    RTH --> RTHP
    RTHP --> TH
    RTHP --> BH
    RTHP --> RLH
    
    URRG --> RG
    URRG --> RGFP
    URRG --> RGTP
    URRG --> RGPP
    
    RGFP --> PD
    RGTP --> PD
    RGPP --> PD
    PD --> FD
    FD --> URD
    FD --> UCD
    FD --> ULD
    FD --> USD
    
    RGP --> CMD
    CMD --> RCMD
    CMD --> CCMD
    CMD --> UCMD
    RCMD --> GPU
    CCMD --> GPU
    UCMD --> GPU
    
    TH --> RGR
    BH --> RGR
    RLH --> RGR
    RGR --> RCMD
```

## 3. Memory Layout and Resource Flow

```mermaid
graph LR
    subgraph "CPU Memory"
        subgraph "Managed Heap (.NET)"
            URP_OBJ[URP Objects]
            PASS_OBJ[Pass Objects]
            DATA_OBJ[Data Structures]
            MAT_OBJ[Material Objects]
            MESH_OBJ[Mesh Objects]
        end
        
        subgraph "Native Memory (C++)"
            CMD_BUF[Command Buffers]
            CULL_RES[Culling Results]
            RENDER_LIST[Renderer Lists]
            SHADER_DATA[Shader Data]
            TEXTURE_DATA[Texture Data]
        end
        
        subgraph "Stack Memory"
            TEMP_DATA[Temporary Data]
            FRAME_DATA[Frame Data]
            PASS_DATA[Pass Data]
        end
    end
    
    subgraph "GPU Memory (VRAM)"
        subgraph "Render Targets"
            COLOR_TEX[Color Targets]
            DEPTH_TEX[Depth Targets]
            SHADOW_TEX[Shadow Maps]
            TEMP_TEX[Temporary Textures]
        end
        
        subgraph "Geometry Buffers"
            VTX_BUF[Vertex Buffers]
            IDX_BUF[Index Buffers]
            INST_BUF[Instance Buffers]
        end
        
        subgraph "Constant Buffers"
            GLOBAL_CB[Global Constants]
            MATERIAL_CB[Material Constants]
            LIGHT_CB[Light Constants]
            SHADOW_CB[Shadow Constants]
        end
        
        subgraph "Shader Resources"
            VS[Vertex Shaders]
            FS[Fragment Shaders]
            VARIANTS[Shader Variants]
            TEXTURES[Textures]
        end
    end
    
    URP_OBJ --> CMD_BUF
    PASS_OBJ --> CMD_BUF
    DATA_OBJ --> SHADER_DATA
    MAT_OBJ --> MATERIAL_CB
    MESH_OBJ --> VTX_BUF
    MESH_OBJ --> IDX_BUF
    
    CMD_BUF --> COLOR_TEX
    CMD_BUF --> DEPTH_TEX
    CMD_BUF --> SHADOW_TEX
    
    CULL_RES --> RENDER_LIST
    RENDER_LIST --> VTX_BUF
    RENDER_LIST --> IDX_BUF
    RENDER_LIST --> INST_BUF
    
    SHADER_DATA --> GLOBAL_CB
    SHADER_DATA --> LIGHT_CB
    SHADER_DATA --> SHADOW_CB
    
    VARIANTS --> VS
    VARIANTS --> FS
    TEXTURE_DATA --> TEXTURES
    
    subgraph "Transparency Specific Resources"
        TRANS_SORT[Transparent Sorting Data]
        BLEND_STATE[Blend State Cache]
        ALPHA_TEX[Alpha Textures]
        DEPTH_COPY[Depth Copy Buffers]
    end
    
    TRANS_SORT --> RENDER_LIST
    BLEND_STATE --> COLOR_TEX
    ALPHA_TEX --> TEXTURES
    DEPTH_COPY --> TEMP_TEX
    
    TEMP_DATA --> TRANS_SORT
    FRAME_DATA --> BLEND_STATE
    PASS_DATA --> DEPTH_COPY
```

## 4. Transparency Resource Dependencies

```mermaid
graph TD
    subgraph "Input Resources"
        OPAQUE_COLOR[Opaque Color Buffer]
        DEPTH_BUFFER[Depth Buffer]
        SHADOW_MAPS[Shadow Maps]
        LIGHT_DATA[Light Data]
        MATERIAL_DATA[Material Properties]
    end
    
    subgraph "Transparency Processing"
        TRANS_SETUP[Transparent Settings]
        TRANS_SORT[Transparent Sorting]
        BLEND_SETUP[Blend State Setup]
        TRANS_RENDER[Transparent Rendering]
    end
    
    subgraph "Intermediate Resources"
        DEPTH_COPY[Depth Copy]
        OPAQUE_COPY[Opaque Copy]
        SORTED_LIST[Sorted Renderer List]
        BLEND_STATES[Blend State Cache]
    end
    
    subgraph "Output Resources"
        FINAL_COLOR[Final Color Buffer]
        UPDATED_DEPTH[Updated Depth Buffer]
        MOTION_VECTORS[Motion Vectors]
    end
    
    OPAQUE_COLOR --> TRANS_SETUP
    DEPTH_BUFFER --> TRANS_SETUP
    SHADOW_MAPS --> TRANS_SETUP
    LIGHT_DATA --> TRANS_SETUP
    MATERIAL_DATA --> TRANS_SETUP
    
    TRANS_SETUP --> DEPTH_COPY
    TRANS_SETUP --> OPAQUE_COPY
    TRANS_SETUP --> TRANS_SORT
    
    TRANS_SORT --> SORTED_LIST
    MATERIAL_DATA --> BLEND_SETUP
    BLEND_SETUP --> BLEND_STATES
    
    SORTED_LIST --> TRANS_RENDER
    BLEND_STATES --> TRANS_RENDER
    DEPTH_COPY --> TRANS_RENDER
    OPAQUE_COPY --> TRANS_RENDER
    SHADOW_MAPS --> TRANS_RENDER
    LIGHT_DATA --> TRANS_RENDER
    
    TRANS_RENDER --> FINAL_COLOR
    TRANS_RENDER --> UPDATED_DEPTH
    TRANS_RENDER --> MOTION_VECTORS
    
    subgraph "Resource Lifetime"
        PERSISTENT[Persistent Resources]
        FRAME_TEMP[Frame Temporary]
        PASS_TEMP[Pass Temporary]
    end
    
    SHADOW_MAPS --> PERSISTENT
    LIGHT_DATA --> PERSISTENT
    MATERIAL_DATA --> PERSISTENT
    
    OPAQUE_COLOR --> FRAME_TEMP
    DEPTH_BUFFER --> FRAME_TEMP
    FINAL_COLOR --> FRAME_TEMP
    
    DEPTH_COPY --> PASS_TEMP
    OPAQUE_COPY --> PASS_TEMP
    SORTED_LIST --> PASS_TEMP
    BLEND_STATES --> PASS_TEMP
```
