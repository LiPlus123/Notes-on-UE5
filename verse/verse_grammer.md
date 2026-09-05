# UE6 Verse 语言语法参考（基于引擎编译器源码）

> 版本：UE6 main（uLang / VerseCompiler）
> 源码依据：
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Parser/VerseGrammar.h`（完整语法/词法/优先级）
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Parser/ReservedSymbols.inl`（保留字）
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Semantics/Effects.h`（效果系统）
> - `Engine/Source/Runtime/VerseCompiler/Private/uLang/SemanticAnalyzer/SemanticAnalyzer.cpp`（宏/语义）
> - `Engine/Plugins/EntityFramework/Source/**/Verse/*.native.verse`（引擎标准库真实代码）
> 目标读者：想在 UE6 中写 Verse 脚本 / 阅读 Verse 标准库的开发者
>
> ⚠️ 说明：本文语法全部来自 UE6 编译器源码（uLang）与引擎自带的 `.verse` 标准库，不依赖网络文档。UE6 的 Verse 是 UEFN Verse 的直接继承者，但语法细节以本文（= 当前 main 分支源码）为准。

---

## 目录

- [UE6 Verse 语言语法参考（基于引擎编译器源码）](#ue6-verse-语言语法参考基于引擎编译器源码)
  - [目录](#目录)
  - [1. Verse 是什么：一句话定位](#1-verse-是什么一句话定位)
  - [2. Verse 的数学原理：类型化 λ 演算](#2-verse-的数学原理类型化-λ-演算)
    - [2.1 概述：Verse 类型系统 = System F: + 效果类型 + 极性子类型](#21-概述verse-类型系统--system-f--效果类型--极性子类型)
    - [2.2 从无类型 λ 演算到 Verse 表达式](#22-从无类型-λ-演算到-verse-表达式)
      - [2.2.1 λ 项的三条规则 → Verse 的表达式](#221-λ-项的三条规则--verse-的表达式)
      - [2.2.2 β-规约 → Verse 的求值](#222-β-规约--verse-的求值)
    - [2.3 简单类型 λ 演算 (λ→) → Verse 的 `CFunctionType`](#23-简单类型-λ-演算-λ--verse-的-cfunctiontype)
      - [2.3.1 箭头类型](#231-箭头类型)
      - [2.3.2 类型安全：Subject Reduction](#232-类型安全subject-reduction)
    - [2.4 System F (λ2) → Verse 的泛型/多态](#24-system-f-λ2--verse-的泛型多态)
      - [2.4.1 类型抽象与类型应用](#241-类型抽象与类型应用)
      - [2.4.2 编译器中的类型变量与实例化](#242-编译器中的类型变量与实例化)
      - [2.4.3 Verse 的类型构造器](#243-verse-的类型构造器)
    - [2.5 System F: → Verse 的子类型约束](#25-system-f--verse-的子类型约束)
      - [2.5.1 有界量化](#251-有界量化)
      - [2.5.2 子类型格操作](#252-子类型格操作)
    - [2.6 极性子类型 (Polarity) → 协变与逆变](#26-极性子类型-polarity--协变与逆变)
      - [2.6.1 函数类型的子类型规则](#261-函数类型的子类型规则)
      - [2.6.2 编译器中的极性实现](#262-编译器中的极性实现)
    - [2.7 效果类型 λ 演算 → Verse 的效果系统](#27-效果类型-λ-演算--verse-的效果系统)
      - [2.7.1 效果标注的箭头类型](#271-效果标注的箭头类型)
      - [2.7.2 效果格](#272-效果格)
      - [2.7.3 与 Haskell Monad 的关系](#273-与-haskell-monad-的关系)
    - [2.8 `CTypeType`：以子类型区间编码"类型的类型"](#28-ctypetype以子类型区间编码类型的类型)
    - [2.9 理论层次总览](#29-理论层次总览)
    - [2.10 为什么选择 λ 演算作为理论基础](#210-为什么选择-λ-演算作为理论基础)
  - [3. 词法结构（Lexical Structure）](#3-词法结构lexical-structure)
    - [3.1 字符集与空白](#31-字符集与空白)
    - [3.2 注释](#32-注释)
    - [3.3 标识符](#33-标识符)
    - [3.4 数字字面量](#34-数字字面量)
    - [3.5 字符字面量](#35-字符字面量)
    - [3.6 字符串字面量与插值](#36-字符串字面量与插值)
    - [3.7 路径字面量 / 模块引用](#37-路径字面量--模块引用)
  - [4. 运算符与优先级](#4-运算符与优先级)
    - [4.1 完整 Token 表](#41-完整-token-表)
    - [4.2 优先级层级（EPrec）](#42-优先级层级eprec)
    - [4.3 常见错误提示（编译器引导）](#43-常见错误提示编译器引导)
  - [5. 类型系统](#5-类型系统)
    - [5.1 基本类型](#51-基本类型)
    - [5.2 Option 与失败类型](#52-option-与失败类型)
    - [5.3 容器类型](#53-容器类型)
    - [5.4 函数类型](#54-函数类型)
    - [5.5 用户自定义类型](#55-用户自定义类型)
  - [6. 块（Block）的三种写法](#6-块block的三种写法)
  - [7. 声明语法](#7-声明语法)
    - [7.1 变量：不可变 / 可变 / 赋值](#71-变量不可变--可变--赋值)
    - [7.2 函数声明](#72-函数声明)
    - [7.3 类 class](#73-类-class)
    - [7.4 结构体 struct](#74-结构体-struct)
    - [7.5 接口 interface](#75-接口-interface)
    - [7.6 枚举 enum](#76-枚举-enum)
    - [7.7 模块 module 与 using/import](#77-模块-module-与-usingimport)
    - [7.8 类型别名 type](#78-类型别名-type)
    - [7.9 自定义属性 attribute](#79-自定义属性-attribute)
  - [8. 属性（Attributes，`@`）](#8-属性attributes)
  - [9. 效果系统（Effects）](#9-效果系统effects)
  - [10. 控制流与并发](#10-控制流与并发)
    - [10.1 if / then / else](#101-if--then--else)
    - [10.2 for / while / loop / repeat](#102-for--while--loop--repeat)
    - [10.3 失败处理与 if 绑定](#103-失败处理与-if-绑定)
    - [10.4 并发原语](#104-并发原语)
  - [11. 综合示例：一段完整的 UE6 Verse 代码](#11-综合示例一段完整的-ue6-verse-代码)
  - [附录 A：保留字全表](#附录-a保留字全表)
  - [附录 B：Node / AST 类型速查](#附录-bnode--ast-类型速查)
    - [相关文档](#相关文档)

---

## 1. Verse 是什么：一句话定位

Verse 是 Epic 为 Unreal Engine 设计的**新编程语言**，由 Simon Peyton Jones 参与设计（源自函数式/逻辑式编程研究，见 *The Verse Calculus*）。

- **UE6 中的地位**：Epic 在 2026 State of Unreal 中确认，UE6 中 **Blueprints 与 Actor 将逐步被 Verse + Scene Graph 取代**（Wikipedia 对该语言的记载也提到了这一官方声明）。
- **运行方式**：Verse 代码先编译为 Verse VM 字节码（`VerseVM`），由引擎运行时执行；`native` 声明直接映射到 C++ 实现。
- **语言气质**：融合了**函数式**（表达式、不可变默认、Lambda）与**命令式**（`var`/`set`、副作用）风格，加上**效果系统**（effect）做编译期副作用检查，以及**失败函数**（failing function）做类型级错误处理。

> 在 UE6 中，Verse 通过 `using { /Verse.org/SceneGraph }` 等模块与 Scene Graph、Entity/Component 系统交互（见第 11 节的综合示例；Scene Graph 的实体/组件系统见本目录 SceneGraph 系列文档）。

---

## 2. Verse 的数学原理：类型化 λ 演算

> 本节从编译器源码出发，阐明 Verse 类型系统与类型化 λ 演算（Typed Lambda Calculus）的对应关系。
> 阅读本节需要读者对 λ 演算的基本概念（λ-项、α-等价、β-规约、Church-Rosser 定理）有一定了解。
> 源码依据：
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Semantics/SemanticTypes.h`（类型系统核心：`CFunctionType`, `CTypeVariable`, `CTypeType`, `CFlowType`, `SemanticTypeUtils`）
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Semantics/TypeVariable.h`（类型变量：`CTypeVariable`）
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Semantics/SemanticFunction.h`（函数定义：`CFunction`）
> - `Engine/Source/Runtime/VerseCompiler/Public/uLang/Semantics/Effects.h`（效果系统：`SEffectSet`）

### 2.1 概述：Verse 类型系统 = System F<sub>:</sub> + 效果类型 + 极性子类型

Verse 的类型系统不是"借鉴"了 λ 演算——它**直接就是** λ 演算的工程化实现。编译器源码中 `CTypeBase` → `CNormalType` → `CFunctionType` / `CTypeVariable` / `CTypeType` 的类型层次，精确对应了从简单类型 λ 演算 (λ→) 到 System F<sub>:</sub> 的理论演进。

核心对应关系：

| 理论层级 | 数学形式 | Verse 语法 | 编译器实现 |
|---|---|---|---|
| 简单类型 λ 演算 (λ→) | $\sigma \to \tau$ | `type{_(X:int):float}` | `CFunctionType` |
| System F (λ2) | $\forall \alpha.\ \sigma$ | `[]T`、`option(T)` | `CTypeVariable` + `Instantiate()` |
| System F<sub>:</sub> | $\forall \alpha <: \sigma.\ \tau$ | `T:subtype(U)` | `CTypeVariable::_NegativeType/_PositiveType` |
| 效果类型 λ 演算 | $\sigma \xrightarrow{E} \tau$ | `<suspends>`、`<transacts>` | `SEffectSet` in `CFunctionType` |
| 极性子类型 | 协变/逆变 | 自动推断 | `ETypePolarity` + `CFlowType` |

Verse 的设计者 Simon Peyton Jones 是 GHC（Haskell 编译器）和 System FC（GHC 的中间语言，基于 System F 扩展）的作者。Verse 的类型系统可以理解为 **System FC 在游戏引擎领域的"实用主义分支"**——保留了类型安全、参数化多态和有界量化，同时加入了效果系统来管理游戏引擎中不可避免的副作用（内存分配、网络同步、物理模拟等）。

### 2.2 从无类型 λ 演算到 Verse 表达式

#### 2.2.1 λ 项的三条规则 → Verse 的表达式

λ 演算的核心语法只有三条生成规则：

1. **变元**：$x \in V$ 是 λ 项
2. **抽象**：若 $M$ 是 λ 项，$x \in V$，则 $\lambda x.M$ 是 λ 项（定义匿名函数）
3. **应用**：若 $M, N$ 是 λ 项，则 $(M\ N)$ 是 λ 项（函数调用）

Verse 中这些概念的直接对应：

```verse
# 规则 1: 变元 → Verse 的标识符引用
X := 42
Y := X          # X 是变元

# 规则 2: 抽象 → Verse 的函数定义
# λx. x + 1 对应:
AddOne := (X:int)<computes>:int = X + 1

# 规则 3: 应用 → Verse 的函数调用
# (λx. x+1) 42 对应:
Result := AddOne(42)
```

#### 2.2.2 β-规约 → Verse 的求值

β-规约是 λ 演算唯一的计算规则：

$$(\lambda x.M)\ N \longrightarrow_\beta M[x := N]$$

即：将函数体 $M$ 中所有自由出现的 $x$ 替换为实际参数 $N$。

在 Verse 中，这就是函数调用的求值过程。例如：

```verse
AddOne := (X:int)<computes>:int = X + 1
AddOne(42)  # β-规约: (X + 1)[X := 42] → 42 + 1 → 43
```

**Church-Rosser 定理（合流性）**在 Verse 中的实际意义：

> 若 $M \twoheadrightarrow_\beta M_1$ 且 $M \twoheadrightarrow_\beta M_2$，则存在 $P$ 使得 $M_1 \twoheadrightarrow_\beta P$ 且 $M_2 \twoheadrightarrow_\beta P$。

对于标记了 `<computes>` 效果的 Verse 函数（纯函数，无副作用），其求值满足合流性——无论编译器按什么顺序优化/求值，最终结果一致。然而，标记了 `<transacts>` 的函数（有副作用）**打破**了合流性，因为副作用引入了求值顺序依赖。这正是 Verse 效果系统的核心动机：**在类型层面区分"可安全重排"的纯代码和"顺序敏感"的有效果代码**。

### 2.3 简单类型 λ 演算 (λ→) → Verse 的 `CFunctionType`

#### 2.3.1 箭头类型

简单类型 λ 演算 (STLC, Simply Typed Lambda Calculus) 为每个 λ 项赋予一个类型。核心规则是**箭头类型引入**：

$$\frac{\Gamma, x:\sigma \vdash M : \tau}{\Gamma \vdash (\lambda x.M) : \sigma \to \tau}$$

其中 $\Gamma$ 是类型上下文（typing context），$\sigma \to \tau$ 表示"从 $\sigma$ 到 $\tau$ 的函数类型"。

在 Verse 编译器源码中，这个箭头类型由 `CFunctionType`（[SemanticTypes.h:1077-1161](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1077-L1161)）精确建模：

```cpp
class CFunctionType : public CNormalType {
    const CTypeBase* _ParamsType;   // 参数类型  → 对应 σ
    const CTypeBase& _ReturnType;   // 返回类型  → 对应 τ
    SEffectSet       _Effects;      // 效果标注  → Verse 扩展
    TArray<const CTypeVariable*> _TypeVariables;  // 类型变量 → System F 扩展
    bool _bClosed;                  // 是否封闭（无自由类型变量）
};
```

Verse 的语法 `type{_(X:int):float}` 在编译器内部构造为 `CFunctionType{ParamsType=int, ReturnType=float, Effects=computes, ...}`。

#### 2.3.2 类型安全：Subject Reduction

STLC 的核心元定理是 **Subject Reduction（归约保类型）**：

$$\Gamma \vdash M : \sigma \quad \text{且} \quad M \longrightarrow_\beta N \quad \Longrightarrow \quad \Gamma \vdash N : \sigma$$

即：类型检查通过的程序，经过任意次 β-规约后，类型不变。这意味着**类型正确的程序不会在运行时因类型错误崩溃**。

Verse 编译器通过 `SemanticTypeUtils::Constrain()` 和 `SemanticTypeUtils::IsSubtype()` 在编译期静态保证这一性质。对于游戏引擎这种需要热更新、网络同步的系统，编译期类型安全是关键的可靠性保障。

### 2.4 System F (λ2) → Verse 的泛型/多态

#### 2.4.1 类型抽象与类型应用

System F（多态 λ 演算，Girard-Reynolds）在 STLC 基础上增加了**类型抽象**和**类型应用**两条规则：

$$\frac{\Gamma \vdash M : \sigma}{\Gamma \vdash \Lambda \alpha.M : \forall \alpha.\sigma} \quad \text{(类型抽象)}$$

$$\frac{\Gamma \vdash M : \forall \alpha.\sigma}{\Gamma \vdash M\ [\tau] : \sigma[\alpha := \tau]} \quad \text{(类型应用)}$$

在 Verse 中：

```verse
# System F:  Λα. λx:α. x    : ∀α. α → α
# Verse:     Identity(T:type)(X:T):T = X

# System F:  Λα. List α      : ∀α. * → *
# Verse:     []T             （CArrayType，类型构造器）
```

#### 2.4.2 编译器中的类型变量与实例化

Verse 编译器中的 `CTypeVariable`（[TypeVariable.h:20-129](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\TypeVariable.h#L20-L129)）是 System F 类型变量的直接实现：

```cpp
class CTypeVariable : public CDefinition, public CNominalType {
    const CTypeBase* _NegativeType;  // 下界（lower bound）
    const CTypeBase* _PositiveType;  // 上界（upper bound）
};
```

每个类型变量有两个边界——这已经超出了纯 System F 的范畴，进入了 System F<sub>:</sub> 的领域（见 §2.5）。

类型实例化由 `SemanticTypeUtils::Instantiate()` 实现（[SemanticTypes.h:1680-1689](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1680-L1689)）：

```cpp
// 对应 System F 的类型应用: M [τ] → M[α := τ]
const CFunctionType* Instantiate(const CFunctionType*, const uint32_t UploadedAtFnVersion);
const CTypeBase* Substitute(const CTypeBase&, ETypePolarity, 
    const TArray<STypeVariableSubstitution>&);
```

`STypeVariableSubstitution`（[SemanticTypes.h:1216-1259](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1216-L1259)）包含正负两个替换目标，因为 Verse 的极性子类型需要分别处理协变和逆变位置：

```cpp
struct STypeVariableSubstitution {
    const CTypeVariable* _TypeVariable;
    const CTypeBase* _NegativeType;  // 替代逆变位置的出现
    const CTypeBase* _PositiveType;  // 替代协变位置的出现
};
```

#### 2.4.3 Verse 的类型构造器

Verse 中的泛型容器类型都是 System F 的类型构造器（type constructor）：

| Verse 语法 | 类型构造器 | 编译器实现 | 种的 System F 类型 |
|---|---|---|---|
| `[]T` | Array | `CArrayType` | $\lambda \alpha.\ \text{Array}\ \alpha$ |
| `[K]V` | Map | `CMapType` | $\lambda \alpha.\lambda \beta.\ \text{Map}\ \alpha\ \beta$ |
| `?T` | Option | `COptionType` | $\lambda \alpha.\ \text{Option}\ \alpha$ |
| `generator(T)` | Generator | `CGeneratorType` | $\lambda \alpha.\ \text{Generator}\ \alpha$ |
| `tuple(A,B)` | Tuple | `CTupleType` | $\lambda \vec{\alpha}.\ \text{Tuple}\ \vec{\alpha}$ |

### 2.5 System F<sub>:</sub> → Verse 的子类型约束

#### 2.5.1 有界量化

System F<sub>:</sub>（有界量化的多态 λ 演算，Cardelli-Wegner）在 System F 的基础上增加了**子类型关系** $<:$ 和**有界量化**：

$$\frac{\Gamma \vdash \tau <: \sigma}{\Gamma \vdash \forall \alpha <: \sigma.\ \tau}$$

Verse 的 `subtype()` 约束直接对应有界量化：

```verse
# F<:  约束:  α <: member_info_interface
# Verse 写法（agent_group.native.verse）:
agent_group_interface<public><native>(member_info:subtype(member_info_interface)) := interface<unique>:
    GetMemberMap<public>()<reads> : [agent]member_info
```

这里 `member_info:subtype(member_info_interface)` 的意思是：$member\_info$ 是 $member\_info\_interface$ 的任意子类型。

#### 2.5.2 子类型格操作

编译器通过 `SemanticTypeUtils` 的四个核心操作实现子类型格的判定（[SemanticTypes.h:1715-1738](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1715-L1738)）：

| 操作 | 数学定义 | 编译器函数 | 用途 |
|---|---|---|---|
| 子类型判定 | $\tau_1 <: \tau_2$ | `IsSubtype(PositiveType1, PositiveType2)` | 类型检查：实参是否为形参的子类型 |
| 约束 | 设 $\tau_1 <: \tau_2$ | `Constrain(PositiveType1, NegativeType2)` | 类型推断：收集子类型约束 |
| 最小上界 (Join) | $\tau_1 \sqcup \tau_2$ | `Join(Type1, Type2)` | `if`/`else` 分支的类型合并 |
| 最大下界 (Meet) | $\tau_1 \sqcap \tau_2$ | `Meet(Type1, Type2)` | 类型精化/交集类型推断 |
| 类型等价 | $\tau_1 \equiv \tau_2$ | `IsEquivalent(PositiveType1, PositiveType2)` | 判断两个类型是否相同 |
| 参数匹配 | $\tau_1$ 匹配 $\tau_2$ | `Matches(PositiveType1, NegativeType2)` | 函数调用时实参-形参匹配 |

`Join` 操作在 `if`/`else` 分支的类型推断中至关重要：

```verse
# 编译器推导: if 分支返回 int, else 分支返回 float
# Join(int, float) = float（最小公共超类型）
Result := if (Cond):
    42          # 类型: int
else:
    3.14        # 类型: float
# Result 的类型推导为 float
```

### 2.6 极性子类型 (Polarity) → 协变与逆变

#### 2.6.1 函数类型的子类型规则

函数类型的子类型规则是 λ 演算中的经典结论——对参数**逆变**、对返回值**协变**：

$$\frac{\sigma_1 <: \tau_1 \quad \tau_2 <: \sigma_2}{\tau_1 \to \tau_2 \quad <: \quad \sigma_1 \to \sigma_2}$$

即：函数 $f: \tau_1 \to \tau_2$ 可以当作 $g: \sigma_1 \to \sigma_2$ 使用的条件是——$f$ 接受的参数范围**更宽**（$\sigma_1$ 是 $\tau_1$ 的子类型），$f$ 返回的结果范围**更窄**（$\tau_2$ 是 $\sigma_2$ 的子类型）。

#### 2.6.2 编译器中的极性实现

Verse 编译器在 `ETypePolarity`（[SemanticTypes.h:1163-1173](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1163-L1173)）中明确建模了极性：

```cpp
enum class ETypePolarity : char { Negative, Positive };

inline ETypePolarity FlipPolarity(ETypePolarity Polarity) {
    // Negative → Positive, Positive → Negative
    // 穿越函数箭头时翻转极性
}
```

- **负极性 (Negative)**：参数位置——逆变。若 $A <: B$，则 `B -> C` $<:$ `A -> C`
- **正极性 (Positive)**：返回位置——协变。若 $A <: B$，则 `C -> A` $<:$ `C -> B`

`CFlowType`（[SemanticTypes.h:1175-1214](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L1175-L1214)）将极性子类型扩展为**信息流分析**：

```cpp
class CFlowType : public CTypeBase {
    ETypePolarity _Polarity;              // 正/负极性
    const CTypeBase* _Child;              // 底层类型
    TSet<const CFlowType*> _FlowEdges;    // 信息流边
};
```

- 正极性 $\approx$ "产生"该类型的值（协变位置）
- 负极性 $\approx$ "消费"该类型的值（逆变位置）
- `_FlowEdges` 记录类型之间的信息流动，用于跨模块/跨网络的安全性分析

### 2.7 效果类型 λ 演算 → Verse 的效果系统

#### 2.7.1 效果标注的箭头类型

经典 λ 演算只关心"是什么类型"，不关心"有什么副作用"。Verse 在类型系统上叠加了**效果标注**，形成效果类型 λ 演算（Effect-Typed Lambda Calculus）：

$$\Gamma \vdash M : \tau\ !\ E$$

其中 $E \subseteq \{\text{suspends}, \text{decides}, \text{diverges}, \text{reads}, \text{writes}, \text{allocates}, \text{dictates}, \text{no\_rollback}\}$ 是效果集合。

在 Verse 中：

```verse
# 纯函数:       int → int  ! ∅           →  <computes>:int
# 可挂起函数:    string → void ! {suspends} →  <suspends>:void
# 事务函数:      agent → void ! {reads, writes, allocates} → <transacts>:void
# 可失败函数:    int → void ! {decides}    →  <decides>:void
```

#### 2.7.2 效果格

效果之间形成**偏序关系**（效果格，Effect Lattice）：

```
converges  (∅)          ← 最纯净：无任何效果，保证终止
    <
computes  ({diverges})  ← 纯计算：可能不终止，但无副作用
    <
reads     ({reads})     ← 读取状态
    <
writes    ({writes})    ← 写入状态
    <
transacts ({diverges, reads, writes, allocates, dictates})  ← 完整事务
```

编译器在 `CFunctionType::_Effects` 中以位掩码存储效果（[Effects.h](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\Effects.h)）：

```cpp
// 效果组合定义
EffectSets::Converges  = {}                                                  // 空集
EffectSets::Computes   = {diverges}                                          // 纯计算
EffectSets::Transacts  = {diverges, reads, writes, allocates, dictates}      // 事务
```

调用检查规则：**调用者的效果必须"覆盖"被调用者的效果**。即，`<computes>` 函数可以调用 `<computes>` 函数，但不能调用 `<transacts>` 函数；`<transacts>` 函数可以调用任何函数。

#### 2.7.3 与 Haskell Monad 的关系

SPJ 在 Haskell 中推广了 Monad 来管理副作用（IO Monad, State Monad 等）。Verse 的效果系统可以看作**Monad 的"语法糖"版本**——不需要 `do` 记法，不需要 `lift`，直接在类型上标注效果，编译器自动传播和检查。

二者的对应关系：

| 概念 | Haskell | Verse |
|---|---|---|
| 纯函数 | `a -> b` | `<computes>:b` |
| 有副作用 | `a -> IO b` | `<transacts>:b` |
| 可失败 | `a -> Maybe b` | `<decides>:b`（或返回 `false`） |
| 效果组合 | Monad Transformer | 效果集合并（位掩码 OR） |
| 效果多态 | `Monad m => a -> m b` | 效果推断（隐式） |

### 2.8 `CTypeType`：以子类型区间编码"类型的类型"

Verse 的 `CTypeType`（[SemanticTypes.h:396-473](d:\MyWorkSpace\engine\UnrealEngine-ue6-main\Engine\Source\Runtime\VerseCompiler\Public\uLang\Semantics\SemanticTypes.h#L396-L473)）是一个精巧的设计——每个"类型的类型"编码为一个**子类型区间**：

```cpp
class CTypeType : public CNormalType {
    const CTypeBase* _NegativeType;  // 区间下界：该类型的所有值都是此类型的超类型
    const CTypeBase* _PositiveType;  // 区间上界：该类型的所有值都是此类型的子类型
};
```

这意味着类型检查器判断"一个类型 $\tau$ 是否满足类型约束 `T:type(A, B)`"时，本质上是判断 $\tau$ 是否落在区间 $[A, B]$ 内：

$$A <: \tau <: B$$

即 $\tau$ 是 $A$ 的超类型，同时是 $B$ 的子类型。

例如，`subtype(component)` 约束在编译器内部编码为：

```cpp
// CTypeVariable for T:subtype(component)
// _NegativeType = component  (下界: T 必须是 component 的超类型)
// _PositiveType = any         (上界: T 不能超过 any)
// 区间: [component, any] → T 是 component 的任意子类型
```

这种"区间编码"比传统 Hindley-Milner 类型推断更灵活——它允许子类型多态而不需要显式的协变/逆变标注，类型检查器自动在区间内传播约束。

### 2.9 理论层次总览

将以上所有层级汇总为一张表：

| 层级 | λ 演算变体 | 新增能力 | Verse 语法对应 | 编译器实现 |
|---|---|---|---|---|
| 0 | 无类型 λ 演算 | 函数抽象/应用 | 表达式求值 | `CExpressionBase` |
| 1 | 简单类型 λ 演算 (λ→) | 类型标注与检查 | `type{_(X:int):float}` | `CFunctionType` |
| 2 | System F (λ2) | 参数化多态（泛型） | `[]T`, `option(T)`, `[K]V` | `CTypeVariable` + `Instantiate()` |
| 3 | System F<sub>:</sub> | 子类型 + 有界量化 | `T:subtype(U)` | `IsSubtype()` / `Constrain()` |
| 4 | 效果类型 λ 演算 | 副作用标注与检查 | `<suspends>`, `<transacts>`, `<decides>` | `SEffectSet` in `CFunctionType` |
| 5 | 极性子类型 | 协变/逆变自动推导 | 隐式（编译器自动推断） | `ETypePolarity` + `CFlowType` |
| 6 | 子类型格 | Join/Meet 类型运算 | `if`/`else` 分支类型合并 | `Join()` / `Meet()` |

### 2.10 为什么选择 λ 演算作为理论基础

Simon Peyton Jones 选择 λ 演算作为 Verse 的理论基础，有三个核心原因：

**1. 类型安全 (Type Safety)**

Subject Reduction 定理保证：类型正确的程序不会在运行时因类型错误崩溃。对于游戏引擎这种需要热更新、网络同步、多线程并发的系统，编译期类型安全是关键的可靠性保障。UE6 的 Verse VM 不需要在运行时做类型检查——所有类型错误在编译期就被捕获。

**2. 可判定性 (Decidability)**

System F<sub>:</sub> 的类型检查是可判定的（尽管在最一般情况下是 NP 难的，但在工程实践中表现良好）。Verse 的 `Constrain()` 系统在编译期就能捕获所有类型错误。这不同于 C++ 的模板（图灵完备的类型系统）或动态语言（运行时类型检查）。

**3. 组合性 (Composability)**

λ 演算的组合性保证了 Verse 的并发原语可以安全地组合。`spawn{}`、`sync{}`、`race{}` 等并发宏在类型层面是"可组合的"——你可以将一个 `sync{}` 嵌套在另一个 `sync{}` 内部，类型系统保证效果传播的正确性。

**一句话总结**：

$$\text{Verse} = \text{System F}_{<:} + \text{效果类型} + \text{并发原语}$$

它在 λ 演算的坚实数学基础上，扩展出了游戏引擎所需的表达力。理解这个数学基础，就能理解 Verse 语法设计的"为什么"：
- 为什么 `=` 是相等比较而不是赋值？——因为 λ 演算中 `=` 天然是等式关系
- 为什么 `set` 是必需的？——因为 λ 演算默认不可变，副作用必须显式标记
- 为什么 `and`/`or`/`not` 而不是 `&&`/`||`/`!`？——因为 λ 演算的"短路求值"语义与逻辑连接词一致
- 为什么有 `<suspends>` 效果？——因为 λ 演算本身不支持挂起/恢复，需要在类型层面显式建模

---

## 3. 词法结构（Lexical Structure）

### 3.1 字符集与空白

- 源文件必须是 **ASCII 或 UTF-8**（编译器直接报 S01 错误）。UTF-8 BOM（`EF BB BF`）允许。
- 空白 = 空格 ` ` 与制表符 `\t`；换行 = `\r` 或 `\n`（或 `\r\n`）。
- **缩进对语法有语义**（见 §6 缩进块），但空格/tab 混用不一致会报 S89。
- 关键词与标识符**区分大小写**。

### 3.2 注释

| 语法 | 含义 | 来源规则 |
|---|---|---|
| `# 注释` | 行注释 | `LineCmt := '#' !'>' {Text} Ending` |
| `<# 注释 #>` | 块注释（可跨行） | `BlockCmt := "<#" !'>' {Text\|NewLine} !'<' "#>"` |
| `<#>` …缩进… | 缩进注释（一直延续到缩进结束） | `IndCmt := "<#>" {Text} Ind {Text\|Line} Ded` |

```verse
# 行注释
<# 块注释
   可以跨多行 #>
<#>
    缩进注释：这块内容直到缩进变浅为止
    全部都是注释
```

⚠️ 注意：
- 行注释 `//` **不是**合法注释。编译器会直接报：`S68: Use # for line comment, not //`。
- 块注释用 `<# #>`（类似 XML 注释），不是 `/* */`。

### 3.3 标识符

```
Ident := Alpha {Alnum} !Alnum ["'" {…} "'"]
Alpha := [A-Za-z_]
Alnum := [A-Za-z0-9_]
```

- 允许字母、数字、下划线，**可以包含单引号后缀段**（引用标识符），例如：
  - `MyVar`、`_private`、`my_function`
  - `x'weird identifier'` —— 单引号内是"引用名"，可包含空格等符号（用于绑定原生 C++ 名）。
- 大小写敏感：`foo`、`Foo`、`FOO` 是三个不同标识符。
- **路径/带版本标识**：模块引用 `/Verse.org` 中的 `Label := Alnum {Alnum|'-'|'.'}`，即标签可含 `-` 和 `.`，但不能以它们开头。

### 3.4 数字字面量

```
Num := Digits ['.' Digits] Exp Units
Exp := [('e'|'E') ['+'|'-'] Digits]
```

| 形式 | 示例 | 说明 |
|---|---|---|
| 十进制整数 | `42` `-7` | |
| 小数 | `3.14` `0.5` | |
| 指数 | `1e3` `2.5e-2` | |
| 十六进制 | `0xFF` `0xdeadbeef` | `0x` 前缀，后接十六进制位 |
| 字符八进制 | `0o141` | `0o` 前缀 → 单个 byte（Char8，§3.5） |
| Unicode 码点 | `0u1F600` | `0u` 前缀 → 单个 Unicode 字符（Char32，§3.5） |
| 单位后缀 | `100ms` `5f32` | `Units := [Alpha {Alpha}]`，作为 `units'` 调用处理 |

