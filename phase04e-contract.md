# Phase 04E 契约（Qwen TTS 运行管理 + LLM 优先 GPU 调度）——实现与测试共同基准

> 冻结于 2026-11-18。`src/` 实现以此为准；测试按此写，**不得**要求实现迁就测试。
> 上游依据：任务书 `04E_任务书_TTS运行管理与GPU调度.md`、`00_项目总纲.md` §4.4/§10、
> `.dsh/notes/phase04d-contract.md`（04D 契约，除了"managed 报 ConfigError"一条被本阶段替换外仍然有效）。

## 1. 目标与边界

- 做：`llm_priority` 默认 GPU 仲裁（`overlap` 仅显式开启）、Qwen TTS `external/managed/disabled`
  三态运行管理（仅 Qwen）、显存/健康/队列观测、按后端区分的播放预缓冲与队列策略、
  `sentence_interleave` 可行性审查（**只出结论，不实现**）。
- 不做：管理 llama-server / STT / VLM / 外部 Agent 的进程；GPU 硬隔离、CUDA 优先级、
  操作系统级抢占；多 TTS 并发（服务端 `max_batch=1`）；`sentence_interleave` 生产实现；
  改 `SpeechProvider`/`SpeechStream` 音频契约；改 MOSS Provider 行为；训练/量化。
- 硬约束：调度是**应用级任务仲裁**，任何文案不得声称能隔离 SM/显存带宽；
  `providers.tts.type` 默认仍是 `none`（04E 不静默打开语音）；
  `overlap` 默认关，开启要在设置页显示实测代价；服务管理只服务 Qwen，接口形状留给 Phase 13。

## 2. 配置契约（`core/config.py`，全部 `extra="forbid"`）

顶层新增 `gpu`：

```python
class GpuConfig(BaseModel):
    policy: Literal["llm_priority", "overlap"] = "llm_priority"
    llm_local_gpu: Literal["auto", "yes", "no"] = "auto"   # auto：按 LLM Provider 的 resource 判定
    llm_wait_timeout_s: float = 180.0        # 等本地 LLM 结束的上限；超时 → 本轮语音降级
    probe_enabled: bool = True               # nvidia-smi 显存探测
    probe_interval_s: float = 2.0            # 探测结果缓存 TTL
    probe_timeout_s: float = 2.0
    whole_card_budget_mb: int | None = None  # 整卡已用显存预算；None = 不限制（默认）
    budget_defer_timeout_s: float = 20.0     # 超预算时最多延迟多久，超时 → 本轮语音降级
    tts_resident_budget_mb: int = 4096       # 验收目标（监控/报告用，不参与拒绝）
    tts_peak_budget_mb: int = 5120
```

`providers.qwen3_tts` 增量：

```python
mode: Literal["external", "managed", "disabled"] = "external"   # 04D 的 managed→ConfigError 被替换
managed: QwenManagedConfig = QwenManagedConfig()

class QwenManagedConfig(BaseModel):
    executable: str | None = None     # tts-server.exe 绝对路径（managed 必填，否则 ConfigError）
    model: str | None = None          # talker GGUF
    codec: str | None = None          # tokenizer/codec GGUF
    cwd: str | None = None            # None → exe 所在目录
    env: dict[str, str] = {}          # 追加/覆盖到子进程环境（不删除父进程变量）
    args: list[str] = []              # 额外 CLI 参数（如 --device 0）
    host: str | None = None           # None → 从 providers.tts.base_url 推导（必须回环）
    port: int | None = None           # 同上
    alias: str = "qwen3-tts"
    lang: str = "Chinese"
    max_batch: int = 1
    startup_timeout_s: float = 120.0  # 等 /health 通过的上限
    shutdown_timeout_s: float = 10.0
    stop_on_exit: bool = True         # 应用退出时终止"本应用启动的"进程；external 永不终止
    restart_on_crash: bool = False
    max_restarts: int = 2
    restart_backoff_s: float = 5.0
    health_monitor_interval_s: float = 10.0   # managed 健康巡检；0 = 关闭
    required_files: list[str] = []    # 额外必检文件；默认自动检查同目录 ggml*.dll
    log_dir: str | None = None        # 子进程 stdout/stderr 落盘目录；None → 用户状态目录
```

