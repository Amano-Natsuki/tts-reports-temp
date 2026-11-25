# 04C 决策：Qwen3-TTS 是否作为桌面助手常驻 TTS 后端

> 依据：`qwen3-tts-voice-quality.md`、`qwen3-tts-emotion.md`、`qwen3-tts-concurrency.md`、本轮原始数据 `benchmarks/04c/`、以及 2026-11-24 回收的 29 条人工试听评分（`docs/listening-sheet-04c.md` + `docs/listening-key-04c.md`）。
> 环境：Windows 10 19045 / RTX 4090 24 GB / i7-13700KF / 32 GB；qwentts.cpp **`2edb35f`**（2026-11-17），MSVC 2022 + CUDA 12.9，`sm_89-real`。
> 日期：2026-11-18 技术实测；**2026-11-24 人工试听回填；2026-11-25 按多音色产品需求修订**。
> **最终判定：`CONDITIONAL GO`（整体，按 VoiceProfile 分级）**——技术门槛通过；29/29 人工试听保留，`vivian` 默认被否，情绪指令不得正式启用；但**不再要求选定唯一最终音色**，改为多 Profile 目录：每个 Profile 独立验收、默认可选。逐 Profile 状态见 §9 与 `voice-profile-design-04c.md`。当前未配置默认 Profile 时，`providers.tts.type` 保持 `none`。

---

## 0. 2026-11-25 多音色修订（本轮收尾）

产品需求已明确：桌面助手需要**多音色自由管理和切换**，默认音色只是用户偏好，不是系统只能有一个永久说话人。因此 04C 的收尾动作从“选一个最终赢家”改为：

1. 保留并解释 29/29 条盲听分数（不重评、不覆盖）；
2. 把候选整理为 `reference_clone` / `builtin_speaker` / `designed_voice` 三类 `VoiceProfile`，可并存；
3. 按 Profile 分别标记：已验证 / 候选待复测 / Pending / NO-GO；
4. 明确用户中文参考音频与 ICL 仍待导入/校对；
5. 为 04F Voice Library 提供数据结构、兼容矩阵和选择优先级建议。

> **不要求用户只选一个声音**；`cute_girl`、`ryan`、`young_female_gentle`、`mature_male_deep`、Base x-vector 等都作为候选 Profile / 待复测 Profile 保留。详细设计见 [`voice-profile-design-04c.md`](voice-profile-design-04c.md)。

---

## 1. 判定

| 维度 | 结果 | 说明 |
| --- | --- | --- |
| 性能（TTFA/RTF/欠载） | ✅ 通过 | 1.7B CV/Base 暖机 TTFA 15–42 ms、RTF 0.09–0.10、71.2 s 长音频零欠载 |
| 显存（常驻/峰值） | ✅ 通过 | 1.7B CV/VD **+3,027 MiB（约 3.17 GB）** 常驻；1.7B Base 克隆 **+3,461/3,592 MiB（约 3.63/3.77 GB）**；均低于 ≤4/≤5 GB |
| 稳定性（20 连发/10 固定样本） | ✅ 通过 | 20/20 成功、零欠载、无崩溃；10 组固定样本全成功；speaker 交替无串音 |
| 取消（断连） | ✅ 通过 | 1.7B 断连后 313 ms 内 GPU 94%→1–5%，新请求 TTFA 61.5 ms |
| 并发（1.7B + 9B） | ✅ 通过（默认 llm_priority） | LLM Decode −61.0%（短）/−61.7%（长重叠）；TTS 不受损；串行首音频 2.7–3.0 s |
| 1.7B 并发数据 | ✅ 已单独留档 | 纠正 04B 摘要的 0.6B/1.7B 口径混用 |
| 情绪稳定性 | ❌ **未通过（已试听）** | GGML Base 完全不支持 instruct；CustomVoice `vivian` 的 7 条情绪/语气中 **6 条不合格**，`fast` 仅“勉强”；人工试听见 §9 |
| 中文音色人工验收 | ❌ **已完成，默认不通过** | 29/29 条已盲听；默认 `vivian` 在短句/中性/情绪上被否；`ryan`、VoiceDesign 个别描述、Base x-vector 有可用候选，见 §9 |
| 中文参考/克隆 | ⚠️ **Pending** | 无用户中文参考；ICL 单项 A04 可懂度 2（不达标）；x-vector 可用但需中文参考复测 |
| 试听数据完整性 | ⚠️ **分数完整，环境记录缺失** | 29/29 有分、无缺分/越界/缺解码；§1 的耳机/播放器/房间未记录（见 `listening-sheet-04c.md` §1） |
| 当前默认音色 `vivian` | ❌ **NO-GO** | 情绪/中性/慢速均被否；仅部分长文本（A08/A16/A26）尚可，不能作为无条件默认 |
| 多 Profile 方案 | ✅ **已定义** | `reference_clone`/`builtin_speaker`/`designed_voice` 并存；默认只是偏好；见 `voice-profile-design-04c.md` |

