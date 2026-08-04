# 引擎主循环

> 本文档基于 Unreal Engine 5.5.4 源码分析，核心代码文件：
> - `Runtime/Launch/Public/LaunchEngineLoop.h`
> - `Runtime/Launch/Private/LaunchEngineLoop.cpp`
> - `Runtime/Launch/Private/Launch.cpp`
> - `Runtime/Launch/Private/Windows/LaunchWindows.cpp`

## 总体概览

引擎初始化完成后，进入主循环。主循环由 `GuardedMain` 中的 while 循环驱动：

```cpp
// Launch.cpp GuardedMain()
while (!IsEngineExitRequested())
{
    EngineTick();  // → GEngineLoop.Tick()
}
```

每帧调用一次 `FEngineLoop::Tick()`。该函数是整个引擎的心跳，协调游戏线程（Game Thread）、渲染线程（Render Thread）和 RHI 线程之间的工作。

---

## Tick 函数的帧流程图解

```
一帧边界
  │
  ├─ TRACE_BEGIN_FRAME(TraceFrameType_Game)
  │
  ├─ 1. 帧前准备
  │   ├─ BeginExitIfRequested()        — 检查是否需要退出
  │   ├─ FThreadHeartBeat::HeartBeat() — 心跳信号
  │   ├─ TickHotfixables()             — 热修复 Tick
  │   └─ LatchRenderThreadConfiguration() — 更新渲染线程配置
  │
  ├─ 2. 影片阻塞检查
  │   └─ FMoviePlayerProxy::BlockingForceFinished()
  │
  ├─ 3. 外部性能分析器同步
  │   └─ FExternalProfiler::FrameSync()
  │
  ├─ 4. 帧统计和命名事件
  │   ├─ FPlatformMisc::BeginNamedEventFrame()
  │   └─ IConsoleManager::CallAllConsoleVariableSinks() — CVar 变更回调
  │
  ├─ 5. 帧时间管理
  │   ├─ GEngine->UpdateTimeAndHandleMaxTickRate()
  │   │   ├─ 更新 FApp::CurrentTime 和 FApp::DeltaTime
  │   │   ├─ 计算帧时间
  │   │   └─ 限制最大帧率（等待以维持目标 FPS）
  │   └─ GEngine->SetSimulationLatencyMarkerStart()
  │
  ├─ 6. 渲染线程：帧开始
  │   ├─ ENQUEUE_RENDER_COMMAND(BeginFrame)
  │   │   └─ BeginFrameRenderThread()
  │   │       ├─ GFrameNumberRenderThread++
  │   │       ├─ GPU Stats 开始帧
  │   │       └─ FCoreDelegates::OnBeginFrameRT.Broadcast()
  │   └─ FScene::StartFrame() — 所有场景帧开始
  │
  ├─ 7. 渲染命令管道开始录制
  │   └─ UE::RenderCommandPipe::StartRecording()
  │
  ├─ 8. 动态分辨率（非编辑器游戏）
  │   └─ EmitDynamicResolutionEvent(BeginFrame)
  │
  ├─ 9. 性能监控
  │   ├─ GEngine->TickPerformanceMonitoring()
  │   └─ ResetAsyncLoadingStats()
  │
  ├─ 10. 帧统计推进
  │   └─ FStats::AdvanceFrame(false, ...)
  │
  ├─ 11. FPS 计算
  │   └─ CalculateFPSTimings()
  │
  ├─ 12. 渲染线程：重置延迟更新
  │   └─ ENQUEUE_RENDER_COMMAND(ResetDeferredUpdates)
  │       ├─ FDeferredUpdateResource::ResetNeedsUpdate()
  │       └─ FlushPendingDeleteRHIResources_RenderThread()
  │
  ├─ 13. 平台消息泵
  │   └─ FPlatformApplicationMisc::PumpMessages(true)
  │       └─ 处理 Windows 消息/事件队列
  │
  ├─ 14. 空闲模式检查
  │   └─ ShouldUseIdleMode()
  │       ├─ 检查 t.IdleWhenNotForeground CVar
  │       ├─ 检查内嵌通信的唤醒状态
  │       └─ 如果 idle → 休眠 0.1s，跳过大部分 Tick
  │
  ├─ 15. Slate 输入处理（非空闲模式）
  │   ├─ SlateApp.PollGameDeviceState()    — 轮询输入设备
  │   └─ SlateApp.FinishedInputThisFrame() — 完成本帧输入处理
  │
  ├─ 16. ★ 游戏引擎核心 Tick ★
  │   └─ GEngine->Tick(DeltaTime, bIdleMode)
  │       ├─ 遍历所有 WorldContext
  │       │   └─ UWorld::Tick()
  │       │       ├─ 网络复制 (NetDriver::Tick)
  │       │       ├─ 物理模拟 (Physics)
  │       │       ├─ Actor Tick (遍历 Actor 列表)
  │       │       ├─ Component Tick
  │       │       ├─ 定时器 (TimerManager::Tick)
  │       │       └─ GC 条件检查
  │       ├─ GameInstance Tick
  │       ├─ GameViewport Tick
  │       └─ LocalPlayer Tick
  │
  ├─ 17. 加载画面等待
  │   ├─ FPreLoadScreenManager::WaitForEngineLoadingScreenToFinish()
  │   └─ 或 GetMoviePlayer()->WaitForMovieToFinish(true)
  │
  ├─ 18. 异步编译任务处理
  │   └─ FAssetCompilingManager::Get().ProcessAsyncTasks(true)
  │
  ├─ 19. Slate 平台和输入 Tick
  │   └─ FSlateApplication::Tick(ESlateTickType::PlatformAndInput)
  │
  ├─ 20. 并发 Slate Tick（可选）
  │   └─ 如果启用了 AsyncEndOfFrameTasks
  │       └─ 在后台线程并发执行 NetDriver::TickFlushAsyncEndOfFrame
  │
  ├─ 21. ★ Slate 应用程序 Tick ★
  │   ├─ FSlateApplication::Tick(TimeAndWidgets 或 Time)
  │   │   ├─ 更新 Widget 动画
  │   │   ├─ 处理 Widget 事件
  │   │   ├─ 绘制所有 Slate Widget
  │   │   │   └─ 生成渲染命令
  │   │   └─ 提交渲染批次
  │   └─ 等待并发任务完成（如果存在）
  │
  ├─ 22. 自动化工具 Tick
  │   ├─ AutomationController::Tick()
  │   └─ AutomationWorker::Tick()
  │
  ├─ 23. 渲染通道录制结束
  │   └─ UE::RenderCommandPipe::StopRecording()
  │
  ├─ 24. 渲染线程：场景结束帧
  │   └─ FScene::EndFrame() — 所有场景帧结束
  │
  ├─ 25. RHI Tick
  │   └─ RHITick(DeltaTime)
  │
  ├─ 26. 延迟任务处理
  │   ├─ FTSTicker::GetCoreTicker().Tick(DeltaTime) — CoreTicker
  │   ├─ FThreadManager::Get().Tick() — 线程管理器
  │   └─ GEngine->TickDeferredCommands() — 延迟控制台命令
  │
  ├─ 27. 帧同步
  │   └─ FFrameEndSync::Sync() — Game/Render/RHI 线程同步
  │
  ├─ 28. 媒体模块 TickPostRender
  │   └─ MediaModule->TickPostRender()
  │
  ├─ 29. 帧结束
  │   ├─ FCoreDelegates::OnEndFrame.Broadcast()
  │   ├─ GFrameCounter++
  │   └─ CSV 统计更新 (UpdateCoreCsvStats_EndFrame)
  │
  ├─ 30. 动态分辨率结束帧
  │   └─ EmitDynamicResolutionEvent(EndFrame)
  │
  ├─ 31. 渲染线程：帧结束
  │   └─ ENQUEUE_RENDER_COMMAND(EndFrame)
  │       └─ EndFrameRenderThread()
  │           ├─ GPU Stats 结束帧
  │           ├─ FCoreDelegates::OnEndFrameRT.Broadcast()
  │           └─ RHICmdList.EndFrame()
  │
  └─ TRACE_END_FRAME(TraceFrameType_Game)
```

