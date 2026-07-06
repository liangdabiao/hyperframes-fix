---
name: hyperframes-render
description: HyperFrames 渲染端 HTML 合成的硬性约束、踩坑清单、必做验证。从 video-spec.md 到 MP4 的最后一步，违反任何一条都可能让渲染静默失败或产出崩坏画面。
type: reference
---

# HyperFrames 渲染侧速查

video-spec.md 写完之后，要把它变成 MP4 / WebM，必须把分镜转成一个 HTML composition 喂给 HyperFrames 渲染。本文件是这个阶段所有"看似常识但实战必踩"的硬约束。

**什么时候读**：用户给出 video-spec.md 并问"怎么生成视频" / "下一步怎么走" / 直接说"渲染"时；也包括自动 / 半自动跑 build_html.py 的 agent。

---

## 整体流程

```
video-spec.md
  ↓  1. 设计 scene 数据结构（labels / 时长 / 文案 / 视觉类型）
scene_plan + timeline.json
  ↓  2. 生成 TTS（如有旁白，按 tts-workflow.md）
audio/scene_XX.mp3
  ↓  3. 量音频时长 → 反推 scene_duration = max(audio_dur + 2.0, 6.0)
timeline.json 更新
  ↓  4. 写 build_html.py 生成 index.html
index.html
  ↓  5. npx hyperframes lint                ← 0 errors 必须
  ↓  6. npx hyperframes inspect              ← 0 issues 必须
  ↓  7. Playwright getComputedStyle 验证     ← 关键 layout 选择器都生效
  ↓  8. ffmpeg 抽 1 帧肉眼检查              ← 兜底
  ↓  9. npx hyperframes render
renders/output.mp4
  ↓  10. ffprobe 验证音轨存在 + 时长 + 分辨率
```

**第 5-8 步不是可选的**。任何一步跳了都可能在最终 MP4 里留下只有肉眼看才看得出来的 bug。

---

## 关键约束（违反即静默失败）

### C1. 用 `window.addEventListener("load")`，不要 `DOMContentLoaded`

GSAP 时间线初始化必须挂在 `window.load` 上。`DOMContentLoaded` 会导致渲染时 `Runtime ready: false`，报错 "Composition has zero duration"，但 lint 完全不报。

```js
// ✓ 对
window.addEventListener("load", function() {
  window.__timelines = window.__timelines || {};
  const tl = gsap.timeline({defaults:{overwrite:"auto"}});
  // tl.fromTo(...) 调用
  window.__timelines["main"] = tl;
});

// ✗ 错
document.addEventListener("DOMContentLoaded", () => {
  const tl = gsap.timeline();
  // ...
});
```

### C2. 每个 `tl.fromTo` 必须带 `overwrite:"auto", immediateRender:true`

不带的话，GSAP 在 headless 渲染时进入懒渲染状态，时间线长度可能被计算成 0。wx-video 的 43 处 `immediateRender` 不是装饰，是必须的：

```js
tl.fromTo("#s1-inner", {opacity:0}, {opacity:1, duration:0.4, ease:"power2.out", overwrite:"auto", immediateRender:true}, 0.00);
```

### C3. HTML scene ID 与 CSS 选择器必须 100% 对齐

这是最阴的 bug。把 ID 从 `s01` 改成 `s1` 时，CSS 里所有 `#s01 .xxx` 也得一起改。改漏一个，那个 scene 的所有 layout 样式（`display:grid`、`font-size` 等）全部失效，内容塌成默认 0 字号小方块挤在屏幕中央。

**Lint / inspect 不会抓这个**，因为结构上元素还在，匹配不上只是 CSS 的事。

**唯一可靠的检测**：在 build 之后、render 之前，用 Playwright 起本地服务、加载 index.html、跑 `getComputedStyle` 验证关键选择器：

```js
const el = document.querySelector('.platform-cloud');
const cs = getComputedStyle(el);
console.assert(cs.display === 'grid', `expected grid, got ${cs.display}`);
console.assert(cs.gridTemplateColumns !== 'none', 'grid columns missing');
```

或更暴力：**直接用 Playwright 截图肉眼检查每个 scene 的可视状态**。