**判定理由**：

- 性能、显存、稳定性、取消、并发五个硬门槛都过；Qwen3-TTS GGML 明显优于 MOSS（MOSS RTF 1.24–1.95、12 GB、CER 36.6%）。
- **不能给 GO**：人工试听已经完成，但默认 `vivian` 与全部情绪指令被哥哥本人否决；中文参考/克隆仍未提供；没有换音色后的 10 文本复测。
- **不是 NO-GO（整体）**：Base x-vector、CustomVoice `ryan`、VoiceDesign `cute_girl` / `young_female_gentle` / `mature_male_deep` 有明确可用候选；技术链路可复现，换音色和复测即可继续。
- **但当前默认配置是 NO-GO**：`vivian` 默认、`vivian` 的 7 条情绪/语气指令、`serena`/`uncle_fu`/`aiden`、0.6B CV、ICL 均不得上线。

---

## 2. 推荐方案（04D 接入配置；Profile 待建，不要求唯一）

### 2.1 后端与模型

| 项 | 冻结值 |
| --- | --- |
| 后端 | qwentts.cpp GGML CUDA，commit **`2edb35f`**（2026-11-17） |
| 构建 | MSVC 2022 + CUDA 12.9，`-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=89-real`；`tts-server.exe` 同目录至少需 `ggml-base.dll` / `ggml-cpu.dll` / `ggml-cuda.dll`（04B 实测）；`qwen.dll` / `yyjson` 的运行时依赖需在 04D 用 dumpbin 核对，不凭猜测打包 |
| Talker | **`qwen-talker-1.7b-customvoice-Q8_0.gguf`**（2,042,834,304 B，本地 convert + quantize 件；字节数与上游 HF GGUF 一致，实测可用）；若选 VoiceDesign/Base，按 §6 注切换 talker |
| Codec | **`qwen-tokenizer-12hz-Q8_0.gguf`**（291,150,624 B） |
| 模型模式 | **主候选 `custom_voice`（`ryan`）**；若选 VoiceDesign/Base，按 §2.2 切换 talker 与模式（`voice_design` / `base`），不要混用 |
| 默认说话人 | **暂不冻结**（2026-11-24 主观试听否决 `vivian`）；候选优先级见 §2.2：VoiceDesign `cute_girl` / `young_female_gentle`、CustomVoice `ryan`、VoiceDesign `mature_male_deep`、Base x-vector |
| 方言预设 | `dylan`（北京话）、`eric`（四川话）仅在 `--lang chinese\|auto` 时触发；默认不用 |
| 语言 | `Chinese` |
| 音频 | `response_format=pcm`，s16le / 24 kHz / mono，流式 chunk（约 80 ms/帧，1920 samples） |
| 预缓冲 | **初始 300 ms**（04D 用真实 `AudioPlayer` 复测；04B 的 2500 ms 不适用于 Qwen） |
| 服务模式 | `external` 或 `managed`；`tts-server` 独立进程，主进程不加载权重 |
| 并发 | `tts-server` 默认 `max_batch=1`；Provider 侧显式排队，不假设可并发合成多段 |
| 取消 | 客户端断连（已验证真停）；不需要显式 cancel 端点 |

### 2.2 Profile 策略（多音色；2026-11-25 修订）

**默认路径：暂不冻结；`providers.tts.type=none` 保持安全默认。**

2026-11-24 的 29 条盲听否决了原默认 `vivian`。产品上允许多个 `VoiceProfile` 并存；用户可设置一个全局默认偏好，也可以暂不设置。未设置默认或当前 Profile 与已加载 Talker 不兼容时，语音进入明确的“未配置/不可用”状态，**不得静默回退到 `vivian`**。

**候选 Profile（可并存；复测顺序不是唯一选择）**：

