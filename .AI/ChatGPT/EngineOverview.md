总体流程
UE 5.5 的主流程由 Launch.cpp 中的全局对象 GEngineLoop 驱动：
GuardedMain
 ├─ PreMainInitDelegate
 ├─ EnginePreInit
 │   ├─ PreInitPreStartupScreen
 │   └─ PreInitPostStartupScreen
 ├─ EngineInit / EditorInit
 ├─ while (!IsEngineExitRequested())
 │    └─ EngineTick
 ├─ EditorExit（编辑器）
 └─ EngineLoopCleanupGuard → EngineExit
 1. 进程入口与异常保护
Launch.cpp:96 的 GuardedMain() 是受保护的主入口：
1.
设置当前线程为 GameThread。
2.
处理 -waitforattach、-WaitForDebugger。
3.
广播 FCoreDelegates::GetPreMainInitDelegate()。
4.
创建 EngineLoopCleanupGuard。
该 Guard 的析构函数会调用 EngineExit()，确保正常返回、初始化失败或异常退出时都尝试执行引擎清理。嵌入式运行模式 bIsEmbedded 下不自动退出，由外部宿主负责生命周期。
 2. PreInit：引擎早期初始化
入口是：
EnginePreInit(CmdLine)
    -> GEngineLoop.PreInit(CmdLine)
FEngineLoop::PreInit() 又拆成两个阶段：
PreInitPreStartupScreen()
PreInitPostStartupScreen()
 2.1 PreInitPreStartupScreen
主要完成“引擎还没有完整启动前”的基础设施准备：
•
设置日志主线程、信号处理和内存 TLS。
•
设置并解析命令行。
•
确定项目名称、项目文件和运行模式：
◦
游戏
◦
编辑器
◦
Dedicated Server
◦
Commandlet
•
初始化 Trace、LLM、内存跟踪。
•
创建控制台输出设备和日志 Backlog。
•
创建线程池、Stats 线程等基础线程设施。
•
加载 CoreUObject。
•
加载早期模块：
◦
Engine
◦
Renderer
◦
RenderCore
◦
RHICore
◦
SlateRHIRenderer
◦
其他平台和项目 PreInit 模块。
•
调用 FEngineLoop::AppInit()，完成：
◦
平台预初始化
◦
FPlatformApplicationMisc::PreInit()
◦
文件系统、配置系统、日志系统初始化
◦
工作目录设置
◦
国际化初始化
◦
平台相关应用初始化。
对应关键位置：
•
LaunchEngineLoop.cpp:1923
•
LaunchEngineLoop.cpp:2822
•
LaunchEngineLoop.cpp:2887
•
LaunchEngineLoop.cpp:2921
•
LaunchEngineLoop.cpp:6378
 2.2 PreInitPostStartupScreen
这个阶段开始建立可运行的渲染和 UI 环境：
•
恢复 PreInitContext。
•
初始化早期加载画面、Movie Player、PreLoadScreen。
•
初始化 Shader 参数和 Shader 类型注册。
•
初始化 HMD。
•
调用 RHIInit()。
•
初始化 Pipeline State Cache、Render Utils、Shader Code Library。
•
初始化 Derived Data Cache、Shader 编译管理器。
•
创建 Slate Renderer。
•
初始化字体、Slate 和渲染相关系统。
•
加载 Startup Core Modules：
◦
Core
◦
Networking
◦
Messaging
◦
Slate
◦
UnrealEd（编辑器）
◦
UMG
◦
其他运行时和开发模块。
•
调用 FPlatformApplicationMisc::PostInit()。
•
启动 Rendering Thread。
•
启动加载画面或启动电影。
•
加载项目和插件的模块阶段：
◦
PreLoadingScreen
◦
PreDefault
◦
Default
◦
PostDefault
•
运行自动化 Smoke Tests。
•
清理 PreInitContext。
这一阶段结束后，平台、文件、日志、配置、对象系统、RHI、渲染线程、Slate 和大部分模块已经准备完毕，但 GEngine 还没有完成正式初始化。
 3. Init：创建并初始化 UEngine
非编辑器流程：
EngineInit()
    -> GEngineLoop.Init()
