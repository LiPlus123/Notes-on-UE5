# Verse 反射

> 本文解析 UE6 中 **Verse 的反射与原生互操作机制**：Verse 类型如何映射进 UObject 反射系统、Verse 与 C++ 如何在两个方向上互调。
>
> 代码位置：
> - 反射核心类：`Engine/Source/Runtime/CoreUObject/Public/VerseVM/`（`VVMVerseClass.h`、`VVMVerseFunction.h`、`VVMVerseStruct.h`、`VVMVerseEnum.h`、`VVMVerseClassFlags.h`、`VVMVerseNativeTypeDesc.h`）
> - 名称修饰：`CoreUObject/Public/VerseVM/VVMNames.h`
> - 原生绑定运行时：`VVMNativeProcedure.h`、`VVMNativeFunction.h`、`VVMNativeConverter.h`、`VVMNativeType.h`、`VVMNativeConstructorWrapper.h`
> - 高层 C++ 互操作 API：`Engine/Plugins/Solaris/Source/VerseNative/`（`VerseInteropMacros.h`、`VerseInteropTypes.h`、`VerseInteropStatics.h`）
> - 原生接口代码生成（VNI）：`Engine/Source/Programs/Solaris/VerseNativeInterfaceGen/`
>
> 与 [verse_vm.md](verse_vm.md)（VM 执行后端）互补：本文聚焦「类型/函数如何反射、如何绑定、如何互调」，不重复 VM 内部的字节码/GC/协程细节。

---

## 1. 总体图景：反射不是一张表，而是三套协同机制

Verse 没有 UE 传统意义上的「一份反射头文件（`.generated.h`）描述所有类型」这种单一实体。它的「反射」实际由**三条互相咬合的机制**构成：

1. **UObject 反射表面**——`UVerseClass` / `UVerseFunction` / `UVerseStruct` / `UVerseEnum` 这组 UObject 类型。它们让「Verse 定义的类型」变成 `UClass`/`UFunction`/`UScriptStruct`/`UEnum`，从而能被 UE 的属性系统、序列化、编辑器、以及 Blueprint 看见并操作。这是「Verse 类型 → UE 反射系统」的方向。

2. **原生值/类型反射（VM 侧）**——`VValue`（NaN-boxed 值）、`VClass`/`VShape`/`VEnumeration`/`VFunction` 这套 VM 内类型描述（emergent type）。它让「C++ 的 UObject 类型」在 Verse 堆里有一个对应的运行期类型表示。这是「UE 类型 → Verse 类型系统」的方向（详见 [verse_vm.md](verse_vm.md) §3–4、§12）。

3. **原生绑定与名称修饰（互操作胶水）**——一套把上面的类型/函数「接起来」的机制：名称修饰（mangling）、thunk 指针、值转换器（`FNativeConverter`）、以及生成这些胶水的代码生成器（VNI / UHT）。这是本文的重点。

一句话概括数据流：

```
            ┌─── UObject 反射 ───────────────────────────┐
Verse 类型 ──► UVerseClass / UVerseFunction / UVerseStruct / UVerseEnum
            └── 可序列化 / 可编辑 / 对 Blueprint 可见 ──┘

            ┌─── 原生绑定 ──────────────────────────────┐
Verse 源码 (<native>) ──VNI/UHT──► C++ thunk (V_NATIVE_BEGIN/END)
C++ 代码 ──FVerseFunction/TVerseFunction──► Verse 函数 (Invoke)
            └── 经 FNativeConverter 做 VValue ↔ C++ 值转换 ──┘
```

---

## 2. UObject 反射表面：`UVerse*` 四兄弟

这四个类是「Verse 类型 → UObject」的桥。它们都**继承自 UE 的反射基类**，因此天生能挂 `FProperty`、能被 `UClass` 的反射设施遍历。

| Verse 类型 | UObject 类 | 继承自 | 关键新增成员 |
|-----------|-----------|--------|-------------|
| Verse `class`/`module`/`interface` | `UVerseClass` | `UClass` | `Class`(VClass)、`Shape`(VShape)、`TaskClasses`、`FunctionMangledNames`、`VerseCallableThunks` |
| Verse 函数 | `UVerseFunction` | `UFunction` | `Callee`(VFunction)、`VerseFunctionFlags`、`AlternateName` |
| Verse 结构体 | `UVerseStruct` | `UScriptStruct` | `Class`(VClass)、`Shape`(VShape)、`Guid`、`FactoryFunction` |
| Verse 枚举 | `UVerseEnum` | `UEnum` | `Enumeration`(VEnumeration)、`QualifiedName` |

三者都携带一个 `const FVniTypeDesc* NativeTypeDesc` 指针（`VVMVerseNativeTypeDesc.h`），用于把「UE 类型」与「它在 VNI 里的 Verse 包/模块/作用域」对应起来（见 §4.2）。

### 2.1 `UVerseClass`

`VVMVerseClass.h` 中，`UVerseClass` 继承 `UClass`，是 Verse 类的 UObject 反射化身。除 `UClass` 自身的字段外，新增一组 Verse 特有的状态：

