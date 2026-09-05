# Niagara 模块代码文件总览

> 本文梳理 Unreal Engine 中 Niagara 相关 C++ 模块的组织方式，以及各模块主要代码文件的作用。
> 源码基准路径：`Engine/Plugins/FX/Niagara/` 与 `Engine/Plugins/FX/` 下的若干关联插件。
> 说明：结构以当前检出的引擎源码（UE6-main）为准，模块划分与 UE5 主线基本一致。

## 1. 模块总览

Niagara 采用「运行时模块 + 编辑器模块 + 关联插件」的分层结构。核心运行时被拆成多个互有依赖的 C++ 模块，把「渲染无关的核心类型」「Shader 编译」「顶点工厂」与「主运行时」解耦，便于编译并行与依赖收敛。

| 模块 | 路径 | 作用 |
| --- | --- | --- |
| `NiagaraCore` | `Source/NiagaraCore/` | 最底层核心：无 Engine 依赖的基础类型（编译哈希、DI 基类、合并/变更通知） |
| `NiagaraShader` | `Source/NiagaraShader/` | GPU Compute Shader 编译与相关辅助（参数构建、清计数、排序、碰撞 Trace 等） |
| `NiagaraVertexFactories` | `Source/NiagaraVertexFactories/` | 各渲染器的顶点工厂（Sprite/Ribbon/Mesh）与间接绘制 |
| `Niagara` | `Source/Niagara/` | 主运行时：System/Emitter/Component、数据集、Data Interface、渲染器、Data Channel、SimCache 等 |
| `NiagaraEditor` | `Source/NiagaraEditor/` | 完整编辑器：脚本图、HLSL 翻译、工厂、工具集、View Model、Sequencer |
| `NiagaraEditorWidgets` | `Source/NiagaraEditorWidgets/` | 编辑器 Slate 控件（Stack、参数面板等 UI） |
| `NiagaraAnimNotifies` | `Source/NiagaraAnimNotifies/` | 动画通知（AnimNotify）播放/控制 Niagara 特效 |
| `NiagaraBlueprintNodes` | `Source/NiagaraBlueprintNodes/` | Data Channel 的蓝图 K2 节点（读/写） |

关联插件（位于 `Engine/Plugins/FX/`）：

| 插件 | 作用 |
| --- | --- |
| `NiagaraFluids` | 流体（Fluid）仿真 |
| `NiagaraUIRenderer` | 在 UMG/Slate 中渲染 Niagara 粒子 |
| `NiagaraMRQ` | Movie Render Queue 集成的 Data Interface |
| `NiagaraSimCaching` | SimCache 的 Sequencer 轨道集成 |
| `NiagaraGaussianSplat` | 3D 高斯泼溅（Gaussian Splat）渲染 |
| `NiagaraNanite` | Nanite 网格渲染器 |
| `NiagaraLightweight` | 轻量 Stateless Emitter 模板 |
| `NiagaraInsights` | Unreal Insights 性能分析（Trace） |

---

## 2. 核心运行时模块

### 2.1 `NiagaraCore` — 无 Engine 依赖的底层核心

依赖只有 `Core` / `CoreUObject` / `VectorVM` / `RenderCore`，刻意不依赖 `Engine`，使「编译/哈希/合并」这类纯逻辑能脱离编辑器与游戏框架独立存在。

| 文件 | 作用 |
| --- | --- |
| `NiagaraCompileHash.h/.cpp` | 编译结果哈希，用于脚本/图编译的缓存与一致性判断 |
| `NiagaraDataInterfaceBase.h` | 所有 Data Interface 的最底层基类，定义参数布局与编译接口 |
| `NiagaraMergeable.h/.cpp` | 可合并对象基类（Diff/Merge），支撑模块与脚本的合并 |
| `NiagaraNotifyOnChanged.h/.cpp` | 变更通知机制（订阅/广播），编辑器图刷新依赖它 |
| `NiagaraCustomVersion.h/.cpp` | 资产序列化版本号（Custom Serialization） |
| `NiagaraCoreModule.h` | 模块入口 |
| `NiagaraShaderFileHash.cpp` | Shader 源文件哈希（配合 Shader 缓存） |