**更彻底的预防**：ID 和 CSS 共享一个变量，不要硬编码两份：

```python
# build_html.py
SCENE_IDS = ['s1', 's2', 's3', 's4', 's5']
for i, s in enumerate(scenes):
    css = f"#{SCENE_IDS[i]} .platform-cloud {{ display: grid; ... }}"
    html_id = f'id="{SCENE_IDS[i]}"'
    # 两个都用同一个 SCENE_IDS[i]
```

### C4. `<audio>` 必须放在 `<body>` 顶层，不能进 composition div

放进 composition 会被静默忽略，渲染出来没声音。**不能**写：

```html
<div data-composition-id="main">
  <audio src="..."> ← 错！放错地方
</div>
```

**必须**写：

```html
<body>
  <div data-composition-id="main" ...>...</div>
  <audio id="narr-s01" src="..." data-start="..." data-duration="..." data-track-index="2" data-volume="1"></audio>
</body>
```

注意：
- `data-track-index="2"`（独立于画面 track 1）
- `data-duration` 必须是**显式秒数**，写 `"auto"` 浏览器能播但渲染没声
- `<audio>` 元素**必须**有 `id` 属性（linter 要求 + renderer 找媒体用）
- 用 `src` 属性（不是 `data-src`）

### C5. 时间线浮点重叠防御

相邻 `<audio>` 元素：`data-start + data-duration` 必须 `<` 下一段 `data-start`。浮点累积可能让两个 clip 在同一 track 上重叠 0.001s，lint 会报 `overlapping_clips_same_track`：

**修法**：把 `data-duration` 向下取整 0.1s，或下一段 `data-start` 加 +0.01s。

```python
# Force-align
for i in range(len(scenes) - 1):
    end = scenes[i]["start"] + scenes[i]["duration"]
    if end > scenes[i+1]["start"]:
        scenes[i]["duration"] = round(scenes[i+1]["start"] - scenes[i]["start"] - 0.1, 1)
```

### C6. `data-duration` 必须用实际音频时长，不是 scene 时长

```html
<!-- ✓ 对：用 ffprobe 测出来的真实时长 -->
<audio data-duration="3.87">

<!-- ✗ 错：用 scene slot 时长 6.0，但实际音频只有 3.87s -->
<audio data-duration="6.0">
```

后者会让渲染器把 audio clip 强行截到 audio 真实长度，留出"无声 2s"，体验崩坏。

### C7. composition 根属性必须齐全

```html
<div data-composition-id="main"
     data-width="1080" data-height="1920"
     data-start="0" data-duration="<total_seconds>">
```

- 缺 `data-composition-id` → lint 报 `root_missing_composition_id`
- 缺 `data-width/data-height` → lint 报 `root_missing_dimensions`
- `data-duration` 与各 scene 的 `data-start + data-duration` 之和必须精确相等

每个 scene 自己的 div：

```html
<div class="scene clip" id="s1"
     data-start="0.0" data-duration="6.0"
     data-track-index="1">
  <div class="scene-inner" id="s1-inner">...</div>
</div>
```

`class="scene clip"` 必须有 `clip` —— 没有它，scene 默认全程可见，与其他 scene 叠加；lint 报 `timed_element_missing_clip_class`。

### C8. GSAP 操作 `#sXX-inner`（scene-inner），不是 `#sXX`

GSAP 时间线进入/退出动画要操作 scene 的 inner 容器（用 `opacity:0/1`），不要直接动 scene 本身（动 scene 会被 `clip` 类的可见性控制覆盖）。

### C9. CSS 用 `@font-face { src: local(...) }` 兜底

如果用 `font-family: 'Inter', 'PingFang SC', sans-serif`，渲染时如果系统没有 Inter，会回退到 PingFang。但**HyperFrames 的 lint 会报 `font_family_without_font_face`**——必须先注册：

```css
@font-face { font-family: 'Inter'; src: local('Inter'); }
@font-face { font-family: 'PingFang SC'; src: local('PingFang SC'); }
```

### C10. 9:16 竖屏布局的硬规则

