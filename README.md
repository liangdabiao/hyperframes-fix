# HyperFrames Skill 变更记录

本项目目的是让 HyperFrames 生成视频适应国内需求，1 流畅的中文语音， 2 中文短视频样式新增， 3 优化整个生成视频流程。
基本上现在一键就可以完成横版竖版的快节奏短视频，例如随便丢给一个文章，一个doc，就能够制作视频！ 太棒！

## 安装指南

1 请你先安装好：HyperFrames
https://mp.weixin.qq.com/s/lZmiPoqHbxAWJq7PWUpKIg

2 下载本项目放进HyperFrames根目录
在claude code /codex 告诉 AI， 请参考 CHANGELOG.md 和 hyperframes-fix/skill/ 内容对 HyperFrames skill 进行优化。 

3 完成和测试：

❯ 配置好 .env 的 minimax-key 和 pexels_api_keys

❯ ❯ 视频制作需求： 请阅读 根目录的 #播客脚本.md，目的是 基于内容总结核心观点，生成信息图， 利用 hyperframes skill 生成流畅的 视频,  要求：1 带语音的视频，2 页面大概20个左右，内容充实, 3
  手机尺寸版本 ，4 信息流快消风格

4，真实效果：
https://www.bilibili.com/video/BV1Mg9qBSE37/?vd_source=86926e418c83af75f6850b5546388a79


## 特别感谢

https://linux.do 
社区 佬友支持！


## 详细修改介绍

> 以下记录了在 D:\HyperFrames-test 项目中对 `.claude/skills/hyperframes/` 所做的所有新增和修改。
> 原始 Skill 来源于 HyperFrames 官方包，以下标注了每项变更的类型。

---

## 一、新增文件 (3个)

### `references/external-tts.md` — 外部 TTS 语音集成 [新增]
- **内容**: 第三方 TTS (MiniMax) 语音合成与 HyperFrames 视频集成的完整工作流
- **涵盖**:
  - 核心原则 (逐场景音频、显式 duration、audio 放在 composition 外)
  - 中文/英文文本长度计算表 (按 speed 和场景时长)
  - **MiniMax speech-2.8-hd 实测语速校准表** (理论 vs 实际偏差 ~28%，推荐"先生成后量测"工作流)
  - 场景时长反向计算公式: `scene_duration = max(audio_dur + 2.0, 6.0)`
  - MiniMax HTTP API 完整调用示例 (Python)
  - ffprobe 验证流程
  - HTML 中 `<audio>` 元素放置规则和属性表
  - **音频轨道防重叠规则** (原因分析表 + 自动修复代码 + HTML 同步注意事项)
  - 故障排查表 (8种常见问题)
  - API Reference (endpoint、参数、voice_id 列表)
- **实战来源**: 制作「称重快餐店创业避坑」和「市场调研日报」两个视频时积累

### `references/news-flash-images.md` — Pexels 图片工作流 [新增]
- **内容**: News Flash 风格的背景图获取完整流程
- **涵盖**:
  - Pexels API 调用方式 (endpoint、auth、参数)
  - 按场景主题的搜索关键词对照表 (9类场景 → 搜索词 → 典型图片)
  - 图片选择标准 (主题匹配、色调、对比度、多样性)
  - CSS 三层架构 (bg-img → overlay → sc) 的完整代码
  - Overlay 透明度指南 (70-75% 用于数据密集场景, 55-65% 用于标题场景)
  - 三段式布局的 HTML/CSS 模板和尺寸注意事项
  - `hyperframes inspect` 的 overflow 处理方案 (`data-layout-allow-overflow` / `data-layout-ignore`)
- **关联**: 被修改的 `SKILL.md` 和 `visual-styles.md` 引用

### `references/tts-workflow.md` — TTS→HTML 端到端工作流 [新增]
- **内容**: 从脚本到渲染视频的完整 6 步流水线模板
- **涵盖**:
  - 流程总览图 (Script → Narration → Audio → Timeline → HTML → MP4)
  - Step 1: 脚本拆分为场景旁白 (中/英文文本长度表)
  - Step 2: TTS 音频生成完整 Python 脚本 (含下载 + ffprobe 测量)
  - Step 3: Timeline 构建脚本 (含防重叠自动对齐)
  - Step 4: HTML 组合接入 (audio 元素属性、放置位置、路径规则)
  - Step 5: Lint & Inspect 验证步骤
  - Step 6: 渲染 + ffprobe 验证音频流
  - 故障排查表 (5种常见问题)
  - 相关文档交叉引用

---

## 二、修改的文件 (3个)

### `visual-styles.md` — 视觉风格库 [修改]
**变更内容:**

1. **Quick Reference 表格** — 新增一行:
   ```
   | News Flash | Urgent, high-impact | Financial news, market updates, data-heavy short videos | Blur Crossfade or Push Slide |
   ```