### 2.2 `Niagara` — 主运行时模块

依赖 `Engine`、`Renderer`、`NiagaraShader`、`NiagaraVertexFactories`、`MovieScene` 等，是 Niagara 的主体。文件可按职责分成以下几组。

#### 2.2.1 系统 / 组件 / 实例（顶层调度）

| 文件 | 作用 |
| --- | --- |
| `Classes/NiagaraSystem.h` / `NiagaraSystemImpl.h` | 系统资产：持有 emitter handle、编译数据、实例参数存储 |
| `Classes/NiagaraEmitter.h` / `NiagaraEmitterBase.h` / `NiagaraEmitterHandle.h` / `NiagaraEmitterImpl.h` | Emitter 资产、句柄、实现 |
| `Classes/NiagaraEffectType.h` | 特效类型资产（LOD、性能预算、Scalability） |
| `Public/NiagaraComponent.h` / `Private/NiagaraComponent.cpp` | 场景入口组件：驱动 SystemInstanceController |
| `Public/NiagaraSystemInstance.h` / `NiagaraSystemInstanceController.h` | 运行时执行核心与控制器抽象 |
| `Private/NiagaraSystemInstance.cpp` | GT / Concurrent / GPU 三阶段 Tick、参数同步、Emitter 执行 |
| `Classes/NiagaraEmitterInstance.h` | Emitter 运行时实例 |
| `Public/NiagaraSystemSimulation.h` / `NiagaraSystemGpuComputeProxy.h` | 批处理仿真与 GPU 代理 |
| `Public/NiagaraWorldManager.h` | 世界级仿真管理器（批处理、LOD、Data Channel 宿主） |
| `Private/NiagaraAsyncCompile.cpp/.h` | 异步编译 |
| `Public/NiagaraComponentPool.h` / `NiagaraComponentPoolMethodEnum.h` | 组件池 |
| `Public/NiagaraCullProxyComponent.h` | 裁剪代理组件 |

#### 2.2.2 数据集 / 参数 / 类型

| 文件 | 作用 |
| --- | --- |
| `Classes/NiagaraDataSet.h` / `NiagaraDataSetCompiledData.h` / `NiagaraDataSetAccessor.h` | 粒子数据集、编译后布局、访问器 |
| `Public/NiagaraParameterStore.h` / `NiagaraParameterBinding.h` / `NiagaraParameters.h` | 参数仓库与绑定 |
| `Public/NiagaraUserRedirectionParameterStore.h` | 用户重定向参数（组件覆盖） |
| `Classes/NiagaraConstants.h` | 内建常量（命名空间、系统参数名） |
| `Public/NiagaraTypes.h` / `NiagaraCommon.h` / `NiagaraDefines.h` | 基础类型、公共定义、宏 |
| `Public/NiagaraTypeRegistry.h` / `NiagaraRuntimeTypeUtilities.h` | 类型注册与运行时工具 |
| `Public/NiagaraVariableMetaData.h` | 变量元数据（编辑器展示信息） |
| `Public/NiagaraVariant.h` | 变体类型容器 |

#### 2.2.3 脚本 / 执行上下文

| 文件 | 作用 |
| --- | --- |
| `Classes/NiagaraScript.h` / `NiagaraScriptSourceBase.h` | 脚本资产与源码基类 |
| `Classes/NiagaraScriptExecutionContext.h` | 脚本执行上下文（VM 字节码执行入口） |
| `Classes/NiagaraComputeExecutionContext.h` | GPU 计算执行上下文 |
| `Public/NiagaraScriptRuntimeCompiledData.h` / `NiagaraScriptRuntimeCookedData.h` / `NiagaraScriptRuntimeData.h` | 脚本运行时编译数据 / 烘焙数据 / 运行时数据 |
| `Public/NiagaraScriptExecutionParameterStore.h` | 脚本执行参数存储 |
| `Private/NiagaraConstants.cpp` | 常量实现 |
| `Public/NiagaraSharedDataUtilities.h` | 共享数据工具 |

