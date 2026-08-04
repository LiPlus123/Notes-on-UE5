# UE5 UObject 系统原理与设计

> 本文档基于 UE5 引擎源代码（分支 5.5）编写，深入解析 UObject 系统的核心架构和设计原理。

---

## 目录

1. [反射与 UHT](#1-反射与-uht)
2. [对象构造与生命周期](#2-对象构造与生命周期)
3. [垃圾回收](#3-垃圾回收)
4. [序列化](#4-序列化)
5. [包与资产](#5-包与资产)

---

## 1. 反射与 UHT

### 1.1 概述

UE5 的反射系统是 UObject 生态的基石。它允许运行时获取类型信息——枚举类成员、查询属性、动态调用函数——这些能力支撑起蓝图、编辑器、网络复制、GC 引用收集、序列化等所有核心子系统。

反射数据由 **Unreal Header Tool (UHT)** 在编译前生成。UHT 解析带有特殊宏（`UCLASS`、`UPROPERTY`、`UFUNCTION` 等）的头文件，生成 C++ 代码将类型信息以静态编译期数据的形式注入程序。程序启动时，这些静态数据被"激活"为运行时的 `UClass`、`UEnum`、`FProperty` 对象。

### 1.2 UHT 架构

在 UE5 中，UHT 是一个 **C# 程序**（不再使用 C++），位于：

```
Engine/Source/Programs/Shared/EpicGames.UHT/
├── Types/                    # 内部 AST 类型表示
│   ├── UhtClass.cs           # UClass 的 AST 节点
│   ├── UhtStruct.cs          # UStruct 的 AST 节点
│   ├── UhtEnum.cs            # UEnum 的 AST 节点
│   ├── UhtFunction.cs        # UFunction 的 AST 节点
│   ├── UhtProperty.cs        # FProperty 的 AST 节点
│   └── Properties/           # 41 个属性类型文件（每种属性一个）
├── Parsers/                  # 解析器
│   ├── UhtHeaderFileParser.cs
│   ├── UhtClassParser.cs
│   ├── UhtFunctionParser.cs
│   └── UhtPropertyParser.cs
├── Specifiers/               # 修饰符处理
│   ├── UhtClassSpecifiers.cs
│   ├── UhtFunctionSpecifiers.cs
│   └── UhtPropertyMemberSpecifiers.cs
└── Exporters/CodeGen/        # 代码生成
    ├── UhtHeaderCodeGeneratorHFile.cs   # 生成 .generated.h
    ├── UhtHeaderCodeGeneratorCppFile.cs # 生成 .gen.cpp
    └── UhtPackageCodeGenerator.cs
```

**处理流水线**：
1. UHT 解析 `.h` 文件，构建内部 AST
2. 修饰符解析器处理 UCLASS/UPROPERTY/UFUNCTION 等关键字和元数据
3. 代码生成器输出两个文件：
   - **`.generated.h`** — 宏展开和类型声明
   - **`.gen.cpp`** — 编译期静态数据表和注册代码

### 1.3 反射宏系统

所有反射宏定义在 `CoreUObject/Public/UObject/ObjectMacros.h` 中。

#### 核心宏

```cpp
// 这些宏为空——它们被 UHT 解析，但对 C++ 编译器无意义
#define UPROPERTY(...)       // 空
#define UFUNCTION(...)       // 空
#define USTRUCT(...)         // 空
#define UENUM(...)           // 空
#define UDELEGATE(...)       // 空
#define UMETA(...)           // 空
#define UPARAM(...)          // 空

// UCLASS 是唯一需要预处理器参与的关键字宏
#define UCLASS(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_PROLOG)

// GENERATED_BODY 展开为对应的标识符
#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY);
```

`BODY_MACRO_COMBINE` 将 `CURRENT_FILE_ID` + `_` + `__LINE__` + 后缀拼接为一个标识符。`CURRENT_FILE_ID` 由 `.generated.h` 文件定义为该头文件的唯一整数 ID。

#### 修饰符关键字命名空间

为了提供 IDE 自动补全支持，修饰符关键字被定义在特定的 C++ 命名空间中：

```cpp
namespace UC {  // UCLASS 关键字
    static constexpr auto Abstract = 1;
    static constexpr auto BlueprintType = 2;
    static constexpr auto config = 3;
    static constexpr auto Transient = 4;
    static constexpr auto Within = 5;
    // ...
}

namespace UF {  // UFUNCTION 关键字
    static constexpr auto BlueprintCallable = 0;
    static constexpr auto Server = 1;
    static constexpr auto Client = 2;
    static constexpr auto NetMulticast = 3;
    static constexpr auto Exec = 4;
    // ...
}

namespace UP {  // UPROPERTY 关键字
    static constexpr auto EditAnywhere = 0;
    static constexpr auto BlueprintReadWrite = 1;
    static constexpr auto Replicated = 2;
    static constexpr auto Category = 3;
    // ...
}

namespace US {  // USTRUCT 关键字
    static constexpr auto BlueprintType = 0;
    static constexpr auto Atomic = 1;
    // ...
}

namespace UM {  // 元数据修饰符
    static constexpr auto ToolTip = 1;
    static constexpr auto DisplayName = 2;
    static constexpr auto ClampMin = 3;
    static constexpr auto ClampMax = 4;
    // ...
}
```

### 1.4 .generated.h 与 .h 的协作

一个典型的 UE 头文件结构如下：

```cpp
// MyClass.h
#pragma once

#include "CoreMinimal.h"
#include "MyClass.generated.h"  // <-- 由 UHT 生成，必须放在所有 UCLASS/USTRUCT 声明之后

UCLASS(BlueprintType)
class MYGAME_API UMyClass : public UObject
{
    GENERATED_BODY()    // <-- 展开为类的样板代码

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Health;

    UFUNCTION(BlueprintCallable)
    void TakeDamage(float Damage);
};
```

**`.generated.h` 生成的内容包含**：
- 该文件中所有类型的宏定义（将 `CURRENT_FILE_ID_X_PROLOG`、`CURRENT_FILE_ID_X_GENERATED_BODY` 映射为实际代码）
- 这些宏在 `UCLASS()` 和 `GENERATED_BODY()` 行处通过行号匹配调用

**`.gen.cpp` 生成的内容包含**：
- 每个属性的 `UECodeGen_Private::F*PropertyParams` 静态数据
- 元数据 `FMetaDataPairParam` 数组
- `IMPLEMENT_CLASS` / `IMPLEMENT_STRUCT` 等注册宏
- `Z_Construct_UClass_*` / `Z_Construct_UScriptStruct_*` 工厂函数

### 1.5 反射类型层次结构

#### UField 体系（基于 UObject 的反射类型）

位于 `CoreUObject/Public/UObject/Class.h`：

```
UObject                          // 所有 UE 对象的根
 +-- UField                      // 反射数据基类，包含 Next 指针形成链表
      +-- UStruct                // 包含字段/属性的类型基类
      |    +-- UScriptStruct     // C++ 结构体的反射
      |    +-- UClass            // C++ 类的反射（最重要）
      |    +-- UFunction         // C++ 函数的反射
      |         +-- UDelegateFunction
      |              +-- USparseDelegateFunction
      +-- UEnum                  // C++ 枚举的反射
```

**UClass** 是最核心的反射类型，它持有：
- `ClassDefaultObject` — 类默认对象 (CDO)
- `ClassConstructor` — 构造函数指针
- `ClassFlags` — EClassFlags 标志位
- `FuncMap` — 函数名到 UFunction 的映射（TMap）
- `PropertyLink` / `RefLink` / `DestructorLink` / `PostConstructLink` — 属性链表
- `ReferenceSchema` — GC 引用模式（用于快速引用遍历）

**UFunction** 持有：
- `Func` — 原生函数指针 (FNativeFuncPtr)
- `FunctionFlags` — 函数标志 (FUNC_*)
- `NumParms` / `ParmsSize` — 参数数量和大小
- `Script` — 蓝图字节码数组

#### FProperty 体系（非 UObject 的属性类型）

位于 `CoreUObject/Public/UObject/Field.h` 和 `UnrealType.h`：

```
FField                           // 所有属性类型的基类
 +-- FProperty                   // 属性基类
      +-- FBoolProperty          // 布尔 / 位域
      +-- FNumericProperty       // 数值抽象基类
      |    +-- FByteProperty     // uint8/枚举
      |    +-- FInt8Property     // int8
      |    +-- FInt16Property    // int16
      |    +-- FIntProperty      // int32
      |    +-- FInt64Property    // int64
      |    +-- FUInt16Property   // uint16
      |    +-- FUInt32Property   // uint32
      |    +-- FUInt64Property   // uint64
      |    +-- FFloatProperty    // float
      |    +-- FDoubleProperty   // double
      +-- FObjectPropertyBase    // 对象引用抽象基类
      |    +-- FObjectProperty   // TObjectPtr<UObject>
      |    +-- FWeakObjectProperty
      |    +-- FLazyObjectProperty
      |    +-- FSoftObjectProperty
      |    +-- FClassProperty    // TSubclassOf<>
      |    +-- FSoftClassProperty
      +-- FInterfaceProperty     // TScriptInterface<>
      +-- FNameProperty          // FName
      +-- FArrayProperty         // TArray<>
      +-- FMapProperty           // TMap<>
      +-- FSetProperty           // TSet<>
      +-- FStructProperty        // 嵌套结构体
      +-- FDelegateProperty      // 单播委托
      +-- FMulticastDelegateProperty // 多播委托
      +-- FEnumProperty          // 作用域枚举
      +-- FStrProperty           // FString
      +-- FTextProperty          // FText
      +-- FOptionalProperty      // TOptional<>
```

#### 属性模板层次结构

FProperty 还使用 C++ 模板层次提供类型安全的操作（`UnrealType.h` 行 1403-1693）：

```
TPropertyTypeFundamentals<TCppType>  // 大小、对齐、类型化 Get/Set、计算的 CPF_ 标志
 +-- TProperty<TCppType, TBase>      // InitializeValue, DestroyValue, CopyValues, GetValueTypeHash
      +-- TProperty_WithEqualityAndSerializer<TCppType, TBase>
                                      // Identical()、SerializeItem()
           +-- TProperty_Numeric<TCppType>
                                      // 数值类型转换
```

### 1.6 反射数据注册流程

启动时的注册过程如下：

**阶段 1 — UHT 生成静态数据**：`.gen.cpp` 中包含编译期常量数据表：

```cpp
// .gen.cpp 中
static FRegisterCompiledInInfo AutoInitialize_UMyClass(...); // 静态全局变量

const UECodeGen_Private::FFloatPropertyParams UMyClass_Health = {
    "Health",
    nullptr,
    (EPropertyFlags)CPF_Edit | CPF_BlueprintVisible,
    UE_ARRAY_COUNT(...),
    METADATA_PARAMS(...),
    ...
};
```

**阶段 2 — 静态初始化器**：`IMPLEMENT_CLASS` 宏（`ObjectMacros.h` 行 2101）创建一个 `FRegisterCompiledInInfo` 静态对象，其构造函数调用 `RegisterCompiledInInfo()`。同样的模式也用于结构体（`FStructRegisterCompiledInInfo`）和枚举（`FEnumRegisterCompiledInInfo`）。

**阶段 3 — GetPrivateStaticClassBody()**：当 `TClass::GetPrivateStaticClass()` 首次被调用时，会创建 `UClass*` 单例，传入：包名、类名、大小/对齐、类标志、构造函数指针、VTable 辅助函数、`FUObjectCppClassStaticFunctions`（AddReferencedObjects 等）、基类引用等。

**阶段 4 — 模块启动**：`FCoreUObjectModule::StartupModule()`（`CoreNative.cpp`）调用 `UClassRegisterAllCompiledInClasses()` 处理所有已注册的编译信息并构造 `UClass` 对象。

**阶段 5 — 属性构造**：属性使用 `UECodeGen_Private::F*PropertyParams` 静态数据构造。每个属性构造函数接受其特定的 Params 结构体。

**阶段 6 — Link()**：所有属性构造完成后，调用 `UStruct::Link()` 建立 `PropertyLink`、`RefLink`、`DestructorLink`、`PostConstructLink` 链表，计算内存偏移，并组合 GC 的引用字节码模式 (Reference Schema)。

### 1.7 元数据系统

元数据系统有两层：

**A. UMetaData（旧版）**：`MetaData.h` 行 25。一个 `UObject` 子类，存储在包中。以 `FWeakObjectPtr -> TMap<FName, FString>` 的映射形式存储每个对象的键值对元数据。用于蓝图生成的内容。

**B. FField 内联元数据（新版）**：`Field.h` 行 756。每个 `FField`（及 `FProperty`、`UField` 等）拥有自己的 `TMap<FName, FString>* MetaDataMap`。API 包括：
- `HasMetaData(Key)` — 检查元数据是否存在
- `FindMetaData(Key)` / `GetMetaData(Key)` — 获取元数据值
- `GetBoolMetaData(Key)` / `GetIntMetaData(Key)` / `GetFloatMetaData(Key)` — 类型化访问
- `SetMetaData(Key, Value)` / `RemoveMetaData(Key)` — 修改

---

## 2. 对象构造与生命周期

### 2.1 类层次结构

UObject 的类层次从低级到高级如下：

```
UObjectBase                          // 最低层——内存、名称、标志、索引
 +-- UObjectBaseUtility              // 标志操作、标记系统、GC 集群
      +-- UObject                    // 完整对象：生命周期、序列化、属性
```

**UObjectBase**（`UObjectBase.h` 行 41-343）持有四个基本数据成员：
- `ObjectFlags` — EObjectFlags
- `InternalIndex` — 在 GUObjectArray 中的索引
- `ClassPrivate` — 指向 UClass 的指针
- `NamePrivate` — FName
- `OuterPrivate` — 指向外部对象的 UObject 指针

构造时调用 `AddObject()` 将对象注册到全局对象哈希表和数组中。析构时验证对象已被正确销毁（Name 必须为 NAME_None，InternalIndex 必须为 INDEX_NONE）。

**UObjectBaseUtility**（`UObjectBaseUtility.h` 行 45-827）添加：
- 安全的标志操作：`SetFlags()`、`ClearFlags()`、`HasAnyFlags()`、`HasAllFlags()`
- GC 标记系统：`Mark()`、`UnMark()`、`HasAnyMarks()`
- `MarkAsGarbage()` / `ClearGarbage()` — 设置/清除 `RF_MirroredGarbage`
- `AddToRoot()` / `RemoveFromRoot()` — 控制根集保护

### 2.2 对象标志 (EObjectFlags)

定义在 `ObjectMacros.h` 行 534-583。关键的与生命周期相关的标志：

| 标志 | 十六进制值 | 含义 |
|------|-----------|------|
| `RF_NeedInitialization` | 0x00000200 | 对象尚未完成初始化。分配时设置，在 `PostConstructInit()` 结束时清除 |
| `RF_NeedLoad` | 0x00000400 | 对象需要从磁盘加载 |
| `RF_NeedPostLoad` | 0x00001000 | 需要调用 PostLoad() |
| `RF_NeedPostLoadSubobjects` | 0x00002000 | 子对象需要实例化和组件引用修复 |
| `RF_BeginDestroyed` | 0x00008000 | BeginDestroy() 已被调用 |
| `RF_FinishDestroyed` | 0x00010000 | FinishDestroy() 已被调用 |
| `RF_ClassDefaultObject` | 0x00000010 | 此对象是其类的 CDO |
| `RF_ArchetypeObject` | 0x00000020 | 可用作实例化模板的对象 |
| `RF_DefaultSubObject` | 0x00040000 | 在类构造函数中创建的子对象模板及其所有实例 |
| `RF_MirroredGarbage` | 0x40000000 | 镜像 EInternalObjectFlags::Garbage 用于快速 GC 检查 |
| `RF_Standalone` | 0x00000002 | 即使未被引用也保留 |
| `RF_Transactional` | 0x00000008 | 参与撤销/重做 |
| `RF_Transient` | 0x00000040 | 不保存到磁盘 |
| `RF_Public` | 0x00000001 | 在其包外可见 |

`RF_PropagateToSubObjects` 掩码（行 601）：
```cpp
#define RF_PropagateToSubObjects (RF_Public | RF_ArchetypeObject | RF_Transactional | RF_Transient)
```
这些标志会从 Outer 传播到 `CreateDefaultSubobject` 创建的子对象。

### 2.3 对象创建路径

#### NewObject() 入口

模板函数（`UObjectGlobals.h` 行 1768-1831）构造 `FStaticConstructObjectParameters` 并调用 `StaticConstructObject_Internal()`。

#### StaticConstructObject_Internal

位于 `UObjectGlobals.cpp` 行 4494-4555。核心的分配和构造函数：

1. 调用 `StaticAllocateObject()` 分配内存并注册到对象系统
2. 调用类构造函数：`(*InClass->ClassConstructor)(FObjectInitializer(Result, Params))`
3. `FObjectInitializer` 在栈上构造，传递给 C++ 构造函数链

#### StaticAllocateObject

位于 `UObjectGlobals.cpp` 行 3380-3520+：

1. 验证：类不能是抽象的，内部对象必须兼容 `ClassWithin`，GC 不能正在进行
2. 对于 CDO，强制名称格式为 `"Default__" + ClassName`
3. 如果 `InName == NAME_None`，通过 `MakeUniqueObjectName()` 生成唯一名称
4. 如果存在具有相同名称/Outer 的对象：要么重用它（子对象回收），要么 fatal
5. 通过 `GUObjectAllocator.AllocateUObject()` 分配原始内存

#### FObjectInitializer

位于 `UObjectGlobals.h` 行 1188-1642。FObjectInitializer 是一个 RAII 对象，在 UObject 构造期间存在于栈上。

**核心成员**：
- `Obj` — 正在构造的对象
- `ObjectArchetype` — 从中复制属性的模板（默认为类 CDO）
- `SubobjectOverrides` — 派生类的默认子对象类型覆盖
- `ComponentInits` — 构造期间创建的需要初始化的子对象列表
- `InstanceGraph` — 用于在复制/加载期间实例化子对象

**构造函数流程**（`Construct_Internal`，行 3759-3781）：
1. 递增 `ThreadContext.IsInConstructor`
2. 保存之前的 `ConstructedObject`，设置为当前对象
3. 将此初始化器推入线程的初始化器栈
4. 调用 `UClass::SetupObjectInitializer()` 应用类级别的子对象覆盖

**析构函数流程**（`~FObjectInitializer`，行 3802-3895）：
1. 递减 `ThreadContext.IsInConstructor`，恢复之前的 `ConstructedObject`
2. 清除 `EInternalObjectFlags::PendingConstruction`
3. 如果没有 `ObjectArchetype`，默认为 `Class->GetDefaultObject()`
4. 调用 `PostConstructInit()` 完成初始化
5. 弹出线程初始化器栈

### 2.4 PostConstructInit

位于 `UObjectGlobals.cpp` 行 3898-4096。在所有 C++ 构造函数返回后运行：

1. 如果 `bShouldInitializePropsFromArchetype`：调用 `InitProperties()` 从原型/CDO 复制属性值
2. 调用 `InitSubobjectProperties()` 初始化所有默认子对象的属性
3. CDO 调用 `LoadConfig()`
4. 需要组件实例化时调用 `InstanceSubobjects()`
5. 对每个子对象调用 `PostReinitProperties()`
6. 对主对象调用 `PostInitProperties()`
7. 调用 `Class->PostInitInstance()`
8. 调用 `CheckDefaultSubobjects()` 验证所有预期的子对象都存在
9. 清除 `RF_NeedInitialization`（带内存屏障以保证线程安全）

### 2.5 CDO（类默认对象）

`UClass::CreateDefaultObject()`（`Class.cpp` 行 4814-4896）：

1. 递归确保父类 CDO 已存在
2. 对于蓝图类，预加载属性并调用 `StaticLink(true)`
3. 以 `RF_Public | RF_ClassDefaultObject | RF_ArchetypeObject` 标志分配 CDO
4. 调用类构造函数，传入包含父类 CDO 作为模板的 `FObjectInitializer`
5. `PostConstructInit` 复制父 CDO 属性、调用 `PostInitProperties()` 等
6. 返回 CDO

**CDO 子对象设置**：当 CDO 创建时，构造函数链运行 `CreateDefaultSubobject()` 调用。这些子对象获得 `RF_DefaultSubObject` 标志，并注册为类的默认子对象。当创建实例时，它们使用 CDO 作为 `ObjectArchetype`，`InstanceSubobjects()` 会创建这些默认子对象的逐实例副本。

### 2.6 默认子对象创建

在 UObject 构造函数中通过 `CreateDefaultSubobject<T>()` 调用。流程（`UObjectGlobals.cpp` 行 5510-5597）：

1. 查找 `SubobjectOverrides` 检查派生类是否覆盖了子对象类
2. 计算子对象标志：`Outer->GetMaskedFlags(RF_PropagateToSubObjects) | RF_DefaultSubObject`
3. 如果 Outer 的原型不是 CDO，在原型中查找匹配的子对象作为模板
4. 调用 `StaticConstructObject_Internal()` 创建子对象
5. 将子对象注册到 `ComponentInits` 以便后续属性初始化
6. 如果 Outer 是 CDO，还通过 `Outer->GetClass()->AddDefaultSubobject()` 注册

### 2.7 生命周期函数

| 函数 | 调用时机 | 用途 |
|------|---------|------|
| `PostInitProperties()` | 构造完成后一次性调用 | 创建持久化蓝图图形帧 |
| `PostReinitProperties()` | 从原型覆盖属性后 | 可多次调用 |
| `PostLoad()` | 仅对反序列化的对象 | 加载配置，验证子对象 |
| `ConditionalPostLoad()` | 异步加载系统调用 | 检查 RF_NeedPostLoad 标志，确保原型先加载 |
| `BeginDestroy()` | 对象被标记为垃圾后 | 断开 linker，重命名为 NAME_None |
| `IsReadyForFinishDestroy()` | BeginDestroy 后轮询 | 检查异步清理是否完成（如渲染资源） |
| `FinishDestroy()` | 对象准备好销毁时 | 销毁非原生属性，最终清理 |
| `ConditionalBeginDestroy()` | GC 调用 | 包装 BeginDestroy，防止重复执行 |
| `ConditionalFinishDestroy()` | GC 调用 | 包装 FinishDestroy，防止重复执行 |

### 2.8 完整对象生命周期图

```
创建
  NewObject<T>(Outer, ...)
    -> StaticConstructObject_Internal(params)
         -> StaticAllocateObject()          // 分配内存，注册到 GUObjectArray，设置 RF_NeedInitialization
         -> ClassConstructor(FObjectInitializer(result, params))  // 运行 C++ 构造函数链
              -> UObjectBase::UObjectBase()         // 设置标志、类、添加到哈希表
              -> UObjectBaseUtility 构造函数
              -> UObject::UObject()                 // 绑定到 FObjectInitializer
              -> 派生类构造函数                      // 调用 CreateDefaultSubobject()
         -> ~FObjectInitializer()
              -> PostConstructInit()
                   -> InitProperties()              // 从原型/CDO 复制属性
                   -> InitSubobjectProperties()     // 初始化子对象属性
                   -> LoadConfig()                  // CDO 加载配置
                   -> InstanceSubobjects()           // 创建逐实例子对象副本
                   -> PostReinitProperties()        // 每个子对象
                   -> PostInitProperties()          // 主对象
                   -> PostInitInstance()            // 类级别钩子
                   -> CheckDefaultSubobjects()       // 验证
                   -> 清除 RF_NeedInitialization
    -> 返回构造完成的对象

加载（从磁盘）
  FLinkerLoad 创建对象
    -> StaticAllocateObject() 设置 RF_NeedLoad|RF_NeedPostLoad|RF_WillBeLoaded
    -> ClassConstructor(FObjectInitializer(...))  // 最小构造
    -> Serialize(FArchive)     // 读取属性数据
    -> ConditionalPostLoad()
         -> PostLoad()
              -> LoadConfig()  //（如果 PerObjectConfig）
              -> CheckDefaultSubobjects()
         -> PostLoadSubobjects()
    -> 清除 RF_NeedLoad，设置 RF_WasLoaded

存活期
  PostEditChange / Modify   // 编辑器撤销/重做
  Serialize / PreSave / PostSave  // 保存
  MarkAsGarbage / ClearGarbage  // GC 交互

销毁（通过 GC）
  GC 标记为不可达
    -> MarkAsGarbage()  // 设置 RF_MirroredGarbage + EInternalObjectFlags::Garbage
    -> ConditionalBeginDestroy()
         -> 设置 RF_BeginDestroyed
         -> BeginDestroy()
              -> SetLinker(NULL, INDEX_NONE)
              -> LowLevelRename(NAME_None)     // 从名称哈希表移除
              -> SetExternalPackage(nullptr)
    -> ConditionalFinishDestroy()
         -> 设置 RF_FinishDestroyed
         -> FinishDestroy()
              -> DestroyNonNativeProperties()
         -> GUObjectArray.ResetSerialNumber()   // 使弱指针失效
    -> GUObjectAllocator 释放内存
         -> ~UObjectBase()
              -> 验证 Name==NAME_None, InternalIndex==INDEX_NONE
```

---

## 3. 垃圾回收

### 3.1 概述

UE5 的垃圾回收器是一个 **并行的、增量的、三色标记-清除** 收集器，具有引用消除和集群优化。核心算法围绕以下概念：

- **三色标记**：使用双缓冲可达标志实现，避免 O(N) 重置
- **并行 BFS**：通过工作窃取在多线程间分配引用遍历
- **增量时间分片**：支持跨帧暂停/恢复以消除卡顿
- **集群优化**：将相关对象组视为单个单元进行标记

### 3.2 FUObjectItem 和全局对象数组

每个 UObject 都由 `GUObjectArray`（一个 `FUObjectArray`）中的一个 `FUObjectItem` 槽位追踪。定义在 `CoreUObject/Public/UObject/UObjectArray.h`。

**FUObjectItem 结构**（紧凑到 4 字节）：
```cpp
struct FUObjectItem {
    UObjectBase* Object;           // 指向实际 UObject 的指针
    int32 Flags;                   // EInternalObjectFlags（位域，原子访问）
    int32 ClusterRootIndex;       // 所属集群索引（正数）或集群索引（负数）
    int32 SerialNumber;           // 弱对象指针序列号
    int32 RefCount;               // 防止销毁的引用计数
};
```

- `SetFlags`/`ClearFlags`/`HasAnyFlags`/`HasAllFlags` — 操作 `EInternalObjectFlags`
- `GetFlags()` 原子读取（`AtomicRead_Relaxed`）
- `AtomicallySetFlag_ForGC` / `AtomicallyClearFlag_ForGC` — 基于 CAS 的原子标志操作（仅供 GC 使用）
- `ClusterRootIndex` — 正数 = 所属集群根索引；负数 = 此对象自身即为集群根

**FUObjectArray**：
- 使用 `FChunkedFixedUObjectArray`，每块 64K 对象
- "忽略 GC" 内存池在开头（`ObjFirstGCIndex`/`ObjLastNonGCIndex`），用于永久加载的启动对象
- `AllocateUObjectIndex` / `FreeUObjectIndex` 管理对象槽位的生命周期
- `AddUObjectCreateListener`/`AddUObjectDeleteListener` 用于创建/销毁通知

### 3.3 GC 使用的内部对象标志

定义在 `CoreUObject/Public/UObject/ObjectMacros.h` 行 613：

```cpp
ReachabilityFlag0  = 1 << 0    // GC 可达性状态（双缓冲）
ReachabilityFlag1  = 1 << 1    // GC 可达性状态（双缓冲）
ReachabilityFlag2  = 1 << 2    // GC 可达性状态
Garbage            = 1 << 21   // 逻辑上已是垃圾，不应被引用
ReachableInCluster = 1 << 23   // 存在对集群中对象的外部引用
ClusterRoot        = 1 << 24   // 对象是集群的根
Native             = 1 << 25   // 原生 UClass
Unreachable        = 1 << 28   // 对象在对象图上不可达
RefCounted         = 1 << 29   // 对象有活动引用计数
RootSet            = 1 << 30   // 对象永不被 GC，即使未被引用
```

三个 `ReachabilityFlag*` 值是双缓冲的：两个被交换以充当 "Reachable" 和 "MaybeUnreachable" 标记，避免了遍历所有对象重置可达性状态。

### 3.4 标记-清除算法（完整 GC 周期）

入口点是 `CollectGarbage()` 和 `TryCollectGarbage()`（`GarbageCollection.cpp` 行 5973+）。

#### 阶段 1：预收集 (PreCollectGarbageImpl)

1. 如果需要，刷新异步加载
2. 广播 `PreGarbageCollect` 委托——用户代码可以添加根对象
3. 锁定 UObject 哈希表——防止发现新对象
4. 获取 GC 锁——阻塞异步线程
5. 设置 `GIsGarbageCollecting = true`
6. 完成任何之前挂起的增量清除

#### 阶段 2：可达性分析（核心标记阶段）

**步骤 2a：MarkObjectsAsUnreachable**（行 4207）
```cpp
// 交换 ReachabilityFlag0 和 ReachabilityFlag1 的含义
FGCFlags::SwapReachableAndMaybeUnreachable();
```
这使得所有对象立即变为 "MaybeUnreachable"，无需迭代。然后特殊状态的对象立即被标记为可达：
- 集群根及其成员（`MarkClusteredObjectsAsReachable()`）
- 根集对象（`EInternalObjectFlags::RootSet`）
- 具有 KeepFlags 的对象（编辑器中通常是 `RF_Standalone`）

**步骤 2b：PerformReachabilityAnalysisPass**（行 4275）

核心图遍历（BFS）：
1. 从 GC 屏障拉取增量累积的对象
2. 加载初始原生引用（来自 GGCObjectReferencer 的 FGCObjects）
3. 处理被屏障标记为可达的集群根
4. 调用 `TFastReferenceCollector::ProcessObjectArray()` 进行 BFS 遍历

**BFS 遍历循环**（`FastReferenceCollector.h` 行 870）：
```cpp
while (true) {
    ProcessObjects(Dispatcher, CurrentObjects);  // 访问当前块中的所有引用
    if (IsTimeLimitExceeded()) { SuspendWork(); return; }  // 增量暂停
    // 获取下一个可达对象块
    Block = RemainingObjects.PopFullBlock<Options>();
    if (!Block) {
        // 并行模式：尝试从其他线程窃取工作
        StealWork(...);
    }
    if (!Block) break;  // 所有可达对象已处理
    CurrentObjects = MakeArrayView(Block->Objects, BlockSize);
}
```

**步骤 2c：ProcessObjects — 逐对象引用收集**（行 954）

对当前工作块中的每个对象：
1. 获取对象的类 Schema（`Class->ReferenceSchema.Get()`——一个 `FSchemaView`）
2. 发出隐式引用（Class、Outer、ExternalPackage）
3. 调用 `VisitMembers()` 遍历所有声明的 GC 引用

**步骤 2d：引用处理**

当发现对另一个 UObject 的引用时：
1. 如果设置了 `EliminateGarbage` 且目标有 `Garbage` 标志：将引用置空（引用消除）
2. 否则：检查目标是否为 MaybeUnreachable
3. 如果是：原子清除 MaybeUnreachable 并设置 Reachable，然后将目标添加到工作队列

**步骤 2e：外部根**

通过 `TraceExternalRootsForReachabilityAnalysis` 委托允许外部系统（如 Verse VM）添加额外的根。

#### 阶段 3：后收集 (PostCollectGarbageImpl)

1. 更新 GC 历史记录
2. 溶解需要溶解的集群
3. 溶解不可达的集群及其所有成员
4. 清除弱引用——通过 `FWeakReferenceEliminator` 将悬空弱指针置空
5. 收集不可达对象——遍历整个对象数组，收集标记为 MaybeUnreachable 的对象到 `GUnreachableObjects`
6. 释放 GC 锁

#### 阶段 4：UnhashUnreachableObjects

对每个不可达对象：调用 `ConditionalBeginDestroy()` 并从名称表中移除。

#### 阶段 5：增量清除 (IncrementalPurgeGarbage)

两个子阶段：

**a) ConditionalFinishDestroy — 第一遍**：
- 对每个不可达对象检查 `IsReadyForFinishDestroy()`
- 如果就绪：调用 `ConditionalFinishDestroy()` -> `FinishDestroy()`
- 如果未就绪：添加到 `GGCObjectsPendingDestruction` 延迟列表

**b) 延迟 FinishDestroy 循环**：
- 持续重试延迟对象的 `IsReadyForFinishDestroy()`
- 支持异步清理（如渲染线程上释放的渲染资源）
- 有可配置的超时时间（`gc.MaxTimeForFinishDestroyGC`，默认 10s）

完成 FinishDestroy 后，对象从内存中删除，其 `FUObjectItem` 槽位返回对象数组。

### 3.5 GC Schema 系统（GC 引用模式）

位于 `CoreUObject/Public/UObject/GarbageCollectionSchema.h`。每个 UClass 有一个 `ReferenceSchema`（`FSchemaView`），将所有 GC 引用紧凑编码为字节码流。

**Schema 字节码指令（EMemberType）**：

```
Reference         // 单个 UObject* 在某个偏移处
ReferenceArray    // TArray<UObject*>
StructArray       // 包含嵌套 Schema 的结构体数组
SparseStructArray // TMap/TSet（带内部 Schema 的稀疏数组）
ARO               // 对当前对象调用 AddReferencedObjects()（隐式停止）
SlowARO           // 调用 AddReferencedObjects 但排队等待并行执行
MemberARO         // 调用结构体级别的 AddReferencedObjects
Stop              // Schema 结束（无 ARO 调用）
Jump              // 推进基指针（用于超出 16 位范围的大偏移成员）
```

**VisitMembers**（`FastReferenceCollector.h` 行 646）遍历字节码：
```cpp
for (const FMemberWord* WordIt = Schema.GetWords(); true; ++WordIt) {
    switch (Member.Type) {
        case Reference:     Dispatcher.HandleKillableReference(...);     break;
        case ReferenceArray: Dispatcher.HandleKillableArray(...);        break;
        case StructArray:   VisitStructArray(...);                       break;
        case ARO:           CallARO(...); return;                        break;
        case SlowARO:       CallSlowARO(...); return;                    break;
        case Stop:          return;
        // ...
    }
}
```

### 3.6 AddReferencedObjects 机制

位于 `Object.h` 行 778。`UObject::AddReferencedObjects` 是一个静态函数，类可以重写以添加无法通过 UProperty 表达的引用（如动态分配的对象、原生指针数组、委托目标）。

```cpp
static void AddReferencedObjects(UObject* This, FReferenceCollector& Collector);
```

`FReferenceCollector` 接口提供：
- `HandleObjectReference(UObject*& Object, ...)` — 添加单个引用
- `HandleObjectReferences(UObject** Objects, int32 Num, ...)` — 添加一批引用
- `MarkWeakObjectReferenceForClearing(UObject**, UObject*)` — 用于弱引用

### 3.7 集群系统

位于 `UObjectArray.h` 和 `UObjectClusters.cpp`。

**目的**：集群将预期共同存活/死亡的对象分组。GC 将整个集群视为单个单元——如果集群根可达，则所有成员都可达，大幅减少 GC 遍历成本。

**FUObjectCluster 结构**：
```cpp
struct FUObjectCluster {
    int32 RootIndex;                     // 集群根对象索引
    TArray<int32> Objects;               // 集群中对象的索引
    TArray<int32> ReferencedClusters;    // 此集群引用的其他集群
    TArray<int32> MutableObjects;        // 集群外仍被引用的对象
    TArray<int32> ReferencedByClusters;  // 反向：引用此集群的集群
    bool bNeedsDissolving;               // 集群需要被溶解
};
```

**GC 期间集群的处理**：
1. 在标记阶段（`MarkClusteredObjectsAsReachable`）：检查每个集群根——如果它有 RootFlags 或不是垃圾，整个集群被标记为可达
2. 成员获得原子标记，清除 MaybeUnreachable
3. 引用的集群递归标记
4. 溶解：如果集群根是垃圾，在 `DissolveUnreachableClusters()` 中溶解集群

### 3.8 增量 GC / 时间分片

**配置**：
- `gc.AllowIncrementalReachability` — 启用增量可达性分析
- `gc.IncrementalReachabilityTimeLimit`（默认 0.005s = 每帧 5ms）
- `gc.AllowIncrementalGather` — 启用增量收集
- `gc.IncrementalBeginDestroyEnabled` — 启用增量 BeginDestroy

**机制**（`ReachabilityAnalysisState.h`）：
1. **时间分片标记**：`TFastReferenceCollector` 在每个工作块后检查时间限制。如果超出，调用 `SuspendWork()` 保存状态
2. **GC 屏障**：增量 GC 进行期间，新创建的对象添加到 `GReachableObjects`
3. **恢复**：下一帧，如果 `GIsIncrementalReachabilityPending` 为 true，从暂停处恢复
4. **增量收集**：使用带 `ParallelFor` 的 `FTimeSlicer`，每线程每 10 个对象轮询一次
5. **增量清除**：使用时间限制逐批处理 `GUnreachableObjects`

### 3.9 线程模型

位于 `CoreUObject/Private/UObject/GCScopeLock.h`。全局同步单例 `FGCCSyncObject` 有三个计数器：

- **AsyncCounter**：如果非零，非游戏线程持有异步锁。GC 运行时这些线程必须阻塞
- **GCCounter**：如果非零，GC 正在运行。阻止其他线程进入
- **GCWantsToRunCounter**：GC 想运行但被阻塞。异步线程可以检查此标志自愿让出

**锁协议**：
- 非游戏线程使用 `FGCScopeGuard`（RAII）：如果 GC 正在运行则等待 `GCUnlockedEvent`，然后递增 AsyncCounter
- GC（游戏线程）使用 `AcquireGCLock()`：设置 GCWantsToRun，等待 AsyncCounter == 0，然后设置 GCCounter

**并行标记**：当设置 `EGCOptions::Parallel` 时，`ProcessAsync()` 启动多个任务图任务。每个工作线程获得自己的 `FWorkerContext` 和工作窃取队列。

### 3.10 双缓冲可达标志

`FGCFlags`（`GarbageCollectionInternalFlags.h`）管理三个 `ReachabilityFlag*` 值来实现无锁可达性切换：

- 两个标志编码 "Reachable" 与 "MaybeUnreachable" 状态（每次 GC 周期通过 `SwapReachableAndMaybeUnreachable()` 交换）
- `Unreachable` 是在完整标记过程后确认真正不可达的对象上设置的最终状态
- 避免了每次 GC 周期在所有对象上的 O(N) 标志重置

---

## 4. 序列化

### 4.1 FArchive 基类

位于 `Core/Public/Serialization/Archive.h`。`FArchive` 类私有继承自 `FArchiveState`（行 1143），将可变状态与序列化 API 分离。

**核心状态模式**（`FArchiveState` 中的位域标志）：
- `ArIsLoading` / `ArIsSaving` / `ArIsTransacting` — 互斥模式
- `ArIsPersistent` — 数据进出磁盘
- `ArIsLoadingFromCookedPackage` — 加载熟化(cooked)数据
- `ArIsFilterEditorOnly` / `ArIsSaveGame` / `ArIsNetArchive` — 上下文过滤器
- `ArIsCooking` — 从 `SavePackageData->CookContext` 派生
- `ArIsObjectReferenceCollector` / `ArIsCountingMemory` — 特殊模式
- `ArUseUnversionedPropertySerialization` — 无版本属性序列化格式

**类型化序列化**（行 1309-1577）：`FArchive::operator<<` 为所有原始类型提供重载：`ANSICHAR`、`WIDECHAR`、`uint8`、`int8`、`uint16`、`int16`、`uint32`、`int32`、`uint64`、`int64`、`float`、`double`、`bool`、`FString`。

**核心虚函数**：`Serialize(void* V, int64 Length)`、`Preload`、`Seek`、`Tell`、`TotalSize`、`Flush`、`Close`、`AttachBulkData`/`DetachBulkData`、`SerializeBulkData`、`Precache`、`SerializeCompressed`。

### 4.2 FArchive 子类层次

#### FMemoryArchive（内存序列化）
位于 `Core/Public/Serialization/MemoryArchive.h`。基类，跟踪 `int64 Offset`。

派生类：
- `FMemoryWriter` / `FMemoryReader` — 通用内存序列化
- `FBufferWriter` / `FBufferReaderBase` — 缓冲区序列化
- `FObjectWriter` / `FObjectReader` — UObject 特定的内存序列化，支持 `ArIgnoreClassRef`、`ArIgnoreArchetypeRef`、`ArNoDelta`

#### FArchiveProxy（装饰器模式）
位于 `Core/Public/Serialization/ArchiveProxy.h`。持有 `FArchive& InnerArchive`，将所有虚方法委托给内部归档器。广泛用于：
- `FArchiveLoadCompressedProxy` / `FArchiveSaveCompressedProxy` — 透明压缩
- `FArchiveFromStructuredArchiveImpl` — 将 `FStructuredArchive` 适配为 `FArchive`
- `FPropertyProxyArchive` — 从字节码序列化 `FField` 引用

#### FArchiveUObject（UObject 感知的归档器）
位于 `CoreUObject/Public/Serialization/ArchiveUObject.h`。扩展 `FArchive`，提供 UObject 指针序列化。所有 `operator<<` 重载用于：`FWeakObjectPtr`、`FLazyObjectPtr`、`FSoftObjectPtr`、`FSoftObjectPath`、`FObjectPtr`。

核心派生类：
- `FLinkerLoad` — 加载 .uasset 文件（也继承 `FLinker`）
- `FLinkerSave` — 保存 .uasset 文件（也继承 `FLinker`）
- `FDuplicateDataWriter` / `FDuplicateDataReader` — 对象复制和事务

### 4.3 FStructuredArchive — 结构化序列化

位于 `Core/Public/Serialization/StructuredArchive.h`。提供基于 `FStructuredArchiveFormatter` 的分层序列化 API。

**容器类型**（来自 `StructuredArchiveSlots.h`）：
- **FStructuredArchiveSlot** — 可容纳任何值类型的槽位
- **FStructuredArchiveRecord** — 命名字段（类似结构体/对象）
- **FStructuredArchiveArray** — 定长槽位序列
- **FStructuredArchiveStream** — 不定长槽位序列
- **FStructuredArchiveMap** — 带键的字符串键槽位

**序列化桥接宏**（`ObjectMacros.h` 行 1755）：
```cpp
#define IMPLEMENT_FARCHIVE_SERIALIZER(TClass) \
    void TClass::Serialize(FArchive& Ar) { \
        TClass::Serialize(FStructuredArchiveFromArchive(Ar).GetSlot().EnterRecord()); \
    }
```
将旧式 `Serialize(FArchive&)` 桥接到新的 `Serialize(FStructuredArchive::FRecord)`。

### 4.4 包文件格式 (FPackageFileSummary)

位于 `CoreUObject/Public/UObject/PackageFileSummary.h`。每个 .uasset 文件顶部的"目录"：

```
[FPackageFileSummary]     -- 魔数、版本、各部分偏移/计数
[Name Table]              -- 去重的 FName 数据
[SoftObjectPaths]         -- 软引用路径
[Import Map]              -- FObjectImport 条目（外部依赖）
[Export Map]              -- FObjectExport 条目（此文件中的对象）
[Depends Map]             -- 依赖关系
[Preload Dependencies]    -- 必须先于其他对象加载的对象
[Export Data]             -- 序列化的 UObject 属性数据
[Bulk Data]               -- 大型二进制数据
[Asset Registry Data]     -- 资产注册表标签
[Package Trailer]         -- 快速随机访问的查找表（UE5）
```

**FPackageFileSummary** 包含：`Tag`（魔数 `PACKAGE_FILE_TAG`）、`FileVersionUE`/`FileVersionLicenseeUE`、`CustomVersionContainer`、`PackageFlags`、名称数量/偏移、软对象路径数量/偏移、导出/导入数量/偏移、依赖偏移、压缩标志、引擎版本信息、流式安装块 ID。

### 4.5 FLinker 系统

#### FLinker（基类，`Linker.h`）
管理"与 Unreal 包相关联的数据"，作为磁盘文件和内存中 UPackage 对象之间的桥接。继承自 `FLinkerTables`，拥有：
- `FPackageFileSummary Summary` — 文件头
- `TArray<FNameEntryId> NameMap` — 名称表
- `TArray<FObjectImport> ImportMap` — 外部引用
- `TArray<FObjectExport> ExportMap` — 包内对象
- `TArray<TArray<FPackageIndex>> DependsMap` — 依赖列表
- `TArray<FSoftObjectPath> SoftObjectPathList` — 软对象路径

#### FLinkerLoad（加载链路器，`LinkerLoad.h`）
也继承 `FArchiveUObject`，使其成为既能读取磁盘又能解释 UObject 序列化的归档器。关键操作：
- `CreateLinker()` — 静态工厂：打开文件，读取摘要和表。支持异步（`CreateLinkerAsync()`）
- `Tick()` — 增量异步加载：编排多步骤过程，每步有时间限制
- `CreateExport()` — 从 FObjectExport 条目创建 UObject 实例
- `Preload()` — 将对象的属性数据从文件序列化到内存对象
- `VerifyImport()` / `VerifyImportInner()` — 验证每个导入解析为有效对象，跟踪 UObjectRedirector
- `CreateImport()` — 解析导入引用，处理循环依赖延迟
- `FixupExportMap()` — 应用类重定向，使旧类名保存的对象以新类加载

#### FLinkerSave（保存链路器，`LinkerSave.h`）
也继承 `FArchiveUObject`。处理包写入磁盘。
- 构造函数打开文件写入器（`Saver`），使用版本信息初始化 `Summary`
- `MapObject()` / `MapName()` / `MapSoftObjectPath()` — 在保存遍历期间分配索引
- 序列化 UObject 引用时将 `UObject*` 写为 `FPackageIndex`
- 管理批量数据（`.ubulk`、`.m.ubulk`、`.opt.ubulk` 文件）
- 构建 PackageTrailer 以支持快速有效载荷查找
- 支持熟化(cooking)：`bProceduralSave`、`SetFilterEditorOnly()`、编辑期属性剥离

### 4.6 UObject::Serialize

位于 `CoreUObject/Private/UObject/Obj.cpp` 行 1572：

1. **预加载**（行 1600-1613）：序列化前确保类数据和 CDO 已加载
2. **特殊模式**（行 1616-1627）：为引用收集器或计数归档器序列化 `LoadName`、`LoadOuter`、`ObjClass`
3. **事务模式**（行 1629-1677）：为撤销/重做序列化名称/Outer/包数据
4. **属性序列化**（行 1682-1693）：
   - 无版本序列化时调用 `SerializeOverriddenProperties`
   - 调用 `SerializeScriptProperties()` 序列化所有类定义的属性
5. **对象 GUID**（行 1719）：`FLazyObjectPtr::PossiblySerializeObjectGuid`
6. **稀疏类数据**（行 1722-1735）：事务模式下 CDO 的稀疏类数据

### 4.7 属性序列化

**SerializeScriptProperties**（`Obj.cpp` 行 1867）：
- 调用 `MarkScriptSerializationStart`/`End`
- 对于 CDO，调用 `StartSerializingDefaults`
- 解析差分对象（原型）以进行差分序列化
- 调用 `ObjClass->SerializeTaggedProperties()` 或 `SerializeVersionedTaggedProperties`

**每个 FProperty** 实现 `SerializeItem(FStructuredArchive::FSlot Slot, void* Value, void const* Defaults)`。各类型实现自己的序列化。

### 4.8 无版本属性序列化 (UPS)

位于 `CoreUObject/Private/Serialization/UnversionedPropertySerialization.cpp`。更快、更紧凑地替代标记属性序列化。假设写入器和读取器共享相同的属性定义。使用 `FUnversionedPropertySerializer` 缓存属性偏移和序列化策略。尽可能序列化为整数、跳过零值以紧凑表示。主要用于熟化(cooked)包。

### 4.9 BulkData 序列化

位于 `CoreUObject/Public/Serialization/BulkData.h`。

**EBulkDataFlags**：
- `BULKDATA_PayloadAtEndOfFile` — 存储在包尾部
- `BULKDATA_SerializeCompressedZLIB` — ZLIB 压缩
- `BULKDATA_OptionalPayload` — 存储在 `.uptnl` 文件
- `BULKDATA_MemoryMappedPayload` — 存储在 `.m.ubulk` 用于内存映射 I/O
- `BULKDATA_DuplicateNonOptionalPayload` — 同时在 `.ubulk` 和 `.uptnl` 中存储
- `BULKDATA_UsesIoDispatcher` / `BULKDATA_DataIsMemoryMapped` — IoStore 运行时标志

### 4.10 序列化流水线总结

```
保存路径：
UObject::Serialize(FStructuredArchive::FRecord)
  --&gt; SerializeScriptProperties(FStructuredArchive::FSlot)
      --&gt; UClass::SerializeTaggedProperties / SerializeVersionedTaggedProperties
          --&gt; [每个 FProperty] FProperty::SerializeItem(FStructuredArchive::FSlot, value, default)
              --&gt; Slot &lt;&lt; value (operator&lt;&lt; 分派到格式化器)
                  --&gt; FStructuredArchiveFormatter::Serialize(T&amp;)
                      --&gt; FArchive::operator&lt;&lt;(T&amp;)  （原始类型）
                      --&gt; FArchive::Serialize(void*, int64)  （原始字节）
                          --&gt; FLinkerSave::Serialize（写入文件）

加载路径：
FLinkerLoad（打开文件，读取 FPackageFileSummary）
  --&gt; 创建包装 loader 的 FStructuredArchive
  --&gt; 对每个导出：UObject::Serialize(FStructuredArchive::FRecord)
      --&gt; SerializeScriptProperties
          --&gt; UClass::SerializeTaggedProperties / 无版本属性序列化
              --&gt; FProperty::SerializeItem
                  --&gt; FastPathLoad&lt;Size&gt;（如果 DEVIRTUALIZE_FLinkerLoad_Serialize）
                  --&gt; FArchive::Serialize（虚函数，从文件读取）

事务路径：
  FDuplicateDataWriter / FObjectWriter（序列化到内存）
  FDuplicateDataReader / FObjectReader（从内存恢复，重新映射对象引用）
```

---

## 5. 包与资产

### 5.1 UPackage 类

位于 `CoreUObject/Public/UObject/Package.h`。UPackage 位于每个 UObject outer 链的根部。它派生自 UObject，代表磁盘上的一个独立文件（`.uasset`、`.umap` 等）。

**核心特征**：
- **Outer 链根**：每个对象都有一个 `Outer`。遍历 `GetOutermost()` 链总是终止于 UPackage
- **脏标志**：`bDirty` 跟踪包是否已修改并需要保存。`SetDirtyFlag()` 广播委托通知编辑器
- **加载状态**：`bHasBeenFullyLoaded` 指示所有导出是否已在某个时刻创建
- **包标志**：`PKG_ContainsMap`（.umap）、`PKG_CompiledIn`（引擎内建）、`PKG_NotExternallyReferenceable`
- **块 ID**：`ChunkIDs` 数组在熟化期间填充，流式安装器使用
- **PersistentGuid**：跨保存持久化的稳定 GUID（UE 5.4+ 使用 `SavedHash` (FIoHash)）
- **与 Linker 的关联**：持有指向其 FLinkerLoad 的指针（如果从磁盘加载），通过 `AdditionalInfo->LinkerLoad`
- **MetaData**：仅编辑器，与包关联的 UMetaData 对象

### 5.2 .uasset 与 .umap 文件

`.uasset` 文件是 UPackage 在磁盘上的序列化形式。`.umap` 是相同的格式，但持有 UWorld/ULevel（由 `PKG_ContainsMap` 指示）。

### 5.3 FPackageIndex — 导入/导出的索引编码

位于 `CoreUObject/Public/UObject/ObjectResource.h`。一种巧妙的编码：
- **值 > 0**：索引到 ExportMap（索引 = 值 - 1）
- **值 < 0**：索引到 ImportMap（索引 = -(值) - 1）
- **值 = 0**：null

### 5.4 FObjectImport（外部引用）

```cpp
struct FObjectImport {
    FName ObjectName;             // 对象名称
    FPackageIndex OuterIndex;     // Outer 在 ImportMap/ExportMap 中的索引
    FName ClassPackage;           // 包含此对象类的包
    FName ClassName;              // 此对象的类
    FName PackageName;            // 包含此对象的包（仅编辑器）
    int32 SourceIndex;            // 在源 linker ExportMap 中的索引（瞬态）
    UObject* XObject;             // 加载的 UObject（瞬态）
    FLinkerLoad* SourceLinker;    // 拥有导出的 linker（瞬态）
};
```

### 5.5 FObjectExport（包内对象）

```cpp
struct FObjectExport {
    FName ObjectName;             // 对象名称
    FPackageIndex OuterIndex;     // Outer（父对象）索引
    FPackageIndex ClassIndex;     // 类定义索引
    FPackageIndex SuperIndex;     // UStructs 的父结构体
    FPackageIndex TemplateIndex;  // 原型引用（熟化加载器）
    EObjectFlags ObjectFlags;     // RF_Load 标志掩码
    int64 SerialSize;             // 序列化数据大小
    int64 SerialOffset;           // 序列化数据的文件偏移
    UObject* Object;              // 加载的 UObject（瞬态）
    bool bIsAsset;                // 此对象是否为资产
    // 预加载依赖索引等
};
```

### 5.6 名称表与导入/导出表

名称表存储 `FNameEntryId` 值而非完整的 FName。实际的名称字符串由全局 FName 系统管理。在加载期间，`FLinkerLoad::operator<<(FName&)` 读取名称索引和实例号，然后创建引用名称表中查找的显示 ID 的 FName。

### 5.7 包加载流程（.uasset → 内存）

1. 按名称请求包（如 `/Game/MyAsset`）
2. `GetPackageLinker()` 创建 `FLinkerLoad`（或查找现有的）
3. `FLinkerLoad::Tick()` 被反复调用（异步加载），每次 tick 推进：
   - 打开文件（通过 PackageResourceManager）
   - `SerializePackageFileSummary()` — 验证魔数，读取版本、偏移
   - `SerializeNameMap()` — 读取名称表
   - `SerializeImportMap()` — 读取所有外部引用
   - `SerializeExportMap()` — 读取所有对象定义
   - `SerializeDependsMap()` — 读取依赖信息
   - `CreateExportHash()` — 构建查找表
   - `FindExistingExports()` — 与已加载对象匹配
   - `FixupImportMap()` — 应用重定向
4. 需要对象时：`CreateExport()` 实例化 UObject，`Preload()` 序列化其属性数据
5. `VerifyImport()` 解析每个导入，必要时跟踪重定向器
6. UPackage 被标记为 `bHasBeenFullyLoaded = true`

### 5.8 包保存流程（内存 → .uasset）

1. 调用 `UPackage::Save()` / `SavePackage()`
2. 构造 `FLinkerSave`，创建文件写入器（`Saver`）
3. 从根资产遍历对象图标记导出
4. 序列化期间，每个 UObject 引用被映射：同包内对象变为导出（FObjectExport），其他包对象变为导入（FObjectImport）
5. 构建 NameMap、ImportMap、ExportMap、DependsMap
6. 使用 FArchiveUObject 序列化写入对象数据
7. 最后写入（或更新）FPackageFileSummary
8. 熟化包：剥离仅编辑器数据，批量数据可能分离到 `.ubulk` 等文件

### 5.9 资产注册表系统

**关键文件**：
- `AssetRegistry/Public/AssetRegistry/IAssetRegistry.h` — 公共接口
- `AssetRegistry/Private/AssetRegistryImpl.h` — 实现
- `AssetRegistry/Private/AssetDataGatherer.h` — 数据收集器

#### IAssetRegistry 公共接口

全局单例（`IAssetRegistry::Get()`）提供：
- **查询**：`GetAssets()`、`GetAssetsByClass()`、`GetAllAssets()`、`EnumerateAllAssets()` — 使用 `FARFilter` 进行复杂过滤
- **依赖**：`GetDependencies()` / `GetReferencers()` — 返回硬引用/软引用/管理引用
- **路径**：`GetAllCachedPaths()`、`GetSubPaths()`、`PathExists()`
- **包数据**：`TryGetAssetPackageData()` 返回文件大小、版本、自定义版本、熟化哈希
- **扫描**：`SearchAllAssets()`、`ScanPathsSynchronous()`、`WaitForCompletion()`
- **重定向器**：`GetRedirectedObjectPath()` 跟踪重定向器链
- **类继承**：`GetAncestorClassNames()`、`GetDerivedClassNames()`

#### FAssetData 核心结构

```cpp
struct FAssetData {
    FName PackageName;           // 长包名（如 /Game/Path/Package）
    FName PackagePath;           // 文件夹路径（如 /Game/Path）
    FName AssetName;             // 包内对象名
    FTopLevelAssetPath AssetClassPath;  // 类路径（如 /Script/Engine.StaticMesh）
    FAssetDataTagMapSharedView TagsAndValues;  // 键值元数据
    FAssetBundleData TaggedAssetBundles; // 捆绑加载
    uint32 PackageFlags;
    TArray<int32> ChunkIDs;
};
```

`FAssetData` 可以从 UObject 构造、从显式字段构造或从缓存反序列化。`IsUAsset()` 检查它是否是其包中的"主"资产。`IsRedirector()` 检查类是否为 `UObjectRedirector`。

#### 资产发现管道

1. **`FAssetDataGatherer`**（后台线程）使用 `FAssetDataDiscovery` 通过目录遍历发现磁盘文件
2. **`FPackageReader`** 打开每个 `.uasset`/`.umap`，读取 FPackageFileSummary、NameMap、ImportMap、ExportMap，**不创建完整的 UObject 实例**（轻量级解析）
3. 组装的 FAssetData 和依赖数据提交到 **`FAssetRegistryState`**，存储所有已知资产的内存数据库
4. 熟化构建中，AssetRegistryState 序列化为单个二进制缓存文件，避免运行时的逐文件解析
5. 编辑器中，收集器监控文件更改（`FFileChangeData`）进行增量更新

#### FAssetPackageData

存储每个包的元数据：`CookedHash`（MD5）、`ChunkHashes`（IoStore 的逐块哈希）、`ImportedClasses`（导出使用的类）、`DiskSize`、`FileVersionUE`、`CustomVersions`、`Extension`。

### 5.10 AssetManager 和主资产类型

位于 `Engine/Classes/Engine/AssetManager.h`。UAssetManager 是负责按游戏特定类型组织资产的单例 UObject。

- **PrimaryAssetType**（`CoreUObject/Public/UObject/PrimaryAssetId.h`）：基于 FName 的标签，如 `Map`、`PrimaryAssetLabel`。每个主资产有 `FPrimaryAssetId = (Type, Name)`
- **主资产扫描**：`ScanPathsForPrimaryAssets()` 发现指定路径下的特定类型资产
- **基于捆绑的加载**：资产声明带命名捆绑的 `FAssetBundleData`。`LoadPrimaryAsset()` 只加载所需引用的子集
- **异步加载**：内部使用 `FStreamableManager`，返回 `TSharedPtr<FStreamableHandle>` 用于跟踪
- **块管理**：`PackageChunkType` 虚拟资产代表安装块。`FindMissingChunkList()` 确定需要安装哪些块
- **规则和覆盖**：`FPrimaryAssetRules` 控制熟化/加载行为

### 5.11 流式安装 / 分块

分块系统在熟化期间将每个包分配到一个或多个整数块 ID（存储在 `UPackage::ChunkIDs` 和 `FPackageFileSummary::ChunkIDs` 中）。平台块安装器（如 `FGenericPlatformChunkInstall`）使用这些 ID 确定在资产可用之前必须下载哪些内容。

`EAssetAvailability::Type` 返回值：DoesNotExist、NotAvailable、LocalSlow、LocalFast。`IAssetRegistry::GetAssetAvailability()` / `PrioritizeAssetInstall()` 与安装器交互。

### 5.12 重定向器（Redirectors）

位于 `CoreUObject/Public/UObject/ObjectRedirector.h`。当资产在编辑器中重命名或移动时，`UObjectRedirector` 被留在旧位置。其唯一目的：只有一个字段 `DestinationObject`，指向新位置。

- **加载**：当 `FLinkerLoad::VerifyImport()` 找到 UObjectRedirector 时，返回 `VERIFY_Redirected`，调用者跟踪重定向到实际对象
- **资产注册表**：重定向器在注册表中可见（`FAssetData::IsRedirector()`），可以在不加载的情况下解析它们
- **PreSave**：保存包时，重定向器被清理（替换为直接引用）
- **CoreRedirects**：单独的系统（`FCoreRedirects`）处理类/结构体/枚举重命名，补充逐包 UObjectRedirector

### 5.13 完整数据流图

```
包加载（.uasset → 内存）：
  请求包 "/Game/MyAsset"
    -> GetPackageLinker() 创建 FLinkerLoad
    -> FLinkerLoad::Tick() 增量执行：
         -> 打开文件 -> 序列化文件摘要 -> 序列化名称表
         -> 序列化导入表 -> 序列化导出表 -> 序列化依赖表
    -> CreateExport() 实例化 UObject
    -> Preload() 序列化属性数据
    -> VerifyImport() 解析外部引用，跟踪重定向器
    -> UPackage 标记为 bHasBeenFullyLoaded = true

包保存（内存 → .uasset）：
  UPackage::Save() / SavePackage()
    -> 创建 FLinkerSave（打开文件写入器）
    -> 遍历对象图标记导出
    -> 序列化：同包对象 → 导出；外部对象 → 导入
    -> 构建 NameMap、ImportMap、ExportMap、DependsMap
    -> 使用 FArchiveUObject 写入对象数据
    -> 构建 PackageTrailer，写入 FPackageFileSummary

资产注册表发现：
  FAssetDataGatherer（后台线程）
    -> 目录遍历发现 .uasset/.umap
    -> FPackageReader 轻量级解析（仅读取表头，不创建 UObject）
    -> FAssetData + 依赖数据提交到 FAssetRegistryState
    -> IAssetRegistry 提供查询 API
    -> 编辑器：FFileChangeData 触发增量重扫描
    -> 熟化时：状态序列化为预构建缓存
```

---

## 参考资料

本文档基于以下 UE5 引擎源代码文件编写（分支 5.5）：

| 模块 | 关键文件 |
|------|---------|
| 反射系统 | `CoreUObject/Public/UObject/Class.h`, `CoreUObject/Public/UObject/Field.h`, `CoreUObject/Public/UObject/UnrealType.h`, `CoreUObject/Public/UObject/ObjectMacros.h`, `CoreUObject/Public/UObject/MetaData.h` |
| 对象系统 | `CoreUObject/Public/UObject/Object.h`, `CoreUObject/Public/UObject/UObjectBase.h`, `CoreUObject/Public/UObject/UObjectBaseUtility.h`, `CoreUObject/Private/UObject/Obj.cpp`, `CoreUObject/Private/UObject/UObjectGlobals.cpp` |
| 垃圾回收 | `CoreUObject/Public/UObject/GarbageCollection.h`, `CoreUObject/Public/UObject/UObjectArray.h`, `CoreUObject/Public/UObject/GarbageCollectionSchema.h`, `CoreUObject/Public/UObject/FastReferenceCollector.h`, `CoreUObject/Private/UObject/GarbageCollection.cpp` |
| 序列化 | `Core/Public/Serialization/Archive.h`, `Core/Public/Serialization/StructuredArchive.h`, `CoreUObject/Public/Serialization/ArchiveUObject.h`, `CoreUObject/Public/UObject/LinkerLoad.h`, `CoreUObject/Public/UObject/LinkerSave.h` |
| 包与资产 | `CoreUObject/Public/UObject/Package.h`, `CoreUObject/Public/UObject/Linker.h`, `CoreUObject/Public/UObject/ObjectResource.h`, `CoreUObject/Public/UObject/ObjectRedirector.h`, `AssetRegistry/Public/AssetRegistry/IAssetRegistry.h` |
| UHT | `Programs/Shared/EpicGames.UHT/Types/`, `Programs/Shared/EpicGames.UHT/Exporters/CodeGen/`, `Programs/Shared/EpicGames.UHT/Parsers/` |
