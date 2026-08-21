# 一、总览：从 OS 输入到 Widget Event 的完整链路

可以先记住这张总图：

```text
OS 消息队列 / 原生窗口系统
    ↓
PlatformApplication / PlatformApplicationMisc
    ↓
平台实现（Windows: FWindowsApplication）
    ↓
MessageHandler（通常是 FSlateApplication）
    ↓
FSlateApplication::OnXXX / ProcessXXX
    ↓
构造 Slate 输入事件（FPointerEvent / FKeyEvent / FCharacterEvent）
    ↓
确定目标路径（FWidgetPath）
    ↓
FEventRouter 按策略路由（Tunnel / Bubble / FocusPath / Leafmost）
    ↓
SWidget::OnPreviewXXX / OnXXX
    ↓
返回 FReply
    ↓
FSlateApplication::ProcessReply
    ↓
更新 Capture / Focus / DragDrop / 光标 / 高精度鼠标 等状态
```

也就是说，Slate 的输入系统本质上分三层：

1. **平台层收原生事件**
2. **Slate 应用层做输入归一化和路由**
3. **Widget 层消费事件并返回 `FReply`**

---

# 二、第一层：OS 如何把消息送到 Unreal

以你当前环境最 relevant 的 Windows 路径为例。

---

## 1) 主循环先 pump OS 消息

正常帧里，`FEngineLoop` 会先 pump 系统消息。  
前面我们已经看到：

文件：`Engine/Source/Runtime/Launch/Private/LaunchEngineLoop.cpp:5548`

```cpp
FPlatformApplicationMisc::PumpMessages(true);
```

这一步的作用是：
- 从 Windows 消息队列取出消息
- 分发给对应窗口过程
- 进入 `ApplicationCore` 平台层

---

## 2) Windows 窗口过程进入 `FWindowsApplication::ProcessMessage`

文件：`Engine/Source/Runtime/ApplicationCore/Private/Windows/WindowsApplication.cpp:1719-1724`

```cpp
LRESULT CALLBACK FWindowsApplication::AppWndProc(HWND hwnd, uint32 msg, WPARAM wParam, LPARAM lParam)
{
    return WindowsApplication_WndProc( hwnd, msg, wParam, lParam );
}

int32 FWindowsApplication::ProcessMessage( HWND hwnd, uint32 msg, WPARAM wParam, LPARAM lParam )
```

也就是说：

- Win32 把消息打到窗口过程
- UE 的平台层入口是 `FWindowsApplication::ProcessMessage(...)`

---

# 三、第二层：平台层如何转发给 Slate

关键点在于：  
`FWindowsApplication` 自己不直接决定 Widget 收到什么，它会把标准化后的输入发给 `MessageHandler`。

这个 `MessageHandler` 通常就是 `FSlateApplication`。

---

## 1) 平台层持有一个通用消息处理器

文件：`WindowsApplication.cpp:1167-1174`

```cpp
void FWindowsApplication::SetMessageHandler( const TSharedRef< FGenericApplicationMessageHandler >& InMessageHandler )
{
    GenericApplication::SetMessageHandler(InMessageHandler);

    for (...)
    {
        (*DeviceIt)->SetMessageHandler(InMessageHandler);
    }
}
```

这说明平台层只依赖 `FGenericApplicationMessageHandler` 接口。  
而 `FSlateApplication` 正是该接口的核心实现之一。

所以链路是：

```text
Windows 消息 -> FWindowsApplication -> MessageHandler -> FSlateApplication
```

---

## 2) 键盘消息转发给 `MessageHandler->OnKeyDown/Up`

文件：`WindowsApplication.cpp:2880-2884`

```cpp
uint32 CharCode = ::MapVirtualKey( Win32Key, MAPVK_VK_TO_CHAR );
const bool Result = MessageHandler->OnKeyDown( ActualKey, CharCode, bIsRepeat );
```

键盘抬起：

文件：`WindowsApplication.cpp:2961-2962`

```cpp
const bool Result = MessageHandler->OnKeyUp( ActualKey, CharCode, bIsRepeat );
```

---

## 3) 鼠标按键消息转发给 `MessageHandler->OnMouseDown/Up`

