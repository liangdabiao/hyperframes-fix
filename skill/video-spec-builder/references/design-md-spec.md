---
name: design-md-spec
description: HyperFrames 自定义主题 design.md 的完整格式规范 —— YAML 头字段速查 + 6 个必备章节 + 一个完整可复用范本。用户在 video-spec-builder 选了"自定义主题"路径后，按本文件生成或修改 design.md。
type: reference
---

# design.md 编写规范

HyperFrames 的视觉主题通过**项目根目录**的一个 `design.md` 文件定义（外加可选的 `tokens.css`）。**没有 `styles/` 子文件夹、不能放别的地方**。

**什么时候读**：用户选定了"自定义主题"路径；用户给你 3 个形容词 / 参考品牌 / 视觉风格描述让你生成主题；或者用户给了一份 design.md 让你改。

**前提**：video-spec-builder 走"自定义主题"路径时，已经在对话里问过用户至少 3 个具体的视觉方向（不要是"高大上 / 科技感"这种形容词），或拿到了参考品牌 / 风格参考链接。

---

## 文件结构（强制）

```markdown
---
[YAML front-matter 块：name / colors / typography / rounded / spacing / motion / atmosphere]
---

## Overview
[1-2 句话总结这套主题的视觉人格]

## Colors
[背景 / 前景 / 强调色怎么用]

## Typography
[hero / body / label / quote 各自什么字体、字号、字重、间距]

## Elevation
[阴影 / 模糊 / 表面叠加规则]

## Components
[卡片 / 按钮 / 标签 / 图标 等的具体样式]

## Do's and Don'ts
[3-5 条绝对要做 + 3-5 条绝对不要做]
```

**6 个 `##` 章节是强制的**。HyperFrames 的渲染器按这 6 个标题去索引样式决策，少一个就降级到默认。

---

## YAML 头字段速查

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | string | ✓ | 主题名（≤ 30 字符），如 `"Spec Mono"` / `"深色极简"` |
| `colors.primary` | hex | ✓ | 主背景色 |
| `colors.on-primary` | hex | ✓ | 主背景上的前景色（通常是文字） |
| `colors.surface` | hex | ✓ | 卡片 / 抬升表面色（比 primary 略亮或略暗） |
| `colors.accent` | hex | ✓ | 强调色，CTA / 钩子 / 关键数据用 |
| `typography.hero` | object | ✓ | 标题字体规则 |
| `typography.body` | object | ✓ | 正文 |
| `typography.label` | object | ✓ | 小标签 / kicker / 序号 |
| `typography.quote` | object | optional | 引用 / 金句 |
| `typography.stat` | object | optional | 大数字 / 关键数据 |
| `rounded.none/sm/md/lg` | px | ✓ | 圆角阶，组件按尺寸挑 |
| `spacing.sm/md/lg/xl/xxl` | px | ✓ | 间距阶（8/16/24/40/64 是常见） |
| `motion.energy` | enum | ✓ | `subtle` / `moderate` / `energetic` |
| `motion.easing.entry` | string | ✓ | GSAP easing 名（`expo.out` 等） |
| `motion.easing.exit` | string | ✓ | 同上 |
| `motion.easing.ambient` | string | optional | 持续动效用（如脉冲） |
| `motion.duration.entrance` | seconds | ✓ | 进场动画时长（推荐 0.4-0.8） |
| `motion.duration.hold` | seconds | ✓ | 入场后保持静止时长（推荐 2-3） |
| `motion.duration.transition` | seconds | ✓ | scene 之间过渡时长（推荐 0.5-0.8） |
| `atmosphere` | list | optional | 装饰元素（dot-grid / hairline-rules / scanlines / registration-marks 等） |

**未列出的字段会触发降级**——比如没填 `rounded.md`，组件只能用 0 圆角或默认 8px。

---

## 各章节写什么 / 不要写什么

### Overview
- 写 1-2 句话描述这套主题的"人格"
- 例子：
  > "深色极简，几何克制，工程感。纯黑底 + 纯白前景 + 单点朱红强调，灵感取自 SpaceX × Grok 内部数据看板。"
- 不要写：技术名（"用 GSAP 做动画"）、渲染细节（"1080×1920 渲染"）

