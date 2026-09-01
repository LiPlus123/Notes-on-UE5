# 物理引擎文件一览

> 本文档基于 UE 5.5+ / UE 6 源码，梳理 Unreal Engine 中 Chaos 物理引擎的完整模块架构。每个模块列出其核心职责、关键子目录与实现的核心算法。

---

目录总览

| 目录 | 模块 | 层级 | 核心职责 |
|---|---|---|---|
| `Engine/Source/Runtime/PhysicsCore/` | PhysicsCore | 引擎层 | 物理接口抽象、场景查询、物理材质基类 |
| `Engine/Source/Runtime/Engine/Public/Physics/` | Engine | 引擎层 | 物理场景、网络同步、即时物理、Chaos 接口 |
| `Engine/Source/Runtime/Engine/Classes/PhysicsEngine/` | Engine | 引擎层 | 物理形体、约束、力、物理动画等 UObject |
| `Engine/Source/Runtime/Experimental/ChaosCore/` | ChaosCore | Chaos 核心 | 数学基础库、精度无关类型、粒子容器 |
| `Engine/Source/Runtime/Experimental/Chaos/` | Chaos | Chaos 核心 | 物理运行时：刚体/软体 FSM、碰撞检测、约束求解 |
| `Engine/Source/Runtime/Experimental/ChaosSolverEngine/` | ChaosSolverEngine | 求解器引擎 | 求解器 Actor、物理事件转发、CVD 桥接 |
| `Engine/Source/Runtime/Experimental/ChaosVehicles/` | ChaosVehicles | 专项物理 | 车辆动力学：引擎、变速器、悬挂、轮胎 |
| `Engine/Plugins/ChaosCloth/` | ChaosCloth | 专项物理 | 布料模拟（XPBD） |
| `Engine/Plugins/ChaosClothAsset/` | ChaosClothAsset | 专项物理 | 布料资产管线（Dataflow 图） |
| `Engine/Plugins/ChaosClothAssetDataflowNodes/` | ChaosClothAssetDataflowNodes | 专项物理 | 布料 Dataflow 节点 |
| `Engine/Plugins/ChaosClothAssetEditorCore/` | ChaosClothAssetEditorCore | 专项物理 | 布料编辑器核心 |
| `Engine/Plugins/ChaosClothAssetEditor/` | ChaosClothAssetEditor | 专项物理 | 布料编辑器 UI |
| `Engine/Plugins/ChaosClothAssetUsdDataflowNodes/` | ChaosClothAssetUsdDataflowNodes | 专项物理 | 布料 USD 导入节点 |
| `Engine/Plugins/ChaosOutfitAsset/` | ChaosOutfitAsset | 专项物理 | 服装资产管理 |
| `Engine/Source/Runtime/Experimental/FieldSystem/` | FieldSystem | 专项物理 | 物理场系统（外力、破坏触发） |
| `Engine/Source/Runtime/Experimental/GeometryCollectionEngine/` | GeometryCollectionEngine | 专项物理 | 破坏系统：可破坏物体、碎片渲染 |
| `Engine/Source/Runtime/Experimental/RigidPhysics/` | RigidPhysics | 专项物理 | 独立刚体物理运行时 |
| `Engine/Source/Runtime/Experimental/ChaosSpatialPartitions/` | ChaosSpatialPartitions | 基础设施 | 通用空间分区数据结构 |
| `Engine/Source/Runtime/Experimental/ChaosVDData/` | ChaosVDData | 基础设施 | CVD 数据定义（USTRUCT） |
| `Engine/Source/Runtime/Experimental/ChaosVisualDebugger/` | ChaosVisualDebugger | 基础设施 | CVD 运行时录制（Trace） |
| `Engine/Plugins/ChaosVD/` | ChaosVD | 调试工具 | CVD 可视化回放工具 |
| `Engine/Plugins/ChaosInsights/` | ChaosInsights | 调试工具 | Insights 物理性能分析 |

## 一、引擎层 —— 物理引擎与游戏线程的桥梁

### 1. `Engine/Source/Runtime/PhysicsCore/`

**核心作用**：物理引擎的引擎无关抽象层，定义物理基础类型、场景查询接口、物理材质基类，以及 Chaos 与 PhysX 的公共接口。引擎代码通常依赖此模块而非直接依赖 Chaos。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `Public/PhysicsCore.h` | 模块入口 |
| `Public/PhysicsInterfaceTypesCore.h` | 物理接口通用类型（`FPhysicsShapeHandle`、`FPhysicsGeometry` 等） |
| `Public/PhysicsInterfaceUtilsCore.h` | 几何创建/转换工具函数 |
| `Public/PhysicsSQ.h` / `SQVisitor.h` / `SQAccelerator.h` | 场景查询（Scene Query）抽象：Raycast、Sweep、Overlap 的加速结构与访问者模式 |
| `Public/Chaos/ChaosEngineInterface.h` | Chaos 专用的引擎接口适配层 |
| `Public/Chaos/ChaosScene.h` | Chaos 场景的引擎侧包装 |
| `Public/Chaos/ChaosPhysicalMaterial.h` | Chaos 物理材质 |
| `Public/Chaos/ChaosConstraintSettings.h` | 约束参数缩放（Joint Stiffness、Drive 刚度/阻尼缩放等） |
| `Public/Chaos/ChaosUserEntity.h` | 用户自定义物理实体 |
| `Public/PhysicalMaterials/` | 物理材质与物理材质属性基类 |
| `Public/PhysXSupportCore.h` | PhysX 兼容层（历史遗留，逐步废弃） |
| `Private/` | 对应实现文件 |

**核心算法**：
- 场景查询（Scene Query）加速：`SQAccelerator` 提供基于空间加速结构的快速 Raycast/Sweep/Overlap
- 物理材质属性系统：`PhysicalMaterial` 定义摩擦、弹性、密度等物理属性

---

### 2. `Engine/Source/Runtime/Engine/Public/Physics/`

