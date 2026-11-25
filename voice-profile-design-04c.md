# 04C 收尾：多音色 VoiceProfile 设计与 04F 实现建议

> 任务：`04C_会话任务文本_多音色需求修订.md`；基于固定 `qwentts.cpp` commit **`2edb35f`**（2026-11-17）的源码/真实 API，以及 04C 已完成的 29 条人工盲听。
> 环境：Windows 10 19045 / RTX 4090 24 GB / i7-13700KF / 32 GB；Qwen3-TTS GGML 1.7B Q8_0。
> 本文是**设计/需求与接口建议**，不实现 Voice Library、不改生产代码、不创建或删除真实用户音色资产。
> 结论：**多 Profile 方案；整体 `CONDITIONAL GO`；不要求选定唯一默认音色；每个 Profile 独立验收。**

---

## 0. TL;DR

1. **一个 `tts-server` 实例只加载一个 Talker Checkpoint**。Base、CustomVoice、VoiceDesign 不能在同一实例中无重启切换；需要不同服务进程/端口或受控重启。
2. **Base 服务可以在同一进程内注册多个参考克隆 voice**：`POST /v1/audio/voices` 可多次注册，`voice` 按名称逐请求切换；注册表是**进程内存**，重启即空，应用必须保存 WAV/`.spk/.rvq` 并在服务启动后重新注册。
3. **CustomVoice 只支持当前模型自带的 9 个 speaker**，注册 voice 会 400；**VoiceDesign 只接受 `instructions` 设计描述**，注册 voice 也会 400。两者与 Base 不是同一模式。
4. 产品层统一用 `VoiceProfile` 呈现三类能力，但必须显示 `profile_type` 与 `talker_mode` 兼容性；**不能把三种底层能力伪装成同一种**。
5. 选择优先级：**per-response > per-session > global default**；没有默认/不兼容时进入明确的“未配置/不可用”，**不得回退到 `vivian`**；每个 utterance 冻结 Profile Snapshot，切换只影响之后新建的请求。
6. 用户中文参考音频/人工校对 `ref_text` **仍未提供**；`reference_clone` 和 ICL 保持 Pending；本次不下载、不克隆第三方声音。
7. 04F 应做 Voice Library/Profile CRUD、导入/校验/资产生命周期、Profile 兼容标记、三级选择、Provider/Runtime 绑定；不做模型训练、不做无重启跨模式切换、不做第三方声音、不承诺情绪控制。

---

## 1. qwentts.cpp 多音色能力（源码/API 证据）

### 1.1 一个服务实例 = 一个 Talker

- `tools/tts-server.cpp:86-90` 只解析一个 `--model`；`:125` 用 `qt_init` 加载该 Talker；没有 reload/switch 路由。
- `src/tts-server.h:623-633` 路由只有 `/v1/audio/speech`、`/v1/models`、`/v1/audio/voices`、`/health`；没有模型管理端点。
- 结论：**切换 Base/CustomVoice/VoiceDesign 必须停止并用另一份 `--model` 启动，或在另一个端口启动第二实例**。一个实例可服务"当前 Talker 模式兼容"的多个 Profile，但跨模式需要 Runtime 切换。

### 1.2 Base：同一实例可注册多个克隆 voice

源码事实：

| 事实 | 证据 |
| --- | --- |
| 注册表是进程内 `unordered_map<string, voice_entry>` | `tools/tts-server.cpp:39-40`；`g_voices` 无磁盘持久化 |
| `POST /v1/audio/voices` 仅 Base 可用 | `tools/tts-server.cpp:151-156`；非 Base 返回 400 `voice registration is only valid for base models` |
| 注册体：`name` + `wav_b64` 或 `spk_b64+rvq_b64`；`ref_text` 可选 | `src/tts-server.h:444-490` |
| 注册 WAV 时服务端调用 `qt_extract_voice_ref`，提取 speaker embedding + RVQ codes | `tools/tts-server.cpp:158-173`；`src/qwen.cpp:535-562` |
| 多个 voice 按 `voice` 名称选择；已注册 voice 优先于同名内置 speaker | `tools/tts-server.cpp:252-279` |
| `GET /v1/audio/voices` 返回内置 speaker（如有）+ registered 名字 | `src/tts-server.h:551-570` |
| `DELETE /v1/audio/voices/{name}` 删除注册项 | `src/tts-server.h:517-527` |
| 每次请求在锁内复制 latents，避免替换/删除时交叉 | `tools/tts-server.cpp:242-272` |

**多 voice 语义**：

