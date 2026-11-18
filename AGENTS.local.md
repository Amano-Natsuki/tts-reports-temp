# 桌面陪伴助手（Desktop Companion）

> 项目档案索引（详细内容按需读 `.dsh/notes/`）。
> 开工先看 `docs/IMPLEMENTATION_STATUS.md`（做到哪、没做什么）与 `docs/ARCHITECTURE.md`（真实模块边界）。
> verified-at: 工作树（Phase 04E 收口：GPU 调度 + Qwen 运行管理；尚未提交）

## 这是什么

本地优先的桌面语音陪伴助手。**Phase 04E：Qwen TTS 运行管理与 LLM 优先 GPU 调度**已完成
（Phase 01 骨架/事件协议、02 桌面交互、03 llama.cpp Provider、04/04D Qwen/MOSS TTS 为基线）：
默认 `gpu.policy=llm_priority`（LLM 生成期间只缓冲语音文本，结束后再合成；`overlap` 仅显式开启，
实测 LLM 并发窗口 −61% / 整轮 −51%）；`providers.qwen3_tts.mode=external|managed|disabled`，
`managed` 由本应用启动/停止 `tts-server.exe`（无 shell、只杀自己启动的进程、失败给可读诊断）。
`sentence_interleave` 判定 **unsupported**（llama.cpp 无暂停/续接 API）。
**语音默认仍关闭**（`providers.tts.type=none`），由用户显式启用；MOSS 保留 **experimental**。

**STT/VAD、MCP/Tool Broker、线上模型、DSH-Codex 适配器仍不存在**；
`TaskRunning` 桌宠状态已能渲染但暂无 `task.*` 事件来源（任务书 12）。

## 关键路径

| 内容 | 位置 |
| --- | --- |
| 入口 / CLI | `src/desktop_companion/main.py`（`--check` 只校验配置，不启 GUI、不联网） |
| 装配 + asyncio 线程 | `src/desktop_companion/app/bootstrap.py`（`startup_timeout_for` / `audio_config_for` / `build_scheduler` / `build_tts_service` / `llm_is_local_gpu`） |
| 一轮对话 / 状态机 | `src/desktop_companion/app/orchestrator.py`（`note_llm_generating` 在 stream_text 前后） |
| **GPU 调度** | `src/desktop_companion/speech/scheduler.py`（`GpuScheduler` / `VramProbe` / `SlotResult`） |
| **Qwen 进程托管** | `src/desktop_companion/services/qwen_tts_runtime.py` + `services/base.py`（Phase 13 复用点） |
| **Provider 协议与类型** | `src/desktop_companion/providers/base.py`、`providers/tts_base.py` |
| **llama.cpp Provider** | `src/desktop_companion/providers/llama_cpp.py`（`resource="local_gpu"`） |
| Provider 工厂 | `src/desktop_companion/providers/factory.py` / `tts_factory.py`（`mode=disabled` → None） |
| 事件协议 / 总线 | `src/desktop_companion/core/events.py`（含 `SpeechState.WAITING`）/ `src/desktop_companion/core/event_bus.py` |
| 配置加载与诊断 | `src/desktop_companion/core/config.py` + `config/default.yaml`（`providers.tts/qwen3_tts.managed`、`gpu.*`） |
| 真实服务诊断脚本 | `scripts/llama_probe.py` → `docs/llama-server-probe.md` |
| 真实模型 + UI 链路冒烟 | `scripts/real_model_smoke.py`（offscreen Qt） |
| 假 llama-server（测试） | `tests/fake_llama_server.py` + `tests/test_llama_cpp_provider.py` |
| 窗口编排 / 关闭策略 | `src/desktop_companion/ui/app_shell.py` |
| 面板+桌宠共享状态 | `src/desktop_companion/ui/ui_state.py` |
| Qt ↔ asyncio 唯一边界 | `src/desktop_companion/ui/qt_event_bridge.py` |
| 契约基准（Phase 03） | `.dsh/notes/phase03-contract.md` |
| 契约基准（Phase 04） | `.dsh/notes/phase04-contract.md` |
| 契约基准（04D） | `.dsh/notes/phase04d-contract.md` |
| **契约基准（04E）** | `.dsh/notes/phase04e-contract.md`（调度/运行管理/集成点/验收口径/§10 偏差） |
| **TTS Provider** | `providers/qwen3_tts.py`（04D 主候选）+ `moss_tts.py`（experimental）+ `tts_base.py` / `mock_tts.py` / `tts_factory.py` |
| **语音编排/播放** | `src/desktop_companion/speech/coordinator.py` / `player.py` / `segmenter.py` |
| Qwen 真实验证记录 | `docs/qwen3-tts-provider.md` + `docs/qwen3-tts-probe.json` + `docs/qwen3-tts-vram.json` |
| **04E 主文档** | `docs/qwen3-tts-runtime-and-scheduling.md`（GPU 策略/服务模式/实测/`sentence_interleave` 结论） |
| **04E 证据** | `docs/gpu-schedule-probe.json`（应用级 6 场景）/ `docs/tts-concurrency-probe.json`+`.md`（直连并发）/ `docs/qwen-runtime-probe.json`（managed 启停） |
| Qwen 10 组中文样例 | `docs/tts-samples-qwen/`（WAV + samples.json + record.md 待人工试听） |
| MOSS 验证记录（历史） | `docs/moss-tts-probe.md` / `docs/moss-tts-probe.json`；`docs/tts-samples/` |
| 实时播放诊断（04A） | `docs/phase04a-realtime-playback-diagnosis.md` + `docs/diagnostics/` + `scripts/playback_diagnosis.py` |

