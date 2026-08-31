# UE 渲染自上而下全景：Camera / Light / Sky / Primitive / Material

> 版本：UE 5.8.1 / UE6-main（源码路径前缀 `Engine/Source/Runtime/`，均已省略；行号对应 UE6-main，5.8 近似）
> 目标读者：想从"一帧是怎么被画出来的"最高层视角，逐层往下看清 UE 渲染体系的人
> 相关文档：
> - [material_shader_and_deferred_shading.md](material_shader_and_deferred_shading.md)（材质 Domain/Blend/ShadingModel 三轴 + Deferred 管线）
> - [MeshComponents.md](MeshComponents.md)（UMeshComponent 各分支的完整选型指南）
> - [FDeferredShadingSceneRenderer.md](../../Rendering/FDeferredShadingSceneRenderer.md)（渲染器每帧 Pass 编排）
> - [SkeletalMeshComp.md](SkeletalMeshComp.md)（骨骼分支深入）

---

## 目录

- [UE 渲染自上而下全景：Camera / Light / Sky / Primitive / Material](#ue-渲染自上而下全景camera--light--sky--primitive--material)
  - [目录](#目录)
  - [1. 总览：一帧画面的五个要素](#1-总览一帧画面的五个要素)
  - [2. Camera：从"眼睛"到 View](#2-camera从眼睛到-view)
    - [2.1 定位：Camera 不是 Primitive](#21-定位camera-不是-primitive)
    - [2.2 一帧视角的产生：FMinimalViewInfo](#22-一帧视角的产生fminimalviewinfo)
    - [2.3 渲染线程的视角：FSceneView / FViewInfo](#23-渲染线程的视角fsceneview--fviewinfo)
    - [2.4 Camera 如何驱动整帧：视锥剔除](#24-camera-如何驱动整帧视锥剔除)
  - [3. Light：光的组件、代理与光照](#3-light光的组件代理与光照)
    - [3.1 组件树（修正继承关系）](#31-组件树修正继承关系)
    - [3.2 光的场景侧：FLightSceneProxy / FLightSceneInfo](#32-光的场景侧flightsceneproxy--flightsceneinfo)
    - [3.3 光类型速查](#33-光类型速查)
    - [3.4 光的阴影](#34-光的阴影)
    - [3.5 光照通道（Lighting Channels）](#35-光照通道lighting-channels)
    - [3.6 光如何进入 Deferred Lighting](#36-光如何进入-deferred-lighting)
  - [4. Sky：大气、云、天光与雾](#4-sky大气云天光与雾)
    - [4.1 四件套与继承关系](#41-四件套与继承关系)
    - [4.2 天空大气：USkyAtmosphereComponent](#42-天空大气uskyatmospherecomponent)
    - [4.3 体积云：UVolumetricCloudComponent](#43-体积云uvolumetriccloudcomponent)
    - [4.4 天光：USkyLightComponent](#44-天光uskylightcomponent)
    - [4.5 指数高度雾：UExponentialHeightFogComponent](#45-指数高度雾uexponentialheightfogcomponent)
    - [4.6 Sky 在帧管线中的位置](#46-sky-在帧管线中的位置)
  - [5. UPrimitiveComponent 及其派生](#5-uprimitivecomponent-及其派生)
    - [5.1 继承链与声明](#51-继承链与声明)
    - [5.2 职责拆分](#52-职责拆分)
    - [5.3 渲染相关属性速查](#53-渲染相关属性速查)
    - [5.4 派生类树](#54-派生类树)
    - [5.5 一个典型派生：FStaticMeshSceneProxy 的诞生](#55-一个典型派生fstaticmeshsceneproxy-的诞生)
    - [5.6 5.8 的新机制：IPrimitiveComponentInterface](#56-58-的新机制iprimitivecomponentinterface)
  - [6. UMaterial 与 UMaterialInstance](#6-umaterial-与-umaterialinstance)
    - [6.1 抽象基类 UMaterialInterface](#61-抽象基类-umaterialinterface)
    - [6.2 UMaterial：资产（表达式图 + 编译产物）](#62-umaterial资产表达式图--编译产物)
    - [6.3 FMaterial 与 Shader 编译/挑选流程](#63-fmaterial-与-shader-编译挑选流程)
    - [6.4 UMaterialInstance：参数覆盖](#64-umaterialinstance参数覆盖)
    - [6.5 MIC vs MID](#65-mic-vs-mid)
    - [6.6 参数如何传到 GPU（UniformExpressionSet）](#66-参数如何传到-gpuuniformexpressionset)
  - [7. 它们如何在同一帧里协作：渲染管线](#7-它们如何在同一帧里协作渲染管线)
    - [7.1 一张时序图：物体入场景](#71-一张时序图物体入场景)
    - [7.2 游戏线程：注册组件](#72-游戏线程注册组件)
    - [7.3 生成代理：FScene::AddPrimitive](#73-生成代理fsceneaddprimitive)
    - [7.4 渲染线程：入场景结构](#74-渲染线程入场景结构)
    - [7.5 静态路径 vs 动态路径](#75-静态路径-vs-动态路径)
    - [7.6 渲染线程看到的世界：FScene 数据结构](#76-渲染线程看到的世界fscene-数据结构)
    - [7.7 最后一公里：从 FStaticMeshBatch 到 GPU DrawCall](#77-最后一公里从-fstaticmeshbatch-到-gpu-drawcall)
    - [7.8 每帧的完整 Pass 顺序](#78-每帧的完整-pass-顺序)
  - [8. 关键概念速查表](#8-关键概念速查表)
  - [9. 延伸 FAQ](#9-延伸-faq)

---

## 1. 总览：一帧画面的五个要素

渲染一个画面，永远可以拆成五个自上而下的问题：

1. **往哪看？** —— **Camera**（相机）决定视点、朝向、视野，进而决定哪些物体"看得到"（剔除）。
2. **用什么照亮？** —— **Light**（光源）决定场景的亮度与颜色分布（直接光照 + 阴影）。
3. **环境是什么样？** —— **Sky**（大气/云/天光/雾）决定"没有光源直射的地方"长什么样（间接光、天空背景、远景雾）。
4. **世界里有啥？** —— **Primitive**（可渲染单元）决定"画哪些几何体、摆在哪"。
5. **表面怎么着色？** —— **Material**（材质）决定每个像素反射什么光。

一句话总览：

> **Camera 选出一个 View（视角）**，用它剔除出**可见的 Primitive 集合**；每个可见 Primitive 由 **Material** 决定表面属性（写进 GBuffer）；**Light 和 Sky** 在光照阶段消费这些属性算出最终颜色。游戏线程的 UObject（`UCameraComponent` / `ULightComponent` / `USkyAtmosphereComponent` / `UPrimitiveComponent` / `UMaterial`）全都通过"快照成纯数据代理 + 渲染命令队列"的方式，变成渲染线程里的 `FSceneView / FLightSceneInfo / FSkyAtmosphereSceneProxy / FPrimitiveSceneProxy / FMaterialRenderProxy`。

```text
            游戏线程（UObject 世界）                   渲染线程（数据世界）
  Camera   UCameraComponent ──GetCameraView()──▶ FMinimalViewInfo → FSceneView/FViewInfo
                                                  （视锥剔除的输入 ★）
  Light    ULightComponent  ──CreateSceneProxy()──▶ FLightSceneProxy → FLightSceneInfo
                                                          │         （进 FScene，生成 shadow map / 光照体积）
  Sky      USkyAtmosphereComponent ──────────────▶ FSkyAtmosphereSceneProxy
           UVolumetricCloudComponent ────────────▶ FVolumetricCloudSceneProxy
           USkyLightComponent      ──────────────▶ （天光，走光代理但进环境光通道）
           UExponentialHeightFogComponent ───────▶ （高度雾数据，整帧生效）
  Primitive UPrimitiveComponent ──CreateSceneProxy()──▶ FPrimitiveSceneProxy
                                                        │ + FPrimitiveSceneInfo
                                                        └─ 八叉树 / GPUScene / 缓存 draw call
  Material UMaterialInterface  ──GetRenderProxy()──▶ FMaterialRenderProxy（uniform buffer + shader）

  每帧：FDeferredShadingSceneRenderer::Render()
         InitViews（Camera 剔除） → PrePass/BasePass（Primitive+Material 写 GBuffer）
         → RenderLights（Light 直接光） → RenderDeferredReflectionsAndSkyLighting（Sky 环境光/反射）
         → 体积云/大气/雾（Sky） → 半透明 → 后处理
```

后文按这个顺序：**Camera → Light → Sky → Primitive → Material → 一帧协作**逐层展开。§5、§6、§7 与旧版 [PrimitiveComponent_And_Material.md](PrimitiveComponent_And_Material.md) 对应。

---

## 2. Camera：从"眼睛"到 View

### 2.1 定位：Camera 不是 Primitive

`UCameraComponent` 直接继承 `USceneComponent`（[CameraComponent.h:32](Source/Runtime/Engine/Classes/Camera/CameraComponent.h#L32)），**不是** `UPrimitiveComponent`：

```text
USceneComponent
 └─ UCameraComponent     ← 有变换、有 FOV 等参数，但没有渲染代理、不进八叉树
```

它不产生任何 draw call、不参与材质渲染，职责只有一个：**每帧算出一个"视角描述"（`FMinimalViewInfo`），驱动整个渲染器往哪看**。真正的渲染侧视图数据是 `FSceneView`（引擎层）和 `FViewInfo`（渲染器层），它们由 Camera 的描述构建，而不是 Camera 本身。

### 2.2 一帧视角的产生：FMinimalViewInfo

每帧由 `APlayerCameraManager` / `AActor` 调用组件的 `GetCameraView`（[CameraComponent.h:268](Source/Runtime/Engine/Classes/Camera/CameraComponent.h#L268)）：

```cpp
ENGINE_API virtual void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView);
```

`FMinimalViewInfo`（[CameraTypes.h:36](Source/Runtime/Engine/Classes/Camera/CameraTypes.h#L36)）是一份**与引擎渲染无关的"相机参数包"**：

| 字段 | 含义 |
|---|---|
| `Location / Rotation` | 视点位置与朝向 |
| `FOV` | 垂直视野角 |
| `AspectRatio` | 宽高比（相机不强制，由 Viewport 决定） |
| `bConstrainAspectRatio` | 是否锁定比例（黑边） |
| `OrthoWidth` / `bIsOrtho` | 正交相机参数 |
| `PostProcessSettings` | 后处理叠加（Bloom/DOF/色调…）——**后处理设置经由相机传递** |
| `ProjectionMode` | 透视 / 正交 |
| `FilmbackSettings` | 虚拟摄像机电影机背（适配宽银幕） |

也就是说：**编辑器里调相机 FOV、勾"固定比例"、加后处理 Blendable，改的都是这份结构**。它是游戏线程侧的"相机快照"。

### 2.3 渲染线程的视角：FSceneView / FViewInfo

`FMinimalViewInfo` 不能直接被渲染器用，要再转成渲染侧对象：

```text
FMinimalViewInfo
   │  （UE::SceneRenderer / GameViewport 组装）
   ▼
FSceneViewInitOptions（SceneView.h:179）
   │  填充 ViewRect / ViewMatrices 初始化参数 / 后处理
   ▼
FSceneView（SceneView.h:1495）      ← 引擎层"一个视图"，包含 ViewMatrices + ViewRect + 后处理
   │  （FSceneRenderer 子类再增强）
   ▼
FViewInfo（SceneRendering.h:1261）  ← 渲染器层"一个视图"，加入 GPU 相关/每帧临时数据
```

关键成员（都在 `FSceneView` / `FViewInfo` 上）：

| 成员 | 位置 | 作用 |
|---|---|---|
| `FViewMatrices ViewMatrices` | [SceneView.h:329](Source/Runtime/Engine/Public/SceneView.h#L329) | **ViewMatrix + ProjMatrix**，顶点从世界空间 → 屏幕的数学变换 |
| `FIntRect ViewRect` / `ConstrainedViewRect` | [SceneView.h:62-69](Source/Runtime/Engine/Public/SceneView.h#L62) | 该视图渲染的屏幕矩形（分辨率、黑边裁剪） |
| `FIntRect UnscaledViewRect` | 同上 | 未缩放原始矩形（TSR 等超采样时与 ViewRect 不同） |
| `FFinalPostProcessSettings FinalPostProcessSettings` | | 后处理参数（已与 Camera 的叠加合并） |
| `FConvexVolume ViewFrustum` | | **视锥体**（六面体），剔除测试的几何体 |
| `FSceneViewState* State` | | 跨帧状态（TAA 历史、曝光、时间累积…） |
| `FViewMatrices PrevViewMatrices` | | 上一帧矩阵（运动矢量 / 重投影） |

> 注意：一个 `FSceneViewFamily`（[SceneView.h:2339](Source/Runtime/Engine/Public/SceneView.h#L2339)）可含多个 View（分屏、VR 双眼、SceneCapture 同帧渲染多个视角），每个 View 一个 `FViewInfo`。

### 2.4 Camera 如何驱动整帧：视锥剔除

Camera 不是被"画"出来的，它是**剔除的源头**。渲染器初始化视图时（`FDeferredShadingSceneRenderer::BeginInitViews/EndInitViews`，[DeferredShadingRenderer.cpp:1614/1907](Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp#L1614)）：

```text
FViewInfo::ViewFrustum（由 ViewMatrices 反推六面体）
   │
   └─▶ 与 PrimitiveOctree 相交测试（见 §7.6）
          │ 命中 → 进可见集合 → 进 RenderList / GPU Scene
          └─ 未命中 → 直接丢弃（连 draw call 都不生成）
```

所以**相机参数一变（FOV/位置/朝向），可见集合就变**；这也是"相机本身不渲染，却决定了渲染量"的根本机制。

---

## 3. Light：光的组件、代理与光照

### 3.1 组件树（修正继承关系）

> ⚠️ 修正：旧版文档把 `ULightComponentBase` 画在 `UPrimitiveComponent` 下，**这是错的**。光的组件基类直接继承 `USceneComponent`（[LightComponentBase.h:15](Source/Runtime/Engine/Classes/Components/LightComponentBase.h#L15)），灯光**不进八叉树、不产生普通网格 draw call**，它有自己的场景结构（`FLightSceneInfo`）。

```text
USceneComponent
 └─ ULightComponentBase                          ← 光照公共接口（颜色/强度/通道/开关）
     ├─ ULightComponent                          ← 具体光类型的公共基类（衰减参数等）
     │   ├─ UDirectionalLightComponent           ← 平行光（DirectionalLightComponent.h:18）
     │   └─ ULocalLightComponent                 ← 有位置的局部光（LocalLightComponent.h:18）
     │       ├─ UPointLightComponent             ← 点光（PointLightComponent.h:19）
     │       │   └─ USpotLightComponent          ← 聚光灯（SpotLightComponent.h:17，继承点光）
     │       └─ URectLightComponent              ← 矩形/面光（RectLightComponent.h:24）
     └─ USkyLightComponent                       ← 天光（SkyLightComponent.h:101，特殊，见 §4.4）
```

各类型的"新增职责"：

- **UDirectionalLightComponent**：无位置、无衰减，只有方向（模拟太阳），支持级联阴影（CSM）和"影响大气"；
- **ULocalLightComponent**：有位置 + 衰减半径（`AttenuationRadius`）+ 光源半径（`SourceRadius`，面光阴影软边）；
- **UPointLightComponent**：径向衰减；**USpotLightComponent** 再加内/外圆锥角（Inner/OuterConeAngle）；
- **URectLightComponent**：矩形面光（面积光），UE5 中常用于高质量软阴影 / 反射。

### 3.2 光的场景侧：FLightSceneProxy / FLightSceneInfo

与 Primitive 对称，灯光的"组件 → 代理 → 场景"链路是：

```text
ULightComponent::CreateSceneProxy()              (LightComponent.h:505)
   └─▶ FLightSceneProxy                           (LightSceneProxy.h:45)   ← 渲染线程可用的光快照
   │      （颜色/强度/衰减/阴影设置/IES/通道…，全拷贝）
   │
FScene::AddLight(Light)                          (RendererScene.cpp:1977)
   ├─ if (!Light->SceneProxy)  Light->SceneProxy = Light->CreateSceneProxy();   // :1984
   ├─ Proxy->SetTransform(...)                                                  // :2003
   ├─ Proxy->LightSceneInfo = new FLightSceneInfo(Proxy, true);                 // :2006
   └─ ENQUEUE_RENDER_COMMAND(BatchAddLightsCommand)                             // :2020
       └─ 渲染线程：EnqueueAdd(LightSceneInfo) → 入 FScene 灯光数组
```

渲染线程侧的数据结构（[LightSceneInfo.h](Source/Runtime/Renderer/Private/LightSceneInfo.h)）：

| 结构 | 位置 | 作用 |
|---|---|---|
| `FLightSceneInfo` | [:207](Source/Runtime/Renderer/Private/LightSceneInfo.h#L207) | 一个已入场景的光，持有 `Proxy`、`Id`、阴影相关状态、衰减数据 |
| `FLightSceneInfoCompact` | [:35](Source/Runtime/Renderer/Private/LightSceneInfo.h#L35) | 紧凑 SoA 结构（位置/颜色/半径/类型…），供 GPU Scene / 聚类光照快速遍历 |
| `FScene::Lights`（数组） | | 渲染线程持有的全部光 |

> 与 Primitive 的关键差异：**Primitive 有八叉树做空间索引；光没有统一空间索引**，而是靠 `FLightSceneInfoCompact` 的紧凑数组 + 每帧 GatherAndSortLights 排序（按是否可见、是否投射阴影等），配合 tile/cluster 灯光索引（`FroxelGrid`）来决定"每个像素受哪些光影响"。

### 3.3 光类型速查

| 类型 | 位置信息 | 衰减 | 阴影 | 典型用途 |
|---|---|---|---|---|
| 平行光 | 只有方向 | 无 | 级联阴影 CSM / VSM | 太阳、月亮 |
| 点光 | 有位置 | 径向（按距离衰减） | 立体 shadow map（cubemap）/ VSM | 灯泡、火把 |
| 聚光 | 有位置+方向 | 径向 × 圆锥 | 单个 shadow map（透视投影） | 手电、舞台灯 |
| 矩形光 | 有位置+法向 | 面衰减 | VSM / Lumen 软阴影 | 棚灯、面积光 |
| 天光 | 环境球 | — | 无（间接光，见 §4.4） | 环境照明 |

强度单位：点/聚/矩形光默认用**流明（lumen）**，平行光用**勒克斯（lux）**（[LightComponent.h:48](Source/Runtime/Engine/Classes/Components/LightComponent.h#L48) 附近 `Intensity` / `IntensityUnits` 相关 UPROPERTY）。

### 3.4 光的阴影

阴影是光的独立子系统，帧内单独跑一遍（`FDeferredShadingSceneRenderer::RenderShadowDepthMaps`，[DeferredShadingRenderer.cpp:2948/2974](Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp#L2948)）：

```text
每帧 RenderShadowDepthMaps
   ├─ 确定哪些光需要阴影（CastShadow + 是否在可见范围内 + 距离衰减）
   ├─ 对每个需要阴影的光，从光的位置/方向渲染场景深度（Shadow Depth Pass）
   │     ├─ CSM：平行光分 4 个级联，按距离分级
   │     └─ VSM（Virtual Shadow Maps，UE5 默认）：把整张 shadow map 当虚拟纹理，
   │          只渲染被占用/需要的页面 → 支持超高分辨率阴影、少内存
   └─ 阴影贴图在 Deferred Lighting 里被采样，做 PCF/PCSS 软阴影
```

UE5 的 **Virtual Shadow Maps（VSM）** 取代了传统 shadow map：它把方向光的阴影空间切成页（page），用 GPU Scene 的实例数据做页面分配，**只有相机看得到的区域才渲染阴影页**，所以能给出接近 16384² 的有效分辨率而内存可接受。

### 3.5 光照通道（Lighting Channels）

每条通道（Channel 0/1/2，红/绿/蓝）是一个独立开关（[PrimitiveComponent.h:421](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L421) 的 `SetLightingChannels` 与灯光的 `AffectsGlobalIllumination` 等）：

- 灯光声明"我属于哪些通道"；
- 物体声明"我接收哪些通道的光"；
- 只有交集非空才互相照亮。

用于"场景中的灯不该照亮角色"这类分层（比如 UI 灯、体积光只作用于特定物体）。

### 3.6 光如何进入 Deferred Lighting

延迟光照阶段（`FDeferredShadingSceneRenderer::RenderLights`，[LightRendering.cpp:1739](Source/Runtime/Renderer/Private/LightRendering.cpp#L1739)）：

```text
RenderLights
   ├─ 遍历 Scene 中可见的光（按 FLightSceneInfoCompact / Froxel 网格）
   ├─ 对每个光：绑定 FDeferredLightPS（LightRendering.cpp:3070 附近 InternalRenderLight）
   │     ├─ 从 GBuffer 解码表面参数（DecodeGBufferData）
   │     ├─ 采样该光的 shadow map / VSM → 阴影因子
   │     ├─ 应用 Light Function 材质（可选，做灯光形状遮罩）
   │     └─ IntegrateBxDF（按材质 ShadingModel 分派，见 material_shader_and_deferred_shading.md §3.4）
   └─ Additive 累加到 SceneColor
```

灯光本身不写 GBuffer——它**消费** BasePass 写好的 GBuffer，逐像素累加光照结果。这就是"延迟着色"的本质：把"光照"和"几何/材质"解耦成两个阶段。

---

## 4. Sky：大气、云、天光与雾

### 4.1 四件套与继承关系

Sky 不是单一系统，而是四个可叠加的组件，各管一件事：

```text
USceneComponent
 ├─ USkyAtmosphereComponent          ← 大气散射（SkyAtmosphereComponent.h:49）
 ├─ UVolumetricCloudComponent        ← 体积云（VolumetricCloudComponent.h:28）
 ├─ UExponentialHeightFogComponent   ← 指数高度雾（ExponentialHeightFogComponent.h:17）
 └─ （天光 USkyLightComponent 在灯分支，见 §3.1 / §4.4）
```

> 注意：**这四者都直接继承 USceneComponent**（不是 UPrimitiveComponent，也不全是 ULightComponentBase）——它们不产普通网格 draw call，各自有独立的渲染路径与代理。

### 4.2 天空大气：USkyAtmosphereComponent

`USkyAtmosphereComponent`（[SkyAtmosphereComponent.h:49](Source/Runtime/Engine/Classes/Components/SkyAtmosphereComponent.h#L49)）模拟地球大气（瑞利散射 + 米氏散射 + 臭氧吸收），产生：

- **天空背景颜色**（天穹）；
- **大气透视**：让远处的物体变蓝变淡（雾化的物理正确版）；
- 供 Lumen / 反射采样的大气 LUT（透射率/多重散射/天空全景查找表）。

其渲染侧代理是 `FSkyAtmosphereSceneProxy`（[SkyAtmosphereComponent.h:282](Source/Runtime/Engine/Classes/Components/SkyAtmosphereComponent.h#L282)），每帧预计算若干 LUT（`FAtmosphereTransmittanceLUT` 等，`SkyAtmosphereRendering.cpp`），再用一个全屏 pass 合成天空与大气透视。

### 4.3 体积云：UVolumetricCloudComponent

`UVolumetricCloudComponent`（[VolumetricCloudComponent.h:28](Source/Runtime/Engine/Classes/Components/VolumetricCloudComponent.h#L28)）在场景中放一个**体积云层**，用 ray marching 渲染：

- 云由一组参数化噪声（密度层 `LayerBottomAltitude/LayerHeight`）描述，无需网格资产；
- 渲染线程代理 `FVolumetricCloudSceneProxy`（[VolumetricCloudComponent.h:243](Source/Runtime/Engine/Classes/Components/VolumetricCloudComponent.h#L243)）；
- 可以独立选"每像素追迹"或"重投影缓存"两种质量档；
- 云会接收平行光（太阳）的照明与阴影，并与大气（背景）合成。

### 4.4 天光：USkyLightComponent

`USkyLightComponent`（[SkyLightComponent.h:101](Source/Runtime/Engine/Classes/Components/SkyLightComponent.h#L101)）是**环境光来源**，继承 `ULightComponentBase` 但**不走普通光的 Deferred 光照**：

- 捕获环境（静态烘焙 cubemap，或 `bRealTimeCapture` [SkyLightComponent.h:108](Source/Runtime/Engine/Classes/Components/SkyLightComponent.h#L108) 实时渲染场景到环境贴图）；
- 提供天空/环境的间接漫反射 + 镜面反射（`SkyIrradianceEnvironmentMap`）；
- 在 `RenderDeferredReflectionsAndSkyLighting`（[DeferredShadingRenderer.cpp:3573](Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp#L3573)）阶段被消费，作为间接光的兜底（当没有 Lumen 时）或与 Lumen 的远场来源（`RenderSkyLight` 在 Lumen 中提供远场天空）。

### 4.5 指数高度雾：UExponentialHeightFogComponent

`UExponentialHeightFogComponent`（[ExponentialHeightFogComponent.h:17](Source/Runtime/Engine/Classes/Components/ExponentialHeightFogComponent.h#L17)）提供传统的**体积雾**（与大气散射互补，艺术家可控）：

- 以高度指数衰减的雾密度（`ExponentialFogDensity` / `FogHeightFalloff`）；
- 可叠加方向光雾色、雾指数分布（`FogDensity`）、体积雾散射（`bEnableVolumetricFog`，需要 Froxel 体积雾）；
- 在 `RenderFog`（[DeferredShadingRenderer.cpp:3903](Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp#L3903)）阶段合成，可作用于云之上（`RenderFogOnClouds`）。

### 4.6 Sky 在帧管线中的位置

在 `FDeferredShadingSceneRenderer::Render()` 里，Sky 相关 Pass 集中在**直接光照之后、半透明之前**（顺序即依赖顺序）：

```text
… RenderLights（直接光） →
RenderDeferredReflectionsAndSkyLighting   (DeferredShadingRenderer.cpp:3573)
    └─ 反射（SSR/Lumen）+ 天光（SkyLight）环境光
RenderVolumetricCloud                      (:3293/3766/3932/4306)  体积云（可对雾/大气）
RenderSkyAtmosphere                        (:3865)                  大气天空合成
RenderFog                                  (:3903)                  指数高度雾（可盖在云上）
→ RenderTranslucency（半透明后画，半透明物体能被雾/大气影响）
```

> 一句话理解 Sky 四件套：**大气**给天穹上色并做真实大气透视，**体积云**在天上画云，**天光**提供环境间接照明，**雾**做艺术可控的远近衰减。它们互补可叠加，全部不经过 Primitive/材质 mesh 路径。

---

## 5. UPrimitiveComponent 及其派生

**AActor 只是宿主**，它携带 **UPrimitiveComponent**（"摆在世界里的一个引用 + 一堆属性"）。组件通过 `CreateRenderState_Concurrent` 把"变换 + 包围盒 + **材质** + 网格资产引用"快照成一个纯数据的 **FPrimitiveSceneProxy**，经渲染命令队列交给 **FScene** 里的 **FPrimitiveSceneInfo**（进八叉树 + 预烘焙 draw call）；每帧渲染器剔除、排序后提交 GPU。着色由 **UMaterial / UMaterialInstance** 提供的 **FMaterialRenderProxy** 决定。

```text
                游戏线程（UObject 世界）                    渲染线程（数据世界）
AActor
  └─ UPrimitiveComponent  ── CreateSceneProxy() ──▶  FPrimitiveSceneProxy
        │ 引用                                    │     （纯数据快照，无 UObject）
        │  UStaticMesh ─────────────▶ 网格顶点/索引/RenderData
        │  UMaterialInstance ───────▶  FMaterialRenderProxy + UniformBuffer
        └─ Bounds / Transform ─────▶  FPrimitiveSceneInfo
                                            │  插入 PrimitiveOctree（剔除）
                                            │  生成 FStaticMeshBatch
                                            └─▶ CacheMeshDrawCommands → CachedDrawLists
                                                          │
                                             每帧由 MeshPassProcessor 提交 GPU
```

### 5.1 继承链与声明

```text
UObject
 └─ UActorComponent            // 有属主、生命周期钩子、可注册
     └─ USceneComponent        // 有变换、父子挂接、包围盒
         └─ UPrimitiveComponent  ← 一切"能画/能撞"的组件的基类
```

声明于 [PrimitiveComponent.h:306-307](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L306)：

```cpp
UCLASS(abstract, HideCategories=(Mobility, VirtualTexture), ShowCategories=(PhysicsVolume), MinimalAPI)
class UPrimitiveComponent : public USceneComponent, public INavRelevantInterface,
    public IInterface_AsyncCompilation, public IPhysicsComponent,
    public FRenderAssetOwnerStreamingState, public IPhysicsBodyInstanceOwner, ...
```

注意它同时是多个接口的**实现者**：导航（`INavRelevantInterface`）、物理（`IPhysicsComponent`、`IPhysicsBodyInstanceOwner`）、贴图流送（`FRenderAssetOwnerStreamingState`）——这也是为什么它"既是渲染单元又是碰撞单元"。

### 5.2 职责拆分

| 职责域 | 关键成员/方法（文件:行） | 说明 |
|---|---|---|
| 渲染代理 | `virtual FPrimitiveSceneProxy* CreateSceneProxy()` [PrimitiveComponent.h:2395](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2395) | **虚函数**，基类返回 NULL，由子类 new 出代理 |
| 渲染代理指针 | `FPrimitiveSceneProxy* SceneProxy` [PrimitiveComponent.h:2130](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2130) | 组件持有的渲染线程侧代理（快照） |
| 渲染状态入口 | `virtual void CreateRenderState_Concurrent(FRegisterComponentContext*)` [PrimitiveComponent.h:2547](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2547) | 注册时把组件送进场景 |
| 生命周期钩子 | `OnRegister()` [PrimitiveComponent.h:2549](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2549)、`OnUnregister()` [:2550](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2550) | 挂载/卸载 |
| 变换更新 | `virtual void SendRenderTransform_Concurrent()` [:2548](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L2548) | 移动后把新变换同步给代理 |
| 材质接口 | `GetMaterial/SetMaterial` [:1566/:1600](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L1566)、`CreateAndSetMaterialInstanceDynamic` [:1615](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L1615) | 注意 `GetMaterials()` 数组在 UMeshComponent 上 |
| 碰撞 | `FBodyInstance BodyInstance` | Chaos/PhysX 碰撞体包装，驱动 Overlap/Trace/Physics |
| 包围盒 | 继承自 `USceneComponent`（`Bounds`），`UpdateBounds()` 在此被 override | 剔除的输入 |

> **关键认知**：组件**不持有任何渲染数据**（顶点、索引、GPU 缓冲都不在组件上），它只持有**资产引用**（`UStaticMesh`、`USkeletalMesh`）和**属性**（变换、材质插槽、阴影开关、可见性、碰撞配置）。真正的渲染数据在资产和渲染线程的代理里。这也是"组件 → 资产"两层分离的架构根源。

### 5.3 渲染相关属性速查

渲染标志位（全部是 `uint8 X : 1` 位域，UPROPERTY 可在编辑器调整）：

| 属性 | 位置 | 作用 |
|---|---|---|
| `bRenderInMainPass` | [:481](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L481) | 是否在 Main Pass（BasePass）中绘制 |
| `bRenderInDepthPass` | [:484](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L484) | 是否写入深度 Pass |
| `bReceivesDecals` | [:489](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L489) | 是否接受贴花 |
| `CastShadow` | [:548](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L548) | 总开关；子项 `bCastDynamicShadow` [:568](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L568)、`bCastStaticShadow` [:571](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L571)、`bCastContactShadow` [:587](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L587)、`bSelfShadowOnly` [:594](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L594)、`bCastHiddenShadow` [:622](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L622)、`bCastShadowAsTwoSided` [:626](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L626) | 阴影细分开关 |
| `bUseAsOccluder` | [:513](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L513) | 是否参与遮挡剔除 |
| `bRenderCustomDepth` | [:712](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L712) | 输出自定义深度（后处理描边用） |
| `bVisibleInSceneCaptureOnly` / `bHideInSceneCapture` | [:715](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L715) | SceneCapture 可见性 |
| `MinDrawDistance` / `LDMaxDrawDistance` / `CachedMaxDrawDistance` | [:327-344](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L327) | 距离剔除（LOD 距离） |
| `bNeverDistanceCull` | [:387](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L387) | 禁用距离剔除 |
| `DepthPriorityGroup` | [:348](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L348) | 半透明等深度组排序 |
| `IndirectLightingCacheQuality` / `LightmapType` | [:355](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L355) | 间接光缓存质量 / 光照贴图类型 |
| 光照通道 | `SetLightingChannels` [:421](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L421) | 三通道光照，决定被哪些灯照亮 |

这些属性几乎都以同样的模式工作：**修改 → `MarkRenderStateDirty()` → 触发 `CreateRenderState_Concurrent` 重建代理**（见 [PrimitiveComponent.cpp](Source/Runtime/Engine/Private/Components/PrimitiveComponent.cpp) 中大量 `SetXxx` 函数的结尾）。

### 5.4 派生类树

```text
UPrimitiveComponent
 ├─ UMeshComponent                          // 材质插槽 OverrideMaterials + GetMaterials()
 │   ├─ UStaticMeshComponent                // 持有 UStaticMesh 资产（.133）
 │   │   ├─ UInstancedStaticMeshComponent   // 实例化渲染（同一网格画 N 份）
 │   │   │   └─ UHierarchicalInstancedStaticMeshComponent  // HISM：层级剔除 + LOD
 │   │   │       └─ UFoliageInstancedStaticMeshComponent   // 植被
 │   │   ├─ USplineMeshComponent            // 沿样条变形
 │   │   └─ (Landscape 相关派生)
 │   └─ USkinnedMeshComponent               // 持有 USkeletalMesh（.279）
 │       └─ USkeletalMeshComponent          // 动画：UAnimInstance（.399）
 │           └─ (UPhysicsAsset 驱动的物理/布娃娃等)
 ├─ UShapeComponent                         // 简单几何碰撞 + 调试渲染
 │   ├─ USphereComponent / UCapsuleComponent / UBoxComponent
 ├─ UTextRenderComponent / UBillboardComponent / UArrowComponent / UDecalComponent …
```

> **注**：灯光（`ULightComponent`）和天空（`USkyAtmosphereComponent` 等）**不属于** UPrimitiveComponent，已在本文档 §3、§4 单独展开——它们各有独立的代理与场景结构，不共用八叉树/材质 mesh 路径。

> ULightComponent 在早期版本属于 UPrimitiveComponent 的派生

各分支"新增职责"一句话：

- **UStaticMeshComponent**：加一个 `UStaticMesh*` 引用（[StaticMeshComponent.h:133](Source/Runtime/Engine/Classes/Components/StaticMeshComponent.h#L133)），创建 `FStaticMeshSceneProxy`；
- **UInstancedStaticMeshComponent**：在组件上存 `TArray<FTransform> PerInstanceSMData`，proxy 里把所有实例合进一个 draw call；
- **USkeletalMeshComponent**：在 Skinned 基础上加 `AnimClass` / `AnimScriptInstance`（[SkeletalMeshComponent.h:399](Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h#L399)），每帧跑动画 → 更新骨骼变换 → 提交给 proxy；
- **UShapeComponent**：无资产，直接用数学几何体（球/盒/胶囊）产生碰撞体和调试线框。

> 详细分支与选型见 [MeshComponents.md](MeshComponents.md)。

### 5.5 一个典型派生：FStaticMeshSceneProxy 的诞生

以静态网格为例看"组件 → 代理"的快照过程（[StaticMeshSceneProxy.cpp:224-270](Source/Runtime/Engine/Private/StaticMeshSceneProxy.cpp#L224)）：

```cpp
FStaticMeshSceneProxy::FStaticMeshSceneProxy(const FStaticMeshSceneProxyDesc& InProxyDesc, ...)
  : FPrimitiveSceneProxy(InProxyDesc, InProxyDesc.GetStaticMesh()->GetFName())
  , RenderData(InProxyDesc.GetStaticMesh()->GetRenderData())   // ← 网格 GPU 数据（LOD 的顶点/索引缓冲）
  , OverlayMaterial(InProxyDesc.GetOverlayMaterial())
  , ForcedLodModel(InProxyDesc.ForcedLodModel)
  , bCastShadow(InProxyDesc.CastShadow)
  , bReverseCulling(InProxyDesc.bReverseCulling)
  , MaterialRelevance(InProxyDesc.GetMaterialRelevance(GetScene().GetShaderPlatform()))
  ...
```

可以看到：

- 代理把 `RenderData`（`FStaticMeshRenderData`，即网格每个 LOD 的顶点/索引/UV/切线等 GPU 缓冲）**直接引用**过来；
- 把 `MaterialRelevance`（这个网格所有材质的"相关性"：是否需要半透明、是否按 View 变化等）预计算好；
- 把阴影、LOD、反转剔除等开关**快照**下来。

**代理生命周期内完全禁止解引用 UObject**（渲染线程可能并行 GC），所以构造时就要把一切拷完。

### 5.6 5.8 的新机制：IPrimitiveComponentInterface

5.8 里渲染器访问组件不再直接依赖 `UPrimitiveComponent` 类，而是通过纯虚接口 `IPrimitiveComponentInterface`（[ComponentInterfaces.h:128-200](Source/Runtime/Engine/Classes/Components/ComponentInterfaces.h#L128)）：

```cpp
virtual FPrimitiveSceneProxy* GetSceneProxy() const = 0;   // :138
virtual FPrimitiveSceneProxy* CreateSceneProxy() = 0;      // :167  ← 纯虚版本
virtual void CreateRenderState(FRegisterComponentContext* Context) = 0;  // :142
virtual void MarkRenderStateDirty() = 0;                   // :140
virtual const FBoxSphereBounds& GetBounds() const = 0;     // :146
...
```

`UPrimitiveComponent` 通过 `UE_DECLARE_COMPONENT_ACTOR_INTERFACE(PrimitiveComponent)`（[PrimitiveComponent.h:310](Source/Runtime/Engine/Classes/Components/PrimitiveComponent.h#L310)）实现它。渲染器侧 `BatchAddPrimitivesInternal` 建代理时就走这条口（[RendererScene.cpp:1455](Source/Runtime/Renderer/Private/RendererScene.cpp#L1455)）：`Primitive->GetPrimitiveComponentInterface()->CreateSceneProxy()`。这是 Epic 正在推的"组件接口"解耦模式，让渲染层可以脱离 UObject 类体系工作（利于流送/并发/无 UObject 的渲染表示）。

---

## 6. UMaterial 与 UMaterialInstance

### 6.1 抽象基类 UMaterialInterface

材质系统的统一入口（[MaterialInterface.h](Source/Runtime/Engine/Public/Materials/MaterialInterface.h)）：

| 虚函数 | 位置 | 作用 |
|---|---|---|
| `GetMaterial()`（纯虚） | [:666](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L666) | 返回"根" UMaterial（穿过实例链） |
| `GetRenderProxy()`（纯虚） | [:725](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L725) | 渲染线程可用的 `FMaterialRenderProxy*` |
| `GetPhysicalMaterial()` | [:732](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L732) | 表面物理材质 |
| `GetParameterValue()` | [:1211](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L1211) | 按名/按 GUID 取参数 |
| `GetBlendMode() / GetShadingModels()` | [:1249](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L1249) | 混合模式 / 着色模型 |
| `RecacheUniformExpressions()` | [:1343](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L1343) | 参数变了之后重建 uniform 表达式 |
| `CacheShaders()` | [:1352](Source/Runtime/Engine/Public/Materials/MaterialInterface.h#L1352) | 保证 Shader 已编译 |

**UMaterial 和 UMaterialInstance 都继承它**，所以游戏/渲染代码永远只对着 `UMaterialInterface*` 说话。

### 6.2 UMaterial：资产（表达式图 + 编译产物）

`UMaterial`（[Material.h:448](Source/Runtime/Engine/Public/Materials/Material.h#L448)）是"蓝图式"资产，分两层：

**① 编辑器层（节点图）**——存在 `UMaterialEditorOnlyData` 里（[:325](Source/Runtime/Engine/Public/Materials/Material.h#L325)）：
- 一堆 `UMaterialExpression` 节点（Constant、ScalarParameter、TextureSample、Add、Clamp、Blend…约 273 种），用连接边连成图；
- 图连向材质输出引脚：`BaseColor / Metallic / Roughness / Normal / Opacity / EmissiveColor …`（即 `FMaterialInput`）；
- 还有 `BlendMode`（[:486](Source/Runtime/Engine/Public/Materials/Material.h#L486)）、`ShadingModel`（[:511](Source/Runtime/Engine/Public/Materials/Material.h#L511)）、`bUsedWithSkeletalMesh` 等用途开关。

**② 运行时层（编译产物）**——真正给渲染器的是编译后的东西：
- `FMaterialResource : FMaterial`（[MaterialShared.h:3211](Source/Runtime/Engine/Public/MaterialShared.h#L3211)），**每个（着色平台, 质量级）一份**，存在 `UMaterial::MaterialResources`（[:1264](Source/Runtime/Engine/Public/Materials/Material.h#L1264)）；
- `UMaterial` 自己的渲染代理叫 `FDefaultMaterialInstance : FMaterialRenderProxy`（Material.cpp:414）；
- `StateId`（[:1239](Source/Runtime/Engine/Public/Materials/Material.h#L1239)）是材质"版本号"，用作 DDC 缓存 key——材质一变，StateId 变，所有依赖它的 Shader 缓存全部失效重编。

### 6.3 FMaterial 与 Shader 编译/挑选流程

```text
FMaterial::CacheShaders (MaterialShared.cpp:2969)
  → BuildShaderMapId          // 由材质属性 + 用途 + 平台折叠成唯一 key
  → FMaterialShaderMap::FindId // 内存 → DDC（派生数据缓存）
     ├─ 命中 → 直接加载 FMaterialShaderMap
     └─ 未命中 → BeginCompileShaderMap (MaterialShared.cpp:3714)
              → Translate：把节点图翻译成 HLSL（MaterialTranslator）
              → 交给 GShaderCompilingManager 异步编译 job
              → 编译完成 SetGameThreadShaderMap / SetRenderingThreadShaderMap
              → SetShaderMapsOnMaterialResources 发布给所有实例
```

关键结构：
- **FMaterialShaderMap**（[MaterialShared.h:1630](Source/Runtime/Engine/Public/MaterialShared.h#L1630)）：一份完整编译结果，内部按 **VertexFactory（顶点工厂，如本地位置/骨骼/实例化）** 再分组成 `FMeshMaterialShaderMap`；
- 查找 key 是 **FMaterialShaderMapId**（[:1304](Source/Runtime/Engine/Public/MaterialShared.h#L1304)）；
- 静态参数的每个组合（Static Switch 等）会额外产生一份 Shader 排列（permutation）。

### 6.4 UMaterialInstance：参数覆盖

`UMaterialInstance` 不重画节点图，而是在父材质之上做**参数覆盖**（[MaterialInstance.h](Source/Runtime/Engine/Public/Materials/MaterialInstance.h)）：

- 覆盖值存 UPROPERTY 数组：`ScalarParameterValues` / `VectorParameterValues` / `TextureParameterValues`（[:773/:777/:785](Source/Runtime/Engine/Public/Materials/MaterialInstance.h#L773)），元素如 `FScalarParameterValue{ ParameterInfo, Value, ExpressionGUID }`；
- **ExpressionGUID** 是连接父材质参数节点的钥匙：参数节点有稳定 GUID，实例用 GUID 找到"我覆盖的是父材质里的哪个参数"；
- 渲染侧代理是 `FMaterialInstanceRenderProxy : FMaterialRenderProxy`（[MaterialInstanceSupport.h:207](Source/Runtime/Engine/Private/Materials/MaterialInstanceSupport.h#L207)），把覆盖参数写进哈希表，并实现 `GetParameterValue` 命中覆盖否则回退父级；
- `UMaterialInstance::RenderProxy`（[MaterialInstance.h:868](Source/Runtime/Engine/Public/Materials/MaterialInstance.h#L868)）持有它。

### 6.5 MIC vs MID

| | UMaterialInstanceConstant (MIC) | UMaterialInstanceDynamic (MID) |
|---|---|---|
| 生命周期 | 资产，可存盘（.uasset） | 运行时临时，`Create()` 创建，不可存盘 |
| 静态参数 | ✅ 可覆盖（触发重编译，产生新 permutation） | ❌ 不能（`HasOverridenBaseProperties` 恒 false） |
| 基础属性（BlendMode 等） | ✅ 可覆盖 | ❌ |
| 典型用途 | 换肤、材质的预制变体 | 运行时特效参数（血条、冰冻、受伤闪红） |
| 关键方法 | 编辑期 API 带 EditorOnly 后缀 | `SetScalarParameterValue / SetVectorParameterValue` |

MID 的运行时更新调用链（这是"改一个数 → GPU 看到"的完整路径）：

```text
SetVectorParameterValue (MaterialInstanceDynamic.h:109)
  → UMaterialInstance::SetVectorParameterValueInternal (MaterialInstance.cpp:4156)  // 写数组
  → GameThread_UpdateMIParameter (MaterialInstance.cpp:608)
      └─ ENQUEUE_RENDER_COMMAND
          → RenderProxy->RenderThread_UpdateParameter (MaterialInstance.cpp:628)
          → RenderProxy->CacheUniformExpressions()
```

### 6.6 参数如何传到 GPU（UniformExpressionSet）

- 材质里的表达式（参数、常量、Time 等）在编译期折叠成 **FUniformExpressionSet**（[MaterialShared.h:694](Source/Runtime/Engine/Public/MaterialShared.h#L694)）；
- `FMaterialRenderProxy::EvaluateUniformExpressions`（[MaterialRenderProxy.h:121](Source/Runtime/Engine/Public/Materials/MaterialRenderProxy.h#L121)）把**当前参数值**填入 uniform buffer（`FillUniformBuffer` [:710](Source/Runtime/Engine/Public/MaterialShared.h#L710)）；
- 参数一变就 `InvalidateUniformExpressionCache`（[MaterialRenderProxy.h:142](Source/Runtime/Engine/Public/Materials/MaterialRenderProxy.h#L142)），下一帧重建 buffer。

> 一句话：**UMaterial = 编译好的 Shader + 表达式图；MIC/MID = 同一份 Shader 上不同参数的一组值**。渲染器对每个材质持有 `FMaterialRenderProxy`，换实例只换参数、不换代码。

---

## 7. 它们如何在同一帧里协作：渲染管线

### 7.1 一张时序图：物体入场景

```text
游戏线程                                            渲染线程
────────                                            ────────
AActor 放置/激活
  └ UActorComponent::RegisterComponent()
      └ RegisterComponentWithWorld()        (ActorComponent.cpp:1967/2079)
          └ ExecuteRegisterEvents()         (ActorComponent.cpp:2510)
              ├ OnRegister()                (PrimitiveComponent.cpp:670)
              └ CreateRenderState_Concurrent()   ★
                  ├ UpdateBounds()
                  └ World->Scene->AddPrimitive(this)   (PrimitiveComponent.cpp:648)
                      └ FScene::AddPrimitive (RendererScene.cpp:1341)
                          └ BatchAddPrimitivesInternal (RendererScene.cpp:1382)
                              ├ CreateSceneProxy() → FPrimitiveSceneProxy     ← 快照
                              ├ new FPrimitiveSceneInfo; Proxy->SceneInfo = ...
                              └ ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand) ─┐
                                                          ◀──────────────────┘ 命令队列
                                                          ├ Proxy->SetTransform
                                                          ├ Proxy->CreateRenderThreadResources
                                                          └ AddPrimitiveSceneInfo_RenderThread
                                                              └ FPrimitiveSceneInfo::AddToScene
                                                                  ├ PrimitiveOctree.AddElement  (剔除)
                                                                  ├ 填充并行数组（GPUScene）
                                                                  └ AddStaticMeshes → CacheMeshDrawCommands → CachedDrawLists
```

（灯光入场景走 `FScene::AddLight`（RendererScene.cpp:1977），Camera 不"入场景"而每帧生成 View——见 §2、§3。）

### 7.2 游戏线程：注册组件

`AActor` 本身不直接进场景，**进场景的是它携带的 UPrimitiveComponent**。注册链路：

1. `RegisterComponentWithWorld()`（[ActorComponent.cpp:1967](Source/Runtime/Engine/Private/Components/ActorComponent.cpp#L1967)）→ `ExecuteRegisterEvents()`（[:2510](Source/Runtime/Engine/Private/Components/ActorComponent.cpp#L2510)）先调 `OnRegister()` 再调 `CreateRenderState_Concurrent()`（[:2526](Source/Runtime/Engine/Private/Components/ActorComponent.cpp#L2526)）；
2. `UPrimitiveComponent::CreateRenderState_Concurrent`（[PrimitiveComponent.cpp:620](Source/Runtime/Engine/Private/Components/PrimitiveComponent.cpp#L620)）：

```cpp
Super::CreateRenderState_Concurrent(Context);
UpdateBounds();
if (ShouldComponentAddToScene() && SceneProxy == nullptr)   // 没隐藏 + DetailMode 允许
{
    if (Context)  Context->AddPrimitive(this);               // 批量注册路径
    else          GetWorld()->Scene->AddPrimitive(this);     // 单个注册路径
}
```

`ShouldComponentAddToScene()` 检查 `bHiddenInGame`、DetailMode、`IsRegistered` 等——不满足就不进场景（例如纯碰撞无渲染的组件）。

### 7.3 生成代理：FScene::AddPrimitive

`FScene::AddPrimitive`（[RendererScene.cpp:1341](Source/Runtime/Renderer/Private/RendererScene.cpp#L1341)）→ `BatchAddPrimitivesInternal`（[:1382](Source/Runtime/Renderer/Private/RendererScene.cpp#L1382)），在**游戏线程**上完成"快照 + 排队"：

1. **创建代理**（[:1453-1462](Source/Runtime/Renderer/Private/RendererScene.cpp#L1453)）：若 `GetSceneProxy()` 为空，调 `GetPrimitiveComponentInterface()->CreateSceneProxy()`（即 5.6 的新接口，落到各子类的 `CreateSceneProxy`，例如 `FStaticMeshSceneProxy`）；
2. **创建 FPrimitiveSceneInfo**（[:1471](Source/Runtime/Renderer/Private/RendererScene.cpp#L1471)）：`new FPrimitiveSceneInfo(Primitive, this)`，并把 `Proxy->PrimitiveSceneInfo = SceneInfo` 挂回去；
3. **缓存变换/包围盒**（[:1475-1489](Source/Runtime/Renderer/Private/RendererScene.cpp#L1475)）：`RenderMatrix = GetRenderMatrix()`、`WorldBounds = Bounds`、`AttachmentRootPosition`，还有 `FMotionVectorSimulation::Get().GetPreviousTransform()`（上一帧变换，用于运动矢量/速度）；
4. **ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)**（[:1507-1518](Source/Runtime/Renderer/Private/RendererScene.cpp#L1507)）：

```cpp
ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)(
  [this, CreateCommands = MoveTemp(CreateCommands)](FRHICommandListBase& RHICmdList)
{
  for (const FCreateCommand& C : CreateCommands)
  {
    C.PrimitiveSceneProxy->SetTransform(RHICmdList, ...);          // 设 LocalToWorld
    C.PrimitiveSceneProxy->CreateRenderThreadResources(RHICmdList); // 建 GPU 资源
    AddPrimitiveSceneInfo_RenderThread(C.PrimitiveSceneInfo, ...);  // 真正入场景
  }
});
```

> `ENQUEUE_RENDER_COMMAND` 是游戏线程 → 渲染线程的标准通道：lambda 被投进**渲染命令队列**，渲染线程按序取出执行。游戏线程绝不直接碰渲染线程的数据，反之亦然。Camera/Light/Sky 的"进场景"走的是同一通道的不同命令（AddLight 的 `BatchAddLightsCommand`、View 的每帧创建）。

### 7.4 渲染线程：入场景结构

`FPrimitiveSceneInfo::AddToScene`（[PrimitiveSceneInfo.cpp:1822](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1822)）：

1. **间接光照缓存**：为需要未烘焙光照预览的静态网格建 ILC uniform buffer（[:1827-1855](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1827)）；
2. **插入八叉树**（[:1917-1934](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1917)）：

```cpp
Scene->PrimitiveOctree.AddElement(FPrimitiveSceneInfoCompact(SceneInfo));  // :1930
```

   `PrimitiveOctree` 是剔除的核心：每帧视锥/距离/遮挡测试都先查八叉树，命中才进渲染。Nanite 网格在低端平台可以跳过八叉树（`bSkipNaniteInOctree`）。
3. **填充并行数组**：`PrimitiveSceneProxies / PrimitiveTransforms / PrimitiveBounds / PrimitiveOcclusionBounds / PrimitiveComponentIds`——这些 SoA（结构数组）是 GPU Scene 的输入，供 GPU 端按 index 查数据；
4. **静态网格入缓存**：`AddStaticMeshes`（[:1604](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1604)）→ `Proxy->DrawStaticElements()`（[:1617](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1617)）产出 `FStaticMeshBatch` 存进 `SceneInfo->StaticMeshes`，随后 `CacheMeshDrawCommands`（[:1655](Source/Runtime/Renderer/Private/PrimitiveSceneInfo.cpp#L1655)）按 `EMeshPass` 预编译成 `FMeshDrawCommand` 写入 `Scene->CachedDrawLists`。

`FPrimitiveSceneInfo` 是渲染线程侧"一个可渲染物体"的完整代表（[PrimitiveSceneInfo.h:269](Source/Runtime/Renderer/Public/PrimitiveSceneInfo.h#L269)）：

| 成员 | 含义 |
|---|---|
| `FPrimitiveSceneProxy* Proxy`（:277） | 反向指回代理 |
| `FPrimitiveComponentId PrimitiveComponentId`（:283） | **两端互相识别的 ID**（游戏线程/渲染线程都认得它） |
| `TArray<FStaticMeshBatch> StaticMeshes` | 静态路径产出的批次 |
| `FScene* Scene`（:361） | 所属场景 |
| `int32 PackedIndex`（:598） | 在并行数组里的槽位 |
| `FOctreeElementId OctreeId` | 在八叉树里的位置 |

### 7.5 静态路径 vs 动态路径

| | 静态路径 | 动态路径 |
|---|---|---|
| 代理方法 | `DrawStaticElements` [PrimitiveSceneProxy.h:439](Source/Runtime/Engine/Public/PrimitiveSceneProxy.h#L439) | `GetDynamicMeshElements` [PrimitiveSceneProxy.h:504](Source/Runtime/Engine/Public/PrimitiveSceneProxy.h#L504) |
| 调用时机 | 入场景时调用一次 | 每帧对每个可见 View 调用 |
| 产物 | `FStaticMeshBatch` → 缓存的 `FMeshDrawCommand`，跨帧复用 | `FMeshElementCollector`，每帧重新收集 |
| 适用 | 静态光照的 StaticMesh（LOD 内） | 可移动组件、粒子、粒子发射器、编辑器预览 |
| 决定性条件 | `SupportsCachingMeshDrawCommands()` [PrimitiveSceneProxy.h:1868](Source/Runtime/Engine/Public/PrimitiveSceneProxy.h#L1868) + `GetViewRelevance` 报告为"静态" | 其余 |

`GetViewRelevance`（[:542](Source/Runtime/Engine/Public/PrimitiveSceneProxy.h#L542)）还返回是否需要阴影、是否可被遮挡、是否只在编辑器显示等，是渲染器决定"这个物体进哪些 pass"的依据。`FStaticMeshSceneProxy` 两者都实现了（[StaticMeshSceneProxy.cpp:1402/1705](Source/Runtime/Engine/Private/StaticMeshSceneProxy.cpp#L1402)）：常态走静态缓存，未烘焙光照预览/强制 LOD 等特殊情况走动态。

### 7.6 渲染线程看到的世界：FScene 数据结构

`FScene`（[RendererScene.cpp:1136](Source/Runtime/Renderer/Private/RendererScene.cpp#L1136) 构造，创建时 `World->Scene = this`）是渲染线程持有的"整个世界"：

```text
FScene
 ├─ PrimitiveOctree（八叉树）            → 剔除（视锥/距离/遮挡）
 ├─ 并行 SoA 数组                        → GPU Scene 上传（Index → Proxy/Bounds/Transform）
 ├─ StaticMeshes（稀疏数组）+ CachedDrawLists  → 预编译 draw call 缓存
 ├─ Lights（FLightSceneInfo* 数组）      → 灯光（含 FLightSceneInfoCompact 紧凑视图）
 ├─ SkyAtmosphere / VolumetricCloud 场景信息
 ├─ PrimitiveSceneProxies / PrimitiveSceneInfos
 └─ GPUScene：把实例数据（变换、自定义数据等）打包上传给 GPU
```

**同步模型**：游戏线程每帧通过 `ENQUEUE_RENDER_COMMAND` 命令队列灌入增删改（`AddPrimitive` / `UpdatePrimitiveTransform` / `ReleasePrimitive` / `AddLight` / 材质替换），渲染线程按序消费；两端靠 **PrimitiveComponentId**（灯光靠 `FLightSceneInfo::Id`）互相识别，谁都不直接解引用对方的对象（渲染线程严禁碰 UObject，避免与 GC 竞争）。

### 7.7 最后一公里：从 FStaticMeshBatch 到 GPU DrawCall

1. **FStaticMeshBatch**（入场景时产生）：记录"哪个代理、哪个 LOD、哪组材质、哪个 VertexFactory、哪种 MeshPass"；
2. **CacheMeshDrawCommands**：把 batch 交给对应 pass 的 **FMeshPassProcessor**（如 BasePass、ShadowDepth），由它调用 `FMeshMaterialShader` 绑定 Shader + uniform buffer + 顶点工厂输入，产出不可变的 **FMeshDrawCommand**；
3. 命令按 pass 分类进 `Scene->CachedDrawLists`，支持按 `bUseAsOccluder`、材质、LOD 等排序、合并；
4. 每帧渲染器只遍历**可见**的 `FPrimitiveSceneInfo`（八叉树剔除后），把其 `FMeshDrawCommand` 从缓存里捞出提交给 RHI → GPU。

**这就是"预烘焙 draw call"的核心收益**：只要物体没动、材质没变、光照相关状态没变，它的 draw call 命令是现成的，每帧只是"提交缓存"，而不是"重新生成命令"。这也是为什么"静态光照 + 静态网格"性能最好、而可移动物体（动态路径）每帧都要重新收集网格元素。

### 7.8 每帧的完整 Pass 顺序

把 §2–§6 的五要素放回 `FDeferredShadingSceneRenderer::Render()`（[DeferredShadingRenderer.cpp:1832](Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp#L1832)）看它们如何协作（各 Pass 详解见 [FDeferredShadingSceneRenderer.md](../../Rendering/FDeferredShadingSceneRenderer.md)）：

```text
0. InitViews（Camera 视角 → 视锥剔除 → 可见集合 + 灯光收集）     ← Camera ★
1. RenderPrePass（Early-Z 深度）
2. RenderShadowDepthMaps（Light 的阴影图 / VSM）                 ← Light
3. RenderBasePass（Primitive + Material → GBuffer）              ← Primitive + Material
4. RenderVelocities
5. 间接光 + RenderLights（直接光照）                             ← Light
6. RenderDeferredReflectionsAndSkyLighting（反射 + 天光环境）     ← Sky(SkyLight) ★
7. RenderVolumetricCloud / RenderSkyAtmosphere / RenderFog       ← Sky(大气/云/雾) ★
8. RenderTranslucency（半透明）
9. AddPostProcessingPasses（后处理）
```

> 一句话总结五要素的分工：**Camera 选视角 → 决定画什么；Primitive+Material 决定表面数据；Light 决定直接光；Sky 决定环境光/天空/雾**。渲染器把这一切编排成上面的固定 Pass 顺序。

---

## 8. 关键概念速查表

| 概念 | 是什么 | 属于哪一侧 |
|---|---|---|
| AActor | 世界里的一个"实体"容器，携带组件 | 游戏线程 |
| UCameraComponent / FMinimalViewInfo | 相机组件 / 一帧的视角参数包 | 游戏线程 |
| FSceneView / FViewInfo | 引擎层视图 / 渲染器层视图（ViewMatrices+ViewRect+视锥） | 渲染线程 |
| ULightComponent（及派生） | 光的组件（平行/点/聚/矩形） | 游戏线程 |
| FLightSceneProxy / FLightSceneInfo | 光的渲染快照 / 已入场景的光 | 渲染线程 |
| USkyAtmosphereComponent / UVolumetricCloudComponent / UExponentialHeightFogComponent | 大气 / 云 / 高度雾组件 | 游戏线程 |
| USkyLightComponent | 天光组件（环境光来源） | 游戏线程 |
| UPrimitiveComponent | 可渲染/可碰撞的最小单元，持有资产引用 + 属性 | 游戏线程 |
| FPrimitiveSceneProxy | 组件的**纯数据快照**，渲染线程唯一可用的表示 | 渲染线程 |
| FPrimitiveSceneInfo | 一个已入场景物体的代表，挂代理 + 批次 + ID | 渲染线程 |
| FScene | 渲染线程的整个世界（八叉树 + 数组 + 灯光 + 缓存） | 渲染线程 |
| PrimitiveComponentId | 两端识别同一个物体的 ID | 两线程共享 |
| UMaterialInterface / UMaterial / UMaterialInstance | 材质抽象 / 资产(Shader+图) / 参数覆盖 | 游戏线程（资产） |
| FMaterialRenderProxy | 渲染线程用的材质代理，负责 uniform buffer | 渲染线程 |
| FMaterialShaderMap / FMeshDrawCommand | 编译好的 Shader 集合 / 一条可提交的 draw call | 渲染线程 |
| ENQUEUE_RENDER_COMMAND | 游戏线程 → 渲染线程的命令队列投递宏 | 通道 |

---

## 9. 延伸 FAQ

**Q：为什么组件不直接持有顶点数据？**
A：因为数据是资产（UStaticMesh），组件只是"引用 + 摆放"。一个网格可以被成百上千个组件引用；如果把数据拷进每个组件，内存爆炸。渲染线程的 proxy 引用同一个 `RenderData`，只多快照"这个组件自己的属性"。

**Q：改材质参数为什么要走命令队列？**
A：参数数组在游戏线程被改（`SetVectorParameterValueInternal`），但 uniform buffer 在渲染线程持有。游戏线程必须把"新值"投递给渲染线程（`RenderThread_UpdateParameter`），否则两个线程同时读写同一 buffer 会数据竞争。同时游戏线程不能等渲染线程，所以是异步投递 + 下一帧生效。

**Q：为什么静态网格渲染快？**
A：`FMeshDrawCommand` 是**不可变、可跨帧复用**的（7.7）。可移动网格每帧走 `GetDynamicMeshElements` 重新收集、重新打包，代价高。这也解释了为什么 UE 强烈建议"静态光照 + 静态网格 + 少用可移动物体"。

**Q：`bCastShadow` 那么多子开关怎么影响渲染？**
A：每个开关决定是否在某个 pass（ShadowDepth / 阴影级联 / 远距阴影）里生成该物体的 draw call。全关的物体不产出阴影命令，节省 shadow pass 成本。

**Q：相机到底在渲染里干了几件事？**
A：三件：①提供视锥做**剔除**（决定渲染量）；②提供 ViewMatrices 做**坐标变换**（世界→屏幕）；③携带后处理设置。相机自身不产生 draw call，但它决定了"渲染器这帧看到什么"。

**Q：为什么灯光要单独一套代理而不是走 Primitive？**
A：光的场景需求与网格完全不同——没有顶点/包围盒/材质，但有衰减半径、圆锥角、IES、shadow map 需要、通道掩码。若让光走 UPrimitiveComponent 的八叉树+材质路径，全是浪费。所以 `ULightComponentBase` 独立继承 `USceneComponent`，配 `FLightSceneProxy` + `FLightSceneInfo`（见 §3.1 修正）。

**Q：天空四件套和 Primitive 有什么关系？**
A：没有直接关系。大气/云/雾组件都不是 UPrimitiveComponent（各自直接继承 USceneComponent），不产 mesh draw call，每帧由独立的 Pass 渲染（RenderSkyAtmosphere / RenderVolumetricCloud / RenderFog）。天光虽然继承 `ULightComponentBase`，也不走普通延迟光照，而是进环境光/反射通路（§4.4）。

**Q：5.8 相比老版本的流程有什么变化？**
A：主要是渲染器通过 `IPrimitiveComponentInterface` 访问组件（5.6），以及 `AddPrimitive` 支持批量（`BatchAddPrimitivesInternal` + `Context->AddPrimitive`），提升大规模关卡加载时的注册效率。核心的"代理快照 + 命令队列 + 八叉树"模型没有变。
