# UE5 GamePlay 框架概述

> 本文档基于 Unreal Engine 5.5 源码分析，全面介绍 UE5 游戏玩法框架的核心类型、生命周期、用法、网络架构等。

---

## 1. 核心类型与类图

### 1.1 类继承层次结构

```
UObject                              (CoreUObject)
├── UActorComponent                   (Engine)  ── 可复用的行为组件
│   ├── USceneComponent               (Engine)  ── 具有 Transform 的组件
│   │   ├── UPrimitiveComponent       (Engine)  ── 可渲染、可碰撞的图元
│   │   │   ├── UCapsuleComponent     (Engine)
│   │   │   ├── UBoxComponent         (Engine)
│   │   │   ├── USphereComponent      (Engine)
│   │   │   ├── USkeletalMeshComponent(Engine)
│   │   │   └── UStaticMeshComponent  (Engine)
│   │   ├── USpringArmComponent       (Engine)  ── 弹簧臂(相机延迟跟随)
│   │   └── UCameraComponent          (Engine)  ── 相机
│   ├── UMovementComponent            (Engine)  ── 移动基类
│   │   ├── UNavMovementComponent     (Engine)  ── 导航移动
│   │   │   └── UPawnMovementComponent(Engine)
│   │   │       └── UCharacterMovementComponent (Engine) ── 角色移动
│   ├── UInputComponent               (Engine)  ── 输入绑定
│   │   └── UEnhancedInputComponent   (EnhancedInput) ── 增强输入
│   ├── UAudioComponent               (Engine)  ── 音频
│   ├── UNiagaraComponent             (Niagara) ── 粒子特效
│   └── UChildActorComponent          (Engine)  ── 子Actor
│
├── AActor                            (Engine)  ── 可放入关卡的对象基类
│   ├── AInfo                         (Engine)  ── 不可放置的纯信息类
│   │   ├── AGameModeBase             (Engine)  ── 游戏模式(仅服务器)
│   │   │   └── AGameMode             (Engine)  ── 扩展GameMode(含Match状态)
│   │   ├── AGameStateBase            (Engine)  ── 游戏状态(复制到所有客户端)
│   │   │   └── AGameState            (Engine)
│   │   ├── APlayerState              (Engine)  ── 玩家状态(复制到所有客户端)
│   │   └── AGameSession              (Engine)  ── 游戏会话(在线子系统)
│   ├── APawn                          (Engine)  ── 可被控制的Pawn
│   │   ├── ACharacter                (Engine)  ── 带骨骼网格、碰撞、移动的角色
│   │   └── ASpectatorPawn            (Engine)  ── 观战Pawn
│   ├── AController                    (Engine)  ── 控制器(非物理)
│   │   ├── APlayerController         (Engine)  ── 人类玩家控制器
│   │   └── AAIController             (AIModule) ── AI控制器
│   ├── AHUD                           (Engine)  ── 平视显示器(HUD)
│   ├── APlayerCameraManager           (Engine)  ── 玩家相机管理器
│   └── AWorldSettings                (Engine)  ── 世界设置
│
├── UPlayer                           (Engine)  ── 表示一个真实玩家
│   └── ULocalPlayer                  (Engine)  ── 本地(非远程)玩家，管理视口和输入
├── UWorld                            (Engine)  ── 世界(包含所有关卡和Actor)
├── ULevel                            (Engine)  ── 关卡(世界的子分区)
├── UGameInstance                     (Engine)  ── 游戏实例(跨关卡持久存在)
└── UEngine                           (Engine)  ── 引擎本身(全局单例 GEngine)
```

**接口继承关系**：
| 类 | 实现的接口 |
|-----|-----------|
| `APawn` | `INavAgentInterface` |
| `AController` | `INavAgentInterface` |
| `AAIController` | `IAIPerceptionListenerInterface`, `IGameplayTaskOwnerInterface`, `IGenericTeamAgentInterface`, `IVisualLoggerDebugSnapshotInterface` |
| `APlayerController` | `IWorldPartitionStreamingSourceProvider` |
| `UCharacterMovementComponent` | `IRVOAvoidanceInterface`, `INetworkPredictionInterface` |
| `UWorld` | `FNetworkNotify` |
| `UGameInstance` / `UPlayer` / `UEngine` | `FExec` (控制台命令执行)
```

### 1.2 核心类关系图

```
                        ┌─────────────┐
                        │ UGameInstance│  (跨关卡持久存在，管理游戏生命周期)
                        └──────┬──────┘
                               │ 拥有
                  ┌────────────┼────────────┐
                  ▼            ▼            ▼
            ┌─────────┐  ┌─────────┐  ┌──────────┐
            │ UWorld  │  │UWorld   │  │ UWorld   │  (每个地图一个World)
            └────┬────┘  └────┬────┘  └────┬─────┘
                 │             │            │
       ┌─────────┼─────────┐   │            │
       ▼         ▼         ▼   │            │
    ┌──────┐ ┌──────┐ ┌──────┐ │
    │ULevel│ │ULevel│ │ULevel│ │
    └───┬──┘ └───┬──┘ └───┬──┘ │
        │        │        │     │
        ▼        ▼        ▼     │
   (Actors)  (Actors) (Actors)  │
                                │
     ┌──────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│                   Game Mode (Server Only)        │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ AGameModeBase│  │ AGameSession │             │
│  └──────┬───────┘  └──────────────┘             │
│         │ 创建/管理                               │
│         ▼                                        │
│  ┌──────────────┐  ┌──────────────┐             │
│  │AGameStateBase│  │  APlayerState│ (每个玩家)   │
│  └──────────────┘  └──────┬───────┘             │
│                           │                      │
└───────────────────────────┼──────────────────────┘
                            │
     ┌──────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│           Controllers & Pawns                 │
