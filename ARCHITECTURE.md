# 架构说明（Phase 04 / 04D 实际实现）

> 对应任务书：`00_项目总纲.md` + `04_任务书_MOSS_TTS与音频播放.md` + `04D_任务书_Qwen3-TTS_GGML_Provider正式接入.md`
> （Phase 01 的骨架与事件协议、Phase 02 的桌面交互层、Phase 03 的 llama.cpp Provider 保留）。
> 本文描述**当前代码真实长什么样**，不是未来蓝图。后续阶段改动公共协议前，请先更新本文。
> 最后核对：2026-11-18（commit 见 `git log -1`）。
>
> **编号说明**：`00_项目总纲.md` §12 用 Phase 01–10 描述阶段，而本任务文件包按 `00`–`14`
> 编号，两者从 07 起不再一一对应（例如任务书 09 = DSH/Codex，总纲 Phase 08 = DSH/Codex）。
> 本文与实现状态文档一律使用**任务书文件编号**，引用跨阶段内容时会写出任务名称以免歧义。

## 1. 分层与依赖方向

```text
        ┌────────────────────────────────────────────────────────────┐
        │ ui/  (PySide6, 只跑在 UI 线程)                              │
        │  app_shell.py      窗口编排/托盘/热键/状态持久化             │
        │  ui_state.py       面板与桌宠共享的唯一状态源                │
        │  main_window.py    交互面板     chat_view.py  气泡视图       │
        │  pet_window.py     桌宠浮窗     pet_state.py  状态映射       │
        │  tray.py / settings_dialog.py / theme.py                    │
        │  hotkey.py         Win32 RegisterHotKey + Qt 原生过滤        │
        │  window_state.py   窗口状态 JSON + 多屏位置回收              │
        │  qt_event_bridge.py  ←─────── 唯一的 Qt ↔ asyncio 边界       │
        └──────────────┬─────────────────────────────────────────────┘
                       │ 提交动作 / 消费事件（Qt Signal）
        ┌──────────────▼─────────────────────────────┐
        │ app/  编排与生命周期                        │
        │  bootstrap.py (AsyncAppRuntime)             │
        │  orchestrator.py                            │
        └──────┬───────────────────────┬──────────────┘
               │                       │
   ┌───────────▼──────┐      ┌────────▼─────────────────┐
   │ core/            │      │ providers/               │
   │ config/events/   │      │ base.py (LLM Protocol)   │
   │ event_bus/models │      │ mock_llm.py              │
   └──────────┬───────┘      │ llama_cpp.py  → httpx    │
              │              │ tts_base.py (TTS Protocol)│
              │              │ qwen3_tts.py → httpx       │
              │              │ moss_tts.py / mock_tts.py  │
              │              └──────────────────────────┘
              │
   ┌──────────▼───────────────────────────────┐
   │ speech/（Phase 04，只被 app 调用）        │
   │  segmenter.py   文本切分                  │
   │  player.py      sounddevice 回调播放      │
   │  coordinator.py 段队列/pump/speech.* 事件 │
   └───────────────────────────────────────────┘
   ┌───────────────────────────────────────────┐
   │ infrastructure/  paths.py, logging.py     │
   │ （叶子模块：不 import 上层业务）           │
   └───────────────────────────────────────────┘
```

依赖规则（当前代码遵守，并由 `tests/test_architecture.py` 静态断言）：

- `core/*` 不 import `app`、`ui`、`providers`、`speech`；只允许依赖 `infrastructure.paths`（无状态平台路径工具）。
- `providers/*` 只依赖 `core`（以及 HTTP 客户端 httpx / TTS 的 numpy）；不依赖 UI、Orchestrator、speech。
- `speech/*` 只依赖 `core` 与 `providers`（`tts_base`）；不 import `app`/`ui`，也不碰 Qt。
- `app/*` 依赖 `core`、`providers`、`speech`；不 import PySide6（用 AST 测试验证）。
- `ui/*` 依赖 `core` 与 `infrastructure`；不 import Provider / Orchestrator / Tool Broker / TTS/STT/语音层；
  只通过 `QtEventBridge` 提交动作、只通过 `UiState` 消费事件。
- `infrastructure.logging` 只在 `TYPE_CHECKING` 下引用 `core.config.LoggingConfig`，运行期无反向依赖。
- **只有 `main.py` 与 `ui/` 允许创建 QApplication**（测试断言）。

## 2. 模块职责

| 文件 | 职责 | 关键约束 |
| --- | --- | --- |
| `main.py` | CLI 解析、配置加载、日志初始化、`--check`、GUI 启动与退出清理 | `--check` 不导入 Qt；GUI 关闭后统一清理 |
| `app/bootstrap.py` | `AppCore` 装配；`AsyncAppRuntime` 托管 asyncio 线程；关闭顺序与残留任务检查 | 跨线程只经 `submit()`/`call()` |
| `app/orchestrator.py` | 单轮对话生命周期、状态机、取消、超时/失败翻译 | 不碰 Qt；不依赖具体 Provider |
| `core/config.py` | Pydantic 配置模型、分层加载、来源/行号诊断、脱敏 | `extra="forbid"`；Phase 02 新增 `ui.pet/panel/tray/hotkey` |
| `core/events.py` | `AppEvent` 信封、`EventType`、payload 构造器 | 04E 只新增 `SpeechState.WAITING` 枚举值，**没有新增事件类型** |
| `core/event_bus.py` | 每订阅者独立队列 + 消费协程 | 发布只入队；异常隔离；队列满丢最旧 |
| `core/models.py` | `UserMessage`/`AssistantMessage`/`SessionInfo`/`TaskInfo`/`ProviderError` | 全部不可变；`extra="forbid"` |
| `providers/base.py` | `LLMProvider` Protocol、`LLMRequest`、`TextDelta`、`ProviderHealth`、`ProviderChunk`/`ProviderCompletion`、`ToolDefinition`/`ToolCallProposal`、`StructuredOutput` | 取消 = 取消消费任务；工具调用只解析不执行 |
| `providers/mock_llm.py` | 分块流式假模型、确定性回复 | 零网络、零模型文件；结构化输出明确报"不支持" |
| `providers/llama_cpp.py` | 真实 llama-server 的流式/非流式 Provider、健康与能力探测、SSE 解析、错误翻译、结构化输出探测 | 只依赖 httpx + core；不加载 GGUF、不管理服务进程 |
| `providers/tts_base.py` | `SpeechProvider`/`SpeechStream` 协议、`AudioFormat`、`AudioChunk`、`SpeechRequest`、`SpeechMetrics` | 音频二进制不进入事件/模型 dump；帧对齐强制校验 |
| `providers/qwen3_tts.py` | **04D 主候选**：`qwentts.cpp` GGML Qwen3-TTS HTTP 适配（一段文本一次 `POST /v1/audio/speech`、串行锁、WAV 格式探针、三段超时、取消=断连、`stream.stats`） | 只依赖 httpx + core；不加载权重、不管理进程；`max_batch=1` 串行 |
| `providers/moss_tts.py` | MOSS-TTS-Realtime HTTP 适配（**experimental**）：`/health` 探测、一会话一回合、推文本、收 PCM、格式校验、错误码、取消/关闭 | 只依赖 httpx + core；不加载权重、不管理服务进程 |
| `providers/mock_tts.py` | 零网络 Mock TTS（24 kHz mono s16le 正弦/静音，可配块大小/延迟/失败注入） | 测试与"没有 TTS 服务也能演示播放链路" |
| `providers/tts_factory.py` | `providers.tts.type` → Provider（none/mock/moss_tts/qwen3_tts） | 缺 `base_url` 防御性 `ConfigError`；MOSS 创建时打 experimental warning |
| `speech/segmenter.py` | 流式文本切分：强/弱边界、首段更短、`max_chars` 硬切、`flush`、轻量清洗 | 不改写句子；跨 feed 的 markdown 靠"缓冲原文 + 重清洗" |
| `speech/player.py` | 单条 `sounddevice.OutputStream` + 回调；线程安全队列、背压、硬上限、静音、欠载统计、WAV 录音、设备降级 | 设备打开在工作线程（5 s 超时）；回调不 await、不碰 asyncio |
| `speech/coordinator.py` | 一轮播报编排：段队列 → pump（open_stream/push→audio）→ player；发 `speech.*` 事件；取消/降级/超时 | 事件按 FIFO 单发射器串行发布；迟到块按 utterance/epoch 丢弃 |
| `speech/scheduler.py` | **04E**：`GpuScheduler`（`llm_priority`/`overlap`、单槽互斥、显存预算、`snapshot()`）+ `VramProbe`（`nvidia-smi`，只读、TTL 缓存） | 应用级任务仲裁，不是 CUDA/显存隔离；等待全事件驱动、可取消 |
| `services/base.py` | **04E**：`ServiceState`/`ServiceStatus`/`ManagedService` 协议（Phase 13 通用 ServiceManager 的复用点） | 只描述生命周期与诊断，不绑定具体服务 |
| `services/qwen_tts_runtime.py` | **04E**：Qwen TTS 进程托管（`external`/`managed`/`disabled`）：启动前检查、argv 分离启动、日志落盘、健康轮询、有限重启、只杀自己启动的进程 | 禁止 `shell=True`；失败给可读诊断、不阻塞 GUI |
| `ui/app_shell.py` | 装配面板/桌宠/托盘/热键；关闭策略；窗口状态读写；错误状态保留 | UI 线程；不碰 Provider/文件（除状态文件） |
| `ui/ui_state.py` | 从 `AppEvent` 归约出聊天记录、助手状态、健康、错误、任务标志 | 唯一 UI 状态源；事件只进这里一次 |
| `ui/main_window.py` | 交互面板：状态/健康/会话、聊天气泡、输入与发送/停止、设置入口 | 只渲染 `UiState`；不推断业务状态 |
| `ui/chat_view.py` | 气泡渲染、流式光标、自动滚动 | 只有用户原本在底部才跟随 |
| `ui/pet_window.py` | 透明无边框桌宠：QPainter 五态动画、拖动、缩放、穿透、右键菜单 | 只负责画与输入；不感知 Provider |
| `ui/pet_state.py` | `AssistantStatus` → `PetState` 映射 | `READY`→`IDLE`；`ERROR` 优先；`task_active` 只覆盖空闲 |
| `ui/tray.py` | 托盘图标与菜单（面板/桌宠/置顶/穿透/设置/退出） | 托盘不可用时信号仍可测，但 `show()` 不执行 |
| `ui/settings_dialog.py` | 窗口行为（生效）+ 主题/音量/语音输入占位（禁用） | 占位项不写配置、不发后端事件 |
| `ui/hotkey.py` | 快捷键文本解析、Win32 注册、Qt 原生事件过滤 | 失败不抛；无后端时发 `registration_failed` |
| `ui/window_state.py` | 窗口状态 JSON（原子写/坏文件兜底）+ 多屏位置回收 | 读写绝不抛；不假设主屏在原点 |
| `ui/theme.py` | 默认主题 QSS、状态色、程序生成图标 | 当前只有一套主题 |
| `ui/qt_event_bridge.py` | asyncio 线程 ↔ Qt 信号；非阻塞提交/取消 | 后台不碰 Widget；UI 不等 Future |
| `infrastructure/logging.py` | 控制台 + 轮转文件日志、JSON 格式、密钥脱敏 | 日志目录不可写时退化为仅控制台 |
| `infrastructure/paths.py` | 平台路径推导（`%APPDATA%`/`%LOCALAPPDATA%`） | 新增 `user_state_path()`；不硬编码用户目录 |
| `scripts/*.py` | 视觉截图与真实平台冒烟 | 不参与应用运行路径 |

