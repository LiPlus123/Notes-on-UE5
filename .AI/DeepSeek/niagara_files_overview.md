# Niagara FX 系统代码文件一览

> 本文档基于 Unreal Engine 6 (UE6) 源码，列出 Niagara 粒子系统的核心文件夹、关键代码文件，介绍其作用与实现的算法。

---

## 一、整体架构概览

### 1.1 Niagara 在引擎中的位置

Niagara 作为 UE 的 FX 插件存在，**所有代码均位于 `Engine/Plugins/FX/` 下**，而非 `Engine/Source/Runtime/`。

```
Engine/Plugins/FX/
├── Niagara/                    ← 主插件（核心）
├── NiagaraFluids/              ← 流体模拟
├── NiagaraSimCaching/          ← 模拟缓存
├── NiagaraUIRenderer/          ← UI 渲染器
├── NiagaraNanite/              ← Nanite 集成
├── NiagaraGaussianSplat/       ← 高斯泼溅
├── NiagaraLightweight/         ← 轻量级发射器
├── NiagaraInsights/            ← 性能分析
├── NiagaraMRQ/                 ← Movie Render Queue 集成
├── NiagaraPreviewContent/      ← 预览内容
├── CascadeNiagaraConverterV2/  ← Cascade 转换器
├── CascadeToNiagaraConverter/  ← 旧版转换器
└── ExampleCustomDataInterface/ ← 自定义数据接口示例
```

### 1.2 核心模块一览

Niagara 主插件 `Source/` 下包含 **8 个 C++ 模块**：

| 模块 | 类型 | 作用 |
|------|------|------|
| **NiagaraCore** | Runtime | 基础类型：编译哈希、版本号、DataInterface 基类 |
| **Niagara** | Runtime | 主运行时：系统、发射器、渲染器、数据接口、GPU 调度 |
| **NiagaraShader** | Runtime | 着色器类型定义、编译管理、GPU 工具着色器 |
| **NiagaraVertexFactories** | Runtime | 顶点工厂（Sprite/Ribbon/Mesh）、GPU 排序、间接绘制 |
| **NiagaraEditor** | Editor | 图编辑器、HLSL 翻译器、编译管线、参数映射 |
| **NiagaraEditorWidgets** | Editor | Slate 控件、细节面板定制 |
| **NiagaraAnimNotifies** | Runtime | 动画通知触发 Niagara 效果 |
| **NiagaraBlueprintNodes** | Runtime | 蓝图 K2 节点（DataChannel 读写） |

### 1.3 架构分层

```
┌─────────────────────────────────────────────────────┐
│                    编辑器层                           │
│  NiagaraEditor  │  NiagaraEditorWidgets              │
│  (图编辑/编译/HLSL翻译)  (Slate控件/细节面板)          │
├─────────────────────────────────────────────────────┤
│                    运行时层                           │
│  NiagaraCore → Niagara → NiagaraShader               │
│  (基础类型)   (核心)    (着色器)                      │
│  NiagaraVertexFactories  │  NiagaraAnimNotifies       │
│  (顶点工厂/GPU排序)       │  (动画通知)                │
│  NiagaraBlueprintNodes                                   │
│  (蓝图节点)                                              │
├─────────────────────────────────────────────────────┤
│                  着色器层                             │
│  Shaders/Private/*.usf  │  Shaders/Private/*.ush     │
│  (计算/渲染入口)          │  (HLSL 模板/工具函数)       │
├─────────────────────────────────────────────────────┤
│                  VM 执行层                            │
│  Engine/Source/Runtime/VectorVM/                     │
│  Engine/Source/Developer/ShaderFormatVectorVM/       │
│  (CPU 字节码虚拟机)  (HLSL→VM 字节码编译器)           │
└─────────────────────────────────────────────────────┘
```

---

## 二、核心 Runtime 模块

### 2.1 NiagaraCore — 基础类型模块

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraCore/`

该模块提供 Niagara 系统最底层的基础设施，不依赖任何其他 Niagara 模块。

| 文件 | 作用 |
|------|------|
| `Public/NiagaraCompileHash.h` | 编译哈希类型，用于 DDC 缓存键 |
| `Public/NiagaraCustomVersion.h` | 自定义版本号，管理资产序列化兼容性 |
| `Public/NiagaraDataInterfaceBase.h` | 所有数据接口的抽象基类 |
| `Public/NiagaraMergeable.h` | 可合并对象基类，支持差异比较与合并 |
| `Public/NiagaraNotifyOnChanged.h` | 变更通知接口 |
| `Internal/NiagaraShaderFileHash.h` | 着色器源文件哈希，用于 DDC 失效判断 |

### 2.2 Niagara — 主运行时模块

**路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/`

#### 2.2.1 系统与发射器（核心生命周期）

| 文件 | 作用 | 关键算法 |
|------|------|----------|
| `Classes/NiagaraSystem.h` | 顶层系统资产，包含多个发射器句柄 | 系统生命周期管理 |
| `Classes/NiagaraEmitter.h` | 发射器资产，定义粒子行为 | 发射器状态机 |
| `Public/NiagaraComponent.h` | 场景组件，挂载并运行 Niagara 系统 | Actor-Component 模式 |
| `Public/NiagaraSystemInstance.h` | 系统运行时实例，每帧 Tick | 系统实例循环 |
| `Classes/NiagaraEmitterInstance.h` | 发射器运行时实例 | 发射器实例循环 |
| `Public/NiagaraWorldManager.h` | 世界级 Niagara 管理器，全局 Tick 调度 | 批处理、可扩展性管理 |
| `Public/NiagaraSystemSimulation.h` | 系统模拟调度器 | 并发模拟任务调度 |
| `Public/NiagaraScalabilityManager.h` | 可扩展性管理器 | 基于平台/距离的 LOD、Cull |