- **`uint32 SolClassFlags`**：`EVerseClassFlags` 位标志（见 §2.5），核心是 `VCLASS_UHTNative` / `VCLASS_NativeBound` / `VCLASS_Module` 等。
- **`Class` / `Shape`**（`WITH_VERSE_VM`）：VM 侧的 `VClass*` 与 `VShape*`——运行期 Verse 类型与对象布局。`Shape` 在 `Link` 前是占位值。
- **`TaskClasses`**：本类所有协程对应的任务类（每个协程一个 `UVerseClass`）。
- **`InitInstanceFunction`**：Verse 构造器的 UFunction 表示（`InitCDOFunctionName`）。
- **`PersistentVars` / `SessionVars`**：可持久化 / 会话级变量，各自指向一个 `FMapProperty`（存变量名 → 值的 map）。
- **`VarAccessors`**：`TMap<FName, FVerseClassVarAccessors>`，为可变量生成的 getter/setter `UFunction`。
- **`DisplayNameToUENameFunctionMap` / `FunctionMangledNames`**：显示名 ↔ UE 名、以及新旧混淆名映射（见 §3.4）。
- **`DirectInterfaces`**：本类直接实现的 `UVerseClass` 接口列表。
- **`VerseCallableThunks`**（VVM 路径）：一张 `{ NameUTF8, FThunkFn }` 表，记录「可从 Verse 调用的原生函数」的 thunk 指针。
- **`NativeTypeDesc`**：`FVniTypeDesc` 指针，VNI 类型描述。

几个关键能力：

- **`ForEachClassInHierarchy`**：遍历「本类 + 所有父类 + 所有接口（递归）」的 `UVerseClass` 层级——许多查询（`FindAccessors`、`FindPredictsVar`、`CanMemberFunctionBeCalledFromPredicts`）都基于它。
- **`GetShape` / `GetShapeForLoadField` / `LoadField` / `PeekField`**：把 `UObject` 上的字段读成 `VValue`。`LoadField` 返回 `FOpResult`（字段为 null 时报运行时错误），`PeekField` 永不报错、读不到就是未初始化 `VValue`。这是 C++ 侧访问 Verse 字段的统一入口。
- **`ForEachVerseFunction`**：遍历类上的 Verse 函数属性，回调 `FVerseFunctionDescriptor{ Function, DisplayName, UEName }`。

### 2.2 `UVerseFunction`

`VVMVerseFunction.h` 中，`UVerseFunction` 继承 `UFunction`，是「一个 Verse 函数」的 UObject 包装：

- **`Callee`**（`TWriteBarrier<VFunction>`）：VM 侧的 `VFunction`（被调对象）。这是 VVM 路径下 `UVerseFunction` 与 VM 函数体的连线。
- **`VerseFunctionFlags`**（`EVerseFunctionFlags`）：`UHTNative` / `UHTTaskUpdate` / `AccessibleFromEngineGameplay` / `CanAccessEpicInternal`。
- **`AlternateName` / `CoercedOriginalName`**：原生函数声明名与预期 Verse 名之间的「别名」（用于改名/coercion 兼容）。
- **`IsVerseGeneratedFunction`**：区分「由 Verse 编译器生成的函数」（非 UHT 原生）与「UHT 生成的原生函数」。

`UVerseFunction` 同时是 BPVM 与 VVM 两条路径的包装：BPVM 下它靠 `UFunction` 的 `exec*` + `FNativeFuncPtr` 执行；VVM 下它持有 `Callee`，由 `Bind()` 建立与 VM 的绑定。

### 2.3 `UVerseStruct`

`VVMVerseStruct.h`：Verse 结构体（值类型）的反射化身，继承 `UScriptStruct`。

- **`Guid`**：用于「旧结构体 ↔ 新结构体」版本匹配（`GetCustomGuid`）。
- **`FactoryFunction` / `OverrideFactoryFunction` / `InitFunction`**：Verse 结构体的构造/初始化函数（`InvokeDefaultFactoryFunction`）。
- **`ModuleClass`**：父模块类。
- **`AssembleReferenceTokenStream`**：为「含 `VValue`/`VCell` 引用」的结构体组装 UE GC 的引用 token 流——这是 Verse 结构体能被 UE GC 正确追踪的关键。

### 2.4 `UVerseEnum`

`VVMVerseEnum.h`：Verse 枚举的反射化身，继承 `UEnum`，持有 `Enumeration`（`VEnumeration`）。文件里还定义了两个「没有值」的占位枚举 `EVerseFalse` / `EVerseTrue`，对应 Verse 的 `false`（无可能值）与 `true`（一个可能值）类型——UHT 不支持空枚举，所以各自塞了一个 dummy case。

### 2.5 类标志 `EVerseClassFlags`

`VVMVerseClassFlags.h`（注释明确要求与 `UnrealEngineTypes.cs` 的 `EVerseClassFlags` 保持同步）：

