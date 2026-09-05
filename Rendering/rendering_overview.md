# 虚幻引擎渲染概览

## 渲染组件

在游戏世界 `UWorld` 中，`AActor` 是逻辑的载体，但它并不直接参与渲染，需要为 `AActor` 添加渲染相关组件。Unreal Engine 中，渲染相关的组件分三大类：
1. `UCameraComponent`：决定观察的视角
2. `UPrimitiveComponent`：决定被渲染物体的几何与材质
3. `ULightComponent`：决定场景的光照

```mermaid
classDiagram
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

    ALight <|-- ADirectionalLight
    ALight <|-- APointLight
    ALight <|-- ARectLight
    ALight <|-- ASpotLight
    ULightComponent <|-- ULocalLightComponent
    ULocalLightComponent <|-- UPointLightComponent
    ULocalLightComponent <|-- URectLightComponent
    UPointLightComponent <|-- USpotLightComponent

    class UCameraComponent {
        +float FieldOfView
        +float AspectRatio
    }

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

相机决定渲染时的观察视角，相机组件 `UCameraComponent` 决定了相机的具体参数，如视野（`FieldOfView`）、投影模式（`ProjectionMode`）等，详见[官方文档](https://dev.epicgames.com/documentation/unreal-engine/using-cameras-in-unreal-engine)。相机与后处理（PostProcess）紧密相关，相机组件 `UCameraComponent` 包含了后处理相关的设置，如 `PostProcessBlendWeight` 和 `PostProcessSettings`。具体的后处理参数，详见[官方文档](https://dev.epicgames.com/documentation/unreal-engine/post-process-effects-in-unreal-engine)

```mermaid
classDiagram
    ACameraActor *-- UCameraComponent : CameraComponent
    USceneComponent <|-- UCameraComponent
    UCameraComponent o-- FPostProcessSettings : PostProcessSettings

    class ACameraActor {
        -UCameraComponent* CameraComponent
        -USceneComponent* SceneComponent
    }
    
    class UCameraComponent {
        +ECameraProjectionMode ProjectionMode
        +float FieldOfView
        +float AspectRatio
        +float OrthoWidth
        +float PostProcessBlendWeight
        +FPostProcessSettings PostProcessSettings;
    }

    class FPostProcessSettings {

    }
```

在 Detials 面板上设置相机相关的参数：

![](../.figures/camera_component_details.jpg)

### 图形与材质

`UPrimitiveComponent` 是图形组件的“抽象”基类，提供了对材质的访问接口 `GetMaterial` 方法。`UMaterialInterface` 是材质的接口，`UMaterial` 和 `UMaterialInstance` 是其具体实现。

```mermaid
classDiagram
    AStaticMeshActor *-- UStaticMeshComponent : StaticMeshComponent
    UPrimitiveComponent  <|-- UMeshComponent
    UMeshComponent <|-- UStaticMeshComponent
    UMeshComponent <|-- USkinnedMeshComponent
    USkinnedMeshComponent <|-- USkeletalMeshComponent
    UMeshComponent o-- UMaterialInterface : OverrideMaterials
    UMaterialInterface <|-- UMaterial
    UMaterialInterface <|-- UMaterialInstance
    UStaticMeshComponent *-- UStaticMesh : StaticMesh

    class UPrimitiveComponent {
        +UMaterialInterface* GetMaterial(int32)*
    }
    
    class UMeshComponent {
        +TArray~UMaterialInterface*~ OverrideMaterials;
    }

    class UStaticMeshComponent {
        -UStaticMesh* StaticMesh
    }

    class AStaticMeshActor {
        -UStaticMeshComponent* StaticMeshComponent
    }

    class UStaticMesh {

    }
```

#### 静态网格体（Static Mesh）

静态网格体（Static Mesh）表示一个不会发生形变的几何体，详见[官方文档](https://dev.epicgames.com/documentation/unreal-engine/static-meshes) 

### 光照

天光：

```mermaid
classDiagram
    AActor <|-- AInfo
    AActor o-- USceneComponent
    USceneComponent <|-- ULightComponentBase
    ULightComponentBase <|-- USkyLightComponent
    AInfo <|-- ASkyLight
    ASkyLight *-- USkyLightComponent : LightComponent

    class ASkyLight {
        -USkyLightComponent* LightComponent
    }