**核心执行流程**:
```
WorldManager::Tick()
  └→ SystemSimulation::Tick()
       └→ EmitterInstance::Tick()
            ├─ CPU路径: ScriptExecutionContext::Execute() → VectorVM
            └─ GPU路径: GpuComputeDispatch → 提交计算着色器
```

#### 2.2.2 脚本执行（VM 运行）

| 文件 | 作用 |
|------|------|
| `Classes/NiagaraScript.h` | 脚本资产，持有编译后的字节码或着色器映射 |
| `Classes/NiagaraScriptExecutionContext.h` | CPU 端脚本执行上下文，封装 VectorVM 调用 |
| `Public/NiagaraScriptExecutionParameterStore.h` | 脚本参数存储，绑定到 VM |
| `Public/NiagaraScriptRuntimeCompiledData.h` | 编译后的运行时数据 |
| `Public/NiagaraScriptRuntimeCookedData.h` | Cook 后的运行时数据 |
| `Public/NiagaraParameterStore.h` | 通用参数存储（键值对） |
| `Public/NiagaraParameterBinding.h` | 参数绑定 |
| `Classes/NiagaraDataSet.h` | 粒子数据集，管理粒子属性的缓冲区 |

#### 2.2.3 渲染器（7 种）

每种渲染器由 `Properties`（配置）和 `Renderer`（执行）两个类组成：

| 渲染器 | Properties 文件 | Renderer 文件 | 渲染内容 |
|--------|----------------|---------------|----------|
| **Sprite** | `Public/NiagaraSpriteRendererProperties.h` | `Public/NiagaraRendererSprites.h` | 面对相机的精灵（Billboard） |
| **Ribbon** | `Public/NiagaraRibbonRendererProperties.h` | `Public/NiagaraRendererRibbons.h` | 粒子拖尾丝带 |
| **Mesh** | `Public/NiagaraMeshRendererProperties.h` | `Public/NiagaraRendererMeshes.h` | 3D 静态/骨骼网格体 |
| **Light** | `Public/NiagaraLightRendererProperties.h` | `Public/NiagaraRendererLights.h` | 动态光源 |
| **Decal** | `Public/NiagaraDecalRendererProperties.h` | `Public/NiagaraRendererDecals.h` | 贴花 |
| **Volume** | `Public/NiagaraVolumeRendererProperties.h` | `Public/NiagaraRendererVolumes.h` | 体积渲染 |
| **Component** | `Public/NiagaraComponentRendererProperties.h` | `Public/NiagaraRendererComponents.h` | 子 Actor 组件 |

**基类**: `Public/NiagaraRenderer.h` + `Public/NiagaraRendererProperties.h`

#### 2.2.4 数据接口（Data Interface）

数据接口是 Niagara 与外部数据交互的桥梁，分为以下几类：

**基础/读写**:
- `Classes/NiagaraDataInterface.h` — 所有 DI 的基类
- `Classes/NiagaraDataInterfaceRW.h` — GPU 读写 DI 基类（Grid、RT 等）

**纹理/渲染目标**:
- `NiagaraDataInterfaceTexture.h` — 2D 纹理采样
- `NiagaraDataInterface2DArrayTexture.h` — 2D 纹理数组
- `NiagaraDataInterfaceCubeTexture.h` — 立方体贴图
- `NiagaraDataInterfaceVolumeTexture.h` — 3D 体积纹理
- `NiagaraDataInterfaceSparseVolumeTexture.h` — 稀疏体积纹理
- `NiagaraDataInterfaceVirtualTextureSample.h` — 虚拟纹理采样
- `NiagaraDataInterfaceRenderTarget2D.h` / `2DArray.h` / `Cube.h` / `Volume.h` — RT 读写
- `NiagaraDataInterfaceIntRenderTarget2D.h` — 整数 RT
- `NiagaraDataInterfaceVolumeCache.h` — 体积缓存

**网格/集合（Grid）**:
- `NiagaraDataInterfaceGrid2DCollection.h` — 2D 网格集合（可读写）
- `NiagaraDataInterfaceGrid2DCollectionReader.h` — 2D 网格只读
- `NiagaraDataInterfaceGrid3DCollection.h` — 3D 网格集合（流体模拟核心）
- `NiagaraDataInterfaceGrid3DCollectionReader.h` — 3D 网格只读
- `NiagaraDataInterfaceNeighborGrid3D.h` — 3D 邻居网格（空间哈希）
- `NiagaraDataInterfaceRasterizationGrid3D.h` — 光栅化 3D 网格

**曲线**:
- `NiagaraDataInterfaceCurveBase.h` — 曲线基类
- `NiagaraDataInterfaceCurve.h` / `ColorCurve.h` / `VectorCurve.h` / `Vector2DCurve.h` / `Vector4Curve.h`

