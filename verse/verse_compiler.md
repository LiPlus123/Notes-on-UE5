# Verse 编译器前端

> 本文解析 UE6 中 Verse 语言编译器（内部称 **uLang**）的前端：从源码文本到语义程序（`CSemanticProgram`）的完整过程。
>
> 代码位置：`Engine/Source/Runtime/VerseCompiler/`，命名空间 `uLang`，头文件统一以 `uLang/` 前缀包含。
>
> Verse 由早期的 **Solaris** 语言演化而来，因此源码中大量遗留 `Sol` / `Solaris` / `VerseFN`（Fortnite 上传版本）等命名。

---

## 1. 编译器整体架构

Verse 编译器是一个**分层、多阶段、可重定向（retargetable）**的编译器 API。它的核心思想是：**前端与后端都可替换**，每个阶段由 `ModularFeature` 机制动态装配，从而允许外部注入自定义的优化 pass 或字节码打包器。

流水线由 `SToolchainParams`（`Public/uLang/Toolchain/Toolchain.h`）描述，各阶段：

| 阶段 | 接口 | 输入 → 输出 |
|------|------|------------|
| **Parser**（语法分析） | `IParserPass` | 源码文本 → VST（Verse 语法树） |
| Post Vst Filters | `IPostVstFilter[]` | VST → VST（优化/去元数据） |
| **Semantic Analyzer**（语义分析） | `ISemanticAnalyzerPass` | VST → AST，做类型检查，产出语义程序 |
| Post Semantic Analysis Filters | `IPostSemanticAnalysisFilter[]` | AST → AST |
| **IR Generator**（IR 生成） | `IIrGeneratorPass` | AST → IR |
| Post IR Filters | `IPostIrFilter[]` | IR → IR |
| **Assembler**（汇编/链接） | `IAssemblerPass` | IR → 字节码，符号链接 |
| Linker | `Link()` | （当前未使用） |

> 官方注释称其为 “five stages of compilation”：`Parser`、`Semantic Analyzer`、`IR Generator`、`Assembler`、`Linker`。此外还有贯穿全程的 **Injection**（`IPreParseInjection` / `IPostSemAnalysisInjection` / `IPreTranslateInjection` 等），用于在阶段前后注入定制逻辑（见 `CompilerPasses/CompilerTypes.h` 的 `SToolchainInjections`）。

### 前端与后端的边界

- **前端** = `Parser` + `Semantic Analyzer`（+ `SourceProject` 源码加载 + `Desugarer` 去糖），产出 **语义化 AST**（`CSemanticProgram`）。
- **后端** = `IR Generator` + `Assembler`（IR 生成与字节码汇编/链接）。

本文聚焦前端，即下图左侧部分：

```
源码文本 ──► Parser ──► VST ──► [PostVstFilters] ──► SemanticAnalyzer ──► CSemanticProgram
                                                         │（内含 Desugarer）
                                                         └── VST → AST → 类型检查/标注
```

### 编译结果标志

`ECompilerResult`（位标志）区分了各阶段是否运行/失败：`Compile_RanSyntaxPass` / `Compile_RanSemanticPass` / `Compile_RanLocalizationPass` / `Compile_RanIrPass` / `Compile_RanCodeGenPass`，以及对应的 `Compile_SyntaxError` / `Compile_SemanticError` 等失败标志。

### 核心入口 `CToolchain`

`Public/uLang/Toolchain/Toolchain.h` 中的 `CToolchain` 暴露了分步编译 API，可按需单独运行某一阶段：

- `BuildProject(...)` — 完整编译 + 链接项目中的所有源码片段
- `ParseSnippet(...)` — 仅解析一个文本片段 → `Vst::Snippet`
- `CompileVst(...)` — 对 VST 执行 `SemanticAnalyzeVst` + `AssembleProgram`
- `SemanticAnalyzeVst(...)` — 仅语义分析 → `CSemanticProgram`
- `ExtractLocalization(...)` — 提取本地化信息
- `IrGenerateProgram(...)` — 仅 IR 生成
- `AssembleProgram(...)` — 语义分析 + 代码生成