#### 2.2.4 Data Interface（数据接口）

Data Interface 是 Niagara 对外部数据的统一读写抽象，分为 `Classes/`（公开）与 `Private/DataInterface/`（部分实现）两处。

- **纹理 / 体数据**：`NiagaraDataInterfaceTexture`、`CubeTexture`、`VolumeTexture`、`2DArrayTexture`、`SparseVolumeTexture`、`VirtualTextureSample`、`RenderTarget2D/2DArray/Cube/Volume`、`IntRenderTarget2D`、`VolumeCache`
- **网格**：`NiagaraDataInterfaceStaticMesh`、`StaticMeshIndirect`、`SkeletalMesh`（含 `NDISkeletalMesh_*Sampling` 采样实现）、`DynamicMesh`、`MeshRendererInfo`、`ArrayMesh`
- **碰撞 / 查询**：`NiagaraDataInterfaceCollisionQuery`、`AsyncGpuTrace`、`NeighborGrid3D`、`NeighborQuery`、`RasterizationGrid3D`、`PhysicsAsset`、`RigidMeshCollisionQuery`
- **网格集合（GPU 仿真网格）**：`Grid2DCollection`、`Grid2DCollectionReader`、`Grid3DCollection`、`Grid3DCollectionReader`
- **曲线 / 噪声 / 场**：`Curve`、`ColorCurve`、`VectorCurve`、`Vector2DCurve`、`Vector4Curve`、`CurlNoise`、`VectorField`
- **音频**：`AudioSpectrum`、`AudioOscilloscope`、`AudioPlayer`
- **场景 / 系统信息**：`Camera`、`SystemProperties`、`EmitterProperties`、`SceneCapture2D`、`GBuffer`、`ParticleRead`、`Occlusion`、`Landscape`、`Spline`
- **数组 / 通用**：`NiagaraDataInterfaceArray*`（Float/Int/NiagaraID/Distribution/Mesh/SpawnBurst）、`DataTable`、`UObjectPropertyReader`、`PropertyInterface`、`Export`、`ConsoleVariable`、`SimpleCounter`、`MemoryBuffer`、`DebugDraw`、`ActorComponent`
- **Data Channel / SimCache**：`NiagaraDataInterfaceDataChannelRead/Write`、`NiagaraDataInterfaceSimCacheReader`
- 基类与工具：`Classes/NiagaraDataInterface.h`、`NiagaraDataInterfaceRW.h`（读写工具）、`NiagaraDataInterfaceUtilities.h`

#### 2.2.5 渲染器（Renderer）

每个渲染器通常对应一份「RendererProperties（资产侧配置）+ Renderer（运行时）+ SceneProxy」的三角结构。

| 文件 | 作用 |
| --- | --- |
| `Public/NiagaraSpriteRendererProperties.h` | 精灵（Sprite）渲染器属性 |
| `Public/NiagaraRibbonRendererProperties.h` | 丝带（Ribbon）渲染器属性 |
| `Public/NiagaraMeshRendererProperties.h` / `NiagaraMeshRendererMeshProperties.h` | 网格（Mesh）渲染器属性 |
| `Public/NiagaraComponentRendererProperties.h` | 组件渲染器属性（把粒子映射为 USceneComponent） |
| `Public/NiagaraLightRendererProperties.h` | 灯光渲染器 |
| `Public/NiagaraDecalRendererProperties.h` | 贴花渲染器 |
| `Public/NiagaraVolumeRendererProperties.h` | 体积渲染器 |
| `Public/NiagaraRenderer.h` / `NiagaraRendererComponents.h` / `NiagaraRendererMeshes.h` / `NiagaraRendererLights.h` / `NiagaraRendererDecals.h` / `NiagaraRendererVolumes.h` | 各渲染器的运行时实现 |
| `Public/NiagaraSceneProxy.h` | 场景代理基类（把仿真数据提交给渲染线程） |
| `Public/NiagaraBoundsCalculator.h` | 包围盒计算 |
| `Public/NiagaraRenderGraphUtils.h` / `NiagaraRenderableMeshInterface.h` / `NiagaraRenderableMeshArrayInterface.h` | 渲染图工具与可渲染网格接口 |

