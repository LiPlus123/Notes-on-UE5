# Unreal Engine 5 渲染管线（Render Pipeline）深度解析

> 基于 `UnrealEngine-ue6-main` 源码（5.8.x 结构等同，个别行号有偏移）。
> 相关背景：[MaterialSystemFramework.md](MaterialSystemFramework.md)、[material_shader_and_deferred_shading.md](material_shader_and_deferred_shading.md)。

---

## 0. 全局视图

UE5 的渲染管线可以分成 **五层**：

```
┌────────────────────────────────────────────────────────────────┐
│  L5. Post-Processing        （Tonemap / TSR / Bloom / DOF …）  │
├────────────────────────────────────────────────────────────────┤
│  L4. Scene Rendering        （BasePass / Lighting / Lumen …）  │
├────────────────────────────────────────────────────────────────┤
│  L3. Mesh Draw Pipeline     （FMeshDrawCommand / GPUScene）    │
├────────────────────────────────────────────────────────────────┤
│  L2. Render Dependency Graph（FRDGBuilder Setup→Compile→Exec） │
├────────────────────────────────────────────────────────────────┤
│  L1. RHI                    （FRHICommandList / D3D12/Vulkan） │
└────────────────────────────────────────────────────────────────┘
              ↑
              │
     Game Thread ↔ Render Thread ↔ RHI Thread ↔ GPU
```

一帧的生命周期同样可以按线程职责分成 **四段**：

```
Game Thread          Render Thread                RHI Thread          GPU
    │                     │                            │                │
    │ Tick / SendAll──►   │                            │                │
    │ EndOfFrameUpdates   │                            │                │
    │                     │                            │                │
    │ BeginRendering────► │ SceneRenderer::Render      │                │
    │ ViewFamily          │   FRDGBuilder Setup        │                │
    │                     │   Compile (barriers/pool)  │                │
    │                     │   Execute → RHICmdList ──► │ Translate ───► │ Draw / Dispatch
    │                     │                            │                │
```

---

## 1. Scene 表征层

### 1.1 FScene — 渲染线程的世界

`Engine/Source/Runtime/Renderer/Private/ScenePrivate.h:440`：

```cpp
class FScene : public FSceneInterface
{
    // 与游戏线程的 UWorld 一一对应，只在渲染线程访问
    FSceneLightInfoArray Lights;                              // :576
    TArray<FLightSceneInfoCompact> InvisibleLights;           // :582
    FSkyLightSceneProxy* SkyLight;                            // :620
    TArray<FLightSceneInfo*> MobileDirectionalLights;         // :694
    TArray<FLightSceneInfo*> DirectionalLights;               // :699
    int32 NumUnbuiltReflectionCaptures;                       // :829
    TMap<...> PrimitiveComponentIdToInfoMap;                  // :497
    TSet<...> PrimitivesNeedingUniformBufferUpdate;           // :541
    TArray<int32> PackedToPersistentIndexMap;                 // :544
    FFXSystemInterface* FXSystem;                             // :448
    TPimplPtr<FSceneDataImpl> SceneDataImpl;                  // :492 → GPU Scene 后端
    // ...
};
```

要点：
- **每个 `UWorld` 对应一个 `FScene`**（含 PIE 世界、Editor 预览世界、Sequencer 子世界等，是多实例的）
- **只在渲染线程读写**——游戏线程通过 `ENQUEUE_RENDER_COMMAND` 提交增删改
- 存的都是 **View-independent** 的数据（几何、光源、反射探针），View 相关的东西在 `FViewInfo`

### 1.2 Primitive 的"两半式"表示

一个物体在渲染系统中永远是一对孪生对象：

| Game Thread 侧 | Render Thread 侧 | 说明 |
|----------------|------------------|------|
| `UPrimitiveComponent` | `FPrimitiveSceneProxy` (Engine 模块) | 用户可派生 |
| — | `FPrimitiveSceneInfo` (Renderer 内部) | 引擎私有 bookkeeping |

**`FPrimitiveSceneProxy`** — `Engine/Source/Runtime/Engine/Public/PrimitiveSceneProxy.h:292`：

```cpp
class FPrimitiveSceneProxy
{
    // 用户 override 的接口
    virtual FPrimitiveViewRelevance GetViewRelevance(...) const;       // :543
    virtual void DrawStaticElements(FStaticPrimitiveDrawInterface*);    // :440
    virtual void GetDynamicMeshElements(...);                          // :505
    virtual void CreateRenderThreadResources(...);                     // :587
    // ...
};
```

每种可渲染组件（StaticMesh、SkeletalMesh、Particle、Landscape、Nanite、Instanced…）派生一个 Proxy 类型，负责**产出 `FMeshBatch`**。

**`FPrimitiveSceneInfo`** — `Engine/Source/Runtime/Renderer/Public/PrimitiveSceneInfo.h:106`：

```cpp
class FPrimitiveSceneInfo : public FDeferredCleanupInterface
{
    FPrimitiveSceneProxy* Proxy;               // :114
    FPrimitiveComponentId PrimitiveComponentId; // :120
    // 光照附着、hit proxy、cached mesh commands、persistent index...
};
```

