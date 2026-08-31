# 一帧画面是如何渲染出来的？—— 主线程 × 渲染线程 × StaticMesh × Material 的完整旅程

> 本文档基于 **UnrealEngine-ue6-main** 源码（结构等同 UE 5.8.x，个别行号有偏移），逐行核对。
> 三篇文档构成一条完整线索：
> - [engine_tick.md](../.AI/DeepSeek/engine_tick.md) —— 引擎主循环（游戏线程每帧干什么）
> - [render_pipeline.md](render_pipeline.md) —— 渲染管线五层与 Deferred 时序（渲染线程每帧干什么）
> - **本文** —— 把两者缝合成"一帧的完整旅程"，并深入回答：**一个 StaticMesh 如何结合 Material 被渲染出来**，以及 **游戏线程与渲染线程如何协作**。

---

## 0. 一帧的全局视图

UE5 渲染一帧，本质是 **四条流水线在时间上错开、协同推进**：

```
Game Thread                Render Thread               RHI Thread              GPU
   │                            │                          │                    │
   │ 逻辑 Tick / 组件状态        │                          │                    │
   │ ENQUEUE_RENDER_COMMAND ───► │ 消费命令（增删改场景）    │                    │
   │                            │                          │                    │
   │ RedrawViewports ──────────► │ RenderViewFamily_RT      │                    │
   │  提交一帧绘制请求            │  构建 RDG 图             │                    │
   │                            │  Execute → 写 RHICmdList ─► 翻译平台 API ─────► │
   │                            │                          │                    │
   │ FFrameEndSync::Sync ◄──────│  (等 N-1 帧渲染完成)      │                    │
   │ GFrameCounter++            │                          │                    │
```

**核心思想**：游戏线程（GT）与渲染线程（RT）**各自持有一份场景副本**，GT 只负责"下单"（把意图通过 `ENQUEUE_RENDER_COMMAND` 丢给 RT），RT 负责"干活"（构建 draw command）。两者通过 **帧号 + 命令队列 + 围栏** 保持一帧的滞后同步，从而并行跑。

---

## 1. 游戏线程：主循环如何发起一帧渲染

主循环在 `FEngineLoop::Tick()`（`Runtime/Launch/Private/LaunchEngineLoop.cpp`），完整 31 步见 engine_tick.md。与渲染相关的最关键路径如下：

```
FEngineLoop::Tick()
  ├─ ⑥ ENQUEUE_RENDER_COMMAND(BeginFrame) → BeginFrameRenderThread()
  │     · GFrameNumberRenderThread++（渲染帧号）
  ├─ ⑯ GEngine->Tick()  ← 游戏逻辑 + 视图提交都在这里
  │     └─ UGameEngine::Tick
  │           └─ UEngine::RedrawViewports
  │                 └─ FViewport::Draw
  │                       └─ UGameViewportClient::Draw
  │                             ├─ 构造 FSceneViewFamilyContext（本次绘制所有 View）
  │                             └─ FRendererModule::BeginRenderingViewFamily   ←★ 提交渲染
  ├─ ㉗ FFrameEndSync::Sync()      ←★ 帧同步点（等上一帧渲染完成）
  ├─ ㉙ GFrameCounter++            （游戏帧号）
  └─ ㉛ ENQUEUE_RENDER_COMMAND(EndFrame) → EndFrameRenderThread() → RHICmdList.EndFrame()
```

**关键点**：GT 每帧真正做"渲染动作"的只有三处 ——
1. 帧开始发 `BeginFrame` 命令（让 RT 对齐帧号）；
2. `RedrawViewports` 把一帧的绘制意图打包提交；
3. 帧末 `FFrameEndSync::Sync()` 做节流同步。

`BeginRenderingViewFamily`（`Runtime/Renderer/Private/SceneRendering.cpp:5394`）内部：