#### 2.2.6 Data Channel（数据通道）

用于系统间 / 系统与外部之间的数据读写通道。

| 文件 | 作用 |
| --- | --- |
| `Public/NiagaraDataChannel.h` / `NiagaraDataChannelCommon.h` / `NiagaraDataChannelPublic.h` | 通道定义与公共类型 |
| `Public/NiagaraDataChannelAsset.h` / `NiagaraDataChannelReference.h` | 通道资产与引用 |
| `Public/NiagaraDataChannelData.h` / `NiagaraDataChannelVariable.h` / `NiagaraDataChannelLayoutInfo.h` | 通道数据、变量、布局 |
| `Public/NiagaraDataChannelHandler.h` / `Private/NiagaraDataChannelManager.cpp/.h` | 通道处理器与世界级管理器 |
| `Public/NiagaraDataChannelAccessor.h` / `NiagaraDataChannelAccessContext.h` | 读/写访问器 |
| `Public/NiagaraDataChannelFunctionLibrary.h` | 蓝图函数库 |
| `Public/NiagaraDataChannel_Global.h` / `_Islands.h` / `_Map.h` / `_GameplayBurst.h` | 通道类型（全局 / 岛屿 / 贴图 / 玩法突发） |

#### 2.2.7 Stateless（无状态发射器）

一套不依赖脚本图、用「模块数组 + 编译生成」的轻量发射器路径。

| 文件 | 作用 |
| --- | --- |
| `Private/Stateless/NiagaraStatelessEmitter.cpp` / `NiagaraStatelessEmitterData.cpp` | 无状态发射器运行时与数据 |
| `Private/Stateless/NiagaraStatelessComputeManager.cpp` | GPU 计算管理 |
| `Private/Stateless/NiagaraStatelessCommon.cpp` | 公共逻辑 |
| `Public/Stateless/NiagaraStatelessDistribution.h` / `NiagaraStatelessRange.h` | 分布与范围类型 |
| `Private/Stateless/Modules/NiagaraStatelessModule_*.cpp` | 各功能模块（重力、拖拽、速度求解、形状定位、缩放、颜色等） |
| `Private/Stateless/Expressions/NiagaraStatelessExpression*.cpp` | 表达式求值（Float/Vec2/Vec3/Vec4/Color） |

#### 2.2.8 SimCache / Baker（仿真缓存 / 烘焙）

| 文件 | 作用 |
| --- | --- |
| `Classes/NiagaraSimCache.h` / `NiagaraSimCacheCapture.cpp/.h` | 仿真缓存资产与捕获 |
| `Classes/NiagaraSimCacheFunctionLibrary.h` | 蓝图函数库 |
| `Classes/NiagaraSimCacheCompare.h` / `NiagaraSimCacheJson.h` / `NiagaraSimCacheCustomStorageInterface.h` | 比较 / JSON / 自定义存储 |
| `Classes/NiagaraBakerSettings.h` / `NiagaraBakerOutput*.h` | 烘焙设置与输出（SimCache/StaticMesh/Texture/SparseVolume/VolumeTexture） |
| `Private/NiagaraBakerOutput*.cpp` | 烘焙输出实现 |
| `Public/NiagaraDataSetReadback.h` | 数据集回读 |

#### 2.2.9 Sequencer / MovieScene

| 文件 | 作用 |
| --- | --- |
| `Public/MovieScene/MovieSceneNiagaraSystemTrack.h` / `MovieSceneNiagaraSystemSpawnSection.h` / `MovieSceneNiagaraTrack.h` | 系统轨道 / 生成段 / 基类轨道 |
| `Public/MovieScene/Parameters/MovieSceneNiagara*ParameterTrack.h` | 各类型参数轨道（Float/Int/Bool/Color/Vector） |

#### 2.2.10 其它运行时支持