---

## 2. 目录结构

```
Engine/Source/Runtime/VerseCompiler/
├── Public/uLang/                  # 公开头文件
│   ├── CompilerPasses/            # 各 pass 接口（IParserPass、ISemanticAnalyzerPass…）
│   ├── Diagnostics/               # 诊断/Glitch 模型
│   ├── Parser/                    # VerseGrammar.h（语法库）、ParserPass.h
│   ├── SemanticAnalyzer/          # SemanticAnalyzer.h、DigestGenerator.h、IRGenerator.h
│   ├── Semantics/                 # 语义 AST：Definition.h、Expression.h、SemanticTypes.h…
│   ├── SourceProject/             # 源码工程模型：SourceProject.h、VerseVersion.h…
│   ├── Syntax/                    # VST 节点：VstNode.h、NodeDecls.inl
│   └── Toolchain/                 # Toolchain.h、ModularFeature.h…
└── Private/uLang/                 # 实现
    ├── Parser/                    # ParserPass.cpp、ReservedSymbols.cpp
    ├── SemanticAnalyzer/          # Desugarer.cpp、SemanticAnalyzer.cpp、DigestGenerator.cpp
    ├── Semantics/                 # 各语义节点实现
    ├── SourceProject/             # 源码索引、依赖解析
    ├── Syntax/                    # VstPrint.cpp（VST 打印）、tLang.cpp
    └── Toolchain/                 # Toolchain.cpp、ProgramBuildManager.cpp…
```

---

## 3. 源码加载（SourceProject）

编译的输入是 `CSourceProject` —— 一组 `CSourcePackage`（源码包）。相关文件：

- `SourceProject/SourceProject.h` — `CSourceProject` / `CSourcePackage`
- `SourceProject/IndexedSourceText.h/.cpp` — 索引化源码文本（建立行列号 ↔ 字符偏移的映射，供 Locus 定位用）
- `SourceProject/SourceFileProject.h` — 从磁盘文件构建工程
- `SourceProject/SourceProjectWriter.h/.cpp` — 序列化工程
- `SourceProject/IFileSystem.h` — 抽象文件系统接口
- `SourceProject/PackageRole.h` / `VerseScope.h` / `VerseVersion.h` / `UploadedAtFNVersion.h` — 包角色、可见性作用域、语言版本号、FN 上传版本（用于向后兼容的 hack）

`CToolchain::BuildProject` 会按**依赖深度对包排序**（`BuildOrderedPackageList`），再逐个包填入 VST 工程。

---

## 4. 词法 / 语法分析（Parser）

Verse 的解析器是**无独立词法器（scannerless）**的：词法识别与语法分析在同一个递归下降框架中完成。它没有独立的 token 流，而是在解析过程中按需“切词”。

### 4.1 语法库 `VerseGrammar.h`

`Public/uLang/Parser/VerseGrammar.h` 是一个 **“依赖无关、零分配的单词法头文件 Verse 语法库”**（约 4400 行），基于 **Pratt / 递归下降 + 优先级爬升（precedence climbing）** 实现，而非传统的 yacc/lemon 生成表。

关键要素：

- **优先级枚举 `EPrec`**：`Never < List < Commas < Expr < Fun < Def < Or < And < Not < Eq < NotEq < Less < Greater < Choose < To < Add < Mul < Prefix < Call < Base < Nothing`
- **结合性 `EAssoc`**：`None / Postfix / InfixLeft / InfixRight`
- **块形态 `EForm`**：`List / Commas`（决定块内元素分隔方式）
- **块标点 `EPunctuation`**：`None / Braces / Parens / Brackets / AngleBrackets / Qualifier / Dot / Colon / Ind`（Verse 的 `()` `[]` `{}` `<>` 与缩进块）
- **捕获位置 `EPlace`**：区分 `UTF8 / Printable / BlockCmt / LineCmt / IndCmt / Space / String / Content` 等不同“文本捕获”语义
- **组合子宏**：`ULANG_GRAMMAR_RUN/SET/LET`（类似 let-binding 的短路错误传播）
- **深度上限**：`VERSE_MAX_EXPR_DEPTH 100`、`VERSE_MAX_INDCMT_DEPTH 3`