**分工原则**：
- Proxy = **对外的多态接口**（用户 override）
- SceneInfo = **对内的固定状态**（引擎不允许用户碰）

### 1.3 View — FViewInfo 与 FSceneViewFamily

**`FSceneViewFamily`**（`Engine/Source/Runtime/Engine/Public/SceneView.h:2339`）
容纳一次绘制的所有 View（分屏 2 view，VR 立体 2 view，SceneCapture N view）。`FSceneViewFamilyContext` 是能自动清理资源的 RAII 版本 (`:2778`)。

**`FViewInfo`**（`Engine/Source/Runtime/Renderer/Private/SceneRendering.h:1261`）
渲染线程的 **per-view** 状态。继承自 `FSceneView`（gameplay 层可见）+ 追加：
- `PrimitiveVisibilityMap` `:1282`（本 view 每个 primitive 的可见位）
- `PrimitiveDefinitelyUnoccludedMap`（HZB 遮挡结果）
- `CachedViewUniformShaderParameters`（本帧 View UB 的数据源）
- Dynamic mesh element storage（本 view 生成的临时 mesh batch）

### 1.4 Primitive 生命周期

```
Game Thread                         Render Thread
    │                                    │
UPrimitiveComponent::RegisterComponent   │
    │                                    │
    ├─ CreateSceneProxy() ─────────►     │  (Component.h:2403 虚函数)
    │       返回 FPrimitiveSceneProxy*   │
    │                                    │
    ├─ FScene::AddPrimitive() ────►      │  (RendererScene.cpp:1042)
    │       ENQUEUE_RENDER_COMMAND       │
    │                                    ▼
    │                          BatchAddPrimitivesInternal
    │                                    │
    │                                    ▼
    │                          AddPrimitiveSceneInfo_RenderThread
    │                                    (RendererScene.cpp:808)
    │                                    │
    │                                    │  · 分配 PersistentIndex
    │                                    │  · 插入 Primitives 数组
    │                                    │  · 标记 GPU Scene 需要上传
    │                                    │  · 缓存 static mesh draw commands
    │                                    ▼
    │                          Scene 中就绪
```

`AddLightSceneInfo_RenderThread` (`ScenePrivate.h:1497`) 是对应的光源版本。

---

## 2. Game Thread → Render Thread 桥

### 2.1 关键抽象

| 抽象 | 位置 | 作用 |
|------|------|------|
| `ENQUEUE_RENDER_COMMAND(Tag)` | `RenderingThread.h:1102` | 声明一个渲染命令 tag，转发到 `FRenderCommandDispatcher::Enqueue<Tag>` (`:994`) |
| `FRenderCommandPipe` | `RenderingThread.h:574` | 异步命令管道，支持 `IsReplaying` (`:211`)、`FSyncScope` (`:233`) |
| `FDeferredCleanupInterface` | `RenderDeferredCleanup.h:10` | 延迟到渲染线程再删除（`FPrimitiveSceneInfo` 就用它） |
| `RenderingThreadMain` | `RenderingThread.cpp:141` | 渲染线程主循环，`StartRenderingThread` 在 `:422` |

### 2.2 SendAllEndOfFrameUpdates

`Engine/Source/Runtime/Engine/Private/LevelTick.cpp:1372 UWorld::SendAllEndOfFrameUpdates()`——每帧 Tick 结束后，游戏线程把所有 `MarkRenderStateDirty()` 过的组件的最新 transform / uniform 状态 flush 给渲染线程。这是"游戏世界 → 渲染世界"的同步点。

之后 `FRendererModule::BeginRenderingViewFamily`（`SceneRendering.cpp:5394`）从主循环那一侧调用，**它会先调 `World->SendAllEndOfFrameUpdates()`**（`:5420`），再把渲染任务丢给渲染线程。

---

## 3. 帧生命周期 — 从 Redraw 到 Render

完整调用链（省略中间无关的间接层）：

```
UGameEngine::Tick
    │
    └─► UEngine::RedrawViewports              (UnrealEngine.cpp:11989)
            │
            └─► FViewport::Draw()             (UnrealClient.cpp:1836)
                    │
                    ├─ EnqueueBeginRenderFrame
                    │
                    └─► UGameViewportClient::Draw   (GameViewportClient.cpp:1569)
                            │
                            │  构造 FSceneViewFamilyContext
                            │  为每个 Player 创建 FSceneView
                            │
                            └─► FRendererModule::BeginRenderingViewFamily
                                        (SceneRendering.cpp:5394)
                                    │
                                    ├─ BeginRenderingViewFamilies (:5399)
                                    │   └─ World->SendAllEndOfFrameUpdates() (:5420)
                                    │
                                    ├─ FSceneRenderBuilder::CreateSceneRenderer
                                    │       (SceneRenderBuilder.cpp:969)
                                    │       │
                                    │       └─► FSceneRenderProcessor::CreateSceneRenderers
                                    │               (SceneRenderBuilder.cpp:475)
                                    │               │
                                    │               ├─ new FDeferredShadingSceneRenderer  :516
                                    │               └─ new FMobileSceneRenderer           :521
                                    │               (按 EShadingPath 分派)
                                    │
                                    └─ ENQUEUE_RENDER_COMMAND(SceneRenderBuilder_Render)
                                            (SceneRenderBuilder.cpp:830)
                                            │
                                            ▼
             ══════════════════ 渲染线程边界 ══════════════════
                                            │
                                            ▼
                          RenderViewFamily_RenderThread    (SceneRendering.cpp:5253)
                                    │
                                    └─► Renderer->Render(GraphBuilder, ...)  (:5268)
                                            │
                                            └─► FDeferredShadingSceneRenderer::Render
                                                    (DeferredShadingRenderer.cpp:1832)
```