| 文件 | 作用 |
| --- | --- |
| `Classes/NiagaraCollision.h` | 碰撞查询 |
| `Public/NiagaraScalabilityManager.h` / `NiagaraScalabilitySettings.h` / `NiagaraScalabilityState.h` | 可伸缩性（Scalability）管理 |
| `Public/NiagaraSettings.h` | 项目级 Niagara 设置 |
| `Public/NiagaraFunctionLibrary.h` | 蓝图/游戏侧函数库 |
| `Public/NiagaraPreviewGrid.h` / `NiagaraEditorPreviewActor.h` | 预览网格与预览 Actor |
| `Public/NiagaraMessageStore.h` / `NiagaraMessageDataBase.h` | 消息存储（编译/验证消息） |
| `Public/NiagaraDebugVis.h` / `NiagaraDebuggerCommon.h` | 调试可视化 |
| `Public/NiagaraComponentSettings.h` | 组件设置 |
| `Classes/NiagaraParameterCollection.h` | 参数集合资产 |
| `Classes/NiagaraLensEffectBase.h` | 镜头效果基类 |
| `Classes/NiagaraOpenVDB.h` / `NiagaraUseOpenVDB.h` / `VolumeCache.h` | OpenVDB 与体积缓存集成 |

---

## 3. Shader 与渲染底层

### 3.1 `NiagaraShader` — GPU 计算 Shader 基础设施

依赖 `RenderCore`、`RHI`、`VectorVM`、`NiagaraCore`。负责 Niagara GPU 仿真的 Shader 类型定义、编译与参数管理。

| 文件 | 作用 |
| --- | --- |
| `NiagaraShader.h` / `NiagaraShaderType.h` | Niagara Shader 与 Shader 类型（`FNiagaraShader`） |
| `NiagaraShaderCompilationManager.h/.cpp` | Shader 编译管理 |
| `NiagaraScriptBase.h` | 脚本 Shader 基类 |
| `NiagaraShared.h` / `NiagaraShaderParticleID.h` | 共享定义 / 粒子 ID 语义 |
| `NiagaraShaderParametersBuilder.h` | Shader 参数构建器（统一 DI 参数绑定） |
| `NiagaraClearCounts.h/.cpp` | GPU 计数清零 |
| `NiagaraBatchedElements.h/.cpp` | 批量元素 |
| `NiagaraGenerateMips.h/.cpp` | Mip 生成 |
| `NiagaraSVTShaders.h/.cpp` | 稀疏体积纹理 Shader |
| `NiagaraRibbonCompute.h/.cpp` | 丝带计算 |
| `NiagaraAsyncGpuTraceProvider.h/.cpp` + `*Gsdf`/`*Hwrt` | 异步 GPU Trace（SDF / HWRT 后端） |
| `NiagaraDistanceFieldHelper.h/.cpp` | 距离场辅助 |
| `NiagaraNeighborQuerySort.h/.cpp` | 邻域查询排序 |
| `NiagaraGPUSceneUtils.h/.cpp` | GPU 场景工具 |
| `NiagaraDebugShaders.h/.cpp` | 调试 Shader |

### 3.2 `NiagaraVertexFactories` — 渲染顶点工厂

依赖 `Engine`、`Renderer`、`RHI`。为各渲染器生成顶点数据并提交绘制。

| 文件 | 作用 |
| --- | --- |
| `NiagaraVertexFactory.h` / `NiagaraSpriteVertexFactory.h` / `NiagaraRibbonVertexFactory.h` / `NiagaraMeshVertexFactory.h` | 通用 / 精灵 / 丝带 / 网格顶点工厂 |
| `NiagaraCutoutVertexBuffer.h/.cpp` | 裁剪顶点缓冲（SubUV 动画） |
| `NiagaraDrawIndirect.h/.cpp` / `NiagaraDispatchIndirect.h/.cpp` | GPU 间接绘制 / 间接分发 |
| `NiagaraSortingGPU.h/.cpp` | GPU 排序 |
| `NiagaraGPURayTracingTransformsShader.h/.cpp` | 光追变换 Shader |