抖音/小红书/视频号/TikTok/Shorts 都是 9:16。CSS 里：
- 字号基线 ≥ 40px（4 米外手机也能看）
- 文字宽度收窄到画面 60%（max-width: 920px @ 1080-wide）
- padding-top/bottom 至少 120px
- grid/flex 子元素 max-width 不要写死 1920 横屏的数值

---

## 必备验证（缺一不可）

### V1. lint 必须 0 errors

```bash
npx hyperframes lint
```

会出错的常见场景（基本都被前面 C1-C9 覆盖了）：
- `root_missing_composition_id` → 加 `data-composition-id`
- `root_missing_dimensions` → 加 `data-width` / `data-height`
- `media_missing_id` → `<audio>` 加 `id`
- `media_missing_src` → 用 `src` 不是 `data-src`
- `timed_element_missing_clip_class` → scene div 加 `class="scene clip"`
- `overlapping_clips_same_track` → 浮点对齐（见 C5）
- `font_family_without_font_face` → 注册 `@font-face`（见 C9）
- `gsap_exit_missing_hard_kill` → exit 动画末尾加 `tl.set({opacity:0}, {end})` 硬关

### V2. inspect 必须 0 issues

```bash
npx hyperframes inspect
```

inspect 会扫整个时间线的所有 sample。**它只看 HTML attribute（class/data-start/data-duration）决定 scene 是否可见，不看 GSAP 状态**。所以 inspect 通过 ≠ 视觉正确（见 C3）。

### V3. Playwright getComputedStyle 验证（关键）

这是 C3 的检测手段。`lint + inspect` 都过的 HTML，可能在浏览器里渲染出来是塌的。**必须**用 Playwright 起本地服务，加载 HTML，对代表性元素跑：

```js
// 1. 容器尺寸
const comp = document.querySelector('[data-composition-id]');
console.assert(comp.getBoundingClientRect().width === 1080);
console.assert(comp.getBoundingClientRect().height === 1920);

// 2. 关键 layout 选择器
const cloud = document.querySelector('.platform-cloud');
const cs = getComputedStyle(cloud);
console.assert(cs.display === 'grid', 'grid 布局没生效');

// 3. 字号
const pain = document.querySelector('.pain');
const fontSize = parseInt(getComputedStyle(pain).fontSize);
console.assert(fontSize >= 60, `pain 字号太小: ${fontSize}px`);

// 4. 每个 scene 的关键子元素都存在且非零
['s1', 's2', 's3', 's4', 's5'].forEach(id => {
  const inner = document.getElementById(id + '-inner');
  console.assert(inner, `scene ${id} inner not found`);
});
```

### V4. 抽帧肉眼检查（兜底）

```bash
# 抽 3 帧（S02 中段、核心金句、结尾）
ffmpeg -ss 9 -i renders/output.mp4 -frames:v 1 s02_check.png
ffmpeg -ss 16 -i renders/output.mp4 -frames:v 1 s03_check.png
ffmpeg -ss 28 -i renders/output.mp4 -frames:v 1 s05_check.png
```

打开图看：
- 文字是否清晰、不挤在角落
- 布局是否符合预期（4 列 grid / 卡片堆叠 / 大字金句）
- 是否有内容溢出/截断

### V5. ffprobe 验证最终 MP4

```bash
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,duration \
  -of default renders/output.mp4
```

确认：
- 同时有 `codec_type=video` 和 `codec_type=audio` 两条流
- `width=1080 height=1920`（9:16 竖屏）
- duration 与 `data-duration` 误差 ≤ 0.5s

---

## 完整 build_html.py 骨架（可复用）