| 复测顺序 | 候选 Profile | 评分（音色相似/自然度/情绪符合/可懂度/机械感） | 当前状态 |
| --- | --- | --- | --- |
| 1 | **VoiceDesign `cute_girl`**（可爱小女孩，音调高，轻快） | **5/5/5/5/1** | 女性/可爱候选；仅 1 条样本，必须补 10 文本并修复“头部吞字” |
| 2 | **CustomVoice `ryan`** | **4/4/5/5/1** | 男性候选；仅 1 条样本，需补测 |
| 3 | **VoiceDesign `young_female_gentle`**（年轻女性，温柔，偏低） | 4/4/4/5/2 | 稳健女性候选；有一点点电音感 |
| 4 | **VoiceDesign `mature_male_deep`**（成年男性，沉稳磁性） | 4/4/3/5/2 | 男性备选；接近真人但还不够 |
| 5 | **Base x-vector**（当前英文参考） | 5 条均值 4.00/3.80/N/A/4.60/2.00 | 自然度最高、但 T07 绕口令差；参考不是中文，需用户提供中文参考后复测 |
| — | **CustomVoice `vivian`** | 12 条均值 3.75/2.58/2.43/4.08/3.25 | **不作为默认**；A08/A16/A26 长文本尚有头/尾吞字，短句/中性/情绪被否 |
| — | `serena` / `uncle_fu` / `aiden` | 4/1/4/5、3/1/5/4、1/1/3/5（单条） | 当前不推荐；不作为候选默认 |
| — | VoiceDesign `news_anchor` / `stern_male` / `old_male_hoarse` | 1/1/2/3/5、1/1/1/3/5、2/2/2/5/4 | 电音/低沉/无起伏，拒绝 |
| — | CustomVoice 0.6B `vivian` | 3/1/2/3/4 | 拒绝 |
| — | Base ICL（EN 自动 ref_text） | 4/4/3/2/2 | 可懂度 2，暂不启用 |

> **这些 Profile 可同时存在，不是“五选一”。** “复测顺序”只表示建议先补哪几个的 10 文本样本；用户可以选择任意组合、设置任意默认偏好，甚至不设默认。
> 对 CustomVoice/VoiceDesign，`音色相似` 列按“音色是否符合期望/角色感”解读，不是克隆余弦。

**当前 `vivian` 的可用边界**：A08（T10）、A16（T01）、A26（T03）三条长文本尚可（自然度 3–5、机械感 1–2），但 A28（T11 中性）自然度 2/机械感 5、A01 slow/A06 sad/A13 gentle 等情绪全部差。**结论：`vivian` 只能作为“长文本候选”保留，不能作为默认，也不能用于情绪控制。**

**进阶路径（用户提供中文参考后）**：

```
mode = base
talker = qwen-talker-1.7b-base-Q8_0.gguf
voice = 用户注册的 ref_xvec 或 ref_icl
ref_wav = 5–15 s、24 kHz+、单声道、干净中文 WAV
ref_text = 人工逐字校对（ICL 必需）
```

- Base x-vector 是本轮自然度最好的组（5 条均值自然 3.80 / 机械 2.00 / 可懂 4.60），但 T07 绕口令 A15 掉到自然 2/可懂 3，且当前参考是英文；**必须用中文参考 + 10 文本复测**。
- ICL 单项 A04 音色 4/自然 4，但可懂度 2（开头英文残留/吐字不清）；**当前不可用**，只能等中文参考 + 人工校对 ref_text 后重测。

**可选路径（造音色）**：

```
mode = voice_design
talker = qwen-talker-1.7b-voicedesign-Q8_0.gguf
instructions = "<中文/英文音色描述>"
```

- VoiceDesign 呈明显双峰：`cute_girl` / `young_female_gentle` / `mature_male_deep` 可用，`news_anchor` / `stern_male` / `old_male_hoarse` 不可用；不能整体判“VoiceDesign 通过”，只能按描述逐条验收。

### 2.3 调度

| 项 | 冻结值 |
| --- | --- |
| 默认策略 | **`llm_priority`**：LLM 生成期间只缓冲/切分文本，不发起 TTS 合成 |
| `overlap` | 默认关闭；仅用户显式开启的实验模式，UI 必须显示"LLM Decode 约 −61%、TTS 块间隔 max 约 164 ms" |
| `sentence_interleave` | **unsupported**：llama.cpp 当前接口不能暂停/续接同一生成请求；只能 cancel + 重新 prefill，04E 不做生产实现 |
| 串行代价 | 337 token 回复 + 12 s 语音：首音频 2.7–3.0 s，整回合 3.8–4.1 s |
| 并发代价 | 1.7B + 9B：LLM 133.9→52.2 tok/s（−61.0%）；0.6B + 9B：141.1→57.6（−59.2%）；0.6B 不保护 LLM |
| 取消 | 新请求到达时断连旧 TTS；已验证 313 ms 内 GPU 降下来，下一请求 61.5 ms 可用 |