## 常用命令

```powershell
uv sync
uv run desktop-companion
uv run desktop-companion --check
uv run pytest
uv run ruff check
uv run ruff format --check

# 真实 llama-server 诊断（本机 27B/9B 在 11435；不要经代理）
uv run python scripts/llama_probe.py --base-url http://127.0.0.1:11435 --cancel --json probe.json
uv run python scripts/real_model_smoke.py --base-url http://127.0.0.1:11435   # 完整 UI 链路

uv run python scripts/ui_preview.py          # 面板/桌面示意截图（原生 windows，离屏）
uv run python scripts/preview_pet_states.py  # 五种桌宠状态截图
uv run python scripts/gui_smoke.py           # 真托盘 + 真全局热键 + 干净退出（会短暂注册热键）

# 语音（04D Qwen 主候选；需先启动 8095 的 tts-server.exe，见 docs/qwen3-tts-provider.md §1）
uv run python scripts/tts_probe.py --tts qwen --runs 3 --json docs/qwen3-tts-probe.json
uv run python scripts/tts_samples.py --tts qwen --sink player --prebuffer-ms 300
uv run python scripts/speech_smoke.py --tts-type qwen3_tts --expect-speech
uv run python scripts/gpu_mem_probe.py --tts qwen --baseline-only --json docs/qwen3-tts-vram.json  # 先停服务
uv run python scripts/gpu_mem_probe.py --tts qwen --skip-llm --json docs/qwen3-tts-vram.json      # 起服务后
uv run python scripts/qwen_voice_register.py --ref-wav <wav> --name <name> --dry-run

# 语音（04E：GPU 调度 / 运行管理；需 llama-server 11435 + tts-server 8095）
uv run python scripts/gpu_schedule_probe.py --scenario baseline --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario priority --baseline-tps <基线> --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario overlap  --baseline-tps <基线> --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario long     --max-tokens 250 --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario turns20  --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario cancel   --cancel-after-s 6.0 --merge --json docs/gpu-schedule-probe.json
uv run python scripts/qwen_runtime_probe.py --port 8096 --json docs/qwen-runtime-probe.json --merge   # managed 真实启停
uv run python scripts/tts_concurrency_probe.py               # 1.7B×9B 直连并发测量（约 4 分钟）

# 语音（Phase 04 MOSS 历史路径；需先启动 MOSS FastAPI，端口 8083）
uv run python scripts/tts_probe.py --tts moss --json docs/moss-tts-probe.json
uv run python scripts/tts_samples.py --tts moss --play --prebuffer-ms 2500
uv run python scripts/speech_smoke.py --tts-type moss_tts --expect-speech
uv run python scripts/gpu_mem_probe.py --tts moss --json docs/gpu-mem-probe.json
```

## 不可破坏的边界

- UI 只能经 `QtEventBridge` 与后台交互；事件只进 `UiState` 一次，面板与桌宠都从它读，**不得各订阅一份**。
- 业务状态只能来自应用事件：窗口/桌宠不推断；AppShell 的错误状态保留只是展示时间窗（默认 2.5s）。
- 后台不得碰 QWidget，UI 不得等待 Future；asyncio 独立线程，不引入 qasync。
- 事件 payload 必须可 JSON 序列化；Phase 03 **没有新增事件类型**；新增协议先改 `docs/ARCHITECTURE.md`。
- **Provider 边界**：OpenAI wire format 只存在于 `llama_cpp.py` 内部；`Orchestrator` 只用
  `stream_text`；`generate_text/generate_structured` 是诊断/后续短任务入口。