### 3.1 SceneRenderer 家族

```
ISceneRenderer
    ↑
FSceneRendererBase                (SceneRendering.h:2166)
    ↑                              持有 FScene*, Allocator, Uniforms
FSceneRenderer                    (SceneRendering.h:2293)
    ↑                              纯虚 Render(FRDGBuilder&, ...) :2415
    ├── FDeferredShadingSceneRenderer  (DeferredShadingRenderer.h:261)
    └── FMobileSceneRenderer           (SceneRendering.h:3022，实现在 MobileShadingRenderer.cpp:1192)
```

---

## 4. RDG — Render Dependency Graph

### 4.1 RDG 要解决的问题

`Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h:45-48` 文件头注释直言：

> "Use the render graph builder to build up a graph of passes and then call Execute() to process them. Resource barriers and lifetimes are derived from _RDG_ parameters in the pass parameter struct provided to each AddPass call."

传统的手写 RHI 代码要程序员自己管：
- **资源屏障 (barriers)** — SRV→UAV 之间的 `Transition`
- **资源生命周期** — 临时 RT 什么时候分配、什么时候释放
- **别名 (aliasing)** — 内存复用
- **异步计算** — Async Compute 队列与图形队列的同步

RDG 把这些**从声明的 shader 参数结构里推导出来**——你只需描述"这个 pass 读什么、写什么"，Builder 自动生成 barrier 和 alias。

### 4.2 三阶段：Setup → Compile → Execute

```
┌───────────────────────────────────────────────────────────────┐
│  Setup 阶段（渲染代码顺序调用）                                │
│     FRDGBuilder::AddPass(Name, ParamStruct, Flags, Lambda)    │
│         │                                                     │
│         └─ 记录一个 FRDGPass：                                │
│              · 参数结构（含 SRV/UAV/RT 声明）                 │
│              · Execute lambda（capture 参数）                 │
│              · Pipeline flag (Graphics / AsyncCompute)        │
└───────────────────────────────────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────────────────┐
│  Compile 阶段（Execute 内部先跑）                             │
│     · 反射解析每个 Pass 的参数结构                            │
│     · 建立资源 read/write 依赖图                              │
│     · 剔除无副作用且无消费者的 pass（dead code elimination）  │
│     · 分配 transient pool                                     │
│     · 计算 barrier / async compute fence                      │
└───────────────────────────────────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────────────────────────────┐
│  Execute 阶段                                                 │
│     · 按拓扑序遍历 pass                                       │
│     · 插入 barrier / begin/end RenderPass                     │
│     · 调用 Pass 的 Execute lambda → 写 FRHICommandList        │
└───────────────────────────────────────────────────────────────┘
```

### 4.3 关键 API

`FRDGBuilder` — `RenderGraphBuilder.h:50`：

```cpp
class FRDGBuilder : public FRDGScopeState
{
    FRDGBuilder(FRHICommandListImmediate&);                    // :66

    // 引入已有资源
    FRDGTextureRef RegisterExternalTexture(...);               // :79 / :84
    FRDGBufferRef  RegisterExternalBuffer (...);               // :90 / :91 / :94

    // 创建 transient 资源
    FRDGTextureRef CreateTexture(FRDGTextureDesc, Name);       // :104
    FRDGBufferRef  CreateBuffer (FRDGBufferDesc, Name);        // :110 / :116

    // 创建视图
    FRDGTextureSRVRef CreateSRV(FRDGTextureSRVDesc);           // :119
    FRDGBufferSRVRef  CreateSRV(FRDGBufferSRVDesc);            // :122
    FRDGTextureUAVRef CreateUAV(FRDGTextureUAVDesc);           // :130
    FRDGBufferUAVRef  CreateUAV(FRDGBufferUAVDesc);            // :138

    // 添加 pass
    template<typename ParamType, typename LambdaType>
    FRDGPassRef AddPass(FRDGEventName, ParamType*, ERDGPassFlags, LambdaType); // :221 / :225 / :233

    // 执行
    void Execute();                                            // :412
};
```

RDG 资源基类 `FRDGResource` — `RenderGraphResources.h:134`；纹理/缓冲子类 `FRDGTexture` `:569`、`FRDGBuffer` `:1307`。
Pass 对象 `FRDGPass` — `RenderGraphPass.h:230`。