```verse
X := 42
Y := 3.14e2
Z := 0xFF          # 255
```

### 3.5 字符字面量

```
CharLit := ''' Printable ''' !''' | ''' CharEsc '''
CharEsc := '\' ('r'|'n'|'t'|'''|'"'|Special)
Special := '\' | '{' | '}' | '#' | '<' | '>' | '&' | '~'
```

```verse
C1 := 'a'
C2 := '\n'
C3 := '\''        # 反斜杠转义单引号
C4 := 0o141       # 'a' 的八进制字节形式（Char8）
C5 := 0u4F60      # 中文"你"的码点（Char32）
```

### 3.6 字符串字面量与插值

```
String := '"' {Interp | CharEsc | !('\'|'{'|'}'|'"') Text} '"'
Interp := '{' List '}'      # 字符串插值 → 编译为 ToString(...)
CharEsc := '\' ('r'|'n'|'t'|'''|'"'|Special)
```

```verse
Name := "Verse"
Print("Hello, {Name}!")          # 插值 {expr} → ToString(expr)

Escape := "tab\t newline\n quote\" slash\\"
BackslashSpecial := "\{ \} \< \> \& \~ \#"
```

要点：
- `{}` 花括号里写**表达式**，运行时拼接（编译器把 `{X}` 展开为 `ToString(X)`）。
- 反斜杠转义支持 `r n t`、`'`、`"`、`\` 以及特殊字符 `{ } < > & ~ #`。
- 字符串里也可以嵌套**标记语言 Markup**：`<tag>…</tag>`、`<tag;expr>`、`&expr;`（见 §3.7 附近说明；这是 Verse 内置的文本模板能力）。