- `mode=disabled` ⇔ 语音不启用（`create_tts_provider` 返回 `None`，与 `type=none` 同效，健康行 detail 写原因）。
- `mode=managed` 时 `base_url` 的主机必须是回环（127.0.0.1 / localhost / ::1），否则 ConfigError：
  不能在别的机器上"托管"进程。`mode=managed` 且 `managed.executable` 为空 → ConfigError（带示例）。
- `mode=managed` 的文件存在性、DLL、端口占用、启动超时**不是** ConfigError，而是运行期可读诊断
  （不得阻塞 GUI 启动）。`mode=external` 忽略 `managed.*`（只记一条 debug）。
- `providers.tts.start_prebuffer_ms` 语义不变（按 Provider 覆盖）；本阶段只调整 `config/default.yaml`
  的注释与推荐值，不静默改用户值。

## 3. `GpuScheduler`（`src/desktop_companion/speech/scheduler.py`）

只做仲裁与观测，不做音频/HTTP。线程模型：**只在 asyncio 事件循环线程内使用**；
`note_llm_*` 是同步方法（由 Orchestrator 调用），`acquire/release` 是 async（由 Coordinator 调用）。

```python
class SlotDecision(StrEnum):
    GRANTED = "granted"        # 可以开始 TTS 合成
    OVERLAP = "overlap"        # overlap 策略下无需等待（同样可以开始）
    BUDGET_DENIED = "budget_denied"   # 显存超预算且等超时 → 本轮语音降级
    TIMEOUT = "timeout"        # 等 LLM/预算超时 → 本轮语音降级

@dataclass(frozen=True)
class SlotResult:
    decision: SlotDecision
    waited_ms: float = 0.0
    reason: str = ""           # 可读原因（进日志/降级事件 detail）
    llm_busy: bool = False
    vram_used_mb: int | None = None

class GpuScheduler:
    def __init__(self, *, config: GpuConfig, clock: Callable[[], float] = time.perf_counter,
                 vram_probe: VramProbe | None = None, llm_local_gpu: bool = True) -> None: ...

    # Orchestrator 调用（同步）
    def note_llm_generating(self, active: bool) -> None: ...   # 唤醒等待者；幂等
    # Coordinator 调用（async）
    async def acquire_tts(self, *, utterance_id: str | None = None) -> SlotResult: ...
    def release_tts(self) -> None: ...                          # 幂等
    # 观测
    @property
    def policy(self) -> str: ...
    @property
    def llm_busy(self) -> bool: ...
    def snapshot(self) -> dict[str, Any]: ...   # policy/llm_busy/tts_active/deferred/wait_ms/decisions/vram
```

语义（必须逐条可测）：

1. `policy=llm_priority` 且 `llm_local_gpu=True`：`LLM generating` 期间 `acquire_tts` **不返回**，
   直到 `note_llm_generating(False)` 或 `llm_wait_timeout_s` 超时（→ `TIMEOUT`）。
2. `llm_local_gpu=False`（mock 或线上模型）时不做 LLM 门控（只可能因显存预算等待）。
3. `policy=overlap` 时立即返回 `OVERLAP`，**不等待**；仍做显存预算检查（超预算同样延迟）。
4. 预算：`whole_card_budget_mb` 为 None → 不做预算检查；否则用 `VramProbe` 读整卡 used，
   `used > budget` 时等待并重试（间隔 `probe_interval_s`），超过 `budget_defer_timeout_s`
   → `BUDGET_DENIED`（附 `vram_used_mb`）。探测不可用（无 nvidia-smi/超时）→ **放行**并记
   `vram=unavailable`（不因为探针坏了就不说话）。
5. `acquire_tts` 被取消 → `CancelledError` 原样上抛；`release_tts` 幂等。
6. 计数：`deferred`（因 LLM 等待过）、`budget_deferred`、`timeout`、`granted`、`overlap`、
   `last_wait_ms`、`max_wait_ms`，进 `snapshot()`（JSON 安全，供日志/探针脚本）。

`VramProbe`（同文件）：`async def read_used_mb(self) -> int | None`；默认实现 = `nvidia-smi
--query-gpu=memory.used --format=csv,noheader,nounits`（`asyncio.create_subprocess_exec`，
**无 shell**），TTL 缓存；`runner` 可注入（测试用）。失败/解析异常 → `None`，并记一次 warning
（日志不刷屏）。

## 4. 运行管理（`src/desktop_companion/services/`）

`services/__init__.py`、`services/base.py`、`services/qwen_tts_runtime.py`。

### 4.1 可复用契约（Phase 13 将泛化，本阶段只做 Qwen 实现）