## 3. 线程模型与关闭

```text
UI 线程（Qt 事件循环）
  ├─ 用户点击/按键/拖动 → QtEventBridge.submit_text()/cancel_active()
  │        └─ AsyncAppRuntime.submit() → run_coroutine_threadsafe()
  ├─ bridge.event_received 信号 → AppShell → UiState.apply_event()
  └─ UiState 信号 → MainWindow（气泡/状态）与 AppShell（桌宠状态）

asyncio 线程（companion-asyncio）
  └─ loop.run_forever()
       ├─ Orchestrator._run_turn 任务（每轮一个）
       └─ EventBus 每个订阅者一个消费协程
            └─ QtEventBridge._forward_event → event_received.emit()
```

为什么不用 qasync（Phase 01 决策保留）：UI 线程与推理/音频/工具线程必须彻底隔离；只有一条
asyncio 循环（在工作线程），不存在两套事件循环互相踩的问题，关闭顺序也更容易验证。
代价是跨线程必须走 `run_coroutine_threadsafe`，这条边界被收敛在 `QtEventBridge` 一处。

并发约束：`Orchestrator.submit_user_message()` 用 `asyncio.Lock` 把"取消上一轮 → 建新轮"
包成临界区。回归测试：`test_concurrent_submits_are_serialized`。

关闭顺序（`main._run_gui` 的 `finally`）：

1. Qt 事件循环退出（窗口关闭触发，或 `AppShell.request_quit()`）；
2. `AppShell.shutdown()`：保存窗口状态 → 注销全局热键 → 隐藏托盘 → 停桌宠动画 → 隐藏窗口；
3. `QtEventBridge.shutdown()`：在 asyncio 线程注销事件订阅；
4. `AsyncAppRuntime.stop()` 在 asyncio 线程依次执行：
   `orchestrator.stop()`（取消进行中的轮次并发出收尾事件）→ `bus.drain(1.5s)` →
   `bus.aclose()`（取消所有消费协程）→ `provider.aclose()`；
   每一步有独立超时（总预算均分），超时记入 `abandoned_steps` 后继续下一步；
5. 取消 loop 中所有残留任务并 `gather`，统计 `leftover_tasks`；
6. `loop.stop()` → `thread.join(剩余预算)` → 记录 `ShutdownReport`。

总预算耗尽时会取消关闭协程，并用 `leftover_tasks=("<unknown: shutdown aborted>",)`
明确标出"残留未知"——绝不假装干净退出。

## 4. 事件协议（Phase 04 扩展 `speech.*`）

### 4.1 信封

```json
{
  "event_id": "uuid4",
  "event_type": "assistant.text.delta",
  "session_id": "uuid4 | null",
  "request_id": "uuid4 | null",
  "task_id": "uuid4 | null",
  "timestamp": "2026-11-17T06:31:22.123456Z",
  "payload": {}
}
```

- `timestamp`：`AwareDatetime`（必须带时区），序列化为 ISO 8601（UTC 输出为 `Z`）。
- `payload`：构造时即用 `json.dumps(..., allow_nan=False)` 校验。
- 顶层未知字段：**允许**（`extra="allow"`）并原样保留；业务载荷放 `payload`。
- `event_type`：严格枚举；新增类型属于协议变更，需要评审后再加。
- 音频/二进制数据不走事件总线（任务书 04 时设计）。

### 4.2 类型与 payload 形状

| event_type | payload | 说明 |
| --- | --- | --- |
| `user.message.created` | `{"message": UserMessage}` | 用户消息落定 |
| `assistant.status.changed` | `{"status", "previous", "reason"?}` | 状态机迁移；UI/桌宠的状态唯一来源 |
| `assistant.text.delta` | `{"delta": str, "index": int}` | 流式增量 |
| `assistant.text.completed` | `{"message", "finish_reason", "chunk_count", "duration_ms"}` | 一轮收尾；`finish_reason ∈ completed/cancelled/timeout/failed` |
| `system.health.changed` | `{"component", "provider", "healthy", "detail", "checked_at", "latency_ms"?}` | 启动时检查一次 Provider |
| `system.error` | `{"error": ProviderError, "severity", "user_message", "context"?}` | 只带用户可读短句；堆栈只进日志 |
| `speech.state.changed` | `{"state", "previous", "muted", "detail"?}` | 语音状态（disabled/idle/**waiting**/synthesizing/playing/muted/degraded）；`waiting` = 段已切好、按 GPU 策略在等本地 LLM |
| `speech.synthesis.started` | `{"utterance_id", "segment_index", "text_chars", "voice", "format"}` | 每段开始合成 |
| `speech.audio.chunk` | `{"utterance_id", "index", "bytes", "duration_ms", "buffered_ms", "format"}` | **默认不发**（`speech.emit_chunk_events=false`，避免淹没总线）；只带元数据 |
| `speech.playback.started` | `{"utterance_id", "buffered_ms", "first_audio_latency_ms"?}` | 首个音频块进入播放 |
| `speech.playback.completed` | `{"utterance_id", "played_ms", "audio_ms", "chunks", "underruns", "dropped_chunks", "duration_ms"}` | 播报结束（含设备统计） |
| `speech.cancelled` | `{"utterance_id", "reason", "dropped_segments", "played_ms"}` | 停止/barge-in/关闭 |
| `speech.error` | `{"error": ProviderError, "user_message", "degraded", "utterance_id"}` | TTS 故障（降级为纯文字，不占用文字错误横幅）；04E 新增调度拒绝码 `tts_scheduler_deferred` / `tts_vram_budget`（同样是降级） |

`format` 一律是 `AudioFormat.as_metadata()` 形状：`{sample_rate, channels, sample_format, frame_bytes}`；
**PCM 永不进入事件总线**（任务书 04 §3，总纲 §8）。

Phase 02 的桌宠 `TaskRunning` 没有新增事件类型：`UiState` 按字符串识别未来的
`task.accepted / task.progress / task.approval_required / task.completed / task.failed /
task.cancelled`，把 `task_active` 暴露给桌宠。当前阶段这些事件没有产生者，因此桌宠不会进入该状态。

### 4.3 UI 侧归约规则（`UiState.apply_event`）

- 用户消息 → 新气泡（`message_id`）；
- 增量 → 当前 `request_id` 的助手气泡；**非当前请求、已收尾请求的迟到增量直接丢弃**
  （Phase 01 的渲染防御原样保留）；
- 收尾 → 用 `message.text`（权威全文）覆盖增量拼接结果，标记 `finish_reason`；
  首个 token 前就取消/超时且没有内容时不画空气泡；
