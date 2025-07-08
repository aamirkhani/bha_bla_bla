# Unity URP Detailed Implementation Diagrams

## 1. DrawObjectsPass Internal Flow

```mermaid
flowchart TD
    START([DrawObjectsPass.Execute]) --> SETUP[Setup Pass Data]
    SETUP --> PROFILE[Begin Profiling Scope]
    PROFILE --> SHADOW_CHECK{Is Transparent & No Shadows?}
    
    SHADOW_CHECK -->|Yes| DISABLE_SHADOWS[TransparentSettingsPass.ExecutePass]
    SHADOW_CHECK -->|No| SORT_SETUP
    DISABLE_SHADOWS --> SORT_SETUP
    
    SORT_SETUP[Setup Sorting Criteria] --> OPAQUE_CHECK{Is Opaque?}
    OPAQUE_CHECK -->|Yes| OPAQUE_SORT[defaultOpaqueSortFlags]
    OPAQUE_CHECK -->|No| TRANS_SORT[SortingCriteria.CommonTransparent]
    
    OPAQUE_SORT --> DEPTH_PRIME_CHECK{Use Depth Priming?}
    TRANS_SORT --> DRAW_SETTINGS
    
    DEPTH_PRIME_CHECK -->|Yes| DEPTH_PRIME_SORT[Optimized Sort Flags]
    DEPTH_PRIME_CHECK -->|No| DRAW_SETTINGS
    DEPTH_PRIME_SORT --> DRAW_SETTINGS
    
    DRAW_SETTINGS[Create Drawing Settings] --> FILTER_SETUP[Setup Filtering Settings]
    FILTER_SETUP --> RENDER_STATE[Setup Render State Block]
    RENDER_STATE --> DRAW_CALL[context.DrawRenderers]
    
    subgraph "DrawRenderers Details"
        DRAW_CALL --> CULL_LOOP[For Each Culled Object]
        CULL_LOOP --> QUEUE_CHECK{In Render Queue Range?}
        QUEUE_CHECK -->|No| CULL_LOOP
        QUEUE_CHECK -->|Yes| LAYER_CHECK{In Layer Mask?}
        LAYER_CHECK -->|No| CULL_LOOP
        LAYER_CHECK -->|Yes| SHADER_CHECK{Has Compatible Shader?}
        SHADER_CHECK -->|No| CULL_LOOP
        SHADER_CHECK -->|Yes| SUBMIT_DRAW[Submit Draw Command]
        SUBMIT_DRAW --> CULL_LOOP
    end
    
    DRAW_CALL --> END_PROFILE[End Profiling Scope]
    END_PROFILE --> END([Pass Complete])
    
    style SHADOW_CHECK fill:#ffcccc
    style DISABLE_SHADOWS fill:#ffcccc
    style TRANS_SORT fill:#ccffcc
    style OPAQUE_SORT fill:#ccccff
```

## 2. Shader Compilation and Variant Selection

```mermaid
flowchart TD
    MAT_CHANGE[Material Property Changed] --> KEYWORD_UPDATE[Update Shader Keywords]
    KEYWORD_UPDATE --> VARIANT_HASH[Calculate Variant Hash]
    VARIANT_HASH --> CACHE_CHECK{Variant in Cache?}
    
    CACHE_CHECK -->|Yes| USE_CACHED[Use Cached Variant]
    CACHE_CHECK -->|No| COMPILE_NEW[Compile New Variant]
    
    COMPILE_NEW --> PREPROCESS[Shader Preprocessing]
    PREPROCESS --> KEYWORD_EVAL[Evaluate Keywords]
    KEYWORD_EVAL --> VARIANT_GEN[Generate Variant Code]
    VARIANT_GEN --> PLATFORM_COMPILE[Platform-Specific Compilation]
    PLATFORM_COMPILE --> CACHE_STORE[Store in Cache]
    CACHE_STORE --> USE_COMPILED[Use Compiled Variant]
    
    USE_CACHED --> GPU_UPLOAD
    USE_COMPILED --> GPU_UPLOAD
    
    GPU_UPLOAD[Upload to GPU] --> BIND_SHADER[Bind Shader for Rendering]
    
    subgraph "Keyword Evaluation Details"
        KEYWORD_EVAL --> SURFACE_TYPE{_SURFACE_TYPE_TRANSPARENT?}
        SURFACE_TYPE -->|Yes| TRANS_KEYWORDS[Enable Transparency Keywords]
        SURFACE_TYPE -->|No| OPAQUE_KEYWORDS[Standard Keywords Only]
        
        TRANS_KEYWORDS --> BLEND_CHECK{Blend Mode?}
        BLEND_CHECK --> ALPHA_STD[Standard Alpha]
        BLEND_CHECK --> ALPHA_PREMUL[_ALPHAPREMULTIPLY_ON]
        BLEND_CHECK --> ALPHA_MOD[_ALPHAMODULATE_ON]
        
        ALPHA_STD --> LIGHTING_KEYWORDS
        ALPHA_PREMUL --> LIGHTING_KEYWORDS
        ALPHA_MOD --> LIGHTING_KEYWORDS
        OPAQUE_KEYWORDS --> LIGHTING_KEYWORDS
        
        LIGHTING_KEYWORDS --> SHADOW_KEYWORDS
        SHADOW_KEYWORDS --> FEATURE_KEYWORDS
        FEATURE_KEYWORDS --> PLATFORM_KEYWORDS
    end
    
    BIND_SHADER --> RENDER_READY[Ready for Rendering]
```