文件：`WindowsApplication.cpp:3049-3060`

```cpp
if (bMouseUp)
{
    return MessageHandler->OnMouseUp( MouseButton, CursorPos ) ? 0 : 1;
}
else if (bDoubleClick)
{
    MessageHandler->OnMouseDoubleClick( CurrentNativeEventWindowPtr, MouseButton, CursorPos );
}
else
{
    MessageHandler->OnMouseDown( CurrentNativeEventWindowPtr, MouseButton, CursorPos );
}
```

---

## 4) 鼠标移动转发给 `OnMouseMove` / `OnRawMouseMove`

文件：`WindowsApplication.cpp:3066-3078`

```cpp
case WM_INPUT:
{
    if( DeferredMessage.RawInputFlags == MOUSE_MOVE_RELATIVE )
    {
        MessageHandler->OnRawMouseMove(DeferredMessage.X, DeferredMessage.Y);
    }
    else
    {
        MessageHandler->OnMouseMove();
    }
}
```

这里区分了：

- `OnRawMouseMove`：原始相对移动，常用于高精度鼠标 / raw input
- `OnMouseMove`：普通光标移动

---

# 四、第三层：`FSlateApplication` 把平台输入变成 Slate 事件

到了这一步，平台原生消息已经进入 Slate 了。

---

# 五、鼠标按下路径：从 OS 到 `SWidget::OnMouseButtonDown`

这是最典型、也最能说明 Slate 路由机制的一条链。

---

## 1) 平台消息进入 `FSlateApplication::OnMouseDown`

文件：`Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:5056-5078`

```cpp
bool FSlateApplication::OnMouseDown(
    const TSharedPtr<FGenericWindow>& PlatformWindow,
    const EMouseButtons::Type Button,
    const FVector2D CursorPos )
{
    FKey Key = TranslateMouseButtonToKey(Button);

    FPointerEvent MouseEvent(
        GetUserIndexForMouse(),
        CursorPointerIndex,
        CursorPos,
        GetLastCursorPos(),
        PressedMouseButtons,
        Key,
        0,
        PlatformApplication->GetModifierKeys()
    );

    return ProcessMouseButtonDownEvent(PlatformWindow, MouseEvent);
}
```

### 这一步做了什么？
把平台层数据标准化为 Slate 自己的 `FPointerEvent`：

- 用户索引
- pointer index
- 当前屏幕坐标
- 上一次坐标
- 当前已按下按键集合
- 本次 effecting button
- modifier keys

也就是说，从这一刻开始，后面就基本不关心 Win32 细节了。

---

## 2) `ProcessMouseButtonDownEvent`：输入预处理、捕获、命中测试

文件：`SlateApplication.cpp:5081-5180`

关键步骤如下。

### a) 更新时间与 capture

```cpp
SetLastUserInteractionTime(this->GetCurrentTime());

if (PlatformWindow.IsValid())
{
    PlatformApplication->SetCapture(PlatformWindow);
}
```

说明：
- 记录用户交互时间
- 平台窗口层先建立鼠标捕获（至少从平台角度保持后续输入连续）

---

### b) 更新按下按钮集合

```cpp
PressedMouseButtons.Add(MouseEvent.GetEffectingButton());
```

---

### c) 先给 Input PreProcessors 机会

文件：`SlateApplication.cpp:5112-5115`

```cpp
if (InputPreProcessors.HandleMouseButtonDownEvent(*this, MouseEvent))
{
    return true;
}
```

这意味着：
- Slate Widget 不是第一手消费者
- 前置输入处理器可以截获事件

常见例子：
- Analog Cursor
- 特殊输入系统
- 某些编辑器级输入拦截器

如果 preprocessor 返回已处理，后面 Widget 就收不到了。

---

### d) 如果当前有 mouse capture，优先发给 captor

文件：`SlateApplication.cpp:5125-5167`

```cpp
if (SlateUser->HasCapture(MouseEvent.GetPointerIndex()))
{
    FWidgetPath MouseCaptorPath = SlateUser->GetCaptorPath(...);

    Reply = FEventRouter::Route<FReply>(..., FEventRouter::FToLeafmostPolicy(MouseCaptorPath), ... OnPreviewMouseButtonDown ...);

    if (!Reply.IsEventHandled())
    {
        Reply = FEventRouter::Route<FReply>(..., FEventRouter::FToLeafmostPolicy(MouseCaptorPath), ... OnMouseButtonDown ...);
    }
}
```