### 4.4 参数结构与自动依赖

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyPassParams, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InSceneColor)          // 读
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutColor)// 写
    RENDER_TARGET_BINDING_SLOTS()                                  // RT 声明
END_SHADER_PARAMETER_STRUCT()

FMyPassParams* Params = GraphBuilder.AllocParameters<FMyPassParams>();
Params->InSceneColor = SceneColor;
Params->OutColor     = GraphBuilder.CreateUAV(MyTarget);

GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyCompute"),
    Params,
    ERDGPassFlags::Compute,
    [Params](FRHIComputeCommandList& RHICmdList)
    {
        // Execute lambda：RDG 已在此前插入好 barrier
        FComputeShaderUtils::Dispatch(RHICmdList, Shader, *Params, GroupCount);
    });
```

**RDG 的核心机制**：Builder 用反射扫描 `FMyPassParams` 里的 `SHADER_PARAMETER_RDG_*` 宏（`ShaderParameterMacros.h:1413`/`:1735`），自动知道该 pass 会读 `InSceneColor`、写 `OutColor`。跨 pass 时自动插入正确的 barrier。

### 4.5 调试

- `RDG_EVENT_SCOPE(GraphBuilder, "Name")` — `RenderGraphEvent.h:479`，压入命名的 GPU breadcrumb（GPU crash 时能看到最后一次 scope 名）
- `RDG_EVENT_SCOPE_STAT(...)` `:480` — 带 GPU stat 计数
- 老的 `RDG_GPU_STAT_SCOPE` `:520` 已 deprecate

---

## 5. FDeferredShadingSceneRenderer::Render — 完整时序

入口：`Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:1832`。

**按调用顺序展开**（省略了大量二级 helper，只列关键 stage）：

```
Render()  DeferredShadingRenderer.cpp:1832
  │
  ├─ BeginInitViews                          :6266 (called from :2149)
  │      · 可见性 gather / GDME 任务提交 / RHI 资源初始化
  │
  ├─ Update GPU Scene                        FGPUScene::Update  GPUScene.cpp:1835
  │      · 上传 primitive / instance / light 到 GPU
  │
  ├─ RenderPrePass (Depth Prepass, Early-Z)  :2473
  │      · FDepthPassMeshProcessor
  │
  ├─ EndInitViews                            :6411 (called from :2403)
  │      · relevance 完成 / per-view uniform 上传
  │
  ├─ RenderOcclusion                         SceneOcclusion.cpp:1451 (called at :2737)
  │      · BeginOcclusionTests   :1232 / :1414
  │      · FHZBOcclusionTester::Submit :951
  │
  ├─ HZB Build                               RenderHzb :570
  │      · BuildHZBFurthest → InitHZBCommonParameter  HZB.cpp:13
  │
  ├─ RenderShadowDepthMaps                   :2948/:2974/:3318
  │      · RenderShadowDepthMapAtlases  ShadowDepthRendering.cpp:1841
  │      · Virtual Shadow Maps 分支：
  │         RenderVirtualShadowMapsNanite    VirtualShadowMapArray.cpp:4363
  │         RenderVirtualShadowMapsNonNanite :4535
  │
  ├─ CompositionLighting.ProcessBeforeBasePass  :2984
  │      · DBuffer Decals
  │
  ├─ Nanite Cull+Raster                      Nanite/NaniteCullRaster.cpp:6651/:6784/:6834
  │      · 生成 Visibility Buffer 而非直接 GBuffer
  │
  ├─ RenderBasePass ★                        :3083
  │      · FBasePassMeshProcessor 走 mesh 路径
  │      · Nanite 材质 shading (Nanite/NaniteShading.cpp)
  │
  ├─ RenderAnisotropy (若有)                 AnisotropyRendering.cpp
  │      · 写 GBufferF
  │
  ├─ RenderVelocities (opaque)               :3401
  │
  ├─ CompositionLighting.ProcessAfterBasePass  :3222/:3418
  │      · Emissive / AO Decal
  │
  ├─ BeginUpdateLumenSceneTasks              LumenSceneRendering.cpp:2149
  ├─ UpdateLumenScene                        :2698
  ├─ RenderLumenSceneLighting                :3064
  │      · BeginGatherLumenLights   LumenSceneDirectLighting.cpp:2063
  │      · RenderDirectLightingForLumenScene :2393
  │      · LumenRadiosity
  │
  ├─ RenderLights ★                          LightRendering.cpp:1739 (called at :3511)
  │      · RenderLightFunction   LightFunctionRendering.cpp:274
  │      · InternalRenderLight   LightRendering.cpp:3070
  │      · Virtual Shadow Map projection composite
  │         VirtualShadowMapProjection.cpp:348 / :806
  │
  ├─ RenderDiffuseIndirectAndAmbientOcclusion  IndirectLightRendering.cpp:957 (:3472/:3562)
  │      · Lumen Screen Probe Gather
  │         LumenScreenProbeGather.cpp:1445 InterpolateAndIntegrate
  │      · LumenReflections   Lumen/LumenReflections.cpp:1219
  │
  ├─ RenderDeferredReflectionsAndSkyLighting  IndirectLightRendering.cpp:2001 (:3573)
  │
  ├─ RenderTranslucency                      TranslucentRendering.cpp:2270 (:3534/:3694)
  │      · 遍历 TranslucencyStandard / AfterDOF / AfterMotionBlur / Modulate 等桶
  │
  ├─ RenderVelocities (translucent)          :3687
  │
  ├─ RenderDistortion                        DistortionRendering.cpp
  │
  ├─ RenderSingleLayerWater                  SingleLayerWaterRendering.cpp
  │
  ├─ RenderFog / Sky Atmosphere
  │
  ├─ Path Tracer (可选)                      PathTracing.cpp
  │
  └─ AddPostProcessingPasses                 PostProcessing.cpp:418 (:4437)