| 标志 | 含义 |
|------|------|
| `VCLASS_NativeBound` | 已绑定到 C++ 原生实现 |
| `VCLASS_UniversallyAccessible` | 可从任意 Verse 路径访问，且在 public scope 包内 |
| `VCLASS_Concrete` | 无需显式设属性即可实例化 |
| `VCLASS_Module` | 代表一个 Verse 模块 |
| `VCLASS_UHTNative` | 由 UHT 创建 |
| `VCLASS_Tuple` | 代表元组 |
| `VCLASS_EpicInternal` / `VCLASS_EpicInternalConstructor` | `epic_internal` 类 / 构造器 |
| `VCLASS_HasInstancedSemantics` | 显式实例引用语义（后向兼容） |
| `VCLASS_FinalSuper` / `VCLASS_Castable` / `VCLASS_Persistable` | 对应 `<final_super>` `<castable>` `<persistable>` 属性 |
| `VCLASS_Parametric` | 函数局部定义（参数化）类 |
| `VCLASS_InternalCodegen` | 编译器为内部目的生成的类，非用户类 |
| `VCLASS_PersonaConstructible` | 可由 Persona 构造 |
| `VCLASS_InheritsFromBlueprint` | （传递）继承自 Blueprint 类 |
| `VCLASS_HasFieldAttribute` | 本类或父类使用 `@field` 属性 |
| `VCLASS_Err_Incomplete` / `VCLASS_Err_Inoperable` / `VCLASS_Err` | 类布局损坏 / 函数字节码错链 |

---

## 3. 名称修饰（Name Mangling）：反射的核心难题

Verse 是**大小写敏感**的语言，而 UE 的 `FName` 是**大小写不敏感**的；Verse 的名字还含 `/`、`:`、`()` 等 UE 名字里非法的字符。因此「把 Verse 名映射成 UE 名、并在两个方向都能还原」是反射机制的底座，集中在 `VVMNames.h`。

### 3.1 装饰名（Decorated Name）

一个 Verse 名字的完整形式是「限定名（qualified name）」：

```
(/Verse/Path/To/Scope:)Name(:int)
```

- 以 `(` 开头即「全路径」——`IsFullPath` 判断。
- `GetQualifierLength` / `GetQualifier` / `RemoveQualifier` 处理前导 `(...)` 限定符；`GetScopePath` 反解出定义作用域。
- `GetDecoratedName(Path, Module, Name)` 把作用域路径、模块、名字拼成装饰名。注释强调**函数名会被装饰两次**：一次用定义所在作用域、一次用基定义作用域（通常相同）。

装饰名的用途贯穿全链路：VM 里按装饰名查找定义（`VPackage::LookupDefinition`）、原生函数按装饰名绑定 thunk、`UVerseClass::GetFunctionMangledName` 按装饰名解析混淆名。

### 3.2 大小写修饰（Case Mangling）

`Private::MangleCasedName(Name, CrcName)`：把一个**大小写敏感**的 Verse 名转成**大小写不敏感**的 UE 名——做法是**在名字前加一段该名字的 CRC**（`CrcName` 用于接口字段的限定名）。`UnmangleCasedName` 反向解出。CRC 保证了「不同大小写、同名」的两个 Verse 名字在 UE 侧得到不同的 `FName`。

### 3.3 Verse ↔ UE 名字互转

- 属性：`VersePropToUEFName` / `UEPropToVerseName`（后者的结果**大小写敏感、绝不可转回 FName**，注释反复警告）。
- 函数：`VerseFuncToUEFName` / `UEFuncToVerseName`。
- 编码：`Private::EncodeName` / `DecodeName` 把 Verse 名里的非法字符编码成 UE 合法字符（目前仅用于函数名）。
- 生命周期：`AddVerseDeadPrefix` / `RemoveVerseDeadPrefix` / `MakeTypeDead` —— 被标记 `DEAD` 的类型改名并移动 outer（热重载/重编译时的旧类型处理）。
- 包路径：`GetVersePackageNameForVni` / `GetUPackagePathForVni` / `GetVersePackageNameForContent` 等，建立「Verse 包名 ↔ UE 包路径」的映射（区分 VNI / content / published / assets 四种来源）。

### 3.4 函数混淆名映射 `FunctionMangledNames`

`UVerseClass::FunctionMangledNames` 是一张 `TMap<FName, FName>`：**旧混淆名 → 新混淆名**（若旧名对应多个可能的新版本，值记 `NAME_None`）。`GetFunctionMangledName` / `FindFunctionMangledName` 沿「本类 → 父类 → 接口」层级查找。

> 设计动机（头文件注释）：代码生成器（codegen）在不同版本可能改用新的混淆规则。为兼容「已经按旧混淆名序列化/引用」的数据，把旧名映射到当前名，实现**改名而不破坏引用**。

---

## 4. 反射的生成：UHT 与 VNI 两条路径

「谁把 C++/Verse 定义生成成上面的 `UVerse*` 类型和胶水代码」有两条路径，由 `FVniPackageDesc::bEnableVNI` 决定（注释原文：*If true, non-imported native types will require VNI generated code. Otherwise UHT will be used.*）。

### 4.1 UHT 路径（`VCLASS_UHTNative`）

旧/BPVM 时代的路径：C++ 类型用 UHT 标记，UHT 直接生成 `UVerseClass`/`UVerseFunction`/`UVerseStruct`/`UVerseEnum`，标志 `VCLASS_UHTNative` / `EVerseFunctionFlags::UHTNative` / `EVerseEnumFlags::UHTNative`。