### 3.7 路径字面量 / 模块引用

```
Path := '/' Label ('@' Label | !'@') {'/' ['(' Path ':)'] Ident} !'/'
```

```verse
using { /Verse.org }
using { /Verse.org/Native }
using { /Verse.org/SceneGraph }
using { /Verse.org/Simulation }
using { /Fortnite.com/Devices }
```

- 以 `/` 开头的路径用于**模块（Module）**引用，语法类似域名路径。
- `using { ... }` 是引入模块的方式；`import` 也是保留字（用于部分导入/类型级导入）。

---

## 4. 运算符与优先级

### 4.1 完整 Token 表

以下 Token 表直接摘自 `VerseGrammar.h` 的 `Tokens[]` 数组（`Symbol` / 优先级 / 结合性 / 模式）。

**关键词（EMode 为 `With`，即可作为宏调用的保留词）：**

| Token | 角色 |
|---|---|
| `alias` `ref` `set` `var` `live` | 变量/绑定修饰 |
| `if` `then` `else` `in` `is` | 条件/绑定/类型测试 |
| `for` `while` `loop` `repeat` `break` `continue` `return` `yield` `until` | 控制流（for/loop/repeat/until 为语义层宏） |
| `spawn` `sync` `race` `branch` `rush` `stop` `await` `defer` | 并发原语（语义层宏） |
| `catch` `do` `when` `where` `over` `next` | 子句/绑定 |
| `not` `and` `or` | 逻辑运算 |
| `to` `at` `of` | 关系/操作 |
| `mutable` `const` `case` | 修饰/模式匹配 |

