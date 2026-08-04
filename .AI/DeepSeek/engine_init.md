# 引擎初始化

> 本文档基于 Unreal Engine 5.5.4 源码分析，核心代码文件：
> - `Runtime/Launch/Public/LaunchEngineLoop.h`
> - `Runtime/Launch/Private/LaunchEngineLoop.cpp`
> - `Runtime/Launch/Private/Launch.cpp`
> - `Runtime/Launch/Private/Windows/LaunchWindows.cpp`

## 总体流程概览

引擎的整个生命周期由全局对象 `GEngineLoop`（类型 `FEngineLoop`）驱动。从操作系统入口到进入主循环，经历了以下关键阶段：

```
WinMain / main()
  └─ GuardedMain()
       ├─ EnginePreInit() → GEngineLoop.PreInit()
       │   ├─ PreInitPreStartupScreen()   // 启动画面前
       │   └─ PreInitPostStartupScreen()  // 启动画面后
       ├─ EditorInit() 或 EngineInit()
       │   └─ GEngineLoop.Init()
       └─ while(!IsEngineExitRequested()) EngineTick()  // 主循环
```

---

## 第一阶段：平台入口 → `GuardedMain`

### Windows 平台入口 (`LaunchWindows.cpp`)

```
WinMain()
  └─ LaunchWindowsStartup()
       ├─ SetupWindowsEnvironment()
       │   ├─ 设置 CRT 参数验证回调
       │   └─ 配置调试选项
       ├─ 处理命令行参数 (ProcessCommandLine)
       ├─ 设置进程亲和性
       ├─ 决定异常处理层级
       │   ├─ 无调试器/debug → GuardedMain() 直接调用
       │   └─ 有 SEH 保护 → GuardedMainWrapper() → GuardedMain()
       └─ LaunchWindowsShutdown() → AppExit()
```

关键细节：
- `NvOptimusEnablement` / `AmdPowerXpressRequestHighPerformance` 导出符号确保使用高性能 GPU
- `D3D12SDKVersion` / `D3D12SDKPath` 导出符号用于 D3D12 重分发
- SEH (`__try/__except`) 在最外层捕获崩溃，生成崩溃报告

### `GuardedMain` 入口 (`Launch.cpp`)

```cpp
int32 GuardedMain(const TCHAR* CmdLine)
```

核心流程：
1. **`FCoreDelegates::GetPreMainInitDelegate().Broadcast()`** — 最早期的初始化回调
2. **`EngineLoopCleanupGuard`** — RAII 守卫，确保 `EngineExit()` 在作用域退出时被调用（非内嵌模式下）
3. **`EnginePreInit(CmdLine)`** — 调用 `GEngineLoop.PreInit()`
4. **`EditorInit(GEngineLoop)` 或 `EngineInit()`** — 根据 `GIsEditor` 选择初始化路径
5. **主循环** — `while(!IsEngineExitRequested()) EngineTick()`
6. **`EditorExit()`** — 编辑器模式下的额外清理

---

## 第二阶段：`PreInit` 阶段

`PreInit` 是 `PreInitPreStartupScreen` 和 `PreInitPostStartupScreen` 的包装器：

```cpp
int32 FEngineLoop::PreInit(const TCHAR* CmdLine)
{
    ArraySlackTrackInit();           // 开启数组松弛追踪
    rv1 = PreInitPreStartupScreen(CmdLine);
    if (rv1 != 0) return rv1;
    rv2 = PreInitPostStartupScreen(CmdLine);
    if (rv2 != 0) return rv2;
    return 0;
}
```

---

## 第三阶段：`PreInitPreStartupScreen` — 启动画面之前

这是引擎初始化中最大、最复杂的函数（约 1600 行）。按执行顺序，核心步骤如下：