```cpp
void FRendererModule::BeginRenderingViewFamilies(FCanvas* Canvas, TConstArrayView<FSceneViewFamily*> ViewFamilies)
{
    // ① 先把组件最新的状态 flush 给渲染代理（transform/材质等）
    World->SendAllEndOfFrameUpdates();          // SceneRendering.cpp:5420
    // ② 把整帧渲染任务打包，ENQUEUE 到渲染线程
    //    → FSceneRenderBuilder 生成 SceneRenderer（Deferred/Mobile）
    //    → ENQUEUE_RENDER_COMMAND(SceneRenderBuilder_Render)   (SceneRenderBuilder.cpp:830)
}
```

`SendAllEndOfFrameUpdates()`（`Runtime/Engine/Private/LevelTick.cpp:1372`）是 **GT→RT 状态同步点**：把本帧所有 `MarkRenderStateDirty()` 的组件的最新 transform / 渲染状态，一次性推给对应的 `FPrimitiveSceneProxy`。

---

## 2. 游戏线程 ↔ 渲染线程的协作机制

### 2.1 ENQUEUE_RENDER_COMMAND —— 异步命令队列

`Runtime/RenderCore/Public/RenderingThread.h:1102`：

```cpp
#define ENQUEUE_RENDER_COMMAND(Type, ...) \
    DECLARE_RENDER_COMMAND_TAG(UE_JOIN(FRenderCommandTag_, Type, __LINE__), Type, __VA_ARGS__) \
    FRenderCommandDispatcher::Enqueue<UE_JOIN(FRenderCommandTag_, Type, __LINE__)>
```

`FRenderCommandDispatcher`（`RenderingThread.h:994`）把 lambda 压入渲染线程的命令队列，RT 在 `RenderingThreadMain`（`RenderingThread.cpp:141`）循环消费。

- **单向异步**：GT 只入队、不等结果，随后继续自己的逻辑 → 两线程并行。
- 队列有界（最多滞后 N 帧），由下面的帧同步保证不积压。

### 2.2 帧号与 One-Frame-Lag

| 变量 | 线程 | 含义 |
|------|------|------|
| `GFrameCounter` | GT | 游戏帧号，每帧 `GFrameCounter++` |
| `GFrameNumberRenderThread` | RT | 渲染帧号，`BeginFrameRenderThread` 里 ++ |
| `ViewFamily->FrameNumber` | GT | 本次 ViewFamily 属于哪一帧（`BeginRenderingViewFamily` 里由 `Scene->GetFrameNumber()` 赋值，SceneRendering.cpp:5461） |

**默认配置下 RT 比 GT 慢一帧**：GT 跑第 N 帧的逻辑时，RT 正在渲染第 N-1 帧。这是吞吐最高的方案（两线程永远不互相等待），代价是输入延迟一帧。

### 2.3 FFrameEndSync::Sync —— 帧末节流同步

`Runtime/RenderCore/Private/FrameEndSync.cpp:51`，两个 CVar 控制同步深度（同文件 :6/:12）：

```cpp
void Sync(EFlushMode FlushMode)
{
    // 渲染线程围栏：最多允许 GT 领先 1 帧（r.OneFrameThreadLag=1 默认）
    RenderThreadFences.Emplace();
    while (RenderThreadFences.Num() > (bFullSync ? 0 : 1))
        RenderThreadFences.RemoveAt(0);       // 淘汰旧围栏，腾出新帧

    // RHI 线程 / SwapChain 围栏（r.GTSyncType）
    //   <=0 : 与 N-1 渲染帧同步，再与 N-m RHI 帧同步（m=2+|GTSyncType|）
    //   ==1 : 与 N-1 RHI 帧同步
    //   ==2 : 与 GPU swapchain 翻转同步（最低延迟，需 VSync）
    PipelineFences.Emplace_GetRef().BeginFence(SyncDepth);
    while (PipelineFences.Num() > NumFramesOverlap)
    {
        PipelineFences[0].Wait(true);         // ← 真正阻塞游戏线程的地方
        PipelineFences.RemoveAt(0);
    }
}
```

