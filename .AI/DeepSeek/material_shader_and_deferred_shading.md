# Material Details 到 Deferred Shading 的完整解析

> 本文分析路径参考 `UnrealEngine-ue6-main`（主线开发分支），5.8.x 结构近似，个别行号会有偏移，符号与文件位置保持一致。
> 相关背景可参见 [MaterialSystemFramework.md](MaterialSystemFramework.md)。

---

## 0. 全景

材质编辑器右侧 Details 面板上三项核心设置决定了整个渲染管线如何处理该材质：

| 设置 | 决定什么 | 影响层面 |
|------|---------|----------|
| **Material Domain** | 材质用途／挂入哪个子系统 | 决定编译哪套 shader、被哪个 renderer 消费 |
| **Blend Mode** | 像素如何和已有内容混合 | 决定挂到哪个 Mesh Pass、使用什么 BlendState |
| **Shading Model** | 表面光照采用哪个 BxDF | 决定 GBuffer 打包方式、Deferred Lighting 分支 |

三者的正交组合最终决定了：
- 需要生成哪几种 shader 变体
- 在 `FDeferredShadingSceneRenderer::Render()` 中被哪些 Pass 处理
- Base Pass 中 GBuffer 如何写入
- Deferred Lighting Pass 中如何被点亮

后文按 **设置 → 编译 → 渲染管线** 三段展开。

---

## 1. Material Domain（材质用途）

### 1.1 枚举定义

`Engine/Source/Runtime/Engine/Public/MaterialDomain.h:12-30`：

```cpp
enum EMaterialDomain : int
{
    MD_Surface,              // 通用表面材质（网格、地形、粒子等）
    MD_DeferredDecal,        // 延迟贴花
    MD_LightFunction,        // 光源函数（Light Function Material）
    MD_Volume,               // 体积雾/云/异质体积
    MD_PostProcess,          // 后处理材质
    MD_UI,                   // Slate/UMG UI
    MD_RuntimeVirtualTexture,// RVT (5.x 后端逐步替换)
    MD_MAX,
};
```

`FMaterialResource::IsUIMaterial()`/`IsPostProcessMaterial()`/`IsLightFunction()` 等接口（`MaterialShared.cpp:1961-1963`）就是对该枚举的封装。

### 1.2 每种 Domain 在 Renderer 里的落点

| Domain | 由哪套 Renderer 代码处理 | 关键入口 |
|--------|--------------------------|----------|
| `MD_Surface` | 主几何管线（Mesh Draw） | `BasePassRendering.cpp:878-879`（编译环境切换） + `FBasePassMeshProcessor` |
| `MD_DeferredDecal` | Composition Lighting Decal 通路 | `DecalRenderingShared.cpp:212/364/388`、`PostProcessMeshDecals.cpp:65-139` |
| `MD_LightFunction` | 光源着色前的一遍 quad 绘制 | `LightFunctionRendering.cpp:51`（VS）/`:139`（PS）/`:274`（`RenderLightFunction`） |
| `MD_Volume` | 体积雾/云的 marching / voxel 化 | `HeterogeneousVolumes.cpp:534-542`、`VolumetricFogVoxelization.cpp:192`、`VolumetricCloudRendering.cpp:320+` |
| `MD_PostProcess` | 后处理链中的一个 pass | `PostProcessMaterial.cpp:252/416/1392` |
| `MD_UI` | Slate 渲染，走 SlateRHIRenderer | `SlateMaterialShader.cpp:31/50` |

判定形式在渲染代码里通常出现在 `ShouldCompilePermutation` 或 shader 编译环境中：

```cpp
// BasePassRendering.cpp:878
if (Parameters.MaterialParameters.MaterialDomain != MD_Surface)
{
    OutEnvironment.SetDefine(TEXT("SCENE_TEXTURES_DISABLED"), 1);
}
```

也即：**Domain 不匹配的 Shader 根本不会被编译进对应的 MeshPass ShaderMap**。渲染时不需要运行时分支——不同 Domain 走的是完全不同的代码路径。

### 1.3 一图看清

```
UMaterial::MaterialDomain
        │
        ├─ MD_Surface ────────► FBasePassMeshProcessor  ► BasePass ► GBuffer
        │                       + Depth/Shadow/Velocity/…
        │
        ├─ MD_DeferredDecal ──► FMeshDecalMeshProcessor ► DBuffer / GBuffer 修改
        │
        ├─ MD_LightFunction ──► FLightFunctionPS       ► 每个受影响光源绘制一次
        │
        ├─ MD_Volume ─────────► Volumetric Fog/Cloud/HV Voxel Injection
        │
        ├─ MD_PostProcess ────► Post Processing Chain（作为 Blendable）
        │
        └─ MD_UI ─────────────► Slate/UMG，走 SlateRHIRenderer
```

---

## 2. Blend Mode（混合模式）

### 2.1 枚举定义

`Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:245-259`：

```cpp
enum EBlendMode : int
{
    BLEND_Opaque,
    BLEND_Masked,
    BLEND_Translucent,
    BLEND_Additive,
    BLEND_Modulate,
    BLEND_AlphaComposite,
    BLEND_AlphaHoldout,
    // Substrate:
    BLEND_TranslucentColoredTransmittance,
    // 别名 / 兼容:
    BLEND_TranslucentGreyTransmittance = BLEND_Translucent,
    BLEND_ColoredTransmittanceOnly     = BLEND_Modulate,
    BLEND_MAX,
};
```