### 含义
如果某个 Widget 已经 capture 了这个 pointer：

- 不再做“鼠标下方命中测试”
- 直接把事件交给 capture 目标路径
- 路由策略是 `FToLeafmostPolicy`，也就是只往 capture 路径最深的叶子控件发

这很好理解：  
拖拽、slider 拖动、分割条拖动时，鼠标可能已经离开原控件，但控件仍应继续收到事件。

---

### e) 没有 capture 时，做命中测试

文件：`SlateApplication.cpp:5171-5180`

```cpp
FWidgetPath WidgetsUnderCursor = LocateWindowUnderMouse(
    MouseEvent.GetScreenSpacePosition(),
    GetInteractiveTopLevelWindows(),
    false,
    SlateUser->GetUserIndex());

Reply = RoutePointerDownEvent(WidgetsUnderCursor, MouseEvent);
```

### 这里是整个输入路由最关键的一步之一
`LocateWindowUnderMouse(...)` 会：

- 从当前屏幕坐标出发
- 在交互窗口集合里找命中的顶层窗口
- 再通过 hittest / arranged widget path 构造一条 `FWidgetPath`

`FWidgetPath` 可以理解成：

> 从顶层窗口一路到“鼠标命中的最深叶子 Widget”的有序路径。

例如：

```text
SWindow
  -> SOverlay
    -> SBorder
      -> SButton
        -> STextBlock
```

这个路径就是后续路由的依据。

---

## 3) `RoutePointerDownEvent`：先 Tunnel，再 Bubble

文件：`SlateApplication.cpp:5203-5253`

---

### a) 更新用户 pointer 位置

```cpp
SlateUser->UpdatePointerPosition(PointerEvent);
```

---

### b) 先走 Preview/Tunnel

文件：`SlateApplication.cpp:5221-5228`

```cpp
Reply = FEventRouter::Route<FReply>(
    this,
    FEventRouter::FTunnelPolicy(WidgetsUnderPointer),
    TransformedPointerEvent,
    [](const FArrangedWidget TargetWidget, const FPointerEvent& Event)
    {
        return TargetWidget.Widget->OnPreviewMouseButtonDown(TargetWidget.Geometry, Event);
    });
```

### `FTunnelPolicy` 是什么？
文件：`SlateApplication.cpp:338-377`

```cpp
class FTunnelPolicy
{
    ...
    bool ShouldKeepGoing() const { return WidgetIndex < RoutingPath.Widgets.Num(); }
    void Next() { ++WidgetIndex; }
}
```

它是 **从根到叶** 的方向。

也就是：

```text
SWindow -> Parent -> Child -> Leaf
```

对应事件：
- `OnPreviewMouseButtonDown`

这类似“预处理/拦截阶段”。

---

### c) Preview 没处理，再走正式 Bubble

文件：`SlateApplication.cpp:5230-5249`

```cpp
Reply = FEventRouter::Route<FReply>(
    this,
    FEventRouter::FBubblePolicy(WidgetsUnderPointer),
    TransformedPointerEvent,
    [this](const FArrangedWidget TargetWidget, const FPointerEvent& Event)
    {
        if (Event.IsTouchEvent())
        {
            TempReply = TargetWidget.Widget->OnTouchStarted(...);
        }

        if (!Event.IsTouchEvent() || (!TempReply.IsEventHandled() && this->bTouchFallbackToMouse))
        {
            TempReply = TargetWidget.Widget->OnMouseButtonDown(...);
        }
        return TempReply;
    });
```

### `FBubblePolicy` 是什么？
文件：`SlateApplication.cpp:379-418`

```cpp
class FBubblePolicy
{
    ...
    bool ShouldKeepGoing() const { return WidgetIndex >= 0; }
    void Next() { --WidgetIndex; }
}
```

它是 **从叶到根** 的方向。

也就是：

```text
Leaf -> Parent -> ... -> SWindow
```

