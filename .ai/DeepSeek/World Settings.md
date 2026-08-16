# UE5 World Settings 解析

## 1. 定位与作用

**World Settings** 是每张地图的全局配置对象，运行时类型为 `AWorldSettings`，定义于 `Engine/Classes/GameFramework/WorldSettings.h`，实现在 `Engine/Private/WorldSettings.cpp`。它是一个不可放置的 `AInfo` Actor：持久关卡拥有其主实例，子关卡关联到该实例。因此，这里的改动通常保存到当前 `.umap`，而不是项目级配置。

在编辑器中可通过 **Window > World Settings** 打开。常用入口也包括关卡蓝图的 **Get World Settings** 节点。

`AWorldSettings` 会复制暂停者、全局/过场时间膨胀、世界重力、高优先级加载和 Nanite 设置；其本身始终相关。因此这些状态在联网游戏中能同步到客户端，但并不意味着所有面板项都会动态复制。

## 2. 先区分三类设置

| 类型 | 保存位置与影响范围 | 例子 |
| --- | --- | --- |
| 地图设置 | 当前地图的 `AWorldSettings` | GameMode Override、Kill Z、World to Meters |
| 项目设置 | `.ini`，作为地图未覆盖时的默认值 | 默认重力、默认 GameMode、导航系统默认配置 |
| 运行时状态 | 由代码改变，部分会复制 | `TimeDilation`、`WorldGravityZ`、`NaniteSettings` |

遇到“改了不生效”时，先检查：当前运行的是不是该地图、地图是否被 GameMode URL/项目配置覆盖，以及该项目是否采用 World Partition 或全动态光照。

## 3. World Partition 的强制规则

World Partition（WP）不是普通开关，而是会改变 World Settings 可用性的世界架构。`SetWorldPartition` 和地图加载时都会调用 `ApplyWorldPartitionForcedSettings()`：

- 强制关闭 **Enable World Composition**；两种大世界流送方案不能共用。
- 强制关闭 **Precompute Visibility**。
- 默认强制开启 **Force No Precomputed Lighting**。只有控制台变量 `r.AllowStaticLightingInWorldPartitionMaps` 非零时，才保留静态光照选项。
- 编辑器中 `LightmassSettings.bWorldPartition` 会设为 `true`，从而切换体积光照贴图的相关字段。

所以 WP 地图应把重点放在 Runtime Grid、Data Layer、HLOD 和动态光照方案上；不要依赖传统 World Composition、预计算可见性或 Lightmass 烘焙作为默认工作流。

## 4. World 与 Physics

| 设置 | 含义与建议 |
| --- | --- |
| **Enable World Bounds Checks** | 启用 Actor 的 `CheckStillInWorld` 检查，包括超出世界边界与低于 Kill Z 的处理。默认开启。 |
| **Kill Z** | Actor 跌落到该 Z 值以下时被销毁/受到伤害的阈值。默认是旧世界半尺寸的负值；通常应针对游戏玩法设为明确的失败高度。 |
| **Kill Z Damage Type** | 触发 Kill Z 时使用的伤害类型；默认环境伤害类型。可用自定义伤害类型驱动死亡、UI 或统计逻辑。 |
| **Override World Gravity** | 启用后使用本地图的 **Global Gravity Z**，否则使用 `UPhysicsSettings::DefaultGravityZ`。 |
| **Global Gravity Z** | 地图重力覆盖值。`GetGravityZ()` 首次访问时缓存 `GlobalGravityZ` 或项目默认重力；网络接收重力复制后会标记该缓存有效。 |
| **Default Physics Volume Class** | 本世界默认物理体积类，决定没有显式 Physics Volume 覆盖处的基础物理行为。 |
| **Physics Collision Handler Class** | 可选的碰撞处理器类，用于统一处理物理碰撞事件。 |
| **World to Meters** | VR 的真实单位映射。默认 `100`，即 $100\ \mathrm{uu}=1\ \mathrm{m}$，符合 UE 默认 $1\ \mathrm{uu}=1\ \mathrm{cm}$。改变它会影响 HMD 的感知比例，不会把现有场景几何自动缩放。 |
| **Broadphase Settings** | 可覆盖项目的物理 Broadphase 配置。MBP 边界必须覆盖需要碰撞的所有区域；漏出边界的 Actor 会失去碰撞。仅在确有大规模物理世界需求时修改。 |

## 5. 时间、网络与加载

### Time Dilation

