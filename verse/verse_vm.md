# Verse VM

> 本文解析 UE6 中 **Verse 虚拟机（VerseVM，代码内简称 VVM）**：一个驻留在 `CoreUObject` 里、拥有**独立并发 GC 堆**与**字节码解释器**的 Verse 语言运行时，并把它与 UE 传统的 **Blueprint 虚拟机（BPVM）** 逐项对比。
>
> 代码位置：
> - Verse VM：`Engine/Source/Runtime/CoreUObject/Public/VerseVM/` 与 `Private/VerseVM/`
> - Blueprint VM：`Engine/Source/Runtime/CoreUObject/Public/UObject/Script.h`（字节码 `EExprToken`）、`Private/UObject/ScriptCore.cpp`（解释器）、`Public/UObject/Stack.h`（`FFrame` 栈帧）
> - Verse 字节码定义：`Engine/Source/Programs/UnrealBuildTool/System/VerseVMBytecodeGenerator.cs`（构建期生成 op 头文件）

---

## 1. Verse VM 是什么

`VerseVM/README.md` 对它的定位是一句话：

> *"The Verse language runtime resident in CoreUObject: a concurrently garbage-collected heap and value model, a bytecode interpreter, and the UObject-reflection surface for Verse types."*

即三个核心职责：

1. **并发 GC 堆 + 值模型**——Verse 自己的一套堆（FrankenGC，与 UE GC 融合），值用 64 位 NaN-boxed 的 `VValue` 表示，对象是 `VCell`（不是 `UObject`）。
2. **字节码解释器**——执行由编译器前端生成的 Verse 字节码（`VProcedure`）。
3. **面向 UObject 的反射表面**——把 Verse 类型暴露给 UObject/Blueprint 反射系统（`UVerseClass` / `UVerseFunction` / `VBPVM*`）。

关键状态（`README.md` front-matter）：

| 项 | 值 |
|----|----|
| `status` | `experimental` |
| 开关 | `WITH_VERSE_VM`（**默认关闭**，库存 target 走 BPVM 路径） |
| 目标 | 启用后**替换**现有 BPVM 执行路径 |
| GC 前提 | 必须开启 `gc.EnableFrankenGC`（VM 入口会 **fatal check**） |
| FP 语义 | `SetupVerse()` 强制 `FPSemantics = Precise`（精确浮点，跨平台确定） |
| 事务 | 字节码整体运行在 AutoRTFM 事务内（open/uninstrumented） |

**VM 只是“执行后端”**，与它配套的兄弟组件彼此解耦：

- **编译器前端** = `VerseCompiler`（uLang），**刻意不依赖** VerseVM 头文件（见 `README` 依赖节）。
- **字节码生成器（codegen）** = `Engine/Plugins/VerseVM` 插件，负责 uLang → 字节码（驱动 `FOpEmitter`），也是**唯一受认可的编译器↔VM 桥梁**。
- **调试器** = 同属 `Engine/Plugins/VerseVM` 的 socket 调试器。
- **C++ 侧高层值 API** = `Engine/Plugins/Solaris/Source/VerseNative`。

数据流：

```
Verse 源码 ──uLang 前端(VerseCompiler)──► 语义程序 ──VerseVM 插件(codegen, FOpEmitter)──► VProcedure(字节码)
                                                                                              │
                                                                                              ▼
                                                 Verse VM 解释器(FInterpreter) ◄── Verse GC 堆(VCell/VValue)
```

---

## 2. 目录结构

`Public/VerseVM/`（公开头，VM 内部实现也放这里，因为 Verse 插件要 include）与 `Private/VerseVM/`（实现）加在一起约 200+ 文件，按职责可粗分：

