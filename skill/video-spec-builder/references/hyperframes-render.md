---
name: hyperframes-render
description: HyperFrames 渲染端 HTML 合成的硬性约束、踩坑清单、必做验证。从 video-spec.md 到 MP4 的最后一步，违反任何一条都可能让渲染静默失败或产出崩坏画面。
type: reference
---

# HyperFrames 渲染侧速查

video-spec.md 写完之后，要把它变成 MP4 / WebM，必须把分镜转成一个 HTML composition 喂给 HyperFrames 渲染。本文件是这个阶段所有"看似常识但实战必踩"的硬约束。

**什么时候读**：用户给出 video-spec.md 并问"怎么生成视频" / "下一步怎么走" / 直接说"渲染"时；也包括自动 / 半自动跑 build_html.py 的 agent。

---

## 🚨 第 0 条（最高优先级 · 渲染必须用官方 CLI）

**渲染就是 `npx hyperframes render`。没有第二种方式。**

### ❌ 绝对不要做的事

- **不要写自定义的 `render_video.js`（puppeteer-core 逐帧截图 + ffmpeg mux）**
- **不要写 Playwright 逐帧捕获脚本**
- **不要从老项目（sellersprite-skills-intro 等）拷它的 render_video.js**
- **不要因为"听起来更可控" / "更熟悉 puppeteer" 而绕开 CLI**

这些老脚本在 sellersprite 时期用过，是因为当时 CLI 还不成熟。**现在 CLI 已经是唯一正确的渲染入口**。自定义脚本的失败模式包括但不限于：
- 漏抓 GSAP 时间线第一帧（headless 启动时序问题）
- audio 元素与视频帧不同步
- scene visibility 逻辑跟 HyperFrames runtime 不一致
- 旁白声音丢失
- 没经过 `lint` / `inspect` 检查
- 无法利用 CLI 的 audio/CSS/font 规范化
- 维护负担：每开一个新项目都重写一遍

### ✅ 渲染的正确流程

```bash
# 1. lint + inspect 必须 0 errors
npx hyperframes lint
npx hyperframes inspect --timeout 30000

# 2. Playwright getComputedStyle 验证（V3，见下文）
python check_layout.py

# 3. 渲染（Windows 竖屏必须 --workers 1 防 OOM）
npx hyperframes render --workers 1 --fps 30 -o renders/<name>.mp4

# 4. ffprobe 验证
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,duration \
  -of default renders/<name>.mp4
```

如果 `npx hyperframes render` 失败，**修 HTML 让它过 lint**，不要绕开 CLI。这是 HyperFrames 设计契约的唯一正解。

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
  ↓  9. npx hyperframes render . --workers 1   ← 低内存环境(<4GB free)必须 `--workers 1`
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

### C7. composition 根属性必须齐全（最高频踩坑，渲染前 100% 检查）

> ⚠️ **写根 div 之前先抄下面这块模板，再改字段。不要从脑子里的记忆"复现"——你一定会漏字段。**

```html
<div data-composition-id="main"
     data-start="0"
     data-duration="<total_seconds>"
     data-width="1080"
     data-height="1920"
     data-fps="30">
```

**5 个属性一个都不能少**：

| 缺哪个 | lint 报什么 | 现象 |
|---|---|---|
| `data-composition-id` | `root_missing_composition_id` | 渲染器找不到入口 |
| `data-start` | `root_composition_missing_data_start` | 渲染器无法开始播放 |
| `data-duration` | `root_missing_duration` | 渲染器算不出总长 |
| `data-width` | `root_missing_dimensions` | 渲染直接失败 |
| `data-height` | `root_missing_dimensions` | 渲染直接失败 |

`data-duration` 与各 scene 的 `data-start + data-duration` 之和必须精确相等。

**实战防呆**：build_html.py / 写 HTML 时把这 5 个属性放在同一行，注释"// C7 五件套"，方便肉眼核对。例子：