- 状态 → 面板状态 + 桌宠状态（`ready` 与 `idle` 对桌宠都是 Idle）；
- 健康/错误 → 面板横幅 + 桌宠错误态（保留约 2.5s 展示，见 §7.4）。

## 5. 一轮对话的时序

### 5.1 正常路径

```text
UI 发送 → Orchestrator.submit_user_message
  ├─ publish user.message.created
  ├─ status: idle → thinking（request_started）
  ├─ provider.stream_text 逐块
  │    ├─ 首块：status thinking → speaking（first_token）
  │    ├─ 每块：speech.feed_text(delta)   ← Phase 04：同步喂给语音层（切段/合成在后台）
  │    └─ 每块：publish assistant.text.delta
  ├─ publish assistant.text.completed（finish_reason=completed）
  ├─ status: speaking → ready（text_completed）
  ├─ speech.finish_text() → pump 收尾（push final）→ 播报
  ├─ speech.state: synthesizing → playing → idle；publish speech.playback.completed
  └─ status: ready → idle（turn_finished / speech_timeout / speech_degraded）
```

状态事件序列（含启动时的一次 `idle`）为：
`idle(startup) → thinking → speaking → ready → idle`；`ready` 期间 `UiState.busy=False`，
用户可以继续输入（新一轮会先停播报再取消本轮）。TTS 未启用时 `speech.*` 完全不出现，
序列与 Phase 03 一致。

### 5.2 取消 / 抢占

- 用户点「停止」→ `cancel_active()` 取消轮次任务；Provider 的异步生成器在 `await` 点收到
  `CancelledError`，不再产生新块；
- 收尾在 `finally` 中完成：`assistant.text.completed(finish_reason=cancelled)` +
  `status: idle(cancelled)`；
- 收尾期间 `_finishing=True`，`cancel_active()` 不会打断收尾事件；
- 任务尚未执行第一步就被取消时，`_finalize_if_never_started()` 补发收尾事件；
- 新一轮消息到达时先取消并等待上一轮（barge-in），临界区由 `_submit_lock` 串行化；
- UI 在忙碌时禁用发送（Phase 06 再设计队列/多任务）。

### 5.3 失败与超时

- `ProviderFailure` → `system.error` → `status: error` → `idle(error_recovered)`，
  并发出 `assistant.text.completed(finish_reason=failed)`；
- 超时（`asyncio.timeout(request_timeout_s)`）→ `system.error(code=provider_timeout)`，
  `finish_reason=timeout`；
- 未预期异常 → 记录堆栈，事件侧只暴露 `internal_error` 短句。
- **播报中被打断**（用户停止 / 新一轮 barge-in）：`speech.cancel()` 立即静音并清空队列，
  `speech.cancelled` 恰好一条；文本收尾事件已经发过（`completed`），不会重复发。
- **播报等待超时**（`speech.playback_wait_timeout_s`）：协调器主动 `player.stop(clear=True)`
  并发 `speech.cancelled(reason=playback_timeout)`，Orchestrator 把状态收回 `idle`，
  不阻塞下一轮。
- **TTS 故障**：`speech.error(degraded=True)` + 状态 `degraded`，本轮不再重试合成；
  文字与状态机不受影响（`finish_reason` 仍是 `completed`）。

## 6. 状态机与桌宠映射

```text
        ┌──────┐  request_started   ┌──────────┐  first_token   ┌──────────┐
        │ idle │ ─────────────────▶ │ thinking │ ─────────────▶ │ speaking │
        └──────┘                    └──────────┘                └────┬─────┘
            ▲                                                       │ text_completed
            │ turn_finished / cancelled / error_recovered           ▼
            │                                                  ┌────────┐
            └──────────────────────────────────────────────────│ ready  │
                                                               └────────┘
   失败/超时：thinking|speaking → error → idle
```

| AssistantStatus | PetState | 说明 |
| --- | --- | --- |
| `idle` / `ready` | Idle | 对话静默 |
| `thinking` | Thinking | 已提交、等待首 token |
| `speaking` | Speaking | 正在接收文本增量 |
| `error` | Error | 错误横幅 + 桌宠红色状态 |
| （未来的 `task_active`） | TaskRunning | 仅当助手状态是 idle/ready 时显示 |

`READY` 保留给音频阶段：文本完成 → `ready`（开始播报）→ `idle`（播报结束/超时/降级）；
桌宠把 `ready` 合并成 Idle，因此"播报中"在桌宠上仍显示为 Idle（面板头部有独立的「语音：播放中…」）。

## 7. UI 窗口层设计（Phase 02 新增）

### 7.1 共享状态：`UiState`

`AppShell` 在构造时把 `QtEventBridge.event_received` 接到 `UiState.apply_event`；面板与桌宠
都只从 `UiState` 取数据，因此**不存在两份聊天历史**。`UiState` 是 QObject，信号包括
`message_added/updated/finished`、`status_changed`、`health_changed`、`error_raised`、
`task_active_changed`，并提供只读快照（`messages/status/health/task_active`）。

UI 事件过滤（当前请求、去重、迟到增量）集中在 `UiState`，窗口只负责渲染；任何事件处理异常
被捕获并记日志，不会掀翻 Qt 事件循环。

### 7.2 窗口与关闭策略：`AppShell`

`AppShell` 负责：

- 创建面板、桌宠、托盘、热键管理器；
- 合并配置默认值与 `ui-state.json` 中上次运行的状态；
- 恢复/夹取窗口位置（多屏、负坐标、任务栏工作区）；
- 协调置顶、穿透、大小、可见性与关闭行为；
- 退出时保存状态并注销热键。

关闭策略：

```text
面板关闭按钮 / Alt+F4
  ├─ close_to_tray=true 且托盘可用 → 隐藏面板 + 首次托盘提示（应用继续运行）
  ├─ close_to_tray=false（或托盘不可用）→ request_quit() → QApplication.quit()
  └─ 已在退出流程中 → 允许正常关闭
```

`QApplication.setQuitOnLastWindowClosed(False)`：窗口显隐由 AppShell 管理，只有 `request_quit()`
会退出事件循环。桌宠 Alt+F4 只隐藏（`closeEvent` 忽略），可从托盘/设置重新显示。
**托盘不可用时 `close_to_tray` 强制为 False**：没有入口的"隐藏到托盘"等于应用失联。

**可见性持久化用"意图"而不是 `isVisible()`**：Qt 在 `app.exec()` 退出阶段会**先**隐藏
顶层窗口（hideEvent 早于 `aboutToQuit`），此时窗口已经不可见；若在 `shutdown()` 里按
`isVisible()` 落盘，就会把"用户上次可见"写成 false，下一次启动只剩托盘。因此
`AppShell` 维护 `_panel_visible_intent` / `_pet_visible_intent`，只在用户动作
（显隐、关闭到托盘、设置/托盘/快捷键唤出）时更新，`_save_state()` 始终写意图。
`shutdown()` 里的 `_shutting_down` 守卫只是额外防御，不承担正确性。

### 7.3 置顶、鼠标穿透与停靠

- 桌宠窗口 flags：`FramelessWindowHint | Tool`，可选 `WindowStaysOnTopHint` / `WindowTransparentForInput`；
  `WA_TranslucentBackground`、`WA_ShowWithoutActivating`、`WA_QuitOnClose=False`；
- 改变 flags 会触发 Qt 的 hide/show；`PetWindow._apply_window_flags()` 保存并恢复位置/可见性，
  期间抑制 `visibility_changed`，避免 shell 看到假的显隐；
- **鼠标穿透默认关闭**；开启后桌宠不再接收鼠标事件，托盘/设置仍可关闭穿透，
  并给一次托盘提示（任务书要求"不要让窗口永久无法点击"）；
- 桌宠默认停靠在**主屏工作区**右下角（`availableGeometry` 已排除任务栏），
  状态文件只保存逻辑坐标；启动时按当前屏幕重新夹取。

### 7.4 错误状态展示保留

Orchestrator 在失败/超时会依次发出 `status: error` 和 `status: idle(error_recovered)`。
如果桌宠严格按事件即时切换，用户几乎看不到红色错误状态。`AppShell` 因此在 Error 上保留
`error_flash_ms`（默认 2.5s）的展示窗口：期间 `idle/ready` 不覆盖它；新的 `thinking/speaking`
（新一轮）会立即覆盖。**这只是展示层的时间窗，不改变任何业务状态**。

### 7.5 全局快捷键

```text
HotkeyManager.register("Ctrl+Alt+Space")
  ├─ parse_hotkey → HotkeySpec(modifiers, vk)     # 显式中文错误，不依赖 QKeySequence 平台行为
  ├─ WindowsHotkeyBackend.register → RegisterHotKey(None, id, mods|MOD_NOREPEAT, vk)
  ├─ 成功 → 安装 QAbstractNativeEventFilter，WM_HOTKEY + id 命中 → activated 信号
  └─ 失败 → registration_failed(可读原因)，AppShell 记日志 + 托盘气泡 + 设置页状态
```