```
Public/VerseVM/
├── VVMBytecode*            # 字节码：VVMBytecode.h、VVMBytecodeOps.gen.h、VVMBytecodeEmitter.h、
│                           #           VVMBytecodeDispatcher.h、VVMBytecodePrinting.h、VVMBytecodeAnalysis.h
├── VVMInterpreter.cpp      # 解释器（FInterpreter，位于 Private/VerseVM/）
├── VVMValue.h / VVMCell.h  # 值模型 / 对象模型
├── VVMHeap* / VVMMarkStack* # 并发 GC：堆、空间、标记栈、写屏障
│   VVMWriteBarrier.h / VVMWeakKeyMap* / VVMWeakCellMap*
├── VVMContext.h            # GC 能力对象（FContext/FAccessContext/…）
├── VVMFrame.h / VVMFunction.h / VVMProcedure.h  # 栈帧 / 函数 / 过程
├── VVMTask* / VVMCoroutine.h / VVMSuspension.h # 任务 / 协程 / 挂起
├── VVMClass.h / VVMVerseClass.h / VVMVerseFunction.h / VVMVerseStruct.h  # 类与 UObject 反射
├── VVMUECodeGen.h / VVMUHTNativeLoader.h        # 原生 UObject 绑定
├── VBPVMDynamicProperty.h / VBPVMFunctionProperty.h / VBPVMFunctionRef.h / VBPVMRuntimeType.h  # Verse↔Blueprint 属性桥
├── VVMDebugger.h / VVMProfilingLibrary.h / VVMSamplingProfiler.h         # 调试 / 采样
├── VVMPersistence.h / VVMStructuredArchiveVisitor.h / VVMJson.h          # 持久化 / JSON
└── Inline/                  # 各 header 的 Inline 展开体（遵循 Inline/ 模式）
```

---

## 3. 值模型：NaN-boxed 的 `VValue`

`VVMValue.h` 的 `VValue` 是整个 VM 的**基本值类型**：一个 64 位、用 **NaN-boxing** 编码的 payload，把整数、浮点、字符、Cell 指针、UObject 指针全部装进同一个 64 位字段（`EncodedBits`）。

编码空间（按高 16 位划分，注释原文）：

```
0x0000...   Cell / Placeholder / UObject / Root / char / transparent VRef（见下）
0x0001... ┐
...        ├  float（用 FloatOffset 偏移量编码的 double）
0xfffc... ┘
0xfffd...   未用
0xfffe...   未用
0xffff...   int32
```

当高 16 位为 `0000` 时，低 4 位再细分：

| 低 4 位 | 含义 |
|---------|------|
| `0b0000` | `VCell*` 指针（GC 堆对象） |
| `0b0001` | `VPlaceholder`（未解析占位，效应/惰性求值用） |
| `0b0010` | Root（含 split depth） |
| `0bX011` | `UObject*` 指针 |
| `0b0100` | `char8` |
| `0b0101` | `char32` |
| `0b0110` | transparent `VRef` |

要点：

- **指针假定 48 位地址空间**（编码注释明确）。
- 非 double 的值都是“非纯 NaN”，VM 只在 float 分支用 IEEE754 NaN 空间，并区分 **PureNaN**（可安全被 box 的 NaN）与 impure NaN（box 前需 purify）。
- 关键判定：`IsCell()` / `IsUObject()` / `IsInt32()` / `IsFloat()` / `IsChar()` / `IsPlaceholder()` / `IsRoot()`。
- **深比较**不走 `operator==`（那只是位相等）：`Equal` 做深相等（可传 `HandlePlaceholder` 处理占位），返回 `ECompares{Neq,Eq,Undecidable,RuntimeError}`。
- **冻结/熔化**：`Freeze`（可变 → 不可变，返回 `FOpResult` 可能 Block）、`Melt`（不可变 → 可变）是 Verse 值语义的关键操作。

这是 Verse VM 与 BPVM **最根本的差异点**之一：BPVM 没有统一的值类型，操作数直接是 UObject 属性内存 + `FProperty` 反射；Verse VM 一切皆 `VValue`。

---

## 4. 对象模型：非多态的 `VCell`

`VVMCell.h` 的 `VCell` 是**由 Verse GC 管理的对象**，既包括 VM 内部结构（`VFrame`/`VProcedure`/`VFunction`…），也包括用户可见的值（数组、map、struct、class 实例）。设计红线（`README` "Deviations"）：

- **不是 UObject**，是 plain C++，类型信息靠**手写的 `VCppClassInfo` vtable**（`DECLARE_BASE_VCPPCLASSINFO` / `DECLARE_DERIVED_VCPPCLASSINFO`）。
- **禁止虚函数**：分派走 `VCppClassInfo`；`static_assert(sizeof(VCell) <= 8)`——头 8 字节是**对象头**（`EmergentTypeOffset` + GC 标志字节 + 互斥字节 + misc 字节）。
- 每个 `VCell` 内嵌一个 `TIntrusiveMutex` 字节（`FCellUniqueLock`）。

对象头（前 8 字节）：