```python
# build_html.py — C7 五件套，渲染前必检
root_attrs = (
    f'data-composition-id="main" '
    f'data-start="0" '                            # ← 必填
    f'data-duration="{total}" '                   # ← 必填
    f'data-width="{W}" '                          # ← 必填
    f'data-height="{H}" '                         # ← 必填
    f'data-fps="30"'                              # ← 可选但推荐
)
html = f'<div {root_attrs}>...'
```

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

### C10. `W, H` Python 常量必须和 CSS 里的 width/height 值严格一致

`build_html.py` 里 `W, H = 1080, 1920` 必须等于 CSS 里 `data-composition-id { width: 1080px; height: 1920px; }`。
不一致会导致 HyperFrames 的 data-* 属性与 CSS 布局打架，抽帧时出现"场景对调"(比如 S01 内容在 S03 时间窗口出现)。

```python
# build_html.py
W, H = 1080, 1920   # ← 改了这里必须改下面 CSS

CSS = f"""
[data-composition-id] {{ width: {W}px; height: {H}px; }}
"""
```

建议 CSS 也用 f-string 引用 `{W}`/`{H}` 变量，不要硬编码两套数字。但注意 CSS 里的 `{` 字符在 f-string 中要写成 `{{`。

### C11. 9:16 竖屏布局的硬规则

抖音/小红书/视频号/TikTok/Shorts 都是 9:16。CSS 里：
- **字号基线 ≥ 64px（老人家在手机上也看得清）；主标题 ≥ 100px；标签 ≥ 28px；正文用深色（≤ #333）不用灰色。24px 以下直接判不通过（手机等效 ~9px）。**
- 文字宽度收窄到画面 60%（max-width: 920px @ 1080-wide）
- padding-top/bottom 至少 120px
- grid/flex 子元素 max-width 不要写死 1920 横屏的数值

### C12. 绝不要在 CSS / JS 里手动控制 `.scene` 的 display / visibility（最高优先级 · 实战血泪）

> ⚠️ **HyperFrames runtime 已经根据每个 scene 的 `data-start` / `data-duration` 自动管 visibility。你只要写好时间线，不要去抢这个活。**

违反这条的失败模式（ai-agent-ch01 实战踩过，最严重的"静默失败"之一）：
- CSS 写 `.scene { display: none }` 当默认隐藏
- JS 写 `showSceneAt(t)` helper，按时间给当前 scene 设 `style.display = "block"`
- 看起来 preview 时一切正常（你给的 `tl.seek(t)` + `showSceneAt(t)` 双管齐下）
- **渲染时 HyperFrames renderer 只 seek 时间线，不会调你的 `showSceneAt`**
- 同时 inline `style.display='block'` 优先级 > CSS class > runtime 的 clip 类切换
- 结果：被 showSceneAt 设过的那个 scene 永远 inline display:block，其他 scene 永远 display:none
- **MP4 80% 时长全白或只有一帧内容**，lint 0 errors / inspect 0 issues / V3 还会误判通过

#### ❌ 绝对不要做的事

```css
/* ✗ 错：CSS 默认隐藏 */
.scene { display: none; }
```

```js
// ✗ 错：手动切换 display 的 helper
function showSceneAt(t) {
  const active = /* 算哪个 scene */;
  document.querySelectorAll('.scene').forEach(s => {
    s.style.display = (s.id === active) ? 'block' : 'none';
  });
}
// ✗ 错：preview / debug 时也调它
let previewTime = 5.0;
showSceneAt(previewTime);
tl.seek(previewTime, false);
```

#### ✅ 正确做法

CSS 不设 display（默认 block 即可），runtime 通过加/去 `clip` 类控制可见性：

```css
/* ✓ 对：不设 display */
.scene {
  position: absolute;
  inset: 0;
  overflow: hidden;
  background: var(--on-primary);
}
```

```js
// ✓ 对：preview / debug 时只 seek 时间线，runtime 自己会管 visibility
let previewTime = 5.0;
tl.seek(previewTime, false);
```