---

## 3. 为什么不选其他方案

| 方案 | 不选的原因 |
| --- | --- |
| **MOSS-TTS-Realtime** | 12 GB、RTF 1.24–1.95、长句 17/50 次欠载、ASR CER 36.6%；04A 已定位根因在模型侧，播放器无独立缺陷 |
| **Torch faster-qwen3-tts 1.7B** | 常驻 5.06 GB > 5 GB 峰值预算；TTFA 302–327 ms；RTF 0.273–0.287；只适合做 Base instruct 对照 |
| **Torch 0.6B** | 3.58/3.74 GB，TTFA 257–286 ms、RTF 0.23；纯 pip 不编译是优点，但性能比 GGML 差一个量级，且需自建 cancel |
| **GGML 0.6B Base** | 2.55/2.61 GB，比 1.7B CV 只省约 0.5 GB；音质/指令跟随弱于 1.7B；仅低显存 fallback |
| **GGML 0.6B CustomVoice** | instruct 可用（客观），但主观 A29 = 3/1/2/3/4，“明显异常”；拒绝 |
| **CV 1.7B `vivian` 作为默认** | 主观 12 条均值自然 2.58/机械 3.25；短句中性 A28 = 2/5、7 条情绪中 6 条差（`fast` 仅勉强）；**NO-GO 默认** |
| **CV 1.7B `serena` / `uncle_fu` / `aiden`** | 单条主观：4/1/4/5、3/1/5/4、1/1/3/5；自然度 1/机械 4–5，拒绝 |
| **Base + instruct（GGML）** | **不可用**：HTTP 400 `--instruct is not supported for base models`；Provider 必须拒绝或禁用该字段 |
| **Base ICL + 英文自动 ref_text** | 主观 A04：音色 4/自然 4，但可懂度 2；客观 CER 4.8–14.3%、出现“Really”英文起头；拒绝 |
| **VoiceDesign 作为整体默认** | 主观双峰：3 条可用、3 条差；只能按描述逐条验收，不能整体默认 |

---

## 4. 遗留风险

| 风险 | 证据 | 缓解 |
| --- | --- | --- |
| **CustomVoice instruct 不稳定** | 客观：`ryan + angry` F0 95→233–268、`ryan + slow` 16.72 s；**主观：vivian 7 条中 6 条差，fast 仅勉强**（neutral 2/5、gentle 1/5、happy 2/4、sad 1/5、angry 1/3、slow 3/5、fast 4/2） | 生产禁用全部情绪指令；最多把 `fast` 留作 experimental 候选，先补测 |
| Base instruct 不可用于 GGML | HTTP 400 `--instruct is not supported for base models` | Provider 层拒绝 `instructions`；不要用提示词包装 |
| 默认 `vivian` 被主观否决 | A28 中性 3/2/2/4/5；A01 slow 4/3/3/4/5；A06 sad 4/1/1/3/5；A13 gentle 4/1/3/4/5 | 不设默认；长文本 A08/A16/A26 仅作候选，需 10 文本复测 |
| 可用候选只有单样本 | `ryan` A20、VD `cute_girl` A05、VD `young_female_gentle` A12、VD `mature_male_deep` A18 均只有 1 条 | 选定前必须补 10 文本（T01/T03/T04/T07/T10 + 5 条情绪/长句）复测 |
| Base T07 绕口令弱 | A15 自然 2/可懂 3/机械 3；A04 ICL 可懂 2 | Base 只用于普通文本；T07 单独复测；ICL 禁用 |
| 设备/环境记录缺失 | `listening-sheet-04c.md` §1 耳机/播放器/房间未记录 | 不影响分数有效性，但复测时补齐；同设备同音量 |
| 中文参考/ICL 未验收 | 无用户中文参考；A04 可懂度 2 | 按 `docs/recording-guide-zh.md` 录制 + 人工校对；用 `tools/ref_audio_check.py` 体检 |
| 0.6B CV 官方口径矛盾 | 官方总表留空 vs 卡片示例用 instruct | 客观可用，但主观 A29 3/1/2/3/4 拒绝；只作历史对照 |
| Windows 转换 locale 坑 | `convert.py` 用 `Path.read_text()` 默认 GBK，UTF-8 vocab 被误读 → BPE 乱码、`[Prompt] FATAL: prompt too short`、502 | **必须 `python -X utf8` / `PYTHONUTF8=1` 运行 convert.py**；已重转并验证 token 0–255 与 Base GGUF 一致 |
| 并发时 TTS 块间隔抖动 | 长重叠 max 164 ms | 300 ms 预缓冲；04D 用真实播放器复测；不要宣称完全无抖动 |
| WDDM 进程级显存不可用 | `nvidia-smi --query-compute-apps used_memory` 全 `[N/A]` | 用整卡相位差；文档已注明，不把 `[N/A]` 当 0 |
| 服务串行 `max_batch=1` | `tts-server` 默认单请求 | Provider 显式排队/忙碌状态；不要并发多段 |