配套的谓词族在 `MaterialShared.h:150-193` 声明（4 种重载分别接 `EBlendMode`/`FMaterial&`/`UMaterialInterface&`/`FMaterialShaderParameters&`）：

```cpp
bool IsOpaqueBlendMode(...);
bool IsMaskedBlendMode(...);
bool IsTranslucentOnlyBlendMode(...);
bool IsTranslucentBlendMode(...);
bool IsAlphaHoldoutBlendMode(...);
bool IsModulateBlendMode(...);
bool IsAdditiveBlendMode(...);
bool IsAlphaCompositeBlendMode(...);
```

### 2.2 Blend Mode → Mesh Pass 分桶

`EMeshPass::Type`（`MeshPassProcessor.h:58-113`）中与 Blend Mode 相关的桶：

| Mesh Pass | 接哪些 Blend Mode |
|-----------|-------------------|
| `BasePass` | `Opaque`、`Masked` |
| `TranslucencyStandard` | `Translucent`、`Additive`、`AlphaComposite`（默认排序桶） |
| `TranslucencyStandardModulate` | `Modulate` |
| `TranslucencyAfterDOF` | 上述但 Material Details 标记为 After DOF |
| `TranslucencyAfterDOFModulate` | Modulate + After DOF |
| `TranslucencyAfterMotionBlur` | After Motion Blur |
| `TranslucencyHoldout` | `AlphaHoldout` |
| `TranslucencyAll` | 汇总入口，命令合流 |

关键是 **同一个 `FBasePassMeshProcessor` 既处理 Opaque/Masked 又处理所有 Translucency 变体**——它由 `EMeshPass::Type` 参数区分行为（`BasePassRendering.cpp:2575+`）。所以在 Base Pass shader 编译时，Blend Mode 通过 `MATERIALBLENDING_*` 宏切换代码路径。

### 2.3 Blend Mode → 编译宏

`MaterialShared.cpp:2897-2996` 的 `switch(BlendMode)` 会向 shader 编译环境注入下列宏：

```cpp
switch (BlendMode)
{
    case BLEND_Masked:        OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_MASKED"),      1); break;
    case BLEND_Translucent:   OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_TRANSLUCENT"), 1); break;
    case BLEND_Additive:      OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_ADDITIVE"),    1); break;
    case BLEND_Modulate:      OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_MODULATE"),    1); break;
    case BLEND_AlphaComposite:OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_ALPHACOMPOSITE"),1); break;
    case BLEND_AlphaHoldout:  OutEnvironment.SetDefine(TEXT("MATERIALBLENDING_ALPHAHOLDOUT"),1); break;
    // ... Substrate 变体
}
```

`BasePassPixelShader.usf` 中大量 `#if MATERIALBLENDING_MASKED` / `#if MATERIALBLENDING_TRANSLUCENT` 分支就是根据这些宏切换的：Opaque 写 GBuffer，Masked 加 `clip()`，Translucent 走前向光照分支。

### 2.4 Blend Mode → BlendState

`TranslucentRendering.cpp:2557-2618` 有一段最典型的 `switch` 把 Blend Mode 映射到具体 `TStaticBlendState`：

| Blend Mode | Src | Dst | 语义 |
|------------|-----|-----|------|
| `Translucent`      | `SrcAlpha`   | `InvSrcAlpha` | 普通半透明 |
| `Additive`         | `One`        | `One`         | 加法（发光/粒子） |
| `Modulate`         | `DestColor`  | `Zero`        | 乘法（染色） |
| `AlphaComposite`   | `One`        | `InvSrcAlpha` | 预乘 alpha |
| `AlphaHoldout`     | `Zero`       | `InvSrcAlpha` | 只扣 alpha 通道 |

后处理材质也有独立的 blend state 表（`PostProcessMaterial.cpp:147-170`），下标即 `EBlendMode`，静态断言 `BLEND_MAX == UE_ARRAY_COUNT`。

---

## 3. Shading Model（光照模型）

### 3.1 C++ 枚举

`Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:720-741`：

```cpp
enum EMaterialShadingModel : int
{
    MSM_Unlit,               // 不参与光照
    MSM_DefaultLit,          // 标准 GGX
    MSM_Subsurface,          // 简单 SSS
    MSM_PreintegratedSkin,   // 预积分皮肤（人脸）
    MSM_ClearCoat,           // 双层（车漆）
    MSM_SubsurfaceProfile,   // 分离式 SSS（精细皮肤）
    MSM_TwoSidedFoliage,     // 双面叶片透射
    MSM_Hair,                // 头发（Marschner）
    MSM_Cloth,               // 布料（Charlie/Ashikhmin）
    MSM_Eye,                 // 眼球
    MSM_SingleLayerWater,    // 单层水
    MSM_ThinTranslucent,     // 薄透明（玻璃）
    MSM_Strata,              // Substrate（新一代）
    MSM_NUM,
    MSM_FromMaterialExpression, // 由节点决定
};
static_assert(MSM_NUM <= 16); // 4-bit 存 GBuffer
```

`FMaterialShadingModelField` 是一个位掩码，允许材质同时声明多个 SM（配合 `From Material Expression` 使用）。

### 3.2 HLSL 侧的 SHADINGMODELID_*

`Engine/Shaders/Private/ShadingCommon.ush:19-35`：