**符号 Token：**

| Token | 含义 | 备注 |
|---|---|---|
| `,` `;` | 分隔符 | |
| `=` | **相等比较** | ⚠️ 不是赋值！见 §4.3 |
| `:=` | 定义/（不可变）声明 | 声明新绑定 |
| `+=` `-=` `*=` `/=` | 复合赋值 | 必须配合 `set` |
| `<` `<=` `>` `>=` `<>` | 比较 / 不等 | `<>` 是不等于 |
| `+` `-` `*` `/` | 算术 | 前缀也可作一元符号 |
| `&` | 乘级中缀运算 / 标记转义 | |
| `to` `..` `->` | 范围/箭头 | `a to b`、`a..b`、`a->b` |
| `=>` | 函数类型/映射箭头 | |
| `.` | 成员访问 | |
| `[` `]` | 索引 / 集合字面量 | |
| `{` `}` | 块 / 字符串插值 | |
| `(` `)` | 分组 / 调用 / 参数 | |
| `:` | 类型注解 / `in` 的替代 | |
| `:)` `:>` | 限定名结束 / 缩进标记 | `(super:)` 用于调用父类 |
| `?` | 前缀：Option 构造；后缀：类型后缀 | `?int`、`int?` |
| `^` | 前缀/后缀运算符（泛型运算符） | |
| `@` | 属性（Attribute）前缀 | |
| `!` | **非法**（用 `not`） | 报 S62 |
| `==` | **非法**（用 `=`） | 报 S65 |
| `'` `"` | 字符/字符串 | |