**核心作用**：物理引擎在 Engine 模块中的公共接口头文件，定义物理场景、物理接口声明、网络物理同步、即时物理模式等。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `PhysScene.h` / `PhysScene_PhysX.h` | 物理场景（`FPhysScene`）：管理物理世界的创建、更新与销毁 |
| `PhysicsInterfaceCore.h` / `PhysicsInterfaceDeclares.h` | 物理接口声明与类型定义 |
| `PhysicsInterfacePhysX.h` | PhysX 后端接口 |
| `PhysicsInterfaceTypes.h` / `PhysicsInterfaceUtils.h` | 物理接口类型与工具函数 |
| `PhysicsFiltering.h` | 碰撞过滤：通道（Channel）与响应（Response） |
| `PhysicsGeometry.h` | 物理几何体定义 |
| `PhysicsQueryHandler.h` / `DefaultPhysicsQueryHandler.h` | 物理查询处理器 |
| `SceneQueryData.h` | 场景查询数据 |
| `AsyncPhysicsData.h` | 异步物理数据（用于网络复制） |
| `AsyncPhysicsInputComponent.h` | 异步物理输入组件 |
| `NetworkPhysicsComponent.h` | 网络物理同步组件 |
| `PhysicsReplicationCache.h` / `PhysicsReplicationQuantization.h` | 物理复制缓存与量化 |
| `SimpleSuspension.h` | 简易悬挂系统 |
| `GenericPhysicsInterface.h` | 通用物理接口（引擎层统一入口） |
| `Experimental/` | 实验性物理接口（Chaos 接口适配） |
| `ImmediatePhysics/` | 即时物理（Immediate Physics）模式：不依赖物理线程，直接在游戏线程同步计算 |
| `Tests/` | 物理测试 |

**`Experimental/` 子目录关键文件**：
| 文件 | 作用 |
|---|---|
| `PhysInterface_Chaos.h` | Chaos 物理接口实现 |
| `PhysScene_Chaos.h` | Chaos 物理场景 |
| `ChaosInterfaceUtils.h` | Chaos 接口工具 |
| `ChaosInterfaceWrapper.h` | Chaos 接口包装器 |
| `ChaosEventRelay.h` / `ChaosEventType.h` | Chaos 事件中继与类型 |
| `ChaosDerivedDataReader.h` | Chaos DDC 数据读取 |
| `ChaosScopedSceneLock.h` | 场景锁 RAII 封装 |

**`ImmediatePhysics/` 子目录**：
| 文件/目录 | 作用 |
|---|---|
| `ImmediatePhysicsSimulation.h` | 即时物理模拟主接口 |
| `ImmediatePhysicsActorHandle.h` | 即时物理 Actor 句柄 |
| `ImmediatePhysicsJointHandle.h` | 即时物理 Joint 句柄 |
| `ImmediatePhysicsAdapters.h` | 即时物理适配器 |
| `ImmediatePhysicsChaos/` | Chaos 后端的即时物理实现 |
| `ImmediatePhysicsPhysX/` | PhysX 后端的即时物理实现（历史遗留） |
| `ImmediatePhysicsShared/` | 即时物理共享类型 |

**核心算法**：
- **碰撞过滤系统**：基于通道（Channel）与响应（Response）的碰撞过滤矩阵
- **物理场景查询**：统一的 Raycast/Sweep/Overlap 接口，支持同步与异步
- **即时物理（Immediate Physics）**：在游戏线程同步执行的简化物理模拟，用于 AnimNode 等场景，避免物理线程延迟
- **网络物理复制**：异步物理数据缓存与量化，支持物理状态的网络同步

---

### 3. `Engine/Source/Runtime/Engine/Classes/PhysicsEngine/`

**核心作用**：物理引擎的 UObject 层定义——`UBodySetup`、`UBodyInstance`、`UPhysicsAsset`、各种物理约束组件、物理力组件等。这是游戏逻辑与物理引擎交互的主要入口。

**关键目录与文件**：
| 文件 | 作用 |
|---|---|
| `BodySetup.h` / `BodyInstance.h` | 物理形体定义与实例（每个 PrimitiveComponent 持有一个 BodyInstance） |
| `PhysicsAsset.h` / `SkeletalBodySetup.h` | 物理资产：骨骼网格的物理形体配置 |
| `ConstraintInstance.h` / `ConstraintTypes.h` | 物理约束（Constraint）：Fixed、Hinge、Slider、BallSocket 等 |
| `ConstraintInstanceBlueprintLibrary.h` | 约束蓝图函数库 |
| `ConstraintDrives.h` / `ConstraintUtils.h` | 约束驱动（Motor/Spring）与工具函数 |
| `PhysicsConstraintComponent.h` | 物理约束组件（场景中可放置的约束 Actor） |
| `PhysicsConstraintActor.h` / `PhysicsConstraintTemplate.h` | 约束 Actor 与模板 |
| `PhysicsHandleComponent.h` | 物理抓取组件（Physics Handle） |
| `PhysicsThrusterComponent.h` / `PhysicsThruster.h` | 推进器组件（推力） |
| `PhysicsSpringComponent.h` | 弹簧组件 |
| `RadialForceComponent.h` / `RadialForceActor.h` | 径向力组件（爆炸/脉冲） |
| `PhysicalAnimationComponent.h` | 物理动画组件：驱动骨骼网格跟随动画，同时保持物理模拟 |
| `PhysicsCollisionHandler.h` | 碰撞处理器 |
| `PhysicsSettings.h` | 物理全局设置（`UPhysicsSettings`） |
| `PhysicsObjectBlueprintLibrary.h` | PhysicsObject 蓝图函数库 |
| `PhysicsObjectExternalInterface.h` | PhysicsObject 外部接口 |
| `ClusterUnionActor.h` / `ClusterUnionComponent.h` | Cluster Union（集群联合）：将多个刚体聚类为一个联合体 |
| `ClusterUnionReplicatedProxyComponent.h` | Cluster Union 网络复制代理 |
| `ChaosBlueprintLibrary.h` | Chaos 蓝图函数库 |
| `AggregateGeom.h` | 聚合几何体（多个几何体的集合） |
| `ShapeElem.h` / `BoxElem.h` / `SphereElem.h` / `SphylElem.h` / `TaperedCapsuleElem.h` / `ConvexElem.h` / `LevelSetElem.h` / `MLLevelSetElem.h` / `SkinnedLevelSetElem.h` / `SkinnedTriangleMeshElem.h` | 各形状元素定义 |
| `BodySetupObjectTextFactory.h` | 物理形体对象文本工厂 |
| `ExternalSpatialAccelerationPayload.h` | 外部空间加速载荷 |
| `EnvironmentalCollisions.h` | 环境碰撞 |
| `PhysicsBodyInstanceOwnerInterface.h` | BodyInstance 所有者接口 |
| `PhysicsLogUtil.h` | 物理日志工具 |
| `RigidBodyBase.h` / `RigidBodyIndexPair.h` | 刚体基类与索引对 |
| `SafePhysicsObjectHandle.h` | 安全物理对象句柄 |