```hlsl
#define SHADINGMODELID_UNLIT               0
#define SHADINGMODELID_DEFAULT_LIT         1
#define SHADINGMODELID_SUBSURFACE          2
#define SHADINGMODELID_PREINTEGRATED_SKIN  3
#define SHADINGMODELID_CLEAR_COAT          4
#define SHADINGMODELID_SUBSURFACE_PROFILE  5
#define SHADINGMODELID_TWOSIDED_FOLIAGE    6
#define SHADINGMODELID_HAIR                7
#define SHADINGMODELID_CLOTH               8
#define SHADINGMODELID_EYE                 9
#define SHADINGMODELID_SINGLELAYERWATER   10
#define SHADINGMODELID_THIN_TRANSLUCENT   11
#define SHADINGMODELID_STRATA             12
#define SHADINGMODELID_SUBSTRATE_TOON     13
#define SHADINGMODELID_NUM                14
#define SHADINGMODELID_MASK              0xF   // 低 4 bit
```

高 4 bit 用于 flag（`ShadingCommon.ush:37-45`）：

```hlsl
#define HAS_ANISOTROPY_MASK       (1 << 4)
#define SKIP_PRECSHADOW_MASK      (1 << 5)
// ...
```

### 3.3 Shading Model → GBuffer 打包

`DeferredShadingCommon.ush`：
- `EncodeShadingModelIdAndSelectiveOutputMask()` `line 295` — 把 4-bit ShadingModelID + flag 打进 `GBufferB.a`
- `DecodeShadingModelId()` `line 301` — 反过来解码

**关键分派**：`ShadingModelsMaterial.ush:48-214` 是一大段 `if/else if(ShadingModel == SHADINGMODELID_*)`，把 material graph 输出的 CustomData（如 SSS 半径、Cloth 表面颜色、Eye 虹膜遮罩、ClearCoat 参数）按 SM 各自的语义打包到 **GBufferD (`GBuffer.CustomData`)**：

```hlsl
// ShadingModelsMaterial.ush（伪示例）
if (ShadingModel == SHADINGMODELID_CLEAR_COAT)
{
    GBuffer.CustomData.x = ClearCoat;
    GBuffer.CustomData.y = ClearCoatRoughness;
}
else if (ShadingModel == SHADINGMODELID_SUBSURFACE_PROFILE)
{
    GBuffer.CustomData = EncodeSubsurfaceProfile(...);
}
else if (ShadingModel == SHADINGMODELID_HAIR)
{
    GBuffer.CustomData.xyz = HairPrimitiveUV;
    // ...
}
```

于是同样 4 通道的 GBufferD，在不同 SM 下承载完全不同的语义——由 SM ID 决定读回时怎么解释。

### 3.4 Shading Model → Deferred Lighting 分派

`ShadingModels.ush:1141-1170` 中的核心分派函数：

```hlsl
FDirectLighting IntegrateBxDF(FGBufferData GBuffer, ...)
{
    switch (GBuffer.ShadingModelID)
    {
        case SHADINGMODELID_DEFAULT_LIT:        return DefaultLitBxDF(...);
        case SHADINGMODELID_SUBSURFACE:         return SubsurfaceBxDF(...);
        case SHADINGMODELID_PREINTEGRATED_SKIN: return PreintegratedSkinBxDF(...);
        case SHADINGMODELID_CLEAR_COAT:         return ClearCoatBxDF(...);
        case SHADINGMODELID_SUBSURFACE_PROFILE: return SubsurfaceProfileBxDF(...);
        case SHADINGMODELID_TWOSIDED_FOLIAGE:   return TwoSidedBxDF(...);
        case SHADINGMODELID_HAIR:               return HairBxDF(...);
        case SHADINGMODELID_CLOTH:              return ClothBxDF(...);
        case SHADINGMODELID_EYE:                return EyeBxDF(...);
        // Substrate: ToonBxDF, ...
    }
}
```

调用链：
1. `DeferredLightPixelShaders.usf:269 DeferredLightPixelMain` — 每个受影响像素的入口
2. `DeferredLightingCommon.ush:455 GetDynamicLighting` — 从 GBuffer 恢复表面参数
3. → `IntegrateBxDF` 上面这个 switch → 具体 SM 的 BxDF

`FDeferredLightPS` 还根据 Tile 分类切换 4 种 permutation：`Simple/Single/Complex/ComplexSpecial`（`LightRendering.cpp:3412-3453`）。**Complex/ComplexSpecial 才启用 SM 分支**，纯 DefaultLit 走 Simple 分支省成本。

---

## 4. MaterialTemplate.ush 解读

`Engine/Shaders/Private/MaterialTemplate.ush`（约 4929 行）是所有材质 shader 的**骨架文件**。文件头注释直言：

> "Filled in by `FHLSLMaterialTranslator::GetMaterialShaderCode` for each material being compiled."

### 4.1 占位符语法：`%{token}`

**不是 printf 的 `%s`/`%d`**——UE 自己实现了一套模板引擎（`FStringTemplateResolver`）。占位符形如：

```
%{token_name}
```

关键 token（散布在 4929 行中）：

