下面按**自底向上**来梳理虚幻引擎（UE）的动画系统，并结合你当前源码里的实际模块来对应说明。

## 检查清单
- [x] 先从最底层“数据与骨架”讲起
- [x] 再往上讲“播放/评估运行时”
- [x] 再讲“图、状态机、蓝图”
- [x] 补充“扩展层”：Control Rig、Motion Matching、后处理、变形等
- [x] 给出源码模块与典型类的对应关系

---

## 一句话总览

UE 的动画系统可以理解成一条自底向上的流水线：

**骨架/动画资源 → Pose 计算与解压 → 动画节点图求值 → AnimInstance 驱动 → SkeletalMeshComponent 应用到模型 → 更高层扩展（Montage / StateMachine / ControlRig / Motion Matching / Deformer / Sequencer）**

---

# 1. 最底层：骨架、蒙皮、基础动画数据

这一层解决的是：**“动画作用在谁身上，数据长什么样”**。

## 1.1 Skeletal Mesh / Skeleton
最底层的动画对象不是蓝图，而是**骨骼模型和骨架**。

- `USkeletalMeshComponent`：真正负责把动画结果应用到角色组件上  
  文件：`Engine/Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h`
- `USkeleton`：定义骨骼层级、曲线、插槽、通知等共享信息
- `USkeletalMesh`：网格 + 绑定骨架 + 蒙皮数据

从 `SkeletalMeshComponent.h` 里也能看到，它内部维护了动画评估时要输出的：
- `ComponentSpaceTransforms`
- `BoneSpaceTransforms`
- `Curve`
- `CustomAttributes`

见 `FAnimationEvaluationContext`：  
`Engine/Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h`

这说明对 UE 来说，动画系统的核心输出不只是“骨骼变换”，还包括：
- 曲线（Curve）
- 自定义属性（Attributes）
- 根运动（Root Motion，虽然不在这个片段里直接展开）

---

## 1.2 AnimationAsset：所有动画资源的基类
动画资源的统一抽象是 `UAnimationAsset`。

源码注释写得很直白：

> “Abstract base class of animation assets that can be played back and evaluated to produce a pose.”

文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimationAsset.h`

也就是说，UE 最底层对动画资源的定义就是：  
**“一个可被播放和评估，并产出 pose 的资源”**。

---

## 1.3 具体动画资源类型
在 `UAnimationAsset` 之上，最常见资源包括：

### `UAnimSequence`
最基础的关键帧动画序列。  
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimSequence.h`

源码前面写得很明确：

> “One animation sequence of keyframes. Contains a number of tracks of data.”

这里可以看到它持有：
- 位移轨道 `FTranslationTrack`
- 旋转轨道 `FRotationTrack`
- 缩放轨道 `FScaleTrack`

这层就是“原始动画数据层”。

---

### `UBlendSpace`
用于根据输入参数在多个动画之间插值。  
文件：`Engine/Source/Runtime/Engine/Classes/Animation/BlendSpace.h`

它本质上还是动画资源，只不过不是单条序列，而是一个**参数化采样器**。

---

### `UAnimMontage`
用于更高层的片段组织、分段、插槽播放、通知和事件控制。  
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimMontage.h`

从源码能直接看到它包含：
- `FCompositeSection`：分段
- `FSlotAnimationTrack`：插槽轨道
- `FBranchingPoint`：分支事件点

所以 Montage 可以理解为：
**“面向游戏逻辑控制的动画编排资源”**，而不只是纯粹的姿势数据。

---

# 2. 低层运行时：Pose、解压、评估上下文

这一层解决的是：**“如何把动画资源变成当前帧的姿势”**。

## 2.1 Pose 是运行时核心中间结果
动画系统的运行时核心并不是直接“播动画”，而是不断构造和修改**Pose**。

在源码里你会看到很多与 Pose 相关的结构：
- `FCompactPose`
- `FCSPose`
- `FA2Pose`
- `FA2CSPose`

例如在 `AnimInstance.h` 中就有 `FA2Pose`、`FA2CSPose`：  
`Engine/Source/Runtime/Engine/Classes/Animation/AnimInstance.h`

而在 `AnimNodeBase.h` 中，节点评估会围绕：
- `FPoseContext`
- `FComponentSpacePoseContext`
- `FAnimationPoseData`

这些类型运转。  
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimNodeBase.h`