**音频**:
- `NiagaraDataInterfaceAudioOscilloscope.h` — 波形可视化
- `NiagaraDataInterfaceAudioPlayer.h` — 音频播放同步
- `NiagaraDataInterfaceAudioSpectrum.h` — 频谱分析

**网格体/物理/世界**:
- `NiagaraDataInterfaceSkeletalMesh.h` — 骨骼网格体采样（骨骼/三角面/顶点）
- `NiagaraDataInterfaceStaticMesh.h` — 静态网格体表面采样
- `NiagaraDataInterfaceSpline.h` — 样条线采样
- `NiagaraDataInterfaceLandscape.h` — 地形高度/法线查询
- `NiagaraDataInterfaceCamera.h` — 相机信息
- `NiagaraDataInterfaceCollisionQuery.h` — 碰撞查询（CPU）
- `NiagaraDataInterfaceAsyncGpuTrace.h` — 异步 GPU 射线追踪
- `NiagaraDataInterfaceOcclusion.h` — 遮挡查询
- `NiagaraDataInterfaceCurlNoise.h` — Curl Noise 力场
- `NiagaraDataInterfaceVectorField.h` — 向量场（ISPC 加速）

**数据通道**:
- `Public/NiagaraDataChannel.h` — 数据通道核心（跨发射器通信）
- `Public/NiagaraDataChannelAsset.h` — 数据通道资产
- `Public/NiagaraDataChannelHandler.h` — 数据通道处理器
- `Public/NiagaraDataChannelAccessor.h` — 数据通道访问器
- `Public/NiagaraDataChannelPublic.h` — 公共数据通道接口
- `Public/NiagaraDataChannel_Global.h` — 全局数据通道
- `Public/NiagaraDataChannel_Islands.h` — 岛屿数据通道
- `Public/NiagaraDataChannel_Map.h` — 映射数据通道
- `Public/NiagaraDataChannel_GameplayBurst.h` — 游戏爆发数据通道

**其他**:
- `NiagaraDataInterfaceArray.h` — 数组 DI（Float/Int/NiagaraID）
- `NiagaraDataInterfaceExport.h` — 粒子数据导出
- `NiagaraDataInterfaceNeighborQuery.h` — 邻居查询
- `NiagaraDataInterfaceParticleRead.h` — 粒子数据读取
- `NiagaraDataInterfaceSimCacheReader.h` — SimCache 读取
- `NiagaraDataInterfaceGBuffer.h` — GBuffer 数据访问
- `NiagaraDataInterfaceDebugDraw.h` — 调试绘制
- `NiagaraDataInterfaceEmitterProperties.h` — 发射器属性
- `NiagaraDataInterfaceSystemProperties.h` — 系统属性
- `NiagaraDataInterfaceUObjectPropertyReader.h` — UObject 属性读取
- `NiagaraDataInterfaceSocketReader.h` — 骨骼 Socket 读取
- `NiagaraDataInterfaceSimpleCounter.h` — 简单计数器
- `NiagaraDataInterfaceMemoryBuffer.h` — 内存缓冲区
- `NiagaraDataInterfaceDataTable.h` — DataTable 查询
- `NiagaraDataInterfaceConsoleVariable.h` — 控制台变量读取
- `NiagaraDataInterfaceDynamicMesh.h` — 动态网格体
- `NiagaraDataInterfaceRibbon.h` — 丝带数据
- `NiagaraDataInterfaceSceneCapture2D.h` — 场景捕获 2D
- `NiagaraDataInterfaceActorComponent.h` — Actor 组件
- `NiagaraDataInterfacePhysicsAsset.h` — 物理资产
- `NiagaraDataInterfacePropertyInterface.h` — 属性接口
- `NiagaraDataInterfaceStaticMeshIndirect.h` — 间接静态网格体

#### 2.2.5 GPU 计算调度

| 文件 | 作用 | 算法 |
|------|------|------|
| `Private/NiagaraGpuComputeDispatch.cpp` | **GPU 调度器核心**：每帧编排发射器的计算 Pass | 多阶段调度、间接 Dispatch、Stage 转换 |
| `Public/NiagaraGpuComputeDispatchInterface.h` | 调度器公开接口 | — |
| `Classes/NiagaraComputeExecutionContext.h` | 每个发射器的 GPU 执行状态 | 缓冲区绑定、参数映射 |
| `Classes/NiagaraGPUInstanceCountManager.h` | GPU 实例计数器管理 | 持久化 32-bit 原子计数器 |
| `Private/NiagaraGPUProfiler.cpp` | GPU 性能分析 | GPU 时间戳查询 |
| `Classes/NiagaraGPUSystemTick.h` | 每帧 GPU 工作容器 | 参数 Blob、DI 代理、调度任务 |
| `Private/NiagaraGpuReadbackManager.cpp` | 异步 GPU→CPU 回读 | 双缓冲、Fence 同步 |
| `Public/NiagaraSystemGpuComputeProxy.h` | 渲染线程代理 | 多发射器批处理 |
| `Private/NiagaraGpuComputeDebug.cpp` | GPU 调试工具 | 调试数据捕获 |

#### 2.2.6 模拟阶段（Simulation Stage）

| 文件 | 作用 |
|------|------|
| `Public/NiagaraSimulationStageBase.h` | 模拟阶段基类，用户可在 GPU 模拟中插入自定义 Stage |
| `Public/NiagaraSimStageData.h` | 每个 Stage 的运行时数据 |
| `Public/NiagaraSimStageExecutionData.h` | 每个 Stage 的编译后执行数据 |
| `Public/NiagaraSimulationStageCompileData.h` | Stage 编译描述符 |

