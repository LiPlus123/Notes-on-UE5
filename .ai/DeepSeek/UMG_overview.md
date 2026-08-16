# UE5 UMG 系统设计与实现原理

## 目录

1. [架构概览](#1-架构概览)
2. [核心类层次结构](#2-核心类层次结构)
3. [Slate 集成层：UMG 与 Slate 的桥梁](#3-slate-集成层umg-与-slate-的桥梁)
4. [Widget Blueprint 编译与 WidgetTree](#4-widget-blueprint-编译与-widgettree)
5. [Widget 生命周期](#5-widget-生命周期)
6. [面板系统与槽位（Panel & Slot）](#6-面板系统与槽位panel--slot)
7. [属性绑定系统](#7-属性绑定系统)
8. [动画系统](#8-动画系统)
9. [输入事件路由](#9-输入事件路由)
10. [渲染管线](#10-渲染管线)
11. [性能优化机制](#11-性能优化机制)
12. [总结](#12-总结)

---

## 1. 架构概览

### 1.1 双层架构

UMG（Unreal Motion Graphics）是 UE5 的 UI 创作框架。它采用**双层架构**：

```
┌─────────────────────────────────────────┐
│         UMG 层 (UObject 世界)             │
│  UWidget / UUserWidget / UPanelWidget    │
│  负责：蓝图编辑、反射、GC、序列化、编辑器集成  │
└──────────────┬──────────────────────────┘
               │ SObjectWidget (桥接层)
┌──────────────▼──────────────────────────┐
│         Slate 层 (C++ 原生世界)           │
│  SWidget / SCompoundWidget / SPanel      │
│  负责：布局、绘制、输入、高性能渲染          │
└─────────────────────────────────────────┘
```

**设计动机**：Slate 是纯 C++ 的 UI 框架，使用大量的模板和共享指针，不兼容 UE 的反射/蓝图/GC 系统。UMG 在每个 Slate Widget 外面包裹一层 `UObject` 派生类，使得 C++ 的 Slate 控件可以被蓝图访问、被 GC 管理、支持属性绑定和动画。

### 1.2 核心设计原则

1. **一一对应**：每个 UMG Widget（`UWidget`）在底层都有一个对应的 Slate Widget（`SWidget`），通过 `TWeakPtr<SWidget> MyWidget` 持有引用。

2. **延迟构造**：UMG Widget 在创建时不会立即构造 Slate Widget。只有在 `TakeWidget()` 首次被调用时，才通过 `RebuildWidget()` 创建底层 Slate 对象。这种设计避免了不必要的 Slate 资源分配。

3. **GC 保护**：`SObjectWidget` 实现了 `FGCObject` 接口，持有 `TObjectPtr<UUserWidget>`，防止正在使用的 UWidget 被 GC 回收。

4. **属性同步**：UMG Widget 的属性变更通过 `SynchronizeProperties()` 推送到 Slate 层，确保两层状态一致。

---

## 2. 核心类层次结构

### 2.1 继承链

```
UObject
 ├── UVisual                          ← 可视对象基类
 │    ├── UWidget                     ← 所有 UMG Widget 的基类
 │    │    ├── UPanelWidget           ← 容器面板基类
 │    │    │    ├── UCanvasPanel
 │    │    │    ├── UHorizontalBox
 │    │    │    ├── UVerticalBox
 │    │    │    ├── UOverlay
 │    │    │    ├── UGridPanel
 │    │    │    ├── UScrollBox
 │    │    │    └── ...
 │    │    ├── UUserWidget            ← 蓝图可编辑的用户 Widget
 │    │    ├── UTextBlock
 │    │    ├── UImage
 │    │    ├── UButton
 │    │    ├── UBorder
 │    │    └── ... (其他 Leaf Widget)
 │    └── UPanelSlot                  ← 槽位基类
 │         ├── UCanvasPanelSlot
 │         ├── UHorizontalBoxSlot
 │         ├── UOverlaySlot
 │         └── ...
```

对应的 Slate 层：

```
SWidget
 ├── SCompoundWidget                  ← 单子节点 Widget 基类
 │    └── SObjectWidget               ← UMG 与 Slate 的桥接器 ⭐
 ├── SPanel                           ← 多子节点面板基类
 │    ├── SConstraintCanvas
 │    ├── SBoxPanel (SHorizontalBox / SVerticalBox)
 │    ├── SOverlay
 │    ├── SGridPanel
 │    └── ...
 └── SLeafWidget                      ← 叶子 Widget 基类
      ├── STextBlock
      ├── SImage
      └── ...
```

### 2.2 UWidget：一切 UMG Widget 的根基

[Widget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Components\Widget.h) 定义了 UMG Widget 的核心。

**关键成员**：

```cpp
class UWidget : public UVisual, public INotifyFieldValueChanged
{
protected:
    TWeakPtr<SWidget> MyWidget;        // 底层 Slate Widget（弱引用）
    TWeakPtr<SObjectWidget> MyGCWidget; // GC 保护的容器 Widget

    UPROPERTY(Instanced)
    TObjectPtr<UPanelSlot> Slot;       // 指向父容器中的槽位

    // 属性绑定代理（Delegate）
    FGetBool bIsEnabledDelegate;
    FGetText ToolTipTextDelegate;
    FGetSlateVisibility VisibilityDelegate;
    // ...
};
```

**核心方法**：

| 方法 | 作用 |
|------|------|
| `TakeWidget()` | 获取或构造底层 Slate Widget |
| `RebuildWidget()` | 子类重写，创建具体的 Slate Widget |
| `SynchronizeProperties()` | 将 UMG 属性同步到 Slate 层 |
| `ReleaseSlateResources()` | 释放 Slate 资源（Widget 被移除时） |

### 2.3 UUserWidget：蓝图 UI 的入口

[UserWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\UserWidget.h) 是用户在蓝图中创建 UI 的基类。

**关键成员**：

```cpp
class UUserWidget : public UWidget, public INamedSlotInterface
{
public:
    UPROPERTY(Transient)
    TObjectPtr<UWidgetTree> WidgetTree;           // Widget 模板树

    UPROPERTY(Transient)
    TArray<TObjectPtr<UUMGSequencePlayer>> ActiveSequencePlayers;  // 活动动画播放器

    TArray<FQueuedWidgetAnimationTransition> QueuedWidgetAnimationTransitions;

    // 蓝图事件
    void OnInitialized();    // 一次性初始化
    void PreConstruct();     // 构造前（编辑器和运行时）
    void Construct();        // Slate 构造完成后
    void Destruct();         // 销毁前
    void Tick();             // 每帧更新
    void OnPaint();          // 绘制重载
};
```

---

## 3. Slate 集成层：UMG 与 Slate 的桥梁

### 3.1 SObjectWidget：双向桥接器

[SObjectWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Slate\SObjectWidget.h) 是整个 UMG 架构中最关键的类。它是 Slate 层的一个 `SCompoundWidget`，同时实现 `FGCObject` 接口。

```cpp
class SObjectWidget : public SCompoundWidget, public FGCObject
{
    TObjectPtr<UUserWidget> WidgetObject;  // 强引用，防止 GC

    // 所有输入事件都转发给 WidgetObject
    virtual FReply OnMouseButtonDown(...) override {
        if (CanRouteEvent())
            return WidgetObject->NativeOnMouseButtonDown(...);
        return FReply::Unhandled();
    }
    virtual FReply OnKeyDown(...) override {
        if (CanRouteEvent())
            return WidgetObject->NativeOnKeyDown(...);
        return FReply::Unhandled();
    }
    // ... 所有其他输入事件同理

    // Tick 转发
    virtual void Tick(...) override {
        if (WidgetObject) WidgetObject->NativeTick(...);
    }

    // 绘制转发
    virtual int32 OnPaint(...) const override {
        return WidgetObject->NativePaint(...);
    }
};
```

**设计要点**：
- `SObjectWidget` 作为 UUserWidget 的根 Slate Widget，内部 `Content` slot 包含实际的 WidgetTree 根节点
- `FGCObject::AddReferencedObjects()` 确保 UUserWidget 在 Slate 还活着时不被 GC
- 通过 `ResetWidget()` 断开引用，允许 Widget 被安全销毁

### 3.2 Widget 构造流程：TakeWidget()

```
TakeWidget()
  ├── 检查 MyWidget 是否有效 → 若有效直接返回
  ├── 调用 RebuildWidget() → 创建底层 SWidget
  ├── 若为 UUserWidget：
  │    ├── 获取 WidgetTree 的根 Widget
  │    ├── 调用根 Widget 的 TakeWidget()
  │    └── 用 SObjectWidget 包裹：
  │         SNew(SObjectWidget, this) [ RootSlateWidget ]
  └── 调用 SynchronizeProperties() → 同步属性
```

**关键代码路径**（[Widget.cpp](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Private\Components\Widget.cpp)）：

```cpp
TSharedRef<SWidget> UWidget::TakeWidget()
{
    TSharedPtr<SWidget> SafeWidget = MyWidget.Pin();
    if (!SafeWidget.IsValid())
    {
        SafeWidget = RebuildWidget();   // 虚函数，子类实现
        MyWidget = SafeWidget;
        OnWidgetRebuilt();              // 构造后回调
    }
    return SafeWidget.ToSharedRef();
}
```

---

## 4. Widget Blueprint 编译与 WidgetTree

### 4.1 WidgetBlueprintGeneratedClass

[WidgetBlueprintGeneratedClass.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\WidgetBlueprintGeneratedClass.h) 是 Widget Blueprint 编译的产物。它继承自 `UBlueprintGeneratedClass`，额外存储了：

```cpp
class UWidgetBlueprintGeneratedClass : public UBlueprintGeneratedClass
{
    UPROPERTY()
    TObjectPtr<UWidgetTree> WidgetTree;    // Widget 模板树（Archetype）

    UPROPERTY()
    TArray<FDelegateRuntimeBinding> Bindings;  // 属性绑定列表

    UPROPERTY()
    TArray<TObjectPtr<UWidgetAnimation>> Animations;  // 动画列表

    UPROPERTY()
    TArray<FName> NamedSlots;              // 命名槽位列表

    UPROPERTY()
    TArray<FName> AvailableNamedSlots;     // 子类可用的命名槽位
};
```

### 4.2 WidgetTree：模板树结构

[WidgetTree.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\WidgetTree.h) 管理 Widget Blueprint 中的 Widget 集合：

```cpp
class UWidgetTree : public UObject, public INamedSlotInterface
{
    UPROPERTY(Instanced)
    TObjectPtr<UWidget> RootWidget;  // 树的根节点

    UPROPERTY()
    TMap<FName, TObjectPtr<UWidget>> NamedSlotBindings;  // 命名槽位绑定
};
```

**WidgetTree 的双重角色**：
1. **编译时**：作为模板（Archetype），存储 Widget 的层级结构和默认属性
2. **运行时**：被复制（Duplicate）到 UUserWidget 实例，形成每个实例独立的 Widget 树

### 4.3 InitializeWidget：运行时初始化

当 `UUserWidget::Initialize()` 被调用时，编译期生成的类开始执行运行时初始化：

```
UUserWidget::Initialize()
  ├── DuplicateAndInitializeFromWidgetTree()
  │    ├── 复制 WidgetTree 模板
  │    ├── 合并 NamedSlot 内容
  │    └── 调用每个 Widget 的 Initialize()
  ├── InitializeNamedSlots()    ← 绑定命名槽位
  ├── WidgetBlueprintGeneratedClass::InitializeWidget(this)
  │    ├── 执行属性绑定 (Bindings)
  │    ├── 绑定动画属性 (BindAnimationsStatic)
  │    └── 调用扩展的 Initialize
  └── NativeOnInitialized()
       └── OnInitialized() ← 蓝图事件
```

### 4.4 FDelegateRuntimeBinding：运行时绑定

```cpp
USTRUCT()
struct FDelegateRuntimeBinding
{
    FString ObjectName;              // 目标 Widget 名称
    FName PropertyName;              // 目标属性名称
    FName FunctionName;              // 源函数/属性名称
    FDynamicPropertyPath SourcePath; // 源属性路径
    EBindingKind Kind;              // Function 或 Property
};
```

运行时，`InitializeWidgetStatic` 遍历所有 `Bindings`，通过名称查找 Widget，然后建立属性到属性或属性到函数的绑定。这本质上是将蓝图中的绑定图转换为 C++ 可高效执行的 TAttribute。

---

## 5. Widget 生命周期

### 5.1 完整生命周期图

```
创建阶段：
  UUserWidget::CreateWidgetInstance()
    ├── NewObject<UUserWidget>
    ├── DuplicateAndInitializeFromWidgetTree()
    └── Initialize()
         ├── WidgetTree 复制与初始化
         └── NativeOnInitialized() → OnInitialized()

构造阶段（首次添加到视口/父容器时）：
  AddToViewport()
    └── TakeWidget()  ← 首次调用触发 RebuildWidget
         ├── WidgetTree 根节点 TakeWidget() → 递归构造整个树
         └── SObjectWidget 包裹根 Slate Widget
    └── AddToScreen → SlateApplication 添加
    └── NativePreConstruct() → PreConstruct()
    └── NativeConstruct() → Construct()

Tick 阶段（每帧）：
  SObjectWidget::Tick()
    └── NativeTick()
         ├── TickActionsAndAnimation() ← 动画更新、Latent Action
         └── Tick() ← 蓝图 Tick 事件

销毁阶段：
  RemoveFromParent()
    └── ReleaseSlateResources()
    └── NativeDestruct() → Destruct()
    └── GC 回收
```

### 5.2 PreConstruct vs Construct vs OnInitialized

| 事件 | 调用时机 | 编辑器 | 运行时 | 用途 |
|------|---------|--------|--------|------|
| `OnInitialized` | 仅一次，NewObject 后 | ✗ | ✓ | 绑定回调、初始化状态 |
| `PreConstruct` | 每次 Slate 重建前 | ✓ | ✓ | 预览相关的外观更新 |
| `Construct` | Slate Widget 构造完成后 | ✗ | ✓ | 运行时 UI 初始化 |

### 5.3 RebuildWidget 模式

每个 UWidget 子类实现 `RebuildWidget()` 来创建其对应的 Slate Widget：

```cpp
// 以 UCanvasPanel 为例
TSharedRef<SWidget> UCanvasPanel::RebuildWidget()
{
    MyCanvas = SNew(SConstraintCanvas);  // 创建 Slate 原生面板
    for (UPanelSlot* Slot : GetSlots())
    {
        if (UWidget* Content = Slot->Content)
        {
            MyCanvas->AddSlot()
                .Expose(Slot)            // 将 UPanelSlot 暴露给 Slate
                [ Content->TakeWidget() ]; // 递归构造子 Widget
        }
    }
    return MyCanvas.ToSharedRef();
}
```

---

## 6. 面板系统与槽位（Panel & Slot）

### 6.1 设计模式

UMG 的面板系统采用**组合模式**：

```
UPanelWidget (UWidget)
  └── Slots: TArray<UPanelSlot*>
       └── UPanelSlot (UVisual)
            ├── Parent: UPanelWidget*   ← 反向引用
            ├── Content: UWidget*       ← 子 Widget
            └── SynchronizeProperties()  ← 虚函数，子类实现
```

每个面板类型都有一个对应的 Slot 类型：

| 面板 | Slate 面板 | Slot 类 |
|------|-----------|---------|
| `UCanvasPanel` | `SConstraintCanvas` | `UCanvasPanelSlot` |
| `UHorizontalBox` | `SHorizontalBox` | `UHorizontalBoxSlot` |
| `UVerticalBox` | `SVerticalBox` | `UVerticalBoxSlot` |
| `UOverlay` | `SOverlay` | `UOverlaySlot` |
| `UGridPanel` | `SGridPanel` | `UGridSlot` |
| `UScrollBox` | `SScrollBox` | `UScrollBoxSlot` |

### 6.2 Slot 属性同步

每个 Slot 类型的 `SynchronizeProperties()` 负责将 UMG 层的布局属性转换为 Slate 层的布局参数：

```
UMG 层                          Slate 层
┌─────────────────┐            ┌──────────────────┐
│ UCanvasPanelSlot │            │ SConstraintCanvas │
│  - LayoutData    │──同步──→   │  - Anchors        │
│  - Anchors       │            │  - Offsets        │
│  - Alignment     │            │  - Alignment      │
│  - ZOrder        │            │  - ZOrder         │
│  - bAutoSize     │            │  - AutoSize       │
└─────────────────┘            └──────────────────┘
```

### 6.3 命名槽位（Named Slot）

命名槽位允许 UserWidget 暴露"插槽"供父类填充内容。这机制支持 Widget 组合和模板继承：

```cpp
// UNamedSlot 是一个特殊的 PanelWidget
class UNamedSlot : public UContentWidget  // 只能容纳单个子节点
{
    // 通过 FName 标识
};

// UUserWidget 实现 INamedSlotInterface
// 通过 NamedSlotBindings 将 FName 映射到 UWidget*
```

---

## 7. 属性绑定系统

### 7.1 设计目标

属性绑定系统允许 Widget 的属性（如 TextBlock 的 Text、Image 的 Brush）动态绑定到数据源（如 PlayerState、GameState 的属性）。当数据源变化时，UI 自动更新。

### 7.2 PROPERTY_BINDING 宏机制

[Widget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Components\Widget.h) 定义了属性绑定的核心宏：

```cpp
// 编辑器版本：加入调试间接层
#define PROPERTY_BINDING(ReturnType, MemberName)
    (MemberName##Delegate.IsBound() && !IsDesignTime())
    ? BIND_UOBJECT_ATTRIBUTE(ReturnType, K2_Gate_##MemberName)
    : TAttribute<ReturnType>(MemberName)

// 运行时版本：直接创建 TAttribute
#define PROPERTY_BINDING(ReturnType, MemberName)
    (MemberName##Delegate.IsBound() && !IsDesignTime())
    ? TAttribute<ReturnType>::Create(
        MemberName##Delegate.GetUObject(),
        MemberName##Delegate.GetFunctionName())
    : TAttribute<ReturnType>(MemberName)
```

**工作原理**：
1. 如果绑定代理已绑定，则创建动态 `TAttribute`（每次获取时调用绑定的函数）
2. 如果绑定代理未绑定，则使用静态值
3. `TAttribute` 是 Slate 的核心机制，支持惰性求值

### 7.3 UPropertyBinding 类

[PropertyBinding.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Binding\PropertyBinding.h)：

```cpp
class UPropertyBinding : public UObject
{
    TWeakObjectPtr<UObject> SourceObject;  // 绑定源对象
    FDynamicPropertyPath SourcePath;       // 源属性路径
    FName DestinationProperty;             // 目标属性名称

    virtual void Bind(FProperty* Property, FScriptDelegate* Delegate);
};
```

**预定义的绑定类型**（避免 VM 调用开销）：

| 绑定类 | 绑定类型 | Slate 类型 |
|--------|---------|------------|
| `UBoolBinding` | `bool` | `ECheckBoxState` |
| `UFloatBinding` | `float` | `float` |
| `UInt32Binding` | `int32` | `int32` |
| `UTextBinding` | `FText` | `FText` |
| `UColorBinding` | `FLinearColor` | `FSlateColor` |
| `UVisibilityBinding` | `ESlateVisibility` | `EVisibility` |
| `UBrushBinding` | `FSlateBrush` | `FSlateBrush` |
| `UWidgetBinding` | `UWidget*` | `TSharedRef<SWidget>` |

### 7.4 绑定注册

```cpp
bool UWidget::AddBinding(FDelegateProperty* DelegateProperty,
    UObject* SourceObject, const FDynamicPropertyPath& BindingPath)
{
    // 1. 查找适合的 BinderClass
    TSubclassOf<UPropertyBinding> BinderClass =
        FindBinderClassForDestination(DelegateProperty);

    // 2. 创建绑定实例
    UPropertyBinding* Binding = NewObject<UPropertyBinding>(...);
    Binding->SourceObject = SourceObject;
    Binding->SourcePath = BindingPath;
    Binding->Bind(DelegateProperty, ...);  // 子类实现具体的绑定逻辑

    // 3. 注册到 NativeBindings
    NativeBindings.Add(Binding);
}
```

---

## 8. 动画系统

### 8.1 架构概览

UMG 的动画系统基于 Sequencer 框架构建：

```
UWidgetAnimation (UMovieSceneSequence)
  └── UMovieScene
       └── UMovieSceneTrack (如 2D Transform Track)
            └── UMovieSceneSection (关键帧段落)

UUMGSequencePlayer (IMovieScenePlayer)
  ├── RootTemplateInstance (FMovieSceneRootEvaluationTemplateInstance)
  └── 负责驱动动画播放

UUMGSequenceTickManager (全局单例)
  ├── 管理所有 Widget 的动画 Tick
  ├── 拥有 Linker (UMovieSceneEntitySystemLinker)
  └── 拥有 Runner (FMovieSceneEntitySystemRunner)
```

### 8.2 UWidgetAnimation

[WidgetAnimation.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\WidgetAnimation.h)：

```cpp
class UWidgetAnimation : public UMovieSceneSequence
{
    UPROPERTY()
    TObjectPtr<UMovieScene> MovieScene;          // Sequencer 场景数据

    UPROPERTY()
    TArray<FWidgetAnimationBinding> AnimationBindings;  // 动画绑定
};
```

`FWidgetAnimationBinding` 定义了动画轨道与 Widget 的映射关系：
```cpp
struct FWidgetAnimationBinding
{
    FName WidgetName;         // 目标 Widget 名称
    FName SlotWidgetName;     // 槽位 Widget 名称
    FGuid AnimationGuid;      // 动画 GUID
    // ...
};
```

### 8.3 UUMGSequencePlayer

[UMGSequencePlayer.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\UMGSequencePlayer.h) 是动画播放的执行单元：

```cpp
class UUMGSequencePlayer : public UObject, public IMovieScenePlayer
{
    UPROPERTY()
    TObjectPtr<UWidgetAnimation> Animation;     // 正在播放的动画

    FMovieSceneRootEvaluationTemplateInstance RootTemplateInstance;  // 求值模板

    // 播放状态
    FFrameTime TimeCursorPosition;    // 当前时间位置
    int32 Duration;                   // 总时长
    int32 NumLoopsToPlay;            // 循环次数
    int32 NumLoopsCompleted;         // 已完成循环数
    float PlaybackSpeed;             // 播放速度
    EUMGSequencePlayMode::Type PlayMode;  // Forward/Reverse/PingPong

    void Tick(float DeltaTime);      // 每帧更新动画
    void Play(...);
    void Stop();
    void Pause();
    void Reverse();
};
```

### 8.4 UUMGSequenceTickManager（全局动画调度器）

[UMGSequenceTickManager.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\UMGSequenceTickManager.h)：

```cpp
class UUMGSequenceTickManager : public UObject
{
    // 注册了动画的 Widget 集合
    TMap<TWeakObjectPtr<UUserWidget>, FSequenceTickManagerWidgetData> WeakUserWidgetData;

    UPROPERTY()
    TObjectPtr<UMovieSceneEntitySystemLinker> Linker;  // ECS Linker

    TSharedPtr<FMovieSceneEntitySystemRunner> Runner;   // 动画执行器

    // 在 Slate 的 PostTick 时更新所有动画
    void HandleSlatePostTick(float DeltaSeconds);
    void TickWidgetAnimations(float DeltaSeconds);
};
```

**Tick 流程**：
```
SlateApplication::PostTick
  └── UUMGSequenceTickManager::HandleSlatePostTick
       └── TickWidgetAnimations
            ├── 遍历所有注册的 UUserWidget
            ├── 处理排队的动画转换
            │    ├── Play / PlayTo
            │    ├── Forward / Reverse
            │    └── Stop / Pause
            ├── Flush (强制同步求值)
            └── 通知动画状态变化 (Started/Finished)
```

### 8.5 Sequencer 集成

UMG 动画复用了 UE5 的 Sequencer 基础设施：

- **ECS 求值系统**：动画数据通过 `UMovieSceneEntitySystemLinker` 在 ECS 中进行求值
- **2D Transform 动画**：通过 `MovieScene2DTransformPropertySystem` 系统处理 Widget 的平移/旋转/缩放
- **Margin 动画**：通过 `MovieSceneMarginPropertySystem` 处理 Margin 属性
- **Material 动画**：通过 `MovieSceneWidgetMaterialSystem` 处理材质参数动画

---

## 9. 输入事件路由

### 9.1 事件流

```
用户输入
    │
    ▼
SlateApplication (FSlateApplication)
    │
    ├── 命中测试 → 找到目标 SWidget
    │
    ▼
SWidget::OnKeyDown / OnMouseButtonDown / ...
    │
    ▼
SObjectWidget::OnKeyDown(...)  ← 拦截所有 UMG Widget 的事件
    │
    │ CanRouteEvent()? ← 安全检查（非设计时、非GC中）
    │
    ▼
UUserWidget::NativeOnKeyDown(...)
    │
    ├── 蓝图实现 → OnKeyDown() (BlueprintImplementableEvent)
    │
    └── 返回 FReply
         ├── FReply::Handled() → 事件被消费
         └── FReply::Unhandled() → 事件冒泡到父 Widget
```

### 9.2 事件类型

SObjectWidget 覆盖了所有 Slate 输入事件（见 [SObjectWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Slate\SObjectWidget.h) 完整声明）：

| 事件类别 | 方法 | 冒泡 |
|---------|------|------|
| 键盘 | `OnKeyDown`, `OnKeyUp`, `OnKeyChar`, `OnPreviewKeyDown` | ✓ |
| 鼠标 | `OnMouseButtonDown/Up`, `OnMouseMove`, `OnMouseWheel` | ✓ |
| 鼠标（预览） | `OnPreviewMouseButtonDown` | 隧道 |
| 焦点 | `OnFocusReceived`, `OnFocusLost`, `OnFocusChanging` | ✗ |
| 触摸 | `OnTouchStarted/Moved/Ended`, `OnTouchGesture` | ✓ |
| 拖拽 | `OnDragDetected/Enter/Leave/Over/Drop` | ✓ |
| 导航 | `OnNavigation` | - |

### 9.3 输入优先级与阻断

```cpp
// UUserWidget 中
UPROPERTY()
int32 Priority;        // 输入处理优先级（数值越高越优先）

UPROPERTY()
uint8 bStopAction:1;   // 是否阻断输入动作的继续传递
```

当 `bStopAction` 为 true 时，处理完的输入事件不会传递给更低优先级的 Widget。

---

## 10. 渲染管线

### 10.1 OnPaint 流程

```
SWindow / GameViewport
  └── Paint 递归遍历
       └── SObjectWidget::OnPaint()
            └── UUserWidget::NativePaint()
                 ├── 蓝图实现 → OnPaint(FPaintContext)
                 └── 返回最大 LayerId
```

**FPaintContext** 提供了蓝图可访问的绘制上下文：

```cpp
struct FPaintContext
{
    const FGeometry& AllottedGeometry;     // 分配的几何区域
    const FSlateRect& MyCullingRect;       // 裁剪矩形
    FSlateWindowElementList& OutDrawElements;  // 绘制元素列表
    int32 LayerId;                          // 当前绘制层级
    const FWidgetStyle& WidgetStyle;        // 样式
    bool bParentEnabled;                    // 父级是否启用
    int32 MaxLayer;                         // 最大层级
};
```

### 10.2 绘制元素（Draw Elements）

Slate 使用延迟渲染模型，`OnPaint` 不直接绘制，而是生成绘制元素：

1. **FSlateDrawElement**：描述一个绘制操作（矩形、文本、渐变等）
2. **FSlateWindowElementList**：收集一帧中所有绘制元素
3. **批处理**：渲染器将元素按 Batch 分组并提交到 RHI

### 10.3 Clipping（裁剪）

```cpp
UPROPERTY()
EWidgetClipping Clipping;  // 裁剪模式
```

| 模式 | 描述 | 性能 |
|------|------|------|
| `Inherit` | 继承父级 | - |
| `ClipToBounds` | 裁剪到自身边界 | 阻止批处理优化 |
| `ClipToBoundsWithoutIntersecting` | 裁剪但不与父级交集 | 阻止批处理优化 |
| `ClipToBoundsAlways` | 始终裁剪 | 阻止批处理优化 |
| `OnDemand` | 按需裁剪 | 较优 |

---

## 11. 性能优化机制

### 11.1 InvalidationBox（失效盒）

`UInvalidationBox` 包裹子 Widget，缓存绘制结果：

- **正常状态**：子 Widget 的绘制结果被缓存为 render data
- **失效时**：缓存被废弃，重新绘制
- **优点**：大幅减少复杂 UI 的每帧绘制开销
- **缺点**：缓存本身有内存开销，不适合频繁变化的 UI

### 11.2 Volatility（易变性）

```cpp
UPROPERTY()
uint8 bIsVolatile:1;  // 标记为"易变"
```

- 当 Widget 被标记为 `bIsVolatile = true` 时，InvalidationBox 不会尝试缓存其绘制结果
- 适用于每帧都在变化的 Widget（如实时倒计时文本）

### 11.3 Tick 频率控制

```cpp
enum class EWidgetTickFrequency : uint8
{
    Never,  // 永远不 Tick
    Auto,   // 自动判断（有蓝图 Tick、Latent Action 或动画时才 Tick）
};
```

`UpdateCanTick()` 动态决定是否需要 Tick：
- 如果有活动动画 → 需要 Tick
- 如果有未完成的 Latent Action → 需要 Tick
- 如果蓝图实现了 Tick → 需要 Tick
- 否则 → 不需要 Tick（Slate 跳过，节省开销）

### 11.4 Widget Pool（Widget 池）

`UUserWidgetPool` 复用时避免频繁创建/销毁：

```cpp
class UUserWidgetPool
{
    TArray<TObjectPtr<UUserWidget>> ActiveWidgets;
    TArray<TObjectPtr<UUserWidget>> InactiveWidgets;

    UUserWidget* GetOrCreateInstance(TSubclassOf<UUserWidget> WidgetClass);
    void Release(UUserWidget* Widget);  // 放回池中
};
```

ListView 和 DynamicEntryBox 内部使用 Widget Pool 来优化列表滚动的性能。

### 11.5 Pixel Snapping（像素对齐）

```cpp
UPROPERTY()
EWidgetPixelSnapping PixelSnapping;  // 像素对齐模式
```

在动画播放时，像素对齐可能导致可见的抖动。关闭像素对齐可以得到更平滑的动画，但可能失去文字清晰度。

---

## 12. 总结

UE5 UMG 系统的设计体现了清晰的**关注点分离**原则：

### 核心设计模式

| 模式 | 描述 |
|------|------|
| **双层架构** | UObject 层（反射/蓝图/GC）+ Slate 层（高性能布局/绘制/输入） |
| **桥接模式** | `SObjectWidget` 连接两层，转发事件，管理 GC |
| **组合模式** | `PanelWidget` + `Slot` 实现灵活的布局嵌套 |
| **模板方法** | `RebuildWidget()` / `SynchronizeProperties()` 由子类实现 |
| **观察者模式** | 属性绑定通过 `TAttribute` + Delegate 实现数据驱动 |
| **对象池** | `WidgetPool` 复用 Widget 减少分配开销 |

### 关键文件索引

| 文件 | 职责 |
|------|------|
| [Widget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Components\Widget.h) | UWidget 基类定义 |
| [Widget.cpp](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Private\Components\Widget.cpp) | Widget 核心实现 |
| [UserWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\UserWidget.h) | UUserWidget 定义 |
| [SObjectWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Slate\SObjectWidget.h) | Slate 桥接器 |
| [WidgetTree.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\WidgetTree.h) | Widget 模板树 |
| [WidgetBlueprintGeneratedClass.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Blueprint\WidgetBlueprintGeneratedClass.h) | 编译产物 |
| [PanelWidget.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Components\PanelWidget.h) | 面板容器基类 |
| [PanelSlot.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Components\PanelSlot.h) | 槽位基类 |
| [PropertyBinding.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Binding\PropertyBinding.h) | 属性绑定基类 |
| [WidgetAnimation.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\WidgetAnimation.h) | 动画资源 |
| [UMGSequencePlayer.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\UMGSequencePlayer.h) | 动画播放器 |
| [UMGSequenceTickManager.h](f:\GitHub\UnrealEngine\Engine\Source\Runtime\UMG\Public\Animation\UMGSequenceTickManager.h) | 全局动画调度 |

### 数据流总结

```
Widget Blueprint 编辑
    │
    ▼ (编译)
UWidgetBlueprintGeneratedClass
    ├── WidgetTree (模板)
    ├── Bindings (属性绑定)
    └── Animations (动画)
    │
    ▼ (运行时实例化)
UUserWidget 实例
    ├── WidgetTree 副本
    ├── 已执行的属性绑定 (TAttribute)
    └── ActiveSequencePlayers (动画播放器)
    │
    ▼ (TakeWidget)
SObjectWidget ──包裹──→ SWidget 树
    │
    ▼ (每帧)
SlateApplication::Tick
    ├── 输入事件 → SObjectWidget → UUserWidget Native 方法
    ├── 动画 Tick → UMGSequenceTickManager → UUMGSequencePlayer
    └── 绘制 → OnPaint → FSlateWindowElementList → RHI 渲染
```