- `FFrameEndSync` 定义于 `RenderingThread.h:40`。
- **作用**：防止 GT 跑得比 RT 快太多（否则命令队列无限膨胀 / 延迟无限大）；同时通过 `r.GTSyncType` 调节"GT 跟到哪一层"来平衡吞吐与延迟。

> 需要**立即同步**（如读回、改材质后立刻渲染）时用 `FlushRenderingCommands()` / `FRenderCommandFence`，它们会阻塞 GT 直到 RT 处理完队列。

---

## 3. 渲染线程：从提交到上屏

`ENQUEUE_RENDER_COMMAND(SceneRenderBuilder_Render)`（`SceneRenderBuilder.cpp:830`）在 RT 上执行：

```
RenderViewFamily_RenderThread                 (SceneRendering.cpp:5253)
  └─► Renderer->Render(GraphBuilder, ...)      (SceneRendering.cpp:5268)
        └─► FDeferredShadingSceneRenderer::Render   (DeferredShadingRenderer.cpp)
              ├─ BeginInitViews（可见性/Relevance/收集 dynamic mesh elements）
              ├─ GPU Scene Update（Primitive/Instance 上传 GPU）
              ├─ RenderPrePass / Occlusion / HZB
              ├─ RenderShadowDepthMaps（VSM）
              ├─ Nanite Cull+Raster → Visibility Buffer
              ├─ RenderBasePass  ★ StaticMesh 在这里真正被画出来
              ├─ RenderLights / Lumen GI / Reflections
              ├─ RenderTranslucency
              └─ AddPostProcessingPasses（Bloom/TSR/Tonemap/...）
```

每个子 pass 内部构造 RDG 图，`FRDGBuilder::Execute()` 编译（barrier/生命周期推导）后写成 `FRHICommandList`，由 RHI 线程翻译成 D3D12/Vulkan 提交 GPU。完整 pass 时序见 render_pipeline.md §5。

---

## 4. ★ StaticMesh 如何结合 Material 被渲染出来（核心问题）

下面把这个链路逐步拆开。一条 StaticMesh 从"内存里的数据"到"屏幕上的一堆像素"，经历 **6 个阶段**。

### 阶段一：StaticMesh 的渲染数据

`UStaticMesh` 的渲染数据在 `FStaticMeshRenderData`（`Runtime/Engine/Public/StaticMeshResources.h:765`），其核心是 **LOD 资源数组**：

```cpp
struct FStaticMeshLODResources                       // StaticMeshResources.h:417
{
    FStaticMeshSectionArray Sections;                // :422 每 Section = 一段子网格（对应一个材质槽）
    FStaticMeshVertexBuffers VertexBuffers;          // :478 顶点数据
    FRawStaticIndexBuffer IndexBuffer;               // :481 索引缓冲
    // ...
};

struct FStaticMeshVertexBuffers                      // :311
{
    FStaticMeshVertexBuffer StaticMeshVertexBuffer;  // 法线/切线/UV（StaticMeshVertexBuffer.h:149）
    FPositionVertexBuffer PositionVertexBuffer;      // 位置（PositionVertexBuffer.h:26）
    FColorVertexBuffer  ColorVertexBuffer;           // 顶点色（ColorVertexBuffer.h:15）
};

struct FStaticMeshSection                            // :193 每 Section 记录
{
    int32 MaterialIndex;      // ← 本 Section 用哪个材质槽！
    int32 FirstIndex, NumTriangles, MinVertexIndex, MaxVertexIndex;  // 索引/顶点范围
    bool bCastShadow; ...
};
```

**一句话**：StaticMesh 把几何拆成多个 Section，每个 Section 通过 `MaterialIndex` 指向一个材质槽。

### 阶段二：材质槽解析 —— Section 到底用哪个 Material

材质槽索引 → 具体 `UMaterialInterface` 的解析在 **proxy 构造时**完成，调用链：