### 3.1 超早期环境设置

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | `FDelayedAutoRegisterHelper::RunAndClearDelayedAutoRegisterDelegates(StartOfEnginePreInit)` | 运行延迟注册 |
| 2 | 设置 `GLog` 主线程 | 确保日志初始化在主线程 |
| 3 | 命令行选项解析 | `statnamedevents`, `verbosenamedevents` |
| 4 | 设置 DebugGame 标志 | `FApp::SetDebugGame(true)` |
| 5 | 注册 Ctrl-C 处理器 (Windows) | `SetGracefulTerminationHandler()` |
| 6 | 初始化内嵌通信 | `FEmbeddedCommunication::Init()` |
| 7 | TLS 缓存设置 | `FMemory::SetupTLSCachesOnCurrentThread()` |
| 8 | 切换到可执行文件目录 | `SetCurrentWorkingDirectoryToBaseDir()` |
| 9 | **设置命令行** | `FCommandLine::Set(CmdLine)` |

### 3.2 游戏名称和项目路径

- `LaunchSetGameName()` — 解析命令行中的游戏名称
- 从环境变量 `UE-CmdLineArgs` 追加命令行参数
- 验证项目文件存在性
- 加载 PreInit DLL（非 Monolithic 构建）用于解密 pak 等

### 3.3 追踪和内存系统

| 步骤 | 操作 |
|------|------|
| Trace 初始化 | `FTraceAuxiliary::Initialize()` + `TryAutoConnect()` |
| LLM 初始化 | `FLowLevelMemTracker::Get().ProcessCommandLine()` |
| MemPro 初始化 | `FMemProProfiler::Init()` |
| TaggedStorage 初始化 | `FPlatformMisc::InitTaggedStorage(1024)` |

### 3.4 输出设备初始化

- 创建控制台输出设备 `GScopedLogConsole`
- 启用日志 backlog
- 初始化 stdout 设备（`-stdout` 参数）
- 设置 `GError` / `GWarn`
- 广播 `OnOutputDevicesInit`

### 3.5 命令行别名和文件重定向

- 从 `CommandLineAliases.ini` 加载命令行别名
- 处理 `-CmdLineFile=` 指定的命令行文件

### 3.6 I/O 和文件系统

| 步骤 | 操作 |
|------|------|
| `FIoDispatcher::Initialize()` | I/O 分发器初始化 |
| 平台文件覆盖 | `LaunchCheckForFileOverride()` — 支持 Pak/网络文件系统等 |
| 添加二进制搜索路径 | `AddExtraBinarySearchPaths()` |
| 文件管理器 | `IFileManager::Get().ProcessCommandLineOptions()` |
| 新异步 I/O | `FPlatformFileManager::Get().InitializeNewAsyncIO()` |

### 3.7 运行模式确定

这是关键阶段，通过命令行参数确定引擎运行模式（4 种互斥模式）：

1. **DedicatedServer** — `IsRunningDedicatedServer()` 预先判断
2. **Commandlet** — 检查以 `Commandlet` 结尾的参数或 `-RUN=` 开关
3. **Editor** — `UE_EDITOR` 构建下默认为编辑器（除非有 `-GAME` 参数）
4. **RegularClient** — 最后的默认模式

设置全局标志：`GIsClient`, `GIsServer`, `GIsEditor`, `PRIVATE_GIsRunningCommandlet`

### 3.8 线程池和任务系统

- `FTaskGraphInterface::Startup()` — 启动任务图系统
- 创建线程池：
  - Editor 模式：`GLargeThreadPool` + `GThreadPool`（`FQueuedThreadPoolWrapper`）
  - 其他模式：`GThreadPool`（`FQueuedLowLevelThreadPool`）
  - `GBackgroundPriorityThreadPool` — 后台线程池
- `FThreadStats::StartThread()` — 启动统计线程

### 3.9 模块加载：`LoadCoreModules` → `LoadPreInitModules`

**LoadCoreModules**: 加载 `CoreUObject` 模块

**LoadPreInitModules**: 按顺序加载以下模块
- `Engine`
- `Renderer`
- `AnimGraphRuntime`
- `SlateRHIRenderer`（非服务器）
- `Landscape`
- `RHICore`
- `RenderCore`
- `TextureCompressor`（含 EditorOnlyData）
- `Virtualization`（未烘焙数据）
- `AudioEditor`（Editor）

### 3.10 渲染系统 CVar 配置

- `InitializeRenderingCVarsCaching()`
- 应用多个 INI 配置段的 CVar 设置：
  - `RendererSettings`, `RendererOverrideSettings`, `StreamingSettings`
  - `GarbageCollectionSettings`, `NetworkSettings`
  - `CookerSettings`（Editor）