│                                               │
│  ┌──────────────────┐    ┌──────────────┐    │
│  │ APlayerController│    │AAIController │    │
│  │  (玩家输入)       │    │  (行为树/AI) │    │
│  └────────┬─────────┘    └──────┬───────┘    │
│           │ Possess             │ Possess     │
│           ▼                     ▼             │
│  ┌──────────────────────────────────────┐    │
│  │           APawn / ACharacter         │    │
│  │  ┌──────────────┐ ┌────────────────┐ │    │
│  │  │UCapsuleComp  │ │USkeletalMesh   │ │    │
│  │  │(碰撞)        │ │(骨骼网格)       │ │    │
│  │  ├──────────────┤ ├────────────────┤ │    │
│  │  │UCharMoveComp │ │...更多组件      │ │    │
│  │  └──────────────┘ └────────────────┘ │    │
│  └──────────────────────────────────────┘    │
│                                               │
│  ┌──────────────┐                             │
│  │    AHUD      │  (每个PlayerController一个) │
│  └──────────────┘                             │
│                                               │
│  ┌──────────────────────┐                     │
│  │APlayerCameraManager  │ (相机管理)          │
│  └──────────────────────┘                     │
└──────────────────────────────────────────────┘
```

### 1.3 各模块文件位置

| 类 | 路径 |
|---|---|
| `UObject` | `CoreUObject/Public/UObject/Object.h` |
| `AActor` | `Engine/Classes/GameFramework/Actor.h` |
| `UActorComponent` | `Engine/Classes/Components/ActorComponent.h` |
| `USceneComponent` | `Engine/Classes/Components/SceneComponent.h` |
| `UPrimitiveComponent` | `Engine/Classes/Components/PrimitiveComponent.h` |
| `APawn` | `Engine/Classes/GameFramework/Pawn.h` |
| `ACharacter` | `Engine/Classes/GameFramework/Character.h` |
| `AController` | `Engine/Classes/GameFramework/Controller.h` |
| `APlayerController` | `Engine/Classes/GameFramework/PlayerController.h` |
| `AAIController` | `AIModule/Classes/AIController.h` |
| `AGameModeBase` | `Engine/Classes/GameFramework/GameModeBase.h` |
| `AGameStateBase` | `Engine/Classes/GameFramework/GameStateBase.h` |
| `APlayerState` | `Engine/Classes/GameFramework/PlayerState.h` |
| `AHUD` | `Engine/Classes/GameFramework/HUD.h` |
| `UWorld` | `Engine/Classes/Engine/World.h` |
| `UGameInstance` | `Engine/Classes/Engine/GameInstance.h` |
| `UCharacterMovementComponent` | `Engine/Classes/GameFramework/CharacterMovementComponent.h` |

---

## 2. 每个类型的生命周期

### 2.1 整体游戏生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                    游戏启动流程                                │
├──────────────────────────────────────────────────────────────┤
│  1. UEngine::Init()                                          │
│  2. UGameEngine::Init()                                      │
│  3. UGameInstance::Init()                                    │
│  4. UGameEngine::LoadMap() ─ 加载地图(创建UWorld)             │
│  5. UWorld::InitWorld()                                      │
│  6. AGameModeBase::InitGame() ─ 初始化游戏模式(仅服务器)       │
│  7. Spawn GameMode, GameState, GameSession                   │
│  8. AGameModeBase::StartPlay() ─ 开始游戏                     │
│  9. Actor 初始化流程 (见2.2)                                  │
│ 10. Tick 循环 ─ 每帧更新                                      │
│ 11. 游戏结束 → 地图切换 / 退出                                │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 AActor 生命周期（核心）

这是 UE5 中最关键的生命周期，官方注释来自 `Actor.h:201-231`：

```
┌──────────────────────────────────────────────────────────────┐
│              AActor 完整生命周期流程                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────┐                                     │
│  │ 1. UObject::PostLoad│  关卡中静态放置的Actor                 │
│  │    (仅静态放置)       │  编辑器和运行时都会调用               │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 2. OnComponentCreated│ Actor生成时，原生组件的调用            │
│  │    (原生组件)         │  蓝图组件在构造时调用                 │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌──────────────────────────┐                                │
│  │ 3. PreRegisterAllComponents│  组件注册前                     │
│  └─────────┬────────────────┘                                │
│            ▼                                                  │
│  ┌──────────────────────────┐                                │
│  │ 4. RegisterComponent     │  为每个组件创建物理/视觉表示       │
│  │    (每个组件)              │  可能分布在多帧中                │
│  └─────────┬────────────────┘                                │
│            ▼                                                  │
│  ┌───────────────────────────┐                               │
│  │ 5. PostRegisterAllComponents│ 所有组件注册完成后的回调        │
│  └─────────┬─────────────────┘                               │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 6. PostActorCreated │  编辑器/游戏运行时创建都会调用         │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌──────────────────────────┐                                │
│  │ 7. UserConstructionScript│  蓝图构造脚本(蓝图Actor)         │
│  │ 8. OnConstruction        │  C++构造逻辑                    │
│  └─────────┬────────────────┘                                │
│            ▼                                                  │
│  ┌─────────────────────────────┐                             │
│  │ 9. PreInitializeComponents  │  组件初始化前                  │
│  └─────────┬───────────────────┘                             │
│            ▼                                                  │
│  ┌─────────────────────────────┐                             │
│  │ 10. InitializeComponent     │  仅当 bWantsInitialize=true │
│  │     Activate (bAutoActivate)│  仅当 bAutoActivate=true    │
│  └─────────┬───────────────────┘                             │
│            ▼                                                  │
│  ┌──────────────────────────────┐                            │
│  │ 11. PostInitializeComponents │  所有组件初始化完成           │
│  └─────────┬────────────────────┘                            │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 12. BeginPlay       │  关卡开始Tick (实际游戏开始)          │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 13. Tick (每帧)      │  每帧调用 TickActor()              │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 14. EndPlay         │  游戏结束/Actor销毁时调用             │
│  │     (EEndPlayReason) │                                     │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌─────────────────────┐                                     │
│  │ 15. Destroyed       │  Actor被标记为销毁                   │
│  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────┘
```

**EndPlay 原因枚举 (`EEndPlayReason`)**：
- `Destroyed` — 被显式销毁
- `LevelTransition` — 关卡切换
- `EndPlayInEditor` — 编辑器结束Play
- `RemovedFromWorld` — 从世界中移除
- `Quit` — 退出游戏

### 2.3 UActorComponent 生命周期

```
创建 → OnComponentCreated → RegisterComponent → 
CreateRenderState_Concurrent → CreatePhysicsState →
InitializeComponent → Activate →
TickComponent (每帧) →
Deactivate → DestroyPhysicsState → DestroyRenderState_Concurrent →
UnregisterComponent → OnComponentDestroyed → 销毁
```

关键标记位：
- `bRegistered` — 是否已注册到场景
- `bRenderStateCreated` — 渲染状态是否已创建
- `bPhysicsStateCreated` — 物理状态是否已创建
- `bWantsInitializeComponent` — 是否需要调用 InitializeComponent
- `bAutoActivate` — 注册后是否自动激活

### 2.4 APlayerController 生命周期

```
创建 → PostInitializeComponents → 
(客户端) ReceivedPlayer → 
(服务器) Possess(Pawn) → BeginPlay →
Tick (处理输入 → 移动Pawn → 更新相机) →
(服务器) UnPossess → EndPlay → 销毁
```

**关键流程**：
1. 服务器上 `AGameModeBase::Login()` 创建 PlayerController
2. 服务器调用 `Possess()` 控制 Pawn
3. 服务器将 PC 复制给客户端
4. 客户端 `ReceivedPlayer()` 被调用
5. `BeginPlay` 开始
6. 每帧处理输入、更新相机、Tick

### 2.5 AGameModeBase 生命周期

```
引擎启动 → UGameEngine::LoadMap() →
(服务器) 根据 World Settings / URL ?game=xxx 确定 GameMode 类 →
InitGame() → (Spawn GameState, GameSession) →
(玩家登录时) PreLogin() → Login() → PostLogin() →
StartPlay() → (触发所有 Actor 的 BeginPlay) →
(游戏结束) → 切换地图或退出
```

### 2.6 AGameStateBase 生命周期

```
GameMode::InitGame() → Spawn GameState → InitGameState() →
PostInitializeComponents() → BeginPlay() →
HandleBeginPlay() → bReplicatedHasBegunPlay = true →
(复制到客户端) → OnRep_ReplicatedHasBegunPlay() →
客户端 BeginPlay/StartMatch
```

### 2.7 UGameInstance 生命周期

```
引擎启动 → UGameInstance::Init() →
(跨关卡持久存在，不会被销毁) →
加载地图 → 创建/切换 World →
(本地玩家) CreateLocalPlayer() →
游戏结束 → UGameInstance::Shutdown()
```

---

## 3. 每个类型的作用与用法

### 3.1 UObject
- **作用**：UE5 中几乎所有对象的基类。提供 GC（垃圾回收）、反射（Reflection）、序列化、网络复制等核心功能。
- **继承关系**：所有以 `U` 前缀开头的类。
- **头文件**：`CoreUObject/Public/UObject/Object.h`
- **关键宏**：
  - `UCLASS()` — 声明一个 UObject 类
  - `UPROPERTY()` — 声明需要反射/GC/序列化的属性
  - `UFUNCTION()` — 声明需要反射的函数（BlueprintCallable, RPC 等）
  - `GENERATED_BODY()` — 生成必要的样板代码

### 3.2 AActor
- **作用**：可以被放入关卡或动态生成的一切对象的基类。Actor 本身不含 Transform，需要 RootComponent 提供空间位置。
- **如何创建**：
  ```cpp
  // 动态生成
  AActor* MyActor = GetWorld()->SpawnActor<AMyActor>(AMyActor::StaticClass(), SpawnLocation, SpawnRotation);
  
  // 或生成延迟生成
  AActor* MyActor = GetWorld()->SpawnActorDeferred<AMyActor>(AMyActor::StaticClass(), SpawnTransform);
  // ... 设置属性 ...
  MyActor->FinishSpawning(SpawnTransform);
  ```
- **核心概念**：
  - 每个 Actor 有一个 `RootComponent`（第一个 USceneComponent）
  - Actor 可以包含多个 Component
  - Actor 负责网络复制和属性同步

### 3.3 UActorComponent
- **作用**：可复用的行为组件，可以添加到任意 Actor。是 Component 的抽象基类。
- **用法**：
  - C++ 中声明为成员变量并通过 `CreateDefaultSubobject<>` 创建
  - 或通过蓝图动态添加
  - 蓝图中通过 `Add Component` 节点添加
- 没有 Transform（位置/旋转/缩放）

### 3.4 USceneComponent
- **作用**：拥有 Transform（位置、旋转、缩放）的 Component，支持父子层级附着。
- **继承**：`UActorComponent` → `USceneComponent`
- **关键功能**：
  - 提供 `RelativeLocation`, `RelativeRotation`, `RelativeScale3D`
  - 支持附着到其他 SceneComponent (`AttachToComponent`)
  - 每个 Actor 必须至少有一个 SceneComponent 作为 RootComponent

### 3.5 APawn
- **作用**：可以被 Controller 控制的 Actor。是所有玩家和 AI 实体的基类。
- **关键特性**：
  - 可以被 `AController::Possess()` 控制
  - 可以接收输入（当被 PlayerController 控制时）
  - 有内置的移动组件 (`UPawnMovementComponent`)
- **重要属性**：
  - `bUseControllerRotationYaw/Pitch/Roll` — 是否使用 Controller 的旋转
  - `AutoPossessPlayer` — 自动被哪个 PlayerController 控制
  - `AIControllerClass` — AI 控制时使用的 Controller 类
  - `BaseEyeHeight` — 眼睛高度（用于相机）

### 3.6 ACharacter
- **作用**：专门为双足行走角色设计的 Pawn 子类，内置碰撞胶囊体、骨骼网格和角色移动组件。
- **内置组件**：
  | 组件 | 类型 | 作用 |
  |------|------|------|
  | `CapsuleComponent` | `UCapsuleComponent` | 碰撞和移动体积 |
  | `Mesh` | `USkeletalMeshComponent` | 骨骼网格渲染 |
  | `CharacterMovement` | `UCharacterMovementComponent` | 移动逻辑（行走、跳跃、飞行、游泳） |
- **移动模式 (EMovementMode)**：`MOVE_None`, `MOVE_Walking`, `MOVE_NavWalking`, `MOVE_Falling`, `MOVE_Swimming`, `MOVE_Flying`, `MOVE_Custom`
- **用法**：大多数游戏中的玩家角色和 NPC 都应该从 ACharacter 派生

### 3.7 AController
- **作用**：非物理 Actor，控制 Pawn 的行为。抽象基类。
- **核心概念**：
  - `ControlRotation` — 控制旋转（决定瞄准/视角方向）
  - `Possess(APawn*)` — 接管 Pawn 的控制权
  - `UnPossess()` — 放弃控制权
  - 包含 `TransformComponent`（USceneComponent），可提供 Transform
- 在网络游戏中，Controller 只存在于服务器（PC 在客户端也存在本地副本）

### 3.8 APlayerController
- **作用**：人类玩家使用的 Controller。
- **核心职责**：
  - 处理玩家输入（键盘、鼠标、手柄）
  - 管理相机 (`PlayerCameraManager`)
  - 管理 HUD (`MyHUD`)
  - 网络通信 (RPC)
- **关键成员**：
  - `PlayerCameraManager` — 相机管理器
  - `MyHUD` — 平视显示器
  - `PlayerInput` — 输入管理器
  - `NetConnection` — 网络连接
- **输入模式**：`FInputModeGameOnly`, `FInputModeUIOnly`, `FInputModeGameAndUI`
- **网络角色**：每个客户端上只存在属于本地玩家的 PC；服务器上存在所有玩家的 PC。

### 3.9 AAIController
- **作用**：AI 控制的 Controller。只在服务器上存在。
- **核心组件**：
  - `BrainComponent` — AI 大脑（行为树或脚本）
  - `UAIPerceptionComponent` — AI 感知（视觉、听觉等）
  - `UPathFollowingComponent` — 路径跟随
  - `UBlackboardComponent` — 黑板（共享数据）
- **关键函数**：
  - `MoveToLocation()` / `MoveToActor()` — 移动到目标
  - `RunBehaviorTree()` — 启动行为树
  - `SetFocus()` — 设置焦点目标
  - `K2_ClearFocus()` — 清除焦点

### 3.10 AGameModeBase / AGameMode
- **作用**：定义游戏规则。**只在服务器上存在**，客户端为 NULL。
- **确定使用哪个 GameMode 类的优先级**：
  1. URL 参数 `?game=xxx`
  2. World Settings 中的 GameMode Override
  3. Project Settings 中的 Default GameMode
- **声明游戏使用的类**：
  ```cpp
  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Classes)
  TSubclassOf<APawn> DefaultPawnClass;       // 默认Pawn类
  TSubclassOf<APlayerController> PlayerControllerClass;
  TSubclassOf<APlayerState> PlayerStateClass;
  TSubclassOf<AGameStateBase> GameStateClass;
  TSubclassOf<AHUD> HUDClass;
  TSubclassOf<AGameSession> GameSessionClass;
  ```
- **关键虚函数**：
  | 函数 | 描述 |
  |------|------|
  | `InitGame()` | 最早调用的初始化函数 |
  | `PreLogin()` | 玩家登录前（可以拒绝） |
  | `Login()` | 玩家正式登录，创建 PlayerController |
  | `PostLogin()` | 登录后的处理 |
  | `StartPlay()` | 开始游戏（触发 BeginPlay） |
  | `Logout()` | 玩家退出 |
  | `RestartPlayer()` | 重生玩家 |
  | `GetDefaultPawnClassForController()` | 获取默认 Pawn 类 |
  | `SetPause()` / `ClearPause()` | 暂停/恢复游戏 |
- **AGameMode 扩展**（相对于 AGameModeBase）：添加了 Match 状态机。

**MatchState 状态流转**：
```
EnteringMap → WaitingToStart → InProgress → WaitingPostMatch → LeavingMap
                                         ↘ Aborted (网络故障等)