---

## 各步骤详解

### 1. 帧前准备

```cpp
BeginExitIfRequested();              // 调用 ProcessDeferredExitIfRequested()
FThreadHeartBeat::Get().HeartBeat(true); // 防止看门狗超时
FGameThreadHitchHeartBeat::Get().FrameStart();
FPlatformMisc::TickHotfixables();    // LiveCoding 等热修复
LatchRenderThreadConfiguration();    // 将游戏线程配置同步到渲染线程
```

- `BeginExitIfRequested()` 检查 `IsEngineExitRequested()` 标志，如果已请求退出则立即处理并返回
- `FThreadHeartBeat` 是看门狗系统，防止长时间无响应
- `FGameThreadHitchHeartBeat` 检测游戏线程卡顿

### 2. 影片阻塞检查

```cpp
FMoviePlayerProxy::BlockingForceFinished();
ensure(!IsMoviePlayerEnabled() || GetMoviePlayer()->IsLoadingFinished());
```

确保在开始新帧之前，任何加载影片（由关卡加载等触发）已完成。

### 5. 帧时间管理 (UpdateTimeAndHandleMaxTickRate)

这是 FPS 控制的核心：

```cpp
GEngine->UpdateTimeAndHandleMaxTickRate();
```

- 更新 `FApp::CurrentTime` 和 `FApp::DeltaTime`
- **最大帧率限制**：根据 `t.MaxFPS` CVar 或 `-FPS=` 命令行参数，如果帧完成太快则主动 sleep
- **固定时间步长**：如果启用（`-UseFixedTimeStep`），`DeltaTime` 固定为 `FixedDeltaTime`
- **平滑帧率**：使用指数平滑来稳定 DeltaTime

