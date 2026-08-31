# 虚幻引擎渲染概览

## 渲染组件

在游戏世界 `UWorld` 中，`AActor` 是逻辑的载体，但它并不直接参与渲染，需要为 `AActor` 添加渲染相关组件。Unreal Engine 中，渲染相关的组件分三大类：
1. `UCameraComponent`：决定观察的视角
2. `UPrimitiveComponent`：决定被渲染物体的几何与材质
3. `ULightComponent`：决定场景的光照

```mermaid
classDiagram
    AActor o-- USceneComponent
    USceneComponent <|-- UCameraComponent
    USceneComponent <|-- UPrimitiveComponent
    UPrimitiveComponent  <|-- UMeshComponent
    UMeshComponent <|-- UStaticMeshComponent
    UMeshComponent <|-- USkinnedMeshComponent
    USkinnedMeshComponent <|-- USkeletalMeshComponent
    UMeshComponent o-- UMaterialInterface : OverrideMaterials
    UMaterialInterface <|-- UMaterial
    UMaterialInterface <|-- UMaterialInstance
    USceneComponent <|-- ULightComponentBase
    ULightComponentBase <|-- ULightComponent
    ULightComponent <|-- UDirectionalLightComponent
    ALight *-- ULightComponent : LightComponent
    AActor <|-- ALight
    ALight <|-- ADirectionalLight
    ALight <|-- APointLight
    ALight <|-- ARectLight
    ALight <|-- ASpotLight
    ULightComponent <|-- ULocalLightComponent
    ULocalLightComponent <|-- UPointLightComponent
    ULocalLightComponent <|-- URectLightComponent
    UPointLightComponent <|-- USpotLightComponent

    class UPrimitiveComponent {
        +UMaterialInterface* GetMaterial(int32)*
    }

    class UMeshComponent {
        +TArray~UMaterialInterface*~ OverrideMaterials;
    }

    class ULightComponent {

    }

    class ALight {
        -ULightComponent* LightComponent
    }
```

### 相机

### 图形与材质

### 光照

## 渲染线程与渲染场景

### 渲染线程

为了提高效率，渲染逻辑和游戏逻辑运行在不同的线程上。渲染线程 `FRenderingThread` 继承自 `FRunnable`，其核心成员如下：

```mermaid
classDiagram
    FRunnable <|-- FRenderingThread
```

#### 渲染命令

```cpp
/** Declares a new render command tag type from a name. */
#define DECLARE_RENDER_COMMAND_TAG(Type, Name, ...) \
    struct UE_JOIN(TSTR_, Name, __LINE__) \
    {  \
        static const char* CStr() { return #Name; } \
        static const TCHAR* TStr() { return TEXT(#Name); } \
        static constexpr ERenderCommandCategory GetCategory() { return __VA_OPT__(ERenderCommandCategory::__VA_ARGS__;) ERenderCommandCategory::Unknown; } \
    }; \
    using Type = TRenderCommandTag<UE_JOIN(TSTR_, Name, __LINE__)>;

/** Enqueues a render command to a render pipe. The default implementation takes a lambda and schedules on the render thread.
 *  Alternative implementations accept either a reference or pointer to an FRenderCommandPipe instance to schedule on an async
 *  pipe, if enabled.
 */
#define ENQUEUE_RENDER_COMMAND(Type, ...) \
    DECLARE_RENDER_COMMAND_TAG(UE_JOIN(FRenderCommandTag_, Type, __LINE__), Type, __VA_ARGS__) \
    FRenderCommandDispatcher::Enqueue<UE_JOIN(FRenderCommandTag_, Type, __LINE__)>
```


```mermaid
classDiagram
    class FRenderCommandList {

    }
    class FRenderCommandDispatcher {

    }
```

### 渲染场景

```mermaid
classDiagram
    FSceneInterface <|-- FScene
    FPrimitiveSceneProxy <|-- FStaticMeshSceneProxy
    FPrimitiveSceneProxy <|-- FSkeletalMeshSceneProxy
    FMaterial <|-- FMaterialResource
    FSceneView <|-- FViewInfo

    class FSceneInterface {
        +void AddPrimitive(UPrimitiveComponent*)*
        +void RemovePrimitive(UPrimitiveComponent*)*
        +void ReleasePrimitive(UPrimitiveComponent*)*
        +void AddLight(ULightComponent*)*
        +void RemoveLight(ULightComponent*)*
    }

    class FPrimitiveSceneProxy {

    }

    class UPrimitivieComponent {
        +FPrimitiveSceneProxy* CreateSceneProxy()*
    }

    class UMaterialInterface {
        +FMaterialRenderProxy* GetRenderProxy()*
    }

    class ULightComponent {
        +FLightSceneProxy* CreateSceneProxy()*
    }

    class FLightSceneProxy {

    }
```

```mermaid
flowchart TB
    subgraph Game Thread
        UCameraComponent/FMinimalViewInfo
        UPrimitivieComponent
        ULightComponent
        UMaterialInterface/UMaterial/UMaterialInstance
    end

    subgraph Intermediate Layer
        FPrimitiveSceneProxy
        FLightSceneProxy
        FMaterialRenderProxy
    end

    subgraph Render Thread
        FSceneView/FViewInfo
        FPrimitiveSceneInfo
        FLightSceneInfo
        FMaterial/FMaterialResource
    end
```

在 SceneRendering.cpp 中，不同版本的 Unreal Engine 略有差异：

Unreal Engine 5.5
```cpp
RenderViewFamilies_RenderThread
```

Unreal Engine 5.6
```cpp
RenderViewFamily_RenderThread
```

```mermaid
classDiagram
    ISceneRenderer <|.. FSceneRendererBase
    FSceneRendererBase <|-- FSceneRenderer
    FSceneRenderer <|-- FDeferredShadingSceneRenderer
    FSceneRenderer <|-- FMobileSceneRenderer
```

## 延迟渲染管线

```mermaid
classDiagram
    IPSOCollector <|.. FMeshPassProcessor
    FMeshPassProcessor <|-- FBasePassMeshProcessor

    class FDeferredShadingSceneRenderer {

    }
```

## RDG & RHI
