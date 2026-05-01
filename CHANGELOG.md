# HyperFrames Skill 变更记录

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

---

## 六、视频制作实战中发现并修复的 Bug (2026-05-02)

> 在制作 4 个竖屏新闻快消风格视频（AI创业、YouTube出海、徐志胜脱口秀、五金店）过程中，发现并修复了多个框架使用层面的 Bug。以下经验已同步写入 `SKILL.md` 的 "Common Pitfalls" 章节。

---

### Bug 1: GSAP `from()` + `immediateRender` 导致卡片永久消失 [严重]

**现象:** 每个 scene 打开后，card 列表瞬间消失，只剩第一个 item 可见。

**根因:** `gsap.from()` 默认 `immediateRender: true`，会立即将所有匹配元素设为"from"状态（opacity:0）。当第二个 `from()` 动画同一元素子集时（如先 `.card` 再 `.card:nth-child(2)`），第二个 `from()` 会捕获已被设为 opacity:0 的状态作为终态——元素从不可见动画到不可见，永久消失。

```js
// BUG — 两个 from() 冲突，第2个卡片捕获了不可见状态作为终态
tl.from("#s06 .card", {y: 50, opacity: 0}, 48.8);
tl.from("#s06 .card:nth-child(2)", {y: 50, opacity: 0}, 49.2);

// FIX — fromTo 显式定义终态 + overwrite 防冲突 + immediateRender 防闪烁
tl.fromTo("#s06 .card:nth-child(1)", {y: 50, opacity: 0}, {y: 0, opacity: 1, duration: 0.6, overwrite: "auto", immediateRender: true}, 48.8);
tl.fromTo("#s06 .card:nth-child(2)", {y: 50, opacity: 0}, {y: 0, opacity: 1, duration: 0.6, overwrite: "auto", immediateRender: true}, 49.2);
```

**关键教训:**
- 永远不要对重叠选择器使用 `gsap.from()`
- 使用 `gsap.fromTo()` 并显式指定终态 `{y:0, opacity:1}`
- 添加 `overwrite:"auto"` 防止多个 tween 争抢同一属性
- 添加 `immediateRender:true`（`fromTo` 默认为 `false`，不添加会导致卡片在动画触发前以 CSS 默认状态闪现一帧）
- 只使用独立 `:nth-child()` 选择器，不混用批量 `.card` 和子选择器

**影响范围:** ai-video、xuzhisheng-video、youtube-video 三个已发布视频

---

### Bug 2: `::after` 渐变遮罩透明度过高，背景图片不可见 [中等]

**现象:** 配置了 `background-image` 的场景，图片完全看不到，背景只有纯色渐变。

**根因:** `::after` 伪元素的 `rgba()` alpha 值设为 0.82-0.90，遮罩几乎完全不透明，完全覆盖了背景图片。

```css
/* BUG — 0.85 遮罩几乎不透明，图片不可见 */
.scene-bg::after { background: linear-gradient(180deg, rgba(0,0,0,0.85), rgba(0,0,0,0.75)); }

/* FIX — 0.55 遮罩半透明，图片清晰可见 */
.scene-bg::after { background: linear-gradient(180deg, rgba(0,0,0,0.55), rgba(0,0,0,0.45)); }
```

**关键教训:**
- 遮罩 alpha 值控制在 **0.40-0.60** 之间
- 上沿（标题侧）可以稍深 0.55-0.60，下沿可以稍浅 0.40-0.50
- 超过 0.70 基本看不到背景图，超过 0.85 完全看不见
- 调整透明度时注意使用批量替换（Python sed/regex），确保只修改 `::after` 渐变中的 alpha，不影响卡片半透明背景色

**影响范围:** 所有 4 个视频

---

### Bug 3: 时间取整导致同轨道 clip 重叠 [中等]

**现象:** `npx hyperframes lint` 报错 `overlapping_clips_same_track`。

**根因:** 从 `timeline.json` 读取时间值后四舍五入到 1 位小数。当 scene A 的 `(start + duration)` 四舍五入后大于 scene B 的 start 时，产生 0.1s 重叠。

**示例:**
```
S08: start=74.8, duration=10.9 → end=85.7
S09: start=85.6 → 85.6 < 85.7 → 重叠 0.1s
```

**关键教训:**
- 从 timeline.json 读取精确值后，必须验证每个 clip 的 `end (start + duration)` ≤ 下一个 clip 的 `start`
- 或者直接使用精确值（保留 2 位小数），不要四舍五入
- 修改 duration 比修改 start 更安全（只影响一个 clip 的结尾，不影响后续依赖链）

**影响范围:** wujin-video（3 处重叠）

---

### Bug 4: 卡片内容只占屏幕顶部 20%，下方大面积空白 [体验]

**现象:** 每个 scene 的 card 列表集中在画面顶部，下方 80% 为空白。

**根因:** `.body` 容器使用 `justify-content: flex-start`，内容贴顶。1-2 张卡片只有 200-400px 高，而可用空间约 1600px。