---

## 4. 编辑器模块

### 4.1 `NiagaraEditor` — 完整编辑器

这是最大的编辑器模块，包含脚本图编辑、HLSL 翻译、资产工厂、系统/发射器编辑工具、View Model、Sequencer 集成等。

**脚本图与节点**（`NiagaraNode*`）：

- `NiagaraNode.h` / `NiagaraNodeInput` / `NiagaraNodeOutput` / `NiagaraNodeOp` / `NiagaraNodeFunctionCall` / `NiagaraNodeCustomHlsl` / `NiagaraNodeAssignment` — 图节点体系
- `NiagaraNodeParameterMapGet/Set`、`NiagaraNodeParameterMapFor`、`NiagaraNodeParameterMapBase` — 参数映射节点
- `NiagaraNodeIf` / `NiagaraNodeSelect` / `NiagaraNodeStaticSwitch` / `NiagaraNodeUsageSelector` / `NiagaraNodeSimTargetSelector` / `NiagaraNodeOutputTag` / `NiagaraNodeConvert` / `NiagaraNodeReroute` / `NiagaraNodeEmitter` / `NiagaraNodeReadDataSet` / `NiagaraNodeWriteDataSet` — 控制流与工具节点
- `EdGraphSchema_Niagara.h` — 图 Schema
- `NiagaraGraph.h` / `NiagaraGraphDataCache.h` / `NiagaraGraphDigest.h` — 脚本图与图缓存
- `NiagaraNodeFactory.h` — 节点工厂

**编译 / 翻译**：

- `NiagaraHlslTranslator.h/.cpp` — 图 → HLSL 翻译核心
- `NiagaraCompiler.h/.cpp` / `INiagaraCompiler.h` — 编译器（VM 字节码 / GPU Shader）
- `NiagaraCompilationBridge.h` / `NiagaraCompilationDigestBridgeImpl.cpp` / `NiagaraCompilationGraphBridgeImpl.cpp` / `NiagaraCompilationTasks.cpp` — 编译桥接与任务
- `NiagaraSystemCompilingManager.h/.cpp` — 系统编译管理
- `NiagaraParameterMapHistory.h/.cpp` — 参数映射历史（编译关键）
- `NiagaraDigestDatabase.h/.cpp` — 编译摘要数据库

**资产工厂与定义**：

- `NiagaraSystemFactoryNew` / `NiagaraEmitterFactoryNew` / `NiagaraScriptFactoryNew` / `NiagaraEffectTypeFactoryNew` / `NiagaraParameterCollectionFactoryNew` / `NiagaraParameterDefinitionsFactory` / `NiagaraSimCacheFactoryNew` / `NiagaraValidationRuleSetFactoryNew` — 各类资产工厂
- `NiagaraAssetDefinition`（`AssetDefinitions/` 子目录）— 资产定义（UE5.1+ 资产系统）

**系统 / 发射器编辑**：

- `NiagaraSystemEditorData.h` / `NiagaraEmitterEditorData.h` — 编辑器数据
- `NiagaraEditorModule.h/.cpp` — 编辑器模块入口
- `Toolkits/` — 编辑器工具集（System/Emitter/ParameterCollection 工具集）
- `ViewModels/` — 堆栈（Stack）、参数面板等 View Model
- `NiagaraStackEditorData.h` / `NiagaraStackFunctionInputBinder.h` — 堆栈编辑数据与输入绑定
- `NiagaraValidationRules.h` — 验证规则
- `NiagaraClipboard.h` — 剪贴板（复制粘贴模块）

**其它工具**：

- `NiagaraBakerRenderer.cpp` / `NiagaraBakerFunctionLibrary.cpp` — 烘焙渲染
- `NiagaraThumbnailRenderer.cpp` — 缩略图
- `NiagaraDebugger.cpp` / `NiagaraOutliner.cpp` — 调试器 / 大纲
- `NiagaraScriptMergeManager.cpp` — 脚本合并
- `NiagaraConvertInPlaceEmitterAndSystemState.cpp` — 就地转换
- `Sequencer/`、`Preview/`、`Utilities/`、`Customizations/`、`Tests/` — Sequencer 集成、预览、工具、细节面板定制、测试
- `NiagaraSimCacheEditorUtils.cpp` / `NiagaraEditorSimCacheUtils.cpp` — SimCache 编辑器工具

