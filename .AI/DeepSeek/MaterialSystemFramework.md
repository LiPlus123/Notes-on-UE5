# Unreal Engine 材质系统框架

## 1. 材质编辑器

UE5 材质编辑器是一个可视化的节点图编辑器，用户通过连线将 `UMaterialExpression` 节点组合，定义材质表面的各种属性（BaseColor、Roughness、Normal 等）。

核心概念：
- **Material（材质）**：拥有完整节点图的资产，编译后生成 shader。对应 `UMaterial`。
- **Material Instance（材质实例）**：不拥有节点图，只覆盖父材质暴露的参数值。对应 `UMaterialInstanceConstant`（编辑器）/ `UMaterialInstanceDynamic`（运行时）。
- **Material Function**：可复用的材质子图，类似函数调用。

继承关系：

```
UMaterialInterface (抽象基类)
    ├── UMaterial (完整材质，含节点图)
    └── UMaterialInstance (参数覆盖)
            ├── UMaterialInstanceConstant (编辑器资产)
            └── UMaterialInstanceDynamic  (运行时创建)
```

`UMaterialInstance::Parent` 类型为 `UMaterialInterface*`，形成链式结构：`MID → MIC → MIC → UMaterial`。

---

## 2. 核心类型

### UMaterial / UMaterialInstance（Game Thread 资产层）

| 类 | 定义位置 | 作用 |
|----|----------|------|
| `UMaterialInterface` | `MaterialInterface.h` | 所有材质的公共接口，提供 `GetMaterial()`、`GetRenderProxy()` |
| `UMaterial` | `Material.h` | 材质资产，持有 `UMaterialExpression[]` 节点图、`FMaterialResource[]`（编译产物） |
| `UMaterialInstance` | `MaterialInstance.h` | 参数覆盖实例，持有 `Parent` 和各类参数值数组 |
| `UMaterialExpression` | `MaterialExpression.h` | 节点图中的一个节点，实现 `virtual int32 Compile(FMaterialCompiler*)` |

### FMaterial / FMaterialResource（Render Thread 编译层）

| 类 | 定义位置 | 作用 |
|----|----------|------|
| `FMaterial` | `MaterialShared.h` | 材质的渲染线程表示，持有 `FMaterialShaderMap`，提供材质属性查询接口 |
| `FMaterialResource` | `MaterialShared.h` | `FMaterial` 的具体实现，桥接 `UMaterial`/`UMaterialInstance` 到渲染层 |
| `FMaterialRenderProxy` | `MaterialRenderProxy.h` | 渲染线程访问材质参数的代理，uniform buffer 的数据源 |
| `FMaterialShaderMap` | `MaterialShared.h` | 编译后的 shader 集合（per quality level / platform），内含各 pass × VF 组合的 bytecode |

### 关系图

```
UMaterial (GT)
    │
    ├── FMaterialResource[] (per quality × platform)
    │       │
    │       └── FMaterialShaderMap
    │               │
    │               ├── TBasePassVS shader bytecode
    │               ├── TBasePassPS shader bytecode
    │               ├── DepthOnly VS/PS
    │               └── Shadow VS/PS ...
    │
    └── FDefaultMaterialInstance (FMaterialRenderProxy)
            → 渲染线程读取 uniform 参数
```

---

## 3. 材质 Shader 编译

### 编译流水线概览

```
UMaterialExpression 节点图
        │  ← 每个节点 Compile() 递归生成代码块
        ▼
FHLSLMaterialTranslator::Translate()
        │  ← 收集所有代码块到 MaterialSourceTemplateParams
        ▼
MaterialTemplate.ush + 占位符替换
        │  ← GetMaterialShaderCode() 调用 FStringTemplateResolver
        ▼
完整 HLSL 字符串（内存）
        │  ← 注册为虚拟路径 "/Engine/Generated/Material.ush"
        ▼
BasePassPixelShader.usf 等引擎 .usf
    #include "/Engine/Generated/Material.ush"
        │  ← ShaderCompileWorker 从虚拟路径映射表解析 include
        ▼
平台编译器 (DXC / SPIRV / Metal)
        │
        ▼
FMaterialShaderMap (平台 bytecode)
```

### 关键源码

编译入口 — `FMaterial::BeginCompileShaderMap()`（MaterialShared.cpp）：
```cpp
bool FMaterial::BeginCompileShaderMap(...)
{
    // 1. Translate: 节点图 → HLSL
    bSuccess = Translate(ShaderMapId, StaticParameterSet, TargetPlatform, NewCompilationOutput, MaterialEnvironment);
    
    // 2. Compile: HLSL → platform bytecode
    NewShaderMap->Compile(this, ShaderMapId, MaterialEnvironment, NewCompilationOutput, ...);
}
```

Translate 内部 — `FMaterial::Translate_Legacy()`：
```cpp
bool FMaterial::Translate_Legacy(...)
{
    FHLSLMaterialTranslator MaterialTranslator(this, ...);
    MaterialTranslator.Translate(false);

    FString MaterialShaderCode = MaterialTranslator.GetMaterialShaderCode();
    // 注入到虚拟路径
    OutMaterialEnvironment->IncludeVirtualPathToContentsMap.Add(
        TEXT("/Engine/Generated/Material.ush"), MoveTemp(MaterialShaderCode));
}
```

### `MaterialTemplate.ush` 的占位符

模板文件 `Engine/Shaders/Private/MaterialTemplate.ush` 中的关键占位符：