```css
/* BUG — flex-start 导致内容贴顶 */
.body { justify-content: flex-start; }

/* FIX — center 使卡片垂直居中 */
.body { justify-content: center; }
```

**补充修复:** 同时发现字号偏小（在 1080×1920 画布上 20-24px 的正文难以阅读），全面上调：
- `.card-t` (卡片标题): 24px → 32px
- `.card-b` (卡片正文): 20px → 26px
- `.tag` (标签): 22px → 28px
- `.highlight` (高亮数字): 28px → 36px
- `.card` padding: 22px → 30px
- 标题 `.t1`: 80px → 100px

**关键教训:**
- 竖屏视频（1080×1920）中，正文至少需要 24px+ 才能清晰阅读
- `justify-content: center` 配合 `flex:1` 是卡片垂直居中的最佳方案
- 字号调整应全局生效（基础 CSS class），而非逐场景修改

**影响范围:** 所有 4 个视频

---

### Bug 5: 多数场景缺少背景图片 [体验]

**现象:** 只有 3-5 个 scene 配了背景图，其余 scene 背景纯色渐变，视觉单调。

**根因:** Pexels API 搜索结果有限，每个视频只下载了 3 张图，仅分配给标题和个别关键场景。

**修复策略:** 复用已有图片。对缺少 `background-image` 的 scene，从可用图片池中循环分配。

**关键教训:**
- 图片数量不足时，优先按主题匹配分配，不匹配的循环复用
- 使用 Python 脚本批量添加，避免手动遗漏
- 注意处理 CSS 多选择器规则（如 `#s11 .scene-bg::after,#s12 .scene-bg::after`），不能用简单字符串替换，会破坏规则结构

---

### 变更同步状态

| Bug | 已修复文件 | 已写入 Skill |
|:---|:---|:---|
| Bug 1: from() 卡片消失 | ai-video, xuzhisheng-video, youtube-video, wujin-video | `SKILL.md` → Common Pitfalls → GSAP from() immediateRender conflict |
| Bug 2: 遮罩过高 | 同上 | `SKILL.md` → Common Pitfalls → Background image gradient overlay opacity |
| Bug 3: 时间取整重叠 | wujin-video | `SKILL.md` → Common Pitfalls → Track overlap from rounded timings |
| Bug 4: 内容贴顶 | 同上 (JS 调整无需写入) | (经验级，无需持久化) |
| Bug 5: 背景图缺失 | 同上 (CSS 调整无需写入) | (经验级，无需持久化) |
| Bug 6: 字号过小 | ai-video, xuzhisheng-video, youtube-video, wujin-video | `SKILL.md` → Animation Guardrails + Common Pitfalls → Font sizes too small for portrait video |

---

### Bug 6: 字号过小，用户两次反馈"字还是很小" [体验→规则]

**现象：** 用户连续两次反馈"字还是很小啊！"。第一次修复时将 card-t 从 24px 增到 32px、card-b 从 20px 增到 26px，用户仍不满意。最终需要大幅提升：card-t 48px、card-b 38px、tag 38px、ht 50px、highlight 52px，同时增加卡片内边距和容器内边距。

**根因：** Skill 中原始字号下限（"60px+ headlines, 20px+ body"）是按横屏 1920×1080 标定的，对竖屏 1080×1920 严重不足。竖屏视频在手机上观看，所有元素必须是"广告牌级"字号。此外，第一次修复时只增加了约 30%，增量不够 — 用户说"字很小"时应该一次性增加 40-50%。

**修复方案（所有 4 个视频）：**

| 属性 | 修复前 | 修复后 | 增幅 |
|:---|:---|:---|:---|
| `.card-t` (卡片标题) | 32px | **48px** | +50% |
| `.card-b` (卡片正文) | 26px | **38px** | +46% |
| `.tag` (标签) | 28px | **38px** | +36% |
| `.tag-s` (副标签) | 24px | **32px** | +33% |
| `.highlight` (高亮) | 36px | **52px** | +44% |
| `.ht` (场景标题) | 34-36px | **48-50px** | +40% |
| `.card` padding | 30px 28px | **44px 40px** | +47% |
| `.sc` padding | 80px 48px 60px 48px | **100px 56px 80px 56px** | +25% |
| `.num-big` | 72px | **88px** | +22% |
| `.brand-bar-text` | 14px | **18px** | +29% |
| `.t2` (副标题) | 40px | **48px** | +20% |
| `.t3` (系列标签) | 28px | **36px** | +29% |
| closing `.t1` | 42px | **52px** | +24% |

**关键教训：**
1. 竖屏 1080×1920 的字号下限必须比横屏高 50-80%，不能直接套用横屏标准
2. 当用户说"字很小"时，不要只增加 10-20%，应该一次性增加 40-50%
3. 场景级 CSS 覆盖（如 `#s05 .ht{font-size:32px}`）也必须同步更新，否则局部字号仍偏小
4. padding 和字号必须同步增长，否则文字大了但卡片没变大，导致溢出