**核心算法**：
- **物理形体的创建与管理**：`UBodySetup` 定义碰撞几何（盒、球、胶囊、凸包、LevelSet），`FBodyInstance` 管理运行时状态
- **约束系统**：支持 Fixed、Hinge、Slider、BallSocket、Prismatic 等约束类型，支持 Motor（驱动）和 Spring（弹簧）
- **物理动画（Physical Animation）**：混合动画驱动与物理模拟，使用 PD 控制器跟踪动画目标姿态
- **Cluster Union**：将多个独立刚体聚类为联合体，减少约束求解复杂度

---

## 二、Chaos 核心 —— 物理模拟运行时

### 4. `Engine/Source/Runtime/Experimental/ChaosCore/`

**核心作用**：Chaos 物理引擎的数学基础库，提供精度无关的类型别名、向量/矩阵/四元数运算、空间数据结构基类、粒子属性容器等。所有 Chaos 模块共享的底层基础。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `Public/Chaos/` | 公开头文件目录 |
| `Public/Chaos/Core.h` | 核心类型别名：`FReal`（double）、`FVec3`、`FRotation3`、`FRigidTransform3` 等 |
| `Public/Chaos/Real.h` | 精度定义：`FReal`（默认 double）、`FRealSingle`（float） |
| `Public/Chaos/Vector.h` | 向量类型（`TVector<T, d>`）：3D 向量运算 |
| `Public/Chaos/Matrix.h` | 矩阵类型（`TMatrix<T, d>`）：3x3 矩阵运算 |
| `Public/Chaos/Transform.h` | 变换类型（`TRigidTransform<T, d>`）：位置+旋转 |
| `Public/Chaos/Quaternion.h` | 四元数类型 |
| `Public/Chaos/Rotation.h` | 旋转类型 |
| `Public/Chaos/Array.h` | 粒子属性数组（`TArrayCollectionArray`、`TArrayCollection`） |
| `Public/Chaos/ArrayCollection.h` | 并行数组集合：按索引对齐的多属性数组 |
| `Public/Chaos/ObjectPool.h` | 对象池（`TObjectPool`）：无堆分配的热路径内存管理 |
| `Public/Chaos/Pair.h` | 值对类型 |
| `Public/Chaos/Tuple.h` | 元组类型 |
| `Public/Chaos/ChaosArchive.h` | 序列化归档器 |
| `Public/Chaos/ChaosCheck.h` | `CHAOS_CHECK`/`CHAOS_ENSURE` 宏（可独立开关的物理断言） |
| `Public/Chaos/ChaosVersion.h` | 版本 GUID：DDC/序列化版本控制 |
| `Public/Chaos/ISPC/` | ISPC 加速的向量运算（SIMD） |
| `Private/` | 对应实现文件 |
| `Tests/` | 单元测试 |
| `README.md` | 模块文档 |

**核心算法**：
- **精度无关数学库**：`FReal` 别名实现 double/float 切换，模板化向量/矩阵支持任意精度
- **并行数组容器**：`TArrayCollection` + `TArrayCollectionArray` 实现按索引对齐的 SOA（Structure of Arrays）数据布局，对缓存友好
- **对象池**：`TObjectPool` 实现热路径零堆分配
- **ISPC SIMD 加速**：关键数学运算（向量叉积、矩阵乘法等）使用 ISPC 编译器生成 SIMD 指令

---

### 5. `Engine/Source/Runtime/Experimental/Chaos/`

**核心作用**：Chaos 物理引擎的主运行时模块——刚体/软体/可变形体模拟、碰撞检测、约束求解、场景管理、空间加速结构、序列化等。这是整个 Chaos 系统的核心。

**关键目录与文件**：