```python
class ServiceState(StrEnum):
    DISABLED = "disabled"
    EXTERNAL = "external"      # 用户自己启动的服务（只观测，不管理）
    STOPPED = "stopped"
    STARTING = "starting"
    RUNNING = "running"
    FAILED = "failed"

@dataclass(frozen=True)
class ServiceStatus:
    name: str
    mode: str                   # external/managed/disabled
    state: ServiceState
    pid: int | None = None
    started_at: str | None = None      # ISO-8601
    exit_code: int | None = None
    restarts: int = 0
    detail: str = ""                   # 可读诊断（一行；给 health detail/日志）
    log_path: str | None = None
    artifacts: Mapping[str, Any] = field(default_factory=dict)  # commit/model/codec/dll 布局等

class ManagedService(Protocol):
    async def start(self) -> ServiceStatus: ...     # 幂等；不抛，失败用状态+detail 表达
    async def stop(self) -> ServiceStatus: ...      # 幂等；绝不终止非本应用启动的进程
    def status(self) -> ServiceStatus: ...
    async def ensure_ready(self, *, timeout: float | None = None) -> bool: ...
```

### 4.2 `QwenTtsRuntime`

```python
QwenTtsRuntime(
    *, settings: Qwen3TTSConfig, base_url: str,
    probe: Callable[[], Awaitable[bool]],       # 注入 Provider 健康检查（不重复实现 HTTP）
    log_dir: Path | None = None, log: logging.Logger | None = None,
)
```

- `start()`：`mode=external/disabled` → 只记录状态，不启动任何进程（`EXTERNAL`/`DISABLED`）。
  `managed`：
  1. 检查 `executable` 存在、`model`/`codec` 存在、同目录 `ggml.dll`/`ggml-base.dll`/
     `ggml-cpu.dll`/`ggml-cuda.dll` 存在（缺失 → `FAILED` + 精确到文件名的 detail，不抛）。
  2. 检查 host:port 是否已被占用（`socket.connect_ex` 或等价；**不**去 kill 任何东西）→ detail
     指“端口被占用（可能已有一个 tts-server 在跑；请改 providers.tts.base_url 或结束它）”。
  3. `asyncio.create_subprocess_exec(exe, "--model", ..., "--codec", ..., "--alias", ...,
     "--host", ..., "--port", ..., "--lang", ..., "--max-batch", ..., *extra_args, cwd=cwd,
     env={**os.environ, **managed.env}, stdout=PIPE, stderr=STDOUT)`；**禁止 shell**。
     同目录 DLL 依赖靠 `cwd`/exe 目录解析（Windows 默认 DLL 搜索含 exe 目录）。
  4. stdout/stderr 用后台 pump 任务逐行写入 `log_path`（`<log_dir>/qwen-tts-server.log`，
     滚动/截断策略写清）并保留最近 N 行（默认 200）在内存供诊断。
  5. 轮询 `probe()` 直到 True 或 `startup_timeout_s`；进程提前退出 → 用退出码 +
     最后日志行生成可读 detail（`0xC0000135` = `STATUS_DLL_NOT_FOUND`，映射成“缺少 DLL”）。
  6. 成功后 `RUNNING`（记录 pid/started_at/artifacts）。
- `stop()`：只终止 `pid` 为自己启动的进程；`stop_on_exit=false` 时**不终止**（只 detach，
  detail 说明"按配置保留进程"）；先 `terminate()`，`shutdown_timeout_s` 后 `kill()`；幂等。
- 崩溃重启：仅 managed 且 `restart_on_crash=true`；进程在 `RUNNING` 后消失 → 按
  `restart_backoff_s` 重启，`restarts` 计数，超过 `max_restarts` → `FAILED` 且不再重启；
  单实例锁，绝不并发拉起两个。
- `status()`：`artifacts` 至少含 `commit`、`model`、`codec`、`executable`、`dlls`（found/missing）、
  `host`、`port`、`log_path`、`tail`（最近日志行）。
- 全部失败路径**不抛**给调用方（`start()` 返回状态）；任何等待都有上限。

## 5. 集成点（Lead 实现；T1/T2 不要改这些文件）

### 5.1 `speech/coordinator.py`