#### 2.2.7 Stateless 发射器（轻量级、无 VM）

Stateless 发射器跳过 VM 直接运行 GPU 计算着色器，适用于简单粒子效果。

**核心类** (`Source/Niagara/Internal/Stateless/`):

| 文件 | 作用 |
|------|------|
| `NiagaraStatelessEmitter.h` | Stateless 发射器定义 |
| `NiagaraStatelessEmitterData.h` | 发射器数据 |
| `NiagaraStatelessEmitterInstance.h` | 运行时实例 |
| `NiagaraStatelessEmitterTemplate.h` | 发射器模板 |
| `NiagaraStatelessComputeManager.h` | 计算管理器 |
| `NiagaraStatelessModule.h` | 模块基类 |
| `NiagaraStatelessExpression.h` | 表达式系统 |

**模块** (`Source/Niagara/Internal/Stateless/Modules/`):
初始化、力学、形状定位、缩放、旋转、朝向、UV 动画、材质参数、渲染属性等 28 个模块。

#### 2.2.8 SimCache（模拟缓存）

| 文件 | 作用 |
|------|------|
| `Classes/NiagaraSimCache.h` | SimCache 核心：录制/回放粒子模拟 |
| `Classes/NiagaraSimCacheCapture.h` | 缓存捕获器 |
| `Classes/NiagaraSimCacheCompare.h` | 缓存比较（回归测试） |
| `Classes/NiagaraSimCacheFunctionLibrary.h` | 蓝图函数库 |
| `Classes/NiagaraSimCacheJson.h` | JSON 导入导出 |

### 2.3 NiagaraShader — 着色器模块

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/`

| 文件 | 作用 |
|------|------|
| `Public/NiagaraShader.h` | Niagara 计算着色器类型（FGlobalShader 子类） |
| `Public/NiagaraShaderCompilationManager.h` | 异步着色器编译管理器 |
| `Public/NiagaraScriptBase.h` | 脚本基类（着色器侧） |
| `Public/NiagaraShaderParametersBuilder.h` | 着色器参数元数据构建器 |
| `Public/NiagaraShaderParticleID.h` | 持久化粒子 ID 计算着色器驱动 |
| `Public/NiagaraRibbonCompute.h` | Ribbon 归约/排序/索引计算着色器驱动 |
| `Public/NiagaraAsyncGpuTraceProvider.h` | 异步 GPU 追踪提供者 |
| `Public/NiagaraSortingGPU.h` | GPU 排序（Bitonic Sort） |
| `Public/NiagaraClearCounts.h` | 计数清除着色器 |
| `Public/NiagaraGenerateMips.h` | Mip 生成着色器 |
| `Public/NiagaraGPUSceneUtils.h` | GPU Scene 写入工具 |
| `Public/NiagaraDistanceFieldHelper.h` | 全局距离场查询 |
| `Public/NiagaraBatchedElements.h` | 批量元素绘制 |
| `Public/NiagaraDebugShaders.h` | 调试着色器 |

### 2.4 NiagaraVertexFactories — 顶点工厂模块

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/`

| 文件 | 作用 |
|------|------|
| `Public/NiagaraVertexFactory.h` | 顶点工厂基类 |
| `Public/NiagaraSpriteVertexFactory.h` | Sprite 顶点工厂 |
| `Public/NiagaraMeshVertexFactory.h` | Mesh 顶点工厂 |
| `Public/NiagaraRibbonVertexFactory.h` | Ribbon 顶点工厂 |
| `Public/NiagaraSortingGPU.h` | GPU 粒子排序（Bitonic Sort） |
| `Public/NiagaraCutoutVertexBuffer.h` | 裁剪顶点缓冲区 |
| `Public/NiagaraDrawIndirect.h` | 间接绘制参数生成 |
| `Public/NiagaraDispatchIndirect.h` | 间接调度参数生成 |
| `Public/NiagaraGPURayTracingTransformsShader.h` | 光追变换计算着色器 |

### 2.5 NiagaraAnimNotifies — 动画通知模块

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraAnimNotifies/`

| 文件 | 作用 |
|------|------|
| `Public/AnimNotify_PlayNiagaraEffect.h` | 动画通知：播放一次性 Niagara 效果 |
| `Public/AnimNotifyState_TimedNiagaraEffect.h` | 动画通知状态：持续播放 Niagara 效果 |

### 2.6 NiagaraBlueprintNodes — 蓝图节点模块

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraBlueprintNodes/`

| 文件 | 作用 |
|------|------|
| `Internal/K2Node_ReadDataChannel.h` | 蓝图节点：读取 DataChannel |
| `Internal/K2Node_WriteDataChannel.h` | 蓝图节点：写入 DataChannel |
| `Internal/K2Node_DataChannelBase.h` | DataChannel 节点基类 |

---

## 三、编辑器模块

