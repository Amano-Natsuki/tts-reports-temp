# Qwen3-TTS 情绪/风格指令与 CustomVoice/VoiceDesign 可用性（04C）

> 任务书：`04C_任务书_Qwen音质情绪与GPU调度验收.md`（§3.2、§3.3、§5 情绪部分）
> 环境：Windows 10 19045 / RTX 4090 24 GB；qwentts.cpp `2edb35f`；Torch 对照为 `faster-qwen3-tts 0.5.4` + `transformers 5.15.1`
> 原始数据：`benchmarks/04c/emotion-torch17b/`、`customvoice-17b/`、`customvoice-06b/`、`voicedesign-17b/`、`emotion-metrics-*.json`、`contract-*.json`
> **状态：客观代理指标完成；29 条人工试听已完成（2026-11-24）。结果：`vivian` 的 7 条情绪/语气中 **6 条不合格**，`fast` 仅“勉强”；全部情绪指令均不得正式使用。详见 `qwen3-tts-decision-04c.md` §9。本文不把“模型有反应”写成“情绪稳定可控”。**

---

## 1. 结论先行

1. **GGML Base 完全不接受 instruct**：`POST /v1/audio/speech` 带 `instructions` → **HTTP 400**，错误原文 `--instruct is not supported for base models`（`benchmarks/04c/contract-base.json`）。所以"Base 情绪指令"在**生产 GGML 后端上不是一个可用能力**；04B 报告里的 Base instruct 数据是 Torch 后端的结果。
2. **Torch 1.7B Base 的 instruct 比 04B 的 0.6B 更"有反应"**：同一句中文、8 组指令 × 3 seed，`slow` 时长 +24.4%、`fast` −22.2%；`happy`/`angry` 的 F0 中位分别 +14.2%/+12.0%，`gentle` 的 voiced ratio 从 0.68 升到 0.85。**但 `sad` 几乎只改停顿/音高分布，不改时长；不能仅凭时长断言"情绪成功"。**
3. **CustomVoice 1.7B 是 GGML 上唯一有 instruct 的路径（但主观不合格）**：`vivian` 上 `fast` 时长 −25%、`slow` +102%、`angry` F0 +48%；`ryan` 上 `angry` F0 从 95 Hz 跳到 233–268 Hz、`slow` 出现过 16.72 s 的重复/拖长异常。**响应真实，但主观试听（2026-11-24）6/7 不合格，`fast` 仅“勉强”，不能称为“稳定情绪控制”**；全部指令标 experimental。
4. **0.6B CustomVoice 也能听 instruct**（官方总表把 0.6B 的 Instruction Control 留空，实测是能用的）：`fast` −28%、`angry` F0 +31%、`slow` +7%；但只测了 2 seed/1 speaker，且音色/自然度弱于 1.7B，只能作低显存备选。
5. **VoiceDesign 1.7B 的描述→音色映射明显**：中文/英文描述都能让 F0 落在 69–389 Hz 的不同区间（详见 `qwen3-tts-voice-quality.md` §4.2）。这是“无参考音频造音色”的可行路径；主观试听 3/6 描述可用（cute/young female/mature male），3/6 不可用。
6. **没有任何一组可以写“情绪验收通过”**：29 条匿名样本已由哥哥本人盲听（2026-11-24），`vivian` 的 7 条情绪/语气中 6 条不合格，`fast` 仅“勉强”；详见 §6。

---

## 2. 关键契约：Base 不能 instruct（GGML）

| 请求 | HTTP | 响应 |
| --- | ---: | --- |
| Base 1.7B + `instructions` | **400** | `{"error":{"message":"--instruct is not supported for base models","type":"server_error"}}` |
| Base 1.7B + `voice=vivian`（未注册） | 400 | `unknown voice 'vivian'`（Base 只有注册的克隆音色，没有 speaker 表） |
| CustomVoice 1.7B 缺 `voice` | 400 | `custom_voice models require --speaker` |
| VoiceDesign 1.7B 缺 `instructions` | 400 | `voice_design models require --instruct` |
| 非 Base 注册 voice | 400 | `voice registration is only valid for base models (loaded: custom_voice/voice_design)` |

原始：`benchmarks/04c/contract-base.json`、`contract-cv.json`、`contract-vd.json`。