## 3. RenderGraph Transparency Pass Implementation

```mermaid
flowchart TD
    RG_RECORD[RenderGraph.BeginRecording] --> ADD_PASS[AddRenderPass: Transparent Forward]
    ADD_PASS --> SETUP_BUILDER[Setup RenderGraphBuilder]
    
    SETUP_BUILDER --> DECLARE_INPUTS[Declare Input Resources]
    DECLARE_INPUTS --> DECLARE_OUTPUTS[Declare Output Resources]
    DECLARE_OUTPUTS --> SET_RENDER_FUNC[SetRenderFunc]
    
    subgraph "Resource Declaration"
        DECLARE_INPUTS --> USE_COLOR[UseColorBuffer: activeColorTexture]
        DECLARE_INPUTS --> USE_DEPTH[UseDepthBuffer: activeDepthTexture]
        DECLARE_INPUTS --> USE_OPAQUE[UseTexture: opaqueTexture]
        DECLARE_INPUTS --> USE_SHADOWS[UseTexture: shadowTextures]
        
        DECLARE_OUTPUTS --> WRITE_COLOR[WriteTexture: finalColorTexture]
        DECLARE_OUTPUTS --> READ_DEPTH[ReadTexture: depthTexture]
    end
    
    SET_RENDER_FUNC --> COMPILE_GRAPH[RenderGraph.Compile]
    COMPILE_GRAPH --> OPTIMIZE[Graph Optimization]
    OPTIMIZE --> EXECUTE_GRAPH[RenderGraph.Execute]
    
    EXECUTE_GRAPH --> EXEC_PASS[ExecutePass Callback]
    
    subgraph "ExecutePass Implementation"
        EXEC_PASS --> GET_RESOURCES[Get Resource Handles]
        GET_RESOURCES --> SET_RT[SetRenderTarget]
        SET_RT --> SET_GLOBALS[Set Global Textures]
        SET_GLOBALS --> SET_CONSTANTS[Set Shader Constants]
        SET_CONSTANTS --> DRAW_RENDERERS[DrawRenderers]
        
        SET_GLOBALS --> OPAQUE_TEX[_CameraOpaqueTexture]
        SET_GLOBALS --> DEPTH_TEX[_CameraDepthTexture]
        SET_GLOBALS --> SHADOW_TEX[_MainLightShadowmapTexture]
        
        SET_CONSTANTS --> VIEW_MATRIX[UNITY_MATRIX_V]
        SET_CONSTANTS --> PROJ_MATRIX[UNITY_MATRIX_P]
        SET_CONSTANTS --> VP_MATRIX[UNITY_MATRIX_VP]
        SET_CONSTANTS --> LIGHT_DATA[_MainLightPosition]
        SET_CONSTANTS --> SHADOW_DATA[_MainLightShadowParams]
    end
    
    DRAW_RENDERERS --> CLEANUP[Resource Cleanup]
    CLEANUP --> END_FRAME[EndFrame]
    
    style DECLARE_INPUTS fill:#e1f5fe
    style DECLARE_OUTPUTS fill:#f3e5f5
    style EXEC_PASS fill:#e8f5e8
```

## 4. Alpha Blending Pipeline Detail

```mermaid
flowchart TD
    FRAG_SHADER[Fragment Shader Output] --> ALPHA_TEST{Alpha Test Enabled?}
    
    ALPHA_TEST -->|Yes| CLIP_CHECK{alpha < _Cutoff?}
    ALPHA_TEST -->|No| BLEND_SETUP
    
    CLIP_CHECK -->|Yes| DISCARD[discard fragment]
    CLIP_CHECK -->|No| BLEND_SETUP
    
    BLEND_SETUP[Setup Blend Operation] --> BLEND_MODE{Blend Mode}
    
    BLEND_MODE --> ALPHA_BLEND[Standard Alpha]
    BLEND_MODE --> PREMUL_BLEND[Premultiplied Alpha]
    BLEND_MODE --> ADD_BLEND[Additive]
    BLEND_MODE --> MUL_BLEND[Multiply]
    
    subgraph "Standard Alpha Blending"
        ALPHA_BLEND --> ALPHA_EQ[Result = Src*SrcAlpha + Dst*(1-SrcAlpha)]
        ALPHA_EQ --> ALPHA_RESULT[Final Color]
    end
    
    subgraph "Premultiplied Alpha Blending"
        PREMUL_BLEND --> PREMUL_EQ[Result = Src + Dst*(1-SrcAlpha)]
        PREMUL_EQ --> PREMUL_RESULT[Final Color]
    end
    
    subgraph "Additive Blending"
        ADD_BLEND --> ADD_EQ[Result = Src*SrcAlpha + Dst]
        ADD_EQ --> ADD_RESULT[Final Color]
    end
    
    subgraph "Multiply Blending"
        MUL_BLEND --> MUL_EQ[Result = Src*DstColor]
        MUL_EQ --> MUL_RESULT[Final Color]
    end
    
    ALPHA_RESULT --> WRITE_FB[Write to Framebuffer]
    PREMUL_RESULT --> WRITE_FB
    ADD_RESULT --> WRITE_FB
    MUL_RESULT --> WRITE_FB
    
    WRITE_FB --> DEPTH_TEST{Depth Test}
    DEPTH_TEST -->|Pass| DEPTH_WRITE{Depth Write Enabled?}
    DEPTH_TEST -->|Fail| DISCARD
    
    DEPTH_WRITE -->|Yes| UPDATE_DEPTH[Update Depth Buffer]
    DEPTH_WRITE -->|No| NEXT_FRAGMENT
    
    UPDATE_DEPTH --> NEXT_FRAGMENT[Process Next Fragment]
    DISCARD --> NEXT_FRAGMENT
    
    style ALPHA_TEST fill:#fff3e0
    style CLIP_CHECK fill:#ffebee
    style BLEND_MODE fill:#e8f5e8
    style DEPTH_TEST fill:#e3f2fd
```