### 3.1 NiagaraEditor — 编辑器核心

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraEditor/`

#### 3.1.1 编译管线

| 文件 | 作用 | 算法 |
|------|------|------|
| `Private/NiagaraHlslTranslator.cpp` | **HLSL 翻译器**：遍历 Niagara 图，生成 HLSL 源码 | 图遍历、命名空间解析、参数映射 |
| `Private/NiagaraCompiler.cpp` | 编译器编排：协调翻译器、交叉编译器、VectorVM 后端、DDC | 多阶段编译流水线 |
| `Private/NiagaraParameterMapHistory.cpp` | 参数映射历史：追踪 Set/Get 链，确定读写与命名空间 | 静态分析、数据流追踪 |
| `Private/NiagaraGraph.cpp` | Niagara 图定义 | 有向图节点管理 |
| `Private/NiagaraNode.cpp` | 图节点基类 | — |
| `Private/NiagaraNodeCustomHlsl.cpp` | 自定义 HLSL 节点 | 内联 HLSL 注入 |
| `Private/NiagaraNodeFunctionCall.cpp` | 函数调用节点 | 模块/函数内联 |
| `Private/NiagaraNodeParameterMapGet.cpp` | 参数获取节点 | 命名空间读取 |
| `Private/NiagaraNodeParameterMapSet.cpp` | 参数设置节点 | 命名空间写入 |
| `Private/NiagaraNodeParameterMapFor.cpp` | 参数映射 For 循环 | 迭代展开 |
| `Private/NiagaraCompilationTasks.cpp` | 异步编译任务 | 任务图调度 |
| `Private/NiagaraSystemCompilingManager.cpp` | 系统编译队列 | 队列管理 |
| `Private/NiagaraCompilationGraphBridgeImpl.cpp` | 编译图桥接 | 图数据抽象 |
| `Private/NiagaraCompilationDigestBridgeImpl.cpp` | 编译摘要桥接 | 摘要数据抽象 |

#### 3.1.2 编辑器工具包

| 文件夹 | 作用 |
|--------|------|
| `Private/Toolkits/` | 系统/发射器/脚本编辑器工具包 |
| `Private/ViewModels/` | MVVM 视图模型（堆栈、层级编辑器） |
| `Private/ViewModels/Stack/` | Niagara 堆栈系统（模块编辑界面） |
| `Private/Widgets/` | Slate 控件（图节点、资产浏览器、向导） |
| `Private/Customizations/` | 细节面板定制（各种 DI、渲染器属性） |
| `Private/Utilities/` | 编辑器工具函数 |
| `Private/Sequencer/` | Sequencer 集成（关卡序列中的 Niagara 轨道） |
| `Private/Commandlets/` | 命令行工具（字节码导出等） |
| `Private/AssetDefinitions/` | 资产定义（System、Emitter、Script 等） |
| `Private/TypeEditorUtilities/` | 类型编辑器工具 |

#### 3.1.3 资产定义

| 文件 | 作用 |
|------|------|
| `AssetDefinition_NiagaraSystem.h` | 系统资产的操作定义 |
| `AssetDefinition_NiagaraEmitter.h` | 发射器资产的操作定义 |
| `AssetDefinition_NiagaraScript.h` | 脚本资产的操作定义 |
| `AssetDefinition_NiagaraSimCache.h` | SimCache 资产的操作定义 |
| `AssetDefinition_NiagaraDataChannel.h` | DataChannel 资产的操作定义 |
| `AssetDefinition_NiagaraParameterCollection.h` | 参数集合资产的操作定义 |
| `AssetDefinition_NiagaraValidationRuleSet.h` | 验证规则集资产的操作定义 |

### 3.2 NiagaraEditorWidgets — 编辑器控件

**路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraEditorWidgets/`

Slate 控件与细节面板定制，包括：
- `SNiagara*` 系列 Slate 控件（参数选择器、概览图、时间轴等）
- `DetailCustomizations/` — 细节面板定制（Grid2D/3D、DataChannel 等）

---

## 四、着色器与 GPU 计算

### 4.1 着色器文件清单

**路径**: `Engine/Plugins/FX/Niagara/Shaders/Private/`

#### 4.1.1 核心计算着色器 (.usf)

| 文件 | 作用 | 算法 |
|------|------|------|
| `NiagaraEmitterInstanceShader.usf` | **主发射器模拟着色器**：Spawn/Update 逻辑的容器 | 注入生成的 HLSL 代码 |
| `NiagaraComputeFreeIDs.usf` | 生成空闲粒子 ID | 原子操作、空闲列表 |
| `NiagaraInitFreeIDBuffer.usf` | 初始化空闲 ID 缓冲区 | 顺序填充 |
| `NiagaraClearCounts.usf` | 清除 GPU 实例计数缓冲区 | 清零 |
| `NiagaraDispatchIndirectArgsGen.usf` | 生成间接调度参数 | 原子计数 → DispatchIndirect 参数 |
| `NiagaraDrawIndirectArgsGen.usf` | 生成间接绘制参数 | 原子计数 → DrawIndirect 参数 |
| `NiagaraSortKeyGen.usf` | 生成排序键 | 深度/距离排序键 |

#### 4.1.2 Ribbon 计算着色器

| 文件 | 作用 | 算法 |
|------|------|------|
| `Ribbons/NiagaraRibbonSortParticles.usf` | 粒子排序为丝带顺序 | 排序 |
| `Ribbons/NiagaraRibbonAggregationStep.usf` | 并行聚合步骤 | 并行前缀和/归约 |
| `Ribbons/NiagaraRibbonAggregationApply.usf` | 聚合应用 | 散射写入 |
| `Ribbons/NiagaraRibbonGenerateIndices.usf` | 生成丝带索引缓冲区 | 索引构建 |
| `Ribbons/NiagaraRibbonVertexReductionPropagation.usf` | 顶点归约传播 | 并行归约 |
| `Ribbons/NiagaraRibbonUVParamCalculation.usf` | UV 参数计算 | 弧长参数化 |