核心抽象：`SToken`（token）、`SSnippet`（源码片段 + 位置）、`SBlock`（一个含标点/形态/元素的块）、`TResult<Syntax, Error>`（解析结果）、`TGenerate<T>`（调用者实现的回调基类）。

### 4.2 `SGenerateVst` —— 从语法动作构建 VST

`Private/uLang/Parser/ParserPass.cpp` 中的 `struct SGenerateVst : Verse::Grammar::TGenerate<SGenerateCommon>` 是解析器的**语义动作**集合，语法库在归约时回调这些方法，把源码片段转成 VST 节点。主要回调：

| 回调 | 作用 |
|------|------|
| `File` | 顶层入口，把块元素组装成 `Clause` |
| `Num` / `NumHex` / `Units` | 数值字面量（整数 / `0x` 十六进制 / `f16/f32/f64` 浮点后缀） |
| `Char8` / `Char32` | 字符字面量（区分 UTF8 code unit / Unicode scalar / 转义） |
| `StringLiteral` / `String` / `StringInterpolate` | 字符串与插值字符串 `"…{expr}…"` |
| `Path` / `Escape` | 路径字面量、转义节点 |
| `Ident` / `QualIdent` | 标识符 / 限定标识符（`A.B`，限定符作为子节点） |
| `PrefixToken` | 前缀运算符（`-` `+` `not` `set` `var` `live` `return` `break` …） |
| `InfixToken` | 中缀运算符（`=` `<` `+` `*` `.` `and` `or` `:` `..` `->` …） |
| `PostfixToken` | 后缀运算符（`?` 失败查询、`^` 指针解引用） |
| `Call` / `Invoke` | 函数调用 `F(x)`、可失败调用 `F[x]`、宏/关键字式调用 `if … then … else` |
| `PrefixBrackets` | `[key]value` 映射类型 / `[]element` 数组类型 |
| `InfixBlock` / `DefineFromType` | 定义（`:=` `=` `=>` `where`）与带类型标注的定义 `X:t=V` |
| `PrefixAttribute` | 前置属性 `@attr` |
| `LineCmt` / `BlockCmt` / `IndCmt` | 注释捕获（行 / 块 / 缩进注释） |

**解析器即做了一部分“语法去糖”**，在构建 VST 时就完成了若干重写，例如：

- `else if` 扁平化为多 `then` 子句的单个 `FlowIf`（`if` 回调内）
- `a.b.c` 成员访问链变换为扁平的 `PrePostCall`（dot 链）
- `X:t=V` 解析成 `(X):=((:t)=V)` 后重排为 `(X:t):=(V)`（`DefineFromType`）
- `=>` 生成 `Lambda`，`where` 生成 `Where` 节点

### 4.3 旧扫描器桥接 `vsyntax`

`Public/uLang/Syntax/vsyntax_types.h` 是遗留的**旧解析器**桥接层，定义了保留字枚举 `res_t`（`of/if/else/upon/where/catch/do/then/until/return/yield/break/continue/at/var/set/and/or/not`）与 `scan_reserved_t` 字符串表。`SGenerateVst::TokenToRes` 把新解析器的块标点映射回旧的保留字枚举，用于兼容旧代码。

### 4.4 入口 `CParserPass`