---

## 5. 需要用户本人完成的事项

1. **盲听已完成（2026-11-24）**：29/29 条已评分并解码；结果见 §9。**不再要求从候选中选唯一音色**。
2. **建立多个 VoiceProfile（可并存）**：
   - 将 `cute_girl`、`ryan`、`young_female_gentle`、`mature_male_deep` 等建成独立 Profile；Base x-vector/ICL 也作为独立 Profile；
   - 用户可设置一个全局默认偏好，也可以暂不设置；还可以按会话/单次回复覆盖；
   - 每个 Profile 独立补 10 文本复测；不是“五选一”。
3. **导入用户个人参考音频（Pending）**：按 `docs/recording-guide-zh.md` 提供 5–15 s、24 kHz+、单声道、干净 WAV；ICL 另需人工逐字校对 `ref_text`，跑 `tools/ref_audio_check.py`；当前仓库中没有找到用户中文参考，不能假定已存在。
4. **情绪指令**：7 条 vivian 情绪中 6 条不合格、`fast` 仅“勉强”；建议全部标 experimental/默认禁用。若必须做情绪，先在选定的单个 Profile 上按指令 ≥20 次抽检；不得因为换了音色就自动继承旧结论。
5. **默认是否切换**：在任何 Profile 完成 10 文本复测前，`providers.tts.type` 保持 `none`；不要静默改默认。未设置默认 Profile 时，语音进入“未配置”状态（文字仍可用），而不是回退到 `vivian`。

---

## 6. 04D 接入配置（音色待定，可直接抄）

```yaml
# 建议写入 04D 的 Provider 配置（具体 YAML 结构按主项目现有 config 风格）
providers:
  tts:
    type: none                 # 安全默认；用户选定音色并复测后才显式改 qwen3_tts
    base_url: http://127.0.0.1:8095
    voice: <user-selected>     # 04F 前仍是单 voice；04F 改为 voice_profile_id，见 voice-profile-design-04c.md
    # 04F 目标：voice_profile_id: <profile-id>；当前 voice 仅为兼容字段
    language: Chinese
    response_format: pcm
    # 初始实验值，04D 用真实 AudioPlayer 复测后再冻结
    start_prebuffer_ms: 300
  qwen3_tts:
    mode: external             # 或 managed
    commit: 2edb35f
    talker: <项目外路径>/qwen-talker-1.7b-customvoice-Q8_0.gguf
    codec:  <项目外路径>/qwen-tokenizer-12hz-Q8_0.gguf
    alias: qwen3-tts
    max_batch: 1
    instruct: disabled         # 6/7 CV 情绪不合格，fast 仅勉强；不要默认透传
speech:
  scheduling: llm_priority     # overlap 仅显式实验；sentence_interleave unsupported
```

> 选 `VoiceDesign` 时 talker 换成 `qwen-talker-1.7b-voicedesign-Q8_0.gguf`，`voice` 留空、`instructions` 填描述；选 Base 时 talker 换成 `qwen-talker-1.7b-base-Q8_0.gguf` 并注册用户参考音色。

启动命令（external 模式，模型/DLL 路径按实际部署调整）：

```powershell
tts-server.exe `
  --model  <models>\qwen-talker-1.7b-customvoice-Q8_0.gguf `
  --codec  <models>\qwen-tokenizer-12hz-Q8_0.gguf `
  --alias  qwen3-tts --host 127.0.0.1 --port 8095 `
  --lang   Chinese --max-batch 1
```

---

## 7. 对主项目状态文档的更新建议

> 以下内容供合并到 `F:\AI_Project_Development\docs\IMPLEMENTATION_STATUS.md`，本轮**没有修改主项目**。