```python
import json, subprocess

W, H = 1080, 1920
SCENE_IDS = ['s1', 's2', 's3', 's4', 's5']  # C3：单数字 ID

# 读 timeline
with open("audio/timeline.json", "r", encoding="utf-8") as f:
    timeline = json.load(f)
scenes = timeline["scenes"]
total = timeline["total_duration"]

def scene_attrs(idx, s):
    return f'class="scene clip" id="{SCENE_IDS[idx-1]}" ' \
           f'data-start="{s["start"]}" data-duration="{s["duration"]}" data-track-index="1"'

# 渲染各 scene 内容（render_s1 / render_s2 / ...）
# ...

# Audio 元素放在 body 顶层（C4）
audio_html = "\n".join([
    f'<audio id="narr-{SCENE_IDS[i].lower()}" src="audio/{s["filename"]}" '
    f'data-start="{s["start"]}" data-duration="{s["audio_dur"]}" '  # C6：用真实时长
    f'data-track-index="2" data-volume="1"></audio>'
    for i, s in enumerate(scenes)
])

# GSAP 时间线（C1, C2）
def build_timeline():
    lines = []
    for i, s in enumerate(scenes):
        sid = SCENE_IDS[i]
        start, end = s["start"], s["start"] + s["duration"]
        lines.append(
            f'  tl.fromTo("#{sid}-inner", {{opacity:0, scale:0.96}}, '
            f'{{opacity:1, scale:1, duration:0.5, ease:"power2.out", '
            f'overwrite:"auto", immediateRender:true}}, {start})'
        )
        if i < len(scenes) - 1:
            lines.append(f'  tl.to("#{sid}-inner", {{opacity:0, duration:0.4, '
                         f'ease:"power2.in", overwrite:"auto"}}, {end - 0.4})')
            lines.append(f'  tl.set("#{sid}-inner", {{opacity:0}}, {end})')
        else:
            lines.append(f'  tl.set("#{sid}-inner", {{opacity:1}}, {end})')
    return "\n".join(lines)

JS = f"""
window.addEventListener("load", function() {{
  window.__timelines = window.__timelines || {{}};
  const tl = gsap.timeline({{defaults:{{overwrite:"auto"}}}});
{build_timeline()}
  window.__timelines["main"] = tl;
}});
"""

# CSS 用同一份 SCENE_IDS（C3 终极预防）
css = "\n".join([
    f"#{sid} .my-class {{ ... }}" for sid in SCENE_IDS
])

# 组装 HTML
html = f"""<!DOCTYPE html>
<html><head>
<style>
@font-face {{ font-family: 'Body'; src: local('PingFang SC'); }}  /* C9 */
{css}
</style>
</head><body>
<div data-composition-id="main"
     data-width="{W}" data-height="{H}"
     data-start="0" data-duration="{total}">
{scene_html}
</div>

{audio_html}
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script>{JS}</script>
</body></html>"""
```

---

## 渲染故障速查表

| 症状 | 真因 | 修复 |
|---|---|---|
| 渲染报 "Composition has zero duration" | `DOMContentLoaded` 没用 `load`（C1） | 改 C1 |
| 报 "Composition has zero duration" 但 GSAP 看起来对 | `tl.fromTo` 缺 `immediateRender:true`（C2） | 加 C2 |
| 渲染出图但内容全挤屏幕中央 ~30% | ID 与 CSS 选择器不对齐（C3） | 改 ID / CSS 对齐，跑 V3 验证 |
| 渲染出图但没有声音 | `<audio>` 进了 composition 内部（C4） | 移到 `<body>` 顶层 |
| 渲染有声音但有片段缺失 | `data-duration` 用了 slot 时长而非 audio 时长（C6） | 改用 `audio_dur` |
| lint 报 `overlapping_clips_same_track` | 浮点累积（C5） | 强制对齐或下取整 0.1s |
| lint 报 `root_missing_*` | composition div 属性不全（C7） | 加 `data-composition-id` / `data-width` / `data-height` |
| 字体 fallback 到默认难看 | 没注册 `@font-face`（C9） | 加 C9 块 |
| 9:16 文字太挤 / 字号太小 | 用了 1920 横屏的 px 值（C10） | 全部按 1080 宽 9:16 重算 |
| 抖音/小红书首页刷不到合理缩略图 | 黑边过多或文字溢出 | 抽首帧用 V4 检查 |

---

## 相关 References

- `tts-workflow.md` — TTS 端到端 6 步流水线
- `external-tts.md` — MiniMax TTS API、速度校准、浮点重叠防御
- `wechat-article-video.md` — 微信文章场景的完整 Python 范本
- `design-md-spec.md` — 自定义主题的 design.md YAML 格式