| 占位符 | 填充内容 |
|--------|----------|
| `%{material_declarations}` | 纹理/采样器声明 |
| `%{pixel_material_inputs}` | `FPixelMaterialInputs` 成员定义 |
| `%{uniform_material_expressions}` | Uniform 参数运算 |
| `%{calc_pixel_material_inputs_initial_calculations}` | 节点图主体计算代码 |
| `%{calc_pixel_material_inputs_normal}` | 法线计算 |
| `%{calc_pixel_material_inputs_other_inputs}` | BaseColor/Roughness 等赋值 |
| `%{get_material_world_position_offset_raw}` | WPO 代码 |

### "/Engine/Generated/Material.ush" 不是磁盘文件

- 它是一个**虚拟路径**，在 `FSharedShaderCompilerEnvironment::IncludeVirtualPathToContentsMap` 中维护
- **每个材质独立生成**一份内容，不同材质的编译 job 中该路径指向不同的字符串
- 31 个 .usf 文件都 `#include` 同一路径，但在各自编译时绑定到当前材质的代码

### 编辑器 vs Cook vs 运行时

| 阶段 | Shader 编译 | UMaterial 内容 |
|------|-------------|----------------|
| **编辑器** | 修改/保存材质时异步编译，结果缓存到 DDC | 完整节点图 + ShaderMap |
| **Cook（打包）** | `CacheResourceShadersForCooking()` 为目标平台编译所有变体 | EditorOnly 数据被剥离，ShaderMap 序列化进 .uasset |
| **真机运行时** | **不编译**，`PostLoad()` 反序列化 ShaderMap bytecode | 无节点图，仅属性元数据 + ShaderMap |
| **玩家看到的"编译 Shader"** | GPU 驱动将 bytecode + 渲染状态 → PSO 管线 | — |

---

## 4. Renderer Base Pass 如何使用 Material Shader

### 绘制架构

```
FPrimitiveSceneProxy
    → GetDynamicMeshElements() / 静态路径
        → FMeshBatch { MaterialRenderProxy, VertexFactory }
            → FMeshPassProcessor::AddMeshBatch()
                → TryGetShaders() 从 FMaterialShaderMap 取 VS/PS
                    → BuildMeshDrawCommands() → FMeshDrawCommand
                        → RHI Submit
```

### MeshPassProcessor 查找 Shader

`FBasePassMeshProcessor::AddMeshBatch()`（BasePassRendering.cpp）：

```cpp
void FBasePassMeshProcessor::AddMeshBatch(const FMeshBatch& MeshBatch, ...)
{
    const FMaterialRenderProxy* MaterialRenderProxy = MeshBatch.MaterialRenderProxy;
    const FMaterial* Material = MaterialRenderProxy->GetMaterialNoFallback(FeatureLevel);
    TryAddMeshBatch(MeshBatch, ..., *Material);
}
```

在 `Process()` → `GetBasePassShaders()` 中：

```cpp
FMaterialShaders Shaders;
Material.TryGetShaders(ShaderTypes, VertexFactoryType, Shaders);
Shaders.TryGetVertexShader(VertexShader);   // TBasePassVS
Shaders.TryGetPixelShader(PixelShader);     // TBasePassPS
```

`Material.TryGetShaders()` 从该材质的 `FMaterialShaderMap` 中，按 shader type + vertex factory type 查找编译好的 shader 变体。

### Base Pass Shader 文件

**Vertex Shader** — `BasePassVertexShader.usf`：
```hlsl
void Main(FVertexFactoryInput Input, out FBasePassVSOutput Output)
{
    FMaterialVertexParameters VertexParameters = GetMaterialVertexParameters(...);
    WorldPosition.xyz += GetMaterialWorldPositionOffset(VertexParameters); // ← Material.ush
    Output.Position = mul(WorldPosition, ViewProj);
}
```

**Pixel Shader** — `BasePassPixelShader.usf`：
```hlsl
#include "/Engine/Generated/Material.ush"  // 当前材质的生成代码

void FPixelShaderInOut_MainPS(...)
{
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(Interpolants, SvPosition);
    FPixelMaterialInputs PixelMaterialInputs;
    CalcMaterialParametersEx(MaterialParameters, PixelMaterialInputs, ...);
    // ↑ 内部调用 CalcPixelMaterialInputs()，执行节点图代码
    
    half3 BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
    half  Metallic  = GetMaterialMetallic(PixelMaterialInputs);
    half  Roughness = GetMaterialRoughness(PixelMaterialInputs);
    half3 Normal    = GetMaterialNormal(MaterialParameters, PixelMaterialInputs);
    // → 写入 GBuffer
}
```

实际入口点在 `PixelShaderOutputCommon.ush`：
```hlsl
void MainPS(FVertexFactoryInterpolantsVSToPS Interpolants, ..., out float4 OutTarget0 : SV_Target0, ...)
{
    FPixelShaderInOut_MainPS(Interpolants, ...);  // 调用上面定义的函数
}
```

### Shader 排列组合

一个最终的 Draw Call 对应的 shader 由三个维度组合：

```
Shader 变体 = Vertex Factory × Material × Pass
```

| 维度 | 决定什么 | 例子 |
|------|----------|------|
| **Vertex Factory** | 顶点数据格式和变换方式 | `FLocalVertexFactory`、`FGPUSkinPassthroughVertexFactory` |
| **Material** | 材质属性计算（Material.ush 内容） | 不同材质不同的节点图代码 |
| **Pass** | 渲染目标和光照模型 | BasePass、DepthOnly、ShadowDepth、Velocity |

C++ 侧的 shader 类型（`TBasePassVS`、`TBasePassPS`）通过 `DECLARE_SHADER_TYPE(..., MeshMaterial)` 注册为 `FMeshMaterialShaderType`，引擎自动为每个 Material × VF 组合编译对应变体并存入 `FMaterialShaderMap`。