### 4.2 优先级层级（EPrec）

编译器内部按从松到紧排序：`List < Commas < Expr < Fun < Def < Or < And < Not < Eq < NotEq < Less < Greater < Choose < To < Add < Mul < Prefix < Call < Base`。

对应到源码 `EPrec` 枚举，实际顺序为（低 → 高）：

```
Never < List < Commas < Expr < Fun < Def < Or < And < Not
     < Eq < NotEq < Less < Greater < Choose < To < Add < Mul
     < Prefix < Call < Base < Nothing
```

| 层级 | 运算符 | 结合性 |
|---|---|---|
| Or | `or` | 右 |
| And | `and` | 右 |
| Not | `not`（前缀） | — |
| Eq | `=` | 左 |
| NotEq | `<>` | 左 |
| Less | `<` `<=` | 右 |
| Greater | `>` `>=` | 右 |
| Choose | `|` | 右 |
| To | `to` `..` `->` | 右 |
| Add | `+` `-` | 左 |
| Mul | `*` `/` `&` | 左 |
| Prefix | `^` `?` `-` `+` `*` 等前缀 | — |
| Call | 调用 `()` `[]` `.` `at` `of` `with` | 左 |

关键词后接子句的（`when` `where` `while` `over` `next` `is` `in`）都在 `Def`/`Fun` 层，属于很低的优先级，保证 `f(x) when Cond` 之类的写法能把 `when` 挂在整个表达式上。