```markdown
### Phase 04C 收尾：多音色 VoiceProfile（2026-11-25，仅文档，未改主项目代码）

- 29/29 条匿名试听分数保留；`vivian` 默认和当前 CustomVoice 情绪指令不通过，不能写“音质/情绪验收通过”。
- 04C **不再要求选定唯一默认音色**：按 `reference_clone` / `builtin_speaker` / `designed_voice` 建立多个
  VoiceProfile；默认音色只是用户偏好，支持全局/会话/单次回复三级选择。
- qwentts.cpp `2edb35f` 实测：一个 `tts-server` 固定加载一个 Talker；Base 可在同一进程注册多个克隆 voice
  （内存注册表，重启丢失，需要应用侧重新注册）；CustomVoice 仅当前 9 个 speaker；VoiceDesign 用固定描述；
  Base/CustomVoice/VoiceDesign 不能在同一已加载实例中无重启切换。
- 当前候选：VD `cute_girl`、CV `ryan`、VD `young_female_gentle`/`mature_male_deep`、Base x-vector
  均为**候选/待复测 Profile**，不是“唯一赢家”；单样本评分不能写成全面通过。
- 用户中文参考音频/人工校对 ref_text 仍未提供；ICL 保持 Pending/No-Go；Base instruct 不可用。
- 当前 `providers.tts.type` 保持 `none`；未设置默认 Profile 时语音进入“未配置”状态，禁止回退 `vivian`。
- 04F 实现 Voice Library / VoiceProfile CRUD、导入与资产生命周期、Profile 兼容标记、三级选择、
  与 04E Runtime 的模型切换协作；本次未改生产代码。
- 详情：`F:\AI_Project\Qwen3-TTS-Experiments\docs\voice-profile-design-04c.md`、
  `qwen3-tts-decision-04c.md` §9。
```

---

## 8. 证据索引

| 内容 | 路径 |
| --- | --- |
| 音色/参考/克隆 | `docs/qwen3-tts-voice-quality.md` |
| 情绪/CustomVoice/VoiceDesign | `docs/qwen3-tts-emotion.md` |
| 并发/串行/显存/取消 | `docs/qwen3-tts-concurrency.md` |
| 原始探测 | `benchmarks/04c/**`（每个 case 有 probe JSON + WAV） |
| 匿名试听 | `samples/04c-listening/A01…A29.wav`、`docs/listening-key-04c.md`、`docs/listening-sheet-04c.md` §10 |
| 试听解码/汇总 | `benchmarks/04c/listening-scores.json`、`benchmarks/04c/listening-summary.md`、`tools/analyze_listening_04c.py` |
| 多音色 Profile 设计 | `docs/voice-profile-design-04c.md` |
| 契约拒绝 | `benchmarks/04c/contract-base.json`、`contract-cv.json`、`contract-vd.json` |
| 模型家族/源码依据 | `research/04c-model-families.md` |

---

## 9. 人工试听回填与多 Profile 判定（2026-11-24 / 2026-11-25）

### 9.1 评分完整性

| 检查项 | 结果 |
| --- | --- |
| 行数 | **29/29**（A01–A29） |
| 解码 | 29/29 在 `listening-key-04c.md` 找到对应样本 |
| 五个评分列 | 29/29 全部有值；无缺分 |
| 取值范围 | 全部 1–5，无越界（本轮没有 0.5 分） |
| 备注 | 29/29 有备注 |
| 环境记录 | §1 已补“日期 2026-11-24 / 试听人哥哥本人 / 音频目录”，**耳机、播放器、房间仍未记录**（不影响分数有效性，但复测时应补齐） |

**一处协议偏差（不改分数）**：模板规定“文本没有情绪要求时，情绪符合记 N/A”。本轮 16 条无情绪指令的样本（Base 5、ICL 1、CV 普通文本 5、CV speaker sweep 4、0.6B 1）也填了 1–5。汇总时**没有把这些行的情绪分计入情绪指令结论**；它们只作为“整体语气”参考。VoiceDesign 的“情绪符合”按“描述/风格符合”解读，不是情绪指令。

> 样本量提醒：每个情绪指令/每个 VoiceDesign 描述基本只有 1 条。下面所有“好/差”都是**单样本听感**，不是统计显著性结论。

### 9.2 模型级汇总（均值；机械感越低越好）