**含义**：04D/04E 若要在生产 GGML 路径上表达情绪，**只能选 CustomVoice 或 VoiceDesign**；Base 的 `instructions` 字段必须被 Provider 拒绝或在配置层禁用，不能透传后静默失败。

---

## 3. Torch 1.7B Base：8 指令 × 3 seed

- 服务：`servers/qwen3_tts_server.py` + `faster-qwen3-tts 0.5.4`，1.7B Base bf16，xvec-only，英文参考；`--chunk-size 12`。
- 文本：T11“今天的工作已经完成了，我们可以一起休息一会儿。”
- seed：42 / 123 / 2026；温度/采样用服务默认；`instruct` 走 Torch 服务的 `instruct` 字段。
- 24/24 成功，0 欠载；TTFA 288–332 ms，RTF 0.265–0.295（Torch 1.7B 的已知水平）。
- 指标：24 kHz 分析域，10 ms 帧；F0 自相关 60–600 Hz（`tools/emotion_metrics.py`）。

| 指令 | n | 时长 ms | F0 中位 Hz | F0 p10–p90 | 语速代理/s | 停顿数 | 停顿总 ms | RMS dBFS | voiced |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 无（neutral） | 3 | 3600 | 178.5 | 151–247 | 5.25 | 2 | 560 | −22.1 | 0.68 |
| gentle（温柔轻柔） | 3 | 3200 | 183.3 | 159–253 | 5.62 | 0 | 0 | −22.7 | **0.85** |
| happy（开心愉快） | 3 | 3200 | **203.8** | 168–270 | 5.68 | 1 | 200 | −21.8 | 0.73 |
| playful（俏皮活泼） | 3 | 3280 | 180.8 | 160–250 | 5.79 | 1 | 240 | −22.2 | 0.79 |
| sad（难过低沉） | 3 | 3520 | 180.8 | 150–223 | 5.68 | 1 | 210 | −21.6 | 0.76 |
| angry（生气严厉） | 3 | 3760 | **200.0** | 168–245 | **4.69** | **3** | **870** | −23.4 | 0.66 |
| slow（非常缓慢） | 3 | **4480** | 176.6 | 152–218 | 5.50 | 2 | 780 | −23.6 | 0.68 |
| fast（非常快） | 3 | **2800** | 178.8 | 153–225 | **6.79** | 0 | 0 | −22.2 | 0.84 |

相对 neutral：

| 指令 | 时长变化 | F0 中位变化 | 语速代理变化 | 读法 |
| --- | ---: | ---: | ---: | --- |
| gentle | −11.1% | +2.7% | +7.0% | 更连、更多 voiced，像"放软" |
| happy | −11.1% | **+14.2%** | +8.2% | 音高上移明显，不只是加快 |
| playful | −8.9% | +1.3% | +10.3% | 主要是节奏变快 |
| sad | −2.2% | +1.3% | +8.2% | **证据最弱**：只看到 p90 变低/停顿略多 |
| angry | +4.4% | **+12.0%** | −10.7% | 音高上移 + 变慢 + 停顿变多，像"压着说" |
| slow | **+24.4%** | −1.1% | +4.8% | **"慢"在 1.7B Torch 上有效**（04B 0.6B 是 −1.3%） |
| fast | **−22.2%** | +0.2% | **+29.3%** | 主要是语速，音高几乎不变 |

**判定**：

- 04B 的"快有效、慢几乎无效"是 **0.6B Torch** 的结论；本轮 **1.7B Torch 的 `slow` 有效（+24.4%）**，说明模型大小/版本会影响指令跟随，不能把 0.6B 的结论外推到 1.7B。
- `happy`/`angry` 不只是时长变化，F0 也有 12–14% 的位移；`sad` 最弱，需要人耳确认是否"难过"。
- **生产后端 GGML Base 不能 instruct**，所以这些 Torch 数据只用于回答"Base 指令到底稳不稳"，不能直接当 04D 配置。

---

## 4. CustomVoice 1.7B：预设说话人 + instruct

### 4.1 说话人清单与客观音高

`GET /v1/audio/voices` 实测 9 个 speaker：`serena, vivian, uncle_fu, ryan, aiden, ono_anna, sohee, eric, dylan`（与官方 `spk_id` 表一致）。T11 的 F0/时长见 `qwen3-tts-voice-quality.md` §4.1；男/女声分离清楚（女声 226–278 Hz，男声 95–167 Hz，日语/韩语女声 241–244 Hz）。

### 4.2 vivian：6 指令 × 2 seed