```
FStaticMeshSceneProxy::FStaticMeshSceneProxy(...)           (StaticMeshSceneProxy.cpp:195)
  └─ for LODIndex: new (LODs) FLODInfo(...)                 (:310-333)
       └─ FLODInfo 构造                                     (:2409)
            └─ SectionInfo.Material = InProxyDesc.GetMaterial(Section.MaterialIndex)  (:2548)
```

`GetMaterial`（`StaticMeshSceneProxyDesc.cpp:121` → `StaticMeshComponentHelper.h:143`）的优先级：

```cpp
UMaterialInterface* FStaticMeshComponentHelper::GetMaterial(const T& Component, int32 MaterialIndex, ...)
{
    // ① 组件级覆盖材质优先（StaticMeshComponent 的 OverrideMaterials）
    if (OverrideMaterials.IsValidIndex(MaterialIndex) && OverrideMaterials[MaterialIndex])
        OutMaterial = OverrideMaterials[MaterialIndex];
    // ② 否则用 StaticMesh 自身的材质槽
    else if (Component.GetStaticMesh())
        OutMaterial = Component.GetStaticMesh()->GetMaterial(MaterialIndex);
    // ③ Nanite override 材质（若启用 Nanite）
    if (OutMaterial && Component.UseNaniteOverrideMaterials(...))
        OutMaterial = OutMaterial->GetNaniteOverride() ...;
    return OutMaterial;
}
```

**这就解释了"给组件某个 Slot 换材质，网格对应 Section 就会变"**：材质槽索引最终落到 `FLODInfo::FSectionInfo::Material`（`UMaterialInterface*`）。

### 阶段三：组件 → 场景代理（CreateSceneProxy）

```
UPrimitiveComponent::RegisterComponent
  └─ CreateSceneProxy()（虚函数）
       └─ UStaticMeshComponent::CreateSceneProxy()       (StaticMeshSceneProxy.cpp:2813)
            └─ FStaticMeshComponentHelper::CreateSceneProxy(...)   (StaticMeshComponentHelper.h:512)
                 └─ ::new FStaticMeshSceneProxy(this, false)       (StaticMeshSceneProxy.cpp:2805)
```

`FStaticMeshSceneProxy` 构造时（`:195`）：
- 保存 `RenderData`（`GetStaticMesh()->GetRenderData()`，:197）；
- 为每个 LOD 构建 `FLODInfo`（含阶段二解析好的 per-Section 材质、lightmap 数据）；
- 计算 `MaterialRelevance`（这个 mesh 需要哪些 pass、是否是透明/双面等）。

创建完成后经 `FScene::AddPrimitive`（`RendererScene.cpp:1042`）ENQUEUE 到渲染线程，形成 `FPrimitiveSceneInfo`（RT 侧 bookkeeping）。此后 **游戏线程的组件与渲染线程的 proxy 解耦**：GT 改组件，通过 `SendAllEndOfFrameUpdates` 更新 proxy；proxy 内部数据渲染线程独占。

### 阶段四：每帧生成 FMeshBatch —— 几何与材质的"第一次合体"

每帧渲染时，proxy 需要把自己翻译成渲染线程能理解的形式：**`FMeshBatch`**（`MeshBatch.h:359`）。

StaticMesh 有两条生成路径：

| 路径 | 入口 | 何时走 |
|------|------|--------|
| **静态路径** | `DrawStaticElements()`（StaticMeshSceneProxy.cpp:1311） | 加入场景时执行一次，产出被**缓存**的 mesh batch（性能关键路径） |
| **动态路径** | `GetDynamicMeshElements()`（:1599） | 每帧重新生成（Movable / 编辑器选中 / 调试视图等） |

两条路径最终都汇聚到同一个函数 —— **`FStaticMeshSceneProxy::GetMeshElement`**（`StaticMeshSceneProxy.cpp:686`），它是"StaticMesh 结合 Material"的核心：