| 占位符 | 填充内容 |
|--------|----------|
| `%{num_material_texcoords_vertex}` `line 111` | 顶点 UV 通道数 |
| `%{num_material_texcoords}` `line 112` | 像素 UV 通道数 |
| `%{num_custom_vertex_interpolators}` `line 113` | 自定义 VS→PS 插值 |
| `%{material_declarations}` `line 364` | Texture/Sampler/Uniform 声明 |
| `%{material_attributes_utilities}` `line 367` | Get/Set MaterialAttribute 辅助函数 |
| `%{pixel_material_inputs}` `line 377` | `FPixelMaterialInputs` 字段 |
| `%{material_pixel_parameter_decls}` `line 654` | `FMaterialPixelParameters` 定制字段 |
| `%{material_vertex_parameter_decls}` `line 757` | `FMaterialVertexParameters` 定制字段 |
| `%{uniform_material_expressions}` `line 3607` | Uniform 表达式求值代码 |
| `%{custom_functions}` `line 3610` | 用户 Custom 节点函数 |
| `%{custom_outputs}` `line 3611` | 自定义输出（Anisotropy、SubsurfaceOpacity 等） |
| `%{get_material_opacity_mask_clip_value}` 起 | 每个 Material Property 的常量 getter |
| `%{get_material_world_position_offset_raw}` `line 4019` | WPO 代码块 |
| `%{evaluate_vertex_material_attributes}` `line 4285` | 顶点属性求值代码块 |
| `%{calc_pixel_material_inputs_...}` `line 4306+` | 主 calc 函数体（法线、其他属性、finite/analytic derivatives 各一份） |
| `#line %{line_number}` `line 4449` | 错误行号映射 |

### 4.2 生成的关键函数（"锚点"）

模板已经写好函数签名，占位符只填充函数体：

| 函数 | 行号 | 作用 |
|------|------|------|
| `FPixelMaterialInputs` (struct) | `375` | 装 BaseColor/Metallic/Normal/… 的容器 |
| `FMaterialLWCData` (struct) | `394` | Large World Coordinates 数据 |
| `FMaterialPixelParameters` (struct) | `654` | PS 输入总参数（含 VF 传来的插值） |
| `FMaterialVertexParameters` (struct) | `757` | VS 输入总参数 |
| `GetMaterialCustomizedUVs()` | `4167` | Customized UV 输出 |
| `GetMaterialPixelDepthOffset()` | `4188` | PDO 输出 |
| `EvaluateVertexMaterialAttributes()` | `4283` | VS 端所有属性一次求值 |
| `CalcPixelMaterialInputs()` | `4301` | PS 端主计算（finite diff 版） |
| `CalcPixelMaterialInputsAnalyticDerivatives()` | `4374` | PS 端主计算（解析导数版） |
| `GetMaterialWorldPositionOffsetRaw()` | `4015` | WPO |
| `GetMaterialCoverageAndClipping()` | `4547` | Opacity Mask / Coverage |
| `CalcMaterialParametersEx()` | `4632` | PS 顶层入口（被 `.usf` 直接调用） |
| `ApplyPixelDepthOffsetToMaterialParameters()` | `4807` | 应用 PDO 后重算参数 |

### 4.3 文件的宏观组织

```
1–46       引擎 include 块（Common.ush 等）
65–107     PostProcess/SceneTexture ID 定义
109–130    UV/Interpolator 计数 + VertexFactory 前向声明
360–430    MaterialAttribute 结构声明
654 / 757  FMaterialPixelParameters / FMaterialVertexParameters
——中段——    数学工具、LWC 辅助、noise/derivative helpers
2050–2600  节点图生成的 MaterialExpression* 辅助函数（由占位符替换进来）
3606–3612  Uniform 表达式块（占位符）
3739–3853  各 Material Property 的常量 getter（Opacity Mask Clip Value 等）
4015–4023  WPO getter
4283–4290  Vertex 属性求值（+ Previous）
4301 / 4374 CalcPixelMaterialInputs 两个变体（法线单独一段 + 其余属性）
4632 / 4716/ 4736  CalcMaterialParameters / Ex / Post
```

### 4.4 一次编译产出的完整 HLSL 图

```
┌─────────────────────────────────────────────────────┐
│  MaterialTemplate.ush（骨架，占位符未填）          │
│  ┌────────────────────────────────────────┐        │
│  │ 4929 行，包含 %{material_declarations}, │        │
│  │ %{uniform_material_expressions}, ...    │        │
│  └────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────┘
              │
              │ FStringTemplateResolver 替换所有 %{...}
              │ 每个材质替换出的字符串都不同
              ▼
┌─────────────────────────────────────────────────────┐
│ /Engine/Generated/Material.ush（**虚拟路径**）      │
│ ← 此材质独有的完整 HLSL                             │
└─────────────────────────────────────────────────────┘
              │
              │ #include from BasePassPixelShader.usf 等
              ▼
        平台编译器（DXC / SPIRV-Cross / Metal）
              │
              ▼
        Shader Bytecode → FMaterialShaderMap
```

---

## 5. 从节点图到 Shader — 编译 Pipeline

### 5.1 三个核心角色

| 角色 | 文件 | 职责 |
|------|------|------|
| `FHLSLMaterialTranslator` | `HLSLMaterialTranslator.h:235` | 遍历节点图，生成 HLSL 代码块 |
| `FMaterialSourceTemplate` | `MaterialSourceTemplate.h/cpp` | 加载 MaterialTemplate.ush，解析 `%{...}` |
| `FStringTemplateResolver` | 同上 | 实际做替换的对象 |

`FHLSLMaterialTranslator` 派生自 `FMaterialCompiler`（`HLSLMaterialTranslator.h:235`），覆写了大量 `virtual int32 XXX(...)`（`h:796+`）——每个 override 对应一种 material expression 能"喷出"的代码原语（Add、Mul、TextureSample、If、Custom、…）。

### 5.2 编译入口