`CParserPass::ProcessSnippet`（`Public/uLang/Parser/ParserPass.h`）实现 `IParserPass` 接口，输入一段 UTF8 文本与版本号，输出 `Vst::Snippet`。解析出的 glitch 直接写入 `SBuildContext::_Diagnostics`。

---

## 5. VST —— Verse 语法树

VST（Verse Syntax Tree）是解析的直接产物，定义在 `Public/uLang/Syntax/VstNode.h` 与 `NodeDecls.inl`。

- 节点统一继承 `Verse::Vst::Node`，用**引用计数智能指针** `TNodeRef<T>` / `TNodePtr<T>` 管理。
- 每个节点带 `SLocus`（源码位置）、`_Children`（子节点）、`_Aux`（辅助子句，如属性/说明符）、前/后置注释、前后换行数，以及 `_MappedAstNode`（到语义 AST 节点的反向指针，供去糖时映射）。
- 节点类型用 `NodeType` 枚举 + 静态 `NodeInfos[]` 表描述：**必需子节点数、优先级、是否支持多子节点、子节点删除行为、是否原子节点（CAtom）**。

`NodeDecls.inl` 中的 `VERSE_ENUM_VSTNODES` 枚举全部 40 种节点（按优先级递增排列）：

```
Project, Package, Module, Snippet              # 结构容器
Where                                          # where 约束
Assignment, Definition                         # 赋值 / 定义
TypeSpec                                       # 类型标注（X:t）
BinaryOpLogicalOr / BinaryOpLogicalAnd         # or / and
PrefixOpLogicalNot                             # not
BinaryOpCompare                                # = <> < <= > >=
BinaryOpArrow                                  # ->
BinaryOpAddSub                                  # + -
BinaryOpMulDivInfix                             # * /
BinaryOpRange                                   # ..
PrePostCall                                     # 调用/成员访问/查询/解引用（核心节点）
Identifier, Operator                           # 原子节点
FlowIf                                          # if / then / else
IntLiteral, FloatLiteral, CharLiteral          # 字面量
StringLiteral, PathLiteral
Interpolant, InterpolatedString                # 字符串插值
Lambda                                          # 匿名函数
Control                                         # return / break / continue
Macro, Clause, Parens, Commas                  # 宏调用 / 子句 / 括号 / 逗号列表
Placeholder, ParseError                         # 错误占位
Escape, Comment, Mutation                       # 转义 / 注释 / set·var·live
```

其中 `PrePostCall` 是 Verse 表达式的“瑞士军刀”：统一承载 `F(x)` 调用、`a.b` 成员访问、`F[x]` 可失败调用、`F?` 查询、`F^` 解引用、`[]T` 类型说明等，通过前置/后置子句（qmark/hat/dot/args）区分。

VST 可被 `VstPrint.cpp`（`PrettyPrintVst`）**精确回环（roundtrip）打印**回等价源码，这也是解析器大量处理注释/换行归属的原因。

---

## 6. 去糖（Desugarer）：VST → AST

`Private/uLang/SemanticAnalyzer/Desugarer.cpp` 的 `DesugarVstToAst(VstProject, Symbols, Diagnostics)` 是语义分析的**第一步**：把 VST 转换为语义 AST（`CAstProject`）。

`CDesugarerImpl::DesugarProject` 的职责：

1. **逐包去糖**：把每个 `Vst::Package` 转成 `CAstPackage`，建立 VST 节点 ↔ AST 节点的映射（`GetMappedVstNode`）。
2. **构建包依赖图**：读取 `Vst::Package::_DependencyPackages`，缺失依赖报 `ErrSemantic_UnknownPackageDependency`。
3. **Tarjan SCC 算法**：把包依赖图分解为**强连通分量**，每个 SCC 作为一个**编译单元（compilation unit）** —— 这是处理循环依赖（如模块互引用）的关键。
4. 此外还包含 `DesugarControl` 等：对 `return/break/continue` 等控制流节点去糖。