- 可以在同一个 Base 服务中连续注册 N 个名字（如 `natsuki_a`、`natsuki_b`），然后每个请求传不同的 `voice`；**不需要重启服务**。
- 所有 voice 共享同一个 Base Talker/Codec/KV/显存；voice latents 是进程内的小数组（spk + rvq），不是每个 voice 一份数 GB 模型。未单独测量每 voice 的 GPU 增量，不能声称"零成本"，但没有第二个 Talker 权重。
- 注册操作可在服务运行时进行；`max_batch=1` 时注册/合成共用串行计算 worker，注册会短暂占用一次提取计算。
- **重启后 `g_voices` 为空**：服务不写 registry 文件（源码中没有持久化路径）；应用必须保存源 WAV 或 `.spk/.rvq`，在启动/重连后重新 `POST /v1/audio/voices`。这是 04F 必须处理的启动流程。
- **x-vector vs ICL**：`ref_text` 为空 → 只用 spk（x-vector）；`ref_text` + codes → ICL（`src/qwen.cpp` 校验 `ref_text` 需要 ref-wav 或 rvq）。ICL 的文本必须人工校对；04C 的英文自动转写已证明不可用。
- **2026-11-25 最小探针**（`benchmarks/04c/multi-profile-probe.json`）：在同一个 Base 1.7B 服务中注册 `probe_a`、`probe_b`，`GET` 同时列出两个 `kind=registered`，两次 `POST /v1/audio/speech` 分别用不同名字均 200（各 145,920 B PCM），`DELETE` 后列表为空；`multi-profile-restart.json` 证明重启后注册表为空（需重新注册）。

### 1.3 CustomVoice：9 个内置 speaker，不能注册克隆

- `tools/tts-server.cpp:140-143` 从 `qt_n_speakers`/`qt_speaker_name` 读取当前模型 speaker 表；04C 实测 1.7B CustomVoice 为 9 个：`serena`/`vivian`/`uncle_fu`/`ryan`/`aiden`/`ono_anna`/`sohee`/`eric`/`dylan`。
- `POST /v1/audio/voices` 在 CustomVoice 上返回 400（源码同上）；不能把用户 WAV 注册成 speaker。
- 请求 `voice` 必须是当前模型 speaker 名；未知 → 400 `unknown voice`；`Instructions` 可选但 04C 判定不稳定。
- 一个 CV Talker 进程可服务这 9 个 speaker；切换 speaker 只改请求字段，不重载模型。
- Profile 的 `speaker_id` 必须与当前加载的 CV 模型兼容；换 CV 模型/版本时 `speaker_id` 可能变化，需重新校验。

### 1.4 VoiceDesign：固定设计描述，不能注册克隆

- 请求必须提供非空 `instructions`，`voice` 为空；否则 400 `voice_design models require --instruct`（`src/qwen.cpp:677-683`）。
- `POST /v1/audio/voices` 在 VoiceDesign 上返回 400；没有 speaker 表（`speakers=0`）。
- `instructions` 是**音色设计描述**（gender/age/pitch/style），不是逐句情绪参数；04C 只验证了 F0/描述响应，没有把 `happy/sad/angry` 当作 VD 的稳定控制手段。
- 生成音频可以通过 `response_format=wav` 导出完整 WAV（`tts-server.h` 的 speech 路由）；服务端没有"保存设计资产"的端点，要持久化只能由应用保存 WAV。
- **VoiceDesign → Base 参考资产**：生成的 WAV 可以作为 Base 的参考 WAV 再注册（新 Profile），并用 `qwen-codec --talker <Base GGUF>` 生成 `.spk/.rvq`；这是离线转换流程，**不是同一服务模式切换**。2026-11-25 最小探针（`benchmarks/04c/multi-profile-vd-to-base.json`）把 04C 生成的 `vd_cute_girl_s42.wav` 在 Base 1.7B 服务上注册为临时 voice 并合成 200，证明 WAV 级别可对接；完整产品流程（保存、试听、确认、转 Profile、清理）仍属 04F，未验收。

### 1.5 一个实例能服务哪些 Profile

| 当前加载模式 | `reference_clone` | `builtin_speaker` | `designed_voice` | 备注 |
| --- | --- | --- | --- | --- |
| Base | ✅ 可注册多个；x-vector/ICL | ❌ 无 speaker 表 | ❌ 无 instruct 设计模式 | 需要用户参考；ICL 需人工 ref_text |
| CustomVoice | ❌ 注册 400 | ✅ 当前模型 9 speaker | ❌ | 只能使用当前 CV 模型 speaker |
| VoiceDesign | ❌ 注册 400 | ❌ | ✅ 每次请求固定描述 | 需 VD Talker；描述固定为 Profile |
| 其他/未加载 | ❌ | ❌ | ❌ | 未配置/不可用，不能猜 |

> 结论：**同一服务实例只能服务同一 Talker 模式的 Profile 子集**；跨模式必须切换服务。应用层要有明确的"当前端点能力"记录，不能靠 `GET /v1/audio/voices`（它不暴露 model_type）。

---

## 2. Profile 类型

| `profile_type` | 来源 | 必需后端 | 请求映射 | 资产 |
| --- | --- | --- | --- | --- |
| `reference_clone` | 用户拥有/授权的参考 WAV | Base | `voice=<registered_name>`；x-vector 无 ref_text；ICL 需 `ref_text`（注册时） | source WAV、normalized WAV、可选 `.spk/.rvq`、ref_text |
| `builtin_speaker` | Talker 内置 speaker | CustomVoice | `voice=<speaker_id>`；可选 `instructions`（experimental） | 无需音频资产；记录 speaker_id/model/version |
| `designed_voice` | 固定自然语言描述 | VoiceDesign | `voice=""`；`instructions=<design_description>` | 固定描述；可选 preview WAV；可选离线生成的 Base ref 资产 |