- 预加载分辨率设置 `UGameUserSettings::PreloadResolutionSettings()`

### 3.11 可伸缩性系统初始化

- `Scalability::InitScalabilitySystem()` — 初始化可伸缩性系统
- `UDeviceProfileManager::InitializeCVarsForActiveDeviceProfile()` — 设备配置
- `Scalability::LoadState()` — 加载可伸缩性状态
- 从 `ConsoleVariables.ini` 加载控制台变量

### 3.12 平台初始化

- `FPlatformMisc::PlatformInit()`
- `FPlatformApplicationMisc::Init()`
- `FPlatformMemory::Init()`

### 3.13 物理学初始化

- `InitGamePhys()` — 初始化物理引擎

### 3.14 文本本地化和 Slate

- 清理 Shader 工作目录
- `InitEngineTextLocalization()` — 引擎文本本地化
- `FSlateApplication::InitHighDPI()` — 高 DPI 初始化
- `UStringTable::InitializeEngineBridge()` — 字符串表
- 音频线程配置
- **Slate 应用创建**：
  - 非专用服务器且非 commandlet → `FPlatformSplash::Show()` + `FSlateApplication::Create()`
  - 否则 → `EKeys::Initialize()` + `FSlateApplication::InitializeCoreStyle()`

### 3.15 Shader 系统初始化

- `FShaderParametersMetadataRegistration::CommitAll()` — Shader 参数元数据
- `FShaderTypeRegistration::CommitAll()` — Shader 类型注册
- `FShaderParametersMetadata::InitializeAllUniformBufferStructs()` — Uniform Buffer 初始化

### 3.16 RHI 初始化

- `PreInitHMDDevice()` — HMD 设备预初始化
- **`RHIInit(bHasEditorToken)`** — RHI 初始化（选择 D3D12/Vulkan 等）
- `PipelineStateCache::Init()` — PSO 缓存
- `RenderUtilsInit()` — 渲染工具初始化
- `FShaderCodeLibrary::InitForRuntime()` — Shader 代码库（运行时）
- `FShaderPipelineCache::Initialize()` — PSO 管线缓存初始化（非编辑器）

### 3.17 DDC 和 Shader 编译

- `UE::DerivedData::GetCache()` — 派生数据缓存初始化
- `UE::DerivedData::GetBuild()` — 派生数据构建初始化
- 创建 `GDistanceFieldAsyncQueue`, `GCardRepresentationAsyncQueue`
- 创建 `GShaderCompilingManager` 和 `GShaderCompilerStats`
- `InitializeShaderTypes()` — 初始化 Shader 类型
- **`CompileGlobalShaderMap(false)`** — 编译全局 Shader Map（如果需要）

### 3.18 启动画面和渲染线程

- `CreateMoviePlayer()` — 创建影片播放器
- `FPreLoadScreenManager::Create()` — 创建预加载画面管理器
- **如果平台支持早期影片播放**：
  - `PostInitRHI()` — RHI 后初始化
  - `InitRenderingThread()` — 启动渲染线程
- **Slate 渲染器初始化**（非服务器）：
  - 创建 Slate 渲染器（NullRHI 或 SlateRHIRenderer）
  - `FSlateApplication::InitializeRenderer()`
  - `FEngineFontServices::Create()`
  - 加载 `PostSplashScreen` 阶段模块
  - 播放预加载画面

### 3.19 Verse VM 启动

- `Verse::VerseVM::Startup()` 或 `verse::FExecutionContext::Create()`

---

## 第四阶段：`PreInitPostStartupScreen` — 启动画面之后

### 4.1 模式最终确定

- 如果不是 Commandlet 模式，完成 RegularClient 检查
- 对可能不认识的 commandlet 给予警告

### 4.2 影片和预加载画面播放

- 如果影片播放器已启用且有早期启动影片：
  - `GetMoviePlayer()->Initialize()` + `PlayEarlyStartupMovies()`
- 否则播放预加载画面 `FPreLoadScreenManager::PlayFirstPreLoadScreen()`