去糖后的 `CAstProject` 存入 `CSemanticProgram::_AstProject`。

---

## 7. 语义分析（SemanticAnalyzer）

`Private/uLang/SemanticAnalyzer/SemanticAnalyzer.cpp` 是前端最大的模块（约 23000 行）。`CSemanticAnalyzer` 的核心是一个 **`CSemanticAnalyzerImpl`**，用**延迟优先级队列（deferred priority queue）**驱动：把待分析的定义按 `EDeferredPri` 优先级排队处理，从而支持**前向引用与相互递归**（先收集声明，再按依赖顺序逐个类型化）。

`CSemanticAnalyzer::ProcessVst(Vst, ESemanticPass)` 按 `ESemanticPass` 分派三个阶段：

### 阶段一：`SemanticPass_Types`（类型声明与解析）

1. `DesugarVstTopLevel` — 去糖（见上）
2. `AnalyzeAstTopLevel` — 顶层分析
3. `Deferred_Module` — 处理模块
4. `LinkCompatConstraints` — 链接兼容性约束
5. `Deferred_Import` — 处理 import
6. `Deferred_ModuleReferences` — 模块引用
7. `Deferred_AccessSpecifiers` — 访问说明符（`public/private/protected/internal…`）
8. `Deferred_Type` — 类型解析
9. `Deferred_ValidateCycles` — 循环校验
10. `Deferred_ClosedFunctionBodyExpressions` — 封闭函数体表达式
11. `LinkOverrides` — 链接 override 关系
12. `Deferred_ValidateType` — 类型校验
13. `SetAllNonLocalDefinitionsCreated` — 标记所有非局部定义已建立（之后可安全缓存定义路径解析）

### 阶段二：`SemanticPass_Attributes`（属性处理）

1. `Deferred_AttributeClassAttributes`
2. `Deferred_Attributes`
3. `Deferred_PropagateAttributes`
4. `Deferred_ValidateAttributes`

### 阶段三：`SemanticPass_Code`（代码体类型检查）

1. `Deferred_NonFunctionExpressions` — 非函数表达式
2. `Deferred_OpenFunctionBodyExpressions` — 开放（可失败）函数体表达式
3. `Deferred_FinalValidation` — 最终校验
4. `AnalyzeCompatConstraints` — 兼容性约束分析

三个阶段依次执行（由 `CToolchain::SemanticAnalyzeVst` 驱动），全部完成后得到一份**已标注类型、已解析引用、已检查效应**的 `CSemanticProgram`。

---

## 8. 语义模型（Semantics）

`Public/uLang/Semantics/` 定义了语义 AST 的节点与类型系统。这是前端最重要的“产出物”。

### 8.1 定义层级 `CDefinition`

`Semantics/Definition.h` 中 `CDefinition` 是所有作用域定义的基类（继承 `CNamed` + `TAstNodeRef<CExpressionBase>` + `CSharedMix`），持有所属作用域、覆盖关系（`_OverriddenDefinition`）、限定符（`SQualifier`）等。

`VERSE_ENUM_DEFINITION_KINDS` 枚举 11 种定义：

| Kind | 含义 | 对应文件 |
|------|------|---------|
| `Class` | 类 / 结构体 / 接口 | `SemanticClass.h` |
| `Data` | 数据成员/字段 | `DataDefinition.h` |
| `Enumeration` | 枚举类型 | `SemanticEnumeration.h` |
| `Enumerator` | 枚举值 | `SemanticEnumeration.h` |
| `Function` | 函数/方法 | `SemanticFunction.h` |
| `Module` | 模块 | `SemanticProgram.h`（`CModule`） |
| `ModuleAlias` | 模块别名（import） | `ModuleAlias.h` |
| `TypeAlias` | 类型别名 | `TypeAlias.h` |
| `TypeVariable` | 类型变量（泛型参数） | `TypeVariable.h` |
| `Union` | 联合类型 | `SemanticUnion.h` |
| `UnionVariant` | 联合成员 | `SemanticUnion.h` |