### Colors
- 4 个核心色（primary / on-primary / surface / accent）各自**怎么用**：
  - `primary` 用作全局背景
  - `on-primary` 用作所有正文 / 标题文字
  - `surface` 用于卡片、面板、抬升容器
  - `accent` 仅用于关键 CTA / 钩子 / 数据高亮 / 关键图标
- 列出每个颜色的 hex 和透明度建议
- 不要写"和 logo 一样"——给具体 hex

### Typography
- 至少 hero / body / label 三档的字体规则
- 每档至少给：`fontFamily` / `fontSize` / `fontWeight`
- 字体必须是**已安装或 web-loadable 的真实字体名**，不能用"现代无衬线"这种描述
- 推荐字体族：
  - 中文：`PingFang SC` / `Microsoft YaHei` / `Noto Sans SC`
  - 西文：`Inter` / `Space Grotesk` / `JetBrains Mono` / `Barlow` / `Instrument Serif`
- 如果用了 HyperFrames 不内置的字体（如 `Instrument Serif`），需要把 `.woff2` 放到项目根的 `fonts/` 文件夹

### Elevation
- shadow / blur / 表面叠加规则
- 例子：
  - "卡片用 `0 8px 24px rgba(0,0,0,0.4)` 阴影"
  - "全屏背景不用阴影"
  - "前景元素加 1px hairline 边"
- 至少 3 条具体规则

### Components
- **必须**覆盖以下组件（每个至少 1 句规则）：
  - 卡片 / 面板
  - 按钮 / CTA
  - 标签 / kicker
  - 数字 / stat
  - 图标 / 装饰元素
- 例子：
  - 卡片：`background: surface; border: 1px solid on-primary @ 8% opacity; rounded: md; padding: lg`
  - CTA：`background: accent; color: primary; rounded: full; padding: md lg`

### Do's and Don'ts
- 至少 3 条 Do + 3 条 Don't
- 必须**具体到视觉决策**，不是泛泛的"保持一致"
- 例子：
  - ✓ "hero 标题用 8rem，2 行以内"
  - ✓ "数字 stat 永远用纯 accent 色"
  - ✗ "不要用太多颜色"
  - ✗ "不要用 emoji 装饰"
  - ✗ "副标题不要超过 16 字"

---

## 完整范本

下面这个范本可以**直接拷走**当起点。基于本仓库的 spec-mono 主题简化：