```

| 状态 | 说明 | 处理函数 |
|------|------|----------|
| `EnteringMap` | 正在进入地图，Actor 尚未 Tick | — |
| `WaitingToStart` | Actor 已在 Tick，等待玩家就绪 | `HandleMatchIsWaitingToStart()` |
| `InProgress` | 正常游戏进行中 | `HandleMatchHasStarted()` |
| `WaitingPostMatch` | 比赛结束，不接受新玩家 | `HandleMatchHasEnded()` |
| `LeavingMap` | 正在离开地图 | `HandleLeavingMap()` |
| `Aborted` | 网络或其他故障中止 | `HandleMatchAborted()` |

AGameMode 关键成员：
| 变量 | 说明 |
|------|------|
| `MatchState` (FName) | 当前比赛状态，复制到客户端 |
| `bDelayedStart` | 等待玩家就绪后再开始 |
| `MinRespawnDelay` (float) | 最小重生延迟 |
| `InactivePlayerArray` | 已断开连接的玩家 PlayerState 列表 |

AGameMode 关键虚函数：
| 函数 | 说明 |
|------|------|
| `StartMatch()` | WaitingToStart → InProgress |
| `EndMatch()` | InProgress → WaitingPostMatch |
| `RestartGame()` | 通过服务器旅行重启 |
| `AbortMatch()` | 不可恢复错误时中止 |
| `ReadyToStartMatch()` | BlueprintNativeEvent，条件满足时开始 |
| `ReadyToEndMatch()` | BlueprintNativeEvent，条件满足时结束 |

### 3.11 AGameStateBase / AGameState
- **作用**：管理游戏的全局状态，**复制到所有客户端**。
- **关键成员**：
  - `PlayerArray` — 所有 PlayerState 的数组
  - `GameModeClass` — 服务器的 GameMode 类（复制到客户端）
  - `AuthorityGameMode` — GameMode 实例（仅服务器非 NULL）
  - `bReplicatedHasBegunPlay` — 游戏是否已开始的复制标志
- **关键函数**：
  - `AddPlayerState()` / `RemovePlayerState()` — 管理 PlayerState
  - `GetPlayerStateFromUniqueNetId()` — 根据唯一 ID 查找 PlayerState
  - `GetServerWorldTimeSeconds()` — 获取同步的服务器时间

### 3.12 APlayerState
- **作用**：每个玩家的状态信息，复制到所有客户端。
- **核心属性**：
  - `PlayerName` — 玩家名称
  - `Score` — 分数
  - `PlayerId` — 玩家 ID
  - `UniqueId` — 网络唯一 ID（由在线子系统管理）
  - `bIsSpectator` — 是否为观战者
  - `bIsABot` — 是否为 AI
  - `Ping` — 网络延迟（压缩后复制）
  - `PawnPrivate` — 关联的 Pawn
- **生命周期**：由 GameMode 在玩家登录时创建，可以与 Controller 分离持久存在。

### 3.13 AHUD
- **作用**：平视显示器（Heads-Up Display），每个 PlayerController 拥有一个。
- **关键成员**：
  - `PlayerOwner` — 拥有此 HUD 的 PlayerController
  - `Canvas` — 用于绘制的画布
- **绘制方式**：
  ```cpp
  // 蓝图中重写 ReceiveDrawHUD
  // C++ 中重写 DrawHUD
  void AMyHUD::DrawHUD()
  {
      Super::DrawHUD();
      // 使用 Canvas 绘制文字、材质、矩形等
      DrawText(TEXT("Hello"), FLinearColor::White, 100, 100, nullptr);
  }
  ```

### 3.14 UWorld
- **作用**：游戏世界的顶层容器，包含所有关卡和 Actor。
- **关键成员**：
  - `PersistentLevel` — 持久关卡
  - `Levels` — 所有关卡（持久+流式加载）
  - `GameMode` / `GameState` — 游戏模式/状态
  - `NetDriver` — 网络驱动
- **迭代 Actor**：
  ```cpp
  for (TActorIterator<AEnemy> It(GetWorld()); It; ++It) { ... }
  for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It) { ... }
  ```
- **Spawn Actor**：
  ```cpp
  FActorSpawnParameters Params;
  Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
  GetWorld()->SpawnActor<AMyActor>(Class, Location, Rotation, Params);
  ```

### 3.15 UGameInstance
- **作用**：游戏实例，从引擎启动到退出一直存在，跨关卡切换持久化。
- **关键成员**：
  - `LocalPlayers` — 本地玩家列表
  - `OnlineSession` — 在线服务管理器
  - `ReferencedObjects` — 被保持存活的对象
- **关键函数**：
  - `Init()` / `Shutdown()` — 初始化和关闭
  - `StartGameInstance()` — 启动游戏实例流程
  - `HandleTravelCommand()` / `HandleDisconnectCommand()` / `HandleReconnectCommand()` — 控制台命令
- **拥有子系统**：
  - `UGameInstanceSubsystem` — 游戏实例级子系统（生命周期最长）

### 3.16 UPlayer / ULocalPlayer
- **UPlayer**：表示一个真实玩家。包含 `PlayerController` 引用和 `CurrentNetSpeed`。
- **ULocalPlayer**（UPlayer 子类）：本地（非远程）玩家。
  - `ViewportClient` — 主视口
  - `CachedUniqueNetId` — 平台唯一标识
  - `Origin` / `Size` — 视口子区域（分屏用）
  - `ControllerId` — 输入控制器 ID
  - `PlatformUserId` — 平台用户 ID
  - `OnControllerIdChanged` / `OnPlatformUserIdChanged` / `OnPlayerControllerChanged` 委托
  - `GetGameInstance()` / `GetWorld()` / `GetLocalPlayer()` 等访问函数

### 3.17 APlayerCameraManager
- **作用**：管理玩家相机，由 PlayerController 拥有。
- **关键功能**：
  - 控制相机视角
  - 管理 ViewTarget（通常是控制的 Pawn）
  - 支持相机混合（Camera Blend）和平滑过渡
- **关键函数**：
  - `SetViewTarget()` — 设置相机目标
  - `GetCameraViewPoint()` — 获取相机位置和方向

---

## 4. 每个类型的重要成员函数和变量

### 4.1 AActor 重要成员

#### 关键成员函数

| 函数 | 签名 | 说明 |
|------|------|------|
| `BeginPlay` | `virtual void BeginPlay()` | Actor 开始游戏时调用 |
| `Tick` | `virtual void Tick(float DeltaTime)` | 每帧更新 |
| `EndPlay` | `virtual void EndPlay(const EEndPlayReason::Type EndPlayReason)` | Actor 生命周期结束时调用 |
| `Destroy` | `bool Destroy(bool bNetForce=false, bool bShouldModifyLevel=false)` | 销毁 Actor |
| `SetActorLocation` | `bool SetActorLocation(FVector, bool bSweep, FHitResult*)` | 设置位置 |
| `SetActorRotation` | `bool SetActorRotation(FRotator, ETeleportType)` | 设置旋转 |
| `SetActorTransform` | `bool SetActorTransform(FTransform, bool bSweep, FHitResult*, ETeleportType)` | 设置 Transform |
| `GetActorLocation` | `FVector GetActorLocation()` | 获取位置 |
| `GetActorRotation` | `FRotator GetActorRotation()` | 获取旋转 |
| `AddActorLocalOffset` | `void AddActorLocalOffset(FVector, bool bSweep, FHitResult*, bool bTeleport)` | 局部坐标偏移 |
| `GetDistanceTo` | `float GetDistanceTo(const AActor*)` | 计算到另一 Actor 的距离 |
| `TakeDamage` | `virtual float TakeDamage(float, FDamageEvent const&, AController*, AActor*)` | 受到伤害 |
| `SetOwner` | `virtual void SetOwner(AActor*)` | 设置所有者 |
| `GetComponents` | `void GetComponents(TArray<T*>&)` | 获取指定类型组件 |
| `FindComponentByClass` | `T* FindComponentByClass<T>()` | 按类型查找组件 |
| `GetLifetimeReplicatedProps` | `virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>&)` | 声明需要复制的属性 |
| `TearOff` | `virtual void TearOff()` | 断开复制（Tear-off） |
| `IsNetRelevantFor` | `virtual bool IsNetRelevantFor(...)` | 是否对某客户端网络相关 |
| `PreReplication` | `virtual void PreReplication(IRepChangedPropertyTracker&)` | 每次复制前调用 |
| `OnConstruction` | `virtual void OnConstruction(const FTransform&)` | 构造/重建时调用 |

#### 关键成员变量 (public/protected)

| 变量 | 类型 | 说明 |
|------|------|------|
| `PrimaryActorTick` | `FActorTickFunction` | 主 Tick 函数配置 |
| `bReplicates` | `uint8:1` | 是否参与网络复制 |
| `bReplicateMovement` | `uint8:1` | 是否复制移动 |
| `bAlwaysRelevant` | `uint8:1` | 对所有客户端总是相关 |
| `bOnlyRelevantToOwner` | `uint8:1` | 仅对 Owner 相关 |
| `bNetTemporary` | `uint8:1` | 发送到客户端后不再更新 |
| `bNetStartup` | `uint8:1` | 是否直接从地图加载 |
| `bTearOff` | `uint8:1` | 断开复制标志 |
| `bCanBeDamaged` | `uint8:1` | 是否可以受到伤害 |
| `bBlockInput` | `uint8:1` | 是否阻止输入 |
| `bHidden` | `uint8:1` | 是否在游戏中隐藏 |
| `bAutoDestroyWhenFinished` | `uint8:1` | 完成后自动销毁 |
| `Owner` | `AActor*` | Actor 的所有者 |
| `Instigator` | `APawn*` | 引起此 Actor 的 Pawn |
| `RemoteRole` | `ENetRole` | 网络角色 |
| `RootComponent` | `USceneComponent*` | 根组件 |
| `Tags` | `TArray<FName>` | 标签数组 |
| `ActorHasTag` | `bool` | 是否有指定标签 |

#### 重要委托 (Delegates)

| 委托 | 签名 | 说明 |
|------|------|------|
| `OnTakeAnyDamage` | `(AActor*, float, UDamageType*, AController*, AActor*)` | 受到任意伤害 |
| `OnTakePointDamage` | `(AActor*, float, AController*, FVector, UPrimitiveComponent*, ...)` | 受到点伤害 |
| `OnActorBeginOverlap` | `(AActor*, AActor*)` | 与另一 Actor 开始重叠 |
| `OnActorEndOverlap` | `(AActor*, AActor*)` | 与另一 Actor 结束重叠 |
| `OnActorHit` | `(AActor*, AActor*, FVector, const FHitResult&)` | 碰撞命中 |
| `OnClicked` | `(AActor*, FKey)` | 被鼠标点击 |
| `OnDestroyed` | `(AActor*)` | 被销毁时 |
| `OnEndPlay` | `(AActor*, EEndPlayReason)` | EndPlay 时 |

### 4.2 UActorComponent 重要成员

#### 关键成员函数

| 函数 | 说明 |
|------|------|
| `InitializeComponent()` | 初始化组件（仅一次） |
| `BeginPlay()` | 组件开始游戏 |
| `TickComponent(float DeltaTime, ELevelTick, FActorComponentTickFunction*)` | 每帧更新 |
| `EndPlay(const EEndPlayReason::Type)` | 组件结束 |
| `RegisterComponent()` | 注册到世界 |
| `UnregisterComponent()` | 从世界注销 |
| `Activate(bool bReset=false)` | 激活组件 |
| `Deactivate()` | 停用组件 |
| `IsActive()` | 是否激活 |
| `GetOwner()` | 获取所属 Actor |
| `GetWorld()` | 获取所属 World |
| `CreateRenderState_Concurrent()` | 创建渲染状态 |
| `DestroyRenderState_Concurrent()` | 销毁渲染状态 |
| `CreatePhysicsState()` | 创建物理状态 |
| `DestroyPhysicsState()` | 销毁物理状态 |
| `OnComponentCreated()` | 组件被创建时 |
| `OnComponentDestroyed(bool bDestroyingHierarchy)` | 组件被销毁时 |
| `GetLifetimeReplicatedProps(...)` | 声明复制属性 |
| `ReplicateSubObjects(...)` | 复制子对象 |

#### 关键成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `PrimaryComponentTick` | `FActorComponentTickFunction` | Tick 配置 |
| `ComponentTags` | `TArray<FName>` | 组件标签 |
| `bRegistered` | `uint8:1` | 是否已注册到场景 |
| `bRenderStateCreated` | `uint8:1` | 渲染状态是否已创建 |
| `bPhysicsStateCreated` | `uint8:1` | 物理状态是否已创建 |
| `bAutoActivate` | `uint8:1` | 注册后自动激活 |
| `bWantsInitializeComponent` | `uint8:1` | 是否需要 InitializeComponent |
| `bReplicates` | `uint8:1` | 是否参与网络复制 |
| `bIsEditorOnly` | `uint8:1` | 是否仅编辑器 |

### 4.3 APlayerController 重要成员函数

| 函数 | 说明 |
|------|------|
| `Possess(APawn*)` | 控制一个 Pawn |
| `UnPossess()` | 放弃控制 Pawn |
| `SetViewTarget(AActor*)` | 设置相机目标 |
| `SetViewTargetWithBlend(AActor*, float, EViewTargetBlendFunction, float, bool)` | 平滑切换相机目标 |
| `SetShowMouseCursor(bool)` | 设置是否显示鼠标 |
| `SetInputMode(FInputModeDataBase)` | 设置输入模式 |
| `AddYawInput(float)` | 添加 Yaw 输入 |
| `AddPitchInput(float)` | 添加 Pitch 输入 |
| `AddRollInput(float)` | 添加 Roll 输入 |
| `GetPlayerViewPoint(FVector&, FRotator&)` | 获取玩家视角位置和方向 |
| `ProjectWorldLocationToScreen(FVector, FVector2D&, bool)` | 世界坐标转屏幕坐标 |
| `GetHitResultUnderCursor(ECollisionChannel, bool, FHitResult&)` | 获取光标下命中结果 |
| `SetPause(bool)` | 设置暂停状态 |
| `ConsoleCommand(FString, bool)` | 执行控制台命令 |
| `ClientTravel(FString, ETravelType, bool, FGuid)` | 客户端切换地图 |
| `ServerRestartPlayer()` | 服务器重生玩家 (RPC) |

### 4.4 ACharacter 重要成员函数

| 函数 | 说明 |
|------|------|
| `Jump()` | 跳跃 |
| `StopJumping()` | 停止跳跃 |
| `Crouch(bool bClientSimulation=false)` | 蹲下 |
| `UnCrouch(bool bClientSimulation=false)` | 站起 |
| `CanJump()` | 是否可以跳跃 |
| `IsJumpProvidingForce()` | 跳跃是否在提供力 |
| `LaunchCharacter(FVector, bool bXYOverride, bool bZOverride)` | 弹射角色 |
| `GetCharacterMovement()` | 获取角色移动组件 |
| `GetMesh()` | 获取骨骼网格组件 |
| `GetCapsuleComponent()` | 获取胶囊体组件 |
| `SetBase(UPrimitiveComponent*, FName, bool)` | 设置站立的基座 |
| `OnRep_ReplicatedBasedMovement()` | 网络复制回调 |
| `CacheInitialMeshOffset(FVector, FRotator)` | 缓存网格偏移 |

### 4.5 AIController 重要成员函数

| 函数 | 说明 |
|------|------|
| `RunBehaviorTree(UBehaviorTree*)` | 运行行为树 |
| `MoveToLocation(FVector, float, bool, bool, bool, bool, ...)` | 移动到指定位置 |
| `MoveToActor(AActor*, float, bool, bool, bool, ...)` | 移动到指定 Actor |
| `StopMovement()` | 停止移动 |
| `GetMoveStatus()` | 获取移动状态 |
| `SetFocus(AActor*, EAIFocusPriority::Type)` | 设置焦点目标 |
| `K2_ClearFocus()` | 清除焦点 |
| `GetAIPerceptionComponent()` | 获取感知组件 |
| `UseBlackboard(UBlackboardData*, UBlackboardComponent*)` | 使用黑板 |
| `K2_SetFocalPoint(FVector)` | 设置焦点位置 |

### 4.6 AGameModeBase 重要成员函数

| 函数 | 说明 |
|------|------|
| `InitGame(FString, FString, FString&)` | 游戏初始化 |
| `StartPlay()` | 开始游戏（触发 BeginPlay） |
| `PreLogin(FString, FString, FUniqueNetIdRepl, FString&)` | 登录前检查 |
| `Login(FString, FString, FUniqueNetIdRepl, FString&)` | 玩家登录 |
| `PostLogin(APlayerController*)` | 登录完成回调 |
| `Logout(AController*)` | 玩家退出 |
| `RestartPlayer(AController*)` | 重生玩家 |
| `RestartPlayerAtPlayerStart(AController*, AActor*)` | 在指定位置重生 |
| `SpawnDefaultPawnFor(AController*, AActor*)` | 为玩家生成默认 Pawn |
| `GetDefaultPawnClassForController(AController*)` | 获取默认 Pawn 类 |
| `ShouldReset(AActor*)` | 是否重置 Actor |
| `HandleSeamlessTravelPlayer(AController*)` | 无缝旅行时处理玩家 |
| `GetSession() → AGameSession*` | 获取游戏会话 |

### 4.7 AGameStateBase 重要成员函数

| 函数 | 说明 |
|------|------|
| `GetServerWorldTimeSeconds()` | 获取同步的服务器时间 |
| `HasBegunPlay()` | 游戏是否已开始 |
| `HasMatchStarted()` | 比赛是否已开始 |
| `HasMatchEnded()` | 比赛是否已结束 |
| `AddPlayerState(APlayerState*)` | 添加玩家状态 |
| `RemovePlayerState(APlayerState*)` | 移除玩家状态 |
| `GetPlayerStateFromUniqueNetId(FUniqueNetIdWrapper)` | 根据 UniqueId 查找 PlayerState |
| `HandleBeginPlay()` | 由 GameMode 调用开始游戏 |

### 4.8 APlayerState 重要成员属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `Score` | `float` | 分数（Replicated） |
| `PlayerId` | `int32` | 玩家 ID（Replicated） |
| `UniqueId` | `FUniqueNetIdRepl` | 网络唯一 ID（Replicated） |
| `PlayerName` | `FString` | 玩家名称（Replicated） |
| `bIsSpectator` | `uint8:1` | 是否观战（Replicated） |
| `bIsABot` | `uint8:1` | 是否 AI（Replicated） |
| `bIsInactive` | `uint8:1` | 是否为不活跃玩家（Replicated） |
| `PawnPrivate` | `APawn*` | 关联的 Pawn（蓝图只读） |
| `ExactPing` | `float` | 精确延迟 |
| `StartTime` | `int32` | 玩家创建时的服务器时间 |

---

## 5. Actor-Component 架构

### 5.1 Component 架构总览

UE5 采用 **对象组合 (Composition)** 而非多重继承的方式，通过 Component 将功能/行为附加到 Actor 上。

```
┌────────────────────────────────────────┐
│              AActor                     │
│  ┌──────────────────────────────────┐  │
│  │  RootComponent (USceneComponent) │  │  ← 必须至少一个
│  │  ┌──────────────────────────┐   │  │
│  │  │  ChildComponent A        │   │  │  ← 嵌套层级
│  │  │  ┌──────────────────┐   │   │  │
│  │  │  │ ChildComponent B │   │   │  │
│  │  │  └──────────────────┘   │   │  │
│  │  └──────────────────────────┘   │  │
│  ├──────────────────────────────────┤  │
│  │  ActorComponent C (无Transform)  │  │  ← 逻辑组件(Input, Audio)
│  ├──────────────────────────────────┤  │
│  │  ActorComponent D               │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