| 目录/文件 | 作用 |
|---|---|
| **`Public/Chaos/`** | Chaos 核心公开头文件 |
| `Public/Chaos/PBDRigidsEvolution.h` | 刚体演化主类：管理刚体粒子的时间步进 |
| `Public/Chaos/PBDRigidsEvolutionGBF.h` | Gauss-Seidel 刚体演化 |
| `Public/Chaos/PBDRigidParticles.h` / `PBDRigidClusteredParticles.h` | 刚体粒子数据 |
| `Public/Chaos/PBDEvolution.h` | PBD（Position Based Dynamics）演化基类 |
| `Public/Chaos/SoftsEvolution.h` | 软体演化（XPBD 求解器） |
| `Public/Chaos/NewtonEvolution.h` | 牛顿力学演化（FEM 软体） |
| `Public/Chaos/GeometryParticles.h` | 几何粒子基类 |
| `Public/Chaos/ParticleHandle.h` | 粒子句柄（SOA 索引包装） |
| **`Public/Chaos/Collision/`** | 碰撞检测子系统 |
| `Collision/CollisionDetector.h` | 碰撞检测器接口 |
| `Collision/NarrowPhase.h` | 窄相碰撞检测 |
| `Collision/ParticlePairMidPhase.h` | 中相碰撞检测（粒子对） |
| `Collision/SpatialAccelerationBroadPhase.h` | 基于空间加速的广相碰撞检测 |
| `Collision/SpatialAccelerationCollisionDetector.h` | 空间加速碰撞检测器 |
| `Collision/PBDCollisionConstraint.h` | 碰撞约束 |
| `Collision/PBDCollisionSolverJacobi.h` / `Simd.h` | Jacobi 碰撞求解器（含 SIMD 版本） |
| `Collision/PBDCollisionContainerSolverJacobi.h` / `Simd.h` | 碰撞容器求解器 |
| `Collision/ContactPoint.h` / `ConvexContactPoint.h` | 接触点计算 |
| `Collision/CollisionFilter.h` / `CollisionFilterBits.h` | 碰撞过滤 |
| `Collision/SimRaycast.h` / `SimSweep.h` | 模拟射线检测与扫描 |
| `Collision/StatsData.h` | 碰撞统计 |
| **`Public/Chaos/Evolution/`** | 演化求解器 |
| `Evolution/SolverBody.h` / `SolverBodyContainer.h` | 求解器刚体表示与容器 |
| `Evolution/SolverConstraintContainer.h` | 求解器约束容器 |
| `Evolution/ConstraintGroupSolver.h` | 约束组分批求解 |
| `Evolution/IndexedConstraintContainer.h` | 索引约束容器 |
| `Evolution/SimulationSpace.h` | 模拟空间（世界/局部） |
| `Evolution/PBDMinEvolution.h` | 最小 PBD 演化 |
| `Evolution/SolverPartitionManager.h` | 求解器分区管理器 |
| **`Public/Chaos/Joint/`** | 关节约束 |
| `Joint/PBDJointSolverGaussSeidel.h` | Gauss-Seidel 关节求解器 |
| `Joint/PBDJointCachedSolverGaussSeidel.h` | 缓存优化的 Gauss-Seidel 关节求解器 |
| `Joint/JointSolverConstraints.h` | 关节约束定义 |
| `Joint/ColoringGraph.h` | 约束图着色（并行化约束求解） |
| **`Public/Chaos/Island/`** | 孤岛系统 |
| `Island/IslandManager.h` | 孤岛管理器：将互不接触的刚体分组为孤岛以并行求解 |
| `Island/IslandGroup.h` / `IslandGraph.h` | 孤岛组与孤岛图 |
| `Island/SolverIsland.h` | 求解器孤岛 |
| **`Public/Chaos/Deformable/`** | 可变形体 |
| `Deformable/ChaosDeformableSolver.h` | 可变形体求解器 |
| `Deformable/GaussSeidelCorotatedConstraints.h` | Corotated 有限元约束（GS 求解） |
| `Deformable/GaussSeidelNeohookeanConstraints.h` | NeoHookean 弹性约束 |
| `Deformable/GaussSeidelWeakConstraints.h` | 弱约束（惩罚法） |
| `Deformable/MuscleActivationConstraints.h` | 肌肉激活约束 |
| **`Public/Chaos/Character/`** | 角色物理 |
| `Character/CharacterGroundConstraint.h` | 角色地面约束 |
| `Character/CharacterGroundConstraintContainer.h` | 角色地面约束容器 |
| **`Public/Chaos/Math/`** | 数学工具 |
| `Math/BlockSparseLinearSystem.h` | 块稀疏线性系统 |
| `Math/Krylov.h` | Krylov 子空间迭代法 |
| `Math/Poisson.h` | Poisson 求解器 |
| **`Public/Chaos/Framework/`** | 框架层 |
| `Framework/PhysicsProxy.h` / `PhysicsProxyBase.h` | 物理代理基类（游戏线程与物理线程的桥梁） |
| `Framework/PhysicsSolverBase.h` | 物理求解器基类 |
| `Framework/Handles.h` | 句柄系统 |
| `Framework/Parallel.h` | 并行框架 |
| `Framework/BufferedData.h` / `MultiBufferResource.h` / `TripleBufferedData.h` | 多缓冲数据（线程安全数据交换） |
| `Framework/ChaosResultsManager.h` | 结果管理器 |
| **`Public/PhysicsProxy/`** | 物理代理实现 |
| `PhysicsProxy/SingleParticlePhysicsProxy.h` | 单粒子代理（普通刚体） |
| `PhysicsProxy/JointConstraintProxy.h` | 关节约束代理 |
| `PhysicsProxy/GeometryCollectionPhysicsProxy.h` | 几何集合代理（破坏系统） |
| `PhysicsProxy/SkeletalMeshPhysicsProxy.h` | 骨骼网格物理代理 |
| `PhysicsProxy/StaticMeshPhysicsProxy.h` | 静态网格物理代理 |
| `PhysicsProxy/ClusterUnionPhysicsProxy.h` | Cluster Union 代理 |
| `PhysicsProxy/CharacterGroundConstraintProxy.h` | 角色地面约束代理 |
| `PhysicsProxy/SuspensionConstraintProxy.h` | 悬挂约束代理 |
| `PhysicsProxy/PerSolverFieldSystem.h` | 每求解器场系统 |
| **`Public/Chaos/Serialization/`** | 序列化 |
| `Serialization/SerializedMultiPhysicsState.h` | 多物理状态序列化 |
| `Serialization/SolverSerializer.h` | 求解器序列化器 |
| **`Public/ChaosVisualDebugger/`** | 可视化调试器运行时 |
| `ChaosVisualDebugger/ChaosVDTraceMacros.h` | CVD 追踪宏 |
| `ChaosVisualDebugger/ChaosVDDataWrapperUtils.h` | CVD 数据包装器工具 |
| **`Public/ChaosDebugDraw/`** | 调试绘制 |
| `ChaosDebugDraw/ChaosDDContext.h` / `ChaosDDScene.h` / `ChaosDDTimeline.h` | 调试绘制上下文/场景/时间线 |
| `ChaosDebugDraw/ChaosDDRenderer.h` | 调试绘制渲染器 |
| **`Public/GeometryCollection/`** | 几何集合（破坏系统） |
| `GeometryCollection/GeometryCollection.h` | 几何集合核心类型 |
| `GeometryCollection/ManagedArrayCollection.h` | 托管数组集合 |
| `GeometryCollection/GeometryCollectionAlgo.h` | 几何集合算法 |
| `GeometryCollection/GeometryCollectionClusteringUtility.h` | 聚类工具 |
| `GeometryCollection/TransformCollection.h` | 变换集合 |
| `GeometryCollection/Facades/` | 立面模式（Facade）访问器 |
| **`Public/Field/`** | 场系统（Field System） |
| `Field/FieldSystem.h` | 场系统核心 |
| `Field/FieldSystemNodes.h` | 场节点 |
| `Field/FieldSystemCoreAlgo.h` | 场核心算法 |
| `Field/FieldSystemNoiseAlgo.h` | 场噪声算法 |
| `Field/FieldSystemSimulationCoreAlgo.h` | 场模拟核心算法 |
| `Field/FieldArrayView.h` | 场数组视图 |
| **`Public/Framework/`** | 框架层 |
| `Framework/Threading.h` | 线程锁（`FPhysicsSceneGuardScopedWrite`/`FPhysicsSceneGuardScopedRead`） |
| `Framework/TimeStep.h` | 时间步管理 |
| **`Public/ChaosInsights/`** | Chaos Insights 集成 |
| `ChaosInsights/ChaosInsightsMacros.h` | Insights 宏 |