```cpp
// HLSLMaterialTranslator.cpp:1173
bool FHLSLMaterialTranslator::Translate(bool bAllowSubgraph)
{
    // ... DDC 命中检查 / job 调度
    return TranslateMaterial();
}

// HLSLMaterialTranslator.cpp:1326
bool FHLSLMaterialTranslator::TranslateMaterial()
{
    // 按 Material Property 分别驱动节点图遍历
    Material->CompilePropertyAndSetMaterialProperty(MP_Normal,           this); // line 1493
    Material->CompilePropertyAndSetMaterialProperty(MP_EmissiveColor,    this); // 1500
    Material->CompilePropertyAndSetMaterialProperty(MP_BaseColor,        this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Metallic,         this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Specular,         this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Roughness,        this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Tangent,          this);
    Material->CompilePropertyAndSetMaterialProperty(MP_ShadingModel,     this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Anisotropy,       this);
    Material->CompilePropertyAndSetMaterialProperty(MP_Opacity,          this);
    Material->CompilePropertyAndSetMaterialProperty(MP_OpacityMask,      this);
    Material->CompilePropertyAndSetMaterialProperty(MP_WorldPositionOffset, this);
    Material->CompilePropertyAndSetMaterialProperty(MP_SurfaceThickness, this);
    Material->CompilePropertyAndSetMaterialProperty(MP_FrontMaterial,    this); // 1536
    // ...
}
```

每个 `CompilePropertyAndSetMaterialProperty` 会顺着材质图从该 Material Attribute 引脚开始，深度优先递归调用连接的 `UMaterialExpression::Compile(FMaterialCompiler*)`——节点自己知道该往 translator 里"喷"什么代码。

递归入口在 `HLSLMaterialTranslator.cpp:4554 / 4609`（通过 `FMaterialExpressionKey` 去重缓存）：

```cpp
int32 CodeChunkIndex = Expression->Compile(this, ExpressionOutputIndex);
```

### 5.3 生成的中间代码

每次翻译产生的中间代码由 chunk 数组表示，最终按四种粒度存储：
- Per-Vertex chunk：VS 内计算
- Per-Pixel chunk：PS 内计算
- Shared chunk：VS→PS 插值
- Uniform chunk：材质 uniform buffer

### 5.4 模板填充

```cpp
// HLSLMaterialTranslator.cpp:2750
FString FHLSLMaterialTranslator::GetMaterialShaderCode()
{
    FStringTemplateResolver Resolver = FMaterialSourceTemplate::Get()
        .BeginResolve(GetShaderPlatform(), &MaterialTemplateLineNumberNew);       // 2754

    Resolver.SetParameterMap(&MaterialSourceTemplateParams);                       // 2756

    // ... 设置 line_number 等收尾参数

    return Resolver.Finalize();                                                    // 2766
}
```

`MaterialSourceTemplateParams` 是在 `TranslateMaterial` 过程中不断 `Add({"token", value})` 建立的一张 map。集中的赋值段在 `HLSLMaterialTranslator.cpp:15864-16071`——把前面遍历图生成的所有 chunk 字符串塞进 token 表。

### 5.5 编译产物注册为虚拟路径

Translator 完成后返回的 HLSL 字符串**并不写盘**，而是注册到共享编译环境：

```cpp
// FMaterial::Translate_Legacy（MaterialShared.cpp）
OutMaterialEnvironment->IncludeVirtualPathToContentsMap.Add(
    TEXT("/Engine/Generated/Material.ush"),
    MoveTemp(MaterialShaderCode));
```

Base Pass 等 `.usf` 文件里的：
```hlsl
#include "/Engine/Generated/Material.ush"
```
会通过 `FShaderCompilerEnvironment` 的虚拟路径映射解析到**当前正在编译的这个材质**的字符串。**同一个虚拟路径在不同材质的编译 job 里指向不同内容。**

### 5.6 Shader 变体存储：FMaterialShaderMap

材质的编译产物按 **Material × Vertex Factory × Shader Type** 三轴组织：

```
FMaterialShaderMap (per material, per platform, per quality)
    ↓
FMaterialShaderMapContent
    ├── ShaderCache[]          ← 无 VF 轴的 shader（PostProcess、LightFunction…）
    │
    └── OrderedMeshShaderMaps[VertexFactoryIndex]
            ↓
        FMeshMaterialShaderMap  ← 与该 VF 关联的所有 pass 变体
            ├── TBasePassVS       bytecode
            ├── TBasePassPS       bytecode
            ├── FDepthOnlyVS/PS
            ├── FShadowDepthVS/PS
            └── ...
```

关键类：
- `FMaterialShaderMap` — `MaterialShared.h:1712`，静态查找入口 `FindId()` `:1723`、DDC 加载 `:1736`、`AcquireMeshShaderMap(VFType)` `:1874`
- `FMaterialShaderMapContent` — `MaterialShared.h:1646`，持有 `OrderedMeshShaderMaps` `:1691`
- `FMeshMaterialShaderMap` — `MaterialShared.h:1618`
- `FMaterialShaderMapId` — `MaterialShared.h:1379`，唯一标识一份 shader map（含 shader 平台、feature level、static switch 状态、quality level 等）

### 5.7 Shader 基类

`FMaterialShader` — `MaterialShader.h:55`：**不**含 VF 轴，供 PostProcess/LightFunction 等 quad-based 材质用。
```cpp
class FMaterialShader : public FShader
{
    using ShaderMetaType = FMaterialShaderType;
    // SetViewParameters / SetParameters / GetShaderBindings
};
```

`FMeshMaterialShader` — `MeshMaterialShader.h:67`：加入 VF 轴，供所有走 Mesh Draw 的 pass 用。
```cpp
class FMeshMaterialShader : public FMaterialShader
{
    using ShaderMetaType = FMeshMaterialShaderType;
    // 增加 VertexFactoryParameters (line 120)
    //     PassUniformBuffer     (line 123)
    //     per-mesh-batch element bindings
};
```