### 5.2 Component 分类

#### 按继承层次分

| 层级 | 类 | 特点 |
|------|-----|------|
| 基类 | `UActorComponent` | 无 Transform，纯逻辑组件 |
| 中间层 | `USceneComponent` | 有 Transform，支持附着层级 |
| 叶子层 | `UPrimitiveComponent` | 可渲染、可碰撞的图元 |

#### 按功能分

| 类型 | 组件 | 说明 |
|------|------|------|
| **物理/碰撞** | `UCapsuleComponent` | 胶囊体（最常用的人物碰撞体） |
| | `UBoxComponent` | 盒体碰撞 |
| | `USphereComponent` | 球体碰撞 |
| **渲染** | `USkeletalMeshComponent` | 骨骼网格（人物、动画） |
| | `UStaticMeshComponent` | 静态网格（场景物体） |
| **移动** | `UCharacterMovementComponent` | 角色移动（行走/跳跃/飞行/游泳） |
| | `UProjectileMovementComponent` | 抛射物移动 |
| | `URotatingMovementComponent` | 旋转移动 |
| **相机** | `UCameraComponent` | 相机 |
| | `USpringArmComponent` | 弹簧臂（相机延迟/碰撞检测） |
| **输入** | `UInputComponent` | 传统输入绑定 |
| | `UEnhancedInputComponent` | 增强输入系统（UE5 推荐） |
| **音频** | `UAudioComponent` | 3D 音效 |
| **特效** | `UNiagaraComponent` | Niagara 粒子系统 |
| **UI** | `UWidgetComponent` | 3D 世界中的 UI Widget |
| **组织** | `UChildActorComponent` | 附加子 Actor |
| | `USceneComponent` | 纯空间变换（用于组织层级） |