- 新增 `scheduler: _SchedulerLike | None = None` 构造参数与 `service: _ServiceLike | None = None`。
- `_pump`：取到首段之后、`provider.open_stream(...)` **之前** `await scheduler.acquire_tts(...)`；
  - `GRANTED/OVERLAP` → 继续；`TIMEOUT/BUDGET_DENIED` → 走既有降级路径
    （`speech.error(degraded=True)`，`user_message` 用 `DEGRADED_USER_MESSAGE` 的
    "语音暂时不可用，本轮只显示文字。"，错误码 `tts_scheduler_deferred` / `tts_vram_budget`，
    detail 带 `reason`/`waited_ms`/`vram_used_mb`）；
  - `finally` 里 `scheduler.release_tts()`（每个取得过槽的轮次恰好释放一次）。
- 等待期间状态 = 新增 `SpeechState.WAITING`（"语音：等待本地 LLM…"）：进入等待时 `_set_state(WAITING,
  detail=原因)`，取得槽后回到 `SYNTHESIZING`。**只有配了 scheduler 且真的等待时才出现这个状态**，
  没有 scheduler 的既有测试序列不变。
- `service`：`start()` 时 `await service.start()`（已经是后台任务则直接返回，不阻塞 UI）；
  `health_check()` 时若 `service.status().state` ∈ {STARTING, FAILED} 用它的 detail；
  `aclose()` 时 `await service.stop()`（幂等；总线已关时不影响）。
- `diagnostics()`（新增，同步、JSON 安全）：state/utterance/queue 段数/等待段数/`chunks`/`audio_ms`/
  `late_chunks_dropped`/`scheduler.snapshot()`/`service.status()`，供探针脚本与日志。

### 5.2 `app/orchestrator.py`

- `SpeechSink` 增加可选 `note_llm_generating(active: bool) -> None`（用 `getattr` 调用，测试假实现
  缺它不报错）；在 `_run_turn` 里：进入 `stream_text` 前 `True`，`async for` 结束/异常后 `False`
  （**不**等播报结束——播报阶段 LLM 已不占 GPU decode）。
- 其它不变：新消息/停止仍然先 `speech.cancel(...)` 再取消本轮。

### 5.3 `app/bootstrap.py`

- `_build_speech`：建 `GpuScheduler`（policy/local_gpu 来自 `config.gpu` + `provider.resource`）、
  必要时建 `QwenTtsRuntime`（`mode=managed`）并注入，传给 `SpeechCoordinator`。
- LLM Provider 暴露资源元数据：`LlamaCppProvider.resource = "local_gpu"`；
  `MockLLMProvider.resource = "none"`（`getattr(provider, "resource", "unknown")`，unknown 按
  `local_gpu` 处理：宁可保护 LLM）。`gpu.llm_local_gpu=yes/no` 强制覆盖。
- 关闭顺序不变；`speech.aclose()` 内部负责 `service.stop()`。

### 5.4 UI

- `SpeechState.WAITING` → 面板头部「语音：等待 LLM…」，tooltip 用 `detail`。
- 设置页「语音播报」组新增只读行：GPU 策略（`llm_priority` / `overlap`）+ 当 `overlap` 时显示
  实测代价提示（来自 `config/default.yaml` 注释与本阶段报告的数字）；管理其余 UI 不变。

## 6. 事件协议

- 新增 `SpeechState.WAITING`（`core/events.py` 的枚举值）；**不新增事件类型**。
- `speech.error` 新增两个稳定错误码：`tts_scheduler_deferred`、`tts_vram_budget`（都是降级，不改变文字结局）。
- `system.health.changed(component="tts")` 继续用于健康行；managed 的启动/失败在 `detail` 里带
  `service=<state>` 与可读原因。

## 7. 测试要求

- `tests/test_gpu_scheduler.py`（T1）：llm_priority 等待/唤醒/超时/取消；overlap 立即放行；
  remote/mock LLM 不门控；预算超限延迟与 `BUDGET_DENIED`；探针不可用放行；计数与 snapshot；
  release 幂等、重复 acquire 不串台。假时钟/假探针注入，不 sleep 真实时间（≥20ms 才用真实 sleep）。
- `tests/test_qwen_tts_runtime.py` + `tests/fake_tts_server_process.py`（T2）：假可执行（真子进程）
  覆盖：正常启动→健康→停止；端口占用；缺 exe/模型/codec/DLL；启动超时；进程早退（退出码映射）；
  `stop_on_exit=false` 不杀；external/disabled 不启动；崩溃重启上限；`status().artifacts` 形状；
  `logs` 落盘与 tail；`stop()` 幂等。