```cpp
bool FStaticMeshSceneProxy::GetMeshElement(
    int32 LODIndex, int32 BatchIndex, int32 SectionIndex,
    uint8 InDepthPriorityGroup, bool bUseSelectionOutline,
    bool bAllowPreCulledIndices, FMeshBatch& OutMeshBatch) const
{
    const FStaticMeshLODResources& LOD = RenderData->LODResources[LODIndex];   // 几何数据
    const FStaticMeshVertexFactories& VFs = RenderData->LODVertexFactories[LODIndex];
    const FStaticMeshSection& Section = LOD.Sections[SectionIndex];

    // ★★★ 材质侧：取出渲染代理
    UMaterialInterface* MaterialInterface = ProxyLODInfo.Sections[SectionIndex].Material; // :707
    const FMaterialRenderProxy* MaterialRenderProxy = MaterialInterface->GetRenderProxy(); // :708
    const FMaterial& Material = MaterialRenderProxy->GetIncompleteMaterialWithFallback(FeatureLevel);

    // ★★★ 几何侧：选择顶点工厂（默认 FLocalVertexFactory）
    VertexFactory = &VFs.VertexFactory;                                    // :745
    OutMeshBatchElement.VertexFactoryUserData = VFs.VertexFactory.GetUniformBuffer();

    // 绑定索引缓冲、图元类型、范围（写入 OutMeshBatch.Elements[0]）
    const uint32 NumPrimitives = SetMeshElementGeometrySource(
        Section, LOD.IndexBuffer, LOD.AdditionalIndexBuffers,
        VertexFactory, bWireframe, bUseReversedIndices, OutMeshBatch);     // :762

    // 组装 FMeshBatch
    OutMeshBatch.SegmentIndex = SectionIndex;
    OutMeshBatch.LODIndex = LODIndex;
    OutMeshBatch.CastShadow = bCastShadow && Section.bCastShadow;
    OutMeshBatch.DepthPriorityGroup = (ESceneDepthPriorityGroup)InDepthPriorityGroup;
    OutMeshBatch.LCI = &ProxyLODInfo;                                      // lightmap 接口
    OutMeshBatch.MaterialRenderProxy = MaterialRenderProxy;               // :781 ←★ 材质代理
    OutMeshBatchElement.MinVertexIndex = Section.MinVertexIndex;
    OutMeshBatchElement.MaxVertexIndex = Section.MaxVertexIndex;
    // ...
}
```

**`FMeshBatch` 就是"几何 × 材质"的第一次合体**，包含三样东西：

```cpp
struct FMeshBatch                       // MeshBatch.h:359
{
    const FVertexFactory* VertexFactory;              // :364  ← 顶点装配方式
    const FMaterialRenderProxy* MaterialRenderProxy;  // :367  ← 材质（着色器来源）
    const FLightCacheInterface* LCI;                  // :370  ← 光照贴图接口
    TArray<FMeshBatchElement, ...> Elements;          // 每个 element = 索引缓冲 + 图元范围
    // EPrimitiveType Type; bUseForDepthPass; bUseAsOccluder; ...
};
```

### 阶段五：材质如何变成着色器（Material → FMaterialRenderProxy → FMaterial）

`GetRenderProxy()` 返回 `FMaterialRenderProxy`（`MaterialRenderProxy.h:105`），这是**渲染线程侧的材质门面**：

| 实现 | 位置 |
|------|------|
| `UMaterial::GetRenderProxy()` | Material.cpp:2374 → 返回 `DefaultMaterialInstance`（一个常驻的 FMaterialInstanceRenderProxy） |
| `UMaterialInstance::GetRenderProxy()` | MaterialInstance.cpp:2215 → `GetInstanceRenderProxy()` |

然后 `FMaterialRenderProxy::GetMaterialNoFallback(FeatureLevel)`（`MaterialInstance.cpp:280`）解析出真正的渲染材质：

```cpp
const FMaterial* FMaterialInstanceRenderProxy::GetMaterialNoFallback(ERHIFeatureLevel::Type Level) const
{
    if (Parent) {
        if (Owner->bHasStaticPermutationResource) {
            // 从 StaticPermutationMaterialResources 找已编译的 FMaterialResource
            const FMaterialResource* StaticPermutationResource = FindMaterialResource(...);
            if (StaticPermutationResource && StaticPermutationResource->GetRenderingThreadShaderMap())
                return StaticPermutationResource;        // ← 有 shader map，直接用
        } else {
            return Parent->GetRenderProxy()->GetMaterialNoFallback(Level);  // 向父级递归
        }
    }
    return nullptr;
}
```