### 5.3 Component 的创建方式

#### C++ 中声明为默认子对象
```cpp
// 在构造函数中
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category=Components)
    TObjectPtr<UStaticMeshComponent> MeshComponent;
    
    AMyActor()
    {
        // 创建默认根组件
        MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
        RootComponent = MeshComponent;
        
        // 添加其他组件
        UBoxComponent* Box = CreateDefaultSubobject<UBoxComponent>(TEXT("CollisionBox"));
        Box->SetupAttachment(MeshComponent);
    }
};
```

#### 蓝图中动态添加
- 在蓝图编辑器中使用 `+ Add Component` 按钮
- 或在 Event Graph 中使用 `Add Component` 节点（运行时动态）

#### 关键配置属性
- `bAutoActivate` — 注册时自动激活
- `bWantsInitializeComponent` — 需要 InitializeComponent 调用
- `bTickInEditor` — 编辑器中是否 Tick
- `TickGroup` — Tick 顺序分组

### 5.4 重要 Component 详解

#### 5.4.1 UCharacterMovementComponent
- **路径**：`Engine/Classes/GameFramework/CharacterMovementComponent.h`
- **继承链**：`UMovementComponent → UNavMovementComponent → UPawnMovementComponent → UCharacterMovementComponent`
- **实现的接口**：`IRVOAvoidanceInterface`, `INetworkPredictionInterface`
- **核心功能**：
  - 处理角色物理移动（行走、跳跃、下落、飞行、游泳）
  - 网络复制移动状态和预测
  - 支持移动模式切换
  - 步高、坡度限制、自定义重力方向