正式点击事件 `OnMouseButtonDown` 就是按这个顺序冒泡。

---

## 4) 焦点可能在鼠标按下后改变

文件：`SlateApplication.cpp:5274-5292`

```cpp
if ((!bFocusChangedByEventHandler || bNeedToActivateWindow) && !Reply.GetUserFocusRecepient().IsValid())
{
    for (int32 WidgetIndex = WidgetsUnderPointer.Widgets.Num() - 1; WidgetIndex >= 0; --WidgetIndex)
    {
        const FArrangedWidget& CurWidget = WidgetsUnderPointer.Widgets[WidgetIndex];
        if (CurWidget.Widget->SupportsKeyboardFocus())
        {
            FWidgetPath NewFocusedWidgetPath = WidgetsUnderPointer.GetPathDownTo(CurWidget.Widget);
            SetUserFocus(PointerEvent.GetUserIndex(), NewFocusedWidgetPath, EFocusCause::Mouse);
            break;
        }
    }
}
```

这说明鼠标点击不只是触发点击事件，还可能：
- 自动把 keyboard focus 交给可聚焦控件

比如点到一个 `SEditableText` 后，后续键盘输入就沿它的 focus path 走。

---

# 六、鼠标移动路径：从 OS 到 `OnMouseMove`

---

## 1) 入口 `FSlateApplication::OnMouseMove`

文件：`SlateApplication.cpp:6060-6118`

```cpp
bool FSlateApplication::OnMouseMove()
{
    const FVector2f CurrentCursorPosition = ...;
    if (LastPlatformCursorPosition != CurrentCursorPosition)
    {
        FPointerEvent MouseEvent(... CurrentCursorPosition, LastPlatformCursorPosition, ...);
        Result = ProcessMouseMoveEvent(MouseEvent);
        LastPlatformCursorPosition = CurrentCursorPosition;
    }
}
```

---

## 2) `ProcessMouseMoveEvent`：先预处理，再命中测试，再路由

文件：`SlateApplication.cpp:6157-6197`

```cpp
if (InputPreProcessors.HandleMouseMoveEvent(*this, MouseEvent))
{
    return true;
}

FWidgetPath WidgetsUnderCursor =
    LocateWindowUnderMouse(MouseEvent.GetScreenSpacePosition(), GetInteractiveTopLevelWindows(), false, MouseEvent.GetUserIndex());

bResult = RoutePointerMoveEvent(WidgetsUnderCursor, MouseEvent, bIsSynthetic);
```

### 核心步骤
- preprocessors 抢先处理
- 命中测试，得到 `WidgetsUnderCursor`
- 路由移动事件

虽然你这次没让我继续往 `RoutePointerMoveEvent` 深挖源码，但从调用点和上面 down/up 的模式可以肯定它会处理：

- `OnMouseMove`
- `OnMouseEnter`
- `OnMouseLeave`
- capture 情况下的 move
- drag-detect / drag-over / drag-leave / drag-enter 之类逻辑

你前面 grep 到的调用点也证实了这一点：

- `OnMouseMove` 在 `5666`、`5758`
- `ProcessMouseMoveEvent` 在 `6157`

---

# 七、键盘路径：从 OS 到 `OnKeyDown`

键盘跟鼠标不同的关键点是：

> **键盘事件不是根据鼠标命中路径走，而是根据 Focus Path 走。**

---

## 1) 平台层转发到 `FSlateApplication::OnKeyDown`

文件：`SlateApplication.cpp:4751-4756`

```cpp
bool FSlateApplication::OnKeyDown(const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat)
{
    FKey const Key = FInputKeyManager::Get().GetKeyFromCodes(KeyCode, CharacterCode);
    FKeyEvent KeyEvent(Key, PlatformApplication->GetModifierKeys(), GetUserIndexForKeyboard(), IsRepeat, CharacterCode, KeyCode);

    return ProcessKeyDownEvent(KeyEvent);
}
```

这一步做的是：
- 平台键码 -> Unreal `FKey`
- 组装 `FKeyEvent`

---

## 2) `ProcessKeyDownEvent`：先 preprocessor，再沿 FocusPath 路由

文件：`SlateApplication.cpp:4779-4783`