其 `FMeshMaterialShaderPermutationParameters`（`MeshMaterialShader.h:32`）比 `FMaterialShaderPermutationParameters` 多一个 `const FVertexFactoryType* VertexFactoryType`——**这是 VF 变体轴的来源**。

Shader 类型工厂：
- `FMaterialShaderType` — `MaterialShaderType.h:89`
- `FMeshMaterialShaderType` — `MeshMaterialShaderType.h:25`

声明宏：
```cpp
// 材质 shader（无 VF 轴）
IMPLEMENT_MATERIAL_SHADER_TYPE(TemplatePrefix, ShaderClass, SourceFilename, FunctionName, Frequency)

// 网格材质 shader（有 VF 轴）—— 直接用 IMPLEMENT_SHADER_TYPE
// ShaderMetaType 为 FMeshMaterialShaderType 时自动走 mesh 路径
```

---

## 6. DeferredShadingSceneRenderer 完整时序

### 6.1 顶层 Render() 一览

`Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:1832 FDeferredShadingSceneRenderer::Render(...)`：

```
┌─ InitViews / VisibilityCull / Update Uniforms ─────────────────┐
│                                                                 │
├─ RenderPrePass (Early-Z)                            :2473       │  ← 消费材质
├─ RenderShadowDepthMaps                              :2948/2974  │  ← 消费材质
├─ CompositionLighting.ProcessBeforeBasePass (DBuffer):2984       │  ← 消费材质 (DeferredDecal 早期)
├─ RenderBasePass (GBuffer 写入)                      :3083       │  ← 消费材质 ★ 主战场
├─ RenderVelocities (opaque)                          :3401       │  ← 消费材质
├─ CompositionLighting.ProcessAfterBasePass (Emissive):3222/3418  │  ← 消费材质 (Decal 后期)
├─ RenderLights (Deferred Lighting)                   :3511       │  ← 光照 PS + LightFunction 材质
├─ RenderDeferredReflectionsAndSkyLighting            :3573       │
├─ RenderTranslucency                                 :3534/3694  │  ← 消费材质
├─ RenderVelocities (translucent)                     :3687       │
├─ AddPostProcessingPasses                            :4437       │  ← 消费 PostProcess 材质
└────────────────────────────────────────────────────────────────┘
```

### 6.2 每个 Pass 用哪个 Mesh Pass Processor

| Pass | Processor 类 | 定义位置 | 用哪种 Shader |
|------|-------------|----------|-------------|
| PrePass / Early-Z | `FDepthPassMeshProcessor` | `DepthRendering.h:199` | `FDepthOnlyVS/PS`（Masked 才需要 PS） |
| BasePass（Opaque/Masked/全 Translucency） | `FBasePassMeshProcessor` | `BasePassRendering.h:983` | `TBasePassVS/PS<LightMapPolicy>` |
| ShadowDepth | `FShadowDepthPassMeshProcessor` | `ShadowRendering.h:174` | `FShadowDepthVS/PS` |
| Translucent Shadow Depth | `FTranslucencyDepthPassMeshProcessor` | `TranslucentLighting.cpp:539` | 半透阴影专用 |
| Velocity | `FOpaqueVelocityMeshProcessor` / `FTranslucentVelocityMeshProcessor` | `VelocityRendering.h:159/202/247` | `FVelocityVS/PS` |
| MeshDecals | `FMeshDecalMeshProcessor` | `PostProcessMeshDecals.cpp:157` | 按 DBuffer/BeforeLighting/Emissive 分组 |
| Anisotropy (GBufferF) | `FAnisotropyMeshProcessor` | `AnisotropyRendering.h:12` | Anisotropy pass |
| SkyPass | `FSkyPassMeshProcessor` | `SkyPassRendering.h:15` | Sky Atmosphere / Sky Material |
| CustomDepth | `FCustomDepthPassMeshProcessor` | `CustomDepthRendering.cpp:459` | 与 DepthPass 结构相似 |
| SingleLayerWater | `FSingleLayerWaterPassMeshProcessor` | `SingleLayerWaterRendering.cpp:2067` | 特殊 forward |
| Distortion | `FDistortionMeshProcessor` | `DistortionRendering.h:27` | 折射变形 |
| Lumen Card Capture | `FLumenCardMeshProcessor` | `Lumen/LumenSceneCardCapture.cpp:383` | 用于 Lumen Card 采集 |
| Ray Tracing | `FRayTracingMeshProcessor` | `RayTracing/RayTracingMaterialHitShaders.h:26` | RT Hit shader |
| HitProxy / Selection (Editor) | `FHitProxyMeshProcessor` / `FEditorSelectionMeshProcessor` | `SceneHitProxyRendering.h:20/48` | 编辑器选中/拾取 |

**注意**：**没有** `FLightFunctionMeshProcessor`——`MD_LightFunction` 材质走单独的 quad 绘制路径（`FLightFunctionVS` + `FLightFunctionPS`，`LightFunctionRendering.cpp:39/128`），不是 mesh draw。这是"Domain 分派"的最直观例子。

### 6.3 Base Pass 内部（Opaque/Masked）

`BasePassPixelShader.usf` 主体：