- **关键属性**：
  | 属性 | 类型 | 说明 |
  |------|------|------|
  | `MaxWalkSpeed` | `float` | 最大行走速度 |
  | `MaxWalkSpeedCrouched` | `float` | 蹲下时最大行走速度 |
  | `MaxFlySpeed` | `float` | 最大飞行速度 |
  | `MaxSwimSpeed` | `float` | 最大游泳速度 |
  | `MaxAcceleration` | `float` | 最大加速度 |
  | `JumpZVelocity` | `float` | 跳跃初速度 |
  | `AirControl` | `float` | 空中控制力 (0-1) |
  | `GravityScale` | `float` | 重力缩放 |
  | `GravityDirection` | `FVector` | 自定义重力方向 |
  | `Mass` | `float` | 质量 |
  | `GroundFriction` | `float` | 地面摩擦力 |
  | `BrakingDecelerationWalking` | `float` | 行走减速 |
  | `MaxStepHeight` | `float` | 最大上台阶高度 |
  | `WalkableFloorAngle` | `float` | 可行走地板最大角度 |
  | `NetworkSmoothingMode` | `ENetworkSmoothingMode` | 模拟代理平滑模式 |
  | `MovementMode` | `EMovementMode` | 当前移动模式 |
- **关键函数**：
  | 函数 | 说明 |
  |------|------|
  | `DoJump(bool bReplayingMoves)` | 执行跳跃 |
  | `SetMovementMode(EMovementMode, uint8)` | 设置移动模式 |
  | `IsWalking()` | 是否在行走 |
  | `IsFalling()` | 是否在下落 |
  | `IsFlying()` | 是否在飞行 |
  | `IsSwimming()` | 是否在游泳 |
  | `CalcVelocity(float, float, bool, float)` | 计算速度（处理加速度和摩擦） |
  | `AddImpulse(FVector, bool)` | 添加冲量 |

#### 5.4.2 USpringArmComponent — 弹簧臂（相机延迟与碰撞检测）
- **路径**：`Engine/Classes/GameFramework/SpringArmComponent.h`
- **继承**：`USceneComponent`
- **作用**：实现相机延迟、碰撞检测的"弹簧臂"，常用于第三人称相机。
- **典型用法**：
  ```cpp
  // 在 Character 构造中
  SpringArm = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArm"));
  SpringArm->SetupAttachment(RootComponent);
  SpringArm->TargetArmLength = 400.0f;      // 相机距离
  SpringArm->bEnableCameraLag = true;        // 位置延迟
  SpringArm->bEnableCameraRotationLag = true; // 旋转延迟
  SpringArm->bDoCollisionTest = true;         // 碰撞检测
  SpringArm->ProbeSize = 12.0f;              // 碰撞探测大小
  SpringArm->bUsePawnControlRotation = true; // 跟随 Pawn 控制旋转

  Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
  Camera->SetupAttachment(SpringArm, USpringArmComponent::SocketName);
  ```
- **关键属性**：
  | 属性 | 类型 | 说明 |
  |------|------|------|
  | `TargetArmLength` | `float` | 目标臂长（相机距离） |
  | `SocketOffset` | `FVector` | 臂末端偏移 |
  | `TargetOffset` | `FVector` | 世界空间起始偏移 |
  | `ProbeSize` / `ProbeChannel` | `float` / `ECollisionChannel` | 碰撞探测设置 |
  | `bDoCollisionTest` | `bool` | 是否启用碰撞检测（相机穿透时自动拉近） |
  | `bUsePawnControlRotation` | `bool` | 是否跟随 Pawn 控制旋转 |
  | `bInheritPitch/Yaw/Roll` | `bool` | 从父组件继承哪些旋转 |
  | `bEnableCameraLag` / `bEnableCameraRotationLag` | `bool` | 启用位置/旋转延迟 |
  | `CameraLagSpeed` / `CameraRotationLagSpeed` | `float` | 延迟速度 |
  | `CameraLagMaxDistance` | `float` | 延迟最大距离 |