```markdown
---
name: 暗色极简
colors:
  primary: "#0A0A0C"          # 接近纯黑，带极轻冷调
  on-primary: "#F5F5F5"       # 主文字色
  surface: "#15151A"          # 卡片表面（比 primary 略亮）
  accent: "#FF5050"           # 朱红强调
typography:
  hero:
    fontFamily: PingFang SC
    fontSize: 6rem
    fontWeight: 800
    letterSpacing: -0.02em
  stat:
    fontFamily: Inter
    fontSize: 9rem
    fontWeight: 900
    letterSpacing: -0.04em
  body:
    fontFamily: PingFang SC
    fontSize: 1.2rem
    fontWeight: 400
    lineHeight: 1.6
  label:
    fontFamily: JetBrains Mono
    fontSize: 0.75rem
    fontWeight: 500
    letterSpacing: 0.22em
    textTransform: uppercase
  quote:
    fontFamily: PingFang SC
    fontSize: 4rem
    fontWeight: 700
    fontStyle: normal
rounded:
  none: 0px
  sm: 2px
  md: 6px
  lg: 12px
spacing:
  sm: 8px
  md: 16px
  lg: 24px
  xl: 40px
  xxl: 80px
motion:
  energy: moderate
  easing:
    entry: "expo.out"
    exit: "power4.in"
    ambient: "sine.inOut"
  duration:
    entrance: 0.6
    hold: 2.0
    transition: 0.5
  atmosphere:
    - dot-grid
    - hairline-rules
---

## Overview
深色极简，几何克制，工程感。暗底 + 高对比文字 + 单点朱红强调，适合讲"挖掘机会 / 数据洞察"类话题。

## Colors
- `primary (#0A0A0C)`：全局背景，承载所有内容
- `on-primary (#F5F5F5)`：标题、正文、关键数字
- `surface (#15151A)`：卡片 / 抬升面板，与 primary 的对比差保持在 5-8% 亮度
- `accent (#FF5050)`：**只**用于 CTA、hook 钩子、关键数据、当前激活态；一屏最多 1-2 处
- 透明度阶梯：前景文字 100% / 60% / 40% / 20% 四档

## Typography
- **hero**：6rem / 800，用于 scene 大标题；最多 2 行，行高 1.2
- **stat**：9rem / 900，用于"8 平台"、"1000+"等关键数字
- **body**：1.2rem / 400，正文与描述，行高 1.6
- **label**：0.75rem / JetBrains Mono / 大写 + 0.22em letterSpacing，用于 kicker / 序号 / 标签
- **quote**：4rem / 700，用于核心金句；不用斜体
- 中文字符宽度西文字符宽度的 1.1 倍，hero 标题西文比例更高时可酌情收紧 letterSpacing

## Elevation
- 卡片：`box-shadow: 0 8px 24px rgba(0,0,0,0.5)` + 1px hairline 边（rgba(245,245,245,0.08)）
- 全屏背景不用阴影
- 浮层（弹窗 / 提示）：`backdrop-filter: blur(20px)` + 内部 surface 配色
- 关键数据 stat 块：轻微外发光 `text-shadow: 0 0 40px rgba(255,80,80,0.3)`

## Components
- **卡片**：`background: surface; border: 1px solid rgba(245,245,245,0.08); rounded: md; padding: lg xl`
- **CTA 按钮**：`background: accent; color: primary; rounded: 100px; padding: md xl; fontWeight: 700`
- **label/kicker**：`color: accent; textTransform: uppercase; letterSpacing: 0.22em`
- **数据 stat**：`color: accent; fontWeight: 900; text-shadow 微发光`
- **图标**：1.5px 描边、纯白或纯朱红，不用 emoji

## Do's and Don'ts
- ✓ hero 标题 6rem，2 行以内，行高 1.2
- ✓ 一屏最多 1 个 accent 高亮元素
- ✓ 卡片用 1px hairline 边区分，不用粗黑边
- ✗ 不用渐变色块背景（纯色 + 透明度变化）
- ✗ 不用 emoji 作 UI 元素
- ✗ 副标题不超过 16 字
```

---

## 跟 video-spec.md 的字段映射

video-spec.md § 4 视觉规范只需要写：

```yaml
theme: design.md（项目根目录）  # 自定义主题都这样写
accent_override: <hex>           # 可选 · 覆盖 design.md 里的 accent
```

不要在 video-spec.md 里再复述 design.md 的所有字段——以项目根 `design.md` 为准。

---

## 从对话生成 design.md 的工作流

1. 已经在 video-spec-builder 对话里跟用户确认了 3 个具体视觉形容词（如"暗色 / 极简 / 工程感"）+ 1-2 个参考品牌（SpaceX 官网 / Stripe 文档 等）
2. 把这 3 个形容词展开成 4 个核心色 + 字体选型 + 间距阶
3. 按本文件模板写 YAML 头 + 6 章节
4. 写到项目根目录的 `design.md`（不是 video-spec.md 旁边）
5. 同时建议用户如需复用 CSS，放一份 `tokens.css` 在项目根（不是 `styles/`）
6. 在 video-spec.md § 4 写 `theme: design.md（项目根目录）`

---

## 速查：什么时候选 8 预设 / 什么时候选自定义

| 场景 | 选 |
|---|---|
| 内部 demo / 测试 / 时间紧 | 8 预设（直接写名字） |
| 已有品牌视觉规范 | 自定义 design.md（按品牌色 / 字体） |
| 用户给了 3 个具体形容词 + 参考品牌 | 自定义 design.md（按本文件生成） |
| 想要"和 XX 品牌很像" | 自定义 design.md + 参考该品牌官网 |
| 完全没方向 / 让 agent 决定 | 8 预设（agent 推一个） |

---

## 相关 References

- `hyperframes-render.md` — design.md 写完后如何落地成可渲染 HTML
- `scene-breakdown.md` — 怎么把 design.md 的视觉规则套到具体分镜
- `question-bank.md` Phase 4 — 主题选择阶段的完整对话策略