`FMaterial` 内部持有 **`FMaterialShaderMap`**（`GetRenderingThreadShaderMap()`），里面是每个 pass（BasePass/Depth/Shadow...）× 每种 VertexFactory 组合编译好的 **`FMeshMaterialShader`**。

**降级机制**（`GetIncompleteMaterialWithFallback`，MaterialRenderProxy.cpp:895）：若 shader map 尚未编译完（编辑器热编译/异步编译中），沿 `GetFallback` 链回退到已编译的默认材质，**保证任何情况下都有东西可画**。

### 阶段六：FMeshBatch → FMeshDrawCommand（Mesh Pass Processor）

`FMeshBatch` 还不能直接画——它要针对**每个渲染 pass** 编译成一条完整的 **`FMeshDrawCommand`**（`MeshPassProcessor.h:1299`：PSO + shader bindings + 顶点流 + draw args）。

每个 pass 有自己的 `FMeshPassProcessor`（虚接口 `AddMeshBatch`，`MeshPassProcessor.h:2130`）。以 BasePass 为例，`FBasePassMeshProcessor::AddMeshBatch`（`BasePassRendering.cpp:2016`）：

```cpp
void FBasePassMeshProcessor::AddMeshBatch(const FMeshBatch& MeshBatch, uint64 BatchElementMask,
                                          const FPrimitiveSceneProxy* PrimitiveSceneProxy, int32 StaticMeshId)
{
    const FMaterialRenderProxy* MaterialRenderProxy = MeshBatch.MaterialRenderProxy;
    while (MaterialRenderProxy) {
        const FMaterial* Material = MaterialRenderProxy->GetMaterialNoFallback(FeatureLevel); // 材质
        if (Material && Material->GetRenderingThreadShaderMap()) {   // shader 就绪
            if (TryAddMeshBatch(MeshBatch, BatchElementMask, PrimitiveSceneProxy, StaticMeshId,
                                *MaterialRenderProxy, *Material))   // ★
                break;
        }
        MaterialRenderProxy = MaterialRenderProxy->GetFallback(FeatureLevel); // 降级
    }
}
```

`TryAddMeshBatch`（`:2057`）内部：
1. 从 `FMaterialShaderMap` 取该 pass 的 shader（`TryGetShaders`）；
2. 判定 blend/depth-stencil/rasterizer 状态；
3. 调用 `FMeshPassProcessor::BuildMeshDrawCommands`（`MeshPassProcessor.inl:57`）：

```cpp
// ① 组装最小 PSO 初始状态
FGraphicsMinimalPipelineStateInitializer PipelineState;
PipelineState.SetupBoundShaderState(VertexDeclaration, MeshProcessorShaders);
PipelineState.RasterizerState = GetStaticRasterizerState<true>(MeshFillMode, MeshCullMode);
PipelineState.BlendState = DrawRenderState.GetBlendState();
PipelineState.DepthStencilState = DrawRenderState.GetDepthStencilState();

// ② 从 VertexFactory 取顶点流
VertexFactory->GetStreams(FeatureLevel, InputStreamType, SharedMeshDrawCommand.VertexStreams);

// ③ 收集 shader bindings（VS/PS/GS 的 uniform/纹理绑定）
PassShaders.VertexShader->GetShaderBindings(...);
PassShaders.PixelShader->GetShaderBindings(...);

// ④ 对每个 BatchElement（一个 draw 单元）FinalizeCommand
for (BatchElementIndex ...) {
    FMeshDrawCommand& MeshDrawCommand = DrawListContext->AddCommand(...);
    FMeshMaterialShader::GetElementShaderBindings(...);
    DrawListContext->FinalizeCommand(..., PipelineState, ..., MeshDrawCommand);
}
```