### 4.2 `NiagaraEditorWidgets` — 编辑器 Slate 控件

依赖 `NiagaraEditor`，把编辑器 UI 拆到独立模块，便于按需加载（编辑器启动优化）。

- `Widgets/` — 参数面板、堆栈项等 Slate 控件
- 与 `NiagaraEditor` 共享部分头文件（`Public/` 与 `Private/` 下可见同名的图节点、Schema 等头文件，供跨模块编译引用）

---

## 5. 关联插件

### 5.1 `NiagaraAnimNotifies` — 动画通知

| 文件 | 作用 |
| --- | --- |
| `AnimNotify_PlayNiagaraEffect.h/.cpp` | 一次性播放 Niagara 特效的动画通知 |
| `AnimNotifyState_TimedNiagaraEffect.h/.cpp` | 持续一段时间的特效通知状态 |

### 5.2 `NiagaraBlueprintNodes` — 蓝图节点

为 Data Channel 提供蓝图 K2 节点。

| 文件 | 作用 |
| --- | --- |
| `K2Node_DataChannelBase.h` / `K2Node_DataChannel_WithContext.h` | 节点基类 / 带上下文节点 |
| `K2Node_ReadDataChannel.h` / `K2Node_WriteDataChannel.h` | 读取 / 写入数据通道节点 |
| `NiagaraBlueprintUtil.h` | 蓝图工具 |

### 5.3 `NiagaraFluids` — 流体仿真

极简插件，主体是让流体相关资产/模块注册进 Niagara。

| 文件 | 作用 |
| --- | --- |
| `INiagaraFluids.h` / `NiagaraFluids.cpp` | 流体模块接口与入口 |

### 5.4 `NiagaraUIRenderer` — UI 渲染

让 Niagara 粒子直接渲染到 UMG/Slate（作为 UI 元素）。

| 文件 | 作用 |
| --- | --- |
| `NiagaraUIComponent.h` / `NiagaraUIComponent.cpp` | UI 场景组件（粒子宿主） |
| `NiagaraUIRendererProperties.cpp` / `NiagaraUISpriteRendererProperties` / `NiagaraUIRibbonRendererProperties` | UI 专用渲染器属性 |
| `NiagaraUIRenderContext.cpp` / `NiagaraUISlateRenderContext` | UI 渲染上下文 |
| `SNiagaraUIWidget.cpp` / `NiagaraUIWidget.cpp` | Slate 控件与 UMG 控件 |
| `NiagaraDataInterfaceUIWidget.cpp` | UI 控件 Data Interface |

### 5.5 `NiagaraMRQ` — Movie Render Queue 集成

| 文件 | 作用 |
| --- | --- |
| `NiagaraDataInterfaceMRQ.h/.cpp` | 提供 MRQ 信息的 Data Interface |
| `NiagaraMRQModule.h/.cpp` | 模块入口 |

### 5.6 `NiagaraSimCaching` — SimCache Sequencer 集成

| 文件 | 作用 |
| --- | --- |
| `MovieSceneNiagaraCacheTrack.h/.cpp` / `MovieSceneNiagaraCacheSection.h/.cpp` / `MovieSceneNiagaraCacheTemplate.h/.cpp` | SimCache 的 Sequencer 轨道 / 段 / 求值模板 |

### 5.7 `NiagaraGaussianSplat` — 3D 高斯泼溅

| 文件 | 作用 |
| --- | --- |
| `NiagaraDataInterfaceGaussianSplat.h/.cpp` | 高斯泼溅数据接口 |
| `NiagaraGaussianSplatRenderer.h/.cpp` | 泼溅渲染器 |
| `NiagaraGaussianSplatActor.h/.cpp` / `NiagaraGaussianSplatSettings.h/.cpp` | Actor 与设置 |
| `NiagaraGaussianSplatEditor/`（`ActorFactory`、`ViewModels`、`Widgets`） | 编辑器集成 |