### 6. 渲染线程帧开始

通过 `ENQUEUE_RENDER_COMMAND` 将命令发送到渲染线程：

```cpp
// 游戏线程
ENQUEUE_RENDER_COMMAND(BeginFrame)([CurrentFrameCounter](FRHICommandListImmediate& RHICmdList)
{
    BeginFrameRenderThread(RHICmdList, CurrentFrameCounter);
});
```

渲染线程执行：
- `TRACE_BEGIN_FRAME(TraceFrameType_Rendering)` — 渲染 Trace 帧边界
- `GFrameNumberRenderThread++` — 渲染帧计数
- GPU Stats 帧开始
- `FCoreDelegates::OnBeginFrameRT.Broadcast()` — 渲染线程开始回调

### 10. 帧统计推进

```cpp
FStats::AdvanceFrame(false, FStats::FOnAdvanceRenderingThreadStats::CreateStatic(&AdvanceRenderingThreadStatsGT));
```

- 推进 STATS 系统到新帧
- 处理上一帧的渲染线程统计数据

### 13. 平台消息泵

```cpp
FPlatformApplicationMisc::PumpMessages(true);
```

在 Windows 上处理 `PeekMessage`/`DispatchMessage`，确保窗口消息（鼠标、键盘、窗口事件等）得到处理。

### 14. 空闲模式

```cpp
bool bIdleMode = ShouldUseIdleMode();
if (bIdleMode)
{
    FPlatformProcess::Sleep(.1f);  // 让出 CPU
}
```

空闲模式条件（全部满足时）：
1. 是游戏（非编辑器/服务器）
2. 支持窗口模式
3. `t.IdleWhenNotForeground` 启用
4. 应用不在前台（无焦点）
5. 所有 World 的 AlwaysLoaded 关卡已加载完毕

在空闲模式下，大部分 Tick 逻辑被跳过。

### 15. Slate 输入处理

```cpp
SlateApp.PollGameDeviceState();       // 轮询键盘/鼠标/手柄等
SlateApp.FinishedInputThisFrame();    // 通知 Widget 输入已处理完毕
```

**这一步发生在 `GEngine->Tick()` 之前**，确保引擎 Tick 时输入数据已准备好。

### 16. ★ 游戏引擎核心 Tick

```cpp
GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);
```

这是引擎每帧中最核心的部分。内部调用链：

```
UGameEngine::Tick() / UEditorEngine::Tick()
  │
  ├─ 遍历所有 WorldContext
  │   └─ UWorld::Tick(ELevelTick::LevelTickAll, DeltaSeconds)
  │       │
  │       ├─ PrePhysics 阶段
  │       │   ├─ 网络复制接收 (NetDriver::TickDispatch)
  │       │   ├─ TickGroups 中的 Actor/Component Tick (TG_PrePhysics)
  │       │   └─ 粒子系统更新
  │       │
  │       ├─ DuringPhysics 阶段
  │       │   ├─ 物理模拟 (StartPhysics/EndPhysics)
  │       │   ├─ TickGroups 中的 Actor/Component Tick (TG_DuringPhysics)
  │       │   └─ 布料模拟
  │       │
  │       ├─ PostPhysics 阶段
  │       │   ├─ TickGroups 中的 Actor/Component Tick (TG_PostPhysics)
  │       │   ├─ 动画更新完成回调
  │       │   └─ 相机更新
  │       │
  │       ├─ 网络复制发送 (NetDriver::TickFlush)
  │       ├─ 定时器 (TimerManager::Tick)
  │       ├─ GC 条件检查
  │       └─ 关卡流式加载更新
  │
  ├─ UGameInstance::Tick()
  ├─ UGameViewportClient::Tick()
  └─ ULocalPlayer::Tick()
```

**TickGroup 顺序**（Actor/Component 按此顺序 Tick）：

| TickGroup | 描述 |
|-----------|------|
| `TG_PrePhysics` | 物理模拟之前的 Tick（如输入处理、AI 逻辑） |
| `TG_StartPhysics` | 物理开始时的 Tick |
| `TG_DuringPhysics` | 物理模拟异步进行中的 Tick |
| `TG_EndPhysics` | 物理结束时的 Tick |
| `TG_PostPhysics` | 物理之后的 Tick（如相机跟随、动画后处理） |
| `TG_PostUpdateWork` | 更新后的清理工作 |