构造入口在 `VVMUECodeGen.cpp` 的 `Verse::CodeGen::Private`：

- `ConstructUVerseClassNoInit` → `UECodeGen_Private::ConstructUClassNoInitHelper<UVerseClass>`，设 `VCLASS_UHTNative`。
- `ConstructUVerseClass` / `ConstructUVerseStruct` / `ConstructUVerseEnum` / `ConstructUVerseFunction`：分别填充 `PackageRelativeVersePath`、`MangledPackageVersePath`、`QualifiedName`、`Guid`、`AlternateName`、`NativeTypeDesc`、`DirectInterfaces` 等。
- `RegisterVerseCallableThunks` → `UVerseClass::SetVerseCallableThunks`。

配套的 `VVMUHTNativeLoader.h`：加载 UHT 原生 Verse 类型时，用 `MergeLoadedProperties` 把磁盘上载入的属性信息合并进「已存在的属性」，支持**重编译后旧类型就地更新**（`ResetUHTNative` / `StripVerseGeneratedFunctions` 配合回滚）。

### 4.2 VNI 路径（`FVniTypeDesc` / `FVniPackageDesc`）

新/VVM 时代由 **Verse 源码驱动**：你在 Verse 里声明 `<native>` 函数/类，VNI 生成器（§6.2）产出 C++ 胶水与描述符。两个核心描述符（`VVMVerseNativeTypeDesc.h`）：

```cpp
struct FVniPackageDesc {
    const FVniPackageName Name;        // { MountPointName, CppModuleName }，对应 UBT 模块名
    const TCHAR* PublicName;           // 用户可见名
    const TCHAR* VersePath;            // 包根模块的 Verse 路径
    const EVerseScope::Type VerseScope;// 可见性作用域
    const TCHAR* const VerseDirectoryPath;
    const FVniPackageName* const Dependencies;  // 增量编译依赖
    const int32 NumDependencies;
    const bool bEnableVNI;             // true 用 VNI 生成，否则用 UHT
};

struct FVniTypeDesc {
    const TCHAR* const UEPackageName;
    const TCHAR* const UEName;
    const FVniPackageDesc* VersePackageDesc;
    const TCHAR* VersePackageName;
    const TCHAR* VerseModulePath;
    const TCHAR* VerseScopeName;
};
```

`FVniTypeDesc` 就是 `UVerseClass/UVerseStruct/UVerseEnum::NativeTypeDesc` 所指向的类型描述：它把一个「UE 类型」与它在 Verse 里的「包 / 模块 / 作用域名」绑定。

### 4.3 自动注册（Auto-Binding Registrar）

`VerseInteropMacros.h` 用**静态对象自注册**把上面的描述符接进运行时，无需手动初始化。关键类型在 `VerseInteropTypes.h`：

- **`FVniPackageAutoRegistrar`**：构造时 `IVerseNativeModule::RegisterVniPackage(this)`，把整个 C++ 模块注册为一个 Verse 包。
- **`FVniTypeAutoRegistrar`**：构造时 `IVerseNativeModule::RegisterVniType(this)`，持有一个 `BindingFuncPtr`（`bool(*)(VPackage*, FUtf8StringView, UStruct*)`，VVM 路径）——即该类型的绑定函数。
- **`TBindingAutoRegistrar<T, BindPtr>`** + **`FVniTypeRegistration`**：每个类型的绑定模板实例；`FVniTypeRegistration` 缓存 `BoundType`（绑定后的 `UField`）与 `BoundRuntime`，提供 `GetBoundClass/GetBoundStruct/GetBoundEnum`。

宏的骨架（`VerseInteropMacros.h`）：

```cpp
V_IMPLEMENT_CLASS(CppModuleName, VClassName, VClassNameString, UClassNameString)
  └─► V_IMPLEMENT_AUTOBINDING_REGISTRAR  →  静态 FVniTypeRegistration{ .TypeDesc = {...} }
                                              + FVniTypeAutoRegistrar(绑定函数)
V_DEFINE_CPP_MODULE_REGISTRAR(...)  └─► 静态 FVniPackageDesc + FVniPackageAutoRegistrar
```

`V_DECLARE_EXEC` / `V_DEFINE_EXEC` 系列则声明/定义每个原生函数的 thunk（§6.3）。`V_VALIDATE_CLASS` / `V_VALIDATE_STRUCT` 用 `static_assert` 在编译期校验「C++ 类名与 Verse 声明一致、继承关系正确、未非法覆写 `PostInitProperties`」等。

---

## 5. VBPVM 属性桥：Verse 值 ↔ UObject 属性

要把「任意 Verse 值 / Verse 函数」放进 UObject 的 `FProperty` 体系，需要一组桥接属性（详见 [verse_vm.md](verse_vm.md) §12，这里只点出反射相关要点）：