### 5.8 `NiagaraNanite` — Nanite 网格渲染

| 文件 | 作用 |
| --- | --- |
| `NiagaraNaniteRendererProperties.h/.cpp` / `NiagaraRendererNanite.h/.cpp` | Nanite 渲染器属性与实现 |
| `NiagaraParameterComponentBinding.h/.cpp` | 参数组件绑定 |
| `NiagaraNaniteShader/` | 相关 Shader 模块 |

### 5.9 `NiagaraLightweight` — 轻量无状态发射器

| 文件 | 作用 |
| --- | --- |
| `NiagaraStatelessEmitterTemplateAsset.h/.cpp` | 无状态发射器模板资产 |
| `Private/Modules/NiagaraStatelessModule_*.cpp` | 少量示例模块（KillBox、速度/力求解与碰撞） |
| `NiagaraLightweightEditor/` | 编辑器集成（工厂、细节面板） |

### 5.10 `NiagaraInsights` — Unreal Insights 分析

| 文件 | 作用 |
| --- | --- |
| `NiagaraProvider.h/.cpp` / `NiagaraAnalyzer.h/.cpp` / `NiagaraTraceModule.h/.cpp` | Trace 提供者 / 分析器 / Trace 模块 |
| `NiagaraInstanceLifecycleTrack` / `NiagaraPerformanceGraphTrack` / `NiagaraDataChannelTrack` | 实例生命周期 / 性能图 / 数据通道轨道 |
| `NiagaraTimingViewExtender.h/.cpp` / `NiagaraTimingViewSession.h/.cpp` | Timing 视图扩展 |
| `Widgets/SNiagaraRangeStatsView` | 范围统计视图控件 |

---

## 6. 模块依赖关系

```
NiagaraCore ──────────────── 无 Engine 依赖的基础类型
    ↑
    ├── NiagaraShader ────── GPU Shader 编译 + 计算辅助
    ├── NiagaraVertexFactories ── 顶点工厂（Sprite/Ribbon/Mesh + 间接绘制）
    └── Niagara ─────────── 主运行时（依赖 Shader / VertexFactories / Engine）
             ↑
             ├── NiagaraEditor ───────── 完整编辑器（依赖 Niagara）
             │         ↑
             │         └── NiagaraEditorWidgets ─ 编辑器 Slate 控件
             ├── NiagaraAnimNotifies ─── 动画通知
             └── NiagaraBlueprintNodes ── Data Channel 蓝图节点
```

各关联插件（`NiagaraFluids`、`NiagaraUIRenderer`、`NiagaraMRQ`、`NiagaraSimCaching`、`NiagaraGaussianSplat`、`NiagaraNanite`、`NiagaraLightweight`）大多依赖 `Niagara`（`NiagaraInsights` 额外依赖 Trace 框架），并在各自作用域内扩展 Niagara 的能力。

---

## 7. 小结

- **分层解耦**：`NiagaraCore`（纯类型）→ `NiagaraShader` / `NiagaraVertexFactories`（渲染底层）→ `Niagara`（主运行时）→ `NiagaraEditor` / `NiagaraEditorWidgets`（编辑器），依赖方向单向且清晰。
- **运行时以「编译数据 + 参数 + 数据集」为中心**：System/Emitter/Component 的职责边界由 `CompiledData`、`ParameterStore`、`DataSet` 串起。
- **Data Interface 是外部数据的统一抽象**：纹理、网格、碰撞、曲线、噪声、场景信息、Data Channel 全部收敛到同一接口体系。
- **渲染器采用「Properties + Renderer + SceneProxy + VertexFactory」三角结构**，把资产配置、运行时更新、线程提交、顶点生成四层拆开。
- **能力以插件形式外挂**：流体、UI 渲染、MRQ、SimCache 轨道、高斯泼溅、Nanite、Insights 都是可独立启停的插件，避免核心模块过度膨胀。