#### 5.4.3 UCameraComponent — 相机视角与设置
- **路径**：`Engine/Classes/Camera/CameraComponent.h`
- **继承**：`USceneComponent`
- **作用**：提供相机视角，定义 FOV、后处理等设置。
- **关键属性**：
  | 属性 | 类型 | 说明 |
  |------|------|------|
  | `FieldOfView` | `float` | 水平视场角（度） |
  | `OrthoWidth` | `float` | 正交投影宽度 |
  | `AspectRatio` | `float` | 宽高比 |
  | `bConstrainAspectRatio` | `bool` | 是否强制宽高比（加黑边） |
  | `ProjectionMode` | `ECameraProjectionMode` | 透视 / 正交 |
  | `PostProcessSettings` | `FPostProcessSettings` | 后处理设置覆盖 |
  | `PostProcessBlendWeight` | `float` | 后处理混合权重 |
- **关键函数**：`GetCameraView(float DeltaTime, FMinimalViewInfo&)` 计算相机视角，`SetFieldOfView()` 等蓝图设置函数

#### 5.4.4 UInputComponent — 输入绑定组件
- **路径**：`Engine/Classes/Components/InputComponent.h`
- **关键成员**：`KeyBindings`, `AxisBindings`, `ActionBindings`, `TouchBindings` 等绑定数组；`Priority` (输入栈优先级)；`bBlockInput` (阻止低优先级组件)
- **关键函数**：`ClearBindingsForObject(UObject*)`, `HasBindings()`, `GetAxisValue()`

#### 5.4.5 UChildActorComponent
- **作用**：将一个 Actor 作为组件附加到另一个 Actor，创建 Actor 层级结构。
- 子 Actor 可以有自己独立的 Tick 和网络复制。

#### 5.4.6 UNiagaraComponent
- **作用**：Niagara VFX 粒子系统（UE5 替代 Cascade 的现代化粒子系统）。
- 继承自 `UFXSystemComponent`，提供高性能 GPU 粒子模拟。

#### 5.4.7 UAudioComponent
- **作用**：3D 空间音效组件，附加到 Actor 上可在世界中播放声音。
- 继承自 `USceneComponent`，支持衰减、空间化等。

#### 5.4.8 UWidgetComponent
- **作用**：在 3D 世界中渲染 UMG Widget（UI），实现世界空间 UI。
- 继承自 `UPrimitiveComponent`，支持与玩家交互（点击、悬停）。

---

## 6. 网络游戏开发

### 6.1 网络架构总览

UE5 使用 **Client-Server (客户端-服务器)** 模型：

```
               ┌───────────┐
               │  SERVER   │  (Authoritative - 权威)
               │  (Listen  │  ← 运行 GameMode, GameState, AI
               │   Server) │  ← 所有重要逻辑都在服务器判定
               └─────┬─────┘
          ┌──────────┼──────────┐
    ┌─────▼─────┐ ┌──▼──────┐ ┌─▼──────────┐
    │ Client 1  │ │Client 2 │ │ Client 3    │
    │(本地玩家)  │ │(远程玩家)│ │ (远程玩家)  │
    │有PC,无GM  │ │有PC,无GM│ │ 有PC,无GM   │
    └───────────┘ └─────────┘ └─────────────┘
```

**核心原则**：
- **服务器是权威的**（Server is authoritative）：所有关键判定都在服务器
- **客户端是模拟的**（Client is predictive/simulated）：客户端预测显示，服务器校正
- **GameMode 只在服务器上存在**
- **PlayerController 在服务器和其本地客户端上存在**
- **GameState 和 PlayerState 在所有机器上都存在（复制）**

### 6.2 ENetRole 网络角色

每个 Actor 在不同机器上有不同的网络角色：

| 角色 | 值 | 含义 |
|------|-----|------|
| `ROLE_None` | 0 | 不参与网络（未复制） |
| `ROLE_SimulatedProxy` | 1 | 在其他客户端上的模拟副本 |
| `ROLE_AutonomousProxy` | 2 | 在本地客户端上的玩家控制副本 |
| `ROLE_Authority` | 3 | 服务器上的权威副本 |

```cpp
// 判断服务器/客户端
if (HasAuthority()) { /* 服务器逻辑 */ }
if (IsLocallyControlled()) { /* 本地玩家 */ }

// 获取网络角色
ENetRole LocalRole = GetLocalRole();    // 本机上的角色
ENetRole RemoteRole = GetRemoteRole();  // 远程机器上的角色
```

**图示**：
```
Server (Authority):
  LocalRole  = ROLE_Authority
  RemoteRole = ROLE_AutonomousProxy (for owning client)

Client (Owning):
  LocalRole  = ROLE_AutonomousProxy
  RemoteRole = ROLE_Authority

Client (Other):
  LocalRole  = ROLE_SimulatedProxy
  RemoteRole = ROLE_Authority
```

### 6.3 网络复制 (Replication)

#### 6.3.1 属性复制
```cpp
// 1. 声明需要复制的属性
UPROPERTY(Replicated)
float Health;

// 2. 在 GetLifetimeReplicatedProps 中注册
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyActor, Health);
    // DOREPLIFETIME_CONDITION(AMyActor, Ammo, COND_OwnerOnly); // 条件复制
    // DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Score, COND_None, REPNOTIFY_Always);
}

// 3. RepNotify 回调（可选）
UFUNCTION()
void OnRep_Health()
{
    // 当 Health 复制到客户端时调用
    UpdateHealthBar();
}
```

#### 6.3.2 RPC (远程过程调用)

```cpp
// Server RPC - 客户端调用，服务器执行
UFUNCTION(Server, Reliable)
void ServerFire(FVector_NetQuantize AimLocation);
void AMyCharacter::ServerFire_Implementation(FVector_NetQuantize AimLocation) { ... }
bool AMyCharacter::ServerFire_Validate(FVector_NetQuantize AimLocation) { return true; }

// Client RPC - 服务器调用，客户端执行
UFUNCTION(Client, Reliable)
void ClientShowDamage(float Damage);
void AMyCharacter::ClientShowDamage_Implementation(float Damage) { ... }

// NetMulticast RPC - 服务器调用，所有客户端执行
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayEffect(FVector Location);
void AMyCharacter::MulticastPlayEffect_Implementation(FVector Location) { ... }
```

**RPC 规范**：
| 规范 | 说明 |
|------|------|
| `Server` | 仅从客户端调用，在服务器执行 |
| `Client` | 仅从服务器调用，在拥有该 Actor 的客户端执行 |
| `NetMulticast` | 仅从服务器调用，在所有客户端执行 |
| `Reliable` | 保证到达（用于关键数据） |
| `Unreliable` | 不保证到达（用于频繁/不重要数据，如移动） |
| `WithValidation` | 添加 Validate 函数（防作弊） |

#### 6.3.3 网络相关性 (Net Relevance)

决定 Actor 是否复制到某个客户端：

```cpp
// 相关性设置
bAlwaysRelevant = true;      // 对所有客户端始终相关
bOnlyRelevantToOwner = true; // 仅对其 Owner 相关
bNetUseOwnerRelevancy = true; // 使用 Owner 的相关性判断

// 距离相关性
NetCullDistanceSquared = 1000000.0f; // 超过此距离不再相关

// 自定义相关性
virtual bool IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, 
    const FVector& SrcLocation) const override { ... }
```

### 6.4 网络移动 (Character Movement Replication)

ACharacter 的内置移动复制机制是 UE5 网络开发的核心组件：

```
Client (AutonomousProxy):
  1. 本地输入 → 本地预测移动
  2. 将移动数据通过 ServerMovePacked RPC 发送给服务器
  3. 将移动保存到 SavedMoves 队列中

Server (Authority):
  4. 接收 ServerMovePacked
  5. 在服务器上重新执行移动
  6. 如果位置不匹配 → 发送 ClientMoveResponsePacked 校正
  7. 否则 → 发送 ACK
  8. 将所有客户端的 Pawn 设为 SimulatedProxy 以进行插值

Other Clients (SimulatedProxy):
  9. 收到复制的位置
  10. 在复制的位置之间进行平滑插值
```

**关键函数和变量**：
- `bReplicateMovement = true` — 启用移动复制
- `ServerMovePacked(FCharacterServerMovePackedBits)` — UE5 新增的高效移动 RPC（替代旧版 ServerMove 系列）
- `ClientMoveResponsePacked(FCharacterMoveResponsePackedBits)` — 服务器位置校正
- `ReplicatedBasedMovement` — 基于物体（如移动平台）的相对移动复制
- `ReplicatedServerLastTransformUpdateTimeStamp` — 用于模拟代理插值

### 6.5 网络最佳实践