可选衍生类型（不是新的后端模式）：

- `designed_voice → reference_clone`：用 VoiceDesign 生成 WAV，经用户确认后作为 Base 参考，建立新的 `reference_clone` Profile；保留来源与派生关系。它不是"VoiceDesign 和 Base 直接互换"。

---

## 3. 兼容矩阵

| Profile / 端点 | Base 1.7B/0.6B | CustomVoice 1.7B/0.6B | VoiceDesign 1.7B | 需要切换服务? |
| --- | --- | --- | --- | --- |
| `reference_clone`（x-vector） | ✅ 可注册/请求 | ❌ 400 | ❌ 400 | 需要 Base 服务 |
| `reference_clone`（ICL） | ✅ 需 ref_text + codes | ❌ | ❌ | 需要 Base 服务 |
| `builtin_speaker` | ❌ 无 speaker 表 | ✅ 9 speaker；当前模型限定 | ❌ | 需要 CV 服务 |
| `designed_voice` | ❌ | ❌ | ✅ `instructions` 必填 | 需要 VD 服务 |

服务端点记录（应用配置/数据库）至少要有：

```text
endpoint_id, provider=qwen3_tts, base_url, talker_mode,
model_id, model_commit, quantization, model_path, codec_path,
status (external/managed/disabled), last_health
```

`GET /v1/audio/voices` 只给名字/kind，不告诉 `model_type`；**应用必须自己记录 `talker_mode`**，不能从接口猜。

---

## 4. VoiceProfile 最小数据结构

### 4.1 SQLite 元数据（建议）

```sql
-- 一个 Profile 只描述"音色身份"，不含逐句情绪参数
CREATE TABLE voice_profiles (
    profile_id          TEXT PRIMARY KEY,          -- UUID/ULID
    display_name        TEXT NOT NULL,
    profile_type        TEXT NOT NULL CHECK (profile_type IN
                          ('reference_clone','builtin_speaker','designed_voice')),
    engine_id           TEXT NOT NULL,             -- e.g. qwen3_tts
    talker_mode          TEXT NOT NULL CHECK (talker_mode IN
                          ('base','custom_voice','voice_design')),
    model_id            TEXT NOT NULL,             -- 逻辑模型/版本，如 qwen3-tts-1.7b-cv-q8
    model_commit        TEXT,                      -- qwentts.cpp 2edb35f
    quantization        TEXT,                      -- Q8_0
    speaker_id          TEXT,                      -- builtin_speaker 必填
    design_instructions TEXT,                      -- designed_voice 必填
    ref_text            TEXT,                      -- reference_clone 可选（ICL 必填）
    ref_text_source     TEXT CHECK (ref_text_source IN ('manual','auto_unverified')) DEFAULT NULL,
    language            TEXT,                      -- Chinese/auto
    tags_json           TEXT NOT NULL DEFAULT '[]',
    user_notes          TEXT,
    source_license_note TEXT,                      -- 来源/授权备注
    validation_status   TEXT NOT NULL CHECK (validation_status IN
                          ('candidate','verified','pending','rejected','blocked')),
    created_at          TEXT NOT NULL,
    updated_at          TEXT NOT NULL
);

CREATE TABLE voice_assets (
    asset_id            TEXT PRIMARY KEY,
    profile_id          TEXT NOT NULL REFERENCES voice_profiles(profile_id),
    role                TEXT NOT NULL CHECK (role IN
                          ('source_wav','normalized_wav','ref_text','spk','rvq','preview_wav')),
    rel_path            TEXT NOT NULL,             -- 相对 voice_library 根
    sha256              TEXT NOT NULL,
    format              TEXT,                      -- wav/pcm/f32...
    sample_rate         INTEGER,
    channels            INTEGER,
    duration_ms         INTEGER,
    bytes               INTEGER,
    backend_generated   INTEGER NOT NULL DEFAULT 0,
    created_at          TEXT NOT NULL,
    UNIQUE(profile_id, role, sha256)
);

-- 只保存 global/session 偏好；per-response 不进 DB
CREATE TABLE voice_preferences (
    scope               TEXT NOT NULL CHECK (scope IN ('global','session')),
    scope_id            TEXT NOT NULL,
    profile_id          TEXT REFERENCES voice_profiles(profile_id),
    updated_at          TEXT NOT NULL,
    PRIMARY KEY(scope, scope_id)
);

-- 可选：端点能力表，供兼容性检查
CREATE TABLE voice_endpoints (
    endpoint_id         TEXT PRIMARY KEY,
    provider             TEXT NOT NULL,
    base_url            TEXT NOT NULL,
    talker_mode          TEXT NOT NULL,
    model_id            TEXT NOT NULL,
    model_commit        TEXT,
    quantization        TEXT,
    status              TEXT NOT NULL,
    last_health_at      TEXT
);
```