**为什么 V3 检测不到**：常见的 `check_layout.py` 写法是手动 forEach 每个 scene `style.display='block'` 再 getComputedStyle —— 这一动作**恰好屏蔽了 bug**（强制 display:block 代替了 runtime 的可见性逻辑）。要真正检测，必须用 V3 里的 **MD5 帧唯一性检查**（见 V3 末尾新增小节）。

**HyperFrames 官方契约原话**（来自 `hyperframes-core`）：
> "HyperFrames already shows/hides clips based on data-start and data-duration. Animating display/visibility on the clip itself races with the framework's own show/hide."

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

⚠️ **inspect 报文字溢出(overflowed bottom Npx)**：竖屏 1080×1920 下，横屏继承来的大字号可能在 flex 居中布局里 line-box 总和超过容器高。这通常不是视觉 bug（overflow 被 hidden/clip 兜住了），但 inspect 保守报 error。修复参考值：hero-title 104→96px, kinetic-quote 80→72px, info-num 88→76px, showcase-title 64→58px, showcase-img max-height 760→680px。

### V3. Playwright getComputedStyle 验证（关键）

这是 C3 的检测手段。`lint + inspect` 都过的 HTML，可能在浏览器里渲染出来是塌的。**必须**用 Playwright 起本地服务，加载 HTML，对代表性元素跑。

**⚠️ 必须用系统 Chrome / Edge，不要 `playwright install chromium`**（~200MB 下载，纯浪费）。这台机器已装 Chrome，0 下载走 `executable_path` 路线。

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # ✓ 用系统 Chrome，不下载 chromium
    b = p.chromium.launch(
        executable_path='C:/Program Files/Google/Chrome/Application/chrome.exe',
        args=['--no-sandbox']
    )
    pg = b.new_page(viewport={"width": 1920, "height": 1080})  # 16:9
    # 9:16 视频改 viewport={"width": 1080, "height": 1920}
    pg.goto("http://localhost:8000/index.html")
    pg.wait_for_load_state("networkidle")

    # 1. 容器尺寸
    cs = pg.evaluate("""() => {
        const c = document.querySelector('[data-composition-id]');
        const r = c.getBoundingClientRect();
        return {w: r.width, h: r.height};
    }""")
    assert cs['w'] == 1920 and cs['h'] == 1080, f"canvas wrong: {cs}"

    # 2. 关键 layout 选择器
    cloud = pg.locator('.platform-cloud').first
    if cloud.count() > 0:
        disp = pg.evaluate("getComputedStyle(document.querySelector('.platform-cloud')).display")
        assert disp == 'grid', f'grid 布局没生效: {disp}'
        cols = pg.evaluate("getComputedStyle(document.querySelector('.platform-cloud')).gridTemplateColumns")
        assert cols != 'none', f'grid columns missing: {cols}'

    # 3. 字号
    pain = pg.locator('.pain').first
    if pain.count() > 0:
        fs = pg.evaluate("parseInt(getComputedStyle(document.querySelector('.pain')).fontSize)")
        assert fs >= 60, f'pain 字号太小: {fs}px'

    # 4. 每个 scene 的 inner 都存在
    for sid in ['s1', 's2', 's3', 's4', 's5']:
        inner = pg.locator(f"#{sid}-inner")
        assert inner.count() == 1, f"scene {sid} inner not found"

    # 5. 抽帧
    pg.screenshot(path='check_first_frame.png')
    b.close()
print("V3 PASSED")
```

**Chrome / Edge 路径回退顺序**（先 `where chrome.exe` / `where msedge.exe` 确认）：

| 浏览器 | 路径 |
|---|---|
| Chrome (优先) | `C:/Program Files/Google/Chrome/Application/chrome.exe` |
| Edge | `C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe` |

**两步都没找到再问用户怎么办，绝对不要直接 `playwright install`**。

#### V3.5 — MD5 帧唯一性检查（必加，专治"只有一个 scene 可见"静默失败）

光跑 getComputedStyle 不够 —— 传统的 check_layout.py 写法会手动 forEach 每个 scene `style.display='block'`，**这一动作恰好屏蔽了 C12 的 bug**（强制 inline display 代替了 runtime 的可见性逻辑）。要在 V3 之后追加一次"不预设 display，纯靠 timeline seek"的截图比对，每个 scene 中段抽一帧，验 MD5 全不同。

```python
# 紧跟在 V3 主流程后面，不要 forEach display
import hashlib