编辑器流程则进入：
EditorInit(GEngineLoop)
编辑器有额外的 UnrealEd 初始化流程，不能简单等同于普通游戏的 FEngineLoop::Init()。
 FEngineLoop::Init() 主要步骤
1.
根据配置选择 Engine 类：
◦
游戏：UGameEngine
◦
编辑器：UUnrealEdEngine
2.
创建全局对象：
GEngine = NewObject<UEngine>(...)
编辑器还会设置：
GEditor = GUnrealEd = ...
3.
调用 GEngine->ParseCommandline()。
4.
初始化计时、帧率、Benchmark 和 -seconds 参数。
5.
调用：
GEngine->Init(this)
这里会建立 World、Viewport、游戏框架、网络、音频等高层 Engine 系统。
6.
广播：
FCoreDelegates::OnPostEngineInit
7.
启动 Session Service、Engine Service、Trace Service。
8.
加载项目和插件的 PostEngineInit 模块。
9.
调用：
GEngine->Start()
10.
等待引擎加载画面或 Movie Player 完成。
11.
设置：
GIsRunning = true;
12.
广播：
FCoreDelegates::OnFEngineLoopInitComplete
此时引擎进入可运行状态。
关键位置：LaunchEngineLoop.cpp:4768。
 4. 主循环
GuardedMain() 中的主循环是：
while (!IsEngineExitRequested())
{
    EngineTick();
}
EngineTick() 只是转发：
GEngineLoop.Tick();
因此，退出条件不在 Tick() 内部决定，而是由全局退出请求状态控制。退出请求可能来自：
•
RequestEngineExit()
•
FPlatformMisc::RequestExit()
•
窗口关闭
•
控制台命令 quit
•
Benchmark 达到帧数或时间限制
•
Fatal Error 或平台终止信号
•
测试退出条件
•
模块加载失败。
 5. 单帧 Tick 流程
FEngineLoop::Tick() 可以概括为：
BeginExitIfRequested
 ├─ 心跳、热更新、配置变化
 ├─ 开始帧追踪和统计
 ├─ 更新时间、DeltaTime、帧率限制
 ├─ PumpMessages
 ├─ 处理 Slate 输入
 ├─ GEngine->Tick
 ├─ Slate Tick
 ├─ 异步任务和网络相关任务
 ├─ Render/RHI Tick
 ├─ 同步 GameThread 与 RenderThread
 ├─ Deferred Commands、Ticker、线程管理器
 ├─ OnEndFrame
 └─ 帧计数递增
 5.1 平台消息处理
关键代码：
FPlatformApplicationMisc::PumpMessages(true);
位置：LaunchEngineLoop.cpp:5793-5801。
它负责从操作系统消息队列取出并分发平台消息，例如：
•
窗口创建、销毁、移动、缩放
•
键盘、鼠标、触摸输入
•
窗口激活和失去焦点
•
系统关闭请求
•
平台窗口事件。
嵌入式运行模式不调用该函数，因为消息由外部宿主应用接收后传入 UE。
 5.2 Slate 输入处理
平台消息 Pump 完成后，Slate 处理积累的输入：
SlateApp.PollGameDeviceState();
SlateApp.FinishedInputThisFrame();
随后在 GEngine->Tick() 后处理游戏世界产生的 Slate 操作，并执行：
FSlateApplication::Get().Tick(ESlateTickType::PlatformAndInput);
之后再执行完整的 Slate 时间、Widget 和绘制 Tick：
FSlateApplication::Get().Tick(
    bRenderingSuspended
        ? ESlateTickType::Time
        : ESlateTickType::TimeAndWidgets
);
因此，Slate 在一帧中分为两个主要阶段：
1.
平台和输入阶段。
2.
UI 时间、Widget 更新和绘制阶段。
 5.3 游戏和网络消息处理
核心游戏逻辑入口：
GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);
位置：LaunchEngineLoop.cpp:5870-5872。
网络消息通常不是由 Launch 层直接解析，而是由 GEngine->Tick() 进一步驱动 World、NetDriver、PlayerController、Actor、组件和游戏逻辑。也就是说：
OS 消息
  -> PlatformApplicationMisc::PumpMessages
  -> Slate 输入
  -> GEngine->Tick
      -> World Tick
      -> NetDriver Tick
      -> Actor / Component Tick
      -> 游戏代码
 5.4 渲染线程和帧同步