### 4.3 Pak 挂载和 Shader 库打开

- 挂载早期启动画面期间安装的 Pak 文件
- `FShaderCodeLibrary::OpenLibrary()` — 打开游戏 Shader 库
- `FShaderPipelineCache::OpenPipelineFileCache()` — 打开 PSO 缓存文件

### 4.4 游戏文本本地化和包初始化

- `InitGameTextLocalization()`
- `FPackageName::RegisterShortPackageNamesForUObjectModules()` — 注册短包名
- 加载 `AssetRegistry` 模块
- `IPackageResourceManager::Initialize()` — 包资源管理器
- `IBulkDataRegistry::Initialize()` (Editor)

### 4.5 UObject 系统初始化

- `FDelayedAutoRegisterHelper::RunAndClearDelayedAutoRegisterDelegates(PreObjectSystemReady)`
- **`ProcessNewlyLoadedUObjects()`** — 处理所有新加载的 UObject，初始化 CDO
- `FDelayedAutoRegisterHelper::RunAndClearDelayedAutoRegisterDelegates(ObjectSystemReady)`

### 4.6 默认材质和流式系统

- `UMaterialInterface::InitDefaultMaterials()` — 初始化默认材质
- `UMaterialInterface::AssertDefaultMaterialsExist()` — 断言默认材质存在
- `UMaterialInterface::AssertDefaultMaterialsPostLoaded()` — 断言后加载完成
- `IStreamingManager::Get()` — 初始化纹理流式系统

### 4.7 启动模块加载：`LoadStartupCoreModules` → `LoadStartupModules`

**LoadStartupCoreModules**:
- `Core`, `Networking`
- `LiveCoding`（开发者构建）
- `Messaging`
- `MRMesh`（非服务器）
- `UnrealEd`, `LandscapeEditorUtilities`, `SubobjectDataInterface`（Editor）
- `SlateCore`, `Slate`, `SlateReflector`
- `UMG`, `EditorStyle`
- `MessageLog`, `CollisionAnalyzer`
- `FunctionalTesting`
- `BehaviorTreeEditor`, `GameplayTasksEditor`
- `AudioEditor`, `StringTableEditor`, `VREditor`
- `Overlay`, `MediaAssets`
- `ClothingSystemRuntimeNv`, `ClothingSystemEditor`
- `WorldPartitionEditor`, `PacketHandler`, `NetworkReplayStreaming`, `MassEntity`

**LoadStartupModules** (按加载阶段依次加载):
- `PreDefault` 阶段模块
- `Default` 阶段模块
- `PostDefault` 阶段模块

> 有一个问题，Plugin 模块和项目的 Game 模块何时加载？

### 4.8 非引擎路径初始化

如果不使用 `WITH_ENGINE`，则执行简化版初始化：
- `InitEngineTextLocalization()` + `InitGameTextLocalization()`
- `FPackageLocalizationManager::InitializeFromDefaultCache()`
- `FPlatformApplicationMisc::PostInit()`

### 4.9 渲染线程启动

如果平台不支持早期影片播放，此时启动：
- `PostInitRHI()`
- `InitRenderingThread()`

### 4.10 引擎加载画面

- 如果注册了 `EngineLoadingScreen` → `FPreLoadScreenManager::PlayFirstPreLoadScreen()`
- 否则播放影片播放器的影片

### 4.11 Commandlet 特殊路径

如果是 Commandlet 模式：
- 查找 Commandlet 类
- 设置 `GIsClient`/`GIsServer`/`GIsEditor`（来自 Commandlet CDO）
- 创建 `GEngine`（EditorEngine 或 GameEngine）
- 调用 `GEngine->Init(this)`
- 广播 `OnPostEngineInit`
- 创建并执行 Commandlet
- Commandlet 执行完成后返回，引擎不进入主循环

### 4.12 收尾工作

- `FCoreUObjectDelegates` 注册 GC 回调
- `PendingCleanupObjects = nullptr`
- 加载 `ProfileVisualizer`, `ProfilerService`
- 初始化高分辨率截图系统
- 预缓存全局 Shader 的 Compute PSO
- `UE::RenderCommandPipe::Initialize()`
- 运行冒烟测试
- `PreInitContext.Cleanup()` — 释放 SlateRenderer 和 SlowTask