- 原生事件过滤采用**组合**（独立 `_NativeEventFilter` 转发给 `HotkeyManager`），不是
  `HotkeyManager(QObject, QAbstractNativeEventFilter)` 多继承：PySide6 6.12 实测多继承下
  Qt 不会回调 Python 覆盖（详见 `hotkey.py` 注释）；
- 失败 fallback：托盘菜单（始终可用）+ 面板内 `QShortcut`（面板有焦点时用同一组合键聚焦输入框）；
- 注册/注销都幂等；退出时 `AppShell.shutdown()` 显式注销，避免系统里残留热键；
- 注册成功但装不上原生事件过滤器（例如没有 `QApplication`）时会回滚注册并发
  `registration_failed`，不会声称"已注册"；
- 面板置顶切换会触发 Qt 的 hide/show；`AppShell` 用 `QSignalBlocker` 抑制这段平台重建产生的
  假 `visibility_changed`（桌宠侧用 `_suppress_visibility_signal` 做同样的事）。

### 7.6 窗口状态持久化

`WindowStateStore` 写 `%APPDATA%\desktop-companion\ui-state.json`（可用 `AppShell` 的
`state_path` 注入测试路径），字段：

```json
{
  "schema_version": 1,
  "pet_x": 2100, "pet_y": 1200, "pet_size_px": 120,
  "pet_visible": true, "pet_click_through": false, "pet_always_on_top": true,
  "panel_x": 700, "panel_y": 240, "panel_width": 920, "panel_height": 680,
  "panel_visible": true, "panel_always_on_top": false,
  "close_to_tray": true
}
```

- **原子写**：同目录临时文件 + `os.replace`；写后回读校验；失败只 warning；
- **坏文件/坏字段/未知 schema**：回退默认值，绝不阻止启动；
- **多屏位置回收**：`clamp_point/clamp_geometry` 保证窗口至少 48×48 可见；
  完全离屏时收回到中心距离最近的屏幕（支持负坐标副屏）；
- 配置文件与状态文件分离：配置是"默认值/部署配置"，状态文件是"上次运行长什么样"。

### 7.7 主题与视觉

`theme.py` 定义唯一一套默认主题（QSS + 状态色 + 程序生成图标），设置页明确标注
"主题（Phase 02 仅一套）"为占位。桌宠的五态视觉在 `pet_window.py` 中用 QPainter 绘制：

- Idle：青蓝球 + 呼吸光环；
- Thinking：琥珀球 + 深琥珀旋转弧 + 轨道点；
- Speaking：绿球 + 三层扩散脉冲环 + 声波条；
- TaskRunning：紫球 + 8 段旋转外环；
- Error：红球 + 脉冲红环 + 几何「!」（不依赖字体，offscreen 下也不会变方块）。

`scripts/ui_preview.py` 与 `scripts/preview_pet_states.py` 用原生 `windows` 平台 +
`WA_DontShowOnScreen` 生成文档截图；透明合成/OS 级穿透/置顶层级仍需真实桌面目视确认。

## 8. 配置

- 模型：`AppConfig` 根 → `logging` / `ui` / `providers`（`llm` + `mock` + `llama_cpp`）/ `services`。
- `providers.llm`：`type ∈ {mock, llama_cpp}`、`model`、`base_url`、`request_timeout_s`；
  `type=llama_cpp` 时 `base_url` 必填（校验期报错，代码里没有默认端口）。
- `providers.llama_cpp`：采样（`temperature/top_p/max_tokens`）、`system_prompt`、
  `thinking ∈ {auto,on,off}`（YAML 的 `on/off` 布尔会被归一成字符串）、
  `tool_calling ∈ {auto,on,off}`（auto 读 `/props.chat_template_caps`，on/off 人工标记诊断）、
  `chat_template_kwargs`、超时/重试/连接池、`api_key_env`（只存变量名）、`context_window_tokens`
  （仅用于核对 `/props`）。
- `ui` 子模型：`pet`（可见性/尺寸/透明度/置顶/穿透/帧率/边距）、`panel`（可见性/置顶/关闭策略）、
  `tray.enabled`、`hotkey`（启用/组合）。
- `providers.tts`（Phase 04 / 04D）：`type ∈ {none, mock, moss_tts, qwen3_tts}`（默认 `none`——TTS 服务
  不能成为启动硬依赖）、`base_url`（`moss_tts` / `qwen3_tts` 必填）、`voice`（MOSS=服务端参考音频；
  Qwen=speaker 名或已注册克隆音色名，出现音频扩展名/路径会在配置期报错）、`language`（仅 Qwen 透传，
  null=不发）、`start_prebuffer_ms`（按后端覆盖全局 `audio.start_prebuffer_ms`，Qwen 300 / MOSS 2500）。
- `providers.moss_tts`：连接/控制/块间/整轮/健康/关闭超时、`expected_sample_rate/channels`（与响应头核对）。
- `providers.qwen3_tts`：`mode ∈ {external, managed, disabled}`（04E：`managed` = 本应用启动/停止
  `tts-server.exe`；`disabled` 同 `type=none`；`managed` 要求 `base_url` 是回环地址且
  `managed.executable` 必填）、`managed` 子模型（executable/model/codec/cwd/env/args/host/port/
  alias/lang/max_batch、`startup_timeout_s`/`shutdown_timeout_s`/`stop_on_exit`/
  `restart_on_crash`/`max_restarts`/`restart_backoff_s`/`health_monitor_interval_s`/
  `required_files`/`log_dir`）、`commit` / `expected_model`（诊断核对）、连接/等头/块间/单段预算/
  健康/等锁超时、`expected_sample_rate/channels`、`chunk_target_ms`（默认 80 → 3840 B）、
  `format_probe`（WAV 头核对）、`validate_voice`、
  `model_type ∈ {unknown, custom_voice, base, voice_design}`（`base` 直接拒绝 instructions）、
  `instructions`（experimental，默认空）、`max_batch`（记录用，仍串行）。
- `gpu`（04E）：`policy ∈ {llm_priority, overlap}`（默认 `llm_priority`）、`llm_local_gpu ∈ {auto,yes,no}`
  （auto 读 Provider 的 `resource` 元数据）、`llm_wait_timeout_s`、`probe_enabled/probe_interval_s/
  probe_timeout_s`（`nvidia-smi` 只读探测）、`whole_card_budget_mb`（默认 null = 不限制）、
  `budget_defer_timeout_s`、`tts_resident_budget_mb`/`tts_peak_budget_mb`（验收目标，只用于监控/报告）。
- `speech`：`enabled`、`muted`、切分（`min_chars/first_min_chars/max_chars/max_wait_ms`）、
  播放背压（`buffer_high_watermark_s/buffer_max_s`）、`playback_wait_timeout_s`、
  `emit_chunk_events`（默认关）、`sanitize`。
- `audio`：`output_device`（名字片段或索引字符串）、`blocksize`、`latency`、
  `start_prebuffer_ms`（0=立即播放；RTF>1 时用首延迟换连续播放；**可被 `providers.tts.start_prebuffer_ms`
  按后端覆盖**）、`underrun_hold_ms`（欠载后等回填再续播；0=旧行为，建议 1000–1500）、`record_dir`。
- 加载顺序：内置默认值 → `config/default.yaml` → 用户配置 → `.env` → 环境变量 `DESKTOP_COMPANION__*`。
- 用户配置查找：`--config` > `DESKTOP_COMPANION_CONFIG` > `%APPDATA%\desktop-companion\config.yaml`。
- 显式指定但文件不存在 → 直接报错；默认位置不存在 → 使用默认值。
- 校验失败会输出每个错误的**配置路径、原因、来源文件:行号（或环境变量名）、当前值**。
- 配置模型 `extra="forbid"`：未知键立刻失败。
- 环境变量值按 YAML 标量解析；数值字段拒绝布尔值；`logging.dir` 拒绝 UNC。
- 优先级按**层**而不是按键名字典序。
- 快捷键文本只做非空校验；语法在运行期解析，失败不阻止启动（记录警告 + 托盘 fallback）。

## 9. Provider 契约与扩展点

### 9.1 协议（Phase 03 扩展，向后兼容）

```python
class LLMProvider(Protocol):
    name: str

    async def health_check(self) -> ProviderHealth: ...  # 不得抛异常
    def stream_text(self, request: LLMRequest) -> AsyncIterator[TextDelta]: ...
    async def aclose(self) -> None: ...  # 幂等
    async def generate_text(self, request: LLMRequest) -> ProviderCompletion: ...  # Phase 03 新增
    async def generate_structured(self, request: LLMRequest, schema: dict[str, Any]) -> StructuredOutput: ...
```

- `Orchestrator` 仍然**只用 `stream_text`**；新增的两个方法服务于诊断/探测与后续短任务，
  不改变一轮对话的事件形状。
- 取消：没有单独的 `cancel(request_id)`；Orchestrator 取消消费任务，httpx 在 `CancelledError`
  传播时释放连接。取消只影响该请求，连接池与其它并发请求继续可用。
- 失败：Provider 抛 `ProviderFailure(ProviderError(...))`；`ProviderError.code` 是稳定短标识；
  `details` 只放已截断、可 JSON 序列化的诊断（响应体 ≤500 字符、绝不带密钥）。
