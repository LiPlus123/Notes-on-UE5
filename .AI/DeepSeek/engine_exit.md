# 引擎退出

> 本文档基于 Unreal Engine 5.5.4 源码分析，核心代码文件：
> - `Runtime/Launch/Public/LaunchEngineLoop.h`
> - `Runtime/Launch/Private/LaunchEngineLoop.cpp`
> - `Runtime/Launch/Private/Launch.cpp`
> - `Runtime/Launch/Private/Windows/LaunchWindows.cpp`

## 总体概览

引擎退出是一个分层有序的关闭过程。退出可以从多个层面触发：

```
触发层:
  1. 用户操作（点击关闭按钮 / Alt+F4）
  2. 控制台命令（"quit" / "exit"）
  3. 系统信号（Ctrl+C / SIGTERM）
  4. 程序内部（基准测试到期 / 致命错误 / Commandlet 完成）

执行层:
  EngineExit() → GEngineLoop.Exit()
                    ├─ GEngine->PreExit()
                    ├─ 各子系统关闭
                    └─ AppPreExit()
                              │
  GuardedMain() EngineLoopCleanupGuard (RAII 析构)
  └─ EngineExit()
                              │
  LaunchWindowsShutdown() → AppExit()
```

---

## 触发机制

### 1. `RequestEngineExit()` —— 请求退出

```cpp
// 设置标志，下一帧 Tick 时处理
RequestEngineExit(TEXT("reason string"));
```

这设置了 `GIsRequestingExit` 标志。在 Tick 开始时 `BeginExitIfRequested()` 会检查此标志并调用 `ProcessDeferredExitIfRequested()`。

### 2. 退出触发点

| 触发路径 | 代码位置 | 说明 |
|----------|----------|------|
| 主循环终止 | `GuardedMain` 的 `while(!IsEngineExitRequested())` 条件 | 主循环结束，`EngineLoopCleanupGuard` 析构调用 `EngineExit()` |
| Tick 中退出 | `BeginExitIfRequested()` | 下一帧开始时处理 |
| 直接调用 | `FPlatformMisc::RequestExit()` | 立即退出进程（绕过正常清理） |
| 快速退出 | `FPlatformMisc::RequestExitWithStatus()` | Commandlet 或性能要求时使用 |

### 3. 控制台命令退出

"quit" 或 "exit" 命令被放入 `GEngine->DeferredCommands`：
```cpp
GEngine->DeferredCommands.Emplace(TEXT("Quit force"));
```
在 Tick 末期的 `GEngine->TickDeferredCommands()` 中处理，最终调用 `RequestEngineExit()`。

---

## 退出流程：`FEngineLoop::Exit()`

以下是 `Exit()` 函数中子系统关闭的完整顺序：

### 第一阶段：立即清理

| 步骤 | 操作 | 代码位置 |
|------|------|----------|
| 1 | **ClearPendingCleanupObjects()** | `Exit()` 开头 |
| | └─ `FlushRenderingCommands()` 确保渲染线程已完成上一帧 | |
| | └─ `delete PendingCleanupObjects` | |
| 2 | `GIsRunning = 0` | 标记引擎停止运行 |
| 3 | `GLogConsole = nullptr` | 释放控制台引用 |

### 第二阶段：预加载/加载画面关闭

| 步骤 | 操作 |
|------|------|
| 4 | `IInstallBundleManager::InstallBundleCompleteDelegate.RemoveAll()` |
| 5 | `FPreLoadScreenManager::Destroy()` — 销毁预加载画面管理器 |
| 6 | `FVisualLogger::Get().TearDown()` — 可视化日志器关闭 |
| 7 | `FAssetCompilingManager::Get().Shutdown()` — 资产编译管理器关闭 |

### 第三阶段：服务和引擎关闭

| 步骤 | 操作 |
|------|------|
| 8 | `delete EngineService` — 引擎服务 |
| 9 | `delete TraceService` — 追踪服务 |
| 10 | `SessionService->Stop()` + `Reset()` — 会话服务 |
| 11 | `delete GDistanceFieldAsyncQueue` — 距离场异步队列 |
| 12 | `delete GCardRepresentationAsyncQueue` — Mesh Card 异步队列 |

### 第四阶段：引擎预退出

```cpp
if (GEngine != nullptr)
{
    GEngine->PreExit();                      // 引擎预退出
}
GEngine->ReleaseAudioDeviceManager();       // 释放音频设备管理器
```

`GEngine->PreExit()` 执行：
- 广播 `OnPreExit` 代理
- 停止所有 World
- 清理 GameInstance
- 销毁 Viewport

### 第五阶段：异步加载和流式处理清理