---

## 第五阶段：`Init` / `EditorInit` 阶段

### `FEngineLoop::Init()`（游戏/服务器路径）

```
Init()
├─ 创建 GEngine（UGameEngine 子类）
│   ├─ 从 INI 读取 GameEngine 类名
│   └─ NewObject<UEngine>(...)
├─ GEngine->ParseCommandline()
├─ InitTime()  // 初始化基准测试变量
├─ GEngine->Init(this)  // 核心引擎初始化
├─ FCoreDelegates::OnPostEngineInit.Broadcast()
├─ 加载 PostEngineInit 阶段模块
├─ SetEngineStartupModuleLoadingComplete()
├─ GEngine->Start()
│   ├─ 启动所有 World
│   └─ 初始化游戏实例等
├─ 等待引擎加载画面完成
├─ FTraceAuxiliary::EnableCommandlineChannels()
├─ 初始化 Media 模块
├─ 加载 AutomationWorker, AutomationController
├─ GIsRunning = true
├─ FThreadHeartBeat::Get().Start()  // 心跳线程
├─ FCoreDelegates::OnFEngineLoopInitComplete.Broadcast()
└─ FExternalProfiler 初始化
```

### `EditorInit(GEngineLoop)`（编辑器路径）

编辑器初始化通过 `EditorInit(GEngineLoop)` 而不是 `GEngineLoop.Init()`：

```cpp
// 在 GuardedMain 中：
if (GIsEditor)
    ErrorLevel = EditorInit(GEngineLoop);
else
    ErrorLevel = EngineInit();  // 即 GEngineLoop.Init()
```

`EditorInit` 会额外初始化 UnrealEd 系统，创建 `UUnrealEdEngine` 而不是 `UGameEngine`。

### `InitTime()` 详细

- 设置 `FApp::CurrentTime`, `MaxFrameCounter`, `MaxTickTime`, `TotalTickTime`
- 解析基准测试参数：
  - `-SECONDS=N` 或 `-BENCHMARKSECONDS=N` — 最大 Tick 时间
  - `-FPS=N` — 固定帧率
  - `MaxFrameCounter = MaxTickTime / FixedDeltaTime`

---

## 第六阶段：模块加载阶段汇总

Unreal Engine 定义了以下模块加载阶段（`ELoadingPhase`），在初始化过程的不同时间点触发：

| 阶段 | 触发时机 | 所在函数 |
|------|----------|----------|
| `PostSplashScreen` | 启动画面显示后 | `PreInitPreStartupScreen` |
| `PreEarlyLoadingScreen` | 早期加载画面前 | `PreInitPostStartupScreen` |
| `PreLoadingScreen` | 加载画面前 | `PreInitPostStartupScreen` |
| `PreDefault` | 默认模块前 | `LoadStartupModules` |
| `Default` | 默认模块 | `LoadStartupModules` |
| `PostDefault` | 默认模块后 | `LoadStartupModules` |
| `PostEngineInit` | 引擎初始化后 | `Init` |

---

## 命令行参数参考

| 参数 | 说明 |
|------|------|
| `-GAME` | 强制以游戏模式运行（覆盖编辑器默认） |
| `-SERVER` | 以专用服务器模式运行 |
| `-RUN=XXX` | 运行指定 Commandlet |
| `-stdout` | 将日志输出到标准输出 |
| `-BENCHMARK` | 基准测试模式 |
| `-Deterministic` | 确定性模式（固定时间步长 + 固定种子） |
| `-UseFixedTimeStep` | 使用固定时间步长 |
| `-FPS=N` | 设置固定 FPS |
| `-SECONDS=N` | 最大 Tick 时间（秒） |
| `-NoLogThread` | 禁用日志线程 |
| `-RenderOffScreen` | 离屏渲染 |
| `-nohmd` | 禁用 HMD |
| `-noinnerexception` | 禁用内部异常处理 |
| `-waitforattach` | 启动时等待调试器附加 |
| `-crashreports` | 始终报告崩溃 |
| `-unattended` | 无人值守模式（无错误对话框） |