- 集成（Lead）：coordinator 在 LLM busy 时**不调用** `provider.open_stream`（用计数器假 Provider），
  LLM 结束 → 恰好调用一次并顺序合成；新请求取消后旧轮 PCM 不串轮；budget 降级不改变
  `assistant.text.completed`；Orchestrator 的 `note_llm_generating` 时序（生成中 True、完成后 False、
  取消/失败也 False）。
- `tests/test_config.py`：新字段默认值/校验（managed 缺 executable、base_url 非回环、policy 枚举、
  budget 关系）、`default.yaml` 与模型默认值一致。
- 全量 `uv run pytest` / `uv run ruff check` / `uv run ruff format --check` 必须全绿。

## 8. 真实 4090 验证（Lead + T3；Mock 不能替代）

1. 固定 9B（`DeepSeek-V4-Pro-Qwen3.5-9B-UNCENSORED-heretic.Q4_K_M`，131072 ctx，11435）与
   Qwen 1.7B CV Q8_0（8095/8096，`--max-batch 1`）。
2. LLM-only decode tok/s（llama-server `timings.predicted_per_second`）× N 次 → 基线；
   `llm_priority` 场景下同一请求的 tok/s ≥ 基线 95%；`overlap` 场景下记录实际掉速（用于 UI 提示）。
3. LLM 完成后 TTS 暖机 TTFA ≤800 ms、RTF ≤0.7；常驻 ≤4 GB、峰值 ≤5 GB（相对服务未启动基线）。
4. 语音生成期与 LLM decode 不重叠；合成结束后即使仍在播放，LLM 可继续用 GPU。
5. 取消/新请求/切会话：旧轮生成与待播取消、无串轮（记录 `late_chunks_dropped`）。
6. `managed` 真实启动/停止 `tts-server.exe`（用 8096 端口避免与 external 验证串台），
   记录 PID/启动时间/退出码/commit/模型路径/DLL 布局；确认不误杀用户手动实例。
7. 15 s 以上音频、20 轮连续会话：无欠载、队列不无限增长、无资源泄漏、服务不僵死。

## 9. 文档交付

- `docs/qwen3-tts-runtime-and-scheduling.md`：GPU 策略、服务模式与启动命令、进程/日志/退出语义、
  资源阈值与观测口径、队列与打断策略、失败恢复、`sentence_interleave` 可行性结论、
  实测数据与复现命令、Phase 13 接口说明。
- 更新 `docs/ARCHITECTURE.md`（新模块 + `SpeechState.WAITING` + 关闭顺序 + §14 指标）、
  `README.md`、`docs/IMPLEMENTATION_STATUS.md`（§12 04E）、`AGENTS.local.md`、本文件 §10。

## 10. 实现—契约偏差（收口时填，2026-11-18）

1. **单槽互斥由等待实现**（契约 §3 未写死第二个 acquire 的语义）：同一时刻只允许一路 TTS 合成，
   第二个 `acquire_tts()` 挂起直到 `release_tts()`，可取消、没有独立超时（barge-in/停止会取消任务，
   避免长句合成被误判卡死）。对应 `test_second_acquire_waits_for_release`。
2. **`whole_card_budget_mb=None` 时不读显存**（即使注入了 `VramProbe`）：避免默认配置在首音频路径上
   多一次 `nvidia-smi`（Windows 上几十~百毫秒）。`snapshot().vram.state` 为 `disabled`；
   `probe_enabled=false` 时为 `unavailable`。
3. **`VramProbe` 快照形状** `{budget_mb, used_mb, state}`，`state ∈ {disabled, unknown, ok, over_budget, unavailable}`；
   超预算等待过但最终放行计入 `budget_deferred`，没有单独的 `denied` 计数。
4. **`stop_on_exit=false` 的 detach 状态取 `EXTERNAL`**（pid 保留、detail 说明"保留"）：
   进程仍在跑但本应用不再管理，用 `STOPPED` 会误导协调器/健康行。
5. **`FAILED` 后手动 `start()` 会重试一次完整启动**（有界）；自动崩溃重启仍严格受 `max_restarts` 封顶。
6. **Windows `terminate()` = `TerminateProcess`**（无优雅退出），`exit_code` 常见为 1；
   `shutdown_timeout_s` 后再 `kill()`。真实 `tts-server.exe` 是直接子进程，无树杀问题。
