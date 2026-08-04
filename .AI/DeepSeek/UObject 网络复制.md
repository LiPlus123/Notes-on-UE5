# UE5 UObject 网络复制：设计与实现深度分析

> 基于 Unreal Engine 5.5 源代码分析。涉及文件均位于 `Engine/Source/Runtime/Engine/` 目录下。

---

## 目录

1. [架构总览](#1-架构总览)
2. [核心数据流](#2-核心数据流)
3. [RepLayout：复制属性布局](#3-replayout复制属性布局)
4. [FObjectReplicator：对象复制器](#4-fobjectreplicator对象复制器)
5. [Changelist 机制：变更追踪](#5-changelist-机制变更追踪)
6. [信道层：UActorChannel 与 UChannel](#6-信道层uactorchannel-与-uchannel)
7. [网络驱动：UNetDriver](#7-网络驱动unetdriver)
8. [PackageMap 与 NetGUID 系统](#8-packagemap-与-netguid-系统)
9. [Actor 网络复制详解](#9-actor-网络复制详解)
10. [属性复制宏系统](#10-属性复制宏系统)
11. [复制条件系统](#11-复制条件系统)
12. [Push Model 属性推送](#12-push-model-属性推送)
13. [可靠性与重传机制](#13-可靠性与重传机制)
14. [ReplicationDriver 与 ReplicationGraph](#14-replicationdriver-与-replicationgraph)
15. [Iris：下一代复制系统](#15-iris下一代复制系统)
16. [总结与最佳实践](#16-总结与最佳实践)

---

## 1. 架构总览

UE5 的网络复制系统是一个多层架构，从上层游戏逻辑到下层网络传输，各层职责分明：

```
┌─────────────────────────────────────────────────────────────┐
│                    Game Layer (游戏层)                       │
│  AActor::GetLifetimeReplicatedProps()                       │
│  DOREPLIFETIME / DOREPLIFETIME_CONDITION 宏                 │
│  Push Model: MARK_PROPERTY_DIRTY_FROM_NAME()                │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│              Replication Driver (复制驱动层)                  │
│  UReplicationDriver (抽象接口)                                │
│  ├── 默认: UNetDriver::ServerReplicateActors()               │
│  └── 可选: UReplicationGraph (基于图的复制)                   │
│  负责: 决定哪些 Actor 需要复制给哪些连接                       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│             Object Replication (对象复制层)                   │
│  FObjectReplicator (每个对象 <-> 每个连接的复制器)            │
│  FRepLayout (每种类型的共享复制布局)                          │
│  FReplicationChangelistMgr (变更列表管理)                     │
│  负责: 比较属性变化、序列化属性数据                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│                Channel Layer (信道层)                        │
│  UActorChannel (Actor 专用信道)                               │
│  UControlChannel (控制信道，握手用)                           │
│  UChannel (基类)                                             │
│  负责: 封装 Bunch、管理信道生命周期                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│              Transport Layer (传输层)                         │
│  UNetDriver (UIpNetDriver / UWebSocketNetDriver 等)          │
│  UNetConnection (每个客户端连接)                              │
│  Packet / Bunch / 可靠性 / 重传                               │
│  负责: 底层网络传输                                           │
└─────────────────────────────────────────────────────────────┘
```

### 关键类型一览

| 类型 | 文件 | 职责 |
|------|------|------|
| `FRepLayout` | [RepLayout.h](Engine/Public/Net/RepLayout.h) | 类型的复制属性布局，每种 UClass/UStruct/UFunction 一份共享实例 |
| `FRepLayoutCmd` | 同上 | 单个属性的复制命令：偏移、大小、序列化方式 |
| `FRepParentCmd` | 同上 | 顶层复制属性的描述：条件、RepNotify、Flags |
| `FObjectReplicator` | [DataReplication.h](Engine/Public/Net/DataReplication.h) | 每个对象-每个连接的单体复制器 |
| `FRepState` | [RepLayout.h](Engine/Public/Net/RepLayout.h) | 每个对象-每个连接的复制状态（发送/接收） |
| `FReplicationChangelistMgr` | 同上 | 每个对象的变更列表管理器（跨连接共享） |
| `UActorChannel` | [ActorChannel.h](Engine/Classes/Engine/ActorChannel.h) | Actor 复制信道 |
| `UNetDriver` | [NetDriver.h](Engine/Classes/Engine/NetDriver.h) | 网络驱动基类 |
| `UReplicationDriver` | [ReplicationDriver.h](Engine/Classes/Engine/ReplicationDriver.h) | 复制驱动抽象接口 |
| `UNetConnection` | [NetConnection.h](Engine/Classes/Engine/NetConnection.h) | 客户端连接 |

---

## 2. 核心数据流

### 2.1 服务器端发送流程

```
UNetDriver::TickFlush()
  └─→ ServerReplicateActors()
        ├─→ ServerReplicateActors_PrepConnections()     // 准备连接，设置 ViewTarget
        ├─→ ServerReplicateActors_BuildConsiderList()   // 构建候选 Actor 列表
        │     ├─ 检查 bPendingNetUpdate
        │     ├─ 检查 NextUpdateTime (频率限制)
        │     ├─ 过滤 RemoteRole == ROLE_None
        │     └─ 过滤 InitialDormant Actors
        ├─→ [可选] ReplicationDriver->ServerReplicateActors()  // ReplicationGraph 路径
        └─→ 对每个 Connection (按优先级排序):
              ├─→ ServerReplicateActors_PrioritizeActors()    // 计算优先级
              └─→ 对每个优先的 Actor:
                    └─→ UActorChannel::ReplicateActor()
                          └─→ FObjectReplicator::ReplicateProperties()
                                ├─→ FRepLayout::CompareProperties()    // 比较新旧值
                                ├─→ FRepLayout::SendProperties()       // 序列化变化
                                └─→ [若启用] 使用共享序列化避免重复工作
```

关键文件：
- [NetDriver.cpp:4941](Engine/Private/NetDriver.cpp) — `ServerReplicateActors_PrepConnections`
- [NetDriver.cpp:5046](Engine/Private/NetDriver.cpp) — `ServerReplicateActors_BuildConsiderList`
- [NetDriver.cpp:5264](Engine/Private/NetDriver.cpp) — `ServerReplicateActors_PrioritizeActors`

### 2.2 客户端接收流程

```
UNetDriver::TickDispatch()
  └─→ 接收 Packet
        └─→ UNetConnection::ReceivedRawPacket()
              └─→ 拆解为 Bunch
                    └─→ 路由到对应 UChannel
                          └─→ UActorChannel::ReceivedBunch()
                                ├─→ 识别 Bunch 类型 (Spawn / Content / RPC)
                                └─→ FObjectReplicator::ReceivedBunch()
                                      ├─→ FRepLayout::ReceiveProperties()  // 反序列化
                                      ├─→ 处理 Unmapped GUIDs (延迟绑定)
                                      └─→ FRepLayout::CallRepNotifies()    // 调用 RepNotify
```

---

## 3. RepLayout：复制属性布局

### 3.1 概述

`FRepLayout` 是整个属性复制系统的核心。**每种类型（UClass/UStruct/UFunction）只有一份 FRepLayout 实例**，该类型的全部实例共享这个布局。它在首次需要时通过 `FRepLayout::CreateFromClass()` 创建并缓存。

文件：[RepLayout.h:1085-1910](Engine/Public/Net/RepLayout.h)

### 3.2 Command 系统

FRepLayout 使用 **Command（命令）** 的概念描述如何读取、写入、比较和序列化属性。

**Parent Command (FRepParentCmd)：描述顶层属性**

```cpp
// RepLayout.h:780 — FRepParentCmd
class FRepParentCmd {
    FProperty* Property;            // 属性指针
    FName CachedPropertyName;       // 缓存属性名
    int32 ArrayIndex;               // 静态数组索引
    int32 ShadowOffset;             // Shadow Buffer 中的偏移
    uint16 CmdStart;                // 子命令起始索引
    uint16 CmdEnd;                  // 子命令结束索引
    ELifetimeCondition Condition;   // 复制条件 (COND_None, COND_OwnerOnly 等)
    ELifetimeRepNotifyCondition RepNotifyCondition; // RepNotify 触发条件
    int32 RepNotifyNumParams;       // RepNotify 函数参数数量
    ERepParentFlags Flags;          // 属性类别标志
};
```

**Child Command (FRepLayoutCmd)：描述序列化细节**

```cpp
// RepLayout.h:856 — FRepLayoutCmd
class FRepLayoutCmd {
    FProperty* Property;      // 属性指针
    uint16 EndCmd;             // 数组内部元素的跳转指令
    uint16 ElementSize;        // 数组元素大小
    int32 Offset;              // 对象内存中的绝对偏移
    int32 ShadowOffset;        // Shadow Buffer 中的绝对偏移
    uint16 RelativeHandle;     // 相对句柄 (1-based)
    uint16 ParentIndex;        // 父命令索引
    uint32 CompatibleChecksum; // 兼容性校验和
    ERepLayoutCmdType Type;    // 属性类型 (PropertyBool, PropertyFloat, PropertyObject 等)
    ERepLayoutCmdFlags Flags;  // 序列化标志
};
```

**Parent Flags（属性类别标志）**

| Flag | 含义 |
|------|------|
| `IsLifetime` | 生命周期属性（绝大多数属性） |
| `IsConditional` | 有条件复制（COND_* 非 None） |
| `IsCustomDelta` | 使用自定义 Delta 压缩 (如 FRepMovement) |
| `IsNetSerialize` | 使用自定义 NetSerialize |
| `IsFastArray` | FastArraySerializer 类型 |
| `HasObjectProperties` | 包含 UObject 引用 |
| `IsStructProperty` | FStructProperty 类型 |

**Command Type（底层类型枚举）**

```cpp
// RepLayout.h:722 — ERepLayoutCmdType
enum class ERepLayoutCmdType : uint8 {
    DynamicArray  = 0,   // 动态数组
    Return        = 1,   // 数组结束 / 流终止
    Property      = 2,   // 通用属性
    PropertyBool  = 3,
    PropertyFloat = 4,
    PropertyInt   = 5,
    PropertyByte  = 6,
    PropertyName  = 7,
    PropertyObject = 8,  // UObject* 引用
    PropertyUInt32 = 9,
    PropertyVector = 10,
    PropertyRotator = 11,
    PropertyPlane = 12,
    PropertyVector100 = 13,
    PropertyNetId = 14,  // FUniqueNetId
    RepMovement   = 15,  // FRepMovement (特殊优化)
    // ... 更多类型
    NetSerializeStructWithObjectReferences = 25,
};
```

### 3.3 属性展开示例

假设有这样的类：

```cpp
UCLASS()
class UMyObject : public UObject
{
    UPROPERTY(Replicated) int32 Health;          // → 1 Parent + 1 Child
    UPROPERTY(Replicated) TArray<float> Scores;  // → 1 Parent + 1 Array Child + N float Children
    UPROPERTY(Replicated) FVector Location;      // → 1 Parent + 0 Child (Struct 展开)
                                                   //   → 3 float Children (X, Y, Z)
};
```

### 3.4 FRepLayout 关键函数

| 函数 | 职责 |
|------|------|
| `CreateFromClass()` | 从 UClass 创建 FRepLayout |
| `CreateShadowBuffer()` | 创建 Shadow Buffer（缓存上一帧属性值） |
| `CreateReplicationChangelistMgr()` | 创建变更列表管理器 |
| `CreateRepState()` | 创建连接级 RepState |
| `CompareProperties()` | 比较源数据与 Shadow Buffer，生成 Changelist |
| `SendProperties()` | 根据 Changelist 序列化属性数据 |
| `ReceiveProperties()` | 反序列化属性数据并写入对象 |
| `CallRepNotifies()` | 触发 RepNotify 回调 |
| `UpdateUnmappedObjects()` | 解析未映射的 GUID 引用 |
| `DiffProperties()` | 通用属性差异比较 |

---

## 4. FObjectReplicator：对象复制器

### 4.1 概述

`FObjectReplicator` 是每个被复制对象在**每个连接**上的复制器。它是连接 FRepLayout（共享的类型信息）和实际网络收发的桥梁。

文件：[DataReplication.h:73-323](Engine/Public/Net/DataReplication.h)

### 4.2 关键成员

```cpp
class FObjectReplicator {
public:
    FNetworkGUID ObjectNetGUID;             // 对象的网络 GUID
    TSharedPtr<FRepLayout> RepLayout;       // 共享的复制布局
    TUniquePtr<FRepState> RepState;         // 连接级复制状态
    UObject* ObjectPtr;                     // 被复制的对象（强引用）
    UNetConnection* Connection;             // 所属连接
    UActorChannel* OwningChannel;           // 所属 Actor Channel

    // RPC 相关
    FOutBunch* RemoteFunctions;             // 待发送的 RPC
    TArray<FRPCPendingLocalCall> PendingLocalRPCs; // 待执行的接收 RPC

    // GUID 追踪
    TSet<FNetworkGUID> ReferencedGuids;     // 引用的网络对象 GUID

    TSharedPtr<FReplicationChangelistMgr> ChangelistMgr; // 变更列表管理

    // 状态标志
    uint32 bLastUpdateEmpty : 1;    // 上次更新没有需要复制的属性
    uint32 bOpenAckCalled : 1;      // 信道已确认打开
    uint32 bForceUpdateUnmapped : 1; // 强制下一帧更新未映射对象
    uint32 bHasReplicatedProperties : 1; // 本帧已复制属性
};
```

### 4.3 核心流程

```cpp
// 1. 发送端: ReplicateProperties
bool FObjectReplicator::ReplicateProperties(FOutBunch& Bunch, FReplicationFlags RepFlags)
{
    // 更新 ChangelistMgr: 比较对象当前属性与 Shadow State
    FRepLayout::UpdateChangelistMgr(RepState, ChangelistMgr, Object, ...);

    // 获取变更列表状态
    FRepChangelistState* ChangelistState = ChangelistMgr->GetRepChangelistState();

    // 序列化所有变化的属性
    RepLayout->ReplicateProperties(RepState, ChangelistState, Data, Class, Channel, Writer, RepFlags);
}

// 2. 接收端: ReceivedBunch
bool FObjectReplicator::ReceivedBunch(FNetBitReader& Bunch, ...)
{
    // 反序列化属性并应用到对象
    RepLayout->ReceiveProperties(OwningChannel, Class, RepState, Object, Bunch, ...);

    // 触发 RepNotify
    RepLayout->CallRepNotifies(RepState, Object);
}
```

---

## 5. Changelist 机制：变更追踪

### 5.1 设计思想

Changelist 是 UE 复制系统中最核心的设计之一。它不是每次都发送完整的对象属性，而是只发送**自上次成功发送以来发生变化的属性**。Changelist 本身不包含属性值，只包含属性句柄（Handle）——描述**哪些属性发生了变化**。

### 5.2 Changelist 结构

Changelist 是一维数组 `TArray<uint16>`，支持嵌套数组。语法如下：

```
Terminator        ::=  0
Handle            ::=  Integer between 1 ~ 65535
Number            ::=  Integer between 0 ~ 65535
Changelist        ::=  <Terminator> | <Handle><Changelist> | <Handle><ArrayChangelist><Changelist>
ArrayChangelist   ::=  <Number><Changelist>
```

Handle 是 **1-based** 的相对索引（0 作为终止符），表示在**当前数组深度**中哪个属性发生了变化。对于嵌套数组，Handle 是针对每个深度的元素类型的。

### 5.3 变更追踪的层级结构

```
FReplicationChangelistMgr (每个对象，跨连接共享)
  └─→ FRepChangelistState
        ├─→ StaticBuffer (Shadow Buffer: 最近一次比较时的属性拷贝)
        ├─→ ChangeHistory[64] (循环缓冲区，存储最近的变更列表)
        │     └─→ FRepChangedHistory
        │           ├─→ TArray<uint16> Changed (变更属性句柄列表)
        │           └─→ FPacketIdRange OutPacketIdRange (发送该变更的包范围)
        ├─→ CompareIndex (递增的比较计数器)
        └─→ SharedSerialization (共享序列化数据)

FRepState (每个对象-每个连接，不共享)
  └─→ FSendingRepState (服务器端发送状态)
        ├─→ ChangeHistory[32] (连接级变更历史)
        ├─→ LastChangelistIndex (上次发送到该连接的变更索引)
        ├─→ LastCompareIndex (上次比较索引)
        ├─→ PreOpenAckHistory (信道打开前的变更暂存)
        ├─→ LifetimeChangelist (从信道打开以来的所有变更)
        ├─→ InactiveChangelist (条件不满足的属性的变更暂存)
        └─→ RecentCustomDeltaState[] (Custom Delta 属性的最近状态)
```

### 5.4 变更列表管理流程

```cpp
// 每次 ReplicateActor 时的流程:

// Step 1: UpdateChangelistMgr — 比较当前属性值与 Shadow State
ERepLayoutResult FRepLayout::UpdateChangelistMgr(
    FSendingRepState* RepState,
    FReplicationChangelistMgr& InChangelistMgr,
    const UObject* InObject,
    const uint32 ReplicationFrame,
    const FReplicationFlags& RepFlags,
    const bool bForceCompare)
{
    // 使用 Push Model 的快速路径
    #if WITH_PUSH_MODEL
    if (!bForceCompare && RepState->LastCompareIndex == ChangelistState->CompareIndex) {
        return ERepLayoutResult::Empty; // 无新变化
    }
    #endif

    // 传统路径: 逐属性比较
    return CompareProperties(RepState, &ChangelistState, Data, RepFlags, bForceCompare);
}

// Step 2: CompareProperties — 将变化写入新的 ChangeHistory 条目
//   使用 FRepLayoutCmd 的偏移信息，逐个比较 Shadow Buffer 和当前对象内存

// Step 3: UpdateChangelistHistory — 管理 ChangeHistory 循环缓冲区
//   若缓冲区满了，合并最旧的条目

// Step 4: SendProperties — 迭代 Changelist 并序列化属性到 Bunch
void FRepLayout::SendProperties(
    FSendingRepState* RepState,
    const FConstRepObjectDataBuffer Data,
    FNetBitWriter& Writer,
    TArray<uint16>& Changed, ...)
{
    FRepHandleIterator Iterator(...);
    while (Iterator.NextHandle()) {
        const FRepLayoutCmd& Cmd = Cmds[Iterator.CmdIndex];
        // 根据 Cmd.Type 选择对应的序列化方式
        // Bool → Writer.WriteBit()
        // Float → Writer << value
        // Object → PackageMap->SerializeObject()
        // Struct → NetSerialize() 或递归
    }
}
```

### 5.5 NAX 重传机制

```cpp
// RepLayout.h:638 — FSendingRepState::ChangeHistory
// 发送的每个 Changelist 记录在循环缓冲区中，跟踪发送包范围

// 当收到 NAK:
void FObjectReplicator::ReceivedNak(int32 NakPacketId) {
    // 标记包含该包的所有 ChangeHistory 条目为 Resend
    // 下次 ReplicateProperties 时会合并这些条目并重发
}

// 当所有关联包都被 ACK:
// 对应的 ChangeHistory 条目从缓冲区移除，释放空间
```

---

## 6. 信道层：UActorChannel 与 UChannel

### 6.1 Channel 层次结构

```
UChannel (Engine/Channel.h)
  ├── UControlChannel    — 连接握手、控制消息 (Hello, Welcome, Login 等)
  ├── UVoiceChannel      — 语音数据传输
  └── UActorChannel      — Actor 及其子对象的属性复制和 RPC
```

### 6.2 UActorChannel 详解

文件：[ActorChannel.h:77-200](Engine/Classes/Engine/ActorChannel.h)

```cpp
UCLASS(transient, customConstructor, MinimalAPI)
class UActorChannel : public UChannel
{
public:
    // === Actor 关联 ===
    TObjectPtr<AActor> Actor;         // 该信道关联的 Actor
    FNetworkGUID ActorNetGUID;        // Actor 的 Guid (Actor 尚未解析时也可用)

    // === 复制器 ===
    TSharedPtr<FObjectReplicator> ActorReplicator;              // Actor 自身的复制器
    TMap<UObject*, TSharedRef<FObjectReplicator>> ReplicationMap; // 子对象复制器表

    // === 时序 ===
    double RelevantTime;     // 上次对客户端可见的时间
    double LastUpdateTime;   // 上次复制的时间

    // === 状态标志 ===
    uint32 SpawnAcked:1;                // 生成确认
    uint32 bForceCompareProperties:1;   // 单帧强制比较全部属性
    uint32 bIsReplicatingActor:1;       // 正在复制中 (防递归)
    uint32 bClearRecentActorRefs:1;     // 关闭时是否清除其他信道的最近引用
    uint32 bHoldQueuedExportBunchesAndGUIDs:1; // 暂缓导出 Bunch

    // === 待处理队列 ===
    TArray<FInBunch*> QueuedBunches;    // 等待 GUID 解析的排队 Bunch

private:
    uint32 bSkipRoleSwap:1;             // 是否跳过 Role/RemoteRole 交换
};
```

### 6.3 UActorChannel::ReplicateActor() 流程

```cpp
// ActorChannel.h:190 — 核心复制入口
int64 UActorChannel::ReplicateActor()
{
    // 1. 检查 Actor 是否准备好复制 (已 BeginPlay)
    if (!IsActorReadyForReplication()) return 0;

    // 2. 设置 bIsReplicatingActor 防递归
    bIsReplicatingActor = true;

    // 3. 如果是首次复制，发送 Spawn 信息
    if (!SpawnAcked) {
        // 写入: Actor Class, Spawn Location/Rotation, NetGUID 分配
        //       Actor NetGUID, Component NetGUIDs
    }

    // 4. Actor 自身属性复制
    ActorReplicator->ReplicateProperties(Bunch, RepFlags);

    // 5. 子对象属性复制 (Component 等)
    for (auto& Pair : ReplicationMap) {
        Pair.Value->ReplicateProperties(Bunch, RepFlags);
    }

    // 6. 发送排队的 RPC
    // ...

    bIsReplicatingActor = false;
}
```

### 6.4 Bunch 结构

ActorChannel Bunch 的格式在代码注释中有详细说明（[ActorChannel.h:46-75](Engine/Classes/Engine/ActorChannel.h)）：

```
+----------------------+--------------------------------------+
| SpawnInfo (仅首次)    | Actor Class, Location, Rotation      |
|                      | Actor NetGUID, Component NetGUIDs    |
+----------------------+--------------------------------------+
| Content Chunks (x N) | 每个复制对象 (Actor + Components):  |
|  - NetGUID ObjRef    |   - 对象 ID (由 PackageMap 压缩)    |
|  - Properties...     |   - 变更的属性 (Handle→Value 序列)  |
|  - RPCs...           |   - RPC 标识符 + 参数              |
+----------------------+--------------------------------------+
| </End Tag>           |  终止标记                            |
+----------------------+--------------------------------------+
```

---

## 7. 网络驱动：UNetDriver

### 7.1 概述

`UNetDriver` 是整个网络系统的顶层管理者。文件：[NetDriver.h](Engine/Classes/Engine/NetDriver.h)

```cpp
UCLASS(Abstract, config=Engine)
class UNetDriver : public UObject
{
    // === 连接管理 ===
    TArray<UNetConnection*> ClientConnections;      // 服务器端：所有客户端连接
    UNetConnection* ServerConnection;               // 客户端：与服务器的连接

    // === 世界关联 ===
    UWorld* World;

    // === 网络对象列表 ===
    TSharedPtr<FNetworkObjectList> NetworkObjectList; // 服务器端：所有可复制对象

    // === 复制驱动 ===
    UReplicationDriver* ReplicationDriver;          // 可选的复制驱动 (如 ReplicationGraph)

    // === 关键方法 ===
    virtual void TickDispatch(float DeltaSeconds);  // 接收网络数据
    virtual void TickFlush(float DeltaSeconds);     // 发送网络数据 (含复制)
    virtual int32 ServerReplicateActors(float DeltaSeconds); // 服务器端复制
};
```

### 7.2 服务器复制管道

`ServerReplicateActors` 在 `NetDriver.cpp` 中实现，包含几个步骤：

1. **PreReplication** (`ServerReplicateActors_PrepConnections`): 为每个连接设置 ViewTarget
2. **BuildConsiderList** (`ServerReplicateActors_BuildConsiderList`): 构建候选列表，过滤：
   - `bPendingNetUpdate` 未设置且未到 `NextUpdateTime` 的 Actor
   - `RemoteRole == ROLE_None` 的 Actor
   - PendingKill 的 Actor
   - 尚未初始化的 Actor
   - 正在 Streaming 的 Level 中的 Actor
   - InitialDormant 的 Actor
3. **PrioritizeActors** (`ServerReplicateActors_PrioritizeActors`): 根据 `GetNetPriority()` 排序
4. **ReplicateActors**: 对排序后的 Actor 调用 `UActorChannel::ReplicateActor()`

### 7.3 连接握手流程

NetDriver.h:76-156 详细描述了握手流程：

```
Client (UPendingNetGame)           Server (UWorld)
  │                                    │
  ├─ NMT_Hello ──────────────────────→│  版本检查
  │←────────────────────── NMT_Challenge │  返回挑战数据
  ├─ NMT_Login (含挑战应答) ──────────→│  验证
  │                                    ├─ AGameModeBase::PreLogin()
  │                                    ├─ AGameModeBase::GameWelcomePlayer()
  │←─────────────────────── NMT_Welcome │  地图信息
  ├─ NMT_NetSpeed ────────────────────→│  设置网速
  │                                    │
  │  ← 握手完成，开始游戏 →              │
```

---

## 8. PackageMap 与 NetGUID 系统

### 8.1 概述

PackageMap 负责 **UObject 指针与 FNetworkGUID 之间的双向映射和序列化**。当属性包含 UObject* 引用时，PackageMap 会将该引用序列化为紧凑的网络 GUID，而不是完整的对象路径。

文件：[PackageMapClient.h](Engine/Classes/Engine/PackageMapClient.h)

### 8.2 FNetGUIDCache

```cpp
// PackageMapClient.h:177 — 全局 GUID 缓存
class FNetGUIDCache {
    TMap<FNetworkGUID, FNetGuidCacheObject> ObjectLookup;  // GUID → 对象信息
    TMap<UObject*, FNetworkGUID> NetGUIDLookup;            // 对象 → GUID (反向)
    TMap<FNetworkGUID, TSet<FObjectReplicator*>> GuidToReplicatorMap; // GUID → 复制器
    TSet<FObjectReplicator*> UnmappedReplicators;           // 有未绑定 GUID 的复制器
};
```

**FNetworkGUID 的生命周期：**

- 服务端首次复制对象时分配 GUID 并发送给客户端
- 客户端收到后在本地创建映射
- 对象销毁时 GUID 被标记为无效
- 客户端可能暂时持有 "Unmapped GUID"（对象尚未加载），后续通过异步加载或延迟绑定解析

### 8.3 UPackageMapClient

```cpp
// PackageMapClient.h:436 — 连接级 PackageMap
class UPackageMapClient : public UPackageMap {
    // 序列化 UObject 引用 → NetGUID
    virtual bool SerializeObject(FArchive& Ar, UClass* Class, UObject*& Object, FNetworkGUID* Guid);

    // 序列化新 Actor 的 Spawn 信息
    virtual bool SerializeNewActor(FArchive& Ar, UActorChannel* Channel, AActor*& Actor);

    // ACK/NAK 管理 (GUID 导出也依赖可靠性确认)
    void ReceivedNak(int32 NakPacketId);
    void ReceivedAck(int32 AckPacketId);

    // GUID → Object 和 Object → GUID 转换
    UObject* GetObjectFromNetGUID(const FNetworkGUID& Guid, bool bIgnoreMustBeMapped);
    FNetworkGUID GetNetGUIDFromObject(const UObject* Object) const;
};
```

### 8.4 Unmapped GUID 延迟绑定

当客户端收到包含尚未加载的对象的属性数据时：

1. 属性被标记为包含 Unmapped GUID
2. 该属性值暂时置为 `nullptr`
3. GUID 加到 `PendingGuidResolves` 集合
4. 后续的 Bunches 被排队（`QueuedBunches`）
5. 当对象加载完成时，`UpdateUnmappedObjects()` 将正确的指针写回属性
6. 排队的数据被重新处理

### 9.5 FClassNetCache / FFieldNetCache

用于高速 RPC 字段查找的缓存系统：

```cpp
// 每个 UClass 一个 FClassNetCache:
//   缓存该类所有 RPC 函数的 FFieldNetCache
//   避免每次 RPC 调用时遍历所有函数进行反射查找

// FFieldNetCache:
//   FieldNetIndex: RPC 在类中的快速索引
//   用于标识 RPC 函数，而非每次序列化函数名
```

---

## 9. Actor 网络复制详解

### 9.1 概述

Actor 是 UE 网络复制的核心对象。所有 Actor 的复制都由 `AActor::GetLifetimeReplicatedProps()` 定义其属性复制列表，由 `UActorChannel` 管理其生命周期。

Actor 复制文件：[ActorReplication.cpp](Engine/Private/ActorReplication.cpp)

### 9.2 AActor 的复制属性注册

```cpp
// ActorReplication.cpp:532
void AActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    // 1. 先处理 Blueprint 生成的复制属性
    UBlueprintGeneratedClass* BPClass = Cast<UBlueprintGeneratedClass>(GetClass());
    if (BPClass != NULL) {
        BPClass->GetLifetimeBlueprintReplicationList(OutLifetimeProps);
    }

    // 2. AActor 核心复制属性 (全部使用 Push Model)
    FDoRepLifetimeParams SharedParams;
    SharedParams.bIsPushBased = true;

    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, bReplicateMovement, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, Role, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, RemoteRole, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, Owner, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, bHidden, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, bTearOff, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, bCanBeDamaged, SharedParams);
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, Instigator, SharedParams);

    // 3. 有条件属性
    //    AttachmentReplication: COND_Custom, REPNOTIFY_Always
    //    ReplicatedMovement: COND_SimulatedOrPhysics, REPNOTIFY_Always
    FDoRepLifetimeParams AttachmentReplicationParams{COND_Custom, REPNOTIFY_Always, true};
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, AttachmentReplication, AttachmentReplicationParams);

    FDoRepLifetimeParams ReplicatedMovementParams{COND_SimulatedOrPhysics, REPNOTIFY_Always, true};
    DOREPLIFETIME_WITH_PARAMS_FAST(AActor, ReplicatedMovement, ReplicatedMovementParams);
}
```

### 9.3 网络优先级

```cpp
// ActorReplication.cpp:47 — GetNetPriority
float AActor::GetNetPriority(const FVector& ViewPos, const FVector& ViewDir,
    AActor* Viewer, AActor* ViewTarget, UActorChannel* InChannel, float Time, bool bLowBandwidth)
{
    // 1. 如果 bNetUseOwnerRelevancy，继承 Owner 的优先级
    if (bNetUseOwnerRelevancy && Owner) {
        return Owner->GetNetPriority(...);
    }

    // 2. ViewTarget 或 Instigator 拥有最高优先级 (×4)
    if (ViewTarget && (this == ViewTarget || GetInstigator() == ViewTarget)) {
        Time *= 4.f;
    }
    // 3. 基于空间关系的优先级调整
    else if (!IsHidden() && GetRootComponent() != NULL) {
        FVector Dir = GetActorLocation() - ViewPos;
        float DistSq = Dir.SizeSquared();

        if ((ViewDir | Dir) < 0.f) {                    // 在视野后方
            if (DistSq > NEARSIGHTTHRESHOLDSQUARED) Time *= 0.2f;
            else if (DistSq > CLOSEPROXIMITYSQUARED) Time *= 0.4f;
        }
        else if (DistSq < FARSIGHTTHRESHOLDSQUARED &&
                 FMath::Square(ViewDir | Dir) > 0.5f * DistSq) {
            Time *= 2.f;                                  // 正前方被注视区域
        }
        else if (DistSq > MEDSIGHTTHRESHOLDSQUARED) {
            Time *= 0.4f;                                 // 远处
        }
    }

    return NetPriority * Time;   // NetPriority 可被子类覆盖
}
```

### 9.4 PreNetReceive / PostNetReceive

```cpp
// ActorReplication.cpp:139 — PreNetReceive
void AActor::PreNetReceive()
{
    // 保存复制前的 Actor 状态，用于后续比较
    SavedbHidden = IsHidden();
    SavedOwner = Owner;
    SavedbRepPhysics = GetReplicatedMovement().bRepPhysics;
    SavedRole = GetLocalRole();
}

// ActorReplication.cpp:147 — PostNetReceive
void AActor::PostNetReceive()
{
    // 在属性复制完成后进行状态同步
    // 例如: 根据 bHidden 变化更新组件可见性
    //       Role 交换后的角色调整
}
```

### 9.5 网络角色 (NetRole)

| Role | 含义 |
|------|------|
| `ROLE_None` | 不复制该 Actor 到此连接 |
| `ROLE_SimulatedProxy` | 模拟代理：由服务器复制，客户端模拟 |
| `ROLE_AutonomousProxy` | 自主代理：客户端拥有控制权 |
| `ROLE_Authority` | 权威端：服务器拥有完全控制权 |

角色在属性复制前由 `FScopedActorRoleSwap` 临时交换，确保 RepNotify 在正确的角色上下文中执行。

### 9.6 网络休眠 (Dormancy)

```cpp
// 休眠类型
enum ENetDormancy {
    DORM_Never,          // 从不休眠
    DORM_Awake,          // 当前活跃
    DORM_DormantAll,     // 对所有连接休眠
    DORM_DormantPartial,  // 对部分连接休眠 (需 ReplicationGraph 支持)
    DORM_Initial,        // 初始休眠，等待显式 FlushNetDormancy
};

// 进入休眠时: Actor 停止每帧比较属性
// 退出休眠时: 强制发送全部属性 (因为休眠期间跳过了所有变更)
```

### 9.7 子对象复制

Actor 的 Component 和注册的子对象也会被复制：

```cpp
// UActorChannel::ReplicationMap 管理每个子对象的 FObjectReplicator
// 子对象通过 UObject::IsSupportedForNetworking() 注册
// 子对象通过 Actor 的 GetSubObjectsForReplication() 或 ReplicateSubobjects() 返回
```

### 9.8 应用：常见 Actor 网络模式

**模式 1：标准服务器权威 Actor**

```cpp
UCLASS()
class AMyReplicatedActor : public AActor
{
    GENERATED_BODY()
public:
    UPROPERTY(Replicated) int32 Score;
    UPROPERTY(ReplicatedUsing=OnRep_Health) float Health;

    UFUNCTION() void OnRep_Health();  // 客户端收到 Health 变化时调用

    void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AMyReplicatedActor, Score);
        DOREPLIFETIME_CONDITION(AMyReplicatedActor, Health, COND_OwnerOnly);
    }
};
```

**模式 2：使用 Push Model 优化高频 Actor**

```cpp
UCLASS()
class AFastMovingActor : public AActor
{
    UPROPERTY(Replicated) FVector Velocity;  // 每帧可能变化

    void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        FDoRepLifetimeParams Params;
        Params.bIsPushBased = true;  // 启用 Push Model
        DOREPLIFETIME_WITH_PARAMS(AFastMovingActor, Velocity, Params);
    }

    void Tick(float DeltaTime) override
    {
        Velocity = CalculateVelocity();
        MARK_PROPERTY_DIRTY_FROM_NAME(AFastMovingActor, Velocity, this);
        // Push Model 跳过每帧比较，只有被标记 dirty 时才复制
    }
};
```

**模式 3：自定义复制条件**

```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // COND_OwnerOnly: 只给 Owner 发送
    DOREPLIFETIME_CONDITION(AMyActor, AmmoCount, COND_OwnerOnly);

    // COND_SkipOwner: 不给 Owner 发送
    DOREPLIFETIME_CONDITION(AMyActor, Position, COND_SkipOwner);

    // COND_Custom: 由 GetReplicatedCustomConditionState 控制
    DOREPLIFETIME_CONDITION(AMyActor, TeamData, COND_Custom);
}

void AMyActor::GetReplicatedCustomConditionState(FCustomPropertyConditionState& OutState) const
{
    // 根据游戏逻辑动态决定是否复制 TeamData
    DOREPDYNAMICCONDITION_INITCONDITION_FAST(AMyActor, TeamData,
        IsOnSameTeamAsConnectionOwner());
}
```

---

## 10. 属性复制宏系统

文件：[UnrealNetwork.h](Engine/Public/Net/UnrealNetwork.h)

### 10.1 宏层级

```
基础注册:
  DOREPLIFETIME(Class, Property)
    └─→ DOREPLIFETIME_WITH_PARAMS(Class, Property, FDoRepLifetimeParams())

条件注册:
  DOREPLIFETIME_CONDITION(Class, Property, COND_OwnerOnly)
    └─→ 创建 LocalDoRepParams.Condition = COND_OwnerOnly
    └─→ DOREPLIFETIME_WITH_PARAMS(Class, Property, LocalDoRepParams)

带 RepNotify 条件:
  DOREPLIFETIME_CONDITION_NOTIFY(Class, Property, COND, REPNOTIFY_Always)

快速版本 (使用编译期 RepIndex):
  DOREPLIFETIME_WITH_PARAMS_FAST(Class, Property, Params)
    └─→ 使用 ENetFields_Private::Property (编译期生成)
    └─→ 无需反射查找，性能更高

禁用属性:
  DISABLE_REPLICATED_PROPERTY(Class, Property)
  DISABLE_ALL_CLASS_REPLICATED_PROPERTIES(Class, SuperClassBehavior)

重置条件:
  RESET_REPLIFETIME_CONDITION(Class, Property, COND_New)
  RESET_REPLIFETIME_WITH_PARAMS(Class, Property, NewParams)

自定义条件:
  DOREPCUSTOMCONDITION_ACTIVE_FAST(Class, Property, Active)
  DOREPCUSTOMCONDITION_SETACTIVE_FAST(Class, Property, Active)

动态条件:
  DOREPDYNAMICCONDITION_INITCONDITION_FAST(Class, Property, ConditionFn)
  DOREPDYNAMICCONDITION_SETCONDITION_FAST(Class, Property, NewCondition)

激活覆盖:
  DOREPLIFETIME_ACTIVE_OVERRIDE(Class, Property, IsActive)
```

### 10.2 FDoRepLifetimeParams

```cpp
// UnrealNetwork.h:134 — 属性复制参数
struct FDoRepLifetimeParams {
    ELifetimeCondition Condition = COND_None;           // 复制条件
    ELifetimeRepNotifyCondition RepNotifyCondition = REPNOTIFY_OnChanged; // RepNotify 触发条件
    bool bIsPushBased = false;                          // 是否使用 Push Model

    #if UE_WITH_IRIS
    CreateAndRegisterReplicationFragmentFunc CreateAndRegisterReplicationFragmentFunction = nullptr;
    #endif
};
```

### 9.3 FAST vs 非 FAST 宏

- **FAST 版本** (`DOREPLIFETIME_WITH_PARAMS_FAST`): 使用编译期生成的 `ENetFields_Private` 枚举值作为 RepIndex，避免反射查找，性能更高。
- **非 FAST 版本** (`DOREPLIFETIME_WITH_PARAMS`): 使用 `FindFieldChecked<FProperty>()` 运行时查找属性，兼容性更好。

---

## 11. 复制条件系统

### 11.1 ELifetimeCondition

```cpp
// CoreNet.h — 属性复制条件
enum ELifetimeCondition {
    COND_None                     = 0,  // 无条件复制
    COND_InitialOnly              = 1,  // 仅初始复制
    COND_OwnerOnly                = 2,  // 仅发送给 Owner
    COND_SkipOwner                = 3,  // 不发送给 Owner
    COND_SimulatedOnly            = 4,  // 仅发送给 Simulated Proxy
    COND_AutonomousOnly           = 5,  // 仅发送给 Autonomous Proxy
    COND_SimulatedOrPhysics       = 6,  // Simulated Proxy 或 Physics
    COND_InitialOrOwner           = 7,  // 初始或 Owner
    COND_Custom                   = 8,  // 自定义条件
    COND_ReplayOrOwner            = 9,  // Replay 或 Owner
    COND_ReplayOnly               = 10, // 仅 Replay
    COND_SimulatedOnlyNoReplay    = 11, // Simulated 但排除 Replay
    COND_SimulatedOrPhysicsNoReplay = 12,
    COND_SkipReplay               = 13, // 非 Replay
    COND_Dynamic                  = 14, // 动态条件 (每帧可变)
    COND_Never                    = 15, // 永不复制
    COND_Max                      = 16,
};
```

### 11.2 条件映射构建

```cpp
// RepLayout.h:135 — BuildConditionMapFromRepFlags
// 根据当前连接的 ReplicationFlags 构建条件映射:
//   bNetInitial → COND_InitialOnly
//   bNetOwner   → COND_OwnerOnly, COND_SkipOwner
//   bNetSimulated → COND_SimulatedOnly, COND_AutonomousOnly 等
//   bReplay     → COND_ReplayOnly, COND_SkipReplay
//   bRepPhysics → COND_SimulatedOrPhysics
```

### 11.3 Custom Condition

Custom Condition 通过 `AActor::GetReplicatedCustomConditionState()` 实现：

```cpp
// ActorReplication.cpp:561
void AActor::GetReplicatedCustomConditionState(FCustomPropertyConditionState& OutActiveState) const
{
    // 子类重写此方法，为 COND_Custom 属性设置 per-connection 条件
}
```

### 11.4 条件不满足时的处理

当某个属性因为条件不满足而无法发送给某连接时，其变更不会丢失：

- 变更被暂存到 `FSendingRepState::InactiveChangelist`
- 条件恢复满足时，`RebuildConditionalProperties()` 会将暂存的变更合并回活动变更列表
- 这确保条件性属性在条件恢复后会发送完整的当前值

---

## 12. Push Model 属性推送

### 12.1 设计动机

传统复制中，每帧都需要比较对象当前属性值与 Shadow State 以检测变化。对于大量属性或高频 Actor（角色移动等），这个比较开销很大。

Push Model 允许游戏代码**主动标记**哪些属性发生了变化，跳过全量比较。

### 12.2 使用方式

```cpp
// 注册时:
FDoRepLifetimeParams Params;
Params.bIsPushBased = true;
DOREPLIFETIME_WITH_PARAMS(AMyActor, Health, Params);

// 修改属性后:
Health -= DamageAmount;
MARK_PROPERTY_DIRTY_FROM_NAME(AMyActor, Health, this);
```

### 12.3 实现原理

Push Model 通过 `FPushModelPerNetDriverState` 追踪每个 NetDriver 上的脏属性状态：

```cpp
// RepLayout.cpp — CompareProperties 中的 Push Model 快速路径:
#if WITH_PUSH_MODEL
if (RepState->LastCompareIndex == ChangelistState->CompareIndex
    && !ChangelistState->HasAnyDirtyProperties()) {
    return ERepLayoutResult::Empty;  // 跳过比较，无变化
}
#endif

// 当 MARK_PROPERTY_DIRTY 被调用时:
//   设置 per-NetDriver 的脏标记 → 下一帧 CompareProperties 会处理
```

### 12.4 ERepLayoutFlags 中的 Push Model 支持

| Flag | 含义 |
|------|------|
| `PartialPushSupport` | 部分属性使用 Push Model |
| `FullPushSupport` | 全部属性 + FastArray 都使用 Push Model |
| `FullPushProperties` | 全部属性使用 Push Model (不含 FastArray) |

当所有属性都是 Push Model 时，`CompareProperties` 可以完全跳过逐属性比较，只检查脏标记。

---

## 13. 可靠性与重传机制

### 13.1 Packet 序号与 ACK/NAK

NetDriver.h:224-320 详细描述了可靠性机制：

```
Packet 序号:
  - 每个 UNetConnection 维护递增的包序号
  - 包序号不会重复使用 (即使重传)

Bunch 序号:
  - 每个 Channel 维护递增的 Bunch 序号 (仅可靠 Bunch)
  - 可靠 Bunch 可以被重传 (保留相同序号)

ACK 处理:
  - 收到的每个包都会发送 ACK (包含已收到包的最高序号)
  - 任何低于已确认序号的 ACK 被忽略
  - 序号间隙被视为 NAK (丢失确认)

重传:
  - 收到 NAK → 找到对应的可靠 Bunch → 重新发送到新包中
  - 所有 ACK 确认 → 从待确认列表中移除
```

### 13.2 Partial Bunch

当 Bunch 大于单个 Packet 承载上限时：

```
Large Bunch → [PartialInitial] [Partial...] [PartialFinal]
              └── Packet A ──┘└── Packet B ──┘└── Packet C ──┘

如果 PartialBunchReliableThreshold (net.PartialBunchReliableThreshold)
超过阈值，整个 Bunch 会被升级为可靠模式，确保原子性传输。
```

### 13.3 Changelist 层面的可靠性

```cpp
// RepLayout 的延迟合并策略:
// - 每个 Changelist 记录 OutPacketIdRange (发送该变更的包范围)
// - NAK 到来: 标记该 Changelist 为 Resend
// - 下次复制: 合并所有 Resend 的 Changelist
// - 缓冲区溢出: 将所有 Changelist 合并为单个 Monolithic Changelist
```

---

## 14. ReplicationDriver 与 ReplicationGraph

### 14.1 ReplicationDriver 接口

`UReplicationDriver` 是复制驱动的抽象接口，允许完全自定义"哪些 Actor 复制给哪些连接"的逻辑。

文件：[ReplicationDriver.h](Engine/Classes/Engine/ReplicationDriver.h)

```cpp
UCLASS(abstract, transient, config=Engine)
class UReplicationDriver : public UObject
{
    // 生命周期
    virtual void SetRepDriverWorld(UWorld* InWorld) = 0;
    virtual void InitForNetDriver(UNetDriver* InNetDriver) = 0;
    virtual void InitializeActorsInWorld(UWorld* InWorld) = 0;

    // Actor 管理
    virtual void AddNetworkActor(AActor* Actor) = 0;
    virtual void RemoveNetworkActor(AActor* Actor) = 0;
    virtual void ForceNetUpdate(AActor* Actor) = 0;

    // 休眠管理
    virtual void FlushNetDormancy(AActor* Actor, bool WasDormInitial) = 0;
    virtual void NotifyActorDormancyChange(AActor* Actor, ENetDormancy OldState) = 0;

    // 核心: 服务器复制入口
    virtual int32 ServerReplicateActors(float DeltaSeconds) = 0;

    // RPC 拦截 (可选)
    virtual bool ProcessRemoteFunction(AActor* Actor, UFunction* Function, ...) { return false; }
};
```

### 14.2 ReplicationGraph

`UReplicationGraph` 是 `UReplicationDriver` 的一个实现，位于 `Engine/Plugins/Runtime/ReplicationGraph/`。

**核心思想：**

ReplicationGraph 将所有需要复制的 Actor 组织成一个**节点图**。每个节点维护一组 Actor 列表，连接通过节点图收集需要复制的 Actor。

```
                  ┌────────────────┐
                  │  Root Node     │
                  └───────┬────────┘
           ┌──────────────┼──────────────┐
           │              │              │
  ┌────────▼─────┐ ┌─────▼──────┐ ┌─────▼──────────┐
  │AlwaysRelevant│ │ Spatialize │ │ Per-Connection  │
  │    Node      │ │    Node    │ │     Nodes       │
  └──────────────┘ └────────────┘ └────────────────┘

每个 Node 持有 Actor 列表 → GatherActorListsForConnection()
  → 按距离/可见性/优先级过滤 → 合并 → 排序 → 逐个复制
```

**关键优势：**

- **共享工作**: 空间化节点对所有连接共享 Actor 分类
- **可扩展**: 新增 Actor 只需加入正确的节点
- **可自定义**: 游戏可以添加自己的节点类型

**基础实现**: [BasicReplicationGraph.h](f:\GitHub\UnrealEngine\Engine\Plugins\Runtime\ReplicationGraph\Source\Public\BasicReplicationGraph.h) 提供了开箱即用的最小实现。

### 14.3 选择复制驱动

```ini
; DefaultEngine.ini
[/Script/OnlineSubsystemUtils.IpNetDriver]
ReplicationDriverClassName="/Script/MyGame.MyReplicationGraph"
```

或通过代码：

```cpp
UReplicationDriver::CreateReplicationDriverDelegate().BindLambda(
    [](UNetDriver* ForNetDriver, const FURL& URL, UWorld* World) -> UReplicationDriver* {
        return NewObject<UMyReplicationDriver>(GetTransientPackage());
    });
```

---

## 15. Iris：下一代复制系统

### 15.1 概述

Iris 是 UE5 中引入的下一代复制系统，代码通过 `UE_WITH_IRIS` 宏条件编译。它是一个可选的替代方案，旨在提供更高的性能和更好的可扩展性。

相关文件路径: `Engine/Source/Runtime/Experimental/Iris/`

### 15.2 Iris 与传统系统的关系

```cpp
// 在现有代码中可以看到 Iris 并行路径:
#if UE_WITH_IRIS
#include "Iris/IrisConfig.h"
#include "Iris/ReplicationSystem/ReplicationSystem.h"
// ...
#endif

// RepLayout.h:148-151 — Iris 特有的 Fragment 注册
#if UE_WITH_IRIS
    UE::Net::CreateAndRegisterReplicationFragmentFunc
        CreateAndRegisterReplicationFragmentFunction = nullptr;
#endif
```

### 15.3 Iris 关键概念

| 概念 | 说明 |
|------|------|
| `UReplicationSystem` | Iris 的顶层复制系统管理器 |
| `ReplicationFragment` | 每个属性或属性组的复制片段 |
| `ReplicationProtocol` | 连接间的复制协议定义 |
| `NetRefHandle` | Iris 中的网络对象句柄 |
| `FNetSerializer` | Iris 的属性序列化器 |

Iris 的目标是提供：
- 更精确的脏数据追踪
- 更好的带宽管理
- 支持大规模多人游戏场景
- 简化的游戏代码接口

> **注意**: 在 5.5 版本中，Iris 仍在实验阶段。游戏应同时支持传统路径以确保兼容性。

---

## 16. 总结与最佳实践

### 16.1 架构关键设计

1. **FRepLayout 是共享的类型级元数据** — 每种类型一份，避免重复
2. **FObjectReplicator 是连接级的实例复制器** — 管理每个连接上的复制状态
3. **Changelist 是增量复制的核心** — 只发送变化的属性，极大节省带宽
4. **Condition 系统在 Changelist 层面过滤** — 不满足条件的变更被暂存而非丢弃
5. **Shadow Buffer 缓存上帧状态** — 用于比较检测变更
6. **Push Model 跳过逐属性比较** — 适用于高频变化的属性

### 16.2 性能优化建议

| 优化 | 说明 |
|------|------|
| **使用 Push Model** | 对高频变化的 Actor（如角色移动），减少 Dirty 检查开销 |
| **合理设置 NetUpdateFrequency** | 远处 Actor 降低更新频率 |
| **使用 ReplicationGraph** | 大规模多人游戏场景，远超默认 ReplicationDriver 的性能 |
| **避免不必要的 RepNotify** | 默认 `REPNOTIFY_OnChanged`，仅在需要时使用 `REPNOTIFY_Always` |
| **善用 Dormancy** | 远距离或不活跃的 Actor 进入休眠，降低 CPU 和带宽 |
| **善用复制条件** | `COND_OwnerOnly` / `COND_SkipOwner` 减少不必要的数据发送 |
| **使用 FastArraySerializer** | 对于 TArray 属性，使用 FastArray 减少完整数组传输 |
| **启用共享序列化** | `net.ShareSerializedData=1` (默认启用)，多个连接共享序列化结果 |

### 16.3 关键控制台变量

| 变量 | 说明 |
|------|------|
| `net.ShareSerializedData` | 共享序列化数据 (默认 1) |
| `net.UsePackedShadowBuffers` | 使用紧凑 Shadow Buffer (默认 1) |
| `net.ShareShadowState` | 跨连接共享 Shadow State 比较 (默认 1) |
| `net.PartialBunchReliableThreshold` | 超过此分包数则升级为可靠 (默认 8) |
| `net.PushModelValidateProperties` | 验证 Push Model 属性标记 (调试) |
| `net.DoPropertyChecksum` | 比较校验和 (调试) |
| `net.DeltaInitialFastArrayElements` | FastArray 初始 Delta 发送 |
| `net.TrackNetSerializeObjectReferences` | 追踪 NetSerialize 对象引用 |

### 16.4 核心类关系图

```
UNetDriver (1个服务器, 1个客户端)
  ├─→ UNetConnection[] (每个客户端一个)
  │     ├─→ UChannel[] (每个连接一组信道)
  │     │     ├─→ UControlChannel (ChIndex=0, 握手)
  │     │     └─→ UActorChannel[] (每个复制 Actor 一个)
  │     │           ├─→ FObjectReplicator (ActorReplicator)
  │     │           │     ├─→ TSharedPtr<FRepLayout> (类型级, 共享)
  │     │           │     ├─→ TUniquePtr<FRepState>
  │     │           │     │     ├─→ FSendingRepState (服务器)
  │     │           │     │     └─→ FReceivingRepState (客户端)
  │     │           │     └─→ TSharedPtr<FReplicationChangelistMgr>
  │     │           │           └─→ FRepChangelistState
  │     │           │                 └─→ FRepStateStaticBuffer (Shadow Buffer)
  │     │           ├─→ TMap<UObject*, FObjectReplicator> (子对象复制器)
  │     │           └─→ AActor* Actor
  │     └─→ UNetConnection::OwningActor (PlayerController)
  │
  └─→ UReplicationDriver* ReplicationDriver (可选)
        └─→ UReplicationGraph (基于图的实现)
              ├─→ UReplicationGraphNode::AlwaysRelevant
              ├─→ UReplicationGraphNode::Spatialize
              └─→ UReplicationGraphNode::PerConnection
```

---

*本文档基于 UE 5.5 引擎源代码分析。核心文件位于 `Engine/Source/Runtime/Engine/` 下的 `Public/Net/` 和 `Private/` 目录。*