你可以把这一层理解为：
- **资源层**提供动画数据
- **Pose 层**负责把数据变成“当前帧骨骼姿态”

---

## 2.2 动画解压 / 压缩
`UAnimSequence` 不只是存关键帧，它还有压缩与解压体系：
- `AnimBoneCompression*`
- `AnimCurveCompression*`
- `FAnimSequenceDecompressionContext`

相关文件分布在：
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimSequence.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimBoneCompression*.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimCurveCompression*.h`

这属于动画系统非常底层但非常关键的一块：  
**磁盘里是压缩过的数据，运行时要高效解压并采样。**

---

## 2.3 曲线、通知、Marker、属性
在 `AnimationAsset.h` 里能看到 Marker 相关结构，比如：
- `FMarkerPair`
- `FMarkerTickRecord`
- `FDeltaTimeRecord`

文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimationAsset.h`

这些用于：
- 同步组（Sync Group）
- Marker 同步
- 时间推进
- 事件触发边界控制

此外动画系统还会处理：
- 曲线（驱动 Morph、材质参数、游戏参数）
- Notify / NotifyState
- 自定义 Attribute

所以底层运行时并不只是“算骨骼”，而是**同时计算姿势、曲线、属性和事件时序**。

---

# 3. 节点层：Anim Node / Anim Graph Runtime

这一层解决的是：**“如何组合多个动画来源，并逐层加工 Pose”**。

这是 UE 动画系统非常核心的一层。

## 3.1 `FAnimNode_Base`：所有动画节点的基类
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimNodeBase.h`

几乎所有动画图节点的运行时结构都继承自 `FAnimNode_Base`。  
它定义了动画节点的基本生命周期，例如：
- 初始化
- CacheBones
- Update
- Evaluate

因此你可以把它看成是动画图运行时的“虚函数接口层”。

---

## 3.2 Asset Player 节点
最基础的一类节点是**资源播放节点**，例如：
- Sequence Player
- BlendSpace Player
- Montage/Slot 相关节点
- Motion Matching 这类更高级播放器节点也会继承这一路体系

例如 `FAnimNode_MotionMatching` 继承自：
- `FAnimNode_BlendStack_Standalone`
- 上层属于资产播放器体系

文件：  
`Engine/Plugins/Animation/PoseSearch/Source/Runtime/Public/PoseSearch/AnimNode_MotionMatching.h`

---

## 3.3 Blend 节点
这类节点负责将多个 Pose 混合：
- 按权重 Blend
- 按骨骼 Layered Blend
- Additive
- Inertialization
- Dead Blending

相关基础节点文件能在：
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_Inertialization.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_DeadBlending.h`
- 以及其他 `AnimNode_*`

---

## 3.4 Skeletal Control 节点
这一层是“对骨骼进行程序化修正”的节点，比如：
- IK
- LookAt
- FABRIK
- CCDIK
- Transform Bone
- Modify Bone

它们的基类是：

`FAnimNode_SkeletalControlBase`  
文件：`Engine/Source/Runtime/AnimGraphRuntime/Public/BoneControllers/AnimNode_SkeletalControlBase.h`

源码注释写得很清楚：

> “A SkelControl is a module that can modify the position or orientation of a set of bones in a skeletal mesh in some programmatic way.”

所以这一层的职责是：  
**在已有 Pose 基础上，对骨骼做程序化修正。**

---

## 3.5 Linked Graph / Layer / Cached Pose
为了模块化动画图，UE 还提供：
- `AnimNode_LinkedAnimGraph`
- `AnimNode_LinkedAnimLayer`
- `AnimNode_SaveCachedPose`