```

### 5.1 InitViews — 可见性

**BeginInitViews / EndInitViews** 是所有 View 准备的分水岭。做的事：
1. **视锥剔除**（每个 View × 每个 primitive）
2. **距离剔除、Cull Distance Volume**
3. **HZB 遮挡查询** 结果回读（上一帧的 result）
4. **计算 relevance**（`FPrimitiveViewRelevance` — 该 primitive 需要哪些 pass）
5. **收集 dynamic mesh elements**（本帧动态生成的 `FMeshBatch`）
6. **准备 View UB / GPU Scene 增量**

### 5.2 GPU Scene — GPU 端持久场景

`FGPUScene::Update` (`GPUScene.cpp:1835`)：把 Primitive/Instance/Light 数据编码成 structured buffer 常驻显存，供 Nanite / VSM / Culling / Instance Draw 用。UE5 的"GPU-driven rendering"就是靠它——Culling / Draw Setup 都能整体在 compute shader 里跑。

### 5.3 Nanite

Nanite 不走传统 Mesh Draw，改为**软件光栅 + Visibility Buffer**：

```
Nanite Cluster 数据（GPU 常驻）
    │
    ▼
[CullRaster]  NaniteCullRaster.cpp:6651
    │  · 视锥/HZB/流式化 LOD 剔除（compute shader）
    │  · 大三角形走硬件光栅（emit micro-triangle）
    │  · 小三角形走软件光栅（写 VisibilityBuffer 64-bit atomic）
    ▼
Visibility Buffer（TriangleID + ClusterID + Depth）
    │
    ▼
[Shading]  NaniteShading.cpp
    │  · 根据 Vis Buffer 的 MaterialID 做 tile 分类
    │  · 每个 tile 只运行相同材质的像素（收敛 divergence）
    │  · 写入 GBuffer
    ▼
GBuffer
```

优点：几何量级不再和 draw call 挂钩，微三角开销大幅降低，天然处理遮挡。

### 5.4 Virtual Shadow Maps

VSM 也依赖 GPU Scene + Nanite：
- 逻辑上是一张 16K × 16K 的巨型 shadow map（每个光源），但物理上分成 128×128 页
- 只为**可见 & 高 mip 需求**的页分配物理内存
- Nanite 几何直接光栅到多页
- Non-Nanite 走传统 mesh 通道，投影时同样分页

关键文件在 `VirtualShadowMaps/VirtualShadowMapArray.cpp`：
- `Initialize` `:971`、`BuildPageAllocations` `:3277`
- `RenderVirtualShadowMapsNanite` `:4363` / `NonNanite` `:4535`
- 投影：`VirtualShadowMapProjection.cpp:348` (`RenderVirtualShadowMapProjectionCommon`)、`:619` 点光单次绘制、`:806` `CompositeVirtualShadowMapMask`

### 5.5 Lumen — 动态 GI 与反射

Lumen 有两个域：**Surface Cache**（Mesh Card 表面缓存）和 **Screen Space**（屏幕探针）。

```
[LumenSceneRendering.cpp]
    BeginUpdateLumenSceneTasks    :2149
    UpdateLumenScene              :2698
        │  ← 上传 MeshCards / SurfaceCache 页
        ▼
[LumenSceneDirectLighting.cpp]
    RenderDirectLightingForLumenScene :2393
        │  ← 光照 Surface Cache
        ▼
[LumenRadiosity.cpp]
    多 bounce 传递
        │
        ▼
[LumenScreenProbeGather.cpp]
    InterpolateAndIntegrate       :1445
        │  ← 屏幕探针发射，采样 Surface Cache
        ▼
DiffuseIndirect / AO / Specular
        │
        ▼
Composite 到 SceneColor
```

反射走 `Lumen/LumenReflections.cpp:1219`；硬件光追分支 `LumenReflectionHardwareRayTracing.cpp`；SSR 回退在 `ScreenSpaceRayTracing.cpp:561`。

### 5.6 Ray Tracing

独立的 RT 通道分布在 `Runtime/Renderer/Private/RayTracing/`：
- `RayTracingScene.cpp` — RT 场景更新（BLAS/TLAS）
- `RayTracingPrimaryRays.cpp`、`RayTracingShadows.cpp`、`RayTracingAmbientOcclusion.cpp`
- `RayTracingTranslucency.cpp`、`RaytracingSkylight.cpp`
- `PathTracing.cpp` — 完整 path tracer（Movie Render Queue 高质量输出）

---

## 6. Mesh Draw Pipeline

### 6.1 从 FMeshBatch 到 FMeshDrawCommand

```
FPrimitiveSceneProxy
    │
    │  DrawStaticElements() → 静态路径
    │  GetDynamicMeshElements() → 动态路径
    │
    ▼