```cpp
uint32 EmergentTypeOffset;      // 到 emergent type 的偏移（用于 O(1) 获取类型/内联缓存）
std::atomic<uint8> GCData{0};   // GC 标志位（bit0 = weak-key 标记）
mutable std::atomic<uint8> Mutex{0}; // 内嵌锁
union { struct { uint8 Misc2; uint8 Misc3; }; uint16 Misc2And3; }; // DeeplyMutable / MeltedNativeStruct 等
```

核心机制：

- **Emergent Type**（`VVMEmergentType*`）——运行期动态形成的类型描述（类、struct、native struct…），`GetEmergentType()` 从 offset 解析。
- **引用遍历**：子类实现 `VisitReferencesImpl`（或 `DEFINE_TRIVIAL_VISIT_REFERENCES`）。该函数会在**收集器线程并发、并行**调用，但保证每对象每轮只调一次（见注释）。
- **Census / 析构钩子**（`ConductCensusImpl` / `~VCell` 相关的 destructor hook）：可能持有分配器/互斥器锁时运行，**不得分配、不得加会跨分配持有的锁**。
- **弱映射**：为并发弱 map 正确性，每个 cell 都有一 bit 可做 weak-key（`AddWeakMapping`/`RemoveWeakMapping`）。

`VHeapValue`（`VVMValue.h` 同文件底部）表示“Verse 面向的值”，`VCell` 表示“VM 内部结构”，二者同源。

---

## 5. 内存管理：并发 mark-sweep（FrankenGC）

`VVMHeap.h` + `VVMMarkStack*` + `VVMWriteBarrier.h` 构成 GC 子系统。

- **并发**：标记由收集器线程并发执行，mutator 通过**写屏障（write barrier）**与之协同。写堆引用必须经 `TWriteBarrier`，**禁止裸指针成员**（`VVMWriteBarrier.h` 明示这条“法律”及其例外）。
- **FrankenGC**：Verse 堆与 UE GC 融合的收集模式，`gc.EnableFrankenGC` 未开启则 VM 入口 **fatal check**。跨堆引用双向合法（Verse `VValue` 可指 `UObject`，`UObject` 的 `FVRestValueProperty` 可指 `VCell`）。
- **空间（spaces）**：`CensusSpace` / `DestructorSpace` / `DestructorAndCensusSpace`——决定对象是否需要 census 钩子或析构钩子。
- **握手（handshake）**：mutator 周期性 `CheckForHandshake` 与收集器同步（栈扫描、停止世界 `VVMStoppedWorld`）。支持**手动栈扫描**（`EnableManualStackScanning` / `ClearManualStackScanRequest`，UE 主线程在 tick 末尾调用）。
- **弱引用**：`VVMWeakKeyMap` / `VVMWeakCellMap` / `VVMWeakBarrier`，配合 cell 上的 weak-key bit 与 weak mapping。

### GC 能力对象（`VVMContext.h`）

线程永远持有一个 context 引用，但**不直接指 context**，而是用**能力（capability）类型**表达 GC 契约——函数拿什么 context，就说明它能对堆做什么：

| 能力 | 能做什么 |
|------|---------|
| `FContext` | 只知道 context，什么都不能做 |
| `FIOContext` | 不能访问堆/分配，但可重新获取访问权 |
| `FAccessContext` | 能访问堆，不能分配/放弃访问 |
| `FRunningContext` | 能访问堆、放弃访问、检查握手、切到分配 |
| `FAllocationContext` | 能访问堆、能分配，不能放弃访问/查握手 |
| `FRunningContextPromise` | UE 侧的逃生舱（线程默认 running，如 `VInt::operator+`），内部走 TLS 查找，**可能昂贵** |

这是 Verse VM 相比 BPVM 的又一大差异：**GC 契约编码进了类型系统**。

---

## 6. 字节码：C# 生成、类型化 op 结构

### 6.1 单源真相：`VerseVMBytecodeGenerator.cs`

与 BPVM 在 `Script.h` 里手写 `EExprToken` 不同，Verse VM 的 op 定义在 **C#** 里（`UnrealBuildTool/System/VerseVMBytecodeGenerator.cs`），**构建期**生成 `VVMBytecodeOps.gen.h`，再经 `VERSE_ENUM_OPS` 宏展开成：

- `enum class EOpcode : uint16`（`VVMBytecode.h`）
- 每个 op 对应的**类型化结构体** `FOp##Name`
- `ToString`、以及各解释器/发射器/分派器要 switch 的 op 列表