### 4.3 常见错误提示（编译器引导）

编译器会主动"纠正"你在别的语言里的习惯，错误信息本身就是语法文档：

| 输入 | 报错 |
|---|---|
| `A && B` / `A || B` / `!A` | `S62: Verse uses 'and', 'or', 'not' instead of '&&', '||', '!'.` |
| `A == B` | `S65: Use a=b for comparison, not a==b` |
| `X += 1`（无 `set`） | `S66: Use 'set' before "x+=" to update variables` |
| `// comment` | `S68: Use # for line comment, not //` |
| 赋值 `X = 5`（X 是不可变绑定） | `S66`（要求 `set` 前缀） |

---

## 5. 类型系统

### 5.1 基本类型

| 类型 | 说明 |
|---|---|
| `logic` | 布尔：`true` / `false` |
| `int` `int8` `int16` `int32` `int64` | 有符号整数 |
| `nat` `nat8` `nat16` `nat32` `nat64` | 无符号整数（natural） |
| `float` `float16` `float32` `float64` `float128` | 浮点 |
| `rational` | 有理数（保留） |
| `char` `char8` `char16` `char32` | 字符 |
| `string` `string8` `string16` `string32` | 字符串 |
| `void` | 无值 |
| `false` | "失败"类型（failing function 的返回） |
| `any` / `unknown` | 顶层类型 |
| `none` | 空值（不常用，Option 用 `?T`） |

### 5.2 Option 与失败类型

```verse
MaybeInt : ?int = false        # Option<int>，空值为 false
SomeVal  : ?int = 5            # Option<int>，有值为 5

if (V := MaybeInt):            # Option 绑定：非空才进入
    Print("有值 {V}")
```

- Option 类型写 `?T` 或 `option(T)`；字面量空值是 `false`，有值就是值本身。
- 用 `if (X := OptValue)` 解包 Option——这是 Verse 的"可空类型"用法。

### 5.3 容器类型

| 类型 | 语法 | 示例（来自引擎标准库） |
|---|---|---|
| 数组 | `[]T` / 字面量 `array{}` | `Items:[]fort_item = array{}` |
| 映射（Map） | `[K]V` / 字面量 `map{}` | `MemberMap<...>:[agent]member_info = map{}` |
| 元组 | `tuple(A, B)` | `listenable(tuple(agent, member_info))` |
| 联合 | `union` / 变体 | union 类型成员即"变体" |

```verse
# 引擎真实代码（agent_group.native.verse）
var<private> MemberMap<epic_internal><native> : [agent]member_info = map{}

# 读/写 map 元素
set MemberMap[Agent] = MemberInfo
ExistingMemberInfo := MemberMap[Agent]     # 返回 Option（键不存在时为 false）
```

### 5.4 函数类型

```verse
# 类型别名中的函数类型（引擎真实代码，CollisionChannel.native.verse）
collision_channel_to_interaction<public> := type{
    _(InChannel:collision_channel)<computes>:collision_interaction
}
```

- 函数类型的参数名可用 `_` 占位；返回类型写在 `:` 后。
- 效果写在参数列表后（`<computes>`），见 §9。

### 5.5 用户自定义类型

`class` / `struct` / `interface` / `enum` / `module` / `type` / `attribute` 见 §7 声明语法。泛型通过**类型参数**实现：

```verse
agent_group_interface<public><native>(member_info:subtype(member_info_interface)) := interface<unique>:
    GetMemberMap<public>()<reads> : [agent]member_info
```

即 `名字<specifiers>(类型参数:subtype(约束)) := interface<unique>:`——类型参数在 `:=` 之前的括号里声明，`subtype(...)` 表达"必须是某接口/类的子类型"。

---

## 6. 块（Block）的三种写法

Verse 块可以用**花括号**、**缩进**、或**冒号+缩进**三种形式（编译器 `Block := Brace | DotSpace Space Def Space | (DotSpace|':') Space Ind List Ded`）：

```verse
# 1) 花括号 Brace
if (X) { Print("A") }

# 2) 缩进块 Ind List Ded（冒号可省略，靠换行缩进）
if (X)
    Print("A")
    Print("B")

# 3) 冒号 + 缩进（最常用）
if (X):
    Print("A")
    Print("B")

# 花括号也常用于单行内联
spawn{Immediate()}
```

**块内分隔符**：行内多条语句用 `;` 或换行分隔；换行后**缩进必须一致**（S89）。

---

## 7. 声明语法

### 7.1 变量：不可变 / 可变 / 赋值

```verse
X := 42          # 不可变绑定（:=），类型推导
Y : int = 7      # 不可变绑定，显式类型
var Z : int = 0  # 可变变量（var）
var W := 3.14    # 可变变量，类型推导

set Z = 5        # 可变变量赋值必须写 set
set W = 2.0
```

| 写法 | 语义 |
|---|---|
| `X := expr` | 声明**不可变**绑定（类型推导） |
| `X : T = expr` | 声明不可变绑定（显式类型） |
| `var X := expr` / `var X : T = expr` | 声明**可变**变量 |
| `set X = expr` | 给可变变量赋值（`set` 是必需的） |
| `set X += 1` | 复合赋值（`+=` `-=` `*=` `/=`） |
| `live` | 存活性修饰（延迟求值/实时性），`set live` |

成员变量示例（组件类里带 Specifier 与属性）：

```verse
new_component_template<public> := class<final_super>(component):
    @editable
    var MyCustomInt<public>:int = 10
```

### 7.2 函数声明

```verse
# 通用形式：
#   Name<specifiers>(Params)<effects> : ReturnType = Body
#   或 Name<specifiers>(Params)<effects> : ReturnType = <单表达式>

OnBeginSimulation<override>():void =
    (super:)OnBeginSimulation()          # 调用父类实现
    Print("OnBeginSimulation")

OnSimulate<override>()<suspends>:void =
    Print("OnSimulate")

Add<public>(A:int, B:int)<computes>:int =
    return A + B
```

要点：
- **参数**：`Name:Type`，可用 `_` 省略参数名（`(_:collision_channel)`），可用 `:Type` 只写类型。
- **效果**：写在 `)` 和 `:` 之间（`<suspends>` `<transacts>` `<computes>` …），见 §9。
- **返回类型**：写在 `:` 后；`void`、具体类型、`result(T,E)`、`false`（失败函数）、`?T`（Option）。
- **函数体**：`=` 后跟块；单表达式函数直接 `= expr`。
- **Specifier**：`<override>` `<public>` `<private>` `<native>` `<native_callable>` `<final>` 等。
- **调用父类**：`(super:)Method()`——`super` 后跟 `:)` 语法。

原生函数（native，无函数体）示例（引擎 IO.native.verse）：

```verse
LoadTextFile<native><public>(Filename:string)<suspends>:result(string, io_error)
Print<public><native>(:string)<suspends>:void
Exit<public>()<suspends>:false = Exit(0)
```

### 7.3 类 class

```verse
MyClass := class:                    # 空基类
    Value : float

Derived := class(MyClass):           # 单继承
    DerivedValue : float

# 多继承 + 多个 class-level specifier（引擎真实代码）
physics_component<native><epic_internal> := class<final><final_super>(component, enableable):
    Enable<native><override>():void

# 空类体用花括号（引擎真实代码）
io_error<native><public> := class{}
```

要点：
- 类声明：`名字<specifiers> := class<class-specifiers>(基类列表):` 或 `:= class(基类):`。
- 继承写在 `class` 后的括号里，可多个（多继承）；接口也在其中。
- class-level specifier：`<final>`（不可再被继承）、`<final_super>`（不能在此类上再覆写/继承层级到此结束）、`<unique>`（单例）、`<abstract>`（抽象）、`<concrete>`、`<computes>`（该类的实例方法默认 `computes` 效果）、`<native>`、`<epic_internal>` 等。
- 成员函数覆写基类方法用 `<override>`；同时标记 `<native>` 表示实现来自 C++。

