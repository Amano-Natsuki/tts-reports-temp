# IMPLEMENTATION_STATUS 更新建议（04C 多音色收尾，2026-11-25）

> 本文件只是**可合并文案**。本次没有修改主项目代码，也没有修改主项目 `docs/IMPLEMENTATION_STATUS.md`。
> 建议由主项目会话在确认后合并到 `F:\AI_Project_Development\docs\IMPLEMENTATION_STATUS.md`（可放在 §13 之后）。

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

## 可选补充到 §11/§12 的一句话

- Provider 文档 `docs/qwen3-tts-provider.md` 里“04C 29 条待听”“默认 vivian”等句子应同步改为：
  “04C 29/29 已听；`vivian` 默认被否；多 Profile 方案见 `voice-profile-design-04c.md`；未设置默认时 `type=none`。”