`TimeDilation`、`CinematicTimeDilation` 和 `DemoPlayTimeDilation` 相乘：

$$
\mathrm{EffectiveTimeDilation}=\mathrm{TimeDilation}\times\mathrm{CinematicTimeDilation}\times\mathrm{DemoPlayTimeDilation}
$$

`SetTimeDilation()` 会把全局倍率夹到 `MinGlobalTimeDilation` 和 `MaxGlobalTimeDilation` 之间。Sequencer 使用 `CinematicTimeDilation`，回放速度使用 `DemoPlayTimeDilation`；游戏代码一般应读取 `GetEffectiveTimeDilation()`，而不是只读取 `TimeDilation`。

`MinUndilatedFrameTime` 与 `MaxUndilatedFrameTime` 定义未膨胀帧时间范围。`FixupDeltaSeconds()` 会先按有效时间膨胀缩放这两个边界，再夹取最终 Delta Seconds。将最大帧时间设得过小会使卡顿后的模拟时间被截断。

### 网络与加载

- **Use Client Side Level Streaming Volumes**：服务器保持流送关卡加载，由各客户端依据 Streaming Volume 独立加载/卸载。适用于客户端视点不同且服务器需要完整世界的场景。
- **High Priority Loading**：复制到客户端，为后台加载分配更多时间。`bHighPriorityLoadingLocal` 是仅本地的对应状态。
- **Reuse Address and Port**：监听 Socket 允许复用地址和端口。若两个服务器实际监听相同端口，数据报可能被分发到两者，除非完全掌握部署环境，否则保持关闭。

## 6. GameMode、AI 与导航

| 设置 | 说明 |
| --- | --- |
| **GameMode Override** | 此地图启动时的默认 GameMode。为空时回退到项目/INI 的默认游戏类型；运行参数和服务器配置仍可能覆盖它。 |
| **Enable AI System** | 控制是否创建 AI 系统；关闭后本地图不创建 AI 系统。编辑器中修改该项或 `AISystemClass` 会要求世界重新创建 AI 系统。 |
| **AI System Class** | AI 系统实现类，通常使用项目默认值。 |
| **Navigation System Config** | 导航系统的实例化配置；其 `NavigationSystemClass` 无效或配置为空时，导航系统不会创建。旧的 `bEnableNavigationSystem` 仅用于兼容旧地图。 |
| **Base Navmesh Data Layers** | 要并入基础 Navmesh 的运行时 Data Layer。编辑器 Data Layer 和不属于任何 Data Layer 的 Actor 也会被纳入。 |
| **Navigation Data Chunk Grid Size / Builder Loading Cell Size** | WP 中导航数据切块及迭代构建的加载尺度。实际加载单元会按 Chunk Grid Size 对齐；更大通常更快，但要求更多内存。 |

## 7. 渲染、光照与可见性

### 预计算可见性

- **Precompute Visibility**：在 Precomputed Visibility Volume 和摄像机轨迹附近生成可见性单元，降低运行时渲染线程开销，代价是内存与烘焙时间。
- **Visibility Cell Size**：默认 `200` uu。值越小，遮挡剔除可能越有效，但单元数、内存和烘焙时间都会增加。
- **Visibility Aggressiveness**：越激进，剔除越多，也越容易出现物体突然显现（popping）。
- **Place Cells Only Along Camera Tracks**：只沿轨迹或投射阴影的表面布置单元。

这些选项不适用于 WP 世界。

### Lightmass 与体积光照

**Force No Precomputed Lighting** 会禁止生成 Lightmap 等预计算光照数据，适合完全动态光照迭代；代价是原本依赖烘焙的光照和阴影效果不再存在。**Force Volumetric Lightmaps Only** 则限制预计算光照仅使用体积光照贴图。

`LightmassSettings` 仅在带编辑器数据的构建中存在：