**核心算法**：

| 算法 | 说明 |
|---|---|
| **PBD（Position Based Dynamics）** | 直接操作粒子位置而非速度/加速度的约束求解方法，无条件稳定，适合大时间步 |
| **XPBD（Extended PBD）** | 引入合规（Compliance）矩阵的 PBD 扩展，约束刚度与时间步无关 |
| **Gauss-Seidel 迭代** | 逐约束串行求解，收敛快但难以并行化；用于关节约束和碰撞约束 |
| **Jacobi 迭代** | 逐约束并行求解，适合 GPU/SIMD 加速；用于碰撞求解器 |
| **GJK/EPA** | 凸体碰撞检测的经典算法：GJK（Gilbert-Johnson-Keerthi）计算最近距离，EPA（Expanding Polytope Algorithm）计算穿透深度 |
| **SAT（Separating Axis Theorem）** | 凸体碰撞检测的分离轴定理 |
| **AABB 树（AABBTree）** | 轴对齐包围盒层次树，用于空间加速查询和碰撞检测 |
| **Bounding Volume Hierarchy** | 包围体层次结构，加速场景查询 |
| **Spatial Hash / Hierarchical Spatial Hash** | 空间哈希，用于粒子和大量物体的快速邻居查找 |
| **Island（孤岛）系统** | 将互不接触的刚体分组为孤岛，每个孤岛内独立求解约束，不同孤岛间可并行 |
| **Graph Coloring（图着色）** | 约束图着色，将约束分为不相交的组，组内并行求解 |
| **Corotated FEM** | 共旋有限元法：提取刚性旋转分量后的线性弹性，用于可变形体模拟 |
| **NeoHookean 弹性** | NeoHookean 超弹性模型，用于大变形可变形体 |
| **Triple Buffering** | 三缓冲数据交换：游戏线程写 → 物理线程读，避免锁竞争 |
| **Cluster Union** | 将多个刚体聚类为联合层级结构，减少求解器约束数量 |
| **Spatial Acceleration Broad Phase** | 基于空间加速结构的广相碰撞检测，快速筛选可能碰撞的物体对 |
| **Mid Phase** | 中相碰撞检测：对广相筛选出的物体对进行更精细的碰撞检测 |
| **Narrow Phase** | 窄相碰撞检测：精确计算接触点、法向量和穿透深度 |

---

## 三、求解器引擎层

### 6. `Engine/Source/Runtime/Experimental/ChaosSolverEngine/`

**核心作用**：Chaos 求解器与 Engine 模块的桥梁——提供可放置到关卡的求解器 Actor、物理事件到游戏逻辑的转发、以及 Chaos Visual Debugger 与 Engine 的桥接。

**关键目录与文件**：
| 文件 | 作用 |
|---|---|
| `Public/Chaos/ChaosSolverActor.h` | 可放置的 Chaos 求解器 Actor（实验性） |
| `Public/Chaos/ChaosSolver.h` | 求解器 UObject 资产 |
| `Public/Chaos/ChaosSolverSettings.h` | 求解器全局设置 |
| `Public/Chaos/ChaosGameplayEventDispatcher.h` | 物理事件到游戏逻辑的转发（碰撞/断裂/移除/碎裂事件） |
| `Public/Chaos/ChaosEventListenerComponent.h` | 事件监听组件 |
| `Public/Chaos/ChaosNotifyHandlerInterface.h` | 物理通知处理器接口 |
| `Public/Chaos/ChaosDebugDrawComponent.h` | 调试绘制组件 |
| `Public/Chaos/ChaosDebugDrawSubsystem.h` | 调试绘制子系统 |
| `Public/Chaos/ChaosVDEngineEditorBridge.h` | CVD 与 Engine 的桥接 |
| `Public/Chaos/ChaosVDRemoteSessionsManager.h` | CVD 远程会话管理器（已废弃） |
| `Public/Chaos/ChaosVDTraceRelayTransport.h` | CVD Trace 中继传输 |
| `Public/Chaos/ChaosSolverEnginePlugin.h` | 插件入口 |
| `Public/Chaos/ChaosSolverComponentTypes.h` | 求解器组件类型 |

**核心算法**：
- 物理事件转发：将物理线程的碰撞/断裂事件异步转发到游戏线程，通过 `IChaosNotifyHandlerInterface` 通知游戏逻辑
- 求解器 Actor：支持在关卡中放置独立求解器实例，控制求解参数

---

## 四、专项物理系统

### 7. `Engine/Source/Runtime/Experimental/ChaosVehicles/`

**核心作用**：基于 Chaos 的车辆物理系统——模拟车辆动力学，包括引擎、变速器、悬挂、轮胎、空气动力学等子系统。

**模块结构**：
- `ChaosVehiclesCore/`：引擎无关的车辆物理核心（纯物理计算，无 UObject 依赖）
- `ChaosVehiclesEngine/`：引擎层的车辆物理封装（UObject 集成）