```cpp
if (InputPreProcessors.HandleKeyDownEvent(*this, InKeyEvent))
{
    return true;
}
```

然后：

文件：`SlateApplication.cpp:4816-4849`

```cpp
TSharedRef<FWidgetPath> EventPathRef = SlateUser->GetFocusPath();
const FWidgetPath& EventPath = EventPathRef.Get();

Reply = FEventRouter::RouteAlongFocusPath(
    this,
    FEventRouter::FTunnelPolicy(EventPath),
    InKeyEvent,
    [](const FArrangedWidget& CurrentWidget, const FKeyEvent& Event)
    {
        return CurrentWidget.Widget->OnPreviewKeyDown(CurrentWidget.Geometry, Event);
    });

if (!Reply.IsEventHandled())
{
    Reply = FEventRouter::RouteAlongFocusPath(
        this,
        FEventRouter::FBubblePolicy(EventPath),
        InKeyEvent,
        [](const FArrangedWidget& SomeWidgetGettingEvent, const FKeyEvent& Event)
        {
            return SomeWidgetGettingEvent.Widget->OnKeyDown(SomeWidgetGettingEvent.Geometry, Event);
        });
}
```

### 这里和鼠标的区别
鼠标：
- 走 **WidgetsUnderCursor**
- 由 hit test 决定

键盘：
- 走 **FocusPath**
- 由当前 keyboard focus 决定

### 顺序仍然是
1. `OnPreviewKeyDown`：Tunnel（根 -> 叶）
2. `OnKeyDown`：Bubble（叶 -> 根）

---

## 3) 如果没人处理，走未处理回调

文件：`SlateApplication.cpp:4866-4869`

```cpp
if (!Reply.IsEventHandled() && UnhandledKeyDownEventHandler.IsBound())
{
    Reply = UnhandledKeyDownEventHandler.Execute(InKeyEvent);
}
```

这一步用于全局兜底逻辑。

---

# 八、字符输入路径：`OnKeyChar`

字符输入和 KeyDown 不完全一样。

- `KeyDown`：更偏按键语义，例如 `Enter`, `Escape`, `Left`, `Ctrl+C`
- `KeyChar`：更偏文本输入语义，例如输入字符 `'a'`, `'中'`, `'@'`

---

## 1) 入口 `OnKeyChar`

文件：`SlateApplication.cpp:4701-4704`

```cpp
bool FSlateApplication::OnKeyChar(const TCHAR Character, const bool IsRepeat)
{
    FCharacterEvent CharacterEvent(Character, PlatformApplication->GetModifierKeys(), 0, IsRepeat);
    return ProcessKeyCharEvent(CharacterEvent);
}
```

---

## 2) `ProcessKeyCharEvent` 直接沿 FocusPath Bubble

文件：`SlateApplication.cpp:4721-4745`

```cpp
TSharedRef<FWidgetPath> EventPathRef = User->GetFocusPath();
const FWidgetPath& EventPath = EventPathRef.Get();

Reply = FEventRouter::RouteAlongFocusPath(
    this,
    FEventRouter::FBubblePolicy(EventPath),
    InCharacterEvent,
    [](const FArrangedWidget& SomeWidgetGettingEvent, const FCharacterEvent& Event)
    {
        return SomeWidgetGettingEvent.Widget->OnKeyChar(SomeWidgetGettingEvent.Geometry, Event);
    });
```

### 注意点
字符输入这里没有 `OnPreviewKeyChar` 这套 tunnel 预览，而是直接 bubble `OnKeyChar`。

这也符合文本输入的实际用法：
- 焦点控件（比如文本框）是主消费者

---

# 九、`FEventRouter`：Slate 事件路由的核心机制

文件：`Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:235-444`

这个类定义了多种路由策略。

---

## 1) `FTunnelPolicy`
从根到叶：
```text
Window -> Parent -> Child -> Leaf
```

用于：
- `OnPreviewMouseButtonDown`
- `OnPreviewKeyDown`

适合父控件先拦截。

---

## 2) `FBubblePolicy`
从叶到根：
```text
Leaf -> Parent -> ... -> Window
```

用于：
- `OnMouseButtonDown`
- `OnKeyDown`
- `OnKeyChar`

