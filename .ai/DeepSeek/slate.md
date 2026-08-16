# UE5 Slate 系统实现原理

> 本文基于 Unreal Engine 5.5 源码（`Engine/Source/Runtime`）整理，聚焦 Slate 的核心实现原理：从模块分层、控件（Widget）对象模型、声明式语法、布局与绘制管线，到事件路由、失效机制与渲染后端。UMG 作为 Slate 的上层封装，其与 Slate 的关系在文末单独说明。

---

## 目录

1. [模块分层与总体架构](#1-模块分层与总体架构)
2. [控件对象模型：SWidget 层级](#2-控件对象模型swidget-层级)
3. [声明式语法（Declarative Syntax）内部实现](#3-声明式语法declarative-syntax内部实现)
4. [槽位与子控件（Slots & Children）](#4-槽位与子控件slots--children)
5. [TAttribute 属性绑定](#5-tattribute-属性绑定)
6. [布局与绘制管线（Prepass / Arrange / Paint）](#6-布局与绘制管线prepass--arrange--paint)
7. [FGeometry 几何与坐标变换](#7-fgeometry-几何与坐标变换)
8. [事件路由（Event Routing）](#8-事件路由event-routing)
9. [失效与快速路径（Invalidation & Fast Path）](#9-失效与快速路径invalidation--fast-path)
10. [渲染管线（DrawElements → Batcher → RHI）](#10-渲染管线drawelements--batcher--rhi)
11. [UMG 与 Slate 的关系](#11-umg-与-slate-的关系)

---

## 1. 模块分层与总体架构

Slate 在引擎中被拆分为多个模块，职责边界清晰，核心目标是让 **UI 逻辑（布局、事件、绘制元素生成）与具体渲染后端（RHI）解耦**。

| 模块 | 路径 | 职责 |
|------|------|------|
| **SlateCore** | `Runtime/SlateCore` | 与渲染器无关的核心：`SWidget` 基类、布局（`FGeometry`、`FChildren`、槽位）、`TAttribute`、输入事件类型、样式基类、绘制元素描述（`FSlateDrawElement`）等。不依赖任何 RHI。 |
| **Slate** | `Runtime/Slate` | 具体控件库（`SButton`、`SBoxPanel`、`STextBlock`、`SWindow` 等）与应用程序框架（`FSlateApplication`、`FEventRouter`、输入路由）。 |
| **SlateRHIRenderer** | `Runtime/SlateRHIRenderer` | RHI 渲染后端：把 `FSlateWindowElementList` 批处理（`FSlateElementBatcher`）成 `FSlateRenderBatch`，再提交给 RHI 渲染。 |
| **SlateNullRenderer** | `Runtime/SlateNullRenderer` | 空渲染器，用于无渲染的 headless 模式（如 `-nullrhi`）。 |
| **UMG** | `Runtime/UMG` | 基于 Slate 的蓝图可视化 UI 框架，用 `UWidget`/`UUserWidget` 封装 `SWidget`。 |

**核心分层思想**：SlateCore 只产出「绘制元素列表」（`FSlateWindowElementList`），不关心如何把它们变成像素；SlateRHIRenderer 消费这个列表完成真正的 GPU 提交。这使 Slate 可以在 Windows/macOS/Linux/移动端甚至无渲染环境运行，只需替换渲染器。

**应用程序入口**：`FSlateApplication` 是单例（`FSlateApplication::CurrentApplication`），基类为 `FSlateApplicationBase`（持有 `Renderer` 指针与 `CurrentBaseApplication`）。它负责驱动整个 Slate 的 tick / 输入 / 绘制循环。

---

## 2. 控件对象模型：SWidget 层级

所有 UI 元素的基类是 `SWidget`（`SlateCore/Public/Widgets/SWidget.h`）。它定义了控件的公共接口与三个纯虚函数（布局与绘制的核心钩子）：

```cpp
class SLATECORE_API SWidget : public FSlateControlledConstruction, public TSharedFromThis<SWidget>
{
    // 三个纯虚函数 —— 每个具体控件必须实现
    virtual int32  OnPaint(const FPaintArgs&, const FGeometry&, const FSlateRect&, FSlateWindowElementList&, int32 LayerId, const FWidgetStyle&, bool bParentEnabled) const = 0;
    virtual FVector2D ComputeDesiredSize(float LayoutScaleMultiplier) const = 0;
    virtual FChildren* GetChildren() = 0;

    // 非虚的公开入口（内部调用上述虚函数 + 通用逻辑）
    int32  Paint(...) const;
    void   SlatePrepass(float LayoutScaleMultiplier);
    void   ArrangeChildren(const FGeometry& AllottedGeometry, FArrangedChildren& ArrangedChildren) const;
    // ...
};
```

`SWidget` 自身聚合了大量状态：`TSlateAttribute<EVisibility> VisibilityAttribute`、`EnabledStateAttribute`、`HoveredAttribute`、`RenderTransformAttribute`；以及若干位域标志 `bCanSupportFocus`、`bCanHaveChildren`、`bForceVolatile`、`bCachedVolatile` 等。它还持有 `FSlateWidgetPersistentState PersistentState` 与 `FWidgetProxyHandle FastPathProxyHandle`（见第 9 节）。

`SWidget` 的继承树被刻意设计为三个直接子类，分别对应三种控件形态：

```
SWidget
├── SLeafWidget          // 叶子：无子控件（bCanHaveChildren = false）
├── SPanel               // 面板：排列任意多个子控件（通过槽位）
└── SCompoundWidget      // 复合控件：恰好一个 ChildSlot
```

- **`SLeafWidget`**（`Widgets/SLeafWidget.h`）：构造函数中 `bCanHaveChildren = false`，`GetChildren()` 返回空。例如 `STextBlock`、`SImage` 这类没有子控件的"原子"控件。
- **`SPanel`**（`Widgets/SPanel.h`）：强制子类实现 `OnArrangeChildren`、`ComputeDesiredSize`、`GetChildren`。子控件存放在 `TPanelChildren<SlotType>` 中。例如 `SBoxPanel`、`SGridPanel`、`SOverlay`。
- **`SCompoundWidget`**（`Widgets/SCompoundWidget.h`）：持有单个受保护的 `ChildSlot`（`FCompoundWidgetOneChildSlot`，即 `TSingleWidgetChildrenWithBasicLayoutSlot`）。大多数复合控件由此派生，例如 `SBorder`、`SButton`。其 `ComputeDesiredSize` 默认委托给 `ChildSlot`。

**关键的「非虚接口 + 虚实现」模式**：`Paint` / `ArrangeChildren` / `SlatePrepass` 是公开非虚函数，内部完成通用工作（如裁剪、`CacheDesiredSize`、递归调用子控件），再回调子类重写的虚函数 `OnPaint` / `OnArrangeChildren` / `ComputeDesiredSize`。这种 Template Method 模式让基类能统一处理失效、裁剪、命中测试等横切逻辑。

---

## 3. 声明式语法（Declarative Syntax）内部实现

Slate 最有辨识度的特性是 C++ 声明式构建语法：

```cpp
SNew(SButton)
    .Text(LOCTEXT("OK", "OK"))
    .OnClicked(this, &FMyClass::OnOkClicked)
    .ContentPadding(FMargin(4.0, 2.0))
[
    SNew(STextBlock).Text(LOCTEXT("Label", "Button"))
];
```

这套语法的全部机制都定义在 `SlateCore/Public/Widgets/DeclarativeSyntaxSupport.h` 中。核心是三个宏与一个模板类。

### 3.1 三个入口宏

```cpp
#define SNew( WidgetType, ... ) \
    MakeTDecl<WidgetType>( #WidgetType, __FILE__, __LINE__, RequiredArgs::MakeRequiredArgs(__VA_ARGS__) ) <<= TYPENAME_OUTSIDE_TEMPLATE WidgetType::FArguments()

#define SAssignNew( ExposeAs, WidgetType, ... ) /* 类似 SNew，但额外绑定到 TSharedRef 变量 */

#define SArgumentNew( InAllocator, WidgetType, ... ) /* 用指定分配器构造 */
```

`SNew` 实际做了两件事：
1. `MakeTDecl<WidgetType>(...)` 生成一个 `TSlateDecl<WidgetType>` 临时对象（声明描述符）；
2. `<<= WidgetType::FArguments()` —— `TSlateDecl::operator<<=` 是构建的触发点。

### 3.2 `TSlateDecl` 与 `operator<<=`

`TSlateDecl<WidgetType>` 持有类型名、文件/行号（调试用）、所需参数（`RequiredArgs`）。其 `operator<<=(const FArguments&)` 的职责是：

```cpp
TSlateDecl<WidgetType>& operator<<=( const typename WidgetType::FArguments& InArgs )
{
    _Widget = SWidgetConstruct(_RequiredArgs);          // 1. 分配并构造控件实例
    InArgs.CallConstruct(_Widget, _RequiredArgs);      // 2. 调用控件的 Construct(FArguments)
    _Widget->CacheVolatility();                        // 3. 缓存易变性
    _Widget->bIsDeclarativeSyntaxConstructionCompleted = true; // 4. 标记构建完成
    return *this;
}
```

即：先通过 `WidgetType` 的 `FArguments` 携带的构造函数（通常是 `SNew` 时由 `MakeRequiredArgs` 决定的构造参数）创建对象，再调用用户控件定义的 `Construct(const FArguments&)` 完成初始化。

### 3.3 `SLATE_BEGIN_ARGS` / `SLATE_END_ARGS`

每个控件在类内部声明一个名为 `FArguments` 的嵌套结构，用于收集链式调用的所有参数：

```cpp
SLATE_BEGIN_ARGS( SButton )
    : _ButtonStyle( &FCoreStyle::Get().GetWidgetStyle<FButtonStyle>("Button") )
    , _HAlign( HAlign_Fill )
{
}
    SLATE_DEFAULT_SLOT( FArguments, Content )       // 默认槽：.Content() 或 [...]
    SLATE_STYLE_ARGUMENT( FButtonStyle, ButtonStyle )
    SLATE_ARGUMENT( EHorizontalAlignment, HAlign )
    SLATE_ATTRIBUTE( FMargin, ContentPadding )
    SLATE_EVENT( FOnClicked, OnClicked )
SLATE_END_ARGS()
```

`SLATE_BEGIN_ARGS` 生成 `struct FArguments : public TSlateBaseNamedArgs<WidgetType>`，构造函数里的 `: _ButtonStyle(...)` 是初始化成员默认值。`SLATE_END_ARGS` 生成 `CallConstruct` 等辅助函数。

**关键宏语义**：

| 宏 | 生成的成员/方法 | 用途 |
|----|----------------|------|
| `SLATE_ARGUMENT(Type, Name)` | `_Name` 成员 + `Name(...)` 赋值器 | 一次性值（如枚举、bool） |
| `SLATE_ATTRIBUTE(Type, Name)` | `TAttribute<Type> _Name` + `Name(TAttribute<Type>)` 及 `Name(Type)` 重载 | 可绑定属性（值或委托） |
| `SLATE_EVENT(Delegate, Name)` | `Delegate _Name` + `Name(...)` / `Name_Static` / `Name_Lambda` / `Name_Raw` / `Name_UObject` | 事件委托，多重重载 |
| `SLATE_STYLE_ARGUMENT(Type, Name)` | `const Type* _Name` | 指向样式的指针 |
| `SLATE_DEFAULT_SLOT(ArgsType, Name)` | 生成 `Name()` + `operator[]` | 唯一默认子槽 |
| `SLATE_NAMED_SLOT(ArgsType, Name)` | 生成 `Name()` | 具名子槽（多个） |
| `SLATE_SLOT_ARGUMENT(Type, Name)` | 槽位专用参数 | 槽位参数 |

> 注：UE5.5 中这些宏已从 `SlateOptMacros.h` 迁移到 `DeclarativeSyntaxSupport.h`；`SLATE_SUPPORTS_SLOT` 已废弃，改用 `SLATE_SLOT_ARGUMENT`。

`SLATE_EVENT` 之所以能生成多种重载，是因为它展开为多个 `Name(...)` 方法，分别接收委托、原始函数指针（`_Raw`）、lambda（`_Lambda`）、UObject 成员（`_UObject`）等，内部统一封装为对应的委托类型。

### 3.4 `FSlateBaseNamedArgs` 提供的通用参数

`FArguments` 的基类 `FSlateBaseNamedArgs` 为所有控件提供了通用参数：`ToolTip`、`Cursor`、`Visibility`、`IsEnabled`、`ForceVolatile`、`Clipping`、`PixelSnappingMethod`、`FlowDirectionPreference`、`RenderOpacity`、`RenderTransform`、`RenderTransformPivot`、`Tag`、`AccessibleText`、`MetaData` 等。因此任何控件都能 `.Visibility(...)`、`.ToolTip(...)`，无需各自重复声明。

### 3.5 子控件挂载：`operator[]`

`SNew(SVerticalBox)[ SNew(SButton) ]` 中，方括号是 `FArguments::operator[]`（由 `SLATE_DEFAULT_SLOT` 生成），它把 `[...]` 内的控件实例挂载到对应的槽位上。挂载的本质是：槽位（`FSlotArguments`）记录子控件引用，随后在父控件 `Construct` 中通过 `ChildSlot.AttachWidget(...)` 正式建立父子关系。

---

## 4. 槽位与子控件（Slots & Children）

Slate 用 **槽位（Slot）** 描述「父控件如何摆放一个子控件」。槽位既承载子控件引用，又承载布局参数（对齐、边距、大小等）。

### 4.1 槽位类层次

```
FSlotBase                                    // 基础：Owner + Widget
└── TSlotBase<SlotType>                      // CRTP：FSlotArguments 声明式构建
    └── 具体槽位，混入布局 mixin：
        TPaddingWidgetSlotMixin              // Padding 属性
        TAlignmentWidgetSlotMixin            // HAlign / VAlign 属性
```

- **`FSlotBase`**（`SlotBase.h`）：仅两个成员 `const FChildren* Owner` 与 `TSharedRef<SWidget> Widget`；提供 `AttachWidget`、`DetachWidget`、`Invalidate`。
- **`TSlotBase<SlotType>`**：通过 CRTP 提供声明式构建用的 `FSlotArguments`，其含 `operator[]`（挂载子控件）、`Expose`（暴露到变量）、`Me()`（返回 `WidgetArgsType&` 支持链式）。

`SCompoundWidget` 的 `ChildSlot` 类型 `TSingleWidgetChildrenWithBasicLayoutSlot` 就是同时混入了 Padding 与 Alignment 两个 mixin 的槽位，因此 `.Padding()`、`.HAlign()`、`.VAlign()` 可用。

### 4.2 子控件集合类层次

`FChildren` 是「子控件集合」的抽象，`GetChildren()` 返回它：

```
FChildren
├── FNoChildren                                  // 叶子控件的空集合
├── TSingleWidgetChildrenWithSlot<Slot>          // 单子控件（仅槽位，无布局）
│   └── TSingleWidgetChildrenWithBasicLayoutSlot // 单子控件（含 Padding/Alignment）
├── TPanelChildren<SlotType>                     // 面板的多槽位集合
├── TSlotlessChildren<SlotType>                  // 无槽位、固定布局的多子控件
├── FCombinedChildren                            // 组合多个集合（如 SGridPanel）
├── TWeakChild<T>                                // 弱引用包装
└── TOneDynamicChild                             // 动态单个子控件
```

`TPanelChildren` 提供 `TPanelChildrenConstIterator` 迭代器，能在遍历时根据控件的 **文本流向（RTL/LTR）** 自动调整顺序——这是 Slate 国际化支持的一部分。

### 4.3 `FArrangedChildren` 与 `FArrangedWidget`

布局阶段（`ArrangeChildren`）的产物是 `FArrangedChildren`，它是 `FArrangedWidget`（`{ TSharedRef<SWidget> Widget; FGeometry Geometry; }`）的数组。父控件在 `OnArrangeChildren` 中为每个子控件计算几何信息并 `AddWidget`。此数组随后用于绘制与命中测试，避免重复计算几何。

---

## 5. TAttribute 属性绑定

`TAttribute<T>`（`Core/Public/Misc/Attribute.h`）是 Slate 数据绑定核心：**一个属性要么是具体值，要么是一个返回该值的委托**。这让属性既能静态赋值，也能动态计算（如跟随玩家血量变化）。

```cpp
class TAttribute<ObjectType>
{
    ObjectType Value;                        // 具体值
    FGetter Getter;                          // 绑定委托（DECLARE_DELEGATE_RetVal(ObjectType, FGetter)）
    bool bIsSet;                             // 是否已赋值

    ObjectType Get() const;                  // 有 Getter 则执行，否则返回 Value
    bool IsBound() const;                    // 是否绑定了 Getter
    void Set(const ObjectType&);             // 设为具体值
    void Bind(BindType...);                  // 绑定委托
    // BindSP / BindRaw / BindUObject / BindUFunction / BindLambda / BindStatic
    static TAttribute Create(FGetter);       // 便捷工厂
    bool IdenticalTo(const TAttribute&) const; // 比较（句柄或值）
};
```

- **`Get()`**：若 `IsBound()` 为真则调用 `Getter.Execute()`，否则返回 `Value`。这是所有属性读取的统一入口。
- **`Bind*` 系列**：对应不同委托绑定目标（`BindSP` 绑定到 shared pointer 成员函数、`BindRaw` 裸指针、`BindUObject` UObject 成员、`BindUFunction` UFUNCTION、`BindLambda` lambda）。
- **`IdenticalTo()`**：比较两个属性是否「相同」——都未绑定则比较值，都绑定则比较委托句柄。用于判断属性是否真的变了（决定是否需要失效重绘）。

在 UE5.5 中，`SWidget` 上的属性还演进出了 **`TSlateAttribute` / `TSlateManagedAttribute`** 体系（`Types/SlateAttribute.h`），`TSlateAttributeRef` 作为其引用包装，支持属性注册与自动失效（见第 9 节）。

**绑定属性的易变性**：当一个属性被绑定委托后，其值可能逐帧变化，因此使用该属性的控件会被标记为 **易变（volatile）**，从而每帧重新计算/绘制（见第 9 节）。

---

## 6. 布局与绘制管线（Prepass / Arrange / Paint）

Slate 的每帧 UI 更新分三个阶段，且是 **自上而下（Prepass 自下而上缓存）** 的三步走：

```
1. Prepass     —— 自底向上计算/缓存期望尺寸（DesiredSize）
2. Arrange     —— 自顶向下分配实际几何（FGeometry）
3. Paint       —— 自顶向下收集绘制元素（DrawElements）
```

### 6.1 Prepass：`SlatePrepass` → `ComputeDesiredSize`

`SWidget::SlatePrepass(float LayoutScaleMultiplier)` 是递归函数，先递归调用所有子控件的 `SlatePrepass`，再调用自身的 `ComputeDesiredSize(LayoutScaleMultiplier)` 并把结果缓存到 `DesiredSize`（通过 `CacheDesiredSize`）。期望尺寸是「布局建议值」，子控件据此告诉父控件「我理想中需要多大」。

```cpp
// SWidget.h
void SWidget::SlatePrepass(float LayoutScaleMultiplier)
{
    // 1. 递归 prepass 子控件
    // 2. 若尺寸失效则：DesiredSize = ComputeDesiredSize(LayoutScaleMultiplier)
}
```

`ComputeDesiredSize` 是纯虚函数：
- `SLeafWidget`（如 `STextBlock`）：根据文本、字体、字号测量所需空间；
- `SPanel`：综合所有子控件的期望尺寸与布局规则（如 `SVerticalBox` 取子控件高度之和）；
- `SCompoundWidget`：默认委托给 `ChildSlot` 的期望尺寸。

### 6.2 Arrange：`ArrangeChildren` → `OnArrangeChildren`

```cpp
void SWidget::ArrangeChildren(const FGeometry& AllottedGeometry, FArrangedChildren& ArrangedChildren) const
{
    OnArrangeChildren(AllottedGeometry, ArrangedChildren);
}
```

`OnArrangeChildren` 也是纯虚函数。父控件拿到父级分配给的 `AllottedGeometry`，为每个子控件计算出「实际几何」（位置 + 尺寸 + 变换），填充到 `ArrangedChildren`。例如 `SVerticalBox` 会按子控件期望高度自顶向下切分空间。`SLeafWidget::OnArrangeChildren` 为空（无子控件），`SCompoundWidget` 默认把整个 `AllottedGeometry` 交给 `ChildSlot`。

### 6.3 Paint：`Paint` → `OnPaint`

```cpp
int32 SWidget::Paint(const FPaintArgs& Args, const FGeometry& AllottedGeometry,
                     const FSlateRect& MyCullingRect, FSlateWindowElementList& OutDrawElements,
                     int32 LayerId, const FWidgetStyle& InWidgetStyle, bool bParentEnabled) const
{
    // 通用逻辑：可见性判断、裁剪、失效检查……
    // 然后调用 OnPaint(...)
    // 再递归调用子控件的 Paint(...)
}
```

`OnPaint` 是控件生成绘制元素的钩子。控件通过 `FSlateWindowElementList::AddUninitialized<EElementType>(...)` 添加 `FSlateDrawElement`（矩形、线条、文本、图元等），这些元素的坐标是 **窗口绝对坐标**（由 `AllottedGeometry` 换算而来）。

**`LayerId`（层级 ID）**：绘制顺序由 `LayerId` 决定，值越大越靠上。子控件绘制时传入递增的 `LayerId`，从而实现兄弟/父子间的正确遮挡。控件在 `OnPaint` 返回值中给出「画完自身后」的最大 `LayerId`，供后续控件继续累加。

### 6.4 三者协同

Prepass 保证父控件在 Arrange 前已知子控件期望尺寸；Arrange 产出每个控件的 `FGeometry`；Paint 用 `FGeometry` 换算绝对坐标生成绘制元素。整个流程由 `FSlateApplication` 驱动（见第 10 节）。

---

## 7. FGeometry 几何与坐标变换

`FGeometry`（`Layout/Geometry.h`）描述一个控件在窗口空间中的位置、尺寸与变换，是布局到绘制之间传递的关键数据结构。其成员基本为 `const`，保证几何信息在 Arrange 后不可变。

```cpp
class FGeometry
{
    const FVector2f Size;                    // 本地尺寸
    const float Scale;                       // 缩放
    const FVector2f AbsolutePosition;        // 绝对位置
    const FVector2f Position;                // 相对父控件的位置
    const FSlateLayoutTransform AccumulatedRenderTransform;  // 累积渲染变换
    // ...
};
```

### 7.1 两种变换

- **`FSlateLayoutTransform`**：仅含缩放 + 平移（2D 平移 + 均匀缩放），用于布局阶段。
- **`FSlateRenderTransform`**：完整的 2D 仿射变换矩阵，用于渲染阶段（支持旋转、剪切等）。

`AccumulatedRenderTransform` 是自根到当前控件的变换累积：`Concat(LocalRenderTransform, LocalLayoutTransform, ParentAccumulatedRenderTransform)`。

### 7.2 关键方法

```cpp
static FGeometry MakeRoot(const FVector2D& InLocalSize, float InScale);       // 根（窗口）
FGeometry MakeChild(const FVector2D& InLocalSize, const FSlateLayoutTransform&) const;  // 派生子控件几何
FPaintGeometry ToPaintGeometry() const;                                       // 转成绘制用几何
FVector2D AbsoluteToLocal(FVector2D Absolute) const;   // 绝对 → 本地
FVector2D LocalToAbsolute(FVector2D Local) const;      // 本地 → 绝对
```

- `MakeRoot` 以窗口尺寸为根创建根几何；
- `MakeChild` 由父几何派生子几何（累加变换），是 Arrange 阶段的核心操作；
- `AbsoluteToLocal` / `LocalToAbsolute` 用于命中测试与坐标换算（如把鼠标屏幕坐标换算到控件本地坐标）。

`SWidget` 末尾内联了 `FGeometry::MakeChild` 的便捷重载，供控件在 `OnArrangeChildren` 中方便地派生子几何。

---

## 8. 事件路由（Event Routing）

输入事件（鼠标、键盘、触控、焦点）从平台层进入 `FSlateApplication` 后，经过一套统一的 **事件路由器（FEventRouter）** 分发到控件树。

### 8.1 路由策略

`FEventRouter`（`Slate/Private/Framework/Application/SlateApplication.cpp`）通过模板类实现四种路由策略：

| 策略 | 方向 | 用途 |
|------|------|------|
| `FBubblePolicy` | 叶子 → 根（冒泡） | 默认：事件从最深命中控件向上传递 |
| `FTunnelPolicy` | 根 → 叶子（隧道） | 预览事件（如鼠标进入/离开） |
| `FToLeafmostPolicy` | 根 → 叶子（直达最深处） | 键盘/焦点类事件 |
| `FDirectPolicy` | 直接派发到指定控件 | 已捕获鼠标的控件 |

路由沿 **`FWidgetPath`**（控件路径）进行。`FWidgetPath` 是从窗口（索引 0）到最深处叶子控件的链表，`GetWidgets()` 返回有序数组，末元素即最深控件。

- **鼠标事件**：先命中测试（`LocateWidgetInWindow` → `FindChildUnderMouse`）得到 `FWidgetPath`，再用 `FBubblePolicy` 从最深控件向根冒泡，直到某个控件返回 `FReply::Handled()`。
- **键盘/手柄事件**：沿焦点路径 `RouteAlongFocusPath` 派发（`FToLeafmostPolicy` 直达焦点控件）。

### 8.2 `FReply`

事件处理函数返回 `FReply`（`Input/Reply.h`），表达「是否已处理」以及「后续动作」：

```cpp
FReply::Handled();      // 已处理，停止冒泡
FReply::Unhandled();    // 未处理，继续冒泡
```

`FReply` 是流式构建器，可叠加多个动作：

```cpp
return FReply::Handled()
    .CaptureMouse(SharedThis(this))     // 捕获鼠标（后续鼠标事件直达本控件）
    .SetUserFocus(SharedThis(this))     // 设置焦点
    .DetectDrag(SharedThis(this), EKeys::LeftMouseButton)  // 启动拖拽检测
    .BeginDragDrop(FDragDropEvent);     // 开始拖放
```

这些动作在路由结束后由 `FSlateApplication` 统一执行（捕获、焦点、拖拽都是「路由后处理」）。

### 8.3 命中测试

`LocateWidgetInWindow` 把屏幕坐标转换到窗口空间后，递归向下做命中测试（`FindChildUnderMouse`），根据控件的 `Visibility`、`IsEnabled`、裁剪区域与几何信息判断鼠标落在哪个子控件上，最终返回一个 `FWidgetPath`。命中测试是每个鼠标事件路由的前置步骤。

---

## 9. 失效与快速路径（Invalidation & Fast Path）

Slate 的性能优化核心是 **脏区域 / 失效驱动**：只有属性真正改变的控件才重新计算布局或重绘，未变化的控件直接复用上一帧缓存的绘制元素（快速路径）。这套机制由「失效根（Invalidation Root）+ 控件代理（Widget Proxy）+ 持久化绘制状态 + 缓存元素列表」四个部分构成。

### 9.1 `EInvalidateWidgetReason`

`SlateCore/Public/Widgets/InvalidateWidgetReason.h` 定义失效原因标志位（`ENUM_CLASS_FLAGS`，可组合）：

```cpp
enum class EInvalidateWidgetReason : uint8
{
    None                 = 0,
    Layout               = 1 << 0,  // 期望尺寸/排列变化（最昂贵，触发 Prepass）
    Paint                = 1 << 1,  // 仅绘制内容改变，不影响尺寸
    Volatility           = 1 << 2,  // 易变性变化
    ChildOrder           = 1 << 3,  // 增删子控件（隐含 Prepass + Layout）
    RenderTransform      = 1 << 4,  // 渲染变换变化
    Visibility           = 1 << 5,  // 可见性变化（隐含 Layout）
    AttributeRegistration= 1 << 6,  // SlateAttribute 绑定/解绑
    Prepass              = 1 << 7,  // 递归重算子级期望尺寸（隐含 Layout）

    PaintAndVolatility   = Paint | Volatility,
    LayoutAndVolatility  = Layout | Volatility,
};
```

关键语义：`Layout` 最昂贵；`ChildOrder`、`Visibility`、`Prepass` 会在 `SWidget::Invalidate` 中被隐式升级为 `Layout`。

### 9.2 失效根（Invalidation Root）

**`FSlateInvalidationRoot`**（`FastUpdate/SlateInvalidationRoot.h`）是失效传播的**边界**：widget 失效只向同一个失效根内的祖先传播，不跨出失效根。这样每帧只需对失效子树做增量更新，而非全窗口重走 Prepass + Paint。

每个失效根持有：
- `FastWidgetPathList`（`FSlateInvalidationWidgetList`）：按深度优先排序的、这颗子树所有 widget 的 `FWidgetProxy` 扁平数组；
- 三个待处理堆：`WidgetsNeedingPreUpdate` / `WidgetsNeedingPrepassUpdate` / `WidgetsNeedingPostUpdate`；
- `CachedElementData`（`FSlateCachedElementData`）：缓存绘制元素；
- `bNeedsSlowPath`：是否需要回退慢路径全量重建。

子类实现两个纯虚函数 `GetRootWidget()`（管辖的根 widget）与 `PaintSlowPath()`。UE5.5 中有三种具体失效根：

| 类 | 位置 | 成为失效根的条件 |
|---|---|---|
| `SInvalidationPanel` | `Slate/Public/Widgets/SInvalidationPanel.h` | `Advanced_IsInvalidationRoot()` 返回 `GetCanCache()` |
| `SRetainerWidget` | `UMG/Public/Slate/SRetainerWidget.h` | `bEnableRetainedRendering` |
| `SWindow` | `SlateCore/Private/Widgets/SWindow.cpp` | `bAllowFastUpdate && GSlateEnableGlobalInvalidation`（全局开关） |

三者都同时继承 `SCompoundWidget` 与 `FSlateInvalidationRoot`。每个失效根还有独立的 `FHittestGrid`（嵌套命中测试网格）。所有失效根注册在全局 `GSlateInvalidationRootListInstance`，并订阅 `FSlateApplicationBase::OnInvalidateAllWidgets` 事件（全局失效时 `HandleInvalidateAllWidgets` 会 `ResetInvalidation` + `bNeedsSlowPath = true`）。

### 9.3 快速路径与 `FWidgetProxy`

慢路径每帧对所有 widget 执行 `Prepass` + `Paint`。快速路径则用一次慢路径建立扁平 `FWidgetProxy` 列表并缓存绘制状态，之后每帧**只重画被标记为脏的 widget**，其余直接复用缓存。

- **`FWidgetProxy`**（`FastUpdate/WidgetProxy.h`）：每个 widget 一个约 32 字节的代理，含 `Widget`、`Index`/`ParentIndex`/`LeafMostChildIndex`（扁平数组位置与子树范围）、`CurrentInvalidateReason`（累积失效原因）、`Visibility`（自身+祖先可见/折叠状态，压缩为 1 字节）等。核心方法 `Repaint()` 用缓存信息直接 `Paint()`，跳过慢路径的逐帧推导。
- **`FSlateWidgetPersistentState`**（`WidgetProxy.h`）：缓存「上次 Paint 时」的全部上下文——`AllottedGeometry`、`DesktopGeometry`、`CullingBounds`、`WidgetStyle`、`CachedElementHandle`、`LayerId`/`OutgoingLayerId`、`InitialClipState`、`IncomingFlowDirection` 等。存在 `SWidget::PersistentState`。
- **每帧流程**：`PaintInvalidationRoot` 若 `bNeedsSlowPath` 则 `ClearAllFastPathData` + `BuildFastPathWidgetList` + `PaintSlowPath`；否则 `PaintFastPath`，只遍历 `FinalUpdateList` 中需要更新的 widget。`ProcessInvalidation` 按 `ProcessPreUpdate → ProcessAttributeUpdate → ProcessPrepassUpdate → ProcessPostUpdate` 分拣，把需要重画的 widget 排序后塞入 `FinalUpdateList`。

### 9.4 失效的增量传播

`SWidget::Invalidate(EInvalidateWidgetReason)` 是入口（`SWidget.cpp`）：先做标志位升级（Volatility→Paint、Prepass→Layout、ChildOrder→Prepass|Layout），然后更新易变性缓存；若在失效根内则 `FastPathProxyHandle.MarkWidgetDirty_NoCheck`，否则走传统慢路径。

**注意：失效并不是立即向上冒泡**。`FWidgetProxy::ProcessLayoutInvalidation` 才是向上传播的关键，且是**增量式**的：

```cpp
// 重新 prepass 计算期望尺寸
const FVector2f NewDesiredSize = WidgetPtr->GetDesiredSize();
// 仅当期望尺寸变化（或 RenderTransform 变化）时才失效父级
if (NewDesiredSize != CurrentDesiredSize || HasAnyFlags(CurrentInvalidateReason, RenderTransform))
{
    if (ParentIndex == FirstIndex())  Root.InvalidateRootLayout(...);   // 已是失效根根节点 → 整根走慢路径
    else { ParentProxy.CurrentInvalidateReason |= Layout; ... }        // 父级加 Layout，继续向上
}
```

即：纯 `Paint` 失效不改变尺寸，就不需要传染父级；只有尺寸/layout 变化才逐级向上传播 `Layout`，直到失效根顶（触发 `InvalidateRootLayout` → `bNeedsSlowPath`）或失效根外（转传统 `Invalidate`）。

### 9.5 易变性（Volatility）

易变 widget 是**每帧必须重画、且不依赖显式 Invalidate 调用**的控件（动画、进度条、时钟等）。易变控件的绘制结果**不参与缓存**。

- `ComputeVolatility()`（`SWidget.h:1581`）默认返回 `false`，子类覆写判断自身是否易变；
- `CacheVolatility()`：`bCachedVolatile = bForceVolatile || ComputeVolatility()`；
- `ForceVolatile(bool)`：强制易变，改变时 `Invalidate(PaintAndVolatility)`；
- `IsVolatileIndirectly()`：`bInheritedVolatility` 表示某祖先易变（易变向下传播）。

`TSlateDecl::operator<<=` 末尾的 `CacheVolatility()` 即在构建时预计算缓存。`UpdateFastPathVolatility` 递归把「父易变 || 自身易变」传给所有子级，并维护易变控件索引表（`ElementIndexList_VolatileUpdateWidget`），供每帧快速枚举。

### 9.6 缓存元素列表

缓存由三个类分层管理：

- **`FSlateCachedElementList`**（`DrawElements.h:53`）：**每个 widget 一份**，含各类型元素容器、`OwningWidget`、`CachedRenderingData`（顶点/索引缓冲）、预生成的 `CachedBatches`；
- **`FSlateCachedElementData`**（`DrawElements.h:145`）：**每个失效根一份**，管理 `CachedBatches`、`CachedElementLists`、`ListsWithNewData`、`CachedClipStates`；
- **`FSlateCachedElementsHandle`**（`DrawElements.h:111`）：widget 指向其缓存列表的弱句柄，存在 `PersistentState.CachedElementHandle`。

绘制时 `FSlateWindowElementList::AddUninitialized` 在允许缓存时经 `FSlateCachedElementData::AddCachedElement` 把元素写入缓存列表；`UpdateWidgetProxy` 在 Paint 后更新句柄与 `OutgoingLayerId`。后续帧未失效的 widget 直接复用其 `CachedBatches` 提交 GPU，**连 `OnPaint` 都不调用**。

### 9.7 缓存 / 批次的 key

`FSlateRenderBatchParams::IsBatchableWith`（`SlateRenderBatch.h:30`）是批次合并与缓存的判据，要求 `Layer`、`ShaderParams`、`Resource`、`PrimitiveType`、`ShaderType`、`DrawEffects`、`DrawFlags`、`SceneIndex`、`ClippingState` 全部相等才能合并——这也解释了「裁剪区不同就不能合并批次」。

---

## 10. 渲染管线（DrawElements → Batcher → RHI）

### 10.1 每帧驱动流程

```
FSlateApplication::Tick(ESlateTickType)
  ├── TickPlatform            (PlatformAndInput)：泵消息、处理输入
  ├── TickTime                (Time)：推进时间
  └── TickAndDrawWidgets      (Widgets)：
        ├── 节流/休眠判断（无输入且无活动定时器则休眠，节省开销）
        └── DrawWindows()
              └── PrivateDrawWindows()
                    ├── DrawPrepass()                    // 对所有可见窗口 SlatePrepass
                    ├── DrawWindowAndChildren(...)        // 每个窗口 PaintWindow 收集绘制元素
                    │     └── WindowToDraw->PaintWindow(...)  → 递归 SWidget::Paint
                    └── Renderer->DrawWindows(OutDrawBuffer)  // 交给渲染后端
```

关键点：
- **`ESlateTickType`**：`Time` / `PlatformAndInput` / `Widgets`（可组合 `TimeAndWidgets`、`All`）。引擎循环通常以 `All` 调用，电影线程可能只调用 `Time`。
- **活动定时器（Active Timers）**：`RegisterActiveTimer` 注册的定时器（如动画、异步轮询）会阻止 Slate 休眠（`bIsSlateAsleep`），因为即使无用户输入，这些定时器仍需驱动 UI 更新。
- **`DrawPrepass`**（`SlateApplication.cpp:1287`）：遍历窗口，对每个窗口调用 `ProcessWindowInvalidation` 处理失效，然后 `SlatePrepass`。它还会处理模态窗口、置顶窗口、通知窗口、自动尺寸调整。
- **`DrawWindowAndChildren`**（`SlateApplication.cpp:1136`）：对每个可见窗口调用 `PaintWindow` 递归绘制，产出该窗口的 `FSlateWindowElementList`；随后绘制拖放内容、软件光标、Widget Reflector 调试信息。
- **`PrivateDrawWindows`**（`SlateApplication.cpp:1345`）：Prepass + 绘制 + 清理已移除窗口 + `Renderer->DrawWindows` 提交。

### 10.2 绘制元素与批处理

绘制阶段 SlateCore 只产出「绘制元素描述」，由 SlateRHIRenderer 消费：

- **`FSlateWindowElementList`**（`Rendering/DrawElements.h:225`）：每个窗口一份，由 `FSlateDrawBuffer::AddWindowElementList` 创建。提供 `AddUninitialized<EElementType>`、`PushClip`/`PopClip`（裁剪栈）、`PushPixelSnappingMethod`（像素对齐栈）、`QueueDeferredPainting`、`PushPaintingWidget`/`PopPaintingWidget`（控件绘制栈，供缓存）、`PushCachedElementData`。

  元素**按类型分组存储**而非单一数组：核心结构 `FSlateDrawElementMap`（`DrawElementCoreTypes.h:158`）是一个 `TTuple`，每个槽位对应一种 `EElementType`，槽内是 `FSlateDrawElementArray<T>`。这样设计是为了缓存友好与按类型批量处理。

- **`EElementType` 枚举**（`DrawElementCoreTypes.h:34`）及对应 payload：

  | 类型 | payload | 用途 |
  |------|---------|------|
  | `ET_Box` | `FSlateBoxElement` | 纹理盒/图片（9-slice） |
  | `ET_Text` / `ET_ShapedText` | `FSlateTextElement` / `FSlateShapedTextElement` | 文本 / 已 shaping 字形 |
  | `ET_Line` | `FSlateLineElement` | 折线/虚线 |
  | `ET_Spline` | `FSlateSplineElement` | Hermite/Bezier 曲线 |
  | `ET_Gradient` | `FSlateGradientElement` | 渐变 |
  | `ET_Viewport` | `FSlateViewportElement` | 3D 场景 viewport |
  | `ET_Border` / `ET_RoundedBox` | `FSlateBoxElement` / `FSlateRoundedBoxElement` | 边框 / 圆角盒 |
  | `ET_Custom` / `ET_CustomVerts` | `FSlateCustomDrawerElement` / `FSlateCustomVertsElement` | 自定义绘制 |
  | `ET_PostProcessPass` | `FSlatePostProcessElement` | 后处理（blur 等） |

  入口是 `FSlateDrawElement::Make*` 静态函数（`MakeBox`/`MakeText`/`MakeLines`/`MakeSpline`/`MakeViewport`/`MakeCustom` 等）。`MakeBoxInternal` 会根据 brush 的 `DrawAs` 自动选择 `ET_Border`/`ET_RoundedBox`/`ET_Box`。

  每个元素基类还保存 `RenderTransform`、`Position`、`LocalSize`、`Scale`、`DrawEffects`、`ClipStateHandle`（裁剪句柄）、`SceneIndex`、`BatchFlags`。`SceneIndex` 用于区分同一窗口内不同 3D 场景 viewport。

### 10.3 `FSlateElementBatcher`：元素 → 渲染批次

`FSlateElementBatcher::AddElements`（`ElementBatcher.cpp:269`）是批处理入口：

1. 处理 uncached 元素（`AddElementsInternal`）：按元素类型逐个遍历，调用对应的顶点生成函数（`AddBoxElements`/`AddTextElement`/`AddLineElements`/`AddGradientElement`/`AddViewportElement` 等）；
2. 处理 cached 元素（`AddCachedElements`）：失效 widget 重新生成批次，未失效的缓存批次直接追加进 `RenderBatches`；
3. `StartMergeRenderBatches` 触发合并。

通用批量模板 `GenerateIndexedVertexBatches` 用三个仿函数（`InElementBatchParamCreator` 生成批次 key、`InElementAdder` 写顶点/索引、`InElementBatchReserver` 预分配容量）：对连续元素段用 `IsBatchableWith` 判断能否并入同批，能则合并，否则新建 `FSlateRenderBatch`。

**批次合并判据**（`FSlateRenderBatchParams::IsBatchableWith`，`SlateRenderBatch.h:30`）：`Layer`、`ShaderParams`、`Resource`、`PrimitiveType`、`ShaderType`、`DrawEffects`、`DrawFlags`、`SceneIndex`、`ClippingState` 全部相等才能合并。跨批次最终合并发生在 `FSlateBatchData::MergeRenderBatches`（`ElementBatcher.cpp:141`），它按 `Layer` 对批次 `StableSort`，并 `CombineBatches` 重写索引偏移后拼到 `FinalVertexData`/`FinalIndexData`。

### 10.4 渲染器抽象与 RHI 提交

`FSlateRenderer`（`SlateCore/Public/Rendering/SlateRenderer.h`）定义渲染器全部职责：`AcquireDrawBuffer`/`ReleaseDrawBuffer`、`DrawWindows(FSlateDrawBuffer&)`、`CreateViewport`、`GetResourceHandle`、`GetResourceCriticalSection` 等。`FSlateRHIRenderer`（`SlateRHIRenderer/Private/SlateRHIRenderer.h`）是其 RHI/GPU 实现，关键特性：

- **三缓冲 `DrawBuffers[3]`**：实现游戏线程填充、渲染线程消费的并行；
- `WindowToViewportInfo`（窗口→RHI viewport 映射）、`ElementBatcher`、`RenderingPolicy`、`ResourceManager`。

**从 `DrawWindows` 到 RHI DrawCall 的完整链路**：

```
FSlateApplication::Tick → DrawWindows() → PrivateDrawWindows()
  ├─ DrawPrepass()                                  // 布局/测量
  ├─ FScopedAcquireDrawBuffer{...}                  // 获取三缓冲之一并加锁
  ├─ DrawWindowAndChildren(...)                     // 递归遍历窗口
  │    └─ WindowToDraw->PaintWindow(...)            // 递归 SWidget::Paint
  │         └─ FSlateDrawElement::Make* → AddUninitialized<EElementType>
  │              └─ 元素 Init() 记录 LayerId / ClipStateHandle / SceneIndex
  └─ Renderer->DrawWindows(OutDrawBuffer)           // FSlateRHIRenderer::DrawWindows
       └─ DrawWindows_Private → ElementBatcher->AddElements(...)
            └─ AddElementsInternal / AddCachedElements → GenerateIndexedVertexBatches
            └─ StartMergeRenderBatches → MergeRenderBatches (按 Layer 排序 + 跨批合并)
       └─ ENQUEUE_RENDER_COMMAND(SlateDrawWindowsCommand)   // 投递渲染线程
            └─ DrawWindows_RenderThread → FRDGBuilder
                 └─ DrawWindow_RenderThread
                      ├─ BuildSlateElementsBuffers (顶点/索引 → FRDGBuffer)
                      └─ AddSlateDrawElementsPass (RDG "ElementBatch" pass)
                           └─ DrawSlateRenderBatch:
                                ├─ SetSlateClipping (scissor / stencil)
                                ├─ SetGraphicsPipelineState (PSO)
                                ├─ SetStreamSource (顶点缓冲)
                                └─ DrawIndexedPrimitive ← 最终 RHI DrawCall
```

### 10.5 裁剪：Scissor 与 Stencil

`EWidgetClipping`（`Layout/Clipping.h`）提供 `Inherit`、`ClipToBounds`、`ClipToBoundsWithoutIntersecting`、`ClipToBoundsAlways`、`OnDemand`（文本溢出优化）。`FSlateClippingManager` 在绘制中维护裁剪栈（`PushClip`/`PopClip`），元素记录 `FClipStateHandle`。

`FSlateClippingState` 有两种裁剪方式：
- **轴对齐 → Scissor**：`RHICmdList.SetScissorRect`，简单高效；
- **非轴对齐 → Stencil**：先用 `FSlateMaskingVS/PS` 着色器把裁剪四边形写入 stencil（第一个 quad `SO_Replace`，后续 `SO_SaturatedIncrement`），绘制时用 `CF_Equal` 测试，只有 stencil 值等于所有裁剪区交集的像素才通过。

因为裁剪态是批次 key 的一部分（`ClippingState`），裁剪区不同就无法合并批次，裁剪态切换还会导致 RDG pass 被 flush。

---

## 11. UMG 与 Slate 的关系

UMG（`Runtime/UMG`）是面向蓝图/编辑器的可视化 UI 框架，它是 Slate 的 **上层封装**，而非替代。

### 11.1 分层与角色分工

```
SlateCore → Slate → SlateRHIRenderer / SlateNullRenderer → UMG
```

- `SWidget`（`SlateCore/Public/Widgets/SWidget.h:153`）是**纯 C++ 类**，继承 `FSlateControlledConstruction` 与 `TSharedFromThis<SWidget>`，由引用计数管理，不参与 GC，蓝图无法直接操作。
- `UWidget`（`UMG/Public/Components/Widget.h:214`）是 `UCLASS(Abstract, BlueprintType, Blueprintable)`，属于 UObject 体系，可被蓝图继承/实例化、编辑器可视化编辑、参与 GC、拥有 `UPROPERTY`/`UFUNCTION`/字段通知等反射能力。

`UWidget` 内部持有两个指向底层 `SWidget` 的弱引用（`Widget.h:1170`）：

```cpp
TWeakPtr<SWidget>       MyWidget;   // 底层真正的 Slate 控件
TWeakPtr<SObjectWidget> MyGCWidget; // 包在 SObjectWidget 里的 GC 容器（仅 UserWidget）
```

### 11.2 `TakeWidget()` / `RebuildWidget()` / `SynchronizeProperties()`

- **`TakeWidget()`**（`Widget.cpp:954`）：外部获取底层 `SWidget` 的统一入口，委托给 `TakeWidget_Private`。
- **`TakeWidget_Private`**（`Widget.cpp:977`）：若 `MyWidget` 无效则调 `RebuildWidget()` 构造并缓存；`UUserWidget` 会被包进 `SObjectWidget` 并缓存到 `MyGCWidget`（防 GC）；首次构建后依次执行 `SynchronizeProperties()` 与 `OnWidgetRebuilt()`。
- **`RebuildWidget()`**（`Widget.cpp:1385`）：基类仅 `ensureMsgf(false)` 兜底返回 `SSpacer`，**每个 UWidget 子类必须重写**，用 `SNew(...)` 构造对应 `SWidget`（如 `UButton::RebuildWidget` 返回 `SNew(SButton)`）——这就是「UWidget 绑定到 SWidget」的具体动作。
- **`SynchronizeProperties()`**（`Widget.cpp:1391`）：把 UWidget 的 `UPROPERTY` 同步到底层 `SWidget`。用 `PROPERTY_BINDING`/`BITFIELD_PROPERTY_BINDING` 等宏（`Widget.h:108`）把 UMG 属性包成 Slate 的 `TAttribute`，从而支持运行时动态绑定（蓝图 delegate）而非一次性拷贝。它同步 `SetEnabled`、`SetVisibility`、`SetClipping`、`SetRenderOpacity`、`ForceVolatile`、`UpdateRenderTransform`、ToolTip、可访问性等。

### 11.3 `SObjectWidget`：桥接 SWidget 与 UObject

`SObjectWidget`（`UMG/Public/Slate/SObjectWidget.h:30`）是桥接核心：

```cpp
class SObjectWidget : public SCompoundWidget, public FGCObject
```

它同时是 Slate 复合控件与 **`FGCObject`**：
- 成员 `TObjectPtr<UUserWidget> WidgetObject`；
- 实现 `AddReferencedObjects`（`SObjectWidget.cpp:102`）把 `WidgetObject` 注册进 GC——只要 `SWidget` 活着，`UUserWidget` 就不会被回收；
- `ResetWidget`（`SObjectWidget.cpp:53`）：析构/移除时 `UnregisterGCObject → NativeDestruct → ReleaseSlateResources`，确保资源即时释放。

更重要的是，`SObjectWidget` 把 Slate 事件**回传**给 UObject：`Tick`→`NativeTick`、`OnPaint`→`NativePaint`、`OnMouseButtonDown`→`NativeOnMouseButtonDown`、`OnKeyDown`→`NativeOnKeyDown` 等。所有事件先经 `CanRouteEvent()`（内部调 `CanSafelyRouteEvent`）在蓝图编译/GC/PostLoad 等不安全时机屏蔽回调。**`SObjectWidget` 就是 Slate 事件与 UObject 蓝图事件之间的适配器**——`UUserWidget` 能收到 `NativeTick`/`NativeOnMouseButtonDown` 全赖于此。

### 11.4 WidgetTree 与 WidgetComponent

- **`UWidgetTree`**（`UMG/Public/Blueprint/WidgetTree.h:19`）：继承 `UObject, INamedSlotInterface`，是 Widget 蓝图的**数据模型**——一套纯 UObject 父子关系（`RootWidget`、`NamedSlotBindings`），可序列化/编辑/蓝图遍历；实际渲染由每个 `UWidget::TakeWidget` 递归生成的 `SWidget` 树完成。提供 `FindWidget`、`RemoveWidget`、`GetAllWidgets`、`ForEachWidget`、`ConstructWidget<T>` 等 API。
- **`UWidgetComponent`**（`UMG/Public/Components/WidgetComponent.h:94`）：继承 `UMeshComponent`，把 UI 放进 **3D 世界（世界空间 UI）**——先把 widget 渲染到 render target，再作为 mesh 纹理显示在世界中；依赖 `FWidgetRenderer`、`SVirtualWindow`、`UTextureRenderTarget2D`，支持 `EWidgetSpace::World/Screen`。

### 11.5 模块依赖

| 模块 | 关键依赖 | 说明 |
|------|---------|------|
| SlateCore | Core、CoreUObject、InputCore、FreeType/ICU/HarfBuzz/Nanosvg | **不依赖 RHI/RenderCore**，纯逻辑与渲染抽象 |
| Slate | SlateCore、InputCore、ImageWrapper | 控件库 + 框架，同样不依赖 RHI |
| SlateRHIRenderer | Slate、SlateCore、RHI、RenderCore、Renderer、Engine | `FSlateRenderer` 的 GPU 实现 |
| SlateNullRenderer | Core、SlateCore | 空渲染器（server/headless） |
| UMG | Slate、SlateCore、Renderer、RHI、SlateRHIRenderer | 建立在 Slate 之上 |

依赖层次：`UMG → Slate + SlateCore → SlateCore`；渲染由 `SlateRHIRenderer`（真实）或 `SlateNullRenderer`（空）通过 `FSlateRenderer` 抽象注入，保证 Slate 与渲染后端解耦。

### 11.6 定位

**Slate 是地基，UMG 是面向内容作者的舒适层**：

- 纯 C++ / 需要极致性能或深度定制的 UI 可直接用 Slate；
- 需要蓝图可视化编辑、设计师友好、运行时动态编辑的 UI 用 UMG；
- UMG 的所有布局、绘制、事件最终都落到 Slate 的机制上（本文前述全部内容对 UMG 同样适用）。

---

## 附：核心文件索引

| 主题 | 文件 |
|------|------|
| 控件基类 | `SlateCore/Public/Widgets/SWidget.h` |
| 声明式语法宏 | `SlateCore/Public/Widgets/DeclarativeSyntaxSupport.h` |
| 槽位基类 | `SlateCore/Public/SlotBase.h` |
| 子控件集合 | `SlateCore/Public/Layout/Children.h` |
| 基础布局槽位 | `SlateCore/Public/Layout/BasicLayoutWidgetSlot.h` |
| 几何 | `SlateCore/Public/Layout/Geometry.h` |
| 属性绑定 | `Core/Public/Misc/Attribute.h` |
| 绘制元素 | `SlateCore/Public/Rendering/DrawElements.h` |
| 失效原因 | `SlateCore/Public/Widgets/InvalidateWidgetReason.h` |
| 事件回复 | `SlateCore/Public/Input/Reply.h` |
| 控件路径 | `SlateCore/Public/Layout/WidgetPath.h` |
| 应用基类 | `SlateCore/Public/Application/SlateApplicationBase.h` |
| 应用实现 | `Slate/Public/Framework/Application/SlateApplication.h` |
| 事件路由器 | `Slate/Private/Framework/Application/SlateApplication.cpp`（`FEventRouter`） |
| 具体控件示例 | `Slate/Public/Widgets/Input/SButton.h` |