- 协议扩展的类型都在 `providers/base.py`：`ProviderChunk`（text/done）、`ProviderCompletion`、
  `TokenUsage`、`ToolDefinition`、`ToolCallProposal`、`StructuredOutput`。
  **OpenAI wire format 只存在于 `llama_cpp.py` 内部**，不越过 Provider 边界。
- 新增实现：在 `providers/factory.py` 增加分支、扩展 `ProviderConfig.type` 的 `Literal`
  （`llama_cpp` 要求显式 `base_url`，配置校验期失败）。Orchestrator/UI 不需要改。

### 9.2 `LlamaCppProvider` 的行为约定

```text
base_url=http://host:port[/v1]
  → 聊天 POST {root}/v1/chat/completions（base_url 已带 /v1 时不重复）
  → 探测 GET  {root}/health、{root}/props、{root}/v1/models
```

- **请求体**：`messages`（可选 system 前缀）、`temperature`、`top_p`、`max_tokens`、`stream`、
  `stream_options.include_usage`（流式）、`chat_template_kwargs`（`thinking != auto` 时注入
  `enable_thinking`）、`tools`（仅调用方显式传入）、`response_format`（仅结构化输出）。
  `model` 只在配置/请求给出时出现；没有任何模型名硬编码。
- **SSE 解析**：自己按字节读（`aiter_bytes`）+ 增量 UTF-8 解码 + 只按 CR/LF 切行（不用
  `aiter_lines()`：`str.splitlines()` 会把 U+2028/U+2029/U+0085 当行分隔符，模型正文含这些字符时
  JSON 帧会被劈开）；`data:` 累积/空行成帧；`[DONE]` 与 `finish_reason` 任一出现即认为流正常结束
  （只有 `[DONE]` 时 `finish_reason=unknown`，不猜成 stop；收到 `[DONE]` 后主动停止读取，
  不等服务端关连接）。两者都没有 → `provider_stream_interrupted`；收尾标记之后才发生的传输错误/
  超时不算失败。
  - `delta.content` → `TextDelta`（`stream_text` 唯一产出）；
  - `delta.reasoning_content` → 只累计字符数（陪伴 UI 不显示思维链），收尾块带 `reasoning_chars`；
  - `delta.tool_calls` → 按 `index` 归并成 `ToolCallProposal`（参数 JSON 解析失败只填 `parse_error`，
    不抛异常、不执行工具；权限与执行属于任务书 07）；
  - `usage`/`timings`/`model` → 收尾块/完成结果。
- **重试边界**：只在"还没产出任何文本块"且错误 retryable 时重试，默认 0 次；已输出文本后绝不重试
  （会重复内容）。400/401/403/上下文超限不重试。模板拒绝 `chat_template_kwargs`（HTTP 400/422/500 +
  正文里的 jinja/chat template/enable_thinking；**不**把裸 `undefined` 当模板拒绝）时去掉该参数
  重试一次，之后本进程不再发送该参数，并记录 `template_kwargs_fallback`。
- **超时**：连接/读写/池超时全部 → `provider_timeout`（retryable）；Orchestrator 把它映射成
  `finish_reason=timeout`，UI 不会把"超时"显示成"失败"。整轮预算仍由
  `providers.llm.request_timeout_s` + `asyncio.timeout` 管。
- **健康/能力探测**：只做 GET，按 `probe_ttl_s`（默认 30s）缓存，**不抛异常**；`ProviderHealth.detail`
  是给 UI 的单行摘要（版本 · 模型 · n_ctx · tools），探测不到的项写"未知"，不猜版本号。
  `/props` 的 `chat_template` 只记录长度 + SHA-256 前 12 位（避免日志里塞 9KB 模板）。
- **上下文**：llama-server 的上下文只由启动参数 `-c` 决定；`context_window_tokens` 仅用于核对
  `/props.n_ctx`，不一致时 warning + 摘要标注，不改变服务端行为。
- **代理**：httpx 客户端 `trust_env=False`。本地回环不应该走代理；另外 httpx 0.28 解析
  `NO_PROXY` 里的 `[::1]` 会抛 `InvalidURL`（本机实测）。线上 Provider 需要代理时自己显式配置。
- 错误码见 `providers/llama_cpp.py::_http_failure` 与 `.dsh/notes/phase03-contract.md` §8：
  `provider_unreachable / provider_timeout / provider_http_error / provider_auth_error /
  provider_unavailable / provider_context_overflow / provider_invalid_request /
  provider_stream_interrupted / provider_invalid_response / provider_closed /
  structured_output_unsupported / structured_output_invalid`。

### 9.3 诊断工具

- `scripts/llama_probe.py`：对真实 llama-server 跑健康/诊断、非流式、流式、工具调用解析、
  结构化输出探测、取消 + 复用，输出人类可读摘要并可 `--json` 落盘。
- `docs/llama-server-probe.md` 记录一次真实运行（b11849 / Qwen3.8-27B / n_ctx 131072）的版本、
  Chat Template 指纹、能力、延迟数字与已知限制；原始报告在 `docs/llama-server-probe.json`。

### 9.4 TTS Provider 契约（Phase 04 / 04D）

```python
class SpeechProvider(Protocol):
    name: str
    format: AudioFormat
    async def health_check(self) -> ProviderHealth: ...      # 不抛
    async def open_stream(self, request: SpeechRequest) -> SpeechStream: ...
    async def aclose(self) -> None: ...                      # 幂等

class SpeechStream(Protocol):
    request_id: str; utterance_id: str; format: AudioFormat
    async def push_text(self, text: str, *, final: bool = False) -> None
    def audio(self) -> AsyncIterator[AudioChunk]
    async def aclose(self) -> None
```

`Qwen3TTSProvider`（**04D 主候选**，`qwentts.cpp` GGML；实测见 `docs/qwen3-tts-provider.md`）：

```text
open_stream(initial_text)  POST /v1/audio/speech {input=initial_text, voice, language, response_format="pcm"}
                           → 等响应头（400/坏 Content-Type/502 在这里就抛）
push_text(seg)             只入队（服务端不支持文本增量）：每段文本各自一次请求
audio()                    POST /v1/audio/speech（逐段、持串行锁）← audio/pcm（s16le/24k/mono）
aclose()                   关闭当前响应（= 服务端断连取消）+ 唤醒队列；幂等
health_check()             GET /health + /v1/models + /v1/audio/voices + WAV 格式探针（TTL 缓存）
```

- **一段文本 = 一个请求**：`SpeechStream` 仍代表"一轮发言"，但内部按段发请求、音频顺序拼接，
  `index` 跨段单调；`SpeechCoordinator`/`AudioPlayer` 不需要知道这个区别。
- **格式验证**：服务端没有采样率/声道响应头 → `expected_*` + `response_format=wav` 的 RIFF 头探针
  核对 24 kHz / mono / 16 bit；不一致判不健康。
- **串行**：服务端 `max_batch=1`，Provider 级 `asyncio.Lock`；等锁超时 → `tts_busy`。
- 错误码（Qwen）：`tts_unreachable / tts_timeout / tts_http_error / tts_unavailable /
  tts_invalid_response / tts_invalid_format / tts_closed / tts_stream_interrupted / tts_text_empty /
  tts_unknown_voice / tts_instruct_unsupported / tts_invalid_request / tts_synthesis_failed / tts_busy`。

`MossTTSProvider`（**experimental**：不是默认服务，实时播放不达标，见 §12.3）的映射（实测见 `docs/moss-tts-probe.md`）：

```text
open_stream()  POST /tts/session/start  {session_id=新 uuid, assistant_text=initial_text, prompt_audio=voice}
push_text()    POST /tts/session/push   {session_id, text, is_final}
audio()        GET  /tts/session/{sid}/audio   ← pcm_s16le / mono / 24 kHz（响应头校验）
aclose()       POST /tts/session/close
health_check() GET  /health（TTL 缓存，detail="MOSS-TTS · sr=… · device · attn"）
```

- 一个 utterance = 一个服务端会话（多段文本推进同一回合，音色/韵律连续；不跨轮复用 KV）。
- 音频按 `frame_bytes` 对齐：跨块半帧拼起来、末尾残帧丢弃并 warning；`AudioChunk` 校验整帧。
- 错误码（MOSS）：`tts_unreachable / tts_timeout / tts_http_error / tts_unavailable / tts_invalid_response /
  tts_invalid_format / tts_session_lost / tts_closed / tts_stream_interrupted / tts_text_empty`。
- `MockTTSProvider` 提供同一契约的零网络实现（24 kHz mono s16le 正弦/静音），测试与演示用。

### 9.5 GPU 调度与服务运行管理（Phase 04E）

```python
class GpuScheduler:                     # speech/scheduler.py
    def note_llm_generating(self, active: bool) -> None      # Orchestrator 同步调用
    async def acquire_tts(self, *, utterance_id=None) -> SlotResult
    def release_tts(self) -> None                            # 幂等
    def snapshot(self) -> dict[str, Any]                     # JSON 安全观测

@dataclass(frozen=True)
class SlotResult:
    decision: Literal["granted", "overlap", "budget_denied", "timeout"]
    waited_ms: float; reason: str; llm_busy: bool; vram_used_mb: int | None

class ManagedService(Protocol):          # services/base.py（Phase 13 复用点）
    async def start(self) -> ServiceStatus
    async def stop(self) -> ServiceStatus
    def status(self) -> ServiceStatus
    async def ensure_ready(self, *, timeout: float | None = None) -> bool
```