```hlsl
// line 930
void FPixelShaderInOut_MainPS(
    FVertexFactoryInterpolantsVSToPS Interpolants,
    FBasePassInterpolantsVSToPS BasePassInterpolants,
    inout FPixelShaderIn PixelShaderIn,
    inout FPixelShaderOut PixelShaderOut)
{
    FMaterialPixelParameters MaterialParameters =
        GetMaterialPixelParameters(Interpolants, PixelShaderIn.SvPosition);

    FPixelMaterialInputs PixelMaterialInputs;
    CalcMaterialParametersEx(MaterialParameters, PixelMaterialInputs, ...);
    //  ↑ 由 /Engine/Generated/Material.ush 提供实现（当前材质的节点图代码）

#if MATERIALBLENDING_MASKED
    // Opacity Mask clip()
    clip(GetMaterialMask(PixelMaterialInputs));
#endif

    // 从 PixelMaterialInputs 抽取 BaseColor/Metallic/Roughness/Normal 等
    FGBufferData GBuffer = (FGBufferData)0;

    // line 1178 / 1983
    SetGBufferForShadingModel(GBuffer, MaterialParameters, PixelMaterialInputs, ...);
    //  ↑ 按 SM 打包 CustomData（就是第 3.3 节讲的分派）

    // line 2404 / 2448
    EncodeGBufferToMRT(...);  // 写到 OutGBufferA..E / Velocity / Anisotropy
}
```

Vertex Shader (`BasePassVertexShader.usf`) 主要负责调用 `GetMaterialWorldPositionOffset(VertexParameters)`（也是 Material.ush 生成的），把 WPO 加到 world position 上，再乘投影矩阵。

### 6.4 GBuffer 布局（Desktop / SHADING_PATH_DEFERRED）

`DeferredShadingCommon.ush:1053 EncodeGBuffer` / `:1161 DecodeGBufferData`，MRT 绑定在 `:1069-1116`：

| MRT | RGB | A | 用途 |
|-----|-----|---|------|
| **GBufferA** | `EncodeNormal(WorldNormal)` | `PerObjectGBufferData` | 世界法线 |
| **GBufferB** | R=Metallic G=Specular B=Roughness | `EncodeShadingModelIdAndSelectiveOutputMask(SM, mask)` | PBR 三件套 + SM ID |
| **GBufferC** | `EncodeBaseColor(BaseColor)` | `EncodeIndirectIrradiance(...)` | 反照率 + 间接光 |
| **GBufferD** | `GBuffer.CustomData.rgba` | — | **Per-SM 自定义 4 通道** |
| **GBufferE** | `GBuffer.PrecomputedShadowFactors` | — | 静态阴影因子（`ALLOW_STATIC_LIGHTING`） |
| **GBufferF** (可选) | Anisotropy tangent | — | 由 Anisotropy Pass 单独写 |
| **Velocity** (可选) | 屏幕空间速度 | — | `WRITES_VELOCITY_TO_GBUFFER` |

**桌面通常 5–7 个 MRT，Mobile 走独立的 `MobileEncodeGBuffer`（同文件 line 637）压到 4 个。**

### 6.5 Deferred Lighting 阶段

```
FDeferredShadingSceneRenderer::RenderLights (LightRendering.cpp:1739)
    │
    │  遍历 Scene.Lights
    │
    ├─ ★ RenderLightFunction (LightFunctionRendering.cpp:274)
    │       │  为当前光源绘制一个受影响区域，PS 采样 MD_LightFunction 材质
    │       │  结果写入 Light Attenuation
    │       └─ FLightFunctionVS/PS
    │
    ├─ InternalRenderLight (LightRendering.cpp:3070)
    │       │
    │       ├─ 选 permutation: Simple / Single / Complex / ComplexSpecial
    │       │  (line 3412-3453)  ← 根据 Tile 内 SM 多样性
    │       │
    │       └─ 绑定 FDeferredLightPS
    │              │
    │              ▼
    │       DeferredLightPixelShaders.usf:269 DeferredLightPixelMain
    │              │
    │              │  从 GBuffer 反解出 FGBufferData
    │              │  DecodeGBufferData(...) ← DeferredShadingCommon.ush:1161
    │              │
    │              ▼
    │       DeferredLightingCommon.ush:455 GetDynamicLighting
    │              │
    │              │  应用阴影、Light Function、衰减
    │              │
    │              ▼
    │       ShadingModels.ush:1141 IntegrateBxDF(GBuffer, ...)
    │              │
    │              │  switch(GBuffer.ShadingModelID)
    │              │
    │              ▼
    │       DefaultLitBxDF / ClearCoatBxDF / HairBxDF / …
    │
    └─ 累加到 SceneColor（Additive Blend）
```

**这里能清楚看到"数据接力"**：
1. Base Pass 材质 shader 把 BaseColor/Metallic/…/CustomData 写进 GBuffer；
2. Lighting Pass **不再有材质 shader 的存在**——统一按 SM ID switch 到对应 BxDF；
3. SM 是唯一在两个阶段之间"传递意图"的通道（+ 4 通道 CustomData 承载 SM 私有数据）。

---

## 7. 数据流总览 — 一个材质球从 Details 到最终像素

以一个 **Domain=Surface / BlendMode=Opaque / ShadingModel=DefaultLit** 的普通材质为例：