### 7.4 结构体 struct

```verse
hit_result<epic_internal><native> := struct:
    ThisEntity<public><native>:?entity
    ThisComponent<public><native>:?component
    HitNormal<public><native>:vector3
    HitLocation<public><native>:vector3
```

- 结构体成员直接写 `字段名<specifiers>:类型`，无需初值也可（native 字段）。
- struct 是值类型（struct），class 是引用类型。

### 7.5 接口 interface

```verse
member_info_interface<public><native> := interface{}

agent_group_interface<public><native>(member_info:subtype(member_info_interface)) := interface<unique>:
    GetMemberMap<public>()<reads> : [agent]member_info
    AddMemberEvent<public> : listenable(tuple(agent, member_info))
```

- 接口里只写**签名**（方法名、效果、返回类型），没有函数体。
- `<unique>`：该接口/类全局只能有一个实例。
- 泛型接口：`名字<specifiers>(类型参数:subtype(约束)) := interface<unique>:`。

### 7.6 枚举 enum

```verse
collision_interaction<public><native> := enum:
    Ignore,
    Overlap,
    Block

# 开放枚举（可扩展）
ue_phys_sleep_type<epic_internal><native> := enum<open>:

# 单行花括号形式（引擎真实代码）
keyframed_movement_net_command<native><epic_internal> := enum{PlayFrom, HaltAt}
```

- 枚举成员用逗号分隔（尾逗号可选）。
- `enum<open>`：允许外部继续加成员；默认枚举是"封闭"的。

### 7.7 模块 module 与 using/import

```verse
BaseModule<public> := module:
    base_class := class:
        Value : float

    derived_class_1 := class(base_class):
        DerivedValue : float
```

- 一个 `.verse` 文件顶层就是一个模块：`模块名<specifiers> := module:`。
- 跨模块引用用 `using { /路径 }`；细粒度导入用 `import`（保留字）。

### 7.8 类型别名 type

```verse
collision_channel_to_interaction<public> := type{
    _(InChannel:collision_channel)<computes>:collision_interaction
}
```

- `名字 := type{ 类型 }`：给函数类型/复杂类型起别名。

### 7.9 自定义属性 attribute

```verse
@attribscope_class
my_class_attrib := class(attribute):
```

- 自定义属性从 `attribute` 继承；加上 `@attribscope_class` 表示它只能标注在 class 上。
- 使用时以 `@属性名` 前缀标注（见 §8）。

---

## 8. 属性（Attributes，`@`）

属性是 `@` 前缀的**元数据标注**，作用于它后面的声明。引擎标准库中常见的：

| 属性 | 含义 | 示例 |
|---|---|---|
| `@editable` | 暴露到编辑器（Detail 面板可编辑） | `@editable var MyCustomInt<public>:int = 10` |
| `@doc("...")` | 文档字符串（支持 `{}` 分段续行） | `@doc("Get the members...{ }Passes in...")` |
| `@replicated("RepNotify")` | 网络复制，参数为 OnRep 函数名 | `@replicated("RepNotify") var Enabled<private><native>:logic=true` |
| `@experimental` | 实验性 API（未稳定） | `@experimental hit_result<...> := struct:` |
| `@available{MinUploadedAtFNVersion := 4000}` | 版本门控（Fortnite 上传版本） | `@available{MinUploadedAtFNVersion := 4000}` |
| `@attribscope_class` | 自定义属性可用的标注目标 | `@attribscope_class my_class_attrib := class(attribute):` |
| `@internal` | 内部可见性 | 自定义属性标注 |

```verse
@editable
var MyCustomInt<public>:int = 10

@replicated("RepNotify")
@experimental
var Enabled<private><native>:logic = true
```

---

## 9. 效果系统（Effects）

效果（effect）是 Verse 的**编译期副作用检查**。函数/接口方法声明时用 `<...>` 标注"它会做什么"，编译器据此检查调用关系（例如：`<transacts>` 函数不能调用带 `no_rollback` 效果的信号器）。

核心效果（`Effects.h` 的 `VERSE_ENUM_EFFECTS`）：

| 效果 | 含义 |
|---|---|
| `<suspends>` | 可挂起（异步/协程，能 `await`/`sleep`） |
| `<decides>` | 可失败（failing function，失败时返回 `false`） |
| `<diverges>` | 可能不终止 / 计算型（不保证收敛） |
| `<reads>` | 读取状态 |
| `<writes>` | 写入状态 |
| `<allocates>` | 分配内存 |
| `<dictates>` | 强制/支配（用于事务与副作用传播） |
| `<no_rollback>` | 不可回滚（不可放在事务中） |

**组合效果**（`EffectSets`）：

| 名称 | = 组合 |
|---|---|
| `<converges>` | 空（保证终止） |
| `<computes>` | `diverges` |
| `<transacts>` | `diverges \| reads \| writes \| allocates \| dictates`（事务：默认效果） |
| `<varies>` | 同 `transacts`（旧名，弃用） |

**默认效果**（函数/类没写效果时的默认值）：

| 声明种类 | 默认效果 |
|---|---|
| class / interface | `<transacts>` |
| function | `<transacts> \| no_rollback` |
| module | `allocates \| dictates \| diverges` |
| 参数化类型 | `converges \| dictates` |

```verse
# 可挂起 + 可失败
IsEnabled<native><override>()<decides><transacts>:void

# 可挂起（协程）
OnSimulate<override>()<suspends>:void =

# 读取效果
GetMemberMap<public>()<reads> : [agent]member_info

# 事务函数（默认也是 transacts）
AddMember<public>(Agent:agent, MemberInfo:member_info)<transacts> : result(void, add_member_error) =
```

---

## 10. 控制流与并发

### 10.1 if / then / else

Verse 的 `if` 是**表达式**，并且支持"失败表达式 + 绑定"：

```verse
# 基础形式（布尔条件）
if (X > 0):
    Print("正数")
else:
    Print("非正数")

# 带 then 子句（多失败表达式作为守卫）
if:
    MemberMap[Agent]                 # 若 Option 为空 → 失败
    set MemberMap[Agent] = MemberInfo # 若任一表达式失败 → 进 else
then:
    SignalMemberInfoChangeEvent(Agent, MemberInfo)
    MakeSuccess()
else if (set MemberMap[Agent] = MemberInfo):
    SignalAddMemberEvent(Agent, MemberInfo)
    MakeSuccess()
else:
    MakeError(add_member_error {})

# Option 绑定（解包）
if (V := MaybeInt):
    Print("有值 {V}")
```

要点：
- `if:` + 缩进守卫（守卫里可以有多条会失败的表达式），`then:` 是成功分支，`else if` 可链式，`else:` 是兜底。
- `if (X := FailingExpr)` 的绑定形式：右侧**会失败**，成功才进入并绑定 `X`。
- `else if` 是语法特例，保证 `if(a){b}else if(c){d}+1` 中 `+1` 挂在整体 `if` 表达式上。

### 10.2 for / while / loop / repeat

```verse
# for：遍历容器（箭头语法 + 可选过滤器）
for (I -> C : S, I < X) { C }     # I 为索引/键，C 为元素；逗号后是过滤器

# while
while (Cond):
    ...

# loop：无限循环（用 break 退出）
loop:
    ...
    break
    continue

# repeat（保留字，语义层宏）
```

`for` 的遍历子句语法（来自编译器测试 `TestExtensionMethod.native.verse`）：
```verse
(S:string).TakeExtension<public>(X:int):string = { for (I -> C : S, I < X) { C } }
```

- `for (Pattern : Collection)` 或 `for (K -> V : Map, 过滤器…)`。
- `break` / `continue` 只允许出现在 `loop` 内（编译器会检查 `Breakable` 上下文）。

### 10.3 失败处理与 if 绑定