- **`FVerseFunctionRef`**（`VBPVMFunctionRef.h`）——见 §7.1。
- **`FVerseFunctionProperty`**：声明在 `FVerseFunctionRef` 上的 `FProperty`，让「Verse 函数值」成为可反射/可序列化的字段。
- **`VBPVMDynamicProperty`**：把任意 `VValue` 作为 UObject 的**动态属性**暴露。
- **`VBPVMRuntimeType`**：Verse 运行期类型（`VType`）的反射包装。
- **`VerseValueProperty` / `VerseClassProperty` / `VerseStringProperty` / `FVRestValueProperty`**（`CoreUObject/Public/UObject/` 下）：存储 `VValue`、`VClass`、`verse::string` 的原生属性类型。`FVRestValueProperty` 是「UObject 里指向 `VCell`」的跨堆引用，是 Verse↔UE 双堆互相引用的关键（见 [verse_vm.md](verse_vm.md) §5 的 FrankenGC）。

`VerseNative` 插件把这些属性与高层 API 串起来：`VerseRuntimeType.h` / `VerseRuntimeTypeProperty.h` 提供 `FRuntimeType`（Verse 类型值的反射表示），`VerseValue.h` 提供 `FVerseValue`（任意 Verse 值的高层包装）。

---

## 6. Verse Call Cpp

**Verse 调用 C++** 的完整链路是：Verse 源码里声明 `<native>` 函数 → 编译器/VNI 生成 C++ thunk 胶水 → 运行期把 thunk 指针绑定到 VM 里的 `VNativeProcedure` → Verse 调用时经 `FNativeConverter` 转换参数、执行 C++ 函数。

### 6.1 两条执行路径

`VerseInteropMacros.h` 通篇用 `#if WITH_VERSE_BPVM` 分叉，同一套宏在两个后端下展开成完全不同的形态：

| 维度 | BPVM 路径（`WITH_VERSE_BPVM`） | VVM 路径（`!WITH_VERSE_BPVM`） |
|------|-------------------------------|-------------------------------|
| thunk 签名 | `void(UObject*, FFrame&, RESULT_DECL)`（UFunction `exec*`） | `FNativeCallResult(FRunningContext, VValue Self, TArrayView<VValue> Args)` |
| 参数搬运 | `Stack.StepCompiledIn<FProperty>` 按 `FProperty` 从栈读 | `FNativeConverter::FromVValue` 逐个 `VValue` 转换 |
| 绑定目标 | `UVerseClass::BindVerseFunction` → `UFunction::SetNativeFunc` | `VNativeProcedure::SetThunk` |
| 事务 | `InvokeInternal` 里 `ProcessEvent`，必要时 `UE_AUTORTFM_TRANSACT` | `V_NATIVE_END` 里 `AutoRTFM::Close`/`Open` |

对应宏：

```cpp
// 声明/定义 thunk（两端一致，展开不同）
V_DECLARE_EXEC(MangledFunctionName)   //  → V_DECLARE_FUNCTION(_exec_<Mangled>___)
V_DEFINE_EXEC(ClassName, MangledFunctionName)

// 参数 marshal 与 native 调用体
V_MARSHALLING_PARAM_BEGIN
V_MARSHAL_PARAM_INT(MyParam)          // 一整套类型分派：INT/FLOAT/STRING/OBJECT/STRUCT/ARRAY/OPTION/FUNCTION…
V_MARSHALLING_END
V_NATIVE_BEGIN(ClassName, bAlwaysOpen, ReturnType)
    // ... 用户原生实现体 ...
V_NATIVE_END(bAlwaysOpen)
```

`V_NATIVE_BEGIN/END` 在 VVM 路径下的核心语义（`VerseInteropMacros.h` 第 402–453 行）：

1. 构造 `__Impl` lambda（`AUTORTFM_ENABLE_IF(!bAlwaysOpen)`），内部 `TToVValue<NativeReturnType> __NativeReturnValue;`。
2. `AutoRTFM::Close(__Impl)` 在**事务内**执行原生实现（`bAlwaysOpen` 则直接跑）。
3. 事务失败 → `__Context.HandleAutoRTFMFailure(__CloseStatus)`；否则把返回值 `FNativeConverter::ToVValue(...)` 包成 `FOpResult::Return`。
4. `__ControlFlow` 支持 `Return` / `Resume` / `Yield` 三种控制流，对应 `decides`（失败）与协程让出。

### 6.2 VNI 代码生成器（VerseNativeInterfaceGen）

生成器位于 `Engine/Source/Programs/Solaris/VerseNativeInterfaceGen/`，是一个 **uLang 工具链注入**——`VniToolchainInjection.h` 里的 `FVerseNativeInterfaceGenerator : IIntraSemAnalysisInjection`，在**语义分析之后**介入（`Ingest`）。

工作流程：

1. `FDefinitionInfoCache` 遍历语义程序，为每个 `CClass`/`CEnumeration`/`CModulePart`/`CFunction` 建立 `FDefinitionInfo`（C++ 名、导入名、Verse 名、混淆名、装饰名）与 `FFunctionInfo`（`bNative` / `bNativeCall` / `bDecides` / `bSuspends` / `bNoRollback` / `bAlwaysOpen` 等）。
2. `FUhtSpecifiers` 读取 Verse 属性（`<native>` `<decides>` `<suspends>` 等），产出 UHT/`UFUNCTION` 说明符。
3. `FNativeInterfaceWriter` 写出每个原生类型/模块的 C++ `.h` 与 `.ipp`：
   - `WriteFunctionThunk` → 生成 thunk（见下）。
   - `WriteFunctionCallable` → 生成 `native_callable`（C++ 可调用的 Verse 函数）声明。
   - `WriteClassDef` / `WriteClassImpl` → `MY_CLASS_DEF()` / `MY_CLASS_IMPL()` 宏。