- **协作关系**：Orchestrator 在 `stream_text` 前后 `note_llm_generating(True/False)`；
  `SpeechCoordinator._pump` 在 `open_stream()` 之前 `await acquire_tts()`，在 `finally` 里
  `release_tts()`；`SpeechState.WAITING` 只在"真的等待"时出现。
- **拒绝（`TIMEOUT`/`BUDGET_DENIED`）不改变文字结局**：`speech.error(degraded=True)`，
  错误码 `tts_scheduler_deferred`/`tts_vram_budget`，本轮 `assistant.text.completed` 仍是 `completed`。
- **单槽互斥**：同一时刻只允许一路 TTS 合成（对应服务端 `max_batch=1`）；第二个等待者可取消。
- **运行管理**：`bootstrap._build_speech` 仅在 `type=qwen3_tts && mode=managed` 时构造
  `QwenTtsRuntime`，由 `SpeechCoordinator.start()` 以**后台任务**启动（冷启动加载模型不阻塞 GUI），
  `aclose()` 负责停止（`stop_on_exit=false` 时只 detach）。`external` 实例从不被终止。
- **观测**：`SpeechCoordinator.diagnostics()` = 播报状态 + 段队列 + 调度器 `snapshot()` + 服务状态；
  `AsyncAppRuntime.diagnostics()["speech"]` 透出它（脚本/日志用，不产生事件）。

## 10. 错误处理与日志

- 错误分层：`CompanionError`（可预期）→ `ConfigError`、`EventBusClosedError`、
  `MessageValidationError`、`ProviderFailure`、`RuntimeNotStartedError`、
  `OrchestratorNotStartedError`。
- 事件侧只给用户可读短句；完整堆栈写日志。
- UI 侧：`UiState` 捕获事件处理异常并记日志；`MainWindow` 用横幅展示 `error_raised`；
  桌宠用 Error 状态；提交失败时把用户输入还原到输入框。
- 日志：控制台 + `%LOCALAPPDATA%\desktop-companion\logs\app.log`（5 MB × 3 轮转），
  行格式含时间、级别、logger 名、行号、消息；可选单行 JSON；密钥脱敏不覆盖异常堆栈（已知限制）。

## 11. 测试策略

| 文件 | 覆盖点 |
| --- | --- |
| `tests/test_config.py` | 默认值、分层覆盖、`.env` 优先级、来源行号、未知键、坏环境变量、脱敏、Phase 02 UI 配置范围、Phase 03 `providers.llama_cpp`（base_url 必填/未知键/连接池约束/YAML on-off 归一） |
| `tests/test_events.py` | 序列化往返、时区、扩展字段、payload JSON 约束 |
| `tests/test_event_bus.py` | 多订阅者顺序、注销、失败隔离、慢订阅者、队列溢出、关闭语义 |
| `tests/test_mock_provider.py` | 分块拼接、索引、健康检查、取消传播、非流式结果、结构化输出"不支持" |
| `tests/test_orchestrator.py` | 状态序列、增量拼接、取消、barge-in、失败/超时、空消息、会话快照、Provider 超时映射 |
| `tests/test_runtime.py` | 线程运行时、幂等停止、停止后提交报错、活动轮次下关闭无残留 |
| `tests/test_logging.py` | 脱敏、文件/JSON 格式、重复初始化 |
| `tests/test_cli.py` | `--check` 退出码、脱敏输出、配置错误诊断 |
| `tests/test_fake_llama_server.py` | 假服务器自身 sanity：路由、SSE 分块、请求记录、断连/挂起可配 |
| `tests/test_llama_cpp_provider.py` | llama.cpp Provider：健康/探测缓存、流式顺序、思考不产出、工具片段归并、错误码映射、超时/断流/畸形帧、重试边界、取消隔离与并发、base_url 三种写法、结构化输出三态、Orchestrator 端到端 |
| `tests/fake_moss_tts_server.py` | 真 asyncio TCP 假 MOSS：五个端点、可配块大小/间隔/首字节延迟/错误状态/错误格式头/断连/不回应；记录文本顺序与 `new_turn` |
| `tests/test_moss_tts_provider.py` | MOSS Provider：健康/TTL、URL 归一、start+push+audio 顺序、多段顺序、帧对齐/半帧/残帧、格式头校验、HTTP/连接/超时/断流错误码、取消隔离、aclose 幂等、voice/initial_text 透传 |
| `tests/fake_qwen_tts_server.py` | 真 asyncio TCP 假 `qwen3-tts`：`/health`、`/v1/models`、`/v1/audio/voices`、`POST /v1/audio/speech`（`pcm` chunked / `wav` RIFF）；可配 wire 块大小/间隔/首字节延迟、400(unknown voice/instruct)/502、错误 Content-Type、断连、不回应、空音频、参考 WAV 头；记录请求 JSON 与客户端断开 |
| `tests/test_qwen3_tts_provider.py` | Qwen Provider（**72 项**）：健康/TTL/WAV 格式探针（一致/不一致/超时）、请求体字段、initial_text 预发、多段=多请求顺序与 index、半帧拼接/残帧、`tts_busy` 串行锁、400/502/503 映射、三段超时、断连、取消（服务端看到断开 + 下一轮可用）、aclose 幂等 |
| `tests/test_mock_tts.py` | Mock TTS：PCM 长度/格式、块大小/延迟、失败注入、工厂分支（none/mock/moss_tts/qwen3_tts） |
| `tests/test_tts_segmenter.py` | 切分：强/弱边界、首段阈值、连续标点、超长无标点、换行、代码块跨 feed、清洗、flush、纯标点 |
| `tests/test_audio_player.py` | 播放器：整帧/背压/硬上限/欠载/静音/停止/完成/超时/设备降级/WAV/预缓冲 |
| `tests/test_speech_coordinator.py` | 协调器：段序、首段短切、max_wait、尾段不丢、barge-in、失败降级、事件顺序、静音、aclose |
| `tests/test_ui_state.py` | 事件归约、去重/迟到丢弃、权威全文覆盖、状态/健康/错误、未来 task.* 兼容 |
| `tests/test_gpu_scheduler.py` | **04E**：调度器 28 项——llm_priority 等待/唤醒/超时/取消、overlap 放行、远端 LLM 不门控、显存预算延迟与拒绝、探针不可用放行、单槽互斥、计数与 snapshot、release 幂等 |
| `tests/test_qwen_tts_runtime.py` | **04E**：运行管理 40 项（真子进程）——external/disabled 不 spawn、managed 启动/停止/幂等、缺 exe/model/codec/DLL/required_files、端口占用（含 connect 失败时的 bind 兜底）、启动超时、早退退出码（含 0xC0000135）、`stop_on_exit=false` detach、崩溃重启上限、健康巡检、日志/tail、artifacts 形状、取消启动的清理 |
| `tests/test_speech_scheduling.py` | **04E**：协调器门控（LLM busy 时 0 次 `open_stream`、唤醒后恰好 1 次）、`WAITING` 状态、调度拒绝降级不改文字结局、取消/新轮隔离、服务后台启停与健康 detail、`diagnostics()` JSON、Orchestrator 的 `note_llm_generating` 时序（正常/取消/异常） |
| `tests/test_chat_view.py` | 气泡渲染、流式光标、自动滚动与"翻历史不被拽回"、清空 |
| `tests/test_pet_window.py` | 五态映射/绘制、拖动/单击、右键菜单、缩放、穿透/置顶 flags、动画启停 |
| `tests/test_ui_widgets.py` | 托盘菜单信号、设置对话框（生效项/占位项/信号阻塞同步） |
| `tests/test_window_state.py` | 缺文件/坏 JSON/坏字段、原子写、多屏夹取（负坐标/离屏/部分可见） |
| `tests/test_hotkey.py` | 解析合法/非法组合、后端成功/失败/异常、替换/幂等、WM_HOTKEY 转发 |
| `tests/test_app_shell.py` | 共享状态、错误保留、托盘显隐/关闭策略、设置实时生效、状态恢复/夹取、端到端 Mock 回合 |
| `tests/test_architecture.py` | 后台不 import Qt、UI 不 import Provider/Tool/语音后端、只有 main/ui 创建 QApplication |
| `tests/test_ui_smoke.py` | offscreen 真窗口 + 真 asyncio：增量显示、停止冻结、Enter/Shift+Enter、错误还原、关闭无残留 |

UI 测试用 `QT_QPA_PLATFORM=offscreen`（`tests/conftest.py` 在任何 PySide6 导入前设置）。
PySide6 是硬依赖：缺 Qt 平台插件时 UI 测试**默认失败**；只有显式设置
`DESKTOP_COMPANION_SKIP_UI=1` 才会 skip 并给出原因，避免"没跑 UI 却全绿"。