| 组别 | 核心设置 | 使用要点 |
| --- | --- | --- |
| General | Static Lighting Level Scale、Indirect Lighting Quality、Indirect Lighting Smoothness、间接光/天光反弹次数 | Level Scale 小于 1 或 Quality 大于 1 会显著增加烘焙时间。反弹数主要影响质量；天光反弹的成本随次数增加。 |
| General | Environment Color / Intensity、Diffuse Boost、Compress Lightmaps | 环境颜色不会作为间接光反弹，且会影响反射捕获亮度，优先使用 Static Skylight。关闭压缩可减轻伪影，但内存与磁盘约增至 4 倍。 |
| Volume Lighting | Volume Lighting Method | **Volumetric Lightmap** 在 GPU 按像素插值，支持动态物体与 Volumetric Fog；**Sparse Volume Lighting Samples** 依赖 CPU 的 Indirect Lighting Cache，增加渲染线程开销且不能给 Volumetric Fog 提供预计算光照。 |
| Volume Lighting | Detail Cell Size、Maximum Brick Memory、Spherical Harmonic Smoothing | Detail Cell Size 减半可能使内存最多增至 8 倍。内存上限不足时，远离几何体的高密度 Brick 会先被丢弃。Smoothing 用于减轻球谐振铃造成的反侧黑斑，但越高方向性越弱。 |
| Occlusion | Use Ambient Occlusion、直接/间接遮蔽比例、Exponent、Max Distance | 只有启用 AO 后，其余 AO 控件才可编辑。若只需要 `PrecomputedAOMask` 材质节点，应生成 AO Mask，并将直接和间接遮蔽比例都设为 0。 |

非 WP 世界的 VLM 使用 **Volumetric Lightmap Maximum Brick Memory Mb**；WP 世界改用 **Volumetric Lightmap Loading Cell Size** 与 **Volumetric Lightmap Loading Range** 控制高细节数据按流送单元加载的尺度和范围。

### 其他渲染项

- **Packed Light And Shadow Map Texture Size**：烘焙 Light/Shadow Map 图集最大尺寸。编辑器会将其修正为 $512$ 到 $4096$ 间的 2 的幂，默认 `1024`。
- **Default Color Scale**：关卡默认颜色缩放。
- **Default Max DistanceField Occlusion Distance**：Mesh Distance Field 的最大遮蔽距离；存在可移动 Skylight 时可被其设置覆盖。
- **Global DistanceField View Distance**：相机周围全局距离场覆盖距离。默认 `20000` uu；增大可覆盖更广，但提高资源开销。
- **Dynamic Indirect Shadows Self Shadowing Intensity**：胶囊间接阴影的自阴影强度。降低可掩盖近似遮挡体带来的伪影。

## 8. World Partition、HLOD 与编辑器网格

| 设置 | 说明 |
| --- | --- |
| **HLOD Setup Asset** | 地图指定的 HLOD 设置资产，优先于地图内嵌设置。若项目设置强制所有地图使用默认 HLOD Setup，则项目资产会成为回退来源。 |
| **Override Base Material** | 覆盖项目级 HLOD Proxy Material；若 HLOD Setup Asset 自己也指定了材质，资产内的覆盖优先。 |
| **Generate Single Cluster For Level** | 为小型子关卡将符合条件的 Actor 合成一个整关卡 Cluster。 |
| **HLOD Baking Transform** | 构建 HLOD 时施加到源网格的变换。 |
| **Instanced Foliage Grid Size** | 分区世界植被 Actor 的网格大小；当前编辑器逻辑不允许直接修改。 |
| **Landscape Spline Meshes Grid Size** | 分区世界 Landscape Spline Mesh 的分区网格大小，仅 WP 世界可编辑。 |
| **Default Placement Grid Size** | 编辑器放置元素的默认网格大小。 |

## 9. Nanite、Audio 与书签

- **Nanite > Allow Masked Materials**：控制 Nanite 是否允许 Masked Material。服务器调用 `SetAllowMaskedMaterials()` 后会以 Push Model 复制；客户端收到更新会重建组件渲染状态，确保 Scene Proxy 刷新。
- **Default Reverb Settings / Default Ambient Zone Settings**：默认应用于世界音频设备的混响与环境区域设置。
- **Default Base Sound Mix**：世界默认基础 Sound Mix。
- **Bookmarks**：World Settings 存储关卡书签。`MaxNumberOfBookmarks` 默认 10，`DefaultBookmarkClass` 决定创建时的书签类型；书签数组允许空位。其主要作用是编辑器工作流，不是玩法存档系统。

## 10. 推荐工作流

1. 新建地图先确定是否使用 World Partition；这会决定流送、HLOD 和静态光照的可用方案。
2. 在项目设置中配置可复用默认值，在 World Settings 中只保留地图确实需要的覆盖，尤其是 GameMode、重力、Kill Z 和 World to Meters。
3. 使用静态光照前先规划 Lightmass Importance Volume、Lightmap UV、烘焙预算和目标平台；不要仅靠提高 Indirect Lighting Quality 修复 UV 或压缩伪影。
4. 需要慢动作时通过 `SetTimeDilation()` 修改全局倍率，Sequencer 使用独立的 Cinematic 倍率；同时核对最小/最大帧时间是否会截断模拟。
5. 改动导航、AI、World Partition 或 HLOD 后，在目标地图及目标网络模式下验证，因为它们都依赖关卡加载、流送与运行时初始化顺序。
# UE5 World Settings 解析