| 模型 | n | 音色相似 | 自然度 | 情绪符合 | 可懂度 | 机械感 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Base GGML 1.7B（x-vector，英文参考） | 5 | 4.00 | 3.80 | N/A | 4.60 | **2.00** |
| Base GGML 1.7B（ICL，EN 自动 ref_text） | 1 | 4.00 | 4.00 | N/A | **2.00** | 2.00 |
| CustomVoice GGML 1.7B（全部） | 16 | 3.56 | **2.38** | 2.43（7 条） | 4.12 | **3.38** |
| CustomVoice GGML 1.7B `ryan`（单项） | 1 | 4.00 | 4.00 | 5.00 | 5.00 | **1.00** |
| CustomVoice GGML 1.7B `vivian`（12 条） | 12 | 3.75 | 2.58 | 2.43（7 条） | 4.08 | 3.25 |
| CustomVoice GGML 0.6B `vivian`（单项） | 1 | 3.00 | **1.00** | 2.00 | 3.00 | **4.00** |
| VoiceDesign GGML 1.7B（全部） | 6 | 2.83 | 2.83 | 2.83（6 条） | 4.33 | 3.17 |

**读法**：Base x-vector 是本轮自然度/机械感最好的组；CustomVoice 1.7B 整体被 `vivian` 的情绪/短句拖到 2.38/3.38；VoiceDesign 整体均值被 3 条差样本拉低，不能代表全部描述。

### 9.3 CV 1.7B 说话人

| speaker | n | 音色相似 | 自然度 | 可懂度 | 机械感 | 主观备注 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| `vivian` | 12 | 3.75 | 2.58 | 4.08 | 3.25 | 长文本 A08/A16/A26 尚可；短句中性/情绪差 |
| `ryan` | 1 | 4.00 | 4.00 | 5.00 | **1.00** | A20 “整体不错”；唯一强项 |
| `serena` | 1 | 4.00 | **1.00** | 4.00 | **5.00** | A25 “情绪异常” |
| `uncle_fu` | 1 | 3.00 | **1.00** | 5.00 | 4.00 | A02 “语调和感情有问题，语速略慢” |
| `aiden` | 1 | **1.00** | **1.00** | 3.00 | **5.00** | A17 “吐字不清晰” |

### 9.4 CV 1.7B 情绪/语气指令（`vivian`，每条 1 样本）

| 指令 | 音色相似 | 自然度 | 情绪符合 | 可懂度 | 机械感 | 判定 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| neutral | 3 | 2 | 2 | 4 | **5** | ❌ 基础中性也被否 |
| gentle | 4 | **1** | 3 | 4 | **5** | ❌ “机器感极重” |
| happy | 4 | 2 | 3 | 5 | 4 | ❌ “有点生硬” |
| sad | 4 | **1** | **1** | 3 | **5** | ❌ “明显不像人类” |
| angry | 3 | **1** | 2 | 4 | 3 | ❌ “情绪明显异常” |
| slow | 4 | 3 | 3 | 4 | **5** | ❌ “语速太慢，十分奇怪” |
| fast | 4 | 4 | 3 | 3 | 2 | ⚠️ “整体勉强还行”；只可 experimental |

### 9.5 VoiceDesign 描述（每条 1 样本）

| 描述 | 音色相似 | 自然度 | 描述/情绪符合 | 可懂度 | 机械感 | 判定 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| 可爱的小女孩，音调高，轻快 | **5** | **5** | **5** | **5** | **1** | ✅ 最强候选；头部吞字 |
| 年轻女性，温柔，偏低 | 4 | 4 | 4 | 5 | 2 | ✅ 稳健候选；一点点电音感 |
| 成年男性，沉稳，磁性 | 4 | 4 | 3 | 5 | 2 | ✅ 男性备选；接近真人但还不够 |
| 老年男性，沙哑、缓慢 | 2 | 2 | 2 | 5 | 4 | ❌ 过于低沉/无起伏 |
| 成熟女性，新闻播报 | 1 | 1 | 2 | 3 | **5** | ❌ 电音感极重 |
| 严肃男性，低音，命令式 | 1 | 1 | 1 | 3 | **5** | ❌ 语调过低 |

### 9.6 候选 Profile 目录（可并存，不是唯一赢家）

**按哥哥的评分，`vivian` 不再作为默认。** 以下 Profile 都可以保留在 Voice Library 中，分别复测；**不是五选一**：