约束/规则：

- `builtin_speaker` 必须有 `speaker_id`；`designed_voice` 必须有 `design_instructions`；`reference_clone` 必须至少有一个 `source_wav` asset。
- `ref_text` 只能来自人工校对（`manual`）才允许 ICL；`auto_unverified` 不允许标记 verified。
- 大文件不入 SQLite；只存相对路径/hash/格式。
- 绝对路径、参考音频内容、ref_text 不暴露给 LLM；LLM 只接收/返回 `profile_id`，由应用侧解析。
- `validation_status` 是**单 Profile** 状态，不允许用整体 04C 结论覆盖。

### 4.2 JSON 示例（API/配置传递）

```json
{
  "profile_id": "01J...",
  "display_name": "Natsuki 中文参考 A",
  "profile_type": "reference_clone",
  "engine_id": "qwen3_tts",
  "talker_mode": "base",
  "model_id": "qwen3-tts-1.7b-base-q8",
  "model_commit": "2edb35f",
  "quantization": "Q8_0",
  "speaker_id": null,
  "design_instructions": null,
  "ref_text_source": "manual",
  "language": "Chinese",
  "assets": {
    "source_wav": "voice_library/01J.../source/ref.wav",
    "normalized_wav": "voice_library/01J.../normalized/ref_24k_mono.wav",
    "ref_text": "voice_library/01J.../ref_text.txt",
    "spk": "voice_library/01J.../cache/ref.spk",
    "rvq": "voice_library/01J.../cache/ref.rvq"
  },
  "validation_status": "pending",
  "source_license_note": "用户本人录制；仅本地使用",
  "tags": ["中文", "女声"],
  "user_notes": ""
}
```

### 4.3 资产目录

```text
voice_library/
  <profile_id>/
    source/ref.wav                 # 原始文件，保留不改
    normalized/ref_24k_mono.wav    # 非破坏性规范化副本
    ref_text.txt                   # 人工校对（若 ICL）
    cache/ref.spk                  # qwen-codec --talker（Base）生成，可选
    cache/ref.rvq                  # qwen-codec 生成，可选
    preview/*.wav                  # 试听/设计样本
```

相对路径以 `voice_library` 为根；源文件 hash + 预处理版本写入元数据；删除只删派生资产，源文件删除需二次确认。

---

## 5. 选择作用域与请求绑定

### 5.1 优先级

```text
per-response override  >  per-session preference  >  global default
```

- 三者均未设置：`Unconfigured` → **语音关闭/未配置**，文字照常；UI 提示“请选择音色”，**不得回退 `vivian`**。
- Profile 与当前 endpoint 的 `talker_mode` 不兼容：`Incompatible` → 提示需要切换/启动对应 Talker 服务；**不得静默换成同名的其他 speaker**。
- 选择是用户偏好，不是系统唯一声音；用户可以有多个 Profile，默认只是“没覆盖时用哪个”。

### 5.2 解析与快照

建议新增（04F）：

```python
class VoiceSnapshot(BaseModel):
    profile_id: str
    profile_type: Literal["reference_clone", "builtin_speaker", "designed_voice"]
    talker_mode: Literal["base", "custom_voice", "voice_design"]
    model_id: str
    speaker_id: str | None
    registered_voice: str | None       # Base 注册名
    design_instructions: str | None
    language: str | None
    asset_hashes: dict[str, str]       # source/ref_text/spk/rvq 版本
    style: "SpeechStyle | None"         # 与音色身份分开
```

- `VoiceResolver.resolve(profile_id, session_id)` 读 per-response override → session → global；返回 Snapshot / Unconfigured / Incompatible。
- `SpeechCoordinator` 在一轮开始时（`_start_turn`/首次 `feed_text`）解析并**冻结** snapshot；放入 `_Turn`；每个 segment 使用同一个 snapshot。用户中途切换只影响**下一次新建的 utterance**；已生成/播放/排队的当前 utterance 不变。
- 逐段请求的 `voice`/`instructions` 从 snapshot 映射，不从全局配置每次重读。
- 后台任务：Task 可带 `voice_profile_id` override；没有则按 session/global 解析；任务完成摘要播报在创建摘要语音时冻结；任务内部不直接开 TTS 通道。

### 5.3 与现有代码的最小绑定