当前规模：**测试计数见 `docs/IMPLEMENTATION_STATUS.md`**（Phase 03 收口时全量 `uv run pytest`）。
Provider 相关测试用一个真 asyncio TCP 假服务器（`tests/fake_llama_server.py`），不需要真实模型。

真实平台证据（手动脚本，不进入 pytest）：

- `scripts/gui_smoke.py`：原生 `windows` 平台、Windows 10 19045、屏幕 2560×1440
  （available 2560×1409，DPR 1.5）；托盘可用、`Ctrl+Alt+Space` 注册成功；
  1.2s 后退出：`leftover_tasks=()`、`errors=()`、`thread_alive=False`、`abandoned_steps=()`、
  状态文件已写入、热键已注销 → `RESULT: OK`；
- `scripts/ui_preview.py` / `scripts/preview_pet_states.py`：原生平台离屏渲染截图（`docs/phase02-*.png`）；
- `scripts/llama_probe.py`：真实 llama-server（b11849 / Qwen3.8-27B / n_ctx 131072）健康、非流式、
  流式、工具调用、结构化输出、取消全部通过；记录见 `docs/llama-server-probe.md`。

## 12. 已知限制

### 12.1 Phase 02 遗留（有意为之 / 未验证）

1. **会话只在内存中**，重启即丢（任务书 11）。
2. **没有真实 TTS/STT/音频**；`speaking` 只是逻辑状态，桌宠绿色状态不代表真的在发声。
3. **TaskRunning 没有事件来源**：渲染与映射已就绪，等任务书 12 的 `task.*` 事件。
4. **主题/音量/语音输入是占位**：设置页明确禁用，不产生后端行为。
5. **多显示器只做了逻辑测试**：开发机为单屏 2560×1440，`clamp_*` 用合成多屏（含负坐标）测试；
   真实双屏/混合 DPI 的拖拽与恢复需实机确认。
6. **DPI 只实测了 1.5 倍率**（原生截图/冒烟）；其他倍率（100%/125%/200%）未实机验证。
7. **透明/置顶/点击穿透依赖 Windows 合成器**：offscreen 测试只验证 flags；
   真实 DWM 下的视觉与命中测试需人工确认。
8. **托盘/全局热键是系统级资源**：快捷键可能被其他程序占用；托盘图标在 Explorer 重启后
   可能需要应用重启才恢复（未处理）。
9. **单实例**：没有防止多开；两个实例会争用同一个状态文件与同一热键 id（任务书 13 处理）。
10. **Hotkey 原生分发链路**只在临时探测脚本中实测（真实按键注入 + `WM_HOTKEY` 旁路），
    自动化测试覆盖的是"过滤器 → manager → 信号"这层转发。
11. **窗口状态只在关闭/显隐/拖动/改开关时保存**：拖动窗口后立即崩溃会丢失最后位置。
12. **UI 不是事件回放器**：窗口重建只从 `UiState` 当前快照恢复，不重放历史事件。
13. **托盘不可用时关闭即退出**：这是刻意的安全降级，不是缺陷。
14. **未做无障碍（屏幕阅读器）专项验收**；控件有 objectName/文本标签，键盘焦点默认可用。

### 12.2 Phase 03 llama.cpp Provider

1. **不管理 llama-server 进程**：启动/关闭/显存预算属于任务书 13；Provider 只连接已运行的服务。
2. **上下文窗口不能按请求设置**：只由服务端 `-c/--ctx-size` 决定；配置项只用于核对 `/props.n_ctx`。
3. **不执行工具**：`ToolCallProposal` 只解析；权限、审批、执行属于任务书 07（Tool Broker）。
   Orchestrator 本轮不向模型传 `tools`，因此正常对话不会产生工具调用。
4. **思考内容不显示**：`reasoning_content` 只计数；默认 `thinking: off`（注入 `enable_thinking=false`），
   若模板拒绝该参数会自动去掉并重试一次（`template_kwargs_fallback`）。
5. **结构化输出只做 JSON 解析**：不重新校验 schema，也不声称"严格符合 schema"（服务端 grammar 负责；
   调用方仍要校验字段）。
6. **重试只在首块之前**：已输出文本后的失败会直接上报，不做透明重试（避免重复内容）。
7. **服务端单槽位时并发会排队**：Provider 侧连接池允许多连接，真正并行由 `llama-server -np` 决定。
8. **能力以 `/props` 为准，可由配置覆盖**：看不到 `chat_template_caps` 时工具能力记为 `unknown`，不猜；
   用户可用 `providers.llama_cpp.tool_calling: on/off` 人工标记（只影响诊断与摘要，不改变请求）。
   `structured_output` 只有跑过 `probe_structured_output()` 才会从 `unknown` 变成 supported/unsupported。
9. **真实性能数字只代表一次运行**：`docs/llama-server-probe.md` 的记录不是性能承诺；长上下文 prefill、
   取消后槽位释放时间没有量化测量（任务书 13 的性能基线负责）。
10. **`trust_env=False`**：Provider 不读系统代理环境变量。若未来需要经代理访问远端模型服务，
    由对应 Provider 显式配置（任务书 08），不要改成全局 trust_env。
11. **`finish_reason=length` 对用户不可见**：`stream_text` 只产出文本，收尾块里的结束原因没有通道进
    Orchestrator；达到 `max_tokens` 的回复仍被记为 `completed`。任务书 04 前应把
    `stream_completion` 纳入协议并给 `assistant.text.completed` 增加 `length` 结束原因。
12. **启动预算**：`AsyncAppRuntime.start_core` 默认等待 `max(10s, health_timeout_s + 5s)`；
    服务端"接受连接但不回应"仍会让启动变慢（但不会假死 10s 就报错）。

### 12.3 Phase 04 TTS / 音频播放

1. **MOSS 的 RTF 1.21–1.29（本机 sdpa+eager）**：合成比实时慢，立即播放会周期性欠载；
   `audio.start_prebuffer_ms` 用首延迟换连续播放，所需预缓冲 ≈ (RTF−1)×音频时长。
   **Qwen3-TTS（04D）RTF 0.09–0.10，不适用这条**；它的 300 ms 预缓冲用
   `providers.tts.start_prebuffer_ms` 单独指定。
2. **设备打开可能阻塞**：本机默认输出是蓝牙设备时，`sd.OutputStream.start()` 可能长时间不返回，
   现改为工作线程 + 5 s 超时 → 降级 null 播放（时序照常、无声）。真实设备插拔/切换未实机验证。
3. **服务端能力固定**：batch=1、采样参数与流式粒度硬编码、没有风格/情绪字段；
   Provider 不做"看起来能配"的假参数。Qwen 的 `instructions` 只透传、标 experimental。
4. **MOSS 冷启动 TTFB**：空闲后首请求 5.7 s、取消后重开曾 21.9 s；稳态 1.4–3.3 s。
   **Qwen 冷启动**：加载后首个 WAV 探针 160 ms，其后稳态 TTFA 13–16 ms（`docs/qwen3-tts-provider.md` §6）。
5. **JSON 不携带 PCM**：`AudioChunk` 只在内存里流动；`speech.audio.chunk` 事件默认关闭。
6. **没有多会话并发播报**：一轮一个 utterance/会话；Qwen 服务端 `max_batch=1`，Provider 用串行锁排队，
   等锁超时报 `tts_busy`；MOSS 由服务端 batch=1 排队。
7. **`first_audio_latency_ms` 口径**：首段切出 → 首块入队（不是设备第一帧）；
   设备统计（underruns/played_ms）在 `playback.completed` 里。Qwen 另有 `stream.stats` 的
   `first_byte_ms`（首个原始字节）与 `first_chunk_ms`（首个整帧块）。
8. **MOSS 到达节奏是突发的**（Phase 04A 实测）：块间隔 P99 0.58–0.69 s、max 1.1 s，
   并发 LLM decode 时 RTF 1.1–1.2 → 3.1；`start_prebuffer_ms` 只买到时间，
   `underrun_hold_ms` 只把碎响合并成停顿。WAV 连贯 ≠ 实时连续。
   详见 `docs/phase04a-realtime-playback-diagnosis.md`。
9. **Qwen 服务端不提供音频格式响应头**：格式靠 `expected_*` + 健康检查的 WAV 头探针核对；
   探针本身有一次短合成（与合成共享串行锁），服务忙时可能延后（`format=unverified`，不判不健康）。
10. **Qwen Base 模型不支持 `instructions`**（HTTP 400）：`providers.qwen3_tts.model_type=base` 时
    Provider 在开流前直接拒绝（`tts_instruct_unsupported`）；CustomVoice 的 instruct 稳定性不足，
    只做透传、不承诺"稳定情绪控制"（04C 证据）。
11. **Qwen 中文音色/克隆人工验收未完成**：`docs/tts-samples-qwen/record.md` 试听列留空；
    Base 克隆需要用户中文参考 + 人工校对 `ref_text`（`scripts/qwen_voice_register.py` 已备）。
12. **`speech_smoke.py --mute` 在 Phase 04 的 Mock/MOSS 路径上就会超时**（本次回归对照：
    `--tts-type mock --mute` 同样 FAILED）；这是既有冒烟脚本的静音路径问题，不属于 04D。

### 12.4 Phase 04E GPU 调度 / 运行管理