op 声明的 DSL 形如（`DefineOps()`）：

```csharp
Inst("Call")
    .Arg("Dest", Role.UnifyDef)
    .Arg("Callee", Role.Use)
    .Arg("Arguments", Role.Use, Arity.Variadic)
    .Arg("NamedArguments", Role.Immediate, Arity.Variadic, "VUniqueString")
    .Arg("NamedArgumentValues", Role.Use, Arity.Variadic)
    .Const("bCalleeYields", "bool", "true")
    .CapturesEffectTokens().CapturesBatchedRefs().Suspends();
```

每个 op 除操作数外还声明了**效应元数据**：`Suspends()`（可能挂起）、`CapturesEffectTokens()` / `CapturesReturnEffectToken()`（捕获效应 token）、`CapturesBatchedRefs()`、`Yields()`（可让出）——这是效应系统在字节码层的直接体现。

### 6.2 操作数角色（`EOperandRole`）

```cpp
enum class EOperandRole : uint8 { Use, Immediate, ClobberDef, UnifyDef };
```

- `Use` / `Immediate`：输入（立即数不参与 def）
- `ClobberDef`：**覆写式定义**（旧值被覆盖）
- `UnifyDef`：**统一式定义**（定义与既有占位值合并，是 Verse 惰性/占位语义的关键）

### 6.3 字节码结构（`VVMBytecode.h`）

- `FOp` 基类：`struct alignas(OpAlignment) FOp { EOpcode Opcode; }`，`OpAlignment = alignof(void*)`（8 字节对齐，避免收集器撕裂）。
- `FRegisterIndex`：约定寄存器——`SELF=0`、`SCOPE=1`（`(super:)` 等捕获）、`PARAMETER_START=2`。
- `FValueOperand`：**标记联合**，同一个 uint32 既表示寄存器（`Index < UNINITIALIZED`）又表示常量（`~Constant.Index`）。
- `FConstantIndex`、`FLabelOffset`（相对跳转）、`FUnwindEdge`（调用范围 → 展开目标）、`FNamedParam`（参数名→寄存器）、`FOpLocation`（op 偏移→源码位置）、`FFailureContextId`。
- `ForEachReg` / `ForEachJump`：遍历 op 用到的寄存器/跳转（供分析器、GC 栈扫描用）。

### 6.4 `VProcedure` 布局（`VVMProcedure.h`）

一个 `VProcedure` 就是**一整块堆 cell**，内含字节码 + 所有配套数组，内存连续紧凑：

```
VProcedure 头（FilePath/Name/各类 Num* 计数）
FNamedParam        NamedParam[NumNamedParameters]
TWriteBarrier<VValue> Constant[NumConstants]     # 常量池
FOp                Ops[NumOpBytes]               # 字节码流（8 字节对齐）
FValueOperand      Operand[NumOperands]          # 操作数表
FLabelOffset       Label[NumLabels]              # 跳转标签
FUnwindEdge        UnwindEdge[NumUnwindEdges]    # 展开边
FOpLocation        OpLocation[NumOpLocations]    # 源码位置映射
FRegisterName      RegisterName[NumRegisterNames] # 寄存器名
```

`CalcLayout`（`FStructBuilder`）算出各段偏移；字节码发射器 `FOpEmitter`（`VVMBytecodeEmitter.h`）逐 op 写入并回填标签。

---

## 7. 指令集：opcode 分类

从生成器的 `DefineOps()` 归纳（`IC` = inline cache 内联缓存；`FastFail` = 可失败快速路径，配 `LeniencyIndicator`）：