4. `FNativeFileGenerator::Emit` 落盘 `.h`/`.ipp` + VNI 包描述符（`EmitVniPackageDescs`）。

`WriteFunctionThunk` 生成的 thunk 结构（`NativeInterfaceWriter.cpp` 第 2024 行起）：

```cpp
V_DECLARE_EXEC(MangledVerseName) {            // _exec_ 包装：检查是否存在原生实现
    V_STATIC_ASSERT_HAS_METHOD(...);
    V_CALL_IMPL(ThisClass, MangledVerseName); // 转发到 _impl_
}
V_DECLARE_IMPL(MangledVerseName, V_HAS_METHOD(ThisClass, MangledVerseName)) {
    V_MARSHALLING_PARAM_BEGIN
        V_MARSHAL_PARAM_* / V_MARSHAL_PARAM_NAMED(...)   // 按类型逐个转换参数
    V_MARSHALLING_END
    V_NATIVE_BEGIN(ThisClass, bAlwaysOpen, ReturnType)
        __NativeReturnValue = V_THIS->CppName(params...); // 真正调用用户 C++ 方法
    V_NATIVE_END(bAlwaysOpen)
    V_REPORT_FUNCTION_CALL("...", "...");                // 函数调用上报（analytics）
}
```

`V_MARSHAL_PARAM_*` 是**按 Verse 类型分派**的一整套宏（`NativeInterfaceWriter.cpp` 第 2899 行起）：`Int`→`int64`、`Float`→`double`、`Logic`→`bool`、`String`、`Char8/Char32`、`Enum`、`Object`（→`TNonNullPtr<T>`）、`Struct`、`Interface`（→`TInterfaceInstance<T>`）、`Array`（→`TArray`）、`Map`、`Option`（→`TOptional`）、`Tuple`、`Function`（→`TVerseFunction<T>`）、`Type`（→`verse::...<T>`）。

### 6.3 原生过程 `VNativeProcedure` 与 Thunk

`VVMNativeProcedure.h`：VVM 里「由 C++ 实现的函数」是一个 `VNativeProcedure : VCell`：

```cpp
using FNativeCallResult = FOpResult;
using Args = TArrayView<VValue>;
using FThunkFn = FNativeCallResult (*)(FRunningContext, VValue Self, Args Arguments);

struct VNativeProcedure : VCell {
    static constexpr char DecoratorString[] = "Native";
    uint32 NumPositionalParameters;
    FThunkFn Thunk;                        // 真正要调的 C++ 函数
    TWriteBarrier<VUniqueString> Name;
};
```

**绑定即「按装饰名查找 + 填 Thunk」**（`VNativeProcedure::SetThunk`，`VVMNativeProcedure.cpp` 第 51 行）：

```cpp
// 原生函数住在公开入口点之下的嵌套 Verse 路径：
// Wrapper: (/.../定义:)(/.../被覆写函数:)FunctionName(...)
// Native:  (/.../定义/(/.../被覆写函数:)FunctionName(...):)Native
Name = Names::GetDecoratedName(VerseScopePath, DecoratedName, DecoratorString);
VNativeProcedure* Procedure = Package->LookupDefinition<VNativeProcedure>(Name);
Procedure->Thunk = NativeThunkPtr;
```

也就是说：Verse 里一个 `<native>` 函数在 VM 里会被编译成「公开入口字节码包装 + 嵌套的 `:Native` 原生过程」。包装处理具名参数/tuple 解包等（原生只支持扁平参数表），真正干活的是 `VNativeProcedure.Thunk`。

### 6.4 值转换 `FNativeConverter`

`VVMNativeConverter.h` 是 VVM 路径的参数/返回值搬运核心，两个方向：

- **`FromVValue(Context, VValue, TFromVValue<T>&)`**：`VValue → C++ T`（`TFromVValue<T>` 是目标存储，`GetValue()` 取结果）。`V_MARSHAL_PARAM_TYPED` 就是用它（第 375–380 行）。
- **`ToVValue(Context, T&&)`**：`C++ T → VValue`（返回值路径用）。

对 `FVerseFunction`/`TVerseFunction` 有专门的特化（`VVMNativeFunction.h` 第 986 行起）：`ToVValue(Context, const FVerseFunction&)` 直接 `return *Function.Function;`（把函数值当 `VValue` 传回去），`FromVValue` 反解。

原生结构体另有 `VVMNativeStruct.h` / `VVMNativeConstructorWrapper.h` / `VVMNativeRef.h` / `VVMNativeTuple.h` / `VVMNativeString.h` / `VVMNativeRational.h` 等，分别处理结构体值、构造器包装、引用、元组、字符串、有理数在 VM 与 C++ 间的表示与转换。

### 6.5 运行期绑定汇总