- **Provider 不做的事**：不加载 GGUF、不启动/关闭 llama-server、不执行工具、不把模型名写进业务逻辑、
  不猜版本号/能力（探测不到就写「未知」）。
- **取消语义**：没有 `cancel(request_id)`；取消消费任务，`CancelledError` 必须原样上抛；
  取消一个请求不得影响连接池或其它并发请求。
- **重试边界**：只在「尚未产出任何文本块」时重试；已输出文本后重试会重复内容，禁止。
- 配置 `extra="forbid"`；运行期 UI 状态写 `ui-state.json`，**不要回写 YAML**；
  `providers.llm.type=llama_cpp` 时 `base_url` 必填（代码里没有默认端口）。
- **httpx 客户端必须 `trust_env=False`**：本地回环不走代理；另外 httpx 0.28 解析 `NO_PROXY` 里的
  `[::1]` 会抛 `InvalidURL`（实测）。
- 密钥只从环境变量读（`api_key_env` 存变量名）；`ProviderError.details` 只放截断的、JSON 安全的诊断。
- 配置文件是默认值，状态文件是"上次运行"；托盘不可用时 `close_to_tray` 强制 False（安全降级）。
- 鼠标穿透默认关闭；开启后必须仍能从托盘/设置关闭。
- 全局热键失败不能阻止启动；托盘 + 面板内 QShortcut 是 fallback。
- 桌宠透明/置顶/穿透 flags 的改变会触发 hide/show：必须保存并恢复位置/可见性，期间抑制假显隐信号。
- 关掉 `main._run_gui` 的 `finally` 顺序：`shell.shutdown()` → `bridge.shutdown()` → `runtime.stop()`。
- **语音层边界**：`speech/*` 只依赖 `core`/`providers.tts_base`，不得 import `app`/`ui`/Qt；
  UI 不 import 语音层（`tests/test_architecture.py` 断言），静音/语音状态只能经事件回传。
- **音频 PCM 永不进事件总线**（`speech.audio.chunk` 默认关闭，只带元数据）。
- **一轮播报 = 一个 utterance（=request_id）= 一个 MOSS 会话**；取消后迟到块必须按
  `utterance_id`+epoch 丢弃，任何"复用同一会话跨轮"的改动都要先改契约。
- **设备打开绝不能阻塞事件循环**：`AudioPlayer._ensure_sink` 在工作线程 + 5 s 超时里做，
  超时降级 null 播放；不要改回同步 `sink.start()`（本机蓝牙设备会让整个应用冻住）。
- **TTS 失败只降级**：`speech.error(degraded=True)` + 状态回 idle；不允许抛给 Orchestrator，
  也不允许把 `assistant.text.completed` 改成 failed。
- **RTF 口径**：`synthesize_ms / audio_ms`（>1 慢于实时）；预缓冲需求 ≈ (RTF−1)×音频时长。
- **实时播放的已知边界（04A）**：到达 RTF 1.10–1.22、块间隔 P99 0.6–0.7 s；预缓冲只买时间，
  `audio.underrun_hold_ms` 只合并碎响；并发 LLM decode 时 RTF≈3.1，长句连续播放**无法保证**。
  不要把 WAV 连贯当成实时验收。
- **Qwen Provider 边界（04D）**：一段文本 = 一次 `POST /v1/audio/speech`（不支持文本增量）；
  `open_stream(initial_text)` 预发首段并等响应头（400/坏 Content-Type 从这里抛）；`push_text` 只入队；
  **`audio()` 必须收到 `push_text("", final=True)` 才会结束**（脚本/探针漏了会挂起——04D 踩过）；
  串行锁（服务端 `max_batch=1`，等锁超时 `tts_busy`）；取消=关闭响应（服务端 `on_chunk` 中止）。
- **Qwen 格式验证**：服务端**没有**采样率/声道响应头 → `expected_*` + 健康检查 WAV 探针
  （`response_format=wav` 解析 RIFF）核对 24 kHz/mono/16 bit；不一致判不健康。