7. **`managed` 的额外防御**（不违背契约）：`managed.host` 非回环 → `FAILED`（配置只校验了 `base_url`）；
   `managed.cwd` 相对路径按 exe 目录解析；`required_files` 相对 cwd；spawn 加 `stdin=DEVNULL` +
   `CREATE_NO_WINDOW`（GUI 不弹黑窗）；env 合并 `{**os.environ, **managed.env}`。
8. **日志策略**：首次 `start()` 截断 `<log_dir>/qwen-tts-server.log`（>1 MiB 先轮转 `.log.1`，只留一份）；
   崩溃重启向同一文件追加 `# 重启` 头；内存 tail 200 行、单行截 2000 字符。
   `log_dir` 优先级：构造参数 > `managed.log_dir` > `%LOCALAPPDATA%\desktop-companion\logs`。
9. **端口占用检测加 `bind` 兜底**（收口时补）：只 `connect_ex` 在 Windows 上可能因 accept 队列塞住
   误报空闲（实测走了 120 s 启动超时才失败）；现在 connect + bind 双信号，
   `test_port_in_use_detects_bound_socket_when_connect_fails` 钉住。做"占用者"的测试/脚本
   **不要设 `SO_REUSEADDR`**（Windows 上允许两个进程同绑一个端口）。
10. **`speech.*` 事件的 `request_id` 是语音层 utterance id**，不是对话轮 `request_id`：
    探针跨层关联按时间窗口做（`docs/gpu-schedule-probe.json` 的 `speech_timeline_ms` 说明这一点）。
11. **Qwen 的 `speech.synthesis.started` 语义是"段交给 Provider"**（`push_text` 只入队，真正的
    HTTP 请求在 `audio()` 里逐段串行发）；应用级测量里"首个 synthesis.started"仍是门控放行后的
    首个可听事件，用它判定"是否与 LLM decode 重叠"是有效的（首个请求立即发）。
12. **测试规模**：全量 `uv run pytest` = **789 passed**（含审查修补新增的竞态/健康刷新/tooltip/配置校验/默认值一致性用例）；
    04E 新增 106 项：调度 28 + runtime 40 + 集成 19 + UI 3 + 配置/事件/工厂/运行时 16。
13. **应用级基线 130.34 与直连基线 131.75 不同**：应用层多了事件发布/切句/JSON 校验开销（约 1%），
    两组数字各自内部对比，不混用。
14. **`ChatTemplate` 关思考**：本机 9B 构建接受 `--reasoning off`；应用级探针另外在 Provider 里注入
    `enable_thinking=false`（`thinking: "off"`），两类证据的 `reasoning_chars` 都为 0。
15. **独立审查发现的修补（2026-11-18 收口）**：
    - **B1 取消竞态**：`_acquire_gpu_slot` 在 `await task` 期间被取消、而 acquire 恰好已完成占槽时，
      原实现只 `not task.done()` 才取消 → 槽位永久占用、后续语音全部静默。修法：`except BaseException`
      里若任务已完成且无异常、decision 是 granted/overlap → 补一次 `release_tts()`；
      回归 `test_cancel_race_after_slot_granted_releases_slot_once`（已实测"去掉修复即失败"）。
    - **M1 WAITING 的 detail 进 tooltip**：`UiState` 新增 `speech_detail_changed` 信号，面板据此写 tooltip；
      `test_ui_shows_waiting_state_with_reason_in_tooltip` 钉住。
    - **M2 managed 健康刷新**：启动期健康检查发生在 `speech.start()` 之前，原实现 `service=starting/failed`
      永远上不了健康行。修法：`_run_service_start` 完成（成功/失败）后重跑健康并发布
      `system.health.changed(component="tts")`；`test_managed_start_failure_publishes_tts_health_with_service_detail`。
    - **M3 回环判据**：`0.0.0.0` 不是回环（通配绑定），从白名单移除；改用 `localhost` 或
      `ipaddress.is_loopback`（因此 `127.0.0.2` 也被接受）。配置期测试覆盖。
    - **m1 状态闪烁**：取到槽后直接进 `SYNTHESIZING`（不再闪 IDLE）。
    - **m7 managed.port 与 base_url 端口不一致**：配置期报错（否则白等 `startup_timeout_s`）。
    - **m6 `mode=disabled`**：设置页 `speech_available` 与健康 detail 都按"未启用"处理。
    - **m4/m5 文档口径**：日志"启动期轮转一次、运行中不滚动"写清；detach 后 stdout 无人读的限制记入已知限制。