FMeshBatch                                  MeshBatch.h:359
    · Elements: TArray<FMeshBatchElement>
    · MaterialRenderProxy*
    · VertexFactory*
    · 图元类型 / lightmap 参考 …
    │
    │  每个 pass 有自己的 FMeshPassProcessor：
    │     FBasePassMeshProcessor, FDepthPassMeshProcessor,
    │     FShadowDepthPassMeshProcessor, ...
    │
    ▼
FMeshPassProcessor::AddMeshBatch()           MeshPassProcessor.h:2130 (virtual)
    │  1. 从 FMaterialShaderMap 取 shader（TryGetShaders）
    │  2. 判定 blend/depth/rasterizer state
    │  3. 生成 shader bindings
    │  4. 打包成 FMeshDrawCommand
    │
    ▼
FMeshDrawCommand                             MeshPassProcessor.h:1299
    · PSO 描述
    · ShaderBindings（uniform buffer / SRV / sampler）
    · VertexStreams
    · IndexBuffer
    · draw args（BaseVertex, FirstIndex, NumPrimitives, InstanceCount）
```

**两条命令产出路径**：

| 路径 | 存储 | 何时构建 | 使用场景 |
|------|------|----------|----------|
| **Cached (Static)** | `FCachedMeshDrawCommandInfo` (`MeshPassProcessor.h:1920`) | Primitive `AddToScene` 时或材质变化时 | 静态 mesh，永久缓存 |
| **Dynamic** | `FDynamicMeshDrawCommandStorage` (`:1740`) | 每帧 InitViews 后重建 | 骨骼、粒子、GDME 动态生成的 |

### 6.2 并行命令生成

`FParallelMeshDrawCommandPass` — `MeshDrawCommands.h:111`：

> "Encapsulates two parallel tasks - mesh command setup task and drawing task."

一个 MeshPass（例如 BasePass）分成两个并行 job：
1. **Setup Task** — 从可见 Primitive 列表并行生成 `FMeshDrawCommand`（多 worker 分块处理）
2. **Draw Task** — 排序、去重、绑定 PSO、发提交任务（`FMeshDrawCommand::SubmitDraw`）

### 6.3 提交到 RHI

`FMeshDrawCommand::SubmitDraw` — `MeshPassProcessor.h:1489`：

```cpp
void FMeshDrawCommand::SubmitDraw(
    const FMeshDrawCommand& RESTRICT MeshDrawCommand,
    /* ... */,
    FRHICommandList& RHICmdList,
    /* ... */)
{
    SubmitDrawBegin(...);                          // :1460  设 PSO / RT / stencil ref
    // 遍历 shader bindings，SetShaderTexture/UniformBuffer/etc.
    // 绑 Vertex / Index Buffer
    RHICmdList.DrawIndexedPrimitive(...);          // 或 indirect 变体 :1476/:1486
    SubmitDrawEnd(...);                            // :1470
}
```

底层 RHI 命令列表：
- `FRHICommandList` — `RHICommandList.h:3626`（普通并行命令列表）
- `FRHICommandListImmediate` — `:4423`（渲染线程直连、可 flush 到 RHI 线程）
- `DrawIndexedPrimitive` — `:3792`
- `SetGraphicsPipelineState` — `:3919` / `:3933`（绑 PSO / precompiled PSO + stencil）

---

## 7. Post Processing

### 7.1 入口与顺序

`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.cpp:418 AddPostProcessingPasses`。

`EPass` 枚举（`:514-565`）以数组下标声明 pass 顺序，Motion Blur/DoF/Bloom/Local Exposure/Film Grain 等由设置驱动内嵌到序列里（`:785-841`）：

```
SceneColor (after Basepass + Lighting + Translucency)
    │
    ├─ Motion Blur                    PostProcessMotionBlur.cpp:1334 AddMotionBlurPass
    │
    ├─ Depth of Field (Diaphragm)     DiaphragmDOF.cpp:1514 AddPasses
    │
    ├─ Subsurface (SSS)               PostProcessSubsurface.cpp:1901 AddSubsurfacePass
    │
    ├─ Eye Adaptation / Histogram     PostProcessEyeAdaptation.cpp:1175 / :1317
    ├─ Local Exposure                 PostProcessLocalExposure.cpp
    │
    ├─ Bloom (Gaussian 或 FFT)        PostProcessBloomSetup.cpp:120 / :197
    │                                 PostProcessFFTBloom.cpp:716 AddFFTBloomPass
    │
    ├─ Temporal AA / TSR              TemporalAA.cpp:707 AddTemporalAAPass
    │                                 TemporalSuperResolution.cpp:1835 / :1903
    │
    ├─ PostProcessMaterialBeforeBloom / AfterTonemapping  (MD_PostProcess 材质插入点)
    │
    ├─ Tonemap (+ Film Grain + Vignette)  PostProcessTonemap.cpp:609 AddTonemapPass
    │
    ├─ FXAA                           PostProcessAA.cpp:58 AddFXAAPass
    ├─ SMAA (可选)                    SubpixelMorphologicalAA.cpp:369 AddSMAAPasses
    │
    ├─ ChannelMask / SelectionOutline / EditorPrimitive  （编辑器）
    │
    ├─ Primary Upscale                （从内部分辨率放大到显示分辨率）
    ├─ Secondary Upscale
    │
    └─ Backbuffer