| 复测顺序 | Profile | 评分 | 当前状态 / 下一步 |
| --- | --- | --- | --- |
| 1 | VoiceDesign `cute_girl` | 5/5/5/5/1 | 女性/可爱候选；仅 1 条，头部吞字；补 10 文本 |
| 2 | CustomVoice `ryan` | 4/4/5/5/1 | 男性候选；仅 1 条；补 10 文本 |
| 3 | VoiceDesign `young_female_gentle` | 4/4/4/5/2 | 稳健女性候选；补 10 文本 |
| 4 | VoiceDesign `mature_male_deep` | 4/4/3/5/2 | 男性备选；补 10 文本 |
| 5 | Base x-vector | 5 条均值 4.00/3.80/N/A/4.60/2.00 | 自然度最高；需中文参考 + T07 复测 |
| — | `vivian` | 12 条均值 3.75/2.58/4.08/3.25 | 长文本候选；不作默认，不可用于情绪 |
| — | `serena`/`uncle_fu`/`aiden`/0.6B/ICL/VD news-stern-old | 见上文 | 当前不建默认 Profile；可作为历史/对照 |

> 对 CV/VD，`音色相似` 是“是否符合期望/角色感”，不是克隆余弦；不能拿它和 Base 的克隆余弦直接横比。

### 9.7 三类路径的 Profile 适用性

| Profile 类型 | 判定 | 依据与条件 |
| --- | --- | --- |
| **reference_clone（x-vector）** | **条件适用** | 自然度 3.80/机械 2.00/可懂 4.60，普通文本 A03/A22/A23/A24 好；T07 A15 弱；当前英文参考不是最终中文音色。需中文 5–15 s 参考 + 10 文本复测。 |
| **reference_clone（ICL）** | **Pending/暂不适用** | A04 音色 4/自然 4，但可懂度 **2**；自动英文 ref_text 导致开头残留/漏字。必须中文参考 + 人工逐字 `ref_text`，当前不得启用。 |
| **builtin_speaker** | **仅 `ryan` 条件候选，`vivian` 不适用** | `ryan` 4/4/5/5/1（单样本）；`vivian` 12 条均值 3.75/2.58/4.08/3.25，短句/中性/情绪差；`serena`/`uncle_fu`/`aiden` 均差。不能整体说“CustomVoice 通过”。 |
| **designed_voice** | **按描述条件适用** | 6 条中 3 条好（cute/young female/mature male）、3 条差（news/stern/old）；不能整体默认，只能逐描述验收。 |

### 9.8 情绪指令的正式可用性

| 指令/能力 | 结论 |
| --- | --- |
| Base instruct（GGML） | ❌ 不可用（HTTP 400） |
| CV vivian: neutral/gentle/happy/sad/angry/slow | ❌ **全部不得正式使用**；主观评分均差 |
| CV vivian: `fast` | ⚠️ **仅 experimental**；4/4/3/3/2，可懂度 3，需重测 |
| CV ryan 的情绪 | 未测；不要因为 `ryan` 基础音色好就推断情绪可用 |
| VoiceDesign 描述 | 不是“情绪控制”；`cute_girl`/`young_female_gentle`/`mature_male_deep` 可作为音色描述，但 `news/stern/old` 不可用 |
| 正式建议 | 默认 `instructions` **禁用**；UI 若保留，只能标 experimental，并且只允许经复测的单一指令（当前没有） |

### 9.9 最终决策

**总判定：`CONDITIONAL GO`（整体）/ 当前默认 `vivian` 与全部正式情绪指令 `NO-GO`。**

- **为什么不是 GO**：`vivian` 默认被否；当前没有完成 10 文本复测的 Profile；用户中文参考/ICL 未验收；情绪全部 experimental。
- **为什么不是 NO-GO（整体）**：技术、显存、并发、取消、稳定性都过；29 条评分给出了多个可继续的候选 Profile（VD `cute_girl`、CV `ryan`、VD `young_female_gentle`/`mature_male_deep`、Base x-vector）。
- **转 GO 的条件（按 Profile 分别判定）**：
  1. 建立 Voice Library，把候选建成**多个** Profile；默认可选，不要求唯一；
  2. 每个 Profile 用同一套 5 维量表补 10 文本（T01/T03/T04/T07/T10 + 5 条长句/情绪）复测，至少 3 条/类；
  3. 情绪指令默认禁用；若以后启用，只在单个 Profile 上按指令 ≥20 次抽检 + 人耳；
  4. Base/ICL 只接受用户中文参考 + 人工校对 `ref_text`；
  5. 在以上完成前，`providers.tts.type` 保持 `none`；未设置默认 Profile 时语音“未配置”，不得回退 `vivian`。