#### 4.1.3 Stateless 发射器着色器

| 文件 | 作用 |
|------|------|
| `Stateless/NiagaraStatelessSimulationDefault.usf` | 默认 Stateless 模拟入口 |
| `Stateless/Template/GenerateCS_PreModules.ush` | 模块前模板代码 |
| `Stateless/Template/GenerateCS_PostModules.ush` | 模块后模板代码 |
| `Stateless/Modules/NiagaraStatelessModule_*.ush` | 19 个 Stateless 模块 HLSL 实现 |

#### 4.1.4 公共包含文件 (.ush)

| 文件 | 作用 |
|------|------|
| `NiagaraCommon.ush` | 顶层定义：LWC 辅助、粒子结构布局、宏 |
| `NiagaraShaderVersion.ush` | 着色器版本号（修改触发全量重编译） |
| `NiagaraParticleAccess.ush` | 粒子属性 GPU 缓冲区读取 |
| `NiagaraVFParticleAccess.usf` | 顶点工厂粒子属性访问 |
| `NiagaraMeshParticleUtils.ush` | 网格体粒子工具 |
| `NiagaraTransformUtils.ush` | 变换工具（四元数、矩阵） |
| `NiagaraQuaternionUtils.ush` | 四元数运算 |
| `NiagaraBaryCentricUtils.ush` | 重心坐标插值 |
| `NiagaraPhysicsCommon.ush` | 物理公共函数 |
| `NiagaraDebugDraw.ush` | 调试绘制（点、线、盒） |
| `NiagaraAsyncGpuTraceCommon.ush` | 异步 GPU 追踪公共类型 |

#### 4.1.5 数据接口 HLSL 模板 (.ush)

每个数据接口都有一个对应的 `.ush` 模板文件，在 HLSL 翻译阶段注入：

**网格/RW**:
- `NiagaraDataInterfaceGrid2DCollection.ush` — 2D 网格读取/写入
- `NiagaraDataInterfaceGrid3DCollection.ush` — 3D 网格读取/写入（流体模拟核心）
- `NiagaraDataInterfaceRenderTarget2DTemplate.ush` — 2D RT 模板
- `NiagaraDataInterfaceRenderTargetVolumeTemplate.ush` — 3D RT 模板

**纹理采样**:
- `NiagaraDataInterfaceTextureTemplate.ush` — 2D 纹理
- `NiagaraDataInterfaceVolumeTextureTemplate.ush` — 体积纹理
- `NiagaraDataInterfaceSparseVolumeTextureTemplate.ush` — 稀疏体积纹理
- `NiagaraDataInterfaceVirtualTextureSampleTemplate.ush` — 虚拟纹理

**几何查询**:
- `NiagaraDataInterfaceSkeletalMesh.ush` — 骨骼网格体采样
- `NiagaraDataInterfaceStaticMeshTemplate.ush` — 静态网格体采样
- `NiagaraDataInterfaceSplineTemplate.ush` — 样条线采样
- `NiagaraDataInterfaceLandscape.ush` — 地形查询

**物理/碰撞**:
- `NiagaraDataInterfaceCollisionQuery.ush` — 碰撞查询
- `NiagaraDataInterfaceAsyncGpuTrace.ush` — 异步 GPU 追踪
- `NiagaraDataInterfaceNeighborQuery.ush` — 邻居查询

**工具**:
- `NiagaraDataInterfaceArrayTemplate.ush` — 数组访问
- `NiagaraDataInterfaceCurveTemplate.ush` — 曲线采样
- `NiagaraDataInterfaceDataTableTemplate.ush` — 数据表查询
- `NiagaraDataInterfaceCamera.ush` — 相机数据

---

## 五、VectorVM 与字节码编译器

### 5.1 VectorVM 运行时

**路径**: `Engine/Source/Runtime/VectorVM/`

VectorVM 是 Niagara CPU 模拟的核心——一个高效的 SIMD 字节码虚拟机。

| 文件 | 作用 | 算法 |
|------|------|------|
| `Private/VectorVM.cpp` | VM 入口 | 虚拟机初始化 |
| `Private/VectorVMRuntime.cpp` | **主执行循环**：操作码分发 | 字节码解释、SIMD 批量处理 |
| `Private/VectorVMOptimizer.cpp` | 字节码优化器 | 死代码消除、常量折叠、操作合并 |
| `Private/VectorVMBridge.cpp` | VM 与 Niagara 的桥接 | 外部数据绑定 |
| `Private/Platforms/VectorVMPlatformGeneric.h` | 通用平台实现 | 标量/SIMD 抽象 |
| `Public/VectorVM.h` | 公开 API | — |

**VM 核心设计**:
- 每条指令操作的是**整个粒子批次**（而非单个粒子）
- 使用 SIMD 寄存器（4-wide）并行处理 4 个粒子
- 通过操作码表分发：Load/Store/Add/Mul/Sin/Cos/Noise 等

### 5.2 ShaderFormatVectorVM — HLSL→VM 编译器