### 8.2 表达式 `CExpressionBase`

`Semantics/Expression.h` 定义语义表达式层级（`CExpr*`），与 VST 的 `BinaryOp*` / `PrePostCall` 对应但更偏语义化：`CExprBinaryArithmetic`、`CExprComparison`、`CExprAssignment`、`CExprCall`、`CExprIdentifier`、`CExprLambda`、`CExprModuleDefinition` 等。

### 8.3 类型系统 `CTypeBase`

`Semantics/SemanticTypes.h` 是类型系统核心。`CSemanticProgram`（`SemanticProgram.h`）持有全局单例类型与类型工厂：

- **内建标量类型**：`_intType`（`CIntType`）、`_floatType`（`CFloatType`）、`_stringAlias`、`_rationalType`、`_char8Type`、`_char32Type`、`_logicType`、`_pathType`、`_rangeType`…
- **特殊类型**：`_falseType` / `_trueType` / `_voidType` / `_anyType` / `_comparableType` / `_persistableType` / `_objectType` / `_typeType`…
- **容器/复合类型工厂**（hash-consed 缓存）：`GetOrCreateArrayType` / `GetOrCreateMapType` / `GetOrCreateOptionType` / `GetOrCreateTupleType` / `GetOrCreatePointerType` / `GetOrCreateFunctionType` / `GetOrCreateConstrainedIntType`…
- **流类型 `CFlowType`**：Verse 特有的**成败流**类型（`ETypePolarity::Positive/Negative`），建模函数可能失败/成功的控制流。
- **`CUnknownType`**：类型推断失败时的兜底类型，保证错误恢复。

### 8.4 效应系统（Effects）

Verse 的**效应（effect）**是类型系统的关键部分（`Semantics/Effects.h`）。`CSemanticProgram` 中缓存了效应类：`suspends`（挂起/协程）、`decides`（失败）、`computes`（纯）、`converges`（保证收敛）、`transacts`（事务 = computes+reads+writes+allocates）、`reads/writes/allocates`（指针读/写/分配）、`varies` 等。

`SEffectSet` 表示一组效应，`FindEffectDescriptor` / `ConvertEffectClassesToEffectSet` 等负责效应集合的合并、分解与校验。效应推断是语义分析阶段三（`Code`）的核心工作之一。

### 8.5 属性（Attributes）

Verse 用属性注解定义，如 `<abstract>` `<final>` `<concrete>` `<intrinsic>` `<native>` `<public>` `<private>` `<deprecated>` `<available>` 等。`CSemanticProgram` 中为每个属性预创建对应的 `CClass`（`_abstractClass`、`_finalClass`、`_publicClass`、`_deprecatedClass`…），属性的作用域限制由 `@attribscope_*` 属性类描述（`_attributeScopeClass`、`_attributeScopeFunction`…）。

相关文件：`Semantics/Attributable.h`、`Semantics/AvailableAttributeUtils.h`、`Semantics/DeprecatedAttributeUtils.h`、`Semantics/AccessLevel.h`、`Semantics/AccessibilityScope.h`。

### 8.6 符号表

`CSemanticProgram::_Symbols` 是程序级符号表（`CSymbolTable`，`Common/Text/Symbol.h`），所有符号（标识符名）intern 到同一张表，用 `CSymbol` 指针比较。`CIntrinsicSymbols`（`SemanticProgram.h`）预注册内建运算符名（如 `_OpNameAdd`、`_OpNameCall`、`_OpNameQuery`）。

---

## 9. 诊断（Diagnostics / Glitch）