`World Settings` 是每张地图的全局配置入口，对应 `AWorldSettings`。它是持久关卡中的特殊 `AInfo` Actor：不可在场景中放置、始终相关（`bAlwaysRelevant`），其部分状态会从服务器复制到客户端。编辑器中可通过 **Settings > World Settings** 打开。

本页基于 `Engine/Classes/GameFramework/WorldSettings.h` 和 `Engine/Private/WorldSettings.cpp` 编写，属性是否显示还会受当前地图是否使用 World Partition、是否启用静态光照等条件影响。

## 使用边界

- 设置保存在当前 `.umap`，不是项目全局默认值；带 `Config` 的字段还可以由游戏配置参与初始化或覆盖。
- `AWorldSettings` 只应由持久关卡持有。子关卡关联后使用持久关卡的世界设置。
- 不要把这里的设置和 Project Settings 混淆：例如重力可在本地图覆盖项目 Physics 的 `DefaultGravityZ`，GameMode Override 可覆盖项目默认 GameMode。
- 面向运行时修改时，优先通过引擎 API，而不是直接改成员：时间倍率使用 `SetTimeDilation`，Nanite 遮罩材质使用 `SetAllowMaskedMaterials`。

## World、Physics 与 Navigation

### World

| 设置 | 作用与建议 |
| --- | --- |
| Enable World Bounds Checks | 启用 Actor 的 `CheckStillInWorld` 边界检查。关闭后不会因为超出世界边界或低于 Kill Z 而由该检查销毁。 |
| Kill Z / Kill Z Damage Type | Actor 低于 `KillZ` 时的销毁高度和伤害类型。默认 `KillZ` 是 `-UE_OLD_HALF_WORLD_MAX1`，伤害类型为环境伤害。 |
| Enable World Composition | 旧版分块世界工作流。World Partition 地图中不可用；新项目应使用 World Partition。 |
| Enable World Origin Rebasing | 随相机远离原点而平移世界原点，主要服务旧版大世界工作流；它在编辑器中依赖 World Composition。 |
| Use Client Side Level Streaming Volumes | 使客户端按 Streaming Volume 自行流送；常见模式是服务器常驻全部关卡，客户端独立加载。 |
| Default Color Scale | 本关卡默认颜色缩放。 |
| Default Physics Volume Class | 此地图默认物理体积使用的类。 |

### Physics 与 VR

| 设置 | 作用与建议 |
| --- | --- |
| Override World Gravity / Global Gravity Z | 勾选覆盖后，`GetGravityZ()` 使用本地图的 `GlobalGravityZ`；否则使用项目 `UPhysicsSettings::DefaultGravityZ`。首次查询时缓存到 `WorldGravityZ`，该值复制到客户端。 |
| Physics Collision Handler Class | 为该地图指定碰撞处理器类。 |
| World to Meters | VR 和物理追踪设备的现实尺度，默认 `100`，即 $100\ \mathrm{uu}=1\ \mathrm{m}$（1 uu = 1 cm）。调整它会改变 HMD 世界比例感。 |
| Broadphase Settings | 选择客户端/服务器是否使用 MBP（Multi Broadphase Pruning），并设置覆盖范围与分区数。`MBPBounds` 必须覆盖需要碰撞的游戏区域，否则范围外 Actor 的碰撞会被禁用。默认设置仅在勾选 Override Default Broadphase Settings 后生效。 |

### Navigation 与 AI

- `Navigation System Config` 为实例化配置；为空或其中的导航系统类无效时，不创建导航系统。
- `AI System` 控制本地图是否创建 AI 系统，`AI System Class` 可替换实现。编辑器修改这两项会要求世界重新创建 AI 系统。
- World Partition 地图还提供 `Base Navmesh Data Layers`：列出的运行时 Data Layer 会被纳入基础 Navmesh；编辑器 Data Layer 和未归属 Data Layer 的 Actor 同样会纳入。
- `Navigation Data Chunk Grid Size`、`Navigation Data Builder Loading Cell Size` 用于分区世界的导航数据构建。加载单元大小会按导航 Chunk 网格尺寸取整，较大值会增加构建时内存需求。