**路径**: `Engine/Source/Developer/ShaderFormatVectorVM/`

| 文件 | 作用 | 算法 |
|------|------|------|
| `Private/VectorVMBackend.cpp` | **代码生成后端**：将 HLSL IR 转为 VM 字节码 | IR 遍历、指令选择、寄存器分配 |
| `Private/VectorVMShaderCompiler.cpp` | 着色器编译器插件入口 | 编译管线集成 |

### 5.3 编译管线（端到端）

```
Niagara Graph (编辑器中)
    │
    ▼
NiagaraHlslTranslator.cpp     ← 图遍历 → 生成 HLSL 源码
    │
    ├─── CPU 路径 ──────────────────────────
    │       │
    │       ▼
    │   ShaderFormatVectorVM (VectorVMBackend)
    │       │  HLSL IR → VM 字节码
    │       ▼
    │   VectorVM (VectorVMRuntime)
    │       │  SIMD 字节码解释执行
    │
    └─── GPU 路径 ──────────────────────────
            │
            ▼
        HLSL 交叉编译器 (DXC/FShaderCompiler)
            │  HLSL → DXIL/SPIR-V
            ▼
        NiagaraShaderCompilationManager
            │  异步编译 + DDC 缓存
            ▼
        NiagaraEmitterInstanceShader.usf
            │  注入生成的 HLSL 代码
            ▼
        NiagaraGpuComputeDispatch
            │  提交计算着色器
            ▼
        GPU 执行
```

---

## 六、核心算法说明

### 6.1 GPU 粒子排序（Bitonic Sort）

**文件**: `Source/NiagaraVertexFactories/Private/NiagaraSortingGPU.cpp`
**着色器**: `Shaders/Private/NiagaraSortKeyGen.usf`

- 使用 **Bitonic Sort**（双调排序）在 GPU 上对粒子排序
- 适合 GPU 并行：O(n log²n) 比较次数，但每次比较可并行
- 排序键：通常为深度值（用于半透明排序）或自定义排序键
- 输出：排序后的索引缓冲区，供渲染器使用

### 6.2 间接调度（Indirect Dispatch）

**文件**: `Source/Niagara/Private/NiagaraGpuComputeDispatch.cpp`
**着色器**: `Shaders/Private/NiagaraDispatchIndirectArgsGen.usf`

- 粒子数量在 GPU 上动态变化（Spawn/Kill）
- 使用 **Indirect Dispatch**：GPU 先计算实际粒子数，写入 Indirect Args Buffer
- 后续计算着色器读取 Indirect Args Buffer 确定 Dispatch 线程组数
- 避免 CPU→GPU 回读，消除同步开销

### 6.3 空闲 ID 分配

**着色器**: `Shaders/Private/NiagaraComputeFreeIDs.usf`、`NiagaraInitFreeIDBuffer.usf`

- 持久化粒子 ID 系统：维护一个空闲 ID 列表
- Spawn 时从空闲列表分配 ID（原子操作 Pop）
- Kill 时将 ID 归还空闲列表（原子操作 Push）
- 使用 GPU 原子计数器管理空闲列表的读写指针

### 6.4 Ribbon 并行归约

**文件**: `Source/NiagaraShader/Private/NiagaraRibbonCompute.cpp`
**着色器**: `Shaders/Private/Ribbons/NiagaraRibbonVertexReduction*.usf`

- 多阶段并行归约流水线：
  1. **初始化**: 每个粒子段计算局部数据
  2. **传播**: 层叠式并行归约，log(n) 步
  3. **最终化**: 产生每顶点数据
- 并行计算丝带弧长、UV 坐标、宽度

### 6.5 网格 3D 集合（Grid/Neighbor Grid）

**文件**: `Source/Niagara/Private/NiagaraDataInterfaceGrid3DCollection.cpp`
**着色器**: `Shaders/Private/NiagaraDataInterfaceGrid3DCollection.ush`

- **Grid3DCollection**: 3D 纹理/缓冲区，支持原子读写
  - 用于流体模拟（速度场、密度场）
  - 支持多个 Grid 之间的 Swap（Ping-Pong 缓冲）
- **NeighborGrid3D**: 空间哈希网格
  - 将粒子映射到 3D 网格单元
  - 快速邻居查询（O(1) 查找附近粒子）
  - 底层使用排序 + 扫描（Counting Sort 变体）

### 6.6 异步 GPU 追踪

**文件**: `Source/Niagara/Private/NiagaraAsyncGpuTraceHelper.cpp`
**着色器**: `Shaders/Private/NiagaraAsyncGpuTrace.ush`

- 两种实现：
  - **GSDF**（Global Signed Distance Field）：使用全局距离场进行追踪
  - **HWRT**（Hardware Ray Tracing）：使用 DXR 硬件光追
- 异步执行：追踪请求在粒子模拟中发出，结果在下一帧可用
- 支持碰撞、遮挡、投影等多种查询

### 6.7 Stateless 发射器

**文件**: `Source/Niagara/Private/Stateless/NiagaraStatelessEmitter.cpp`

- 轻量级发射器，**跳过 VM 层**直接运行 GPU 计算着色器
- 模块以 `.ush` 模板形式存在，编译时内联
- 适用于简单粒子效果（高性能、低开销）
- 模块包括：力学（重力、阻力）、形状定位、缩放、旋转、朝向、UV 动画等