**产出 `FMeshDrawCommand`**：一条已经绑定好 PSO、shader、顶点流、索引缓冲、draw args 的"可以立刻执行的绘制命令"。

### 静态命令的缓存（性能关键）

StaticMesh 静态路径的 mesh batch 及其编译结果 **不会每帧重建**，而是缓存：

- `FStaticMeshSceneExtension::AddStaticMeshes`（`StaticMeshes/StaticMeshSceneExtension.cpp:442`）在场景更新时执行 `DrawStaticElements`（`:478`），把 FMeshBatch 存入 Scene 的 `StaticMeshes` 数组；
- `CacheMeshDrawCommands`（`:517`）随后对每个 mesh batch × 每个 pass 调用 `PassMeshProcessor->AddMeshBatch`（`:629`），把编译好的 `FMeshDrawCommand` 以 `FCachedMeshDrawCommandInfo` 缓存（`RendererScene.cpp:5538/:5612` 挂接为异步任务）。

动态路径的 mesh batch 则每帧经 `FParallelMeshDrawCommandPass::DispatchPassSetup`（`MeshDrawCommands.h:126`）重新编译，存 `FDynamicMeshDrawCommandStorage`。

### 提交：真正画到屏幕上

BasePass 绘制时（`BasePassRendering.cpp:956 RenderBasePass` → `RenderBasePassInternal` :1332）：

```cpp
// BasePassRendering.cpp:1501 / :1585
if (auto* Pass = View.ParallelMeshDrawCommandPasses[EMeshPass::BasePass]; Pass && bShouldRenderView)
{
    Pass->BuildRenderingCommands(GraphBuilder, Scene->GPUScene, PassParameters->InstanceCullingDrawParams);
    GraphBuilder.AddPass(..., [&View, Pass, PassParameters](FRDGAsyncTask, FRHICommandList& RHICmdList)
    {
        SetStereoViewport(RHICmdList, View, 1.0f);
        Pass->Draw(RHICmdList, &PassParameters->InstanceCullingDrawParams);   // ← 真正提交
    });
}
```

`FMeshDrawCommand::SubmitDraw`（`MeshPassProcessor.h:1489`）最终：
- 设 PSO / render target / stencil；
- 绑 shader bindings、顶点流、索引缓冲；
- 发 `RHICmdList.DrawIndexedPrimitive(...)`（或 indirect 变体）。

这些 RHI 命令经 RDG Execute 写出，RHI 线程翻译成 D3D12/Vulkan 提交 GPU——StaticMesh × Material 变成屏幕上的一组像素。

---

## 5. 一帧完整时序图（Game ↔ Render ↔ RHI ↔ GPU）

以一个 Deferred + Nanite 场景、静态网格为例：

```
Game Thread                        Render Thread                        RHI Thread / GPU
────────────────                    ─────────────────                    ────────────────
FEngineLoop::Tick
  ├─ BeginFrame ──────────────────► BeginFrameRenderThread()
  ├─ GEngine->Tick()
  │   Actor/Component Tick
  │   UStaticMeshComponent::MarkRenderStateDirty()
  │
  ├─ RedrawViewports
  │   FViewport::Draw
  │     GameViewportClient::Draw
  │       BeginRenderingViewFamily
  │         World->SendAllEndOfFrameUpdates()   // 组件状态→proxy
  │         CreateSceneRenderer
  │         ENQUEUE_RENDER_COMMAND ───────────► RenderViewFamily_RenderThread
  │                                              FDeferredShadingSceneRenderer::Render
  │                                                ├─ BeginInitViews（可见性/Relevance）
  │                                                ├─ GPU Scene Update
  │                                                ├─ PrePass / HZB / Shadow
  │                                                ├─ Nanite Cull+Raster（Visibility Buffer）
  │                                                ├─ RenderBasePass
  │                                                │    · 静态网格：查缓存 FMeshDrawCommand
  │                                                │    · 动态网格：GetDynamicMeshElements
  │                                                │       → FBasePassMeshProcessor::AddMeshBatch
  │                                                │       → BuildMeshDrawCommands → FMeshDrawCommand
  │                                                │    · SubmitDraw → RHICmdList.DrawIndexedPrimitive ─► 翻译 → GPU
  │                                                ├─ Lights / Lumen / Translucency / PostProcess
  │                                                └─ RDG.Execute() 写 FRHICommandList ─────────────► RHI 线程 → GPU
  │
  ├─ FFrameEndSync::Sync()  (等上一帧渲染完)
  ├─ GFrameCounter++
  └─ EndFrame ───────────────────► EndFrameRenderThread() → RHICmdList.EndFrame()
```