| 现有 | 04F 建议 |
| --- | --- |
| `providers/tts_base.py::SpeechRequest.voice: str|None` | 增加可选 `voice_snapshot: VoiceSnapshot|None`；保留 `voice` 兼容旧 Provider/MOSS |
| `speech/coordinator.py` 的 `self._voice`（单一 voice） | 注入 `VoiceResolver`/`voice_snapshot_provider`；`_start_turn` 冻结 snapshot；`_pump` 传 snapshot；不再从单一 `_voice` 读 |
| `providers/qwen3_tts.py::open_stream` 使用 `self._voice`/`self._instructions` | 按 snapshot 映射：Base registered name / CV speaker id / VD design instructions；先校验 `talker_mode == config.model_type`；不兼容抛 `tts_profile_incompatible` |
| `Qwen3TTSConfig.voice` | 04F 保留为兼容/默认旧值；新增 `voice_profile_id` 或由 resolver 注入；默认 `providers.tts.type=none` |
| `speech.synthesis.started` payload `voice` | 增加可选 `voice_profile_id`/`profile_type`（additive；不新增事件类型） |
| `speech/scheduler.py` / `services/qwen_tts_runtime.py` | 记录 active endpoint `talker_mode`；Profile 不兼容时提示/受控重启；不把切换伪装成即时参数 |
| `SpeechPlayer` | 不改；仍按 utterance 播放；切换不会改变已入队音频 |

### 5.4 切换/不兼容/中断

- **同一模式、不同 Profile**：Base 多 voice 可以在同一服务切换；CV 不同 speaker 也可以；VD 不同描述需要重新生成，但同一 VD 服务可逐请求不同 `instructions`（是否作为不同 Profile 由应用缓存 preview 决定）。切换发生在 utterance 边界。
- **不同 Talker 模式**：必须切换服务；当前 04E `managed` 可启停，但没有"reload model" API；04F 要加受控 start/stop（或提示 external 用户手动重启）。切换前：
  - 停止新建 TTS 请求；当前 utterance 可让播放器播完（若服务仍可用），或取消并降级为文字；
  - 不把旧服务音频混入新 Profile；新 utterance 用新 endpoint；
  - 失败时文字仍可用，UI 明确“语音不可用/需要启动服务”。
- **删除正在使用的 Profile**：先停止/替换引用；冻结中的 snapshot 如果资产被删除，当前 utterance 已加载的 latents/音频可继续，后续新请求失败并提示；不静默换声。
- **重启/重连**：Base 服务重启后重新注册所有 `reference_clone`（从 WAV 或 `.spk/.rvq`）；注册成本按 04C 测得约 0.39–0.55 s/voice（暖机），数量多时启动可并行/串行分批；不要在 UI 线程做。

---

## 6. 用户个人参考音频导入

### 6.1 当前仓库盘点（2026-11-25）

| 文件 | 格式 | 时长 | 采样率/声道 | 状态 |
| --- | --- | ---: | --- | --- |
| `F:\AI_Project\Qwen3-TTS-Experiments\assets\ref_zh_prompt_audio1_16k.wav` | WAV PCM | 11.73 s | 16 kHz / mono | **英文** MOSS 女声；仅技术对照，不是用户中文参考 |
| `...\assets\ref_zh_prompt_audio1_24k.wav` | WAV PCM | 11.73 s | 24 kHz / mono | 同上，重采样副本 |
| `F:\AI_Project\MOSS-TTS\assets\audio\reference_zh*.wav` | WAV/MP3 | 3.1–14.4 s | 48 kHz 等 | **上游 demo/第三方声音**，未作为用户资产、未用于克隆 |
| 用户个人中文参考 | — | — | — | **未在仓库/实验目录中找到**；不得假定已存在 |

现有英文参考的体检值：峰值 -4.00 dBFS、RMS -22.19 dBFS、静音 14.8%、无削波（`docs/ref-audio-check-example.json`）；但它不能替代中文参考。

### 6.2 待用户提供清单

- [ ] **每个目标音色 1 段** 5–15 s（推荐 8–12 s）中文、单人、干净 WAV；24/48 kHz，单声道，16/24-bit；峰值 -6~-3 dBFS；无削波、无 BGM、无混响、无强降噪/AGC。
- [ ] 每段对应的**人工逐字校对** `ref_text`（数字/英文按实际读法；ICL 必需；x-vector 可空）。
- [ ] 来源/授权备注：本人录制 / 已获得书面授权；不得第三方声音。
- [ ] 如果希望“设计音色”：提供 VoiceDesign 描述文本；先出 preview，再决定是否转为 Base 参考资产。

导入规范：

1. 保留原文件，不覆盖；复制到 `voice_library/<profile_id>/source/` 并计算 SHA-256；
2. 用 `tools/ref_audio_check.py` 检查格式/时长/峰值/RMS/静音/削波/DC；失败给可读诊断；
3. 生成独立 `normalized/ref_24k_mono.wav`（24 kHz mono），记录预处理参数/版本；
4. `ref_text` 人工校对后保存；标记 `ref_text_source=manual`；未校对只能 `auto_unverified`，不能进入 ICL 验证态；
5. Base 注册：`POST /v1/audio/voices`（WAV 或 `.spk/.rvq` + ref_text）；注册成功后 `GET` 核对；
6. 可选 `qwen-codec --model tokenizer --talker <Base GGUF> -i ref.wav` 生成 `ref.rvq`/`ref.spk`，作为缓存；服务重启后可用它们快速重注册；
7. 默认仅本地处理，不自动上传；删除源文件必须二次确认，派生缓存可单独删除。

> 当前 `reference_clone` 的验收状态：**Pending**；在用户提供中文参考 + 人工 `ref_text` 前，不得标记 verified，也不得默认启用 ICL。