### 6.8 Curl Noise

**文件**: `Source/Niagara/Private/NiagaraDataInterfaceCurlNoise.cpp`

- 生成**无散度噪声场**（∇·F = 0），用于流体风格运动
- 基于 **Perlin/Simplex Noise** 的旋度计算
- 粒子在 Curl Noise 场中运动自然呈现涡旋效果

---

## 七、其他 Niagara 相关插件

### 7.1 NiagaraFluids — 流体模拟

**路径**: `Engine/Plugins/FX/NiagaraFluids/`

基于 Grid3DCollection 的流体模拟模板，提供：
- 速度场平流（Advection）
- 压力求解（Pressure Solve）
- 涡度约束（Vorticity Confinement）
- 密度/温度场模拟

**着色器**: `Shaders/NiagaraFluids.ush`、`Shaders/NiagaraFFT.ush`

### 7.2 NiagaraSimCaching — 模拟缓存

**路径**: `Engine/Plugins/FX/NiagaraSimCaching/`

- 录制 Niagara 模拟到缓存文件
- 支持 GPU 资源缓存
- 回放时无需重新模拟
- 编辑器集成（Sequencer 时间轴）

### 7.3 NiagaraUIRenderer — UI 渲染

**路径**: `Engine/Plugins/FX/NiagaraUIRenderer/`

- 在 UMG 控件中渲染 Niagara 粒子
- 支持 UI 材质
- 独立的渲染管线

### 7.4 NiagaraNanite — Nanite 集成

**路径**: `Engine/Plugins/FX/NiagaraNanite/`

- 将 Niagara 粒子作为 Nanite 几何体渲染
- 适用于大规模粒子（数百万级）

### 7.5 NiagaraGaussianSplat — 高斯泼溅

**路径**: `Engine/Plugins/FX/NiagaraGaussianSplat/`

- 3D 高斯泼溅渲染
- 基于 Niagara 粒子系统实现

### 7.6 NiagaraLightweight — 轻量级发射器

**路径**: `Engine/Plugins/FX/NiagaraLightweight/`

- 极简发射器实现
- 适用于移动端或低性能平台

### 7.7 NiagaraInsights — 性能分析

**路径**: `Engine/Plugins/FX/NiagaraInsights/`

- Unreal Insights 集成
- Niagara 专用性能追踪

### 7.8 NiagaraMRQ — Movie Render Queue

**路径**: `Engine/Plugins/FX/NiagaraMRQ/`

- Movie Render Queue 集成
- 高质量离线渲染 Niagara 效果

---

## 八、附录：关键路径速查

### 8.1 目录结构速查

```
Engine/Plugins/FX/Niagara/
├── Shaders/Private/           ← 所有 Niagara 着色器 (.usf/.ush)
├── Source/
│   ├── NiagaraCore/           ← 基础类型 (6 个核心文件)
│   ├── Niagara/               ← 主运行时 (200+ 文件)
│   │   ├── Classes/           ←   UObject 头文件
│   │   ├── Public/            ←   公开 API 头文件
│   │   │   ├── Stateless/     ←     Stateless 发射器
│   │   │   ├── MovieScene/    ←     Sequencer 集成
│   │   │   └── DataInterface/ ←     数据接口
│   │   ├── Internal/Stateless/←   Stateless 内部头文件
│   │   └── Private/           ←   实现文件
│   │       ├── DataInterface/ ←     数据接口实现
│   │       ├── MovieScene/    ←     Sequencer 实现
│   │       └── Stateless/     ←     Stateless 实现
│   ├── NiagaraShader/         ← 着色器类型与编译
│   ├── NiagaraVertexFactories/← 顶点工厂与 GPU 排序
│   ├── NiagaraEditor/         ← 编辑器核心
│   ├── NiagaraEditorWidgets/  ← 编辑器控件
│   ├── NiagaraAnimNotifies/   ← 动画通知
│   └── NiagaraBlueprintNodes/ ← 蓝图节点
└── Content/                   ← 资产内容

Engine/Source/Runtime/VectorVM/         ← CPU 字节码 VM
Engine/Source/Developer/ShaderFormatVectorVM/ ← HLSL→VM 编译器
```

### 8.2 端到端追踪路径

| 过程 | 入口文件 | 核心文件 |
|------|----------|----------|
| 编译 (CPU) | `NiagaraHlslTranslator.cpp` | `VectorVMBackend.cpp` → `VectorVMRuntime.cpp` |
| 编译 (GPU) | `NiagaraHlslTranslator.cpp` | `NiagaraShaderCompilationManager.cpp` → `NiagaraEmitterInstanceShader.usf` |
| CPU 执行 | `NiagaraSystemSimulation.cpp` | `NiagaraScriptExecutionContext.cpp` → `VectorVMRuntime.cpp` |
| GPU 执行 | `NiagaraGpuComputeDispatch.cpp` | `NiagaraEmitterInstanceShader.usf` → Indirect Dispatch |
| 渲染 | `NiagaraRenderer*.cpp` | `Niagara*VertexFactory.cpp` → `NiagaraSortingGPU.cpp` |

---

> 本文档基于 UE6 源码 (`Engine/Plugins/FX/Niagara/`) 整理，文件路径为相对于引擎根目录的相对路径。所有文件均可在 `d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\` 下找到。