- `Public/uLang/Diagnostics/Diagnostics.h` — `CDiagnostics` 累积整个编译过程的 glitch。
- `Public/uLang/Diagnostics/Glitch.h` — `SGlitch` 是诊断单元，含 `SGlitchLocus`（文件 + `SLocus` 位置）与 `SGlitchResult`（级别 + 消息）。
- 诊断级别：`Warning`（2000–3000 段）与 `Error`。
- 诊断 id 用枚举 `EDiagnostic` 统一编号，消息前缀 `vErr:`（见 `SGenerateVst::Err`）。
- 前端相关类别：`ErrSyntax_*`（语法错误）、`ErrSemantic_*`（语义错误）、`WarnParser_*` / `WarnSemantic_*`（解析/语义警告）。

---

## 10. Digest 生成（DigestGenerator）

`Private/uLang/SemanticAnalyzer/DigestGenerator.cpp` 生成**文本摘要（digest）**：对语义化后的某个包，把其公共定义（可按需包含 `internal` / `epic_internal`）**回写**成一份规范化的 Verse 源码文本（`OutDigestCode`）。

用途：**增量编译 / 缓存**。通过对比包的 digest 与依赖清单（`OutDigestPackageDependencies`）判断某个包（及其下游）是否需要重编译，避免对未变更的包重复做完整语义分析。这也是 Verse 支持海量 UGC 代码（Fortnite）快速编译的关键。

---

## 11. 其他前端辅助组件

- **`QualifierUtils.cpp`**（`Semantics/QualifierUtils.h`）：`QualifyAllAnalyzedIdentifiers` 把所有已解析的标识符在 VST 中回填限定名（自动加限定前缀）；`FindUnresolvedIdentifiers` / `VerifyAllQualified` 用于校验标识符解析完整性（对应 `SBuildParams::_bQualifyIdentifiers`）。
- **`Effects.cpp` / `Effects.h`**：效应集合运算。
- **`CaptureScope.cpp` / `CaptureControlScope.h`**：捕获作用域、控制流作用域。
- **`AccessLevel.cpp` / `AccessibilityScope.cpp`**：访问级别与可达性。
- **`IntegerLiteralParser.cpp`**：整数/有理数字面量解析（`IntOrInfinity.h`）。
- **`AstMatcher.cpp` / `AstPrinter.cpp`**：AST 匹配与打印（调试/测试用）。
- **`Toolchain/`**：`ModularFeatureManager`（按 feature 装载各 pass）、`ProgramBuildManager`（多包构建编排）、`CommandLine`（命令行解析）、`AvailableAttributeVstFilter`（`@available` 属性过滤）。

---

## 12. 小结

Verse 前端（uLang）的完整数据流：

```
CSourceProject（源码工程）
      │  BuildProject / 依赖排序
      ▼
CParserPass ── VerseGrammar（Pratt 递归下降）──► VST（40 种节点，可 roundtrip 打印）
      │
      ▼
[PostVstFilters]
      │
      ▼
CSemanticAnalyzer
      ├─ Desugarer：VST → CAstProject（Tarjan SCC → 编译单元）
      └─ 三阶段语义 pass（Types → Attributes → Code），延迟优先级队列驱动
      │
      ▼
CSemanticProgram（语义程序）
      ├─ _AstProject（语义 AST）
      ├─ 全局类型 / 内建类型工厂
      ├─ 效应类 / 属性类 / 内建符号
      └─ 诊断（CDiagnostics）
      │
      ▼（后端，本文略）
IRGenerator → Assembler → 字节码 / 链接
```

设计要点：

1. **可重定向**：前端/后端通过 `ModularFeature` 装配，允许外部替换或注入 pass。
2. **scannerless 递归下降解析**：词法语法合一，语法库单头文件、零分配，强调快与可预测。
3. **VST 精确回环**：保留注释/换行，支持无损格式化与自动限定名回填。
4. **延迟优先级队列的语义分析**：多遍（Types/Attributes/Code）+ 优先级队列，优雅处理前向引用、循环依赖与效应推断。
5. **Digest 增量编译**：为大规模 UGC 场景提供缓存与增量重编译能力。