| 类别 | op |
|------|----|
| **算术**（Suspend） | `Add` `Sub` `Mul` `Div` `Mod`；一元 `Neg` `Query`；`MutableAdd`；`FastAppendToArray` `CanFastAppendToArrayFastFail` |
| **快速失败比较** | `LtFastFail` `LteFastFail` `GtFastFail` `GteFastFail` `EqFastFail` `NeqFastFail`（各带 `LeniencyIndicator` + `OnFailure` 跳转） |
| **快速失败访问** | `ArrayIndexFastFail` `TypeCastFastFail` `QueryFastFail` |
| **数据搬运** | `Move` `MoveTrailed` `MoveNonComparable` `Reset` `ResetNonTrailed` |
| **控制流** | `Jump` `JumpIfInitialized` `Switch`（变参）`Return` `ReturnTrailed` `ResumeUnwind` `Err` `Tracepoint`；失败上下文 `BeginFailureContext` `EndFailureContext` `EndFastFailureContext` |
| **任务/协程** | `SelfTask` `BeginTask` `CallTask` `EndTask` `BeginAwait` `AwaitSuccess` `EndAwait` `BeginBatch` `EndBatch` `Yield` `NewSemaphore` `WaitSemaphore` |
| **调用** | `Call` `CallWithSelf`（支持命名参数 `NamedArguments`） |
| **引用（冻结/熔化）** | `NewRef` `NewPersistentOrSessionWeakMapRef` `RefGet` `RefSet` `RefSetLive` `RefCallDomain` `Freeze` `FreezeIfAccessor` `Melt` |
| **容器** | `Length` `LengthWithEffects` `CallSet` `CallSetLive` `NewArray` `NewMutableArray` `NewMutableArrayWithCapacity` `ArrayAdd` `InPlaceMakeImmutable` `NewOption` `NewUnionVariant` `GetUnionVariantPayload` `GetUnionVariantTag` `NewMap` `MapKey` `MapValue` |
| **类/对象/模块** | `NewClass` `BindNativeClass` `ConstructNativeDefaultObject` `LoadImport` `JumpIfDefaultSubObject` `BeginModule` `EndModule` `EndModuleData` `NewObject` `NewObjectICClass` |
| **字段** | `LoadField` `LoadFieldICOffset/Constant/Function/NativeFunction/Accessor` `LoadFieldFromSuper` `CreateField` `CreateFieldICValueObjectConstant/ValueObjectField/NativeStruct/UObject` `UnifyField` `InitializeVar` `SetField` `SetFieldLive` `UnifyNativeObject` `UnwrapNativeConstructorWrapper` |
| **作用域/闭包** | `NewScope` `NewFunction` `LoadParentScope` `LoadCapture` |
| **性能剖析** | `BeginProfileBlock` `EndProfileBlock` |

对比 BPVM 的 `EExprToken`（约 110 个，偏“对象/属性内存”导向：`EX_Let`/`EX_Context`/`EX_VirtualFunction`/`EX_DynamicCast`/`EX_SetArray`…），Verse 的 op 集合明显**更高层、偏函数式语义**（容器构造、冻结/熔化、效应/挂起、失败上下文）。

---

## 8. 栈帧与寄存器

`VVMFrame.h` 的 `VFrame` 是**堆分配的 `VCell`**（不是 C++ 栈结构），一台**显式寄存器机**：

```cpp
struct VFrame : VCell {
    FOp* CallerPC;                    // 调用点 PC
    TWriteBarrier<VFrame> CallerFrame; // 调用者帧（链表式调用栈）
    VReturnSlot ReturnSlot;            // 返回值槽
    TWriteBarrier<VProcedure> Procedure;
    const uint32 NumRegisters;
    VRestValue Registers[];           // 寄存器数组（可变长尾数组）
};
```

- `Registers[]` 是 `VRestValue`（`VVMRestValue.h`，带 effect token 的 rest 值），按 `Procedure.NumRegisters` 分配。
- 寄存器约定（`FRegisterIndex`）：`SELF=0`、`SCOPE=1`、`PARAMETER_START=2`。
- `GlobalEmptyFrame` 是根空帧；帧可 `CloneWithoutCallerInfo`（宽松执行时用）。

调用栈不是操作系统栈，而是 `CallerFrame` 的**显式链表**，配合 `FUnwindEdge` 支持异常/失败展开。

---

## 9. 解释器与分派（`VVMInterpreter.cpp`）

解释器主体是 `FInterpreter`（约 6600 行的 .cpp），核心是一个 **成员函数指针表 + `[[clang::musttail]]` 尾调用** 的分派：

```cpp
using FDispatchFn = V_PRESERVE_NONE FOpResult::EKind (FInterpreter::*)(const bool);
using FDispatchArray = TStaticArray<FDispatchFn, OpcodeCount>;
inline static FDispatchArray TraceDispatchArray = {};     // 带 trace
inline static FDispatchArray NoTraceDispatchArray = {};    // 无 trace

V_FORCEINLINE FOpResult::EKind Dispatch(bool bHasOutermostPCBounds) {
    Context.CheckForHandshake(...);          // 先握手（在装载 opcode 之前）
    EOpcode Opcode = State.PC->Opcode;
    FDispatchFn Fn = bPrintTrace ? TraceDispatchArray[OpInt] : NoTraceDispatchArray[OpInt];
    [[clang::musttail]] return (this->*Fn)(bHasOutermostPCBounds);  // 尾调用进 handler
}
```