| 步骤 | 操作 |
|------|------|
| 13 | `FlushAsyncLoading()` 或 `CancelAsyncLoading()` | 取决于 `s.FlushStreamingOnExit` 配置 |
| 14 | `SetAsyncLoadingAllowed(false)` | 禁止新的异步加载 |
| 15 | `UTexture2D::CancelPendingTextureStreaming()` | 取消待处理的纹理流式传输 |
| 16 | `IStreamingManager::BlockTillAllRequestsFinished()` | 等待所有流式请求完成 |
| 17 | `FAudioDeviceManager::Shutdown()` | 关闭音频设备管理器 |

### 第六阶段：UI 系统关闭

| 步骤 | 操作 |
|------|------|
| 18 | **`FSlateApplication::Shutdown()`** | 关闭所有窗口，销毁 Slate 应用 |
| 19 | `FEngineFontServices::Destroy()` | 销毁引擎字体服务 |

### 第七阶段：模块和 PSO 关闭

| 步骤 | 操作 |
|------|------|
| 20 | `FModuleManager::UnloadModule("AssetTools")` | Editor |
| 21 | `FModuleManager::UnloadModule("WorldBrowser")` |
| 22 | `ClearMaterialPSORequests()` | 清除待处理的 PSO 编译请求 |
| 23 | `PipelineStateCache::WaitForAllTasks()` | 等待所有 PSO 任务完成 |

### 第八阶段：`AppPreExit()` — 应用预退出

```cpp
AppPreExit();
```

核心操作：
- `FCoreDelegates::OnPreExit.Broadcast()` — 预退出回调
- 关闭 DDC Pak 文件（如果正在创建 Pak）
- `FCoreDelegates::OnExit.Broadcast()` — 退出回调
- `GFrameNumberRenderThread = MAX_uint32` — 标记无更多帧，防死锁
- `GIOThreadPool->Destroy()` — 销毁 I/O 线程池
- `Verse::VerseVM::Shutdown()` — 关闭 Verse VM
- 删除 `GShaderCompilingManager`, `GShaderCompilerStats`, `GODSCManager`

### 第九阶段：物理引擎关闭

| 步骤 | 操作 |
|------|------|
| 24 | `TermGamePhys()` | 终止游戏物理引擎 |

### 第十阶段：资源管理器和 Shader 系统关闭

| 步骤 | 操作 |
|------|------|
| 25 | `IBulkDataRegistry::Shutdown()` | Editor |
| 26 | `IPackageResourceManager::Shutdown()` | 包资源管理器 |
| 27 | `FModuleManager::UnloadModule("AssetRegistry")` | 卸载资产注册表模块 |
| 28 | **`ShutdownRenderingThread()`** | 停止渲染线程 |
| 29 | `FShaderPipelineCache::Shutdown()` | 关闭 PSO 管线缓存 |
| 30 | `ShutdownGlobalShaderMap()` | 释放全局 Shader Map |
| 31 | `FShaderCodeLibrary::Shutdown()` | 关闭 Shader 代码库 |

### 第十一阶段：I/O 分发器关闭

| 步骤 | 操作 |
|------|------|
| 32 | `UE::DerivedData::IoStore::TearDownIoDispatcher()` | DDC I/O |
| 33 | `FIoDispatcher::Shutdown()` | I/O 分发器 |

### 第十二阶段：虚拟化系统关闭

| 步骤 | 操作 |
|------|------|
| 34 | `UE::Virtualization::Shutdown()` | 虚拟化系统 (Editor) |

### 第十三阶段：模块卸载

| 步骤 | 操作 |
|------|------|
| 35 | `IHotReloadInterface::SaveConfig()` | 保存热重载配置 |
| 36 | **`FModuleManager::Get().UnloadModulesAtShutdown()`** | 按加载逆序调用所有模块的 `ShutdownModule()` |

### 第十四阶段：流式系统和线程池关闭

| 步骤 | 操作 |
|------|------|
| 37 | `IStreamingManager::Shutdown()` | 流式管理器 |
| 38 | `FRealtimeGPUProfiler::SafeRelease()` | GPU 分析器 |
| 39 | `DestroyMoviePlayer()` | 销毁影片播放器 |
| 40 | `FThreadStats::StopThread()` | 停止统计线程 |
| 41 | `GThreadPool->Destroy()` | 销毁线程池 |
| 42 | `GLargeThreadPool->Destroy()` | 销毁大线程池 (Editor) |
| 43 | `GBackgroundPriorityThreadPool->Destroy()` | 销毁后台线程池 |

### 第十五阶段：RHI 和任务图关闭

| 步骤 | 操作 |
|------|------|
| 44 | `RHIExit()` | 退出 RHI 子系统 |
| 45 | **`FTaskGraphInterface::Shutdown()`** | 关闭任务图系统 |
| 46 | `FPlatformMisc::ShutdownTaggedStorage()` | 关闭 TaggedStorage |
| 47 | `FWindowsPlatformPerfCounters::Shutdown()` | Editor Windows |
| 48 | `FFrameProProfiler::TearDown()` | FramePro 分析器关闭 |

---