Verse 的**失败函数**（failing function）是类型的一部分：

```verse
# 声明：<decides> 效果 + 返回 false
Exit<native><public>(ExitCode:int)<suspends>:false

# 使用：必须用 if/条件上下文处理失败
if (Result := TrySomething()):
    # 成功
else:
    # 失败
```

- 返回 `false` / `<decides>` 的函数可能"失败"。
- 失败必须在**条件上下文**（`if`、`for` 过滤器、`and`/`or` 守卫等）中被处理，否则编译报错。
- 更显式的做法是返回 `result(T, E)`：`result(void, add_member_error)`，配合 `MakeSuccess()` / `MakeError(...)`。

### 10.4 并发原语

语义分析器把以下标识符解析为**内置宏**（`_InnateMacros`）：

| 宏 | 语义 |
|---|---|
| `spawn{}` | 在后台并发执行，立即返回 |
| `sync{}` | 并行执行所有子表达式，**等全部完成** |
| `race{}` | 并行执行，**第一个完成的胜出**（取消其余） |
| `branch{}` | 并行执行，选择一个（非确定性） |
| `rush{}` | 并行执行，等第一个完成（更宽松的 race） |
| `await` | 等待一个可挂起表达式（版本门控） |
| `defer` | 延迟执行（作用域结束时运行） |
| `stop` | 停止/取消并发 |
| `loop` / `case` / `for` / `map` / `assert_keep` | 迭代 / 模式匹配 / 映射 |

```verse
# 引擎真实代码：spawn 后台任务
SpawnImmediate<public><native_callable>():task(void) = spawn{Immediate()}

# sync：并行执行所有分支并等待全部完成
sync:
    TaskA()
    TaskB()

# race：谁先完成谁赢
race:
    SlowPath()
    FastPath()
```

---

## 11. 综合示例：一段完整的 UE6 Verse 代码

把引擎自带的 `ComponentTemplate.verse`（UE6 组件模板）作为标准范式：

```verse
# Copyright Epic Games, Inc. All Rights Reserved.

using { /Verse.org }
using { /Verse.org/Native }
using { /Verse.org/SceneGraph }
using { /Verse.org/Simulation }

# A Verse-authored component that can be added to entities
new_component_template<public> := class<final_super>(component):

    # A custom variable you can expose to the editor
    @editable
    var MyCustomInt<public>:int = 10

    # Runs when the component should start simulating in a running game.
    OnBeginSimulation<override>():void =
        # Run OnBeginSimulation from the parent class before
        # running this component's OnBeginSimulation logic
        (super:)OnBeginSimulation()

        # TODO: Place logic to run when the component starts simulating here
        Print("OnBeginSimulation")

    # Runs when the component should start simulating in a running game.
    # Can be suspended throughout the lifetime of the component. Suspensions
    # will be automatically cancelled when the component is disposed or the
    # game ends.
    OnSimulate<override>()<suspends>:void =
        # TODO: Place simple suspends logic to run for this component here
        Print("OnSimulate")
```

**逐行语法拆解：**

| 片段 | 语法点 |
|---|---|
| `using { /Verse.org }` | 模块引入（§3.7） |
| `new_component_template<public> := class<final_super>(component):` | 类声明：名字 specifier + class specifier + 基类 + 冒号缩进块（§7.3） |
| `@editable` | 属性（§8） |
| `var MyCustomInt<public>:int = 10` | 可变成员变量 + 类型注解 + 初值（§7.1） |
| `OnBeginSimulation<override>():void =` | 方法：specifier + 空参 + `:void` + 块体（§7.2） |
| `(super:)OnBeginSimulation()` | 调用父类实现（§7.2） |
| `OnSimulate<override>()<suspends>:void =` | 效果 `<suspends>`：可挂起协程（§9） |
| `# 注释` | 行注释（§3.2） |

---

## 附录 A：保留字全表

来自 `ReservedSymbols.inl`（`EIsReservedSymbolResult::Reserved`），按首字母排序。**「预留未来」**列标注的是被保留但当前只留名、语法尚未开放的关键字（`ReservedFuture`）。

| 类别 | 关键字 |
|---|---|
| 声明 | `class` `struct` `interface` `enum` `module` `type` `trait` `subclass` `subtype` `function` `macro` `syntax` `instance` `constructor` `tuple` `union` |
| 变量/绑定 | `var` `set` `ref` `alias` `live` `let` `local` `mutable` `const` `given` `where` |
| 类型 | `int` `int8/16/32/64` `nat` `nat8/16/32/64` `float` `float16/32/64/128` `rational` `logic` `char8/16/32` `string8/16/32` `void` `option` `any` `none` `unknown` |
| 控制流 | `if` `else` `then` `for` `while` `loop` `repeat` `until` `break` `continue` `return` `yield` `when` `over` `next` `case` `match` `guard` |
| 并发 | `spawn` `sync` `race` `branch` `rush` `stop` `await` `defer` `block` |
| 效果 | `suspends` `transacts` `converges` `diverges` `computes` `decides` `fails` `succeeds` `reads` `writes` `allocates` `dictates` `interacts` `iterates` `introspects` `varies` `verify` `throws` `abstract` `abstracts` `invariant` |
| 面向对象 | `super` `Self` `abstract` `final` `closed` `open` `unique` `concrete` `pervasive` `persistent` `persistable` `scoped` `weak` |
| 其他 | `and` `or` `not` `in` `is` `as` `at` `of` `to` `do` `catch` `try` `assert` `ensure` `find` `first` `last` `known` `import` `using` `external` `native` `native_callable` `intrinsic` `agent` `embed` `random` `permutation` `sequence` `bag` `fold` `markup` `content` |

**语义分析器中实际解析为内置宏的**（重点记忆）：

```
assert_keep, await, branch, case, defer, for, if,
loop, map, race, rush, spawn, sync, when
```

> 提示：`spawn`/`sync`/`race`/`branch`/`rush` 等在语法上是宏调用（`spawn{}` 形式），这就是为什么 token 表里没有它们的中缀优先级——它们由语义分析器直接解释。

---

## 附录 B：Node / AST 类型速查

来自 `NodeDecls.inl`（`VST_NODE` 枚举）。语法树节点即"语言允许的结构"清单：

| 类别 | 节点 |
|---|---|
| 容器 | `Project` `Package` `Module` `Snippet` |
| 声明/定义 | `Definition` `Assignment` `TypeSpec` `Mutation` `Where` |
| 字面量 | `IntLiteral` `FloatLiteral` `CharLiteral` `StringLiteral` `PathLiteral` |
| 标识符 | `Identifier` `Operator` `Lambda` `Interpolant` `InterpolatedString` |
| 二元运算 | `BinaryOpLogicalOr` `BinaryOpLogicalAnd` `BinaryOpCompare` `BinaryOpArrow` `BinaryOpAddSub` `BinaryOpMulDivInfix` `BinaryOpRange` |
| 前缀 | `PrefixOpLogicalNot` `PrePostCall` |
| 控制流 | `FlowIf` `Control` `Clause` |
| 调用 | `PrePostCall`（调用/成员访问） |
| 其他 | `Parens` `Commas` `Macro` `Escape` `Comment` `Placeholder` `ParseError` |

这些节点与编译器 `EAstNodeType`（如 `Flow_If` `Flow_Defer` `Ir_For` `Invoke_MapFormer`）一一对应，是语法文档映射到编译器实现时的索引。

---

### 相关文档

- UE6 场景图/组件：本目录 `SceneGraph` 系列
- 引擎源码：`Engine/Source/Runtime/VerseCompiler/`（uLang 语法 + 语义分析）
- 引擎标准库示例：`Engine/Plugins/EntityFramework/Source/**/Verse/*.native.verse`
- 编译器测试：`Engine/Source/Programs/LowLevelTests/VerseCompilerTests/Resources/testVerseFiles/`