相关文件：
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedAnimGraph.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedAnimLayer.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_SaveCachedPose.h`

这属于**图级别复用和模块化**能力。

---

# 4. 实例层：`UAnimInstance`

这一层解决的是：**“谁来驱动动画图，并与角色逻辑交互”**。

## 4.1 `UAnimInstance` 是动画运行时实例
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimInstance.h`

它是挂在 `USkeletalMeshComponent` 上的动画实例对象，负责：
- 驱动动画蓝图
- 管理状态机
- 管理 Montage
- 同步组
- Notify
- 根运动
- 变量暴露给图节点

可以把它理解成：

**动画图的运行时宿主（runtime host）**

---

## 4.2 `USkeletalMeshComponent` + `UAnimInstance`
这两个类是动画系统运行时最关键的配对：

- `USkeletalMeshComponent`：承载模型、组件更新、最终骨骼结果
- `UAnimInstance`：生成本帧动画 Pose

大体关系是：

1. `SkeletalMeshComponent` 每帧触发动画更新
2. `AnimInstance` 更新变量与状态
3. 动画图节点树 Update / Evaluate
4. 得到最终 Pose / Curve / Attribute
5. `SkeletalMeshComponent` 应用到渲染和物理

---

# 5. 蓝图编译层：Anim Blueprint / Generated Class

这一层解决的是：**“编辑器里的动画蓝图，如何变成运行时可执行结构”**。