#### Do's
1. **服务器验证所有关键操作**：永远不要信任客户端发来的数据
2. **使用 WithValidation**：对 Server RPC 添加 Validate 函数
3. **区分可靠/不可靠 RPC**：移动用 Unreliable，伤害/事件用 Reliable
4. **使用 `DOREPLIFETIME_CONDITION`**：按条件复制，减少带宽
5. **对频繁更新的属性使用 RepNotify**：在客户端立即响应
6. **`HasAuthority()`** 保护服务器逻辑
7. **对位置使用量化类型**：`FVector_NetQuantize`, `FVector_NetQuantize10`, `FVector_NetQuantize100` 减少带宽
8. **UE5 使用 Iris 复制系统**（可选）：在 `DefaultEngine.ini` 中启用 `[SystemSettings] net.Iris.UseIrisReplication=1`

#### Don'ts
1. **不要在客户端上执行权威逻辑**
2. **不要依赖客户端同步来做公平竞技**（反作弊）
3. **不要在客户端直接修改复制属性**（会被服务器覆盖）
4. **避免大量使用 NetMulticast**（所有客户端都收到）

### 6.6 网络相关变量速查 (AActor)

| 变量 | 默认 | 说明 |
|------|------|------|
| `bReplicates` | false | 是否复制（必须在服务器设置为 true） |
| `bReplicateMovement` | false | 是否复制移动（ACharacter 默认 true） |
| `bAlwaysRelevant` | false | 对所有客户端始终相关 |
| `bOnlyRelevantToOwner` | false | 仅对 Owner 相关 |
| `bNetTemporary` | false | 生成时发送，之后不再更新 |
| `bNetLoadOnClient` | false | 客户端加载地图时创建 |
| `bNetUseOwnerRelevancy` | false | 使用 Owner 的相关性判断 |
| `bCallPreReplication` | false | 复制前调用 PreReplication |
| `NetCullDistanceSquared` | 0 | 网络裁剪距离平方 |
| `NetUpdateFrequency` | 100 | 每秒更新频率 |
| `MinNetUpdateFrequency` | 2 | 最小每秒更新频率 |
| `NetPriority` | 1.0 | 网络优先级（带宽分配权重） |

### 6.7 Iris 复制系统 (UE5.5)

UE5.5 引入了新的 Iris 复制系统（默认仍使用旧系统），可通过配置启用：

```ini
[SystemSettings]
net.Iris.UseIrisReplication=1
```

Iris 特点：
- 更高效的带宽利用
- 更好的优先级管理
- 简化的复制配置
- 与旧版 API 兼容

### 6.8 网络开发常见模式

#### 模式 1：服务器驱动的伤害系统
```cpp
// MyCharacter.h
UFUNCTION(Server, Reliable, WithValidation)
void ServerApplyDamage(float Damage, AActor* DamageCauser);
void ServerApplyDamage_Implementation(float Damage, AActor* DamageCauser);
bool ServerApplyDamage_Validate(float Damage, AActor* DamageCauser);

// 在客户端检查条件，请求服务器
void AMyCharacter::OnTakeHit(float Damage)
{
    if (IsLocallyControlled())
    {
        ServerApplyDamage(Damage, GetInstigator());
    }
}
```

#### 模式 2：属性复制
```cpp
// 在服务器上修改属性，自动复制到所有客户端
UPROPERTY(ReplicatedUsing = OnRep_Team)
int32 TeamId;

void AMyPlayerState::SetTeam(int32 NewTeam)
{
    if (HasAuthority())
    {
        TeamId = NewTeam;
        // 自动复制到客户端，触发 OnRep_Team
    }
}
```

#### 模式 3：游戏状态同步
```cpp
// AGameState - 服务器修改后自动复制
UPROPERTY(Replicated)
int32 RemainingTime;

// 客户端读取
if (AGameStateBase* GS = GetWorld()->GetGameState())
{
    int32 Time = GS->RemainingTime;
}
```

---

## 附录 A：核心生命周期函数调用顺序

### A.1 各阶段对比

| 阶段 | UObject | AActor | UActorComponent | UWorld |
|------|---------|--------|-----------------|--------|
| **创建/加载** | `PostInitProperties()` (C++构造后) | `PostActorCreated()` | `OnComponentCreated()` | `InitWorld()` |
| | `PostLoad()` (从磁盘) | `PreRegisterAllComponents()` | `RegisterComponent()` / `OnRegister()` | `CreateWorld()` |
| | | `PostRegisterAllComponents()` | | |
| **初始化** | | `PreInitializeComponents()` | `InitializeComponent()` | |
| | | `PostInitializeComponents()` | | |
| **开始游戏** | | `BeginPlay()` | `BeginPlay()` | `BeginPlay()` |
| **Tick** | | `Tick(float)` / `TickActor()` | `TickComponent(float)` | `Tick()` |
| **结束游戏** | | `EndPlay(EEndPlayReason)` | `EndPlay(EEndPlayReason)` | |
| **销毁** | `BeginDestroy()` | `Destroyed()` | `OnComponentDestroyed()` | `CleanupWorld()` |
| | `IsReadyForFinishDestroy()` | | `UninitializeComponent()` | |
| | `FinishDestroy()` | | `OnUnregister()` | |

### A.2 Actor 初始化详细流程

```
在编辑器中放置的 Actor:
  PostLoad → PreRegisterAllComponents → RegisterComponent → 
  PostRegisterAllComponents → PostActorCreated → 
  ExecuteConstruction(UserConstructionScript + OnConstruction)

运行时 Spawn 的 Actor:
  OnComponentCreated → PreRegisterAllComponents → RegisterComponent → 
  PostRegisterAllComponents → PostActorCreated → 
  ExecuteConstruction(UserConstructionScript + OnConstruction) →
  PreInitializeComponents → InitializeComponent + Activate → 
  PostInitializeComponents → BeginPlay → Tick(循环)

Actor 销毁:
  RemoveFromWorld → EndPlay → Destroy → UnregisterAllComponents → 
  DestroyPhysicsState → DestroyRenderState → OnComponentDestroyed
```

## 附录 B：游戏启动和地图加载流程

```
UEngine::Init()
  → UGameEngine::Init()
    → UGameInstance::Init()
      → CreateLocalPlayer() (每个本地玩家)
        → 创建 ULocalPlayer
      → LoadMap()
        → 创建 UWorld
        → GEngine->LoadMap()
          → UWorld::InitWorld()
            → 基于 World Settings/URL/Default 确定 GameMode 类
            → SpawnPlayActor() ← 在服务器上生成 GameMode, GameState
              → AGameModeBase::InitGame()
                → Spawn GameState, GameSession
                → InitGameState()
          → UGameInstance::StartGameInstance()
            → 世界初始化完成
          → 对于每个连接的客户端:
            → AGameModeBase::PreLogin() → Login() → PostLogin()
              → Spawn PlayerController
              → PlayerController → 客户端复制
                → 客户端: ReceivedPlayer()
          → AGameModeBase::StartPlay()
            → 触发所有 Actor 的 BeginPlay()
              → Tick 循环开始
```

## 附录 C：常用宏速查

| 宏 | 用途 |
|-----|------|
| `UCLASS()` | 声明 UObject/AActor 类，配置元数据 |
| `USTRUCT()` | 声明 UStruct |
| `UENUM()` | 声明枚举 |
| `UPROPERTY(...)` | 声明属性（反射、GC、序列化、网络复制） |
| `UFUNCTION(...)` | 声明函数（蓝图可调用、RPC 等） |
| `GENERATED_BODY()` | 生成类样板代码 |
| `GENERATED_UCLASS_BODY()` | 生成类样板代码（带默认构造函数） |
| `GENERATED_USTRUCT_BODY()` | 生成结构体样板代码 |
| `UFUNCTION(Server, Reliable, WithValidation)` | Server RPC |
| `UFUNCTION(Client, Reliable)` | Client RPC |
| `UFUNCTION(NetMulticast, Unreliable)` | Multicast RPC |
| `DOREPLIFETIME(Class, Prop)` | 注册复制属性 |
| `DOREPLIFETIME_CONDITION(Class, Prop, Cond)` | 条件复制 |
| `DOREPLIFETIME_CONDITION_NOTIFY(Class, Prop, Cond, Notify)` | 条件复制+通知 |