---

## 7. VoiceProfile 与 SpeechStyle 分离

| 概念 | 决定 | 当前能力 |
| --- | --- | --- |
| `VoiceProfile` | 谁在说话（身份） | 三类 Profile；多 Profile 可并存；按 Profile 验收 |
| `SpeechStyle` | 这句话如何表达（情绪/语气/语速） | 当前没有稳定、可正式启用的通用能力 |

| 模式 | `instructions` | 结论 |
| --- | --- | --- |
| Base GGML | 不支持（HTTP 400） | 必须拒绝；不得用提示词伪造 |
| CustomVoice | 可选，按 speaker 生效 | `vivian` 6/7 情绪不合格，`fast` 仅“勉强”；所有 speaker/指令分别 experimental |
| VoiceDesign | 固定音色设计描述 | 不是逐句 `happy/sad/angry`；只能作为 `designed_voice` 的固定描述 |

结论：**换 VoiceProfile 不等于获得情绪控制**；不要把 `cute_girl` 或 `ryan` 的语音身份验收外推为情绪验收。04F 可以保留 `SpeechStyle` 接口位，但默认关闭；后续如要开放，必须按 Profile/speaker/指令逐项验收。

---

## 8. 现有候选 Profile 目录（29 条评分，不是唯一赢家）

| Profile 候选 | 类型 | 当前证据 | 状态 |
| --- | --- | --- | --- |
| `cute_girl`（可爱小女孩，高音，轻快） | designed_voice | A05 = 5/5/5/5/1；n=1；头部吞字 | **candidate**，补 10 文本 |
| `ryan` | builtin_speaker | A20 = 4/4/5/5/1；n=1 | **candidate**，补 10 文本 |
| `young_female_gentle` | designed_voice | A12 = 4/4/4/5/2；n=1 | **candidate**，补 10 文本 |
| `mature_male_deep` | designed_voice | A18 = 4/4/3/5/2；n=1 | **candidate**，补 10 文本 |
| Base x-vector（英文参考） | reference_clone | 5 条均值 4.00/3.80/N/A/4.60/2.00；T07 弱 | **conditional**，需中文参考 + T07 复测 |
| `vivian` | builtin_speaker | 12 条均值 3.75/2.58/2.43/4.08/3.25；情绪差 | **default NO-GO**；长文本候选，不默认 |
| `serena`/`uncle_fu`/`aiden` | builtin_speaker | 4/1/4/5、3/1/5/4、1/1/3/5（n=1） | 当前不建默认；可保留对照 |
| `news_anchor`/`stern_male`/`old_male_hoarse` | designed_voice | 1/1/2/3/5、1/1/1/3/5、2/2/2/5/4 | **rejected** |
| CV 0.6B `vivian` | builtin_speaker | A29 3/1/2/3/4 | **rejected** |
| Base ICL（EN auto ref_text） | reference_clone | A04 4/4/3/2/2 | **Pending/No-Go** |

> 这些 Profile 可以同时存在于 Voice Library；默认只是偏好，不是“必须选一个”。单条样本评分不得写成“稳定、全面验收通过”。

---

## 9. 04F 实现建议

### 9.1 必做

1. **数据层**：`voice_profiles` / `voice_assets` / `voice_preferences`（SQLite）；迁移脚本；资产根目录；相对路径与 hash；不把 WAV/PCM/模型缓存放进 DB。
2. **导入与资产生命周期**：WAV 导入、格式/时长/采样率/声道/峰值/静音/削波检查、非破坏性 normalized 副本、人工 ref_text、来源/授权备注；保留原件；删除源文件二次确认。
3. **三类 Profile**：
   - `reference_clone`：Base 注册/多 voice 缓存/重启重注册/ICL 人工文本门槛；
   - `builtin_speaker`：记录当前 CV 模型 speaker_id/模型版本；不注册 voice；
   - `designed_voice`：固定 description；preview；可选 VoiceDesign→Base 离线转换。
4. **兼容性**：active endpoint `talker_mode`/model_id；Profile 兼容标记；不兼容时提示；不能无重启跨模式；`managed` 时受控 start/stop；`external` 时提示用户。
5. **选择与快照**：global/session/per-response 解析；utterance 冻结；切换只影响新 utterance；无默认/不兼容进入未配置/不可用，禁止回退 `vivian`；后台任务继承/覆盖规则。
6. **Provider 绑定**：`Qwen3TTSProvider` 接受 `VoiceSnapshot` 并映射到 `voice`/`instructions`；新增 `tts_profile_incompatible`/`tts_profile_unavailable` 错误码（或复用现有错误码并加 detail）；不把绝对路径/ref_text 传给 LLM。
7. **UI**：Voice Library 列表/试听/重命名/删除/默认设置/会话覆盖/单次覆盖；显示 Profile 类型、兼容性、验证状态、样本量；明确 experimental 能力。
8. **测试**：
   - Profile schema/迁移；resolver 优先级；snapshot 冻结；无 `vivian` 回退；
   - WAV 导入校验；non-destructive；hash；
   - Base 多 voice 注册/切换/重启重注册（假服务 + 真服务可选）；
   - 非 Base 注册 400；跨模式不兼容错误；
   - VoiceDesign→Base 离线转换（如果实现）；
   - 04D/04E 现有 Provider/调度/播放测试不回归。