**`ChaosVehiclesCore/Public/` 关键文件**：
| 文件 | 作用 |
|---|---|
| `SimModule/SimulationModuleBase.h` | 模拟模块基类 |
| `SimModule/EngineModule.h` | 引擎模块：转速-扭矩曲线，油门响应 |
| `SimModule/TransmissionModule.h` | 变速器模块：档位切换、传动比 |
| `SimModule/ClutchModule.h` | 离合器模块 |
| `SimModule/WheelModule.h` | 车轮模块 |
| `SimModule/AxleModule.h` | 车轴模块 |
| `SimModule/SuspensionBaseInterface.h` | 悬挂接口 |
| `SimModule/AerofoilModule.h` | 空气动力学翼面模块 |
| `SimModule/ChassisModule.h` | 底盘模块 |
| `SimModule/MotorModule.h` / `TorqueSimModule.h` | 电机/扭矩模块 |
| `SimModule/ThrusterModule.h` | 推进器模块 |
| `SimModule/SimModuleTree.h` | 模拟模块树（层级组织） |
| `SimModule/VehicleBlackboard.h` | 车辆黑板（模块间数据共享） |
| `SimModule/ModuleInput.h` | 模块输入 |
| `SimModule/DeferredForcesModular.h` | 延迟力计算 |
| `EngineSystem.h` | 引擎系统 |
| `TransmissionSystem.h` | 变速器系统 |
| `SuspensionSystem.h` | 悬挂系统 |
| `TireSystem.h` | 轮胎系统（Pacejka 等轮胎模型） |
| `WheelSystem.h` | 车轮系统 |
| `SteeringSystem.h` / `SteeringUtility.h` | 转向系统 |
| `AerodynamicsSystem.h` / `AerofoilSystem.h` | 空气动力学系统 |
| `ThrustSystem.h` | 推力系统 |
| `ArcadeSystem.h` | 街机风格的简化车辆物理 |
| `SimpleVehicle.h` | 简易车辆 |
| `VehicleSystemTemplate.h` | 车辆系统模板 |
| `VehicleUtility.h` | 车辆工具函数 |

**核心算法**：
- **模块化车辆模拟**：基于 `SimModuleTree` 的层级化模拟模块，引擎→离合器→变速器→车轴→车轮，力沿树传递
- **Pacejka 轮胎模型**：使用 Pacejka 魔术公式计算轮胎侧向力与纵向力
- **悬挂系统**：弹簧-阻尼悬挂模型，支持碰撞检测
- **空气动力学**：下压力与阻力计算
- **车辆黑板（Blackboard）**：模块间通过黑板共享数据（转速、扭矩、车速等），实现解耦

---

### 8. `Engine/Plugins/ChaosCloth/`

**核心作用**：基于 Chaos 的布料模拟插件——提供布料物理模拟、布料渲染、布料编辑器的完整功能。

**模块结构**：
| 模块 | 作用 |
|---|---|
| `ChaosCloth` | 运行时布料模拟（XPBD 求解器） |
| `ChaosClothEditor` | 编辑器工具：布料资产的编辑与预览 |

**核心算法**：
- **XPBD 布料模拟**：使用 Extended Position Based Dynamics 进行布料约束求解，包括拉伸约束、弯曲约束、碰撞约束等
- **布料碰撞检测**：与角色（骨骼网格）的碰撞检测与响应
- **布料蒙皮**：将布料粒子绑定到骨骼动画
- **自碰撞（Self Collision）**：布料自身的碰撞检测

---

### 9. `Engine/Plugins/ChaosClothAsset/`

**核心作用**：Chaos 布料资产管线——支持通过 Dataflow 图构建布料资产的创建、编辑与导入流程。

**模块结构**：
| 模块 | 作用 |
|---|---|
| `ChaosClothAsset` | 布料资产运行时类型 |
| `ChaosClothAssetEngine` | 布料资产引擎集成 |
| `ChaosClothAssetTools` | 布料资产工具 |

**关联插件**：
| 插件 | 作用 |
|---|---|
| `ChaosClothAssetDataflowNodes` | 布料资产的 Dataflow 节点（用于 Cloth Editor 的图编辑） |
| `ChaosClothAssetEditorCore` | 布料资产编辑器核心 |
| `ChaosClothAssetEditor` | 布料资产编辑器 UI |
| `ChaosClothAssetUsdDataflowNodes` | 布料资产的 USD 导入 Dataflow 节点 |

---

### 10. `Engine/Plugins/ChaosOutfitAsset/`

**核心作用**：Chaos 服装资产——管理多层布料服装的资产类型，支持服装组合与布料物理的集成。

**模块结构**：
| 模块 | 作用 |
|---|---|
| `ChaosOutfitAssetDataflowNodes` | 服装资产的 Dataflow 节点 |
| `ChaosOutfitAssetEditor` | 服装资产编辑器 |
| `ChaosOutfitAssetEngine` | 服装资产引擎集成 |

---

### 11. `Engine/Source/Runtime/Experimental/FieldSystem/`

**核心作用**：场系统（Field System）——基于节点图的物理场系统，用于在运行时施加外力、破坏、锚定等物理效果。支持在 Chaos 求解器内作为外部力/约束注入。

**模块结构**：
- `Source/FieldSystemEngine/`：引擎层场系统封装

**关键文件**：
| 文件 | 作用 |
|---|---|
| `Field/FieldSystemActor.h` | 可放置在场中的场系统 Actor |
| `Field/FieldSystemAsset.h` | 场系统资产 |
| `Field/FieldSystemComponent.h` | 场系统组件 |
| `Field/FieldSystemObjects.h` | 场系统对象（各种场节点） |
| `Field/FieldSystemComponentTypes.h` | 场系统组件类型 |

**核心算法**：
- **场节点图**：基于图的可组合场系统，节点包括 `RadialFalloff`、`UniformVector`、`NoiseField`、`CullingField` 等
- **噪声算法**：Perlin 噪声、随机噪声，用于产生自然扰动
- **场与物理的交互**：场作为外力、内部应力、锚定约束注入 Chaos 求解器
- **破坏系统集成**：场用于触发和控制 Geometry Collection 的断裂与碎裂

---

### 12. `Engine/Source/Runtime/Experimental/GeometryCollectionEngine/`

**核心作用**：几何集合引擎（Geometry Collection Engine）——基于 Chaos 的破坏系统，支持可破坏物体的创建、模拟、渲染与缓存。