| 指令 | 时长 ms（42 / 2026） | F0 中位（42 / 2026） | 语速代理（42 / 2026） | 停顿数 | RMS |
| --- | ---: | ---: | ---: | ---: | ---: |
| neutral | 3840 | 225.6 | 4.95 | 1 | −20.6 |
| gentle | 5120 / 4320 | 188.8 / 219.4 | 5.27 / 5.32 | 1 / 1 | −19.4 / −18.2 |
| happy | 3840 / 4960 | 242.8 / 240.4 | 4.69 / 5.24 | 1 / 1 | −20.7 / −20.3 |
| sad | **7360 / 7600** | 193.4 / 250.7 | 4.48 / 3.42 | 2 / 4 | −24.5 / −23.6 |
| angry | 3840 / 4720 | 245.1 / **333.6** | 4.95 / 5.30 | 1 / 1 | −18.8 / −18.0 |
| slow | **7760 / 8480** | 206.1 / 187.9 | 3.99 / 4.13 | 2 / 4 | −19.8 / −23.2 |
| fast | 3840 / 2880 | 209.0 / 292.6 | 4.95 / 5.90 | 1 / 0 | −18.7 / −19.1 |

读法：

- `fast` 相对 neutral 时长 −25%（seed 2026 更强），`slow` +102–121%，`sad` +92–98% 且停顿增多；**instruct 对韵律的影响比 Base 更明显**。
- `angry` 的 F0 从 226 升到 245–334，音量也更高（RMS 高 2–3 dB）；但 seed 间差异较大（245 vs 334），**不能保证每次都是同一种"生气"**。
- `sad` 用大幅放慢 + 增加停顿来表达，听感可能更像"拖"而不是"难过"，必须人耳判断。

### 4.3 ryan：同矩阵的稳定性问题

| 指令 | 时长 ms（42 / 2026） | F0 中位（42 / 2026） | 备注 |
| --- | ---: | ---: | --- |
| neutral | 4080 | 95.4 | 低音男声 |
| gentle | 4720 / 6560 | 102.7 / 104.9 | 尚可 |
| happy | 3360 / 4560 | 138.9 / 103.9 | F0 不稳定 |
| sad | 6720 / 6400 | 106.7 / 128.2 | 放慢 |
| angry | 4480 / 3200 | **268.2 / 233.4** | F0 从 95 跳到 233–268，像换了个人/假声；**风险** |
| slow | **16720** / 6640 | 102.7 / 116.6 | 16.72 s 异常长，疑似重复/拖长；**风险** |
| fast | 3040 / 4640 | 151.6 / 105.2 | 不稳 |

**判定**：CustomVoice 的 instruct 不是"选个 speaker + 写句话就稳定"；`ryan + angry/slow` 已有明显不稳定证据（音色漂移/超长）。若要用于生产，建议先固定 `vivian`/`serena` 等中文母语 speaker，并对每条指令做 ≥20 次稳定性抽检 + 人耳确认。

### 4.4 0.6B CustomVoice：能听 instruct，但只作低显存备选

官方总表把 0.6B-CustomVoice 的 Instruction Control 留空，但官方卡片正文/示例又用了 instruct，口径矛盾。实测（`vivian`，2 seed）：

| 指令 | 时长 ms | F0 中位 | 语速代理 | 相对 neutral |
| --- | ---: | ---: | ---: | --- |
| neutral | 4560 | 231.7 | 4.39 | — |
| fast | 3280 | 265.9 | 5.18 | 时长 −28.1%，F0 +14.8% |
| slow | 4880 | 242.9 | 3.89 | 时长 +7.0%，F0 +4.8% |
| angry | 4960 | 304.4 | 4.44 | 时长 +8.8%，F0 **+31.4%** |

**结论**：0.6B CV 在 GGML 上**确实接受并响应 instruct**（回答了官方口径矛盾），但慢速/情绪幅度与稳定性不如 1.7B；17 条简单句 CER 0%，TTFA 14–37 ms，RTF 0.077–0.083，零欠载。只建议在显存不足时作为 fallback。

---

## 5. VoiceDesign 1.7B：描述生成音色

18 次（8 组中文描述 ×2 seed + 2 组英文描述 ×1），全部成功；TTFA 15–41 ms，RTF 0.093–0.099，零欠载；简单句 CER 0%。F0 表见 `qwen3-tts-voice-quality.md` §4.2：