```

### 后处理

```mermaid
classDiagram
    AActor o-- USceneComponent
    AActor <|-- ACameraActor
    ACameraActor *-- UCameraComponent : CameraComponent
    USceneComponent <|-- UCameraComponent
    UCameraComponent o-- FPostProcessSettings : PostProcessSettings
    AActor <|-- ABrush
    ABrush <|-- AVolume
    AVolume <|-- APostProcessVolume
    APostProcessVolume *-- FPostProcessSettings : Settings
    class APostProcessVolume {
        +FPostProcessSettings Settings
    }


    class ACameraActor {
        -UCameraComponent* CameraComponent
        -USceneComponent* SceneComponent
    }
    
    class UCameraComponent {
        +ECameraProjectionMode::Type ProjectionMode
        +float FieldOfView
        +float AspectRatio
        +float OrthoWidth
        +float PostProcessBlendWeight
        +FPostProcessSettings PostProcessSettings;
    }

    class FPostProcessSettings {

    }
```

### 全局光照

### 大气与雾气

详见[官方文档](https://dev.epicgames.com/documentation/unreal-engine/environmental-light-with-fog-clouds-sky-and-atmosphere-in-unreal-engine)

全局高度雾：

```mermaid
classDiagram
    AActor <|-- AInfo
    AActor o-- USceneComponent
    USceneComponent <|-- UExponentialHeightFogComponent
    AInfo <|-- AExponentialHeightFog
    AExponentialHeightFog *-- UExponentialHeightFogComponent : Component

    class AExponentialHeightFog {
        -UExponentialHeightFogComponent* Component
    }
```

局部体积雾：

```mermaid
classDiagram
    AActor <|-- AInfo
    AActor o-- USceneComponent
    USceneComponent <|-- ULocalFogVolumeComponent
    AInfo <|-- ALocalFogVolume
    ALocalFogVolume *-- ULocalFogVolumeComponent : LocalFogVolumeVolume

    class ALocalFogVolume {
        -ULocalFogVolumeComponent* LocalFogVolumeVolume
    }
```

大气：

```mermaid
classDiagram
    AActor <|-- AInfo
    AActor o-- USceneComponent
    USceneComponent <|-- USkyAtmosphereComponent
    AInfo <|-- ASkyAtmosphere
    ASkyAtmosphere *-- USkyAtmosphereComponent : SkyAtmosphereComponent

    class ASkyAtmosphere {
        -USkyAtmosphereComponent* SkyAtmosphereComponent
    }
```

体积云：

```mermaid
classDiagram
    AActor <|-- AInfo
    AActor o-- USceneComponent
    USceneComponent <|-- UVolumetricCloudComponent
    AInfo <|-- AVolumetricCloud
    AVolumetricCloud *-- UVolumetricCloudComponent : VolumetricCloudComponent

    class AVolumetricCloud {
        -UVolumetricCloudComponent* VolumetricCloudComponent
    }
```

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

    class UCameraComponent {
        +void GetCameraView(float, FMinimalViewInfo&)*
        +void AddOrUpdateBlendable(...)
        +void RemoveBlendable(...)
    }

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

    class FMinimalViewInfo {
        +FVector Location
        +FRotator Rotation
        +float FOV
        +float PerspectiveNearClipPlane
        +float AspectRatio
        +ECameraProjectionMode ProjectionMode
        +float PostProcessBlendWeight
        +FPostProcessSettings PostProcessSettings
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

在游戏的主线程，通过 `UCameraComponent::GetCameraView()` 函数，获取当前相机的 `FMinimalViewInfo` 信息，它是游戏线程相机属性的快照。



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

## 延迟渲染管线与着色器

```mermaid
classDiagram
    IPSOCollector <|.. FMeshPassProcessor
    FMeshPassProcessor <|-- FBasePassMeshProcessor

    class FDeferredShadingSceneRenderer {

    }
```

### Shader

## RDG & RHI