要点：

- **执行状态** `FExecutionState{ FOp* PC; VFrame* Frame; ... }`，PC 是 typed `FOp*` 指针（而非 BPVM 的 `uint8*` + 手动读取）。
- 每个 op 的 handler 是一个成员函数 `FOpNameImpl`，通过宏 `OP(OpName)` / `OP_WITH_IMPL` 展开，统一走 `OP_IMPL_HELPER`（返回 `FOpResult`）。
- `FOpResult::EKind`（`VVMOpResult.h`）区分 `Block`（挂起等待）/ `Error` / 正常，是效应挂起与失败的统一出口。
- **握手检查在每次装载 opcode 前**（`CheckForHandshake`），保证与并发 GC 协同。
- `musttail` 尾调用 + 静态函数指针表 = 现代 C++ 版的“computed goto”，避免巨型 switch 的分支预测/代码膨胀问题。

另有一个模板 `DispatchOp`（`VVMBytecodeDispatcher.h`）用于 visitor/发射器/打印等**非解释器**场景，走 `switch` 分派。

---

## 10. 效应、协程与并发

Verse 的效应系统在 VM 层的落地：

- **挂起/恢复**：`FOpResult::Block` + `VSuspension`（`VVMSuspension.h`）+ 挂起队列（`UnblockedSuspensionQueue`）。op 声明 `Suspends()`，即可能阻塞在未解析的 effect token / 占位上。
- **效应 token**：`CapturesEffectTokens` / `CapturesReturnEffectToken` 显式在线程间传递“效应令牌”，`EffectToken` 是 `VValue`，可能是占位（未 resolve 时挂起）。`BumpEffectEpoch` 记录效应代次。
- **任务**：`VTask`（`VVMTask.h`）+ `VTaskGroup`，op `BeginTask/CallTask/EndTask/Yield`；任务有 phase（`CancelRequested`/`CancelStarted`…）与 `Suspend`/`CancelChildren`。
- **协程等待**：`BeginAwait/AwaitSuccess/EndAwait`、`WaitSemaphore`、`NewSemaphore`。
- **失败上下文（fast-fail / leniency）**：Verse 的 `decides`（可失败）语义映射为**结构化失败处理**——`BeginFailureContext`/`EndFailureContext` 圈定范围，`*FastFail` op 失败时跳 `OnFailure`，`LeniencyIndicator` 追踪未决挂起，`ExecuteFastFailureThenOrElseLeniently` 决定是走失败分支还是宽松续行。

对比：BPVM 没有一等公民的效应/协程；它的“异步”靠 **latent action** + `FlowStack`（`EX_PushExecutionFlow`/`EX_PopExecutionFlow`），函数让出后由引擎稍后重新调用。

---

## 11. 事务与 AutoRTFM

- 顶层入口 `EnterVM` / `TransactThenEnterVM`（经 `Inline/VVMEnterVMInline.h` 汇入单入口）。
- VM 字节码整体跑在 **AutoRTFM 事务（open/uninstrumented）** 内；README 明确：*AutoRTFM 不会替你回滚堆写入*，需要回滚的 mutation 路径要**手动**经写屏障钩子 + `FTransaction` 写日志实现。
- BPVM 的事务是“事后补”的几个 op（`EX_AutoRtfmTransact/StopTransact/AbortIfNot/Abort`），而 Verse VM 是**内生于执行模型**（`VVMTransaction.h`、`FTransaction` 写日志）。

---

## 12. UObject 集成与 VBPVM 反射桥

Verse 类型要对 UObject/Blueprint 反射可见，靠一组桥接（前缀 `VVMVerse*` 与 `VBPVM*`）：