```

### 7.2 抗锯齿的三条路

| 方法 | 文件 | 用途 |
|------|------|------|
| **TAA** | `TemporalAA.cpp:707` | 传统时域抗锯齿 |
| **TSR** | `TemporalSuperResolution.cpp:1835`/`:1903` | UE5 主力，支持超分辨率上采样 |
| **FXAA** | `PostProcessAA.cpp:58` | 单帧后处理（画质弱但便宜） |
| **SMAA** | `SubpixelMorphologicalAA.cpp:369` | 亚像素形态学 |
| **MSAA** | `MobileShadingRenderer.cpp:447 / :1942 / :2101` | **仅前向/移动**，硬件多重采样 |

TSR 的 kernel graph 非常庞大（`TemporalSuperResolution.cpp` 主体从 `:1835` 一直延伸到 `:3839`），包括：History Rejection、Motion Rejection、Anti-Flicker、Frequency Decomposition、Kernel Solve、DilateVelocity、TranslucencyFallback 等 20+ 个子 pass。

### 7.3 后处理材质插入点

用户 `MD_PostProcess` 材质根据 Blendable Location 挂在几个固定点（`AddPostProcessingPasses` 内部循环）：
- Before Bloom
- Before Tonemapper (Replacing Tonemapper)
- After Tonemapper
- SSR Input / After Motion Blur / Before Translucency 等（视版本增删）

---

## 8. RHI 层与 PSO

### 8.1 命令列表

```
渲染线程                RHI 线程            GPU 驱动
   │                       │                   │
   │ FRHICommandListImmediate                  │
   │    ← 记录 SetPSO / SetSRV / Draw          │
   │                       │                   │
   │ Enqueue ────────────► │                   │
   │                       │ 平台后端：         │
   │                       │   D3D12 CmdList   │
   │                       │   Vulkan CmdBuffer│
   │                       │   Metal Encoder   │
   │                       │                   │
   │                       │ Submit ─────────► │ → GPU 队列
```

- `FRHICommandList` (`RHICommandList.h:3626`) — 通用命令列表，可从任何 job 并行录制
- `FRHICommandListImmediate` (`:4423`) — 渲染线程独有，能立即 flush；`RHIThread` 消费队列后翻译到平台 API

### 8.2 Pipeline State Object (PSO)

关键 API：
- `SetGraphicsPipelineState(FGraphicsPipelineStateInitializer, ...)` — `:3919`
- `SetGraphicsPipelineState(FRHIGraphicsPipelineState*, StencilRef)` — `:3933`（预编译版本）

**PSO 缓存**：UE 会在 `FMeshDrawCommand` 生成时同步注册 PSO 请求，避免运行时 hitching。启动时会读取 `.upipelinecache` 预热。

### 8.3 Shader Parameter 声明宏

`Engine/Source/Runtime/RenderCore/Public/ShaderParameterMacros.h`：

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParams, )              // :1413
    SHADER_PARAMETER(float, MyScalar)
    SHADER_PARAMETER_TEXTURE(Texture2D, MyTex)
    SHADER_PARAMETER_SAMPLER(SamplerState, MySampler)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, MyRDGTex)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, MyUAV) // :1735
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<float>, MyBufSRV)
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

宏在 C++ 侧生成有反射信息的结构体（RDG 依靠它推导依赖），同时匹配 shader 侧的 `.usf` 常量声明。**这是 UE5 里绑定 shader 资源的规范方式**——手写 `SetShaderTexture` 已属遗留代码。

---

## 9. 一帧的完整链路总览

以一个 Deferred + Nanite + Lumen + TSR 的普通场景为例：

```
Game Thread                     Render Thread                          GPU
──────────────                  ────────────────                       ──────
Tick(dt)
  Actor::Tick / AnimBP / …
  UPrimitiveComponent::
    MarkRenderStateDirty
                                                                       
SendAllEndOfFrameUpdates
  flush component 到 proxy
                                                                       