2. **Mood → Style Guide 表格** — 新增一行:
   ```
   | Urgent, data-heavy, news, social | News Flash |
   ```

3. **新增风格 #9: News Flash (信息流快消)** — 完整风格定义，包含:
   - YAML token (colors, typography, rounded, spacing, motion, background)
   - 五大核心原则 (三段式布局、高对比色彩、粗犷描边、实景背景、密集数据行)
   - CSS 架构代码示例
   - 场景类型对照表
   - 动画注意事项

### `SKILL.md` — Skill 主文件 [修改]
**变更内容:**

1. **Step 1: Design system** — 预设数量从 8 改为 9
2. **References 章节** — 修改 visual-styles.md 描述 (8→9)，新增 external-tts.md、tts-workflow.md 和 news-flash-images.md 条目
3. **Quality Checks → Inspect** — 新增 **Inspect Strategy** 子节:
   - 时间戳选择策略 (只采样可见场景，避免隐藏场景误报)
   - Windows 超时处理 (`--timeout 30000`)
   - 批量 vs 分批检查建议
   - 隐藏场景误报处理 (`data-layout-ignore`)
   - 错误严重程度说明 (error/warning/info)

### `references/video-composition.md` — 视频构图规则 [修改]
**变更内容:**

新增 **Portrait Mode (1080×1920)** 章节，包含:
- 横屏 vs 竖屏尺寸对照表 (字号、间距、卡片)
- 布局策略 (垂直 flex 堆叠，避免水平网格)
- 转场方向 (竖屏用 y 轴推入)
- 背景图适配 (portrait orientation 参数)
- 密集内容场景的溢出处理方案
- 照片背景上的数据可读性技巧 (overlay 透明度、text-stroke、card 背景)

---

## 三、文件变更总览

| 文件路径 | 变更类型 | 变更范围 |
|---|---|---|
| `references/external-tts.md` | **新增** | 180行，完整文件 |
| `references/news-flash-images.md` | **新增** | 135行，完整文件 |
| `references/tts-workflow.md` | **新增** | 145行，完整文件 |
| `visual-styles.md` | 修改 | +3处表格更新, +1个完整风格定义 (~120行) |
| `SKILL.md` | 修改 | +Step1 更新, +3个 References 条目, +Inspect Strategy 子节 (~30行) |
| `references/video-composition.md` | 修改 | +Portrait Mode 章节 (~55行) |

---

## 四、变更已整合项

以下实践经验均已写入 Skill 文档:

| 实践经验 | 写入位置 |
|---|---|
| MiniMax TTS 语速校准表 (理论 vs 实测) | `external-tts.md` → "Speed calibration" |
| 音频轨道防重叠规则 (原因+修复代码) | `external-tts.md` → "No overlaps on same track" |
| Pexels API 图片工作流 | `news-flash-images.md` → 完整文件 |
| 竖屏布局指南 (尺寸表+布局策略) | `video-composition.md` → "Portrait Mode" |
| Inspect 实战策略 (时间戳/超时/误报) | `SKILL.md` → "Inspect Strategy" |
| TTS→HTML 端到端工作流模板 | `tts-workflow.md` → 完整文件 |

---

## 五、`SKILL.md` 修改详述

`SKILL.md` 是 HyperFrames Skill 的主文件，定义了完整的视频制作流程、规则和检查清单。以下列出所有新增和修改内容的逐行对照。

### 修改 1: Step 1: Design system — 预设数量更新 (第33行)

**位置**: `### Step 1: Design system` → 第1项选项

**原文**:
```
1. **User named a style or mood?** → Read [visual-styles.md](./visual-styles.md) for the 8 named presets. Pick the closest match.
```

**改为**:
```
1. **User named a style or mood?** → Read [visual-styles.md](./visual-styles.md) for the 9 named presets (including News Flash 信息流快消). Pick the closest match.
```

**变更说明**:
- 预设数量 `8` → `9`
- 括号内标注新增的 `News Flash 信息流快消` 风格，便于 agent 知道第9个预设是什么
- **原因**: 新增了 News Flash (信息流快消) 风格定义（见 `visual-styles.md` 修改记录）

---

### 修改 2: Quality Checks → Visual Inspect — 新增 Inspect Strategy 子节 (~30行)

**位置**: `## Quality Checks` → `### Visual Inspect` 之后、`### Contrast` 之前

**新增完整内容**:

```markdown
### Inspect Strategy

**Timestamp selection matters.** The inspector measures ALL elements in the DOM, including hidden scenes (opacity:0). A timestamp like `t=0.5` will report overflow from every scene stacked in the document, even though only one is visible. This floods results with false positives.

**Use `--at` to sample only at visible timestamps:**

\```bash
# Calculate when each scene is visible, then sample at those times
npx hyperframes inspect --at 2,10,45,100,150 --no-contrast --timeout 30000
\```

**Windows timeout.** The default 10s navigation timeout is too short on Windows. Always specify `--timeout 30000` (30s).

**Batch vs individual.** More than 8 timestamps may hit timeout. Run in smaller batches:

\```bash
npx hyperframes inspect --at 2,10,45 --no-contrast --timeout 30000
npx hyperframes inspect --at 100,150,175 --no-contrast --timeout 30000
\```

**Hidden-scene false positives.** If an error references a scene that isn't active at the sampled timestamp (e.g., scene4 text overflow at t=2 when scene1 is playing), the error is cosmetic — the content is clipped by `overflow:hidden` and never visible. Options:
- **Ignore** — the error won't affect rendered output
- **`data-layout-ignore`** — add to the hidden scene's element to skip inspection entirely

**Error severity:**
- `error` — real visual issue, must fix
- `warning` — container overflow, usually fixable
- `info` — canvas overflow on hidden/offscreen content, usually safe to ignore
```

**变更说明**:
- **时间戳选择策略**: 解释了 inspect 测量 DOM 中所有元素（包括隐藏场景）的行为，指导使用 `--at` 只采样可见场景的时间点
- **Windows 超时处理**: 默认 10s 在 Windows 上不够，必须 `--timeout 30000`
- **批量 vs 分批**: 超过 8 个时间戳可能超时，建议分批运行
- **隐藏场景误报处理**: 解释了为什么隐藏场景的 overflow 报告是误报，提供两种处理方式（忽略或 `data-layout-ignore`）
- **错误严重程度**: 定义了 error/warning/info 三级的含义和处理优先级
- **原因**: 在制作竖版播客视频时，inspect 报告了大量隐藏场景的误报（opacity:0 的场景内容仍被测量），浪费了大量调试时间。总结出来的策略可以避免后续项目重复踩坑

---

### 修改 3: References 章节 — 新增 3 个条目 + 修改 1 个条目描述

**位置**: `## References (loaded on demand)`

#### 3a. 新增 `external-tts.md` 条目

```markdown
- **[references/external-tts.md](references/external-tts.md)** — Third-party TTS integration (MiniMax, OpenAI, etc.) with per-scene audio segmentation, speed calibration, overlap prevention, and composition wiring. Read when adding voiceover from an external TTS API to a multi-scene composition.
```

**说明**: 指向新增的外部 TTS 集成参考文档，覆盖 MiniMax/OpenAI 等 API 的逐场景音频分段、语速校准、防重叠和 composition 接入。

#### 3b. 新增 `tts-workflow.md` 条目

```markdown
- **[references/tts-workflow.md](references/tts-workflow.md)** — End-to-end TTS→HTML workflow: script splitting, audio generation, timeline calculation, composition wiring, lint/render pipeline. Read when producing a full voiceover video from scratch.
```

**说明**: 指向新增的 TTS→HTML 端到端工作流文档，覆盖从脚本拆分到最终渲染的完整 6 步流水线。

#### 3c. 新增 `news-flash-images.md` 条目

```markdown
- **[news-flash-images.md](references/news-flash-images.md)** — Pexels API integration workflow for News Flash style: search queries by scene theme, image selection criteria, overlay darkness guide, three-segment layout sizing, and inspect overflow handling.
```

**说明**: 指向新增的 Pexels 图片工作流文档，覆盖 News Flash 风格的背景图获取、CSS 三层架构和 inspect 溢出处理。

#### 3d. 修改 `visual-styles.md` 条目描述

**原文**:
```
- **[visual-styles.md](visual-styles.md)** — 8 named visual styles with hex palettes, GSAP easing signatures, and shader pairings. Read when user names a style or when generating design.md.
```

**改为**:
```
- **[visual-styles.md](visual-styles.md)** — 9 named visual styles with hex palettes, GSAP easing signatures, and shader pairings. Includes News Flash (信息流快消) for urgent, data-heavy financial/news content with Pexels background images. Read when user names a style or when generating design.md.
```

**变更说明**:
- 数量 `8` → `9`
- 新增 `Includes News Flash (信息流快消) for urgent, data-heavy financial/news content with Pexels background images` 描述
- 让 agent 在用户提到"新闻"、"数据密集"、"金融"等内容时，知道应选择 News Flash 风格

---

### 变更汇总表

| 修改项 | 位置 (SKILL.md) | 类型 | 行数 |
|:---|:---|:---|:---|
| 预设数量 8→9 + News Flash 标注 | Step 1: Design system | 修改 | ~1行 |
| Inspect Strategy 子节 | Quality Checks → Visual Inspect 之后 | **新增** | ~30行 |
| `external-tts.md` References 条目 | References 章节 | **新增** | ~1行 |
| `tts-workflow.md` References 条目 | References 章节 | **新增** | ~1行 |
| `news-flash-images.md` References 条目 | References 章节 | **新增** | ~1行 |
| `visual-styles.md` 描述更新 | References 章节 | 修改 | ~1行 |
| **合计** | | | **~35行** |