- **`UVerseClass` / `UVerseFunction` / `UVerseStruct` / `UVerseEnum`**（`VVMVerseClass.h` 等）：UObject 侧的 Verse 类型反射。`UVerseClass` 提供 `GetFunctionMangledName`（Verse 名 ↔ UFunction 混淆名）。
- **`FVerseFunctionRef`**（`VBPVMFunctionRef.h`）：BPVM 眼中“Verse 函数值”的表示 = `{ 绑定对象 SelfObject, 函数名 FunctionName }`，`FindFunction()` 解析到 `UFunction`（可现场改写混淆名）。`VerseNative` 的 `FVerseFunction` 派生自它。
- **`FVerseFunctionProperty`**（`VBPVMFunctionProperty.h`）：声明在 `FVerseFunctionRef` 上的 `FProperty`。
- **`VBPVMDynamicProperty`**：把 `VValue`（任意 Verse 值）作为 UObject **动态属性**暴露。
- **`VBPVMRuntimeType`**：Verse 运行期类型（`VType`）的反射包装。
- **原生绑定**：`VVMUECodeGen.h`（UHT 生成 native loader）+ `VVMUHTNativeLoader`、`VVMNativeFunction/Procedure/Struct/Ref`，以及 `ConstructNativeDefaultObject` / `BindNativeClass` / `LoadImport` 等 op。

> 术语辨析：`VBPVM` = **Verse ↔ Blueprint Property/VM** 桥（把 Verse 值映射到 UObject 属性系统），而非独立的第二套 VM。Blueprint VM 本体仍是 `Script.h`/`ScriptCore.cpp` 那套；`WITH_VERSE_VM` 开启后 Verse VM **接管**其执行路径。`Stack.h` 里另有 `WITH_VERSE_BPVM` 与 `ResolveVerseCaller`（Verse 调用方回溯，供混合调用栈报告）。

---

## 13. 持久化与调试

- **持久化**：`VCell::Serialize` / `SerializeLayout` / `SerializeImpl` + `VVMStructuredArchiveVisitor`（`SerializeIdentity` 决定是否保留身份去重）；`VVMJson.h` + `ToJSON`/`FromJSON`（Analytics/Persistence/Persona 三种格式）。`VProcedure::SerializeLayout` 反序列化时按 layout 重建。
- **调试**：`VVMDebugger`（socket 调试器，实现在 VerseVM 插件）、`Tracepoint` op（字节码里埋调试标记）、`FOpLocation` 的 op→源码映射、`GetLocation`。
- **剖析**：`VVMSamplingProfiler` + `BeginProfileBlock/EndProfileBlock` op。

---

## 14. 与 Blueprint VM 对比

### 14.1 总览表

| 维度 | **Blueprint VM（BPVM）** | **Verse VM（VVM）** |
|------|--------------------------|---------------------|
| 位置 | `CoreUObject/Public/UObject/Script.h` + `Private/UObject/ScriptCore.cpp` | `CoreUObject/Public|Private/VerseVM/` |
| 执行语言 | UObject Blueprint（Kismet）字节码 | Verse 字节码 |
| 字节码存储 | `UFunction::Script`（`TArray<uint8>` 扁平流） | `VProcedure`（堆 cell，含 ops/operands/labels/constants 分离段） |
| 指令编码 | 1 字节 opcode + **内联操作数**（变长，靠 `CodeSkipSize`/`VariableSize` 前进） | 每个 op 是**类型化结构体** `FOp##Name`，操作数在独立 `Operand[]` 表 |
| 指令定义 | C++ 手写 `EExprToken`（~110 个） | C# 生成（`VerseVMBytecodeGenerator.cs`）→ `EOpcode`（百余个） |
| 机器模型 | 表达式/栈混合：`Stack.Step` 递归求值，结果经 `RESULT_PARAM` 缓冲 | 显式**寄存器机**：`VFrame.Registers[]`，`Self/Scope/Param` 约定 |
| 值模型 | 原始 UObject 属性内存 + `FProperty` 反射，无统一值类型 | NaN-boxed 64 位 `VValue`（统一值类型） |
| 分派 | `GNatives[]`（`FNativeFuncPtr` 表，computed-goto 风格） | `DispatchArray[]`（成员函数指针表 + `[[clang::musttail]]` 尾调用） |
| 栈帧 | C++ `FFrame` 结构（`PreviousFrame` 链表 + `FVirtualStackAllocator`） | 堆分配 `VFrame`（`VCell`，`CallerFrame` 链表 + 寄存器 + 返回槽） |
| GC | UObject GC（mark-sweep，统一进引擎） | **并发** mark-sweep（FrankenGC），写屏障、emergent type、cell 非 UObject |
| GC 契约 | 隐式（线程单例 `FBlueprintContext`） | 显式（能力类型 `FAccessContext`/`FAllocationContext`…） |
| 并发 | 单线程（游戏线程），latent 靠 `FlowStack` | 任务/协程/效应，挂起-恢复，并发 GC |
| 效应/失败 | 无（异常/abort + latent 重入） | 一等公民：`Suspends/decides/yields`，FastFail op + 失败上下文 |
| 事务 | 事后补 `EX_AutoRtfm*` 若干 op | 内生于执行模型（open 事务 + 写屏障回滚 + `FTransaction`） |
| 反射/互操作 | `UFunction`/`FProperty` 原生反射 | `UVerseClass`/`UVerseFunction` + `VBPVM*` 属性桥（`FVerseFunctionRef`…） |
| 调试 | breakpoint / instrumentation / 采样 | socket 调试器 + `Tracepoint` op + 采样剖析器 |
| 持久化 | `UFunction` 序列化 | `VCell`/`VProcedure` StructuredArchive + JSON |