**关键文件**：
| 文件 | 作用 |
|---|---|
| `GeometryCollection/GeometryCollectionComponent.h` | 几何集合组件（核心渲染+模拟组件） |
| `GeometryCollection/GeometryCollectionActor.h` | 几何集合 Actor |
| `GeometryCollection/GeometryCollectionObject.h` | 几何集合 UObject |
| `GeometryCollection/GeometryCollectionCache.h` | 几何集合缓存（录制/回放模拟） |
| `GeometryCollection/GeometryCollectionDebugDraw.h` | 调试绘制 |
| `GeometryCollection/GeometryCollectionDebugDrawActor.h` | 调试绘制 Actor |
| `GeometryCollection/GeometryCollectionDebugDrawComponent.h` | 调试绘制组件 |
| `GeometryCollection/GeometryCollectionEngineConversion.h` | 引擎转换（StaticMesh → GeometryCollection） |
| `GeometryCollection/GeometryCollectionEngineRemoval.h` | 碎片移除 |
| `GeometryCollection/GeometryCollectionEngineUtility.h` | 引擎工具 |
| `GeometryCollection/GeometryCollectionEngineTypes.h` | 引擎类型 |
| `GeometryCollection/GeometryCollectionDamagePropagationData.h` | 损伤传播数据 |
| `GeometryCollection/GeometryCollectionExternalRenderInterface.h` | 外部渲染接口 |
| `GeometryCollection/GeometryCollectionHitProxy.h` | 点击代理 |
| `GeometryCollection/GeometryCollectionParticlesData.h` | 粒子数据 |
| `GeometryCollection/GeometryCollectionRenderLevelSetActor.h` | LevelSet 渲染 Actor |
| `GeometryCollection/GeometryCollectionRootProxyRenderer.h` | 根代理渲染器 |
| `GeometryCollection/GeometryCollectionBlueprintLibrary.h` | 蓝图函数库 |
| `GeometryCollection/GeometryCollectionISMPoolActor.h` | ISM 池 Actor |
| `GeometryCollection/GeometryCollectionISMPoolComponent.h` | ISM 池组件 |
| `GeometryCollection/GeometryCollectionISMPoolSubSystem.h` | ISM 池子系统 |
| `GeometryCollection/PhysicsAssetSimulation.h` | 物理资产模拟 |
| `GeometryCollection/DerivedDataGeometryCollectionCooker.h` | DDC 烹饪器 |
| `GeometryCollection/CollectionMaterialFacade.h` | 材质 Facade |
| `GeometryCollection/GeometryCollectionCreationParameters.h` | 创建参数 |
| `GeometryCollection/GeometryCollectionEditorSelection.h` | 编辑器选择 |
| `GeometryCollection/GeometryCollectionEngineSizeSpecificUtility.h` | 尺寸相关工具 |
| `ChaosBlueprint.h` | Chaos 蓝图 |
| `ChaosBreakingEventFilter.h` | 断裂事件过滤器 |
| `ChaosCollisionEventFilter.h` | 碰撞事件过滤器 |
| `ChaosRemovalEventFilter.h` | 移除事件过滤器 |
| `ChaosTrailingEventFilter.h` | 尾迹事件过滤器 |
| `ChaosFilter.h` | 事件过滤器基类 |

**核心算法**：
- **破坏系统**：基于几何集合的层级破坏——物体被击中后裂解为预设碎片，碎片间由 Cluster 连接
- **损伤传播**：基于应变和碰撞力的损伤累积与传播模型
- **ISM 池渲染**：使用 Instanced Static Mesh 池高效渲染大量碎片
- **模拟缓存**：支持录制和回放破坏模拟，用于过场动画
- **LevelSet 碰撞**：使用 LevelSet 表示碎片碰撞几何

---

### 13. `Engine/Source/Runtime/Experimental/RigidPhysics/`

**核心作用**：刚体物理（RigidPhysics）——一个独立的、更轻量级的刚体物理运行时，不与 Chaos 共享代码路径。提供刚体、关节约束、场景管理、材质等基础物理功能。

**关键文件**：
| 文件 | 作用 |
|---|---|
| `RigidPhysics/RigidBody.h` | 刚体定义 |
| `RigidPhysics/RigidBodyHandle.h` | 刚体句柄 |
| `RigidPhysics/RigidBodyContainer.h` | 刚体容器 |
| `RigidPhysics/RigidBodyContainerHandle.h` | 刚体容器句柄 |
| `RigidPhysics/JointConstraint.h` | 关节约束 |
| `RigidPhysics/JointConstraint6DOF.h` | 6-DOF 关节约束 |
| `RigidPhysics/JointConstraintHandle.h` | 关节约束句柄 |
| `RigidPhysics/RigidScene.h` | 刚体场景 |
| `RigidPhysics/RigidSceneHandle.h` | 刚体场景句柄 |
| `RigidPhysics/RigidSceneSettings.h` | 刚体场景设置 |
| `RigidPhysics/RigidContext.h` | 刚体上下文 |
| `RigidPhysics/RigidFactory.h` | 刚体工厂 |
| `RigidPhysics/RigidGeometryCollection.h` | 刚体几何集合 |
| `RigidPhysics/RigidMaterials.h` | 刚体材质 |
| `RigidPhysics/RigidMaterialIndex.h` | 刚体材质索引 |
| `RigidPhysics/RigidModifier.h` | 刚体修改器 |
| `RigidPhysics/RigidShapeInstance.h` | 刚体形状实例 |
| `RigidPhysics/RigidShapeInstanceSetup.h` | 刚体形状实例设置 |
| `RigidPhysics/RigidObjectId.h` / `RigidObjectPtr.h` | 对象 ID 与指针 |
| `RigidPhysics/RigidObjectRegistry.h` | 对象注册表 |
| `RigidPhysics/RigidSceneRegistry.h` | 场景注册表 |
| `RigidPhysics/RigidPhysicsService.h` | 物理服务 |
| `RigidPhysics/RigidLockable.h` | 线程安全锁 |
| `RigidPhysics/RigidFwd.h` | 前向声明 |
| `RigidPhysics/RigidLog.h` | 日志 |
| `RigidPhysics/RigidTyped.h` | 类型系统 |
| `RigidPhysics/Geometry/` | 几何体定义 |
| `RigidPhysics/Internal/` | 内部实现 |

**核心算法**：
- 独立的刚体物理求解器，与 Chaos 主模块并行存在
- 支持 6-DOF 关节约束
- 轻量级场景管理

---

## 五、空间与数据基础设施

### 14. `Engine/Source/Runtime/Experimental/ChaosSpatialPartitions/`