- **VVM 路径**：`UVerseClass::BindVerseCallableFunctions(VersePackage, VerseScopePath)` 遍历 `VerseCallableThunks`，逐个 `VNativeProcedure::SetThunk`（`VVMVerseClass.cpp` 第 2218 行）。thunk 表来自 `SetVerseCallableThunks`（`RegisterVerseCallableThunks` 在 UHT codegen 里调用）。VM 侧的 `BindNativeClass` / `ConstructNativeDefaultObject` / `LoadImport` op 完成类级绑定与原生默认对象构造。
- **BPVM 路径**：`UVerseClass::SetNativeFunctionPointers` 遍历 `NativeFunctionLookupTable`，`Function->SetNativeFunc(...)` + 置 `FUNC_Native`（第 318 行）；`BindVerseFunction` / `BindVerseCoroClass` 按装饰名把 `FNativeFuncPtr` 绑到 `UFunction`。

---

## 7. Cpp Call Verse

**C++ 调用 Verse** 的入口是一组「Verse 函数值」类型。核心矛盾：`CoreUObject`（反射层）不知道 VM 的存在，但 `FVerseFunctionProperty` 又得声明在这些类型上。解决方式——**基类放进 CoreUObject、调用逻辑放进 VerseNative 插件**，并强制 layout 一致。

### 7.1 `FVerseFunctionRef`（基类，只表示、不调用）

`VBPVMFunctionRef.h` 定义「BPVM 眼中的 Verse 函数值」= `{ SelfObject, FunctionName }`：

```cpp
struct FVerseFunctionRef {
    TObjectPtr<UObject> SelfObject;         // 函数绑定的对象
    mutable FName FunctionName{NAME_None};  // 该对象类上 UFunction 的混淆名
    UFunction* FindFunction() const;        // 解析成 UFunction，必要时改写为混淆名
};
```

`FindFunction()` 是精髓：先 `SelfObject->FindFunction(FunctionName)` 找；找不到则 `Cast<UVerseClass>(SelfObject->GetClass())->GetFunctionMangledName(FunctionName)` **现场把非混淆名改写成混淆名**（`FunctionName` 是 `mutable` 的）。所以这个值可以「携带未混淆名」，直到被查询时才落地。

### 7.2 `FVerseFunction`（加调用逻辑，双实现）

`VVMNativeFunction.h` 的 `FVerseFunction : public FVerseFunctionRef`，按后端有两个实现：

**BPVM 版**：持有 `FVerseFunctionRef` 的 `{SelfObject, FunctionName}`，调用时 `FindFunction()` 拿到 `UFunction`，经 `_Verse_Private___::Invoke`（栈上 alloca 参数缓冲 + `ProcessEvent`）。提供 `Invoke` / `VoidInvoke` / `RefInvoke`（返回 `EVerseNativeCallResult`）/ `TaskInvoke` / `DecidesInvoke`。

**VVM 版**：持有 `TWriteBarrier<VFunction> Function`（真正的 VM 函数值）。构造时两种来源：
- `(ExecContext, InContext, DecoratedFunctionName)`：`UVerseClass::LoadField` 从 UObject 字段里读（`LoadField` 返回 `FOpResult`，永远应 `IsReturn`）。
- `(ExecContext, VersePackageName, VerseScopePath, DecoratedFunctionName)`：`GlobalProgram->LookupPackage` → `LookupDefinition<VFunction>` 按装饰名直接查模块函数。

调用核心 `InvokeInternal`：`FNativeConverter::ToVValue` 把每个 C++ 参数转成 `VValue`，`Context.EnterVM([&]{ Function->Invoke(Args) })` 进入 VM 执行，再 `FNativeConverter::FromVValue` 把返回值转回 C++。失败（`decides`）对应 `FOpResult::Fail` → `TOptional{}`；协程用 `Function->Spawn` + `FVerseTask`。

**关键约束**：`static_assert(sizeof(FVerseFunction) == sizeof(FVerseFunctionRef));`——因为 `FVerseFunctionProperty` 声明在**基类**上，按基类 layout 寻址字段，派生类**不得新增数据成员**（否则会静默描述错每个反射的函数字段）。

### 7.3 `TVerseFunction<Signature>` 特化

`TVerseFunction<ReturnType(ParamTypes...)>` 按签名把 `operator()` 绑定到正确的 `Invoke`，让 C++ 能以**自然语法**调用 Verse 函数（`VNI` 按函数类型实例化它）。特化覆盖：

- 普通返回：`operator()(ExecContext, Args...) → TIfNoRuntimeError<ReturnType>`。
- 不完整/引用返回：`EVerseNativeCallResult(ReturnType&, ...)`，`RefInvoke` 经第二参数出结果。
- `void` 返回：`void(ParamTypes...)`，`VoidInvoke`。
- `decides`：`ReturnType(FDecidesContext, ...)` → `TVerseDecidesFunctionBase`，`DecidesInvoke` 返回 `TOptional<ReturnType>`；`TransactThenInvoke` 在事务内调用。
- 协程：`FVerseResult(TVerseCall<ReturnType>, ...)` → `TaskInvoke`，返回 `FVerseTask`。

对单个 tuple 参数的函数，还额外提供 `operator()(ExecContext, const TNativeTuple<ArgTypes...>&)`，以 tuple 方式调用——这解决「接受单个 tuple 参数的函数」在两种调用形式下语义不同的观察问题。