- **Qwen 的 Base 不支持 instructions**（HTTP 400）：`providers.qwen3_tts.model_type=base` 开流前直接拒绝；
  CustomVoice 的 instruct 只透传、标 experimental（04C：稳定性不足）。
- **预缓冲按后端区分**：Qwen 用 `providers.tts.start_prebuffer_ms: 300`，MOSS 用 2500；
  `audio.start_prebuffer_ms` 全局默认不动（`bootstrap.audio_config_for` 覆盖）。
- **Qwen 默认不启用**：`providers.tts.type` 仍是 `none`；是否切换默认要用户确认，不得静默改。
- **GPU 调度边界（04E）**：`GpuScheduler` 是**应用级任务仲裁**——只决定"何时向本地 TTS 发合成请求"，
  不做 CUDA 优先级/SM/显存带宽隔离；文案与文档不得声称相反。
  `llm_priority` 下"取到槽才 `open_stream`"，`release_tts()` 必须在 `_pump` 的 `finally` 恰好一次；
  `note_llm_generating` 在 Orchestrator 的 `stream_text` 前后必须成对（取消/异常也要发 False）。
- **`SpeechState.WAITING` 只在真的等待时出现**（立即放行不能闪状态）；没有 scheduler 时状态序列与 04D 一致。
- **调度拒绝只降级语音**：`tts_scheduler_deferred` / `tts_vram_budget` 走 `speech.error(degraded=True)`，
  `assistant.text.completed` 仍是 `completed`。
- **Qwen 运行管理边界（04E）**：`managed` 只用 `asyncio.create_subprocess_exec`（**禁止 `shell=True`**）；
  只终止自己启动的 pid；`external` 实例永不碰；`stop_on_exit=false` 只 detach；
  文件/DLL/端口/启动超时是**运行期可读诊断**（不阻塞 GUI、不抛给调用方）；重启有上限+退避。
- **`services/*` 只服务 Qwen TTS**：通用 ServiceManager 留给 Phase 13；不要在本阶段扩成全局管理器。
- **`speech.*` 事件的 `request_id` 是语音层自己的 utterance id**，与对话轮 `request_id` **不是同一个值**；
  跨层关联要按时间窗口/utterance 做，不能按 `request_id` join。
- **`speech_smoke.py --mute` 在 Mock/MOSS 路径本就会超时**（既有问题，与 04D/04E 无关）；
  真机验收用非静音模式。

## 坑 / 注意

- **窗口可见性用"意图"落盘，不用 `isVisible()`**：Qt 在 `app.exec()` 退出阶段会先 hide 顶层窗口
  （hideEvent 早于 aboutToQuit）；按 isVisible 保存会把上次可见写成 false → 下次启动只剩托盘。
  `AppShell._panel_visible_intent/_pet_visible_intent` 只在用户动作时更新；
  `tests/test_app_shell.py::test_visibility_intent_survives_teardown_hide` 钉住这条。
- 修改窗口 flags（置顶/穿透）会触发 hide/show：面板用 `QSignalBlocker`、桌宠用
  `_suppress_visibility_signal` 抑制假 `visibility_changed`；否则托盘文案会闪、状态会多写。
- 托盘只响应单击：Windows 双击会先 Trigger 再 DoubleClick，两者都接会双 toggle。
- 聊天气泡正文强制 `Qt.TextFormat.PlainText`，否则 `<b>`/`<img src=file:///...>` 会被当富文本。
- **气泡换行**：QBoxLayout 里直接给 QFrame 加 alignment 会按"最小高度"布局，QLabel wordWrap
  第二行起被裁掉；必须整行容器 + QHBoxLayout + stretch，并用 QFontMetrics 显式定宽。
- `QPixmap` 必须先有 `QGuiApplication`：测试里构造 `TrayController`/`theme.make_app_icon` 要带 `qapp`。
- PySide6 6.12 下 `class HotkeyManager(QObject, QAbstractNativeEventFilter)` 多继承**收不到原生回调**；
  用独立 `_NativeEventFilter` 组合转发（勿改回）。
- 托盘/快捷键是系统资源：**自动化测试默认禁用**（`tests/helpers.make_config`）。
- **YAML 1.1 把 `on`/`off` 当布尔**：`providers.llama_cpp.thinking` 已做归一（`ThinkingMode`），
  写 `thinking: off` 和 `"off"` 等价；`--check` 输出永远是字符串。