---

## 6. 关键结论

1. **一帧 = 四条流水线错开并行**：GT 下单 → RT 建 draw command → RHI 翻译 → GPU 执行。RT 默认比 GT 慢一帧（`r.OneFrameThreadLag=1`），`FFrameEndSync::Sync` 在帧末节流，防止队列无限膨胀。
2. **GT↔RT 协作 = 单向异步命令队列 + 帧号对齐**：`ENQUEUE_RENDER_COMMAND` 只入队不等结果；需要同步时用 `FlushRenderingCommands` / `FRenderCommandFence`。
3. **StaticMesh 的渲染本质是"几何数据 × 材质"的两次合体**：
   - **第一次**：proxy 把 LOD 几何（Sections/VertexBuffers/IndexBuffer）+ 材质槽解析出的 `UMaterialInterface` 合成为 `FMeshBatch`（`GetMeshElement`，StaticMeshSceneProxy.cpp:686）；
   - **第二次**：各 pass 的 `FMeshPassProcessor::AddMeshBatch` 把 `FMeshBatch` 编译成可直接执行的 `FMeshDrawCommand`（PSO + shader bindings + 顶点流）。
4. **材质 = FMaterialRenderProxy → FMaterial → FMaterialShaderMap**：shader map 里是"pass × vertex factory"组合的编译产物；未就绪时走 `GetFallback` 降级链。
5. **静态网格的 draw command 被缓存**（`FStaticMeshSceneExtension::CacheMeshDrawCommands`），每帧只做可见性筛选 + 提交，不做重新编译——这是 StaticMesh 相比动态网格性能高一个量级的根本原因。

---

## 参考文件（基于 UnrealEngine-ue6-main）

| 主题 | 文件 |
|------|------|
| 主循环 | `Runtime/Launch/Private/LaunchEngineLoop.cpp` |
| GT→RT 桥 | `Runtime/RenderCore/Public/RenderingThread.h:1102` |
| 帧同步 | `Runtime/RenderCore/Private/FrameEndSync.cpp:51` |
| 视图提交 | `Runtime/Renderer/Private/SceneRendering.cpp:5394` |
| SceneRenderer | `Runtime/Renderer/Private/DeferredShadingRenderer.cpp` |
| StaticMesh 数据 | `Runtime/Engine/Public/StaticMeshResources.h:417` |
| StaticMesh Proxy | `Runtime/Engine/Private/StaticMeshSceneProxy.cpp:686/1311/1599` |
| 材质槽解析 | `Runtime/Engine/Public/StaticMeshComponentHelper.h:143` |
| 材质代理 | `Runtime/Engine/Public/Materials/MaterialRenderProxy.h:105` |
| Shader 解析 | `Runtime/Engine/Private/Materials/MaterialInstance.cpp:280` |
| Mesh Pass | `Runtime/Renderer/Public/MeshPassProcessor.h:2130` / `MeshPassProcessor.inl:57` |
| BasePass | `Runtime/Renderer/Private/BasePassRendering.cpp:2016` |
| 静态命令缓存 | `Runtime/Renderer/Private/StaticMeshes/StaticMeshSceneExtension.cpp:442/517` |
| 并行命令 | `Runtime/Renderer/Private/MeshDrawCommands.h:111` |