RedrawViewports
  FViewport::Draw
  GameViewportClient::Draw
  BeginRenderingViewFamily
    构造 FSceneViewFamilyContext
    构造 FViewInfo[]
    CreateSceneRenderer
      → FDeferredShadingSceneRenderer

  ENQUEUE_RENDER_COMMAND ────►   RenderViewFamily_RenderThread
                                   FRDGBuilder Setup:
                                     Render()
                                       BeginInitViews (可见性/GDME/Relevance)
                                       GPU Scene Update ──────────────► compute: 上传 Primitive/Instance SB
                                       Depth Prepass                    ─► VS/PS: Early-Z
                                       EndInitViews
                                       Occlusion + HZB                  ─► compute: HZB mip chain
                                       ShadowDepthMaps (+VSM Nanite)    ─► Nanite CS + 常规 raster
                                       DBuffer Decals
                                       Nanite Cull+Raster               ─► compute: Vis Buffer atomic
                                       Base Pass Mesh Draw              ─► VS/PS: → GBuffer[A..F]
                                       Nanite Shading                   ─► compute: Vis→GBuffer
                                       Emissive Decals
                                       Velocity
                                       Lumen Scene Lighting             ─► compute: Surface Cache
                                       Deferred Lights                  ─► PS: GBuffer → SceneColor
                                         · LightFunction quad
                                         · Complex/Simple tile
                                       Lumen ScreenProbeGather          ─► compute: 屏幕探针 GI
                                       Lumen Reflections                ─► compute (可 HWRT)
                                       Sky + Reflection Composite
                                       Translucency                     ─► VS/PS: Forward Lighting
                                       Distortion
                                       Fog / Atmosphere
                                       PostProcess
                                         Motion Blur → DoF → SSS
                                         → Bloom → Eye Adaptation
                                         → TSR (超分)
                                         → Tonemap + Film Grain
                                         → FXAA
                                         → Upscale
                                       Present
                                                                        
                                   FRDGBuilder.Execute():
                                     · Compile: 分析 barrier / transient
                                     · Execute: 遍历 pass，写 CmdList

                                   FRHICommandListImmediate ───────────► RHI Thread
                                                                            ├─ 翻译到 D3D12/Vulkan
                                                                            └─ Submit ────────► GPU
                                                                                                 │
                                                                                                 └─► 上屏
```

---

## 10. 关键结论

1. **Scene 是 View-independent 的孪生世界**：`FScene` 只在渲染线程活动；每个 `UPrimitiveComponent` 有 Proxy + SceneInfo 的两半式代理。
2. **GT↔RT 通过 `ENQUEUE_RENDER_COMMAND` 单向异步**：游戏线程只是"下单"，渲染线程持有自己的一份场景副本。
3. **RDG 把 barrier / lifetime 从命令式变声明式**：写 pass 时描述读写关系，Builder 自动生成 barrier；`Setup → Compile → Execute` 三阶段。
4. **DeferredShadingSceneRenderer::Render 就是整帧的骨架**：几十个子 pass 顺序调用；每个 pass 内部构造 RDG 图。
5. **Mesh Draw Pipeline 走 `FMeshBatch → FMeshDrawCommand`**：Cached 走静态路径永久保存，Dynamic 每帧重建；并行 setup + draw 分两个 job。
6. **Nanite / VSM / Lumen 是 UE5 的三大 GPU 驱动特性**：都建立在 GPU Scene + Compute Shader 之上，脱离了传统 draw-call-per-primitive 模型。
7. **RHI 在最底层**：`FRHICommandListImmediate` 是渲染线程写入的命令流，被 RHI 线程翻译成 D3D12/Vulkan/Metal 后提交到 GPU。
8. **PostProcess 是 fullscreen 化的独立子图**：不依赖 mesh draw，全走 RDG compute/quad pass。TSR 是当前的主力抗锯齿+超分方案。

---

## 参考文件（快速跳转）

| 主题 | 主要文件 |
|------|---------|
| Scene | `Renderer/Private/ScenePrivate.h` / `RendererScene.cpp` |
| Primitive | `Engine/Public/PrimitiveSceneProxy.h` / `Renderer/Public/PrimitiveSceneInfo.h` |
| View | `Engine/Public/SceneView.h` / `Renderer/Private/SceneRendering.h` |
| SceneRenderer | `Renderer/Private/DeferredShadingRenderer.h/.cpp` |
| GT→RT 桥 | `RenderCore/Public/RenderingThread.h` / `Engine/Private/LevelTick.cpp` |
| RDG | `RenderCore/Public/RenderGraphBuilder.h` / `RenderGraphResources.h` / `RenderGraphPass.h` |
| Mesh Draw | `Renderer/Public/MeshPassProcessor.h` / `Engine/Public/MeshBatch.h` |
| RHI | `RHI/Public/RHICommandList.h` |
| GPU Scene | `Renderer/Private/GPUScene.cpp` |
| Nanite | `Renderer/Private/Nanite/NaniteCullRaster.cpp` / `NaniteShading.cpp` |
| Lumen | `Renderer/Private/Lumen/LumenSceneRendering.cpp` / `LumenScreenProbeGather.cpp` |
| VSM | `Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp` |
| Shadow | `Renderer/Private/ShadowSetup.cpp` / `ShadowDepthRendering.cpp` |
| PostProcess | `Renderer/Private/PostProcess/PostProcessing.cpp` |
| TSR | `Renderer/Private/PostProcess/TemporalSuperResolution.cpp` |