适合最具体的目标先处理，父级兜底。

---

## 3) `FToLeafmostPolicy`
只发给最深叶子节点。

用于：
- capture 场景下直接发给当前捕获目标
- 避免重新 hit test 全树冒泡

---

## 4) `RouteAlongFocusPath`
这是 `Route(...)` 的一个语义包装：
- 路由依据不是鼠标命中路径
- 而是当前 `FocusPath`

---

# 十、`FWidgetPath`：为什么它是中枢

不管是鼠标还是键盘，最终都要落在一条路径上。

---

## 鼠标事件的路径来源
```cpp
LocateWindowUnderMouse(...)
```

得到的是：
- 鼠标当前命中的窗口
- 从根到叶的 arranged widget path

---

## 键盘事件的路径来源
```cpp
SlateUser->GetFocusPath()
```

得到的是：
- 当前用户的 focus widget path

---

所以可以把 Slate 输入总结成：

- **指针输入**：靠 hit test 找路
- **非指针输入**：靠 focus 找路

---

# 十一、Widget 收到事件后，`FReply` 如何反向影响系统

这部分非常关键，很多人只看到 `OnMouseButtonDown`，却没意识到真正的系统状态变更发生在 `ProcessReply()`。

文件：`Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:3264`

---

## 1) `ProcessReply()` 是事件分发后的“状态提交阶段”

```cpp
void FSlateApplication::ProcessReply(...)
```

Widget 在 `OnMouseButtonDown` / `OnKeyDown` 里返回 `FReply`，但：

- capture
- focus
- drag-drop
- mouse lock
- 高精度鼠标

这些都不是 Widget 自己直接改，而是通过 `FReply` 描述“请求”，再由 `FSlateApplication` 统一执行。

---

## 2) 释放 capture / 焦点 / 结束 drag-drop

文件：`SlateApplication.cpp:3274-3296`

```cpp
if (SlateUser->HasCapture(PointerIndex) && (TheReply.ShouldReleaseMouse() || bStartingDragDrop))
{
    SlateUser->ReleaseCapture(PointerIndex);
}

if (TheReply.ShouldReleaseUserFocus())
{
    ...
}

if (TheReply.ShouldEndDragDrop())
{
    SlateUser->CancelDragDrop();
}
```

---

## 3) 启动拖拽

文件：`SlateApplication.cpp:3298-3359`

```cpp
if (bStartingDragDrop)
{
    SlateUser->SetDragDropContent(...);

    // 先给旧 hover 路径发 OnMouseLeave
    // 再给当前路径发 OnDragEnter
}
```

这说明：
- 拖拽不是简单布尔值
- 启动拖拽时 Slate 会补发一串过渡事件

---

## 4) 设置 pointer capture

文件：`SlateApplication.cpp:3373-3380`

```cpp
TSharedPtr<SWidget> RequestedMouseCaptor = TheReply.GetMouseCaptor();

if (RequestedMouseCaptor.IsValid() && !bStartingDragDrop)
{
    if (SlateUser->SetPointerCaptor(PointerIndex, RequestedMouseCaptor.ToSharedRef(), CurrentEventPath))
    {
        ...
    }
}
```

这就是为什么控件只要返回：

```cpp
return FReply::Handled().CaptureMouse(AsShared());
```

后续 move/up 会继续发给它。

---

## 5) 高精度鼠标模式

文件：`SlateApplication.cpp:3467-3473`

```cpp
if (TheReply.ShouldUseHighPrecisionMouse())
{
    PlatformApplication->SetCapture(Window->GetNativeWindow());
    PlatformApplication->SetHighPrecisionMouseMode(true, Window->GetNativeWindow());
}
```

这通常用于：
- FPS camera 控制
- viewport look-around
- editor 视口拖动

---

# 十二、一个完整鼠标点击示例

假设你点中了一个 `SButton`，大致过程是：