9. **运行时**：`QwenTtsRuntime` active endpoint 记录/重启；Profile 切换与播放/合成取消边界；不误杀外部进程。

### 9.2 依赖的既有接口

- `providers/tts_base.py`：`SpeechRequest`/`SpeechStream`/`SpeechProvider`；`AudioFormat`。
- `speech/coordinator.py`：一轮一个 utterance、segment 队列、取消/降级；需要扩展 voice snapshot，但不改段序/事件契约。
- `providers/qwen3_tts.py`：HTTP 契约、voice 校验、instructions 拒绝、`max_batch=1` 串行、断连取消。
- `services/qwen_tts_runtime.py` / `speech/scheduler.py`（04E）：managed 启停、llm_priority、非硬隔离。
- `scripts/qwen_voice_register.py`：WAV 校验/注册逻辑可复用。
- `tools/ref_audio_check.py`（实验目录）：导入检查。
- `qwen-codec`：Base `.spk/.rvq` 缓存生成。
- `docs/qwen3-tts-provider.md`：HTTP 契约与错误码。

### 9.3 明确不做

- 不在一个 `tts-server` 实例中无重启切换 Base/CustomVoice/VoiceDesign；
- 不训练/微调/重新量化模型；
- 不下载或克隆第三方声音；
- 不把 VoiceDesign 描述当逐句情绪控制；
- 不承诺 CustomVoice 情绪稳定；
- 不把参考音频/绝对路径/ref_text 发给 LLM；
- 不把 WAV/PCM/模型缓存写入 SQLite；
- 不实现 GPU 硬隔离/显存分区；
- 不修改 04D/04E 已验证的 Provider/播放/调度语义（只做兼容扩展）。

---

## 10. 最终报告必须回答的 8 个问题

1. **单个服务实例支持哪些 Profile？**
   一个 `tts-server` 固定加载一个 Talker：Base 实例支持 `reference_clone`（可多个注册 voice）；CustomVoice 实例支持 `builtin_speaker`（当前模型 9 个）；VoiceDesign 实例支持 `designed_voice`。跨模式需要另一个服务/重启；不能在同一实例无重启切换。

2. **多个参考 WAV 能否在一个 Base 服务中分别注册并按名称切换？缓存/重启要求？**
   可以。`POST /v1/audio/voices` 可多次注册，`voice` 名称逐请求切换；注册表是内存 map，**服务重启即空**；应用保存 WAV 或 `.spk/.rvq`，启动后按 Profile 重注册；注册约 0.39–0.55 s/voice（暖机，04C），不注册也可从 WAV 现场提取；`max_batch=1` 时注册/合成串行。

3. **三类 Profile 如何产品化统一又不伪装？**
   统一 `VoiceProfile` 模型 + Voice Library UI；用 `profile_type`/`talker_mode`/`model_id`/`compatibility` 显式展示；请求由 resolver 映射到 Base registered name / CV speaker_id / VD instructions；UI 标签写清“参考克隆 / 内置 speaker / 描述设计”，绝不把 VD 描述说成情绪或把 CV 说成克隆。

4. **最小数据结构与请求绑定？**
   `voice_profiles` + `voice_assets` + `voice_preferences`（SQLite；资产文件系统）；Profile 关键字段见 §4；`SpeechRequest.voice_snapshot`（冻结 snapshot）替代单一 `voice` 语义；Provider 负责 snapshot→后端字段。

5. **全局/会话/单次如何解析？utterance 如何不串音色？**
   per-response > per-session > global；解析后冻结 snapshot；当前 utterance/队列保持 snapshot，切换只影响下一次新 utterance；无默认/不兼容 → 未配置/不可用，绝不回退 `vivian`；删除/切换在 utterance 边界处理。

6. **用户参考音频未导入/核验时哪些必须 Pending？**
   `reference_clone` 全部 Pending（无中文参考）；ICL 额外要求人工 ref_text，Pending/No-Go；任何 n=1 候选的 10 文本复测未做 → candidate；情绪全部 experimental；默认 Profile 不得设置。

7. **04F 做什么、依赖什么、不做什么？**
   见 §9：Voice Library/Profile CRUD、导入/校验/资产、兼容标记、三级选择、Provider/Runtime 绑定、测试；依赖现有 Provider/Coordinator/Runtime/Scheduler/注册脚本；不做模型训练、不做无重启跨模式、不做第三方声音、不承诺情绪、不把资产放 DB。

8. **最终结论与逐 Profile 状态？**
   整体 **CONDITIONAL GO**；**没有任何 Profile 达到 GO**（都缺 10 文本复测/中文参考）。逐 Profile：