### 21. ★ Slate 应用程序 Tick

```cpp
FSlateApplication::Get().Tick(ESlateTickType::TimeAndWidgets);
```

- **Time**: 更新所有 Widget 动画、过渡效果
- **Widgets**: 重绘所有需要更新的 Slate Widget
  - 遍历 Widget 层级树
  - 为脏 Widget 调用 `OnPaint()`
  - 生成 DrawElements 列表
  - 提交渲染批次到渲染线程

### 25. RHI Tick

```cpp
RHITick(FApp::GetDeltaTime());
```

- 更新 RHI 级别的帧统计
- 处理 GPU 内存预算
- 提交渲染命令到 GPU

### 26. 延迟任务处理

```cpp
FTSTicker::GetCoreTicker().Tick(FApp::GetDeltaTime());  // CoreTicker 回调
FThreadManager::Get().Tick();                            // 线程统计/清理
GEngine->TickDeferredCommands();                         // 延迟控制台命令执行
```

- `FTSTicker` 是核心延迟动作系统，允许注册延迟回调
- 控制台命令（如 `quit`）被放入 `GEngine->DeferredCommands`，在这里执行

### 27. 帧同步

```cpp
FFrameEndSync::Sync();
```

确保游戏线程和渲染线程在帧边界正确同步。与 `FRenderCommandFence` 配合使用。

### 29. 帧结束

```cpp
FCoreDelegates::OnEndFrame.Broadcast();  // 帧结束回调
GFrameCounter++;                          // 帧计数器递增
```

- `OnEndFrame` 是游戏线程帧结束的代理，许多系统订阅此事件做清理
- `GFrameCounter` 是全局帧计数器，用于各种系统

### 31. 渲染线程帧结束

```cpp
ENQUEUE_RENDER_COMMAND(EndFrame)([...](FRHICommandListImmediate& RHICmdList)
{
    EndFrameRenderThread(RHICmdList, CurrentFrameCounter);
});
```

渲染线程执行：
- `GPU_STATS_ENDFRAME` — GPU 统计帧结束
- `FCoreDelegates::OnEndFrameRT.Broadcast()` — 渲染线程结束回调
- `RHICmdList.EndFrame()` — RHI 帧提交，GPU 开始执行命令

---

## 线程协调

UE5 使用多线程架构，主循环中的线程协调如下：

```
游戏线程 (Game Thread)              渲染线程 (Render Thread)          RHI 线程
     │                                      │                              │
     ├─ Tick() 开始                         │                              │
     ├─ BeginFrame ──────────────────────→  ├─ BeginFrameRenderThread      │
     ├─ GEngine->Tick()                     │                              │
     │   └─ UWorld::Tick()                  │                              │
     ├─ Slate Tick (绘制)                   │                              │
     │   └─ 生成 DrawElements ───────────→  │                              │
     ├─ EndFrame ────────────────────────→  ├─ EndFrameRenderThread        │
     │                                      ├─ 渲染命令执行 ─────────────→ ├─ GPU 执行
     ├─ FFrameEndSync::Sync()  ←──────────→ │  ←──── 同步点 ────────────→ │
     ├─ GFrameCounter++                     │                              │
     └─ Tick() 结束                         │                              │
```

---

## 关键全局变量

| 变量 | 说明 |
|------|------|
| `GFrameCounter` | 全局帧计数器，游戏线程，每帧递增 |
| `GFrameCounterRenderThread` | 渲染线程的帧计数器副本 |
| `GFrameNumberRenderThread` | 渲染帧编号 |
| `GStartTime` | 引擎启动时间（秒） |
| `FApp::CurrentTime` | 当前帧时间（秒） |
| `FApp::DeltaTime` | 上一帧到当前帧的时间间隔 |
| `FApp::FixedDeltaTime` | 固定时间步长（启用时） |
| `GIsRunning` | 引擎是否正在运行 |
| `GUseThreadedRendering` | 是否使用多线程渲染 |

---

## 特殊情况

### 基准测试模式 (`-BENCHMARK`)

- 在 `MaxFrameCounter` 帧后，或 `TotalTickTime > MaxTickTime` 时自动退出
- 前 6 帧不计入 `TotalTickTime`（排除加载开销）

### 嵌入模式 (`bIsEmbedded`)

- 引擎作为库被外部应用使用
- 不进行消息泵（由外部应用处理）
- 可在空闲时主动休眠

### Commandlet 模式

- Commandlet 在 `PreInitPostStartupScreen` 中启动执行
- 执行完成后直接调用 `RequestEngineExit()`
- 不进入主循环

### 固定时间步长 (`-UseFixedTimeStep` / `-Deterministic`)

- `FApp::DeltaTime` 保持固定值
- `UpdateTimeAndHandleMaxTickRate` 会等待以维持固定帧率