## 5.1 `UAnimBlueprint`
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimBlueprint.h`

源码注释：

> “An Anim Blueprint is essentially a specialized Blueprint whose graphs control the animation of a Skeletal Mesh.”

它是编辑器资产层的表示，负责：
- 指定 `TargetSkeleton`
- 包含动画图、状态机图
- 配置多线程更新等选项

---

## 5.2 `UAnimBlueprintGeneratedClass`
文件：`Engine/Source/Runtime/Engine/Classes/Animation/AnimBlueprintGeneratedClass.h`

这是动画蓝图编译后的结果。  
它会保存：
- 编译后的节点属性布局
- 状态机数据
- 调试信息
- 图到运行时结构的映射

所以从架构上看：

- `UAnimBlueprint` = 编辑器资产
- `UAnimBlueprintGeneratedClass` = 编译产物
- `UAnimInstance` = 运行时实例

这是 UE 动画蓝图体系的三层关系。

---

# 6. 状态机与高层流程控制

这一层解决的是：**“动画怎么切换、怎么组织逻辑流程”**。

虽然你这次问的是“系统模块”，但状态机是上层最常见组织方式。

在 `AnimBlueprintGeneratedClass.h` 里可以看到很多状态机调试结构，例如：
- `FStateMachineDebugData`
- `FStateMachineStateDebugData`

这反映出状态机是 Anim Blueprint 编译系统的重要组成部分。

这层常见模块包括：
- State Machine
- Transition Rule
- Conduit
- Sync Group
- Slot
- Montage

它们本质上不是更底层的数据结构，而是**动画逻辑组织层**。

---

# 7. 蒙太奇与插槽系统（Montage / Slot）

这是介于“图”和“游戏逻辑”之间的一层。

## 7.1 Montage
`UAnimMontage` 主要服务于：
- 攻击动作
- 技能动作
- 受击
- 分段跳转
- 事件控制
- 上半身覆盖播放

## 7.2 Slot
Slot 的作用是把 Montage 注入到动画图中的某个位置。  
比如：
- Locomotion 是基础层
- 上半身 Slot 播放攻击动作
- 最终和基础层混合

这层常用于游戏玩法驱动动画。

---

# 8. 程序化动画扩展层

这一层是 UE 现代动画系统很重要的增强部分。

## 8.1 Control Rig
Control Rig 让你在动画系统里引入更强的程序化 Rig 求解。

运行时节点：
- `FAnimNode_ControlRig`  
文件：`Engine/Plugins/Animation/ControlRig/Source/ControlRig/Public/AnimNode_ControlRig.h`

源码说明：

> “Animation node that allows animation ControlRig output to be used in an animation graph”

说明它是以**AnimGraph 节点**的方式接入动画流水线的。

所以从分层看，Control Rig 不是替代动画系统，而是插在**AnimGraph 运行时节点层**之上的高级模块。

---

## 8.2 Skeletal Controls / IK
虽然 IK 也可由 Control Rig 做，但传统动画图里也有独立骨骼控制体系，即前面说的：
- `FAnimNode_SkeletalControlBase`

这是更经典的程序化骨骼修正模块。

---

# 9. 数据驱动高级动画：Motion Matching / Pose Search

这是 UE5 动画系统里一个更“现代”的上层模块。

## 9.1 Motion Matching 节点
文件：  
`Engine/Plugins/Animation/PoseSearch/Source/Runtime/Public/PoseSearch/AnimNode_MotionMatching.h`

`FAnimNode_MotionMatching` 本质上是一个特殊的资产播放器节点，但它播放的不是“固定动画”，而是：
- 在 Pose 数据库中搜索
- 根据当前运动意图选最优姿势
- 再平滑过渡过去

所以它位于分层中的位置是：

**动画图节点层之上的高级数据驱动播放模块**

---

## 9.2 Pose Search 模块
和 Motion Matching 配套的通常有：
- PoseSearch Database
- 查询特征构建
- 姿势评分
- 历史轨迹/未来轨迹特征

这已经属于“动画决策系统”，比普通 Sequence Player 更高一层。

---

# 10. 后处理与附加动画层

这一层解决的是：**“主动画求值之后，还要不要再做一遍处理”**。

典型包括：
- Post Process Anim Blueprint
- Additive 修正
- 面部动画叠加
- 物理骨骼修正
- Cloth / Secondary Motion 协同

在 `SkeletalMeshComponent` 的评估上下文里你能看到：
- `AnimInstance`
- `PostProcessAnimInstance`

见 `FAnimationEvaluationContext`：  
`Engine/Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h`

这说明 UE 在主动画实例之外，还支持**后处理动画实例**。

---

# 11. 变形与渲染耦合层

这层已经接近最终显示了。

它包括：
- Morph Target 曲线驱动
- ML Deformer / Deformer Graph（若启用相关插件）
- Cloth
- Skinning
- 骨骼矩阵上传给渲染线程

这部分严格说已经不完全属于“动画图系统”，但属于**动画结果落地到最终角色表现**的重要层级。

---

# 12. 编辑器与开发模块

如果你从源码模块角度看，UE 动画系统不是一个单体，而是分散在多个模块里。

## 12.1 核心运行时模块
最关键的运行时模块通常有：

### `Engine`
包含基础动画类型与核心类：
- `UAnimationAsset`
- `UAnimSequence`
- `UAnimInstance`
- `UAnimBlueprint`
- `UAnimBlueprintGeneratedClass`
- `USkeletalMeshComponent`

主要路径：
- `Engine/Source/Runtime/Engine/Classes/Animation/`
- `Engine/Source/Runtime/Engine/Classes/Components/`

---

### `AnimGraphRuntime`
包含运行时动画图节点，尤其是骨骼控制与图求值相关支持。  
比如：
- `FAnimNode_SkeletalControlBase`

路径：
- `Engine/Source/Runtime/AnimGraphRuntime/`

---

## 12.2 编辑器相关模块
常见有：
- `AnimGraph`
- `BlueprintGraph`
- `UnrealEd`
- 动画编辑器相关模块

这些负责：
- 动画蓝图节点编辑
- 状态机编辑器
- 调试可视化
- 资源导入与重定向

---

## 12.3 插件化高级动画模块
从你当前源码里能看到至少这些：

### `ControlRig`
- 运行时：`Engine/Plugins/Animation/ControlRig/Source/ControlRig/`
- 编辑器：`.../ControlRigEditor/`
- 开发支持：`.../ControlRigDeveloper/`

### `PoseSearch`
- Motion Matching / Pose Search  
路径：
`Engine/Plugins/Animation/PoseSearch/Source/Runtime/`

还有其他常见动画相关插件，比如：
- IK Rig
- IK Retargeter
- Deformer / ML Deformer
- Live Link
- Gameplay Cameras / Sequencer 相关联动模块

---

# 13. 如果按“自底向上模块分层”总结，可以这样记

我建议你把 UE 动画系统记成下面这 8 层：

## 第 1 层：骨架与网格层
- `USkeleton`
- `USkeletalMesh`
- `USkeletalMeshComponent`

## 第 2 层：动画资源层
- `UAnimationAsset`
- `UAnimSequence`
- `UBlendSpace`
- `UAnimComposite`
- `UAnimMontage`
- `UPoseAsset`

## 第 3 层：采样 / 压缩 / Pose 数据层
- 动画解压
- 曲线采样
- Marker / Sync
- `FCompactPose`
- `FAnimationPoseData`

## 第 4 层：AnimGraph 运行时节点层
- `FAnimNode_Base`
- Asset Player
- Blend
- Additive
- Cached Pose
- Linked Graph / Layer
- Inertialization

## 第 5 层：骨骼控制与程序化修正层
- `FAnimNode_SkeletalControlBase`
- IK / LookAt / FABRIK / CCDIK
- Transform Bone
- Modify Curve

## 第 6 层：实例驱动层
- `UAnimInstance`
- Montage 实例
- StateMachine 运行时
- Notify / Root Motion / Sync Group

## 第 7 层：蓝图与逻辑组织层
- `UAnimBlueprint`
- `UAnimBlueprintGeneratedClass`
- State Machine
- Transition
- Slot
- Montage

## 第 8 层：高级扩展层
- Control Rig
- Motion Matching / Pose Search
- Post Process Anim BP
- Deformer / ML Deformer
- Sequencer / Live Link / Runtime retargeting

---

# 14. 一个更直观的“执行路径”

以“角色跑步并开枪”为例，执行链路大致是：

1. `USkeletalMeshComponent` 触发动画更新  
2. `UAnimInstance` 更新速度、方向、是否开枪等变量  
3. AnimGraph 中 Locomotion 状态机选择跑步 BlendSpace  
4. Slot 节点叠加射击 Montage  
5. IK / Skeletal Control 修正手部或脚部  
6. 若接入 `ControlRig`，进一步做程序化调整  
7. 输出最终 Pose、Curve、Attributes  
8. `USkeletalMeshComponent` 应用到骨骼并驱动渲染/物理

---

# 15. 对应你当前源码中最值得先读的几个入口

如果你想从源码真正“自底向上”读，我建议顺序是：

1. `Engine/Source/Runtime/Engine/Classes/Animation/AnimationAsset.h`
2. `Engine/Source/Runtime/Engine/Classes/Animation/AnimSequence.h`
3. `Engine/Source/Runtime/Engine/Classes/Animation/AnimNodeBase.h`
4. `Engine/Source/Runtime/AnimGraphRuntime/Public/BoneControllers/AnimNode_SkeletalControlBase.h`
5. `Engine/Source/Runtime/Engine/Classes/Animation/AnimInstance.h`
6. `Engine/Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h`
7. `Engine/Source/Runtime/Engine/Classes/Animation/AnimBlueprint.h`
8. `Engine/Source/Runtime/Engine/Classes/Animation/AnimBlueprintGeneratedClass.h`
9. `Engine/Plugins/Animation/ControlRig/Source/ControlRig/Public/AnimNode_ControlRig.h`
10. `Engine/Plugins/Animation/PoseSearch/Source/Runtime/Public/PoseSearch/AnimNode_MotionMatching.h`

---

如果你愿意，我下一步可以继续给你整理一版：

1. **“UE 动画系统源码模块图（Runtime/Editor/Plugin）”**，或者  
2. **“从 `TickComponent` 到 AnimGraph Evaluate 的调用链”**。  

这两种都很适合深入理解源码。