每帧会向 Render Thread 投递：
•
BeginFrame
•
Scene StartFrame
•
Scene EndFrame
•
RHI Tick
•
EndFrame
最后通过：
FFrameEndSync::Sync();
同步 Game Thread、Render Thread 和 RHI Thread，保证本帧资源和渲染命令处于一致状态。
 6. 退出流程
 6.1 发起退出
EngineExit() 首先设置退出请求：
RequestEngineExit(TEXT("EngineExit() was called"));
然后调用：
GEngineLoop.Exit();
退出请求使主循环条件失败：
IsEngineExitRequested() == true
如果退出发生在 Tick 期间，Tick() 开头的：
BeginExitIfRequested();
也会处理平台级退出状态。
 6.2 FEngineLoop::Exit()
退出顺序大致是：
1.
清理上一帧待删除对象。
2.
设置：
GIsRunning = 0;
3.
停止 Engine Service、Trace Service、Session Service。
4.
调用：
GEngine->PreExit();
5.
Flush 或取消异步加载。
6.
禁止新的异步加载请求。
7.
等待纹理 Streaming 和资源请求完成。
8.
关闭音频设备。
9.
关闭 Slate 和窗口。
10.
清理字体、Asset Registry、Virtualization 等模块。
11.
等待 Pipeline State Cache 任务完成。
12.
调用：
AppPreExit();
13.
销毁物理系统。
14.
停止 Rendering Thread。
15.
关闭 Shader、RHI、IoDispatcher。
16.
按逆序卸载模块。
17.
停止 Stats、线程池和 TaskGraph。
18.
调用 RHIExit()。
关键位置：LaunchEngineLoop.cpp:5003-5237。
 6.3 AppPreExit() 与 AppExit()
AppPreExit() 属于受保护的正常退出阶段：
FCoreDelegates::OnPreExit.Broadcast();
FCoreDelegates::OnExit.Broadcast();
并清理 DDC、Shader Compiler、IO Thread Pool 等仍依赖引擎系统的对象。
AppExit() 是更底层的平台退出阶段，负责：
•
FPlatformApplicationMisc::TearDown()
•
FPlatformMisc::PlatformTearDown()
•
销毁配置系统
•
关闭日志
•
关闭本地化和国际化
•
关闭 Trace。
AppExit() 使用静态布尔值保证只执行一次。
 7. 正常退出、初始化失败和嵌入式模式的差异
 初始化失败
GuardedMain() 在 PreInit 或 Init 返回非零错误码时直接返回，但局部 Guard 仍会触发 EngineExit()，执行可用的清理流程。
 编辑器
编辑器初始化和退出由：
EditorInit(GEngineLoop)
EditorExit()
参与处理，普通游戏的 GEngineLoop.Init() 不是编辑器全部初始化逻辑。
 嵌入式引擎
嵌入式模式下：
•
不进入 GuardedMain() 自己的 Tick 循环。
•
不主动 Pump 平台消息。
•
外部应用负责消息循环和帧调用。
•
外部应用负责最终生命周期，Cleanup Guard 不自动调用 EngineExit()。
 核心结论
UE 的启动逻辑不是一个简单的 Init -> Tick -> Exit，而是分层完成：
平台和命令行
 -> CoreUObject / Module
 -> 文件、配置、日志、应用层
 -> RHI / Shader / Slate / Rendering Thread
 -> UEngine / World / Game Framework
 -> 每帧消息、输入、游戏、网络、渲染
 -> 资源同步和模块逆序关闭
其中：
•
操作系统窗口消息主要在 FPlatformApplicationMisc::PumpMessages() 处理。
•
UI 输入和窗口逻辑由 Slate 的多个 Tick 阶段处理。
•
游戏、World、网络和 Actor 消息由 GEngine->Tick() 向下分发。
•
退出通过全局退出请求打断主循环，再由 FEngineLoop::Exit() 按依赖关系逆序清理。