## 退出流程：`AppPreExit()` — 应用预退出

`AppPreExit` 在 `FEngineLoop::Exit()` 的第八阶段被调用，也在 `GuardedMain` 的退出路径中被调用。

```cpp
void FEngineLoop::AppPreExit()
{
    FCoreDelegates::OnPreExit.Broadcast();    // 预退出广播
    
    // DDC Pak 文件关闭 (CreatePak 命令)
    
    FCoreDelegates::OnExit.Broadcast();       // 退出广播
    
    GFrameNumberRenderThread = MAX_uint32;     // 防死锁
    
    GIOThreadPool->Destroy();                 // IO 线程池
    
    Verse::VerseVM::Shutdown();               // Verse VM
    
    delete GShaderCompilingManager;
    delete GShaderCompilerStats;
    delete GODSCManager;
}
```

---

## 退出流程：`AppExit()` — 最终退出

`AppExit` 是最后被调用的函数，在 `LaunchWindowsShutdown()` 中：

```cpp
void FEngineLoop::AppExit()
{
    // 使用 bCalledOnce 保证只执行一次
    UE_LOG(LogExit, Log, TEXT("Exiting."));
    
    FPlatformApplicationMisc::TearDown();      // 平台应用层清理
    FPlatformMisc::PlatformTearDown();         // 平台清理
    
    GConfig->Exit(); delete GConfig;           // 配置系统退出
    GLog->TearDown();                          // 日志系统退出
    
    FTextLocalizationManager::TearDown();      // 文本本地化
    FInternationalization::TearDown();         // 国际化
    
    FTraceAuxiliary::Shutdown();              // Trace 系统退出
}
```

---

## 退出流程：Windows 完整路径

```
Windows 正常退出:
  ┌─ 用户关闭窗口 / Alt+F4
  ├─ FWndProc → FSlateApplication::OnWindowClose()
  ├─ RequestEngineExit("SlateWindowClose")
  ├─ 下一帧 BeginExitIfRequested()
  ├─ while(!IsEngineExitRequested()) 条件不满足 → 主循环退出
  ├─ EditorExit() (如果是编辑器)
  ├─ EngineLoopCleanupGuard 析构 → EngineExit()
  │   └─ GEngineLoop.Exit()
  │       ├─ [15 个阶段的子系统关闭]
  │       └─ AppPreExit()
  ├─ LaunchWindowsShutdown()
  │   └─ AppExit()
  └─ return ErrorLevel → 进程退出

崩溃/致命错误退出:
  ├─ ReportCrash() / GError->HandleError()
  ├─ LaunchStaticShutdownAfterError()
  │   └─ TermGamePhys()
  ├─ FPlatformMallocCrash::Get().PrintPoolsUsage()
  └─ FPlatformMisc::RequestExit(true, ...)

Commandlet 退出:
  ├─ Commandlet->Main() 返回
  ├─ RequestEngineExit("Commandlet finished")
  ├─ 返回 ErrorLevel
  ├─ GuardedMain 中检查 ErrorLevel
  └─ return ErrorLevel → 退出

Console (Ctrl+C) 退出:
  ├─ Windows Ctrl+C Handler
  ├─ FPlatformMisc::RequestExit(false, "Ctrl+C")
  ├─ 下一帧 BeginExitIfRequested()
  └─ 正常退出流程
```

---

## 关键退出代理 (Delegates)

| 代理 | 调用位置 | 说明 |
|------|----------|------|
| `FCoreDelegates::OnPreExit` | `AppPreExit()` | 在核心退出流程之前广播 |
| `FCoreDelegates::OnExit` | `AppPreExit()` | 核心退出时广播 |
| `FCoreDelegates::OnBeginFrame` | `Tick()` 第 3 步 | 每帧开始（退出帧也会触发） |
| `FCoreDelegates::OnEndFrame` | `Tick()` 第 29 步 | 每帧结束（退出帧也会触发） |

---

## RAII 退出保障

```cpp
// GuardedMain 中的 CleanupGuard 确保 EngineExit 在函数退出时被调用
struct EngineLoopCleanupGuard 
{ 
    ~EngineLoopCleanupGuard()
    {
        if (!GUELibraryOverrideSettings.bIsEmbedded)
        {
            EngineExit();  // GEngineLoop.Exit()
        }
    }
} CleanupGuard;
```

这意味着即使初始化过程中出错（通过 `return ErrorLevel` 提前返回），`EngineExit()` 也会被调用，确保所有已初始化的子系统得到清理。

---

## 内嵌模式退出

当引擎以内嵌模式运行时（`GUELibraryOverrideSettings.bIsEmbedded == true`）：

- `EngineLoopCleanupGuard` 不会调用 `EngineExit()`
- 引擎的 Tick 由外部应用驱动
- 退出由外部应用负责调用清理函数
- `FEmbeddedCommunication::AllowSleep()` 允许引擎休眠