### 7.4 值转换与类型

- `FNativeConverter` 对 `TVerseFunction`/`FVerseFunction` 有特化（§6.4 末）。
- `FNativeType`（`VVMNativeType.h`）：C++ 里「Verse 类型值」的不透明包装。VVM 版内部 `TWriteBarrier<VType>`，从 `UVerseClass`/`UVerseStruct` 的 `Class` 或 `GlobalProgram->LookupImport` 取得；提供 `AsUEStructNullable/AsUEClassChecked/Subsumes`。`TNativeSubtype<T, Base, bAllowInvalid>` 做子类型约束（castable / concrete / castable+concrete 三种变体）。

### 7.5 按显示名查找

`UVerseClass::FindVerseFunctionByDisplayName(Object|Class, DisplayName)`（`WITH_VERSE_BPVM` 路径）从一个 UObject 实例/类按 Verse 显示名找到 `FVerseFunctionDescriptor{ Function, DisplayName, UEName }`——这是 C++ 侧「拿到一个可调用的 Verse 函数」的高层入口，配合 `ForEachVerseFunction` 遍历。

---

## 8. 小结

### 8.1 一张图看清双向互调

```
                         Verse 源码
                             │
              ┌──────────────┴──────────────┐
              │   语义分析 (uLang 前端)        │
              └──────────────┬──────────────┘
                             │ CSemanticProgram
              ┌──────────────┴──────────────────────────┐
              │ VNI 注入 (IIntraSemAnalysisInjection)     │
              │   FNativeInterfaceWriter / FileGenerator  │
              └───┬───────────────────────┬──────────────┘
       C++ .h/.ipp (thunk 胶水)        VNI 包/类型描述符
        V_DECLARE_EXEC + V_NATIVE_BEGIN/END  FVniPackageDesc / FVniTypeDesc
              │                                  │
              ▼                                  ▼
   ┌───────────────────────────────┐   FVniPackageAutoRegistrar
   │ 运行期绑定                     │   FVniTypeAutoRegistrar → IVerseNativeModule
   │  BPVM: UFunction::SetNativeFunc │         │
   │  VVM : VNativeProcedure::SetThunk│◄────────┘
   └──────────────┬────────────────┘
                  │
        Verse ──Call──► C++ :  VValue --FromVValue--> C++ 参数 → V_THIS->Method()
        C++   ──Call──► Verse:  FVerseFunction/TVerseFunction.Invoke → EnterVM → VFunction::Invoke
                  │
                  └── FNativeConverter 双向做 VValue ↔ C++ 值转换
```

### 8.2 三个核心设计

1. **反射是「双面」的**：一面是 `UVerse*` 把 Verse 类型映射进 UObject（Verse 类型可序列化、可编辑、对 Blueprint 可见）；另一面是 `VClass/VShape/VEnumeration` 把 UObject 类型映射进 VM（C++ 类型在 Verse 堆里有运行期表示）。二者通过 `NativeTypeDesc`（`FVniTypeDesc`）对齐。

2. **名称修饰是反射的底座**：大小写不敏感化（CRC mangling）、装饰名（`(...)Name`）、Verse↔UE 名互转、旧混淆名映射（`FunctionMangledNames`），共同解决了「大小写敏感 Verse ↔ 大小写不敏感 FName」「含非法字符 ↔ UE 合法名」「版本升级改名不破坏引用」三个难题。

3. **双向互调复用同一套「原生过程」抽象**：Verse→C++ 是 `<native>` 函数编译成 `VNativeProcedure` + thunk（`FThunkFn = FOpResult(FRunningContext, VValue, Args)`）；C++→Verse 是 `FVerseFunction` 持有 `VFunction` 再 `Invoke`。两端参数/返回值统一走 `FNativeConverter`，事务/效应/失败语义分别由 `V_NATIVE_BEGIN/END` 的 `AutoRTFM` 与 `FOpResult` 承载。

### 8.3 与 UE 传统反射的对比

| 维度 | UE 传统（UHT） | Verse（UHT + VNI） |
|------|---------------|-------------------|
| 反射来源 | C++ `.generated.h`（UHT 从 C++ 生成） | 双来源：UHT 从 C++ 生成 `UVerse*`，VNI 从 **Verse 源码**生成 C++ 胶水 |
| 类型实体 | `UClass/UFunction/UScriptStruct/UEnum` | 同名派生 `UVerseClass/UVerseFunction/UVerseStruct/UVerseEnum` |
| 名字 | `FName`（大小写不敏感） | Verse 大小写敏感 → 需 CRC mangling + 装饰名 |
| 原生绑定 | `UFunction::SetNativeFunc` + `exec*` | 双后端：`SetNativeFunc`（BPVM）/ `VNativeProcedure::SetThunk`（VVM） |
| 值转换 | `FProperty` 直接读写属性内存 | `FNativeConverter` 在 `VValue` 与 C++ 值间双向转换 |
| 代码生成时机 | 构建期（UHT） | 语义分析后注入（VNI），与编译管线一体 |
| 跨语言调用 | Blueprint ↔ C++ 经 `FProperty` | Verse ↔ C++ 经 `FVerseFunction`/`TVerseFunction` + `FNativeConverter` |