scene_mids = [
    ("s1", 2.5), ("s2", 8.0), ("s3", 14.0), ("s4", 22.0),
    ("s5", 30.0), ("s6", 36.0), ("s7", 50.0), ("s8", 65.0),
    ("s9", 75.0), ("s10", 84.0),
]
hashes = {}
for sid, t in scene_mids:
    # 关键：只 seek，不动 .scene 的 display
    pg.evaluate(f"window.__timelines['main'].seek({t}, false)")
    pg.wait_for_timeout(300)
    png_bytes = pg.screenshot()
    hashes[sid] = hashlib.md5(png_bytes).hexdigest()

# 所有 hash 必须互不相同；相邻 scene 之间相同 = 那段时间里画面没变 = 静默失败
unique = len(set(hashes.values()))
assert unique == len(scene_mids), (
    f"C12 regression: only {unique}/{len(scene_mids)} unique frames. "
    f"Most likely cause: CSS .scene {{display:none}} + JS showSceneAt 强制 inline display. "
    f"hashes: {hashes}"
)
```

如果这一步失败：直接看 index.html 是否有 `.scene { display: none }` 或 `showSceneAt` 函数，删干净再 re-render。这一步是 C12 的检测手段，**也是 C12 bug 唯一能被机器发现的途径**。

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
| 渲染 OOM(malloc of size failed) + 竖屏 1080×1920 | 多 worker Chrome 进程吃满 ~2.4GB free 内存 | `--workers 1` 减小并发内存峰值 |
| 抽帧场景对调(S01 内容出现在 S03 时间) | Python `W,H` 常量与 CSS width/height 不一致（C10） | 确保 `W,H=1080,1920` 和 CSS `width:1080px;height:1920px` 严格一致 |
| inspect 报文字 overflowed bottom | 横屏大字号在竖屏 flex 居中布局里 line-box 超出容器（V2） | 参考值微缩:hero-title 104→96, kinetic-quote 80→72, info-num 88→76 |
| 渲染出图但没有声音 | `<audio>` 进了 composition 内部（C4） | 移到 `<body>` 顶层 |
| 渲染有声音但有片段缺失 | `data-duration` 用了 slot 时长而非 audio 时长（C6） | 改用 `audio_dur` |
| lint 报 `overlapping_clips_same_track` | 浮点累积（C5） | 强制对齐或下取整 0.1s |
| lint 报 `root_missing_*` | composition div 属性不全（C7） | 加 `data-composition-id` / `data-width` / `data-height` |
| 字体 fallback 到默认难看 | 没注册 `@font-face`（C9） | 加 C9 块 |
| 9:16 文字太挤 / 字号太小 | 用了 1920 横屏的 px 值（C11） | 全部按 1080 宽 9:16 重算 |
| 抖音/小红书首页刷不到合理缩略图 | 黑边过多或文字溢出 | 抽首帧用 V4 检查 |
| **MP4 大部分全白 / 只有一个 scene 内容** | CSS `.scene { display:none }` + JS `showSceneAt(t)` helper 给某 scene 设 inline `style.display='block'`，渲染时 runtime 切 visibility 被 inline override（C12） | 删 CSS 的 `display:none` + 删 `showSceneAt`，让 runtime 用 `clip` 类自管 visibility；用 V3.5 MD5 帧唯一性检查兜底 |

---

## 相关 References

- `tts-workflow.md` — TTS 端到端 6 步流水线
- `external-tts.md` — MiniMax TTS API、速度校准、浮点重叠防御
- `wechat-article-video.md` — 微信文章场景的完整 Python 范本
- `design-md-spec.md` — 自定义主题的 design.md YAML 格式