## GameMode、Network 与 Tick

### GameMode 和网络

- `GameMode Override` 是此地图的默认 `AGameModeBase`。未设置时回退到项目/INI 的默认游戏模式。
- `GameNetworkManager Class` 指定网络游戏生成的网络管理器类。
- `Reuse Address and Port` 让监听套接字复用地址与端口。仅在能够保证没有其他服务器监听相同端口时使用，否则数据包可能被两个服务器接收。
- `High Priority Loading` 会复制到客户端，提高后台加载优先级；`High Priority Loading Local` 仅用于本地客户端加载。

### 时间控制

有效时间倍率不是单独的 `Time Dilation`，而是：

$$
	ext{EffectiveTimeDilation} = \text{TimeDilation} \times \text{CinematicTimeDilation} \times \text{DemoPlayTimeDilation}
$$

- `SetTimeDilation(NewValue)` 会把全局倍率钳制到 `MinGlobalTimeDilation` 和 `MaxGlobalTimeDilation` 之间；三个倍率的默认值均为 `1`。
- `Cinematic Time Dilation` 由 Sequencer 慢动作使用，`Demo Play Time Dilation` 用于回放速度。
- `Min/Max Undilated Frame Time` 约束未考虑倍率的帧时间；运行时的 `FixupDeltaSeconds` 会先按有效倍率缩放上下界，再钳制 `DeltaSeconds`。它们分别等价于最快和最慢 FPS 的倒数。
- `TimeDilation` 与 `CinematicTimeDilation` 会复制；`DemoPlayTimeDilation` 不复制。

## Rendering、Nanite 与 Audio

### Rendering

| 设置 | 作用与建议 |
| --- | --- |
| Default Max DistanceField Occlusion Distance | Mesh Distance Field 的默认最大遮蔽距离；可被可移动 Sky Light 覆盖。默认 `600`。 |
| Global DistanceField View Distance | 全局距离场覆盖相机周围的距离。默认 `20000`；增大可覆盖更远区域，但会提高资源开销。 |
| Dynamic Indirect Shadows Self Shadowing Intensity | Capsule Indirect Shadow 自阴影强度。降低可掩盖近似遮挡体导致的瑕疵；默认 `0.8`。 |
| Packed Light And Shadow Map Texture Size | 打包光照/阴影贴图最大尺寸。编辑后引擎会强制调整到 $512$ 至 $4096$ 的 2 的幂。 |
| Precomputed Visibility | 生成预计算可见性单元，减少渲染线程工作量，但增加运行时内存和光照构建时间。`Visibility Cell Size` 越小，剔除更有效但构建和内存成本越高；Aggressiveness 越高，剔除越积极也越可能产生 pop。 |

### Nanite

`Nanite Settings > Allow Masked Materials` 决定 Nanite 是否允许遮罩材质。该结构以 Push Model 方式复制；服务端调用 `SetAllowMaskedMaterials` 后会标记属性脏，并在客户端重建组件渲染状态。因此不要只在客户端直接修改它。

### Audio

- `Default Reverb Settings`：世界默认混响，供 Audio Volume 使用。
- `Default Ambient Zone Settings`：应用 Ambient Volumes 的 SoundClass 的默认内部/外部区域参数。
- `Default Base Sound Mix`：世界默认基础 SoundMix。

组件注册完成后，世界会把前两项交给当前音频设备作为默认音频设置。

## Lightmass 与 Volumetric Lightmap

此部分主要针对静态/预计算光照。全动态光照项目通常应启用 `Force No Precomputed Lighting`，以缩短迭代时间；代价是通常依赖预计算的光照和阴影交互会消失。

### 通用 Lightmass

| 设置 | 含义 |
| --- | --- |
| Static Lighting Level Scale | 场景相对真实尺度。默认 `1`（1 uu = 1 cm）；低于 1 会显著增加构建时间，大场景可用 2 或 4 降低成本。 |
| Num Indirect Lighting Bounces | 点光、聚光和方向光的间接反弹次数。第 1 次最有影响也最耗时，后续收益递减。 |
| Num Sky Lighting Bounces | 天光与自发光反弹次数；其成本随次数增加而增加。 |
| Indirect Lighting Quality / Smoothness | 前者提高 GI 求解采样、显著增加构建时间；后者控制间接光平滑度，增大将减少细节。提高 Quality 时通常可略降 Smoothness。 |
| Environment Color / Intensity | 上半球常量环境光。它不参与间接反弹且会使反射捕获亮度不准确，优先使用 Static Skylight。Intensity 为 0 时颜色不可编辑。 |
| Diffuse Boost | 缩放全场景材质的漫反射贡献。 |
| Compress Lightmaps | 压缩光照贴图。关闭可减少压缩伪影，但内存与磁盘空间约增加 4 倍。 |