**核心作用**：Chaos 空间分区系统——提供通用的空间分区数据结构与算法，用于加速物理场景的空间查询和碰撞检测。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `Public/ChaosSpatialPartitions/ISpatialPartition.h` | 空间分区接口 |
| `Public/ChaosSpatialPartitions/SpatialHandle.h` | 空间句柄 |
| `Public/ChaosSpatialPartitions/SpatialClassification.h` | 空间分类 |
| `Public/ChaosSpatialPartitions/QueryData.h` | 查询数据 |
| `Public/ChaosSpatialPartitions/Visitors.h` | 空间访问者模式 |
| `Public/ChaosSpatialPartitions/Common.h` | 公共类型 |
| `Public/ChaosSpatialPartitions/Algorithms/` | 空间分区算法 |
| `Public/ChaosSpatialPartitions/Collections/` | 空间分区集合 |
| `Public/ChaosSpatialPartitions/Library/` | 空间分区库 |
| `Tests/` | 单元测试 |

**核心算法**：
- **通用空间分区框架**：抽象的空间分区接口，支持多种分区策略（均匀网格、KD 树、BVH 等）
- **空间访问者模式**：通过 Visitor 遍历空间分区，执行查询、更新等操作
- **空间分类**：基于空间属性的对象分类

---

### 15. `Engine/Source/Runtime/Experimental/ChaosVDData/`

**核心作用**：Chaos Visual Debugger 数据定义——定义 CVD 录制/回放使用的数据结构（USTRUCT），不依赖 Chaos 运行时，仅供 CVD 录制侧和回放侧共享。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `Public/` | CVD 数据包装器（USTRUCT） |
| `Private/` | 实现文件 |

**核心设计**：
- 与 Chaos 运行时零依赖——数据包装器是纯反射 USTRUCT，不依赖 Chaos/ChaosCore
- 录制侧与回放侧共享的序列化契约

---

### 16. `Engine/Source/Runtime/Experimental/ChaosVisualDebugger/`

**核心作用**：Chaos Visual Debugger 运行时（ChaosVDRuntime）——录制物理模拟状态（粒子、碰撞、约束、查询）到 UE Trace 流，供 ChaosVD 工具离线回放分析。

**关键目录与文件**：
| 文件/目录 | 作用 |
|---|---|
| `Public/ChaosVDRuntimeModule.h` | 运行时模块：管理录制生命周期 |
| `Public/DataWrappers/` | 数据包装器：录制数据的 USTRUCT 定义 |
| `Private/` | 录制逻辑实现 |

**核心算法**：
- **周期性全量捕获**：按 `p.Chaos.VD.TimeBetweenFullCaptures` 间隔录制完整物理状态
- **Trace 通道录制**：通过 `ChaosVDChannel` Trace 通道录制，支持实时流式传输
- **录制模式**：支持 Full（全量）、Light（轻量）、QueryOnly（仅查询）等录制模式

---

## 六、工具与调试

### 17. `Engine/Plugins/ChaosVD/`

**核心作用**：Chaos Visual Debugger 前端工具——可视化回放 Chaos 物理模拟录制数据，支持逐帧分析、碰撞可视化、约束可视化、粒子状态查看等。

**模块结构**：
| 模块 | 作用 |
|---|---|
| `ChaosVD` | CVD 主工具：UI 框架、回放控制、数据解码 |
| `ChaosVDBlueprint` | CVD 蓝图支持 |
| `ChaosVDBuiltInExtensions` | CVD 内置扩展 |

---

### 18. `Engine/Plugins/ChaosInsights/`

**核心作用**：Chaos Insights——将 Chaos 物理性能数据集成到 Unreal Insights 性能分析工具中，支持物理求解器耗时、碰撞检测耗时、约束求解耗时等性能分析。

**模块结构**：
| 模块 | 作用 |
|---|---|
| `ChaosInsightsAnalysis` | 物理性能数据分析 |
| `ChaosInsightsUI` | Insights 中的物理面板 UI |

---

## 七、模块依赖关系图

```mermaid
flowchart TB
    subgraph EngineLayer[引擎层]
        PhysicsCore[PhysicsCore<br/>物理接口抽象]
        EnginePhysics[Engine/Public/Physics<br/>物理场景与网络]
        PhysicsEngine[Engine/Classes/PhysicsEngine<br/>UObject 物理组件]
    end

    subgraph ChaosCoreGroup[Chaos 核心]
        ChaosCore[ChaosCore<br/>数学基础库]
        Chaos[Chaos<br/>物理运行时]
        ChaosSpatial[ChaosSpatialPartitions<br/>空间分区]
    end

    subgraph SolverEngine[求解器引擎]
        SolverEngine[ChaosSolverEngine<br/>求解器-引擎桥接]
    end

    subgraph Specialized[专项物理系统]
        Vehicles[ChaosVehicles<br/>车辆物理]
        Cloth[ChaosCloth<br/>布料模拟]
        ClothAsset[ChaosClothAsset<br/>布料资产管线]
        OutfitAsset[ChaosOutfitAsset<br/>服装资产]
        FieldSystem[FieldSystem<br/>场系统]
        GeometryCollection[GeometryCollectionEngine<br/>破坏系统]
        RigidPhysics[RigidPhysics<br/>独立刚体物理]
    end

    subgraph DebugTools[调试与工具]
        ChaosVD[ChaosVD<br/>可视化调试器]
        ChaosVDData[ChaosVDData<br/>CVD 数据定义]
        ChaosVDRuntime[ChaosVisualDebugger<br/>CVD 运行时录制]
        ChaosInsights[ChaosInsights<br/>性能分析]
    end

    PhysicsCore --> EnginePhysics
    EnginePhysics --> PhysicsEngine
    PhysicsCore --> ChaosCore
    ChaosCore --> Chaos
    Chaos --> ChaosSpatial
    Chaos --> SolverEngine
    Chaos --> Vehicles
    Chaos --> Cloth
    Chaos --> FieldSystem
    Chaos --> GeometryCollection
    Chaos --> RigidPhysics
    SolverEngine --> PhysicsEngine
    ChaosVDRuntime --> ChaosVDData
    ChaosVD --> ChaosVDData
    ChaosVD --> ChaosVDRuntime
    ChaosInsights --> Chaos
    ClothAsset --> Cloth
    OutfitAsset --> ClothAsset
```