- **思考模型**：开启 thinking 时 `reasoning_content` 会先吃掉 `max_tokens`（实测 `max_tokens=64` 时正文为空、
  `finish_reason=length`）；默认 `thinking: "off"` 注入 `enable_thinking=false`。
- **SSE 收尾**：`finish_reason` 与 `[DONE]` 任一出现即视为正常结束；只有 `[DONE]` 时
  `finish_reason=unknown`（不猜 stop）。两者都没有 → `provider_stream_interrupted`。
- **`/health` 200 只代表进程可用**：加载中返回 503 + `{"status":"loading model"}` → `provider_unavailable`。
- **llama-server 的上下文只能由启动参数 `-c` 决定**：`context_window_tokens` 只做核对，不改变服务端。
- **`api_key_env` 是变量名不是密钥**：`mask_secrets` 不会把它打码成 `***`（有测试钉住）。
- `uv` 把进度/日志写 stderr；PowerShell 里读中文日志要 `-Encoding UTF8`（或 node），否则乱码 ≠ 编码坏了。
- 跑 `llama_probe.py` 时设 `$env:PYTHONIOENCODING='utf-8'`，否则管道输出按 GBK 编码会乱码。
- 干净关闭的日志标志：`窗口层已启动` + `已退出（清理耗时 …ms）`；否则会打印残留清单。
- **`app.quit()` 在 QTimer 槽里偶发不退出**（PySide6，实测约 1/5）：离屏脚本改用
  `processEvents()` 泵循环；主窗口走正常关闭路径不受影响。
- **MOSS 的 `NO_PROXY` 坑**：客户端必须 `trust_env=False`（`[::1]` 会让 httpx 抛 `InvalidURL`）；
  服务端由 `moss_launch.py` 自己重写 `NO_PROXY`。真实服务 TTFB/RTF 见 `docs/moss-tts-probe.md`。
- **PowerShell `Set-Content -Encoding UTF8` 会写 BOM**：本会话改过的 7 个文件被补过 BOM，
  已统一剥掉；改 Python/配置文本优先用 `write`/`edit` 工具，别用 PowerShell 回写。
- **`Start-Process -RedirectStandardOutput` 在本机必失败**（环境里同时有 `NO_PROXY`/`no_proxy`，
  PowerShell 报"已添加项"）→ 要捕获服务端输出用 `cmd /c "... > log 2>&1"`。
- **`tts-server.exe` 有一次进程内崩溃**：20 轮连测第 10 轮 `0xc0000409`（`ucrtbase.dll`/BEX64），
  客户端按设计降级为纯文字；重跑 20 轮未复现。别把它误判成调度/配置 bug；`managed.restart_on_crash` 可兜底。
- **测播放时长别用 `speech.playback.completed.duration_ms`**：它只覆盖 `wait_done` 里
  `_wait_playback` 的时间（播放可能早已开始，甚至合成被播放器背压限速）；
  真实时长看 `audio_ms`/`played_ms` 或播放开始/结束事件的墙钟差。
- **null sink 是实时的**（20ms/480 帧），但"合成结束时的等待时长"很短；`underruns=0`
  在 Qwen RTF≈0.09 下是预期，不是测量失效。
- **端口占用检测要 bind 兜底**：只 `connect_ex` 可能因 accept 队列塞住误报空闲（Windows 实测），
  `_port_in_use` 现在 connect + bind 双信号；测试里做"占用者"**不要设 `SO_REUSEADDR`**
  （Windows 上它允许两个进程同绑一个端口，场景就失效了）。

## 下一步

**Phase 04E 已收口**（调度器/运行管理/集成/测试/真实 4090 验收/文档；`789` 项测试全绿，
待提交）。剩余人工项（承袭 04C/04D）：`docs/tts-samples-qwen/record.md` 试听、
中文参考克隆验收、情绪指令抽检。

下一阶段候选（按总纲路线图）：**05 STT/VAD**（`UserMessage.source="voice"` 已预留；
"说话时暂停播报"放 Orchestrator）；或 **06 Tool Broker**。
`llm_priority` 之后的演进方向：`overlap` 仍标实验；Phase 13 用 `services/base.py` 的
`ServiceState/ServiceStatus/ManagedService` 统一管理 llama-server/STT。
**历史**：04D 已收口（Qwen Provider）；04A 已收口（MOSS 播放器不再继续优化，根因=模型侧 RTF>1）；
MOSS 保留 `experimental`、不设默认服务。