- 中文描述能把 F0 放到 69–389 Hz：`严肃男性` 69–73、`老年男性` 81–89、`成年男性` 88–107、`年轻女性/新闻主播` 183–226、`活泼少女/可爱女孩` 352–389。
- **英文描述也能控制中文输出**（young female 198.5、mature male 82.0），说明 instruct 的语言与输出语言可分离。
- 风险：同描述不同 seed 的 F0 可能差 10–40 Hz（如 mature male 88 vs 107、cute girl 387 vs 389 较稳），音色一致性需要试听；描述里的"温柔/知性"等主观词是否被正确执行，机器指标判不了。

**定位**：VoiceDesign 是"没有用户参考音频时，按描述造一个候选音色"的可行路径；它不是克隆，不能声称"像某个真人"。

---

## 6. 人工试听结果（2026-11-24）

**7 条 vivian 情绪/语气中 6 条不合格，`fast` 仅“勉强”**，详见 `qwen3-tts-decision-04c.md` §9：

| 指令 | 音色相似/自然度/情绪符合/可懂度/机械感 | 结论 |
| --- | --- | --- |
| neutral | 3/2/2/4/5 | ❌ 基础中性也被否 |
| gentle | 4/1/3/4/5 | ❌ 机器感极重 |
| happy | 4/2/3/5/4 | ❌ 有点生硬 |
| sad | 4/1/1/3/5 | ❌ 明显不像人类 |
| angry | 3/1/2/4/3 | ❌ 情绪明显异常 |
| slow | 4/3/3/4/5 | ❌ 语速太慢、奇怪 |
| fast | 4/4/3/3/2 | ⚠️ 仅“勉强”，只能 experimental |

**VoiceDesign 6 条描述**：`cute_girl` 5/5/5/5/1、`young_female_gentle` 4/4/4/5/2、`mature_male_deep` 4/4/3/5/2 可用；`news_anchor`/`stern_male`/`old_male_hoarse` 不可用。描述不是情绪指令，不能当情绪控制。

**结论**：Base instruct（GGML）不可用；CustomVoice `vivian` 没有可正式启用的情绪指令；`fast` 只能作为 experimental 候选重测。若需要稳定情绪，必须先换主观合格音色并做 ≥20 次/指令抽检。

---

## 7. 未完成项与风险

| 项 | 状态 | 说明 |
| --- | --- | --- |
| 人工试听（情绪/自然度） | **✅ 已完成（2026-11-24）** | 6/7 不合格，`fast` 仅“勉强”；不能签“情绪验收通过” |
| Base instruct on GGML | **不可用** | 400；Provider 必须拒绝或禁用 |
| CustomVoice 20 次/指令稳定性 | 未做 | 本轮每指令 2 seed；ryan 已暴露漂移 |
| 0.6B CV 的情绪矩阵 | 只做 1 speaker × 3 指令 ×2 seed | 不足以推荐为情绪路径 |
| Torch 1.7B Base 的 10 组质量 | 未做 | 仅用于 instruct 对照；不作为生产候选 |

---

## 8. 复现命令（节选）

```powershell
# Torch 1.7B Base（仅用于 Base instruct 对照；显存 5.06 GB，超 04D 峰值预算）
powershell -File run\start_torch_server.ps1 -Model F:\AI_Project\Qwen3-TTS-Experiments\models\Qwen3-TTS-12Hz-1.7B-Base -Port 8093
python harness\04c_run.py run --url http://127.0.0.1:8093 --cases scenarios\04c_emotion_base_torch17b.json --out-dir benchmarks\04c\emotion-torch17b

# GGML CustomVoice 1.7B / 0.6B
python harness\04c_run.py run --url http://127.0.0.1:8096 --cases scenarios\04c_customvoice_17b.json --out-dir benchmarks\04c\customvoice-17b
python harness\04c_run.py run --url http://127.0.0.1:8097 --cases scenarios\04c_customvoice_06b.json --out-dir benchmarks\04c\customvoice-06b

# VoiceDesign 1.7B
python harness\04c_run.py run --url http://127.0.0.1:8098 --cases scenarios\04c_voicedesign_17b.json --out-dir benchmarks\04c\voicedesign-17b

# 客观指标
python tools\emotion_metrics.py benchmarks\04c\customvoice-17b --out benchmarks\04c\emotion-metrics-cv17b.json
```
