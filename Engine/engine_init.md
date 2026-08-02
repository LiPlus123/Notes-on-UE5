# 引擎初始化

> 本文档基于 Unreal Engine 5.5.4 源码分析，核心代码文件：
> - `Runtime/Launch/Public/LaunchEngineLoop.h`
> - `Runtime/Launch/Private/LaunchEngineLoop.cpp`
> - `Runtime/Launch/Private/Launch.cpp`
> - `Runtime/Launch/Private/Windows/LaunchWindows.cpp`

初始化又可以分为 `PreInit` 和 `Init` 两个阶段：
1. `PreInit` 阶段又分为两个阶段：
   1. `FEngineLoop::PreInitPreStartupScreen`：完成启动画面之前的前置初始化工作
   2. `FEngineLoop::PreInitPostStartupScreen`：完成启动画面之后的前置初始化工作
2. `Init` 阶段根据环境的不同，执行 `EngineInit` 或 `EditorInit`。`EditorInit` 有额外的 UnrealEd 的初始化流程。

## `PreInit` 阶段

### `PreInitPreStartupScreen` 阶段

### `PreInitPostStartupScreen` 阶段

### 命令行

## `EngineInit` 阶段

## `EditorInit` 阶段