```
┌────────────────────────── 编辑器 ─────────────────────────────┐
│  UMaterial (节点图 + Details 设置)                            │
│      Domain=Surface, Blend=Opaque, SM=DefaultLit              │
└───────────────────────────┬──────────────────────────────────┘
                            │
             ┌──────────────┴───────────────┐
             │ FMaterial::BeginCompileShaderMap     (MaterialShared.cpp)
             │      │
             │      ├─ FMaterial::Translate_Legacy
             │      │      │
             │      │      └─ FHLSLMaterialTranslator::Translate
             │      │             ├─ TranslateMaterial (:1326)
             │      │             │    └─ CompilePropertyAndSetMaterialProperty × N
             │      │             │           └─ UMaterialExpression::Compile()
             │      │             │
             │      │             └─ GetMaterialShaderCode (:2750)
             │      │                    └─ FStringTemplateResolver.Finalize
             │      │                          └─ MaterialTemplate.ush + %{...} 替换
             │      │
             │      │  产物：完整 HLSL 字符串
             │      │  注册为虚拟路径 /Engine/Generated/Material.ush
             │      │
             │      └─ FMaterialShaderMap::Compile
             │             │
             │             ├─ Enum 所有相关 shader type × VF type
             │             │    (Domain=Surface + Blend=Opaque → 需要
             │             │     BasePass、Depth、Shadow、Velocity 等)
             │             │
             │             └─ Dispatch to ShaderCompileWorker
             │                     ▼
             │             DXC/SPIRV/Metal 平台编译
             │                     ▼
             │             FMaterialShaderMap: bytecode 存好
             │
             ▼
┌────────────────────── 运行时（PIE 或打包后） ─────────────────┐
│                                                                │
│  FPrimitiveSceneProxy::GetDynamicMeshElements                  │
│      → FMeshBatch { MaterialRenderProxy, VertexFactory }       │
│                                                                │
│  每帧 FDeferredShadingSceneRenderer::Render (:1832)：          │
│                                                                │
│  1. RenderPrePass                                              │
│     └─ FDepthPassMeshProcessor 用 FDepthOnlyVS/PS（Opaque 无 PS）│
│                                                                │
│  2. RenderShadowDepthMaps                                      │
│     └─ FShadowDepthPassMeshProcessor 用 FShadowDepthVS/PS      │
│                                                                │
│  3. RenderBasePass ★                                           │
│     └─ FBasePassMeshProcessor                                  │
│         │  TryGetShaders(TBasePassVS<...>, TBasePassPS<...>)   │
│         │  从 FMaterialShaderMap[thisVF] 拿 bytecode            │
│         │                                                       │
│         ▼                                                       │
│     GPU:                                                        │
│     VS: BasePassVertexShader.usf                                │
│         → GetMaterialWorldPositionOffset() [Material.ush]      │
│         → 投影                                                  │
│     PS: BasePassPixelShader.usf                                 │
│         → CalcMaterialParametersEx() [Material.ush]            │
│         → SetGBufferForShadingModel(SM=DefaultLit)             │
│         → EncodeGBuffer → 5+ MRT 写出                          │
│                                                                │
│  4. RenderVelocities                                           │
│  5. RenderLights                                               │
│     └─ InternalRenderLight → FDeferredLightPS                  │
│         ├─ DecodeGBufferData 从 5+ MRT 恢复表面参数            │
│         ├─ IntegrateBxDF (switch SM_ID)                        │
│         │  → DefaultLitBxDF                                    │
│         └─ 累加到 SceneColor                                   │
│                                                                │
│  6. RenderTranslucency (若有半透物体)                          │
│  7. PostProcess (Tonemapper, Bloom, ...)                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 若把设置切换成"Translucent"：

- 3 步 BasePass 中 Blend 分支进 `MATERIALBLENDING_TRANSLUCENT` 路径 → **不写 GBuffer**，而是**当场做 forward lighting**（在同一个 PS 内累加所有影响该物体的光源）
- 挂到 `EMeshPass::TranslucencyStandard` 桶而非 `BasePass` 桶
- BlendState 变成 `SrcAlpha, InvSrcAlpha`
- **不再经过第 5 步的 Deferred Lighting**（GBuffer 里根本没这个物体）

### 若把设置切换成"Domain=LightFunction"：

- **不生成** BasePass/Shadow/Velocity 等 mesh 变体，只生成 `FLightFunctionVS/PS` 变体
- 完全不出现在 `RenderBasePass` 中
- 只在 `RenderLights` 内被 `RenderLightFunction` 挑出来，作为其绑定光源的调制 mask 使用

### 若把设置切换成"Domain=PostProcess"：

- 不进入任何 Mesh Draw / MeshPass Processor
- 在 `AddPostProcessingPasses` 阶段被拾取为一个 blendable，作为 fullscreen quad 绘制
- 使用 `PostProcessMaterial.cpp:147-170` 里那张 blend state 表

---

## 8. 关键结论一句话总结

1. **Material Domain 决定编译哪套 Shader 变体**，也就同时决定它被哪个子系统"看见"。渲染时几乎不再有分支——不同 Domain 就是不同的执行路径。
2. **Blend Mode 决定挂到哪个 Mesh Pass 以及 BlendState**，同时通过 `MATERIALBLENDING_*` 宏在 Base Pass shader 里做静态代码切换（写 GBuffer vs Forward Lighting）。
3. **Shading Model 决定 GBuffer.CustomData 的语义和 Deferred Lighting 分支**，是 Base Pass 和 Lighting Pass 之间唯一的"表面类型"传递通道。
4. **MaterialTemplate.ush 是骨架，节点图 + `FHLSLMaterialTranslator` 生成占位符内容，`FStringTemplateResolver` 做替换**，最终作为虚拟路径 `/Engine/Generated/Material.ush` 被多个 pass 的 `.usf` include。
5. **同一份材质 → `FMaterialShaderMap` → 多个 `FMeshMaterialShaderMap`（每 VF 一个）→ 每个 pass 一份 VS+PS bytecode**。Renderer 的每个 MeshPass Processor 根据 (Material, VF, Pass) 三元组从中取对应 bytecode 生成 `FMeshDrawCommand` 提交给 RHI。
