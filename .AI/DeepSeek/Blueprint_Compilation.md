# UE5 蓝图编译与虚拟机详解

> 本文档基于 UE5 引擎源代码（分支 5.5）编写，深入解析蓝图编译器和虚拟机的设计与实现，涵盖从 `UEdGraph` 到 `UFunction::Script[]` 字节码的完整路径。

---

## 目录

1. [编译概览](#1-编译概览)
2. [编译流水线](#2-编译流水线)
3. [编译数据模型](#3-编译数据模型)
4. [中间表示 IR](#4-中间表示-ir)
5. [节点处理架构](#5-节点处理架构)
6. [字节码生成](#6-字节码生成)
7. [虚拟机架构](#7-虚拟机架构)
8. [字节码指令集参考](#8-字节码指令集参考)
9. [执行细节](#9-执行细节)
10. [编译管理器与编译策略](#10-编译管理器与编译策略)

---

## 1. 编译概览

### 1.1 蓝图编译的本质

蓝图编译器 `FKismetCompilerContext` 的任务是将**可视化的节点图（UEdGraph + UEdGraphNode + UEdGraphPin）**翻译为**可执行的字节码（UFunction::Script[]）**。翻译的最终产物是 `UBlueprintGeneratedClass`，它继承自 `UClass`，因此编译后的蓝图类**就是标准的 UObject 类型**，拥有 CDO、属性链表、函数映射表等所有 UObject 反射系统特性。

```
源代码：UEdGraph (事件图、函数图、宏图)
    │
    ▼ [FKismetCompilerContext]
    │
编译产物：UBlueprintGeneratedClass : UClass
    ├── UFunction* × N  (每个蓝图函数/事件，Script[] 存储字节码)
    ├── FProperty* × N  (每个蓝图变量，标准反射属性)
    └── CDO             (类默认对象，存储变量默认值)
```

### 1.2 源码位置

| 模块 | 路径 | 内容 |
|------|------|------|
| KismetCompiler 头文件 | `Engine/Source/Editor/KismetCompiler/Public/` | 编译器接口、中间表示定义 |
| KismetCompiler 实现 | `Engine/Source/Editor/KismetCompiler/Private/` | 编译器实现、字节码后端 |
| 虚拟机 (CoreUObject) | `Engine/Source/Runtime/CoreUObject/Public/UObject/Script.h` | 字节码指令枚举、执行引擎常量 |
| 虚拟机 (CoreUObject) | `Engine/Source/Runtime/CoreUObject/Public/UObject/Stack.h` | FFrame 栈帧定义 |
| 虚拟机 (CoreUObject) | `Engine/Source/Runtime/CoreUObject/Private/UObject/ScriptCore.cpp` | 主执行循环、全部操作码处理器 |
| 编译管理 | `Engine/Source/Editor/Kismet/Public/BlueprintCompilationManager.h` | 批量编译调度 |

### 1.3 核心编译类型

```cpp
// KismetCompiler.h 核心类型：
FKismetCompilerContext       // 编译器主类，持有所有编译状态
FKismetFunctionContext        // 单个函数的编译上下文
FKismetCompilerVMBackend      // 字节码生成后端
FNodeHandlingFunctor          // 节点处理函子基类
FBPTerminal                   // IR：变量/字面量的抽象引用
FBlueprintCompiledStatement   // IR：编译期语句（三地址码形式）
FScriptBytecodeWriter         // 字节码写入器（序列化操作码+操作数到 Script[] 数组）
```

---

## 2. 编译流水线

### 2.1 入口：Compile()

```
FKismetCompilerContext::Compile()
    ├── CompileClassLayout(EInternalCompilerFlags::None)   // 阶段1：类布局
    └── CompileFunctions(EInternalCompilerFlags::None)     // 阶段2：函数编译+字节码生成
```

`Compile()` 方法极其简洁——仅两行调用。但每个阶段内部有复杂的子流程。

### 2.2 阶段1：CompileClassLayout()

此阶段负责构建类结构：创建 UFunction 对象、创建 FProperty、建立类型签名信息。**此阶段不生成字节码。**

```
CompileClassLayout(InternalFlags)
│
├── 1. PreCompile()
│   └── 广播 OnPreCompile 事件，派生类扩展点
│
├── 2. 验证父类 & 确保 GeneratedClass
│   ├── check(Blueprint->ParentClass && ParentClass->GetPropertiesSize())
│   ├── EnsureProperGeneratedClass(TargetUClass)  // 确保是 UBlueprintGeneratedClass
│   └── 如果 GeneratedClass == SkeletonGeneratedClass，创建新类
│
├── 3. EarlyValidation（仅 Full 编译）
│   └── 遍历所有图，调用所有 UK2Node::EarlyValidation()
│
├── 4. ValidateVariableNames()
│   └── 验证变量名不与父类冲突、名称合法
│
├── 5. ValidateComponentClassOverrides()（如允许）
│   └── 验证组件类覆盖合法性
│
├── 6. CleanAndSanitizeClass(TargetClass, OldCDO)
│   ├── 保存 OldCDO 引用（用于 CopyTermDefaultsToDefaultObject）
│   ├── 清除 TargetClass 上的所有属性和函数（PurgeClass）
│   ├── 移除旧属性和旧函数对象
│   └── 保存需要跨重编译存活的子对象
│
├── 7. 设置 ClassFlags
│   ├── NewClass->ClassFlags |= (ParentClass->ClassFlags & CLASS_Inherit)
│   ├── NewClass->ClassCastFlags |= ParentClass->ClassCastFlags
│   └── 如果是接口蓝图：CLASS_Interface
│
├── 8. RegisterClassDelegateProxiesFromBlueprint()
│   └── 扫描函数图和事件图，注册委托代理及捕获变量
│
├── 9. CreateClassVariablesFromBlueprint()
│   ├── 遍历 Blueprint->NewVariables
│   ├── 为每个变量调用 CreateVariable() 创建 FProperty 对象
│   └── 属性挂在 NewClass 上，通过 PropertyLink 链入
│
├── 10. AddInterfacesFromBlueprint(NewClass)
│   ├── 遍历 Blueprint->ImplementedInterfaces
│   └── 在 NewClass->Interfaces 中添加 FImplementedInterface 条目
│
├── 11. CreateFunctionList()
│   │
│   ├── a. CreateAndProcessUbergraph()
│   │   ├── 创建 ConsolidatedEventGraph（合并后的事件图）
│   │   ├── MergeUbergraphPagesIn() 合并所有事件图页
│   │   ├── ExpandTunnelsAndMacros() 展开隧道和宏
│   │   ├── ExpansionStep() 节点展开
│   │   ├── 裁剪孤立节点
│   │   ├── 创建事件桩（Event Stub）：为每个 UK2Node_Event 创建代理函数图
│   │   ├── 验证图中无野卡引脚
│   │   └── CreateFunctionContext() → FunctionList 添加 UbergraphContext
│   │
│   ├── b. 处理 DelegateSignatureGraphs
│   │   └── 每个委托签名图 → ProcessOneFunctionGraph()
│   │
│   ├── c. 处理接口图
│   │   └── 为每个接口函数创建桩函数
│   │
│   └── d. 处理函数图
│       ├── 遍历 Blueprint->FunctionGraphs
│       ├── 遍历 Blueprint->MacroGraphs
│       └── ProcessOneFunctionGraph(SourceGraph)
│           ├── 克隆源图（编译副本）
│           ├── RegisterConvertibleDelegates() 检测可转换委托
│           ├── ExpandTunnelsAndMacros() 展开隧道和宏实例
│           ├── ReplaceConvertibleDelegates() 替换可转换委托
│           ├── ExpansionStep() 节点展开（所有 UK2Node::ExpandNode）
│           ├── 裁剪孤立节点
│           ├── ValidateSelfPinsInGraph() 验证自引脚
│           ├── VerifyValidOverrideEvent/Function() 验证覆盖合法性
│           ├── ValidateNoWildcardPinsInGraph() 验证无野卡
│           └── CreateFunctionContext() → FunctionList
│
├── 12. PrecompileFunction() × N
│   │ （先处理委托签名函数，然后处理其他函数）
│   │
│   ├── a. PruneIsolatedNodes()
│   │   └── 从 EntryPoint 遍历执行连线，裁剪不可达节点
│   │
│   ├── b. TransformNodes(Context)
│   │   └── 遍历 LinearExecutionList，对每个节点调用 FNodeHandlingFunctor::Transform()
│   │       └── 节点在编译前进行转换，如 CallFunction 替换为特定的变体节点
│   │
│   ├── c. 创建 UFunction 对象
│   │   ├── 设置函数名称、FunctionFlags（FUNC_BlueprintCallable 等）
│   │   ├── 设置 metadata（友好名称、工具提示、分类等）
│   │   └── 将 UFunction 作为子对象挂载到 NewClass
│   │
│   ├── d. CreateParametersForFunction()
│   │   └── 为函数入口节点的每个输入/输出引脚创建 FProperty 参数
│   │       └── 参数属性通过 UFunction::ChildProperties 链接
│   │
│   ├── e. CreateLocalVariablesForFunction()
│   │   ├── 创建局部变量属性
│   │   ├── 创建事件图局部变量（EventGraphLocals）
│   │   │   如果 UsePersistentUberGraphFrame()，变量放到持久帧
│   │   └── 创建用户自定义局部变量
│   │
│   ├── f. CreateExecutionSchedule()
│   │   └── 拓扑排序节点执行顺序
│   │       ├── 从 EntryPoint 开始
│   │       ├── 沿 exec 连线广度优先遍历
│   │       ├── 纯函数节点推迟到被非纯函数节点引用时
│   │       └── 结果写入 Context.LinearExecutionList
│   │
│   └── g. RegisterNets() × 所有节点
│       └── 为每个节点的每个引脚创建 FBPTerminal
│           ├── 执行引脚可能不创建 Terminal
│           ├── 数据引脚创建对应的 Local/Literal/Parameter Terminal
│           └── 注册到 Context.NetMap[Pin] = Terminal
│
├── 13. InitializeGeneratedEventNodes()
│   └── 委托签名函数编译后，绑定生成的事件节点
│
├── 14. 创建 UberGraph Frame 属性（如果需要）
│   └── CreateVariable(FPointerToUberGraphFrame::StaticStruct())
│
└── 15. Bind() + StaticLink()
    ├── NewClass->Bind()   // 将类绑定到父类
    └── NewClass->StaticLink(true)   // 链接属性链表（递归 SuperStruct）
```

### 2.3 阶段2：CompileFunctions()

此阶段负责生成字节码。在阶段1已经构建了类结构和函数签名后，此阶段生成每个函数的 `Script[]` 字节码数组。

```
CompileFunctions(InternalFlags)
│
├── 0. [可选] CreateLocalsAndRegisterNets()
│   └── 如果 PostponeLocalsGenerationUntilPhaseTwo 标志置位
│
├── 1. ForEach Valid Function：
│   │
│   ├── a. CompileFunction(Context)
│   │   │
│   │   ├── 遍历 Context.LinearExecutionList（按拓扑排序）
│   │   │
│   │   ├── 对每个节点：
│   │   │   ├── 查找该节点类型的 FNodeHandlingFunctor（通过 NodeHandlers 映射）
│   │   │   ├── 调用 Functor->Compile(Context, Node)
│   │   │   │   ├── 生成一系列 FBlueprintCompiledStatement
│   │   │   │   ├── 添加到 Context.StatementsPerNode[Node]
│   │   │   │   └── 同时加入 Context.AllGeneratedStatements（无序清单）
│   │   │   └── 纯函数链内联：
│   │   │       └── 如果当前节点是非纯函数，且其输入引脚连自纯函数，
│   │   │           则将纯函数的语句列表复制到当前节点之前
│   │   │
│   │   └── 纯函数去重：
│   │       └── 如果同一纯函数被多次引用且值依赖相同，可复用之前的结果
│   │
│   ├── b. PostcompileFunction(Context)
│   │   │
│   │   ├── ResolveGotoFixups()
│   │   │   └── 将 goto 语句的目标：从 Pin 引用 → 实际的 Statement 指针
│   │   │
│   │   ├── ResolveStatements()
│   │   │   ├── ResolveGotoFixups()
│   │   │   ├── FinalSortLinearExecList()  // 最终排序
│   │   │   │   └── 按跳转目标和执行依赖重新线性排序
│   │   │   ├── MergeAdjacentStates()
│   │   │   │   └── 消去紧邻的无条件跳转（跳到下一条语句直接 fall-through）
│   │   │   └── 标记 bUseFlowStack（如果语句需要执行流栈）
│   │   │
│   │   └── FinishCompilingFunction(Context)
│   │       ├── 为函数设置最终 Metadata 和 Flags
│   │       ├── UFunction::Bind()  // 绑定到对象
│   │       └── UFunction::StaticLink()  // 链接属性偏移
│   │
│   └── 收集整个类的必要元数据：NumReplicatedProperties 等
│
├── 2. FinishCompilingClass()
│   ├── 设置 ClassWithin、ClassConfigName 等
│   ├── 设置 HasInstrumentation 标志
│   └── 设置 bHasNonImportedCookedData
│
├── 3. PropagateValuesToCDO(NewCDO, OldCDO)
│   ├── 从 OldCDO 复制所有属性值到 NewCDO
│   ├── 应用 CopyTermDefaultsToDefaultObject()
│   │   └── 从 DefaultPropertyValueMap 应用编译期记录的默认值
│   └── PostCDOCompiled() 通知 CDO 编译完成
│
├── 4. FKismetCompilerVMBackend::GenerateCodeFromClass()
│   │
│   ├── 对每个函数调用 ConstructFunction(FuncContext, bIsUbergraph, bGenerateStubOnly)
│   │   │
│   │   ├── FScriptBytecodeWriter 写入字节码
│   │   │
│   │   ├── 对每个 FBlueprintCompiledStatement：
│   │   │   └── GenerateCodeForStatement(Writer, Statement)
│   │   │       根据 Statement.Type 写入对应的 EExprToken 操作码和操作数
│   │   │
│   │   ├── 写入 EX_Return + EX_Nothing（函数结束）
│   │   └── 字节码存入 Function->Script[]
│   │
│   └── 可选的桩生成模式：只生成函数框架不含函数体
│
├── 5. BuildDynamicBindingObjects()
│   └── 将事件节点绑定到对应的委托
│
├── 6. PostCompile()
│   └── 广播 OnPostCompile 事件
│
└── 7. CRC 签名计算
    └── 计算编译签名的 CRC 值，存入 Blueprint->CrcLastCompiledSignature
```

### 2.4 编译完整时序图

```
编辑器点击 Compile
    │
    ▼
FBlueprintCompilationManager::CompileSynchronously()
    │
    ▼
FKismetCompilerContext::Compile()
    │
    ├──► CompileClassLayout()
    │    ├── CleanAndSanitizeClass()       [清除旧产物]
    │    ├── CreateClassVariables()        [创建 FProperty]
    │    ├── CreateFunctionList()          [创建 UFunction 壳]
    │    │    ├── ExpandTunnelsAndMacros() [展开宏/隧道]
    │    │    └── ExpansionStep()          [UK2Node::ExpandNode()]
    │    └── PrecompileFunction() × N      [创建参数/局部变量/执行调度]
    │
    ├──► CompileFunctions()
    │    ├── CompileFunction() × N         [生成 FBlueprintCompiledStatement]
    │    ├── PostcompileFunction() × N     [Resolve+Sort+Merge]
    │    └── GenerateCodeFromClass()       [生成字节码→UFunction::Script[]]
    │
    └──► PostCDOCompiled()
         └── NewClass->ClassDefaultObject->PostCDOCompiled()
```

---

## 3. 编译数据模型

### 3.1 编译状态持有者：FKismetCompilerContext

编译器的主上下文，在编译期间持有所有状态和中间数据。其核心成员分为几个层次：

**输入数据：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `Blueprint` | `UBlueprint*` | 被编译的蓝图资产 |
| `Schema` | `UEdGraphSchema_K2*` | 图的模式（定义引脚类别常量、连接规则等） |
| `CompileOptions` | `FKismetCompilerOptions` | 编译选项（完全/骨架/增量等） |

**输出产物：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `NewClass` | `UBlueprintGeneratedClass*` | 编译生成的类 |
| `OldClass` | `UBlueprintGeneratedClass*` | 旧的生成类（重编译时存在） |

**中间状态：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `FunctionList` | `TIndirectArray<FKismetFunctionContext>` | 全部函数的编译上下文 |
| `NodeHandlers` | `TMap<TSubclassOf<UEdGraphNode>, FNodeHandlingFunctor*>` | 节点类型 → 处理器映射 |
| `ConsolidatedEventGraph` | `UEdGraph*` | 合并后的事件图 |
| `UbergraphContext` | `FKismetFunctionContext*` | 事件图合并后的函数上下文 |
| `DefaultPropertyValueMap` | `TMap<FName, FString>` | 属性默认值映射 |
| `TimelineToMemberVariableMap` | `TMap<UTimelineTemplate*, FProperty*>` | 时间轴 → 成员变量映射 |
| `PostCDOCompileSteps` | `TArray<TFunction<...>>` | CDO 编译后的延迟步骤 |
| `CallsIntoUbergraph` | `TMap<UEdGraphNode*, UEdGraphNode*>` | 调用事件图的节点映射 |

**内部编译器标志（EInternalCompilerFlags）：**
| 标志 | 说明 |
|------|------|
| `PostponeLocalsGenerationUntilPhaseTwo` | 局部变量生成推迟到第二阶段 |
| `PostponeDefaultObjectAssignmentUntilReinstancing` | CDO 赋值推迟到重实例化 |
| `SkipRefreshExternalBlueprintDependencyNodes` | 跳过刷新外部蓝图依赖节点 |

### 3.2 函数编译上下文：FKismetFunctionContext

单个蓝图函数的完整编译上下文。对应 `FunctionList` 中的每一项。

**来源数据：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `Blueprint` | `UBlueprint*` | 所属蓝图 |
| `SourceGraph` | `UEdGraph*` | 编译后的源图（经过展开和转换） |
| `EntryPoint` | `UK2Node_FunctionEntry*` | 函数入口节点 |
| `Schema` | `const UEdGraphSchema_K2*` | 图模式 |

**产物：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `Function` | `UFunction*` | 生成的 UFunction 对象 |
| `NewClass` | `UBlueprintGeneratedClass*` | 所属的生成类 |
| `LinearExecutionList` | `TArray<UEdGraphNode*>` | 节点拓扑排序后的线性执行顺序 |
| `StatementsPerNode` | `TMap<UEdGraphNode*, TArray<FBlueprintCompiledStatement*>>` | 每个节点生成的语句列表 |
| `AllGeneratedStatements` | `TArray<FBlueprintCompiledStatement*>` | 所有动态分配的语句（无序，用于清理） |

**终端容器——六类终端数组：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `Parameters` | `TIndirectArray<FBPTerminal>` | 函数参数终端（入口出口的输入输出） |
| `Results` | `TIndirectArray<FBPTerminal>` | 返回值终端 |
| `VariableReferences` | `TIndirectArray<FBPTerminal>` | 类成员变量引用终端 |
| `PersistentFrameVariableReferences` | `TIndirectArray<FBPTerminal>` | 持久帧变量引用终端（UberGraph 专用） |
| `Literals` | `TIndirectArray<FBPTerminal>` | 字面量终端 |
| `Locals` | `TIndirectArray<FBPTerminal>` | 局部变量终端 |
| `EventGraphLocals` | `TIndirectArray<FBPTerminal>` | 事件图局部变量终端 |
| `LevelActorReferences` | `TIndirectArray<FBPTerminal>` | 关卡 Actor 引用终端 |
| `InlineGeneratedValues` | `TIndirectArray<FBPTerminal>` | 内联生成值（无局部变量的直接值传递） |

**映射表：**
| 成员 | 类型 | 说明 |
|------|------|------|
| `NetMap` | `TMap<UEdGraphPin*, FBPTerminal*>` | 引脚到终端的映射（核心查找表） |
| `LiteralHackMap` | `TMap<UEdGraphPin*, FBPTerminal*>` | 字面量引脚的额外映射 |
| `GotoFixupRequestMap` | `TMap<FBlueprintCompiledStatement*, UEdGraphPin*>` | goto 修复请求：语句→目标执行引脚 |
| `ImplicitCastMap` | `TMap<UEdGraphPin*, FImplicitCastParams>` | 隐式类型转换（float↔double） |

**特性标志：**
| 标志 | 说明 |
|------|------|
| `bIsUbergraph` | 是否为事件图（UberGraph）合并函数 |
| `bCannotBeCalledFromOtherKismet` | 不可从其他蓝图调用（内部/仅C++使用） |
| `bIsInterfaceStub` | 是否为接口桩函数 |
| `bIsConstFunction` | 是否为 const 函数 |
| `bCreateDebugData` | 是否需要创建调试数据 |
| `bUseFlowStack` | 是否使用执行流栈（Latent 函数必需） |

---

## 4. 中间表示 IR

蓝图编译器使用**两级中间表示**。第一级是 `FBPTerminal`——对变量、字面量、参数等"数据终端"的抽象引用。第二级是 `FBlueprintCompiledStatement`——类似三地址码（Three Address Code）的操作语句。

### 4.1 第一层 IR：FBPTerminal（终端）

终端是"值"的抽象——它可以是变量引用、字面量常量、函数参数、返回值等。它统一了编译器处理各种数据来源的方式。

```cpp
struct FBPTerminal
{
    FString Name;                         // 终端名称
    FEdGraphPinType Type;                 // 终端类型信息
    bool bIsLiteral;                      // 是否为字面量
    bool bIsConst;                        // 是否为常量
    bool bIsSavePersistent;               // 是否保存到持久帧
    bool bPassedByReference;              // 是否按引用传递

    UObject* Source;                      // 来源对象（节点）
    UEdGraphPin* SourcePin;               // 来源引脚
    FBPTerminal* Context;                 // 上下文终端（如果该终端需要上下文访问）

    FProperty* AssociatedVarProperty;     // 关联的 FProperty（非字面量终端）
    TObjectPtr<UObject> ObjectLiteral;    // 对象字面量
    FText TextLiteral;                    // FText 字面量
    FString PropertyDefault;              // 属性默认值字符串
    FBlueprintCompiledStatement* InlineGeneratedParameter;  // 内联生成参数

private:
    EVarType VarType;                     // 变量引用类型（互斥）
    EContextType ContextType;             // 上下文类型（互斥）
};
```

#### 4.1.1 变量引用类型（EVarType）

终端的 `VarType` 决定了虚拟机如何访问这个终端所代表的值：

| EVarType | 字节码指令 | 说明 |
|----------|-----------|------|
| `EVarType_Local` | `EX_LocalVariable` | 函数局部变量，存储在 FFrame::Locals 帧内存中 |
| `EVarType_Default` | `EX_DefaultVariable` | 类的默认变量，从 CDO 读取 |
| `EVarType_Instanced` | `EX_InstanceVariable` | 实例变量（`this->Property`），从对象实例读取 |
| `EVarType_SparseClassData` | `EX_ClassSparseDataVariable` | 稀疏类数据变量（存储在 USparseDelegateFunction 的单独数据块中） |

#### 4.1.2 上下文类型（EContextType）

当终端本身用作函数调用的"目标上下文"时，`ContextType` 指示这是哪种上下文：

| EContextType | 说明 |
|-------------|------|
| `EContextType_Object` | 普通对象上下文 |
| `EContextType_Class` | 类上下文（用于调用静态函数或访问 CDO） |
| `EContextType_Struct` | 结构体上下文 |

#### 4.1.3 终端的关键方法

| 方法 | 说明 |
|------|------|
| `CopyFromPin(UEdGraphPin*, FString)` | 从引脚复制类型和名称信息 |
| `IsTermWritable()` | 是否可写（非字面量且非常量） |
| `IsLocalVarTerm()` | 是否为局部变量终端 |
| `IsDefaultVarTerm()` | 是否为默认变量终端 |
| `IsInstancedVarTerm()` | 是否为实例变量终端 |
| `IsSparseClassDataVarTerm()` | 是否为稀疏类数据终端 |

### 4.2 第二层 IR：FBlueprintCompiledStatement（编译语句）

编译语句类似于三地址码，表示一个独立的"操作"。每种操作由一个 `EKismetCompiledStatementType` 枚举值和若干操作数组成。

```cpp
struct FBlueprintCompiledStatement
{
    EKismetCompiledStatementType Type;      // 语句类型

    FBPTerminal* FunctionContext;           // 函数上下文对象（KCST_CallFunction）
    UFunction* FunctionToCall;              // 要调用的 UFunction（KCST_CallFunction）
    FBlueprintCompiledStatement* TargetLabel; // 跳转目标（KCST_Goto 等）

    int32 UbergraphCallIndex;               // 事件图调用参数索引
    FBPTerminal* LHS;                       // 左值：赋值目标 / 函数返回值
    TArray<FBPTerminal*> RHS;               // 右值：参数列表 / 赋值来源

    bool bIsJumpTarget;                     // 是否为跳转目标
    bool bIsInterfaceContext;               // 是否为接口上下文
    bool bIsParentContext;                  // 是否为父类上下文（Super）

    UEdGraphPin* ExecContext;               // 执行引脚上下文（KCST_WireTraceSite）
    TArray<UEdGraphPin*> PureOutputContextArray;  // 纯节点输出引脚上下文
    FString Comment;                        // 注释文本
};
```

#### 4.2.1 完整语句类型枚举

```cpp
enum EKismetCompiledStatementType
{
    KCST_Nop = 0,                    // 无操作
    KCST_CallFunction = 1,           // 调用 FunctionToCall(LHS, RHS...)
    KCST_Assignment = 2,             // LHS = RHS[0]
    KCST_CompileError = 3,           // 编译错误
    KCST_UnconditionalGoto = 4,      // goto TargetLabel
    KCST_PushState = 5,              // FlowStack.Push(TargetLabel) — Latent 核心
    KCST_GotoIfNot = 6,              // if (!RHS[0]) goto TargetLabel
    KCST_Return = 7,                 // return RHS[0]
    KCST_EndOfThread = 8,            // if (FlowStack.Num()) Pop+goto else return
    KCST_Comment = 9,                // 注释（调试用）
    KCST_ComputedGoto = 10,          // goto 到由 RHS 计算出的索引

    KCST_EndOfThreadIfNot = 11,      // if (!RHS[0]) EndOfThread
    KCST_DebugSite = 12,             // NOP 带地址记录（断点目标）
    KCST_CastObjToInterface = 13,    // Interface = ObjToInterface(Object)
    KCST_DynamicCast = 14,           // DynamicCast<TargetClass>(Object)
    KCST_ObjectToBool = 15,          // Object != nullptr
    KCST_AddMulticastDelegate = 16,  // Delegate->Add(EventDelegate)
    KCST_ClearMulticastDelegate = 17,// Delegate->Clear()
    KCST_WireTraceSite = 18,         // 连线追踪点（编辑器执行高亮）
    KCST_BindDelegate = 19,          // 绑定对象+函数名到委托
    KCST_RemoveMulticastDelegate = 20,// Delegate->Remove(EventDelegate)
    KCST_CallDelegate = 21,          // Delegate->Broadcast(...)
    KCST_CreateArray = 22,           // 构建数组值
    KCST_CrossInterfaceCast = 23,    // CrossInterfaceCast(Interface)
    KCST_MetaCast = 24,              // MetaCass<TargetClass>(Object)
    KCST_AssignmentOnPersistentFrame = 25, // LHS_on_PersistentFrame = RHS[0]
    KCST_CastInterfaceToObj = 26,    // InterfaceToObjCast(Interface)
    KCST_GotoReturn = 27,            // goto 到函数的返回标签
    KCST_GotoReturnIfNot = 28,       // if (!RHS[0]) goto 返回标签

    KCST_SwitchValue = 29,           // 值匹配 Switch（编译为 EX_SwitchValue）
    KCST_DoubleToFloatCast = 30,     // (float)DoubleValue
    KCST_FloatToDoubleCast = 31,     // (double)FloatValue

    // 仪表化扩展（用于性能分析）
    KCST_InstrumentedEvent,          // 仪表事件
    KCST_InstrumentedEventStop,      // 仪表事件停止
    KCST_InstrumentedPureNodeEntry,  // 仪表纯节点入口
    KCST_InstrumentedWireEntry,      // 仪表连线入口
    KCST_InstrumentedWireExit,       // 仪表连线出口
    KCST_InstrumentedStatePush,      // 仪表状态推入
    KCST_InstrumentedStateRestore,   // 仪表状态恢复
    KCST_InstrumentedStateReset,     // 仪表状态重置
    KCST_InstrumentedStateSuspend,   // 仪表状态挂起
    KCST_InstrumentedStatePop,       // 仪表状态弹出
    KCST_InstrumentedTunnelEndOfThread, // 仪表隧道线索内

    KCST_ArrayGetByRef,              // 按引用获取数组元素（外部可能需要内部元素地址）
    KCST_CreateSet,                  // 构建 Set 值
    KCST_CreateMap,                  // 构建 Map 值
};
```

#### 4.2.2 语句与字节码的映射关系

| IR 语句（KCST_*） | 生成的字节码 |
|-------------------|-------------|
| `KCST_Nop` | `EX_Nothing` |
| `KCST_CallFunction` | `EX_Context`/`EX_FinalFunction`/`EX_VirtualFunction` 等 |
| `KCST_Assignment` | `EX_Let`（或 `EX_LetObj`/`EX_LetBool`/`EX_LetDelegate` 等特定版本） |
| `KCST_UnconditionalGoto` | `EX_Jump` |
| `KCST_GotoIfNot` | `EX_JumpIfNot` |
| `KCST_ComputedGoto` | `EX_ComputedJump` |
| `KCST_SwitchValue` | `EX_SwitchValue` |
| `KCST_Return` | `EX_Return` |
| `KCST_PushState` | `EX_PushExecutionFlow` |
| `KCST_EndOfThread` | `EX_PopExecutionFlow` |
| `KCST_EndOfThreadIfNot` | `EX_PopExecutionFlowIfNot` |
| `KCST_DebugSite` | `EX_Breakpoint` |
| `KCST_WireTraceSite` | `EX_WireTracepoint` 或 `EX_Tracepoint` |
| `KCST_CastObjToInterface` | `EX_ObjToInterfaceCast` |
| `KCST_CrossInterfaceCast` | `EX_CrossInterfaceCast` |
| `KCST_CastInterfaceToObj` | `EX_InterfaceToObjCast` |
| `KCST_DynamicCast` | `EX_DynamicCast` |
| `KCST_MetaCast` | `EX_MetaCast` |
| `KCST_DoubleToFloatCast` | `EX_Cast` (CST_DoubleToFloat) |
| `KCST_FloatToDoubleCast` | `EX_Cast` (CST_FloatToDouble) |
| `KCST_CreateArray` | `EX_SetArray` ... `EX_EndArray` |
| `KCST_CreateSet` | `EX_SetSet` ... `EX_EndSet` |
| `KCST_CreateMap` | `EX_SetMap` ... `EX_EndMap` |
| `KCST_AssignmentOnPersistentFrame` | `EX_LetValueOnPersistentFrame` |
| `KCST_AddMulticastDelegate` | `EX_AddMulticastDelegate` |
| `KCST_ClearMulticastDelegate` | `EX_ClearMulticastDelegate` |
| `KCST_RemoveMulticastDelegate` | `EX_RemoveMulticastDelegate` |
| `KCST_BindDelegate` | `EX_BindDelegate` |
| `KCST_CallDelegate` | `EX_CallMulticastDelegate` |
| `KCST_ObjectToBool` | `EX_JumpIfNot`（作为条件判断的一部分） |
| `KCST_ArrayGetByRef` | `EX_ArrayGetByRef` |

---

## 5. 节点处理架构

### 5.1 架构设计

每种 `UK2Node` 子类关联一个 `FNodeHandlingFunctor` 子类，通过双重映射创建：

1. **NodeHandler 注册链**：`UK2Node::CreateNodeHandler()` 虚函数返回专用的 Handler
2. **Compiler 缓存**：编译器启动时调用所有节点的 `CreateNodeHandler()`，结果缓存在 `FKismetCompilerContext::NodeHandlers` 映射中

```
编译时：
  UEdGraphNode → NodeHandlers[Node->GetClass()] → FNodeHandlingFunctor
    ├── Transform(Context, Node) — 编译前节点转换
    ├── RegisterNets(Context, Node) — 为节点的引脚创建 FBPTerminal
    └── Compile(Context, Node)   — 生成 FBlueprintCompiledStatement 语句列表
```

### 5.2 处理函子基类

```cpp
class FNodeHandlingFunctor
{
public:
    FKismetCompilerContext& CompilerContext;

    // 主编译方法 —— 生成该节点的语句列表
    virtual void Compile(FKismetFunctionContext& Context, UEdGraphNode* Node) {}

    // 编译前转换 —— 在 PrecompileFunction 中调用，节点可以在此被替换或修改
    virtual void Transform(FKismetFunctionContext& Context, UEdGraphNode* Node) {}

    // 注册单个引脚的网络终端
    virtual void RegisterNet(FKismetFunctionContext& Context, UEdGraphPin* Pin) {}

    // 注册节点的所有引脚
    virtual void RegisterNets(FKismetFunctionContext& Context, UEdGraphNode* Node);

    // 是否需要在执行调度前注册网络（仅用于函数入口和返回节点）
    virtual bool RequiresRegisterNetsBeforeScheduling() const { return false; }

protected:
    // 解析范围变量并注册终端
    void ResolveAndRegisterScopedTerm(FKismetFunctionContext& Context,
        UEdGraphPin* Net, TIndirectArray<FBPTerminal>& NetArray);

    // 生成简单的 then goto 到下一个执行引脚
    FBlueprintCompiledStatement& GenerateSimpleThenGoto(
        FKismetFunctionContext& Context, UEdGraphNode& Node, UEdGraphPin* ThenExecPin);
    FBlueprintCompiledStatement& GenerateSimpleThenGoto(
        FKismetFunctionContext& Context, UEdGraphNode& Node);

    // 验证并注册字面量网络
    bool ValidateAndRegisterNetIfLiteral(FKismetFunctionContext& Context, UEdGraphPin* Net);
    virtual FBPTerminal* RegisterLiteral(FKismetFunctionContext& Context, UEdGraphPin* Net);
};
```

### 5.3 关键节点处理器

编译器通过 `RegisterCompilerForBP()` 静态方法注册蓝图类型对应的处理器工厂。

#### FKCHandler_CallFunction（函数调用处理器）

处理 `UK2Node_CallFunction` 及其子类。这是最复杂的处理器之一：

1. **RegisterNets**：为函数的目标引脚（self）、输入参数、输出参数创建对应的 FBPTerminal
2. **Compile**：
   - 如果目标是非纯函数：在调用前插入 InputPinValues 赋值语句
   - 创建 `KCST_CallFunction` 语句，`FunctionToCall` 指向目标 UFunction
   - 如果目标是 Latent 函数：插入 `KCST_PushState` 再进入函数调用
   - 在调用后插入 OutputPinValues 提取语句
   - 生成 `GenerateSimpleThenGoto()` 跳转到下一执行节点

#### FKCHandler_ExecutionSequence（序列处理器）

处理 `UK2Node_ExecutionSequence`：生成一系列 `KCST_UnconditionalGoto`，将执行链导向各个 then 引脚，每个 then 引脚的处理完后再跳到下一个。

#### FKCHandler_IfThenElse（分支处理器）

处理 `UK2Node_IfThenElse`：
1. 在条件引脚上生成比较代码（加载变量、比较）
2. 生成 `KCST_GotoIfNot`：条件为 false 时跳转到 else 分支
3. then 分支末尾生成 `KCST_UnconditionalGoto` 跳到汇合点
4. else 分支末尾生成同样的跳转

#### FKCHandler_Passthru（直通处理器）

最简单的处理器——仅生成一个 goto 到下一个执行节点。用于 KnotNode 等被消除后仅需传递执行的节点。

### 5.4 核心编译循环

```cpp
// CompileFunction 中每个节点的处理（简化）：
for (UEdGraphNode* Node : Context.LinearExecutionList)
{
    FNodeHandlingFunctor* Handler = NodeHandlers.FindRef(Node->GetClass());
    if (!Handler) { /* 错误: 没有已注册的 handler */ }

    Handler->Compile(Context, Node);

    // 纯函数链内联：
    // 如果当前节点有连接来自纯函数的输入引脚，
    // 递归地将这些纯函数节点的语句合并到当前节点之前
    for (UEdGraphPin* InputPin : Node->Pins)
    {
        if (InputPin->Direction == EGPD_Input && InputPin->LinkedTo.Num() > 0)
        {
            UEdGraphNode* SourceNode = InputPin->LinkedTo[0]->GetOwningNode();
            if (SourceNode->IsNodePure())
            {
                // 将该纯函数的所有语句复制到当前节点之前
                Context.CopyAndPrependStatements(Node, SourceNode);
            }
        }
    }
}
```

---

## 6. 字节码生成

### 6.1 FKismetCompilerVMBackend

`FKismetCompilerVMBackend` 是编译器的字节码后端，负责将 `FBlueprintCompiledStatement` 列表序列化为 `UFunction::Script[]` 字节码数组。

```cpp
class FKismetCompilerVMBackend : public IKismetCompilerBackend
{
protected:
    UBlueprint* Blueprint;
    UEdGraphSchema_K2* Schema;
    FCompilerResultsLog& MessageLog;
    FKismetCompilerContext& CompilerContext;

    // Ubergraph 语句标签映射（用于 goto 偏移计算）
    TMap<FBlueprintCompiledStatement*, CodeSkipSizeType> UbergraphStatementLabelMap;

public:
    // 主入口：从类生成全部代码
    void GenerateCodeFromClass(UClass* SourceClass,
        TIndirectArray<FKismetFunctionContext>& Functions, bool bGenerateStubsOnly);

protected:
    // 构建单个函数的完整字节码（头+体+尾）
    void ConstructFunction(FKismetFunctionContext& FunctionContext,
        bool bIsUbergraph, bool bGenerateStubOnly);
};
```

### 6.2 FScriptBytecodeWriter

字节码被存储为 `TArray<uint8>`，写入器 `FScriptBytecodeWriter` 是一个 `FArchive` 子类，将数据序列化到此数组：

```cpp
class FScriptBytecodeWriter : public FArchiveUObject
{
public:
    TArray<uint8>& ScriptBuffer;   // 目标字节码缓冲区

    void Serialize(void* V, int64 Length) override
    {
        // 分配空间并直接内存拷贝
        int32 iStart = ScriptBuffer.AddUninitialized(IntCastChecked<int32>(Length));
        FMemory::Memcpy(&ScriptBuffer[iStart], V, Length);
    }

    // 覆写 FName 序列化以匹配 XFERNAME 宏格式
    FArchive& operator<<(FName& Name) override;

    // 覆写 UObject* 序列化为 ScriptPointerType
    FArchive& operator<<(UObject*& Res) override;

    // 覆写 FField* 序列化为 ScriptPointerType
    FArchive& operator<<(FField*& Res) override;
};
```

关键设计：**FScriptBytecodeWriter 将对象指针和属性指针序列化为程序内的原始指针值（ScriptPointerType）**。这依赖于蓝图字节码仅在内存中执行、不跨平台存储的前提。在 SavePackage 期间，`Script[]` 中的指针需要修复为序列化引用。运行时加载时，`Script[]` 中的指针会被修正为实际内存地址。

### 6.3 语句到字节码的生成

`GenerateCodeForStatement()` 是一个巨大的 switch 语句，将每种 `EKismetCompiledStatementType` 映射为对应的字节码指令序列。以下是一些关键语句的字节码生成模式：

#### KCST_CallFunction → 函数调用字节码

```
如果目标是虚拟调用（Virtual）：
    写入 EX_Context 或 EX_Context_FailSilent
    写入 SkipOffset（如果上下文为空时需跳过的字节量）
    写入子表达式（上下文对象）
    写入 EX_VirtualFunction 或 EX_LocalVirtualFunction
    写入 FName（函数名称）
    写入 EX_EndFunctionParms
    写入 EX_Return 或 EX_Nothing（对应无返回值）

如果目标是最终调用（Final/NonVirtual，编译期已知函数指针）：
    写入 EX_FinalFunction 或 EX_LocalFinalFunction 或 EX_CallMath
    写入 UFunction* 指针
    写入 EX_EndFunctionParms
    写入 EX_Return 或 EX_Nothing
```

#### KCST_Assignment → 赋值字节码

```
LHS 是实例变量：
    写入 EX_Let（通用赋值）
    写入 FProperty*（目标属性）
    写入 RHS 表达式的字节码（变量读取或字面量）

LHS 是局部变量：
    写入 EX_Let
    写入 FProperty*（局部变量属性）

LHS 是对象指针：
    写入 EX_LetObj

LHS 是布尔：
    写入 EX_LetBool
```

#### KCST_GotoIfNot → 条件跳转字节码

```
写入 RHS 条件表达式（如 EX_InstanceVariable + FProperty*）
写入 EX_JumpIfNot
写入跳转偏移 CodeSkipSizeType（目标标签在当前字节码中的偏移）
```

#### KCST_PushState → 执行流栈字节码

```
标记目标标签为跳转目标
写入 EX_PushExecutionFlow
写入跳转偏移 CodeSkipSizeType  ← 这个地址被存入 FlowStack[]
```

### 6.4 函数字节码的最终结构

每个编译后的 `UFunction::Script[]` 包含：

```
┌─────────────────────────────────┐
│ [参数表达式1]                    │  ← 可选：参数的默认值表达式
│ [参数表达式2]                    │
│ ...                             │
│ EX_EndFunctionParms             │  ← 函数参数结束标记
├─────────────────────────────────┤
│ [函数体：一系列操作的字节码]       │  ← 由 CompileFunction 生成
│   EX_InstanceVariable  FProperty*│    读取变量
│   EX_FinalFunction  UFunction*  │    调用另一个函数
│   EX_JumpIfNot  Offset          │    条件跳转
│   ...                           │
├─────────────────────────────────┤
│ EX_Return                       │  ← 返回标记
│ [返回表达式] 或 EX_Nothing       │  ← 返回值（或无返回值）
│ EX_Nothing                      │  ← 函数体结束
└─────────────────────────────────┘
```

---

## 7. 虚拟机架构

### 7.1 设计概览

蓝图虚拟机是一个**字节码解释器**，基于以下核心设计：

1. **单字节操作码**：每个指令由 1 字节的 `EExprToken` 操作码 + 可变长度的操作数组成
2. **256 条目分发表**：`GNatives[256]` 数组存储每个操作码的处理函数指针，`FFrame::Step()` 通过 `GNatives[*Code++]` 完成单周期分发
3. **栈帧链**：每次函数调用创建新的 `FFrame`，通过 `PreviousFrame` 指针形成调用链
4. **基于属性的内存模型**：所有数据操作通过 `FProperty` 驱动——变量的大小、偏移、复制语义均由 FProperty 决定
5. **执行流栈**：Latent 函数通过 `FlowStack` 实现暂停/恢复语义

### 7.2 FFrame —— 执行栈帧

```cpp
struct FFrame : public FOutputDevice
{
    // ── 核心执行状态 ──
    UFunction* Node;              // 当前正在执行的 UFunction
    UObject* Object;              // 函数所属的对象实例 (this)
    uint8* Code;                  // 当前指令指针，指向 Script[] 中的下一字节
    uint8* Locals;                // 局部变量帧内存的起始指针

    // ── 属性求值状态 ──
    FProperty* MostRecentProperty;        // 最近读取/评估的 FProperty
    uint8* MostRecentPropertyAddress;     // 最近评估属性的内存地址
    uint8* MostRecentPropertyContainer;   // 最近评估属性的容器地址

    // ── 执行流控制 ──
    FlowStackType FlowStack;      // TArray<CodeSkipSizeType>，执行流栈
                                  // Latent 函数暂停时将返回地址压入此栈

    // ── 调用链 ──
    FFrame* PreviousFrame;        // 调用者的栈帧（调用链的上一环）

    // ── Out 参数 ──
    FOutParmRec* OutParms;        // Out 参数链表

    // ── 编译内函数路径 ──
    FField* PropertyChainForCompiledIn;  // 编译内函数的属性链
    UFunction* CurrentNativeFunction;    // 当前执行的 native 函数

    // ── 异常控制 ──
    bool bAbortingExecution;      // 中止标志：如果为 true，VM 停止执行并展开栈
    bool bArrayContextFailed;     // 数组上下文求值失败标志
};
```

**FFrame 构造过程：**

```cpp
inline FFrame::FFrame(UObject* InObject, UFunction* InNode,
    void* InLocals, FFrame* InPreviousFrame, FField* InPropertyChainForCompiledIn)
    : Node(InNode)
    , Object(InObject)
    , Code(InNode->Script.GetData())      // Code 指向 Script[] 起始位置
    , Locals((uint8*)InLocals)
    , PreviousFrame(InPreviousFrame)
    , ...
```

**FFrame 析构时**自动执行：
- 从线程局部执行追踪栈中弹出自己
- 将 `bAbortingExecution` 传播到 `PreviousTrackingFrame`（确保异常状态跨越 C++↔蓝图调用边界）

### 7.3 核心分发机制

```cpp
// GNatives[256] — 全局操作码分发表
COREUOBJECT_API FNativeFuncPtr GNatives[EX_Max];  // EX_Max = 0xFF = 256

// 单指令分发（VM 热路径中最关键的代码）
void FFrame::Step(UObject* Context, RESULT_DECL)
{
    int32 B = *Code++;                     // 读取操作码，Code 指针前进 1
    (GNatives[B])(Context, *this, RESULT_PARAM);  // 通过 256 条目表间接调用
}
```

每一步：
1. 当前 Code 指针指向的字节被解释为 `EExprToken` 操作码
2. Code 指针前进 1
3. `GNatives[opcode]` 中的处理函数被调用，传入上下文对象、当前栈帧和结果缓冲区

### 7.4 操作码注册机制

操作码在引擎启动时通过静态初始化注册：

```cpp
// GRegisterNative — 在静态初始化期间由 IMPLEMENT_VM_FUNCTION 宏调用
COREUOBJECT_API uint8 GRegisterNative(int32 NativeBytecodeIndex, const FNativeFuncPtr& Func)
{
    static bool bInitialized = false;
    if (!bInitialized)
    {
        bInitialized = true;
        // 首次注册：将所有 256 个槽位初始化为 execUndefined
        for (uint32 i = 0; i < UE_ARRAY_COUNT(GNatives); i++)
        {
            GNatives[i] = &UObject::execUndefined;
        }
    }
    // 将处理函数写入指定索引
    GNatives[NativeBytecodeIndex] = Func;
    return 0;
}
```

类似地，类型转换操作码（`ECastToken`）通过 `GCasts[CST_Max]` 表和 `GRegisterCast()` 注册。

### 7.5 主执行循环

```cpp
void ProcessLocalScriptFunction(UObject* Context, FFrame& Stack, RESULT_DECL)
{
    UFunction* Function = (UFunction*)Stack.Node;
    uint8 Buffer[MAX_SIMPLE_RETURN_VALUE_SIZE];  // 64 字节返回值缓冲区

#if DO_BLUEPRINT_GUARD
    // 检查无限循环逃逸 (BpET.Runaway > 1,000,000 次迭代)
    if (BpET.bRanaway) { /* 零值返回 + abort */ }

    // 检查递归深度 (bp.ScriptRecurseLimit，默认 120)
    if (++BpET.Recurse == GScriptRecurseLimit) { /* InfiniteLoop 异常 */ }
#endif

    // 主字节码循环
    while (*Stack.Code != EX_Return && !Stack.bAbortingExecution)
    {
#if DO_BLUEPRINT_GUARD
        if (BpET.Runaway > GMaximumScriptLoopIterations)  // 默认 1,000,000
        { /* RunawayLoop 异常 */ }
#endif
        Stack.Step(Stack.Object, Buffer);     // 1 字节读取 + 1 次表跳转
    }

    // 处理返回值
    if (!Stack.bAbortingExecution)
    {
        Stack.Code++;  // 跳过 EX_Return
        if (*Stack.Code != EX_Nothing)
            Stack.Step(Stack.Object, RESULT_PARAM);  // 求值返回表达式
        else
            Stack.Code++;  // 跳过 EX_Nothing
    }

#if DO_BLUEPRINT_GUARD
    --BpET.Recurse;  // 恢复递归计数器
#endif
}
```

### 7.6 安全防护机制

| 防护 | 默认值 | 位置 | 说明 |
|------|--------|------|------|
| `GMaximumScriptLoopIterations` | 1,000,000 | ScriptCore.cpp | 单次蓝图函数执行的字节码指令数上限 |
| `GScriptRecurseLimit` | 120 | ScriptCore.cpp | 最大蓝图递归深度 |
| `bRanaway` | false | FBlueprintContextTracker | 全局逃逸标志（一旦触发停止所有蓝图执行） |
| `bAbortingExecution` | false | FFrame | 当前帧的中止标志（传播给父帧和子帧） |

### 7.7 执行流栈（FlowStack）

执行流栈是实现 **Latent 函数（异步延迟函数）**的核心机制：

```
正常执行：
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │ 节点 A       │───→│ 节点 B       │───→│ 返回          │
  └─────────────┘     └─────────────┘     └─────────────┘

带有 Latent 函数的执行：
  ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
  │ 节点 A       │───→│ Delay(2秒)    │───→│ (返回到      │
  └─────────────┘     │ PushState(B)  │     │  引擎等待)    │
                      │ 返回引擎       │     └─────────────┘
                      └──────────────┘            │
                                          2 秒后引擎恢复执行
                                                   │
                      ┌──────────────┐             │
                      │ 节点 B       │←────────────┘
                      │ PopState → 继续执行      │
                      └──────────────┘
```

关键机制：
- `EX_PushExecutionFlow(Offset)`：将当前 Code + Offset 压入 FlowStack，然后继续执行
- `EX_PopExecutionFlow`：从 FlowStack 弹出地址并跳转
- `EX_PopExecutionFlowIfNot`：条件成立时弹出并跳转
- **Latent 函数**：调用 `UKismetSystemLibrary::Delay()` 等方法时，函数在 PushState 后返回，引擎的 `FLatentActionManager` 记住待恢复的函数。条件满足后，引擎调用 `ProcessEvent()` 恢复执行，`EX_PopExecutionFlow` 跳转到暂停前的下一个节点

---

## 8. 字节码指令集参考

### 8.1 指令总览

蓝图 VM 使用约 90 个字节码操作码（`EExprToken`）。操作码是单字节（`uint8`），允许最多 256 种指令。操作数跟在操作码之后，以可变长度形式存在。

### 8.2 变量访问指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_LocalVariable` | 0x00 | `FProperty*` | 将局部变量（Locals+Property->Offset）求值为 MostRecentPropertyAddress |
| `EX_InstanceVariable` | 0x01 | `FProperty*` | 将对象实例变量求值；FProperty 决定偏移和大小 |
| `EX_DefaultVariable` | 0x02 | `FProperty*` | 从 CDO 读取默认变量值 |
| `EX_LocalOutVariable` | 0x48 | `FProperty*` | 局部 out 参数（引用传递）；通过 FOutParmRec 链表返回地址 |
| `EX_ClassSparseDataVariable` | 0x6C | `FProperty*` | 访问稀疏类数据变量（`SparseClassDataStruct` 中的成员） |

### 8.3 赋值指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_Let` | 0x0F | `FProperty*`, 值表达式 | 通用赋值：`FProperty::CopyCompleteValueFromVariable(Dest, Src)` |
| `EX_LetBool` | 0x14 | `bool` 表达式 | 布尔赋值。优化版本：直接使用 `true`/`false` 字面值而非 IntConst |
| `EX_LetObj` | 0x5F | `FObjectPropertyBase*`, 对象值 | 对象指针赋值：`FObjectPropertyBase::SetObjectPropertyValue()` |
| `EX_LetWeakObjPtr` | 0x60 | `FWeakObjectProperty*`, 对象值 | 弱对象指针赋值 |
| `EX_LetDelegate` | 0x44 | `FDelegateProperty*`, 委托值 | 单播委托赋值 |
| `EX_LetMulticastDelegate` | 0x43 | `FMulticastDelegateProperty*`, 委托值 | 多播委托赋值 |
| `EX_LetValueOnPersistentFrame` | 0x64 | `FProperty*`, 值 | 向持久帧（UberGraph Frame）赋值 |
| `EX_BitFieldConst` | 0x11 | `FBoolProperty*`, bool 值 | 向单个位字段赋值 |

### 8.4 函数调用指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_FinalFunction` | 0x1C | `UFunction*` | 调用非虚函数（编译期已确定目标，通过指针调用） |
| `EX_LocalFinalFunction` | 0x46 | `UFunction*` | 本地最终函数调用（优化版：跳过 CallSpace 检查，假定本地执行） |
| `EX_VirtualFunction` | 0x1B | `FName`（函数名） | 通过名称调用虚函数（运行时查找 vtable 或 FuncMap） |
| `EX_LocalVirtualFunction` | 0x45 | `FName`（函数名） | 本地虚函数调用（优化版：跳过 CallSpace 检查） |
| `EX_CallMath` | 0x68 | `UFunction*` | 调用纯数学函数（无副作用，直接在 CDO 上执行以提高安全性） |
| `EX_CallMulticastDelegate` | 0x63 | （栈上已有委托引用） | 广播多播委托：遍历 `InvocationList`，对每个接收者调用 `ProcessEvent()` |
| `EX_Context` | 0x19 | SkipOffset, 上下文表达式, 调用指令 | 切换对象上下文并调用函数（将 Object 替换为上下文求值结果） |
| `EX_Context_FailSilent` | 0x1A | SkipOffset, 上下文表达式, 调用指令 | 同上，但上下文为空时静默跳过（不报错） |
| `EX_ClassContext` | 0x12 | SkipOffset, 类上下文表达式, 调用指令 | 切换到类默认对象上下文（静态函数调用） |
| `EX_InterfaceContext` | 0x51 | SkipOffset, 接口表达式, 调用指令 | 切换到接口上下文 |
| `EX_StructMemberContext` | 0x42 | `FProperty*`, `FProperty*`（结构体内的属性） | 切换上下文到结构体成员 |
| `EX_EndFunctionParms` | 0x16 | 无 | 函数参数结束标记（用于跳过快进非默认参数） |

### 8.5 控制流指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_Jump` | 0x06 | `CodeSkipSizeType` | 无条件跳转到 Code + Offset |
| `EX_JumpIfNot` | 0x07 | 条件表达式, `CodeSkipSizeType` | 条件为 false/0/null 时跳转 |
| `EX_ComputedJump` | 0x4E | 整数表达式 | 跳转到 Code + (整数 * sizeof(CodeSkipSizeType)) |
| `EX_SwitchValue` | 0x69 | 值表达式, N, N×(值, 偏移), 默认偏移 | 值匹配跳转：比较值与 N 个 case，匹配则跳到对应偏移，否则默认偏移 |
| `EX_PushExecutionFlow` | 0x4C | `CodeSkipSizeType` | 将 (当前 Code + Offset) 压入 FlowStack |
| `EX_PopExecutionFlow` | 0x4D | 无 | 如果 FlowStack 非空：弹出地址并跳转；否则：如同 Return |
| `EX_PopExecutionFlowIfNot` | 0x4F | 条件, 无 | 条件为 false 时执行 `EX_PopExecutionFlow` |
| `EX_Return` | 0x04 | 可选的返回表达式 | 函数返回：退出主循环 |
| `EX_EndOfScript` | 0x53 | 无 | 脚本结束标记 |

### 8.6 上下文与对象指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_Self` | 0x17 | 无 | 获取 Self 对象（当前函数所属的 Object） |
| `EX_NoObject` | 0x2A | 无 | 空对象常量 |
| `EX_NoInterface` | 0x2D | 无 | 空接口常量 |
| `EX_StaticLoadObject` | (已废弃) | — | — |
| `EX_DeprecatedOp4A` | 0x4A | — | 被 EX_InstanceDelegate 取代的废弃操作码 |

### 8.7 常量指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_IntConst` | 0x1D | `int32` | 32 位整数常量 |
| `EX_Int64Const` | 0x35 | `int64` | 64 位整数常量 |
| `EX_UInt64Const` | 0x36 | `uint64` | 64 位无符号整数常量 |
| `EX_FloatConst` | 0x1E | `float` | 单精度浮点常量 |
| `EX_DoubleConst` | 0x37 | `double` | 双精度浮点常量 |
| `EX_IntConstByte` | 0x2C | `uint8` | 单字节整数常量（0-255） |
| `EX_IntZero` | 0x25 | 无 | 整型零 |
| `EX_IntOne` | 0x26 | 无 | 整型一 |
| `EX_True` | 0x27 | 无 | 布尔 True |
| `EX_False` | 0x28 | 无 | 布尔 False |
| `EX_StringConst` | 0x1F | ANSI 字符串（null-terminated） | ANSI 字符串常量 |
| `EX_UnicodeStringConst` | 0x34 | UTF-16 字符串（长度前缀） | Unicode 字符串常量 |
| `EX_TextConst` | 0x29 | `EBlueprintTextLiteralType`, 字符串数据 | FText 常量（支持 5 种模式） |
| `EX_NameConst` | 0x21 | `FName`（ComparisonIndex, DisplayIndex, Number） | FName 常量 |
| `EX_ObjectConst` | 0x20 | `UObject*` | 硬对象引用常量 |
| `EX_SoftObjectConst` | 0x67 | `FSoftObjectPath` | 软对象引用常量 |
| `EX_FieldPathConst` | 0x6D | `FFieldPath` | 字段路径常量 |
| `EX_StructConst` | 0x2F | `UScriptStruct*`, 成员值, `EX_EndStructConst` | 结构体字面量常量 |
| `EX_VectorConst` | 0x23 | `FVector3f`（3×float） | 向量常量 |
| `EX_Vector3fConst` | 0x41 | `FVector3f`（3×float） | Vector3f 常量 |
| `EX_RotationConst` | 0x22 | `FRotator3f`（Pitch, Yaw, Roll float） | 旋转常量 |
| `EX_TransformConst` | 0x2B | `FTransform3f`（Rotation, Translation, Scale） | 变换常量 |
| `EX_PropertyConst` | 0x33 | `FProperty*` | FProperty 指针常量（用于反射操作） |
| `EX_SkipOffsetConst` | 0x5B | `CodeSkipSizeType` | 跳过偏移常量（用于 EX_Skip） |

**FText 常量模式（EBlueprintTextLiteralType）：**
| 模式 | 说明 | 操作数 |
|------|------|--------|
| `Empty` | 空文本（`FText::GetEmpty()`） | 无 |
| `LocalizedText` | 本地化文本 | Source, Key, Namespace 三个字符串 |
| `InvariantText` | 文化不变文本 | 一个字符串 |
| `LiteralString` | 字面量字符串（`FText::FromString()`） | 一个字符串 |
| `StringTableEntry` | 字符串表条目 | TableID, Key 两个字符串 |

**

### 8.8 容器指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_SetArray` | 0x31 | 元素表达式* N, `EX_EndArray` | 填充数组元素（末尾追加） |
| `EX_EndArray` | 0x32 | 无 | 数组构建结束 |
| `EX_SetSet` | 0x39 | 元素表达式* N, `EX_EndSet` | 填充 Set 元素 |
| `EX_EndSet` | 0x3A | 无 | Set 构建结束 |
| `EX_SetMap` | 0x3B | (键表达式, 值表达式)* N, `EX_EndMap` | 填充 Map 元素 |
| `EX_EndMap` | 0x3C | 无 | Map 构建结束 |
| `EX_ArrayConst` | 0x65 | 元素表达式* N, `EX_EndArrayConst` | 数组常量 |
| `EX_EndArrayConst` | 0x66 | 无 | 数组常量结束 |
| `EX_SetConst` | 0x3D | 元素表达式* N, `EX_EndSetConst` | Set 常量 |
| `EX_EndSetConst` | 0x3E | 无 | Set 常量结束 |
| `EX_MapConst` | 0x3F | (键表达式, 值表达式)* N, `EX_EndMapConst` | Map 常量 |
| `EX_EndMapConst` | 0x40 | 无 | Map 常量结束 |
| `EX_ArrayGetByRef` | 0x6B | `FArrayProperty*`, 索引表达式 | 按引用获取数组元素（返回元素地址，用于外部直接修改） |

### 8.9 类型转换指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_DynamicCast` | 0x2E | `UClass*`, 对象表达式 | 安全动态类型转换（Cast<T>），失败时返回 nullptr |
| `EX_MetaCast` | 0x13 | `UClass*`, 类对象 | 元类转换（ClassCast）：将 `AActor::StaticClass()` 转换为 `AMyActor::StaticClass()` |
| `EX_Cast` | 0x38 | `ECastToken`（1字节） | 基础类型转换：`ECastToken` 指示转换类型（如 float↔double） |
| `EX_ObjToInterfaceCast` | 0x52 | `UClass*`（接口类）, 对象表达式 | 对象→接口转换 |
| `EX_CrossInterfaceCast` | 0x54 | `UClass*`（目标接口）, 接口表达式 | 接口→接口转换 |
| `EX_InterfaceToObjCast` | 0x55 | `UClass*`（对象类）, 接口表达式 | 接口→对象转换 |

**ECastToken 枚举：**
| 值 | 名称 | 说明 |
|----|------|------|
| 0x00 | `CST_ObjectToInterface` | 对象→接口 |
| 0x01 | `CST_ObjectToBool` | 对象→布尔（非空检查） |
| 0x02 | `CST_InterfaceToBool` | 接口→布尔（非空检查） |
| 0x03 | `CST_DoubleToFloat` | double→float |
| 0x04 | `CST_FloatToDouble` | float→double |

### 8.10 委托指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_AddMulticastDelegate` | 0x5C | 委托, 要添加的委托 | `Delegate->AddUnique(EventDelegate)` |
| `EX_RemoveMulticastDelegate` | 0x62 | 委托, 要移除的委托 | `Delegate->Remove(EventDelegate)` |
| `EX_ClearMulticastDelegate` | 0x5D | 委托 | `Delegate->Clear()` |
| `EX_BindDelegate` | 0x61 | 委托, 对象, FName（函数名） | 将对象+函数名绑定到委托 |
| `EX_InstanceDelegate` | 0x4B | `UFunction*`（或委托签名） | const 引用到委托或普通函数对象，用于回调参数 |

### 8.11 调试与仪表指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_Breakpoint` | 0x50 | 无 | 断点：编辑器模式下触发调试器暂停，否则同 `EX_Nothing` |
| `EX_WireTracepoint` | 0x5A | 无 | 连线追踪点：编辑器模式下高亮当前执行的连线 |
| `EX_Tracepoint` | 0x5E | 无 | 通用追踪点 |
| `EX_InstrumentationEvent` | 0x6A | `EScriptInstrumentation::Type`, 数据 | 仪表化事件（性能分析用） |
| `EX_Nothing` | 0x0B | 无 | NOP（无操作） |
| `EX_NothingInt32` | 0x0C | `int32` | 带 int32 参数的 NOP（反汇编调试用标记） |

### 8.12 杂项指令

| 操作码 | 值 | 操作数 | 语义 |
|--------|---|--------|------|
| `EX_Assert` | 0x09 | bool 表达式 | 断言：如果 false 则触发断点或中止（DEBUG 模式） |
| `EX_Skip` | 0x18 | `CodeSkipSizeType` | 跳过后续 N 字节的代码 |
| `EX_EndParmValue` | 0x15 | 无 | 可选参数默认值结束标记 |
| `EX_AutoRtfmTransact` | 0x70 | 无 | AutoRTFM：后续代码在事务中运行 |
| `EX_AutoRtfmStopTransact` | 0x71 | `EAutoRtfmStopTransactMode` | AutoRTFM：退出事务（优雅退出/中止退出/中止退出并中止父事务） |
| `EX_AutoRtfmAbortIfNot` | 0x72 | bool 条件 | AutoRTFM：条件 false 时中止事务 |

---

## 9. 执行细节

### 9.1 完整函数调用路径

#### 从蓝图调用蓝图函数

```
用户代码调用: Object->ProcessEvent(FindFunction("MyBPFunc"), &Params)
    │
    ▼
UObject::ProcessEvent(UFunction* Function, void* Parms)
    │  分配帧内存（Locals），准备参数数据
    │
    ▼
UFunction::Invoke(UObject* Obj, FFrame& Stack, void* Result)
    │  if (Function->HasAnyFunctionFlags(FUNC_Native))
    │      Function->Func(Obj, Stack, Result)   // C++ Thunk 路径
    │  else
    │      ProcessScriptFunction(Obj, Fn, Stack, Result, ProcessLocalScriptFunction)
    │
    ▼
ProcessScriptFunction<Processor>(...)
    │  创建子 FFrame NewStack(Obj, Function, Locals, &Stack)
    │  遍历参数：对 ChildProperties 链中的每个参数
    │      Stack.Step(Object, ParamAddr)  // 求值实际参数
    │  处理 OutParms 链表
    │  Processor(Object, NewStack, Result)  // → ProcessLocalScriptFunction
    │  处理返回值和 Out 参数写回
    │  析构局部变量
    │
    ▼
ProcessLocalScriptFunction(Context, Stack, Result)
    │  while (*Stack.Code != EX_Return && !Stack.bAbortingExecution)
    │      Stack.Step(Stack.Object, Buffer)
    │      //     ↓
    │      // int32 B = *Code++;
    │      // (GNatives[B])(Context, *this, RESULT_PARAM);
    │      //     ↓
    │      // 每条操作码处理函数执行一个完整的字节码操作
    │  // 处理返回值
    │  Stack.Step(Stack.Object, RESULT_PARAM)
    │
    ▼ [返回到 ProcessScriptFunction]
```

#### 从蓝图调用 C++ Native 函数

```
蓝图字节码: EX_FinalFunction(UFunction*)
    │
    ▼
execFinalFunction / execLocalFinalFunction
    │  从字节码读取 UFunction* 指针
    │
    ▼
ProcessLocalFunction(Context, Fn, Stack, Result)
    │  检测 FUNC_Native 标志
    │
    ▼
Fn->Invoke(Context, Stack, Result)
    │  调用 Fn->Func (Thunk function)
    │
    ▼
exec* 函数 (如 execK2_SomeFunction):
    P_GET_INT32(Param1);          // 从 Stack 的字节码流提取参数
    P_GET_FLOAT(Param2);
    P_FINISH;                     // 检查是否有错误
    P_NATIVE_BEGIN;
    ActualFunctionImpl(Param1, Param2);  // 实际的 C++ 函数
    P_NATIVE_END;
```

### 9.2 局部变量存储模型

每个 `UFunction` 调用时，局部变量的内存通过 `FProperty` 的 `Offset` 属性确定：

```
帧内存布局（由 UFunction::ChildProperties 中的属性定义）：
  Locals + 0:       int32 MyIntVariable        (Offset=0)
  Locals + 4:       float MyFloatVariable      (Offset=4)
  Locals + 8:       FVector MyVectorVariable   (Offset=8)
  Locals + 20:      UObject* MyObjRef          (Offset=20)
  ...

内存分配：
  - 小帧：栈上分配（alloca），通过 UE_VSTACK_ALLOC 宏
  - 大帧：通过 FVirtualStackAllocator（可配置，默认为 alloca）
  - UberGraph 帧：持久分配在对象上（通过 FPointerToUberGraphFrame）
```

### 9.3 属性求值模式

VM 中的属性求值不是"将值读入虚拟机寄存器"，而是**将内存地址记录到 FFrame 中**：

```
EX_InstanceVariable(FProperty* Prop):
    1. MostRecentProperty = Prop
    2. 根据 VarType:
       - Instanced:  MostRecentPropertyAddress = Object + Prop->GetOffset_ForInternal()
       - Default:    MostRecentPropertyAddress = Object->GetClass()->GetDefaultObject() + Prop->Offset
       - Local:      MostRecentPropertyAddress = Locals + Prop->Offset
    3. 后续指令通过 MostRecentPropertyAddress 和 Prop 读取或写入值
```

实际上，`EX_InstanceVariable` 不"读取"任何东西——它只设置地址。**下一个指令**（如 `EX_Let`、`EX_JumpIfNot`）才消费这个地址。

### 9.4 赋值语义 (EX_Let)

```
EX_Let(FProperty* DestProp):
    1. 从 Code 读取 DestProp
    2. 根据 DestProp 的 VarType 确定目标地址
    3. Stack.Step() 求值源表达式 → MostRecentPropertyAddress
    4. 调用 DestProp->CopyCompleteValueFromVariable(DestAddr, SrcAddr)
    // 这是 FProperty 的虚函数：不同类型的属性有不同的复制实现
    // 例如 FIntProperty::CopyCompleteValueFromVariable 就是 memcpy(4 bytes)
    // FObjectProperty::CopyCompleteValueFromVariable 涉及引用计数管理
```

`EX_Let` 的妙处在于：**赋值逻辑完全委托给 FProperty**。代码只需指定 DestProp + 求值 Src，实际的复制语义（大小、引用计数、位字段等）全部由 FProperty 的虚函数表自动处理。

### 9.5 函数调用细节

以 `EX_FinalFunction` 为例——从字节码调用编译期已知的非虚函数：

```
execFinalFunction(Context, Stack, Result):
    1. UFunction* Function = Stack.ReadObject<UFunction>()
       // ReadObject 从 Code 读取 UFunction*，自动前进 Code 指针

    2. 分配局部变量帧：
       uint8* Frame = (uint8*)alloca(Function->PropertiesSize)

    3. 初始化帧：
       // 从 CDO 复制属性值（对于有默认值的变量）
       for (FProperty* Prop = Function->ChildProperties; Prop; Prop = Prop->Next)
           Prop->InitializeValue_InContainer(Frame)

    4. 评估所有参数：
       for (FProperty* Prop = Function->ChildProperties; Prop; Prop = Prop->Next)
           Stack.Step(Stack.Object, Frame + Prop->Offset)
           // 每调用一次 Step，执行一个完整的表达式来求值参数

    5. 读取 EX_EndFunctionParms (0x16)

    6. 处理 OutParms
       for (FProperty* Prop; has OutParm flag; Prop = Prop->Next)
           // 建立 FOutParmRec 链表 — 把参数的地址传出去供被调函数使用

    7. ProcessLocalFunction(Context, Function, NewStack, Result)

    8. 处理输出参数写回 (OutParms)
       for (FOutParmRec* Out = OutParms; Out; Out = Out->Next)
           Out->Property->CopyCompleteValue(Out->PropAddr, ...)

    9. 销毁局部变量
       for (FProperty* Prop = ...; Prop; Prop = Prop->Next)
           Prop->DestroyValue_InContainer(Frame)
```

### 9.6 上下文切换 (EX_Context)

`EX_Context` 是实现 `obj->Method()` 调用语义的关键：

```
execContext(Context, Stack, Result):
    1. 读取 SkipOffset (CodeSkipSizeType)
    2. 保存当前 Context 到临时变量 (OldContext)
    3. Stack.Step() → 求值上下文表达式（产生新 Context）
    4. 如果新 Context 为 null：
         if (EX_Context)：FatalError("Attempted to call function on NULL object")
         if (EX_Context_FailSilent)：Code += SkipOffset（跳过整个函数调用）
    5. 如果新 Context 非 null：
         Stack.Step(NewContext, Result)  // 在新 Object 上下文中执行调用
    6. 恢复 OldContext
```

这种设计使得 `Actor.Component.Function()` 这种链式调用成为可能——每个 `.` 操作符编译为一个 `EX_Context`。

### 9.7 异常中止传播

蓝图执行中的 `bAbortingExecution` 标志用于异常处理：

```
触发方式：
  - FBlueprintCoreDelegates::ThrowScriptException() 被调用
  - 用户通过调试器手动中止执行
  - AutoRTFM 事务中止

传播链：
  FFrame 析构时：
      if (PreviousTrackingFrame)
          PreviousTrackingFrame->bAbortingExecution |= bAbortingExecution;

效果：
  while (*Stack.Code != EX_Return && !Stack.bAbortingExecution)
      // 一旦 bAbortingExecution 为 true，循环退出
      // Code 跳到 EX_Return 之后
  // 返回 CLEARED 值（零值）
```

这实现了跨 C++↔蓝图边界的异常样式的展开（unwinding），而无需 C++ 异常。

---

## 10. 编译管理器与编译策略

### 10.1 FBlueprintCompilationManager

编译管理器是蓝图编译的中央调度器，负责：

- **排队编译**：积累多个待编译蓝图并批量处理
- **依赖排序**：按继承层次排序（父类先编译）
- **增量编译**：只重编译已变更和依赖它的蓝图
- **骨架编译**：快速生成类结构而不生成字节码
- **异步编译**：在编辑器空闲时后台编译
- **编译缓存**：维护编译状态避免重复编译

**编译触发类型（EKismetCompileType）：**
| 类型 | 说明 |
|------|------|
| `Full` | 完整编译：生成所有字节码和完整类布局 |
| `SkeletonOnly` | 骨架编译：仅生成类布局和函数签名，不需要完整字节码 |
| `BytecodeOnly` | 仅字节码：在已有类结构上重新生成字节码 |
| `Cpp` | C++ 类编译后的蓝图重新链接 |

### 10.2 编译选项 (FKismetCompilerOptions)

```cpp
struct FKismetCompilerOptions
{
    EKismetCompileType CompileType;    // 编译类型
    bool bSaveIntermediateResults;    // 是否保存中间结果（调试用）
    // ... 其他选项
};
```

### 10.3 骨架编译（Skeleton Compile）

骨架编译是编辑器快速类型解析的核心技术：

```
骨架编译 = CompileClassLayout（仅创建类结构 + 函数签名） + 跳过 CompileFunctions

产物：
  SkeletonGeneratedClass (UBlueprintGeneratedClass)
      ├── 所有函数签名 （UFunction 无 Script[]）
      ├── 所有属性       （FProperty 完整）
      └── 骨架 CDO       （用于编辑器值预览）

用途：
  - 编辑器实时类型检测（引脚类型匹配）
  - 代码补全和上下文提示
  - 蓝图间依赖解析
  - 预览默认值
```

### 10.4 Compile-on-Load

蓝图在加载时自动检查是否需要重编译：

1. `UBlueprint::PostLoad()` 调用
2. 检查 `bRecompileOnLoad` 标志或 `BlueprintSystemVersion` 不匹配
3. 如果需要重编译：
   - 旧生成的类被暂存
   - 启动完整编译
   - `FBlueprintCompileReinstancer` 用新类替换旧类的所有实例
   - CDO 值从旧 CDO 迁移到新 CDO

### 10.5 熟化（Cooking）

在发布构建中：
- 编辑器数据（`NewVariables`、`UbergraphPages`、`FunctionGraphs` 等）被剥离
- 仅保留 `UBlueprintGeneratedClass`（含 `Script[]` 字节码）和 CDO
- 字节码保留：VM 在运行时正常运行
- 组件数据预烘焙：`CookedComponentInstancingData` 存储组件模板的序列化数据，运行时直接反序列化

---

## 附录：执行流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                      蓝图函数调用生命周期                            │
│                                                                  │
│  UObject::ProcessEvent(Function, Params)                          │
│    │                                                             │
│    ├→ 分配局部变量帧（Locals = alloca(Function->PropertiesSize)）  │
│    │                                                             │
│    ├→ FFrame Stack(Object, Function, Locals, PreviousFrame)      │
│    │   Code → Function->Script.GetData()                         │
│    │                                                             │
│    ├→ ProcessScriptFunction()                                    │
│    │   │                                                         │
│    │   ├→ 解析参数：对每个 ChildProperties                         │
│    │   │   Stack.Step(Object, ParamAddr)                         │
│    │   │                                                         │
│    │   ├→ 建立 OutParmRec 链表（针对 Out 参数）                     │
│    │   │                                                         │
│    │   └→ ProcessLocalScriptFunction(Object, Stack, Result)      │
│    │       │                                                     │
│    │       ├→ while (*Code != EX_Return)                          │
│    │       │   │                                                 │
│    │       │   └→ Stack.Step(Object, Buffer)                     │
│    │       │       │                                             │
│    │       │       ├→ B = *Code++                                │
│    │       │       └→ (GNatives[B])(Context, Stack, Result)      │
│    │       │           │                                         │
│    │       │           ├→ EX_InstanceVariable:                   │
│    │       │           │   MostRecentProperty = ReadProperty()   │
│    │       │           │   MostRecentPropertyAddress =           │
│    │       │           │       Object + Property->Offset         │
│    │       │           │                                         │
│    │       │           ├→ EX_FinalFunction:                      │
│    │       │           │   递归 → ProcessLocalFunction           │
│    │       │           │   → UFunction::Invoke                   │
│    │       │           │   → [新栈帧上的 VM 主循环]               │
│    │       │           │                                         │
│    │       │           ├→ EX_Let:                                │
│    │       │           │   DestProp = ReadProperty()             │
│    │       │           │   Stack.Step() 求值源                   │
│    │       │           │   DestProp->CopyCompleteValueFromVar()  │
│    │       │           │                                         │
│    │       │           ├→ EX_JumpIfNot:                          │
│    │       │           │   求值条件                                 │
│    │       │           │   if (!条件) Code += ReadCodeSkipCount  │
│    │       │           │                                         │
│    │       │           └→ EX_PushExecutionFlow:                  │
│    │       │               FlowStack.Push(Code + Offset)        │
│    │       │                                                     │
│    │       ├→ 处理返回值 (EX_Return)                                │
│    │       └→ 恢复递归深度计数器，根据需要抛出异常                    │
│    │                                                             │
│    ├→ ProcessOutParms：将 Out 参数值写回调用者的参数结构             │
│    ├→ 销毁局部变量（调用 DestroyValue_InContainer）                 │
│    └→ FFrame 析构（传播 bAbortingExecution）                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 参考资料

本文档基于以下 UE5 引擎源代码文件编写（分支 5.5）：

| 模块 | 关键文件 |
|------|---------|
| 编译器主类 | `Engine/Source/Editor/KismetCompiler/Public/KismetCompiler.h` |
| 编译器实现 | `Engine/Source/Editor/KismetCompiler/Private/KismetCompiler.cpp` |
| 中间表示 | `Engine/Source/Editor/KismetCompiler/Public/BPTerminal.h`, `BlueprintCompiledStatement.h` |
| 函数上下文 | `Engine/Source/Editor/KismetCompiler/Public/KismetCompiledFunctionContext.h` |
| 节点处理器 | `Engine/Source/Editor/KismetCompiler/Public/KismetCompilerMisc.h`, `Private/KismetCompilerMisc.cpp` |
| 字节码后端 | `Engine/Source/Editor/KismetCompiler/Private/KismetCompilerVMBackend.cpp`, `KismetCompilerBackend.h` |
| 字节码指令集 | `Engine/Source/Runtime/CoreUObject/Public/UObject/Script.h` |
| 栈帧定义 | `Engine/Source/Runtime/CoreUObject/Public/UObject/Stack.h` |
| VM 执行循环 | `Engine/Source/Runtime/CoreUObject/Private/UObject/ScriptCore.cpp` |
| 编译管理器 | `Engine/Source/Editor/Kismet/Public/BlueprintCompilationManager.h` |
| 蓝图生成类 | `Engine/Source/Runtime/Engine/Classes/Engine/BlueprintGeneratedClass.h` |