### AO

启用 `Use Ambient Occlusion` 后，才可编辑 AO 材质遮罩、直接/间接遮蔽比例、指数、完全遮蔽样本比例、最大遮蔽距离及 AO 可视化。若只需 `PrecomputedAOMask` 材质节点，应生成 AO 遮罩，并将直接和间接遮蔽比例设为 0。

### 体积光照

`Volume Lighting Method` 有两种模式：

- **Volumetric Lightmap**：默认方案。在 Lightmass Importance Volume 中计算自适应体素网格，GPU 可逐像素插值，适合可移动物体和 Volumetric Fog；Importance Volume 外复用边界体素。`Detail Cell Size` 减半，内存最坏可增加约 8 倍。
- **Sparse Volume Lighting Samples**：在静态表面及重要体积内放置样本，经 Indirect Lighting Cache 在 CPU 上插值；会增加渲染线程开销，且 Volumetric Fog 不受其预计算光照影响。此模式使用 `Volume Light Sample Placement Scale` 控制采样密度。

在 Volumetric Lightmap 模式下：

- 非 World Partition 地图使用 `Volumetric Lightmap Maximum Brick Memory Mb` 限制 Brick 内存，超限时优先丢弃离几何体最远的高密度 Brick。
- World Partition 地图改用 `Volumetric Lightmap Loading Cell Size` 与 `Volumetric Lightmap Loading Range` 控制高精度体积光照的流送单元和加载范围。
- `Spherical Harmonic Smoothing` 只在出现 SH 振铃造成的反向黑斑时生效；0 为不平滑，1 为强平滑并牺牲方向性。

## World Partition、HLOD 与编辑器网格

World Partition 已设置到地图时，`ApplyWorldPartitionForcedSettings()` 会执行以下规则：

1. 强制关闭 `Enable World Composition` 和 `Precompute Visibility`。
2. 当控制台变量 `r.AllowStaticLightingInWorldPartitionMaps` 为 0（默认路径）时，强制启用 `Force No Precomputed Lighting`。
3. 编辑器中的 Lightmass 设置记录该地图为 World Partition，以切换体积光照相关字段。

因此，在 World Partition 地图中不要试图依赖旧 World Composition、预计算可见性，或在默认 CVar 配置下依赖静态光照。地图载入时这些限制还会再次应用。

`HLOD Setup Asset` 用于指定该地图的 HLOD 配置资产；未指定时可使用地图内 `Hierarchical LOD Setup`，或被项目 `HierarchicalLODSettings` 的强制默认配置接管。HLOD 基础材质的优先级为：HLOD Setup Asset 的覆盖材质、地图的 `Override Base Material`、项目默认基础材质。

World Partition 编辑器专用网格参数包括 Instanced Foliage、Landscape Spline Mesh 和导航数据块。Foliage 网格尺寸由分区系统管理，不能在该面板修改；勾选 `Show Instanced Foliage Grid` 仅显示预览网格，不影响运行时数据。

## 常见决策

- **开放世界新地图**：使用 World Partition；接受其关闭旧 World Composition 和预计算可见性的限制；按项目渲染策略决定是否开启 `r.AllowStaticLightingInWorldPartitionMaps`。
- **全动态 Lumen/Nanite 场景**：启用 `Force No Precomputed Lighting`，重点调整距离场、Nanite 和流送设置，不必为 Lightmass 构建成本买单。
- **烘焙光照场景**：保持预计算光照开启，先使用默认 Lightmass 参数；仅在有明确噪点、泄漏或细节需求时小步提高 Quality 或减小静态光照尺度，并监控构建时间与体积光照内存。
- **网络慢动作**：由服务器调用 `SetTimeDilation`；不要只改客户端值。Sequencer 的 Cinematic 倍率会继续与全局倍率相乘。
- **VR 尺度异常**：先检查 `WorldToMeters` 是否仍为 `100`，再检查内容是否遵循 1 uu = 1 cm 的单位约定。