1. **应用级仲裁，不是硬件优先级**：`GpuScheduler` 只决定"本应用何时向本地 TTS 发合成请求"；
   不做 CUDA stream 优先级、不隔离 SM/显存带宽、不能阻止别的进程抢 GPU。
2. **门控依赖 Orchestrator 通知**：`note_llm_generating` 是"信任上游"的设计——若未来有
   代码绕过 Orchestrator 直接调 Provider（如工具链短任务），调度器看不到，需要显式补通知。
3. **`llm_wait_timeout_s`（180 s）超时会降级语音**：正常情况下 LLM 轮次有
   `request_timeout_s` 兜底；万一门没被打开，宁可本轮不说话，也不把 pump 永久挂住。
4. **显存预算默认关闭**（`whole_card_budget_mb=null`）：默认配置下不启动 `nvidia-smi`
   （省掉首音频路径上一次子进程）。开启后只做"开始合成前"的检查，不阻断已开始的合成。
5. **`nvidia-smi` 取第一块卡**：多卡未验证；`probe_enabled=false` → 快照 `vram.state="unavailable"`。
6. **WDDM 进程级显存不可得**：只有整卡 used 与 PID 列表；文档不伪造进程级数字。
7. **managed 只看得到自己的子进程**：不扫描端口去"认领"别人的 `tts-server.exe`；
   端口占用时只报告、不 kill。重启策略只在"进程消失"时触发，探针失败但进程活着不重启。
8. **日志轮转只在启动期做一次**：`qwen-tts-server.log` 在每次 `start()` 时截断（>1 MiB 先转 `.log.1`，
   只留一份），运行中不滚动；`stop_on_exit=false` detach 后 pump 停止读 stdout，实例后续日志丢失
   （Phase 10 日志清理 / Phase 13 ServiceManager 再统一）。
9. **`sentence_interleave` 不支持**：llama.cpp 没有暂停/续接同一生成请求的 API，
   `gpu.policy` 不接受该值（详见 `docs/qwen3-tts-runtime-and-scheduling.md` §7）。

## 13. 对后续阶段的接口约定

- **任务书 04（TTS/音频）**：已完成（见 §9.4、§12.3、§14）。新增 `speech.*` 事件；
  Orchestrator 在 `ready → idle` 之间插入播报阶段；桌宠 `speaking` 不需要新状态。
- **任务书 04D（Qwen Provider）**：已完成（见 §9.4、`docs/qwen3-tts-provider.md`）。新增
  `providers/qwen3_tts.py`、`Qwen3TTSConfig`、`providers.tts.type=qwen3_tts`；
  **没有新增事件类型、没有改 `SpeechCoordinator`/`AudioPlayer` 契约**。默认仍 `type=none`。
- **任务书 04E（TTS 运行管理与 GPU 调度）**：已完成（见 §9.5、`docs/qwen3-tts-runtime-and-scheduling.md`）。
  新增 `speech/scheduler.py`、`services/*`、`CoreConfig.gpu`、`providers.qwen3_tts.managed` 与
  `mode=disabled`；`SpeechState.WAITING` 是唯一协议增量（无新事件类型）。
  **Phase 13** 复用 `ServiceState/ServiceStatus/ManagedService`，不要在 04E 之外再写一套服务管理。
- **任务书 05（STT）**：`UserMessage.source="voice"`；UI 只需在输入框回填转写文本（已有 `focus_input`），
  不改变 `UiState` 归约；"说话时暂停播报"建议放 Orchestrator（`speech.cancel("listen")`）。
- **任务书 07（Tool Broker）**：`ToolDefinition` / `ToolCallProposal` 已是规范化类型；Broker 负责
  权限分层、schema 校验、审批与审计。Provider 只把模型提出的调用建议交出来，不执行。
- **任务书 08（线上强模型）**：新增 Provider 实现（独立于 `llama_cpp`），沿用
  `ProviderCompletion`/`ProviderChunk`；密钥从环境变量读取，不写配置。
- **任务书 11（会话持久化）**：`LLMRequest.messages` 是完整会话上下文，持久化层把历史注入
  `SessionInfo` 即可；不要绕过 Orchestrator 直接调 Provider。
- **任务书 12（后台任务中心）**：产生 `task.*` 事件后，`UiState` 的 `task_active` 会自动驱动
  `TaskRunning` 桌宠状态；面板可以增加任务卡片，不需要改壳层的事件入口。
- **任务书 13（服务生命周期）**：llama-server / MOSS 的启动/停止/健康监控由 Runtime Manager 负责；
  两个 `health_check()` 都可以直接复用为探针；`docs/moss-tts-probe.md` §5 是显存预算基线。
- 公共协议（事件类型、payload 字段、Provider 协议、任务状态）变更必须先更新本文并在状态文档记录。

## 14. 语音测量口径（Phase 04 / 04D）

| 指标 | 定义 | 落点 |
| --- | --- | --- |
| `ttfb_ms` | `open_stream()`（MOSS：`/start` 发出；Qwen：首个 `POST /v1/audio/speech` 发出）→ 首个音频块到达客户端 | `SpeechMetrics.ttfb_ms` / `tts_probe.py` / `stream.stats.first_byte_ms` |
| `first_audio_ms` | 首段文本切出 → 播放器输出第一帧（含缓冲门限） | `SpeechMetrics.first_audio_ms` |
| `first_byte_ms` / `first_chunk_ms` | **Qwen 专有**：请求发出 → 首个原始字节 / 首个整帧块 yield | `QwenStreamStats`（非协议字段） |
| `synthesize_ms` | MOSS：`/start` → 音频流结束；Qwen：首个请求发出 → 最后一段读完（含串行等待） | `SpeechMetrics.synthesize_ms` / `QwenStreamStats.synth_ms` |
| `audio_ms` | 收到的 PCM 总时长（整帧累计） | `SpeechMetrics.audio_ms` / `QwenStreamStats.audio_ms` |
| `rtf` | `synthesize_ms / audio_ms`（**>1 = 比实时慢**） | `SpeechMetrics.rtf` / probe 报告 |
| `max_buffered_ms` | 播放期间待播缓冲最大值 | `SpeechMetrics.max_buffered_ms` / `PlaybackReport` |
| `underruns` | 播放回调需要数据但没有的次数（只在 `end_utterance` 前统计） | `PlaybackReport.underruns` |

> 口径提醒：`scripts/tts_samples.py --sink player` 的 `rtf` 是**整轮含实时播放**的比率（背压会让
> 合成被播放拖住，≈1.0），不能当作合成 RTF；合成 RTF 看 `tts_probe.py` / `QwenStreamStats`。
> Qwen 04D 实测：TTFA 13–16 ms、RTF 0.09–0.10、10 组样例 0 underrun、常驻 +3.1 GB / 峰值 +3.2 GB
> （详见 `docs/qwen3-tts-provider.md` §6）。
| `played_ms` | 设备实际输出帧数换算的时长（含欠载补的静音） | `PlaybackReport.played_ms` |
| `pause_episodes` | 欠载后进入 hold（等回填再续播）的次数 | `PlaybackReport.pause_episodes` |
| `hold_silence_ms` | 因 hold 有意输出静音的总时长 | `PlaybackReport.hold_silence_ms` |
| `vram_mib` | `nvidia-smi --query-gpu=memory.used` 整卡口径；基线 = 无模型时 | `gpu_mem_probe.py` / `VramProbe` |
| `llm_decode_tps` | llama.cpp 收尾帧 `timings.predicted_per_second`（缺失时用"增量数/首末间隔"近似，两口径都记录） | `scripts/gpu_schedule_probe.py` |
| `waiting_ms` | `GpuScheduler.acquire_tts()` 的等待时长（含等 LLM 与等显存预算） | `SlotResult.waited_ms` / `snapshot().last_wait_ms` |
| `deferred` / `budget_deferred` | 因"本地 LLM 生成中"挂起过 / 因"整卡显存超预算"延迟过的 acquire 次数 | `GpuScheduler.snapshot()` |
| `service_state` | `external/stopped/starting/running/failed` + `detail`（可读诊断） | `QwenTtsRuntime.status()` / `SpeechCoordinator.diagnostics()["service"]` |

04E 口径提醒：`llm_priority` 的验收是"LLM decode tok/s 与 LLM-only 基线差异 ≤ 测量波动
（目标 ≥95%）"；`overlap` 的代价用同一 prompt 的 tok/s 比值表示。两者都必须与
"TTS 合成是否落进 LLM 首末增量之间"一起看（`scripts/gpu_schedule_probe.py` 会直接统计
`speech_overlapped_llm_turns`）。


本机基线（2026-11-18，4090 / sdpa+eager）：TTFB 0.5–3.3 s（冷启动 5.7 s，极端 21.9 s）、
到达 RTF 1.10–1.22（并发 LLM decode 时 3.1）、块间隔 P99 0.58–0.69 s、
TTS 显存 ≈12.0 GB、9B+TTS 并发峰值 21.4–22.1 GB、27B+TTS 不可用（12.5 tok/s）。
诊断与对照实验见 `docs/phase04a-realtime-playback-diagnosis.md`。