1. Windows 产生 `WM_LBUTTONDOWN`
2. `FWindowsApplication::ProcessMessage(...)`
3. 调 `MessageHandler->OnMouseDown(...)`
4. 实际进入 `FSlateApplication::OnMouseDown(...)`
5. 构造 `FPointerEvent`
6. `ProcessMouseButtonDownEvent(...)`
7. preprocessors 先尝试处理
8. 若无 capture，则 `LocateWindowUnderMouse(...)` 命中到 `SButton`
9. 生成 `FWidgetPath`
10. `RoutePointerDownEvent(...)`
11. 先 Tunnel：
    - `SWindow::OnPreviewMouseButtonDown`
    - 父容器 Preview
    - ...
12. 若没人处理，再 Bubble：
    - `SButton::OnMouseButtonDown`
    - 若它处理了，停止冒泡
13. `SButton` 返回 `FReply::Handled().CaptureMouse(...).SetUserFocus(...)` 之类
14. `FSlateApplication::ProcessReply(...)`
15. Slate 更新 capture / focus / 可能进入拖拽检测

---

# 十三、一个完整键盘输入示例

假设某个 `SEditableText` 已有 focus，用户按下 `A`：

1. Windows 产生 `WM_KEYDOWN`
2. `FWindowsApplication::ProcessMessage(...)`
3. 调 `MessageHandler->OnKeyDown(...)`
4. 进入 `FSlateApplication::OnKeyDown(...)`
5. 构造 `FKeyEvent`
6. `ProcessKeyDownEvent(...)`
7. preprocessors 先处理
8. 取 `SlateUser->GetFocusPath()`
9. 先 Tunnel：`OnPreviewKeyDown`
10. 再 Bubble：`OnKeyDown`
11. 如果系统随后产生 `WM_CHAR`
12. 再进入 `OnKeyChar(...)`
13. 沿 FocusPath Bubble 到 `SEditableText::OnKeyChar(...)`
14. 文本控件真正插入字符

所以文本输入一般是：
- `OnKeyDown` 负责按键语义
- `OnKeyChar` 负责字符落字

---

# 十四、你可以这样抓住 Slate 输入系统的核心模型

---

## 模型 1：平台层只负责“把输入送进来”
- `PumpMessages`
- `FWindowsApplication::ProcessMessage`
- `MessageHandler->OnXXX`

---

## 模型 2：SlateApplication 负责“把输入变成可路由事件”
- 构造 `FPointerEvent` / `FKeyEvent`
- 命中测试 / 获取 FocusPath
- `FEventRouter` 路由
- 管理 `FSlateUser`

---

## 模型 3：Widget 只负责“处理事件并声明结果”
- `OnPreviewMouseButtonDown`
- `OnMouseButtonDown`
- `OnPreviewKeyDown`
- `OnKeyDown`
- `OnKeyChar`

Widget 不直接操纵全局输入状态，而是通过 `FReply` 声明：
- 我处理了
- 我要 capture
- 我要 focus
- 我要开始拖拽
- 我要高精度鼠标

---

# 十五、最后给一个精炼总结

从 OS 接收输入到 Widget Event 触发，完整过程是：

1. **OS 产生原生输入消息**
2. **平台层**（如 `FWindowsApplication`）在 `PumpMessages` 后收到并解析消息
3. 平台层通过 `MessageHandler->OnXXX` 转交给 **`FSlateApplication`**
4. `FSlateApplication` 把原始输入包装成 Slate 事件对象：
   - `FPointerEvent`
   - `FKeyEvent`
   - `FCharacterEvent`
5. 根据输入类型确定目标路径：
   - 鼠标/触摸：`LocateWindowUnderMouse(...)` → `FWidgetPath`
   - 键盘/字符：`GetFocusPath()`
6. 通过 `FEventRouter` 按策略路由：
   - Preview 用 Tunnel
   - 正式事件用 Bubble
   - capture 时可直达 leaf/captor
7. 对应 `SWidget::OnXXX` 被调用
8. Widget 返回 `FReply`
9. `FSlateApplication::ProcessReply(...)` 根据 `FReply` 统一更新输入系统状态

---

如果你愿意，我下一步可以继续做两件事中的任意一个：

1. **继续深挖鼠标移动链**：`RoutePointerMoveEvent()`、`OnMouseEnter/Leave`、Hover、DragDetect、DragOver  
2. **继续深挖 UMG 层**：Slate `SWidget` 事件是怎么再传到 `UUserWidget` / `SObjectWidget` / Blueprint 事件里的