### 14.2 本质差异（三条主线）

1. **从“属性内存机器”到“统一值机器”**。BPVM 的字节码直接读写 UObject 属性内存，操作数由 `FProperty` 描述、按字节大小搬运；Verse VM 一切操作数都是 `VValue`，值是 NaN-boxed 的统一表示，语义（深比较、冻结/熔化、类型判断）在值层完成。

2. **从“单线程 + 事后事务”到“并发 GC + 内生效应”**。BPVM 是游戏线程单线程执行，GC 是 UE 传统 mark-sweep；Verse VM 拥有独立并发 GC 堆（FrankenGC + 写屏障 + 能力上下文），并把挂起/失败/让出建模为字节码一级的效应 token 与 `FOpResult::Block`。

3. **从“手写字节码表”到“单源生成 + 类型化字节码”**。BPVM 的 opcode 在 C++ 手写枚举，操作数是变长内联字节；Verse 的 opcode 在 C# 里一份定义，构建期生成 `EOpcode` + 类型化 `FOp##Name` + 各类分派器，操作数角色（`Use/ClobberDef/UnifyDef`）与效应元数据都从这一份描述推导。

### 14.3 相似之处

- **分派内核同构**：都是“按 opcode 查函数指针表 → 跳进 handler”，只不过 BPVM 是全局 `GNatives`、Verse 是解释器内 `DispatchArray` + musttail。
- **都驻留 CoreUObject**、都经 AutoRTFM 与事务体系集成（BPVM 是补丁式 op，Verse 是内生）。
- **都面向 UObject 反射**：BPVM 天生反射，Verse 经 `UVerse*`/`VBPVM*` 桥接打通。
- **都有 runaway/运行时错误守卫**：BPVM `FBlueprintContextTracker`（`DO_BLUEPRINT_GUARD`），Verse `VVMVerseHangDetection` + `RaiseVerseRuntimeError`。

---

## 15. 小结

```
                    ┌─────────────────────── Verse VM ───────────────────────┐
Verse 字节码(VProcedure, C# 生成 op)                                          │
        │                                                                     │
        ▼                                                                     │
FInterpreter：DispatchArray[opcode] ──musttail──► FOp##NameImpl               │
        │  (每 op 前 CheckForHandshake)                                       │
        ├─ VFrame(寄存器机) ── VValue(NaN-boxed) ── VCell(非多态, 写屏障)      │
        ├─ 效应 token / 挂起(Block) / 任务(VTask) / 失败上下文(FastFail)       │
        ├─ AutoRTFM open 事务 + FTransaction 写日志                           │
        └─ UObject 桥：UVerseClass / FVerseFunctionRef / VBPVM* 属性          │
                    └───────────────────────────────────────────────────────┘
                                     ▲ 取代（WITH_VERSE_VM 开启后）
Blueprint VM：GNatives[EExprToken] ─► exec* (ScriptCore.cpp) ── FFrame(属性内存机)
```

**一句话**：Blueprint VM 是一台**贴着 UObject 反射的、单线程属性内存机**；Verse VM 则是一台**带并发 GC、统一 NaN-boxed 值模型、一等公民效应/协程/事务的显式寄存器机**——字节码从 C# 单源生成、以类型化 op 结构呈现，GC 契约编码进上下文能力类型，执行路径在 `WITH_VERSE_VM` 开启后接管原 BPVM。