| Profile | 判定 |
| --- | --- |
| `cute_girl`（designed_voice） | **CONDITIONAL**（n=1；补 10 文本 + 修头部吞字） |
| `ryan`（builtin_speaker） | **CONDITIONAL**（n=1；补 10 文本） |
| `young_female_gentle`（designed_voice） | **CONDITIONAL**（n=1；补 10 文本） |
| `mature_male_deep`（designed_voice） | **CONDITIONAL**（n=1；补 10 文本） |
| Base x-vector（reference_clone） | **CONDITIONAL**（需用户中文参考 + T07 复测） |
| `vivian`（builtin_speaker） | **NO-GO 作为默认**；长文本候选，不用于情绪 |
| `serena` / `uncle_fu` / `aiden` | **NO-GO** |
| VoiceDesign `news_anchor` / `stern_male` / `old_male_hoarse` | **NO-GO** |
| CustomVoice 0.6B `vivian` | **NO-GO** |
| Base ICL（EN auto ref_text） | **NO-GO / Pending**（可懂度 2；需中文参考 + 人工 ref_text） |

   情绪：全部 experimental/NO-GO 正式使用；Base instruct 不可用。不要求唯一默认；`providers.tts.type` 保持 `none` 直到某个 Profile 完成 10 文本复测并被用户确认。

---

## 11. IMPLEMENTATION_STATUS 更新建议（可合并文案）

> 以下仅文案；本次**没有修改主项目代码**，也没有把任何候选写成最终验收通过。

```markdown
### Phase 04C 收尾：多音色 VoiceProfile（2026-11-25，仅设计与文档）

- 04C 保留 29/29 条人工评分；不再要求选定唯一默认音色，改为 `reference_clone` /
  `builtin_speaker` / `designed_voice` 多 Profile（全局/会话/单次三级选择）。
- `qwentts.cpp 2edb35f` 实测：一个 `tts-server` 固定一个 Talker；Base 可在同一进程注册多个
  clone voice（内存 registry，重启丢失，应用需重注册）；CustomVoice 仅当前 9 个 speaker；
  VoiceDesign 用固定 `instructions` 描述；Base/CV/VD 不能在同一实例无重启切换。
- 当前候选：VD `cute_girl`/`young_female_gentle`/`mature_male_deep`、CV `ryan`、Base x-vector
  均为候选/待复测 Profile；单条样本不作为全面验收；`vivian` 默认与所有正式情绪指令 NO-GO。
- 用户中文参考音频、人工校对 ref_text、ICL 仍未完成 → `reference_clone` Pending/No-Go；
  Base instruct 不可用；情绪全部 experimental。
- 04F 范围：Voice Library/Profile CRUD、导入与非破坏性资产、兼容矩阵、三级选择、Profile Snapshot、
  Provider/Runtime 绑定；不做模型训练、不做无重启跨模式、不做第三方声音、不承诺情绪。
- 当前 `providers.tts.type` 仍为 `none`；未设置默认 Profile 时语音“未配置”，禁止回退 `vivian`。
- 本次只更新报告/设计文档，未改生产代码、Provider、SpeechCoordinator、AudioPlayer、Runtime 或主配置。
```

---

## 12. 证据索引

| 内容 | 路径/证据 |
| --- | --- |
| qwentts 服务路由/模型加载 | `tools/tts-server.cpp:86-143`、`src/tts-server.h:623-633` |
| Base 多 voice 注册表 | `tools/tts-server.cpp:39-40, 151-226, 252-279`；`src/tts-server.h:444-527, 551-570` |
| 模式校验 | `src/qwen.cpp:655-710` |
| reference 提取 | `src/qwen.cpp:535-562`；`src/qwen.h:182-213` |
| `.spk/.rvq` | `tools/qwen-codec.cpp:37-52, 202-240` |
| 04D Provider 契约 | `F:\AI_Project_Development\docs\qwen3-tts-provider.md` |
| 04C 评分/汇总 | `qwen3-tts-decision-04c.md` §9、`listening-sheet-04c.md`、`benchmarks/04c/listening-scores.json` |
| 多 Base voice 最小探针 | `benchmarks/04c/multi-profile-probe.json`、`multi-profile-restart.json` |
| VoiceDesign→Base WAV 探针 | `benchmarks/04c/multi-profile-vd-to-base.json`、`tools/multi_profile_probe.py` |
| 04C 音色/情绪/并发 | `qwen3-tts-voice-quality.md`、`qwen3-tts-emotion.md`、`qwen3-tts-concurrency.md` |
| 总纲 v2.1 | `00_项目总纲.md` §1.1/§2/§3.4/§12 |

### 未验证/待确认

- VoiceDesign 生成 WAV → Base reference → `.spk/.rvq` 的端到端产品链路：源码层可行，04C 未做完整产品验证。
- 多个 Base voice 的每 voice GPU/CPU 开销：未单独测量；只确认共享一个 Talker，没有第二份模型权重。
- 服务重启后自动重注册的耗时/并发策略：需 04F 实测。
- 多个 VoiceDesign `instructions` 的长期一致性和 seed 稳定性：当前 n=1/描述，不足以定论。