## 5. Shadow Handling for Transparency

```mermaid
flowchart TD
    TRANS_SETUP[Transparent Settings Pass] --> SHADOW_SETTING{Receive Shadows?}
    
    SHADOW_SETTING -->|No| DISABLE_MAIN[Disable Main Light Shadows]
    SHADOW_SETTING -->|Yes| ENABLE_SHADOWS[Enable Shadow Sampling]
    
    DISABLE_MAIN --> DISABLE_ADD[Disable Additional Light Shadows]
    DISABLE_ADD --> SET_EMPTY_PARAMS[Set Empty Shadow Parameters]
    
    SET_EMPTY_PARAMS --> MAIN_SHADOW_PARAMS[_MainLightShadowParams = (0,0,0,-1)]
    SET_EMPTY_PARAMS --> ADD_SHADOW_PARAMS[_AdditionalLightsShadowParams = (0,0,0,-1)]
    SET_EMPTY_PARAMS --> SHADOW_MATRIX[_MainLightWorldToShadow = Identity]
    
    ENABLE_SHADOWS --> SETUP_MAIN[Setup Main Light Shadows]
    SETUP_MAIN --> SETUP_ADD[Setup Additional Light Shadows]
    
    subgraph "Main Light Shadow Setup"
        SETUP_MAIN --> MAIN_SHADOW_MAP[Bind Main Shadow Map]
        MAIN_SHADOW_MAP --> MAIN_MATRIX[Setup Shadow Matrix]
        MAIN_MATRIX --> MAIN_PARAMS[Set Shadow Parameters]
        MAIN_PARAMS --> CASCADE_SETUP[Setup Cascade Data]
    end
    
    subgraph "Additional Light Shadow Setup"
        SETUP_ADD --> ADD_SHADOW_MAP[Bind Additional Shadow Maps]
        ADD_SHADOW_MAP --> ADD_MATRICES[Setup Shadow Matrices]
        ADD_MATRICES --> ADD_PARAMS[Set Shadow Parameters]
        ADD_PARAMS --> ATLAS_SETUP[Setup Shadow Atlas]
    end
    
    CASCADE_SETUP --> SHADER_CONSTANTS
    ATLAS_SETUP --> SHADER_CONSTANTS
    MAIN_SHADOW_PARAMS --> SHADER_CONSTANTS
    ADD_SHADOW_PARAMS --> SHADER_CONSTANTS
    SHADOW_MATRIX --> SHADER_CONSTANTS
    
    SHADER_CONSTANTS[Upload Shader Constants] --> TRANS_RENDER[Transparent Rendering]
    
    subgraph "Fragment Shader Shadow Sampling"
        TRANS_RENDER --> SAMPLE_SHADOWS{Sample Shadows?}
        SAMPLE_SHADOWS -->|No| NO_SHADOW[shadowAttenuation = 1.0]
        SAMPLE_SHADOWS -->|Yes| CALC_SHADOW_COORD[Calculate Shadow Coordinates]
        
        CALC_SHADOW_COORD --> SAMPLE_MAIN[Sample Main Light Shadow]
        SAMPLE_MAIN --> SAMPLE_ADD[Sample Additional Light Shadows]
        SAMPLE_ADD --> COMBINE_ATTEN[Combine Attenuation]
        
        NO_SHADOW --> APPLY_LIGHTING
        COMBINE_ATTEN --> APPLY_LIGHTING[Apply Lighting with Shadows]
    end
    
    APPLY_LIGHTING --> FINAL_COLOR[Final Fragment Color]
    
    style SHADOW_SETTING fill:#fff3e0
    style DISABLE_MAIN fill:#ffebee
    style ENABLE_SHADOWS fill:#e8f5e8
    style SAMPLE_SHADOWS fill:#e3f2fd
```
