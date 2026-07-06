---
name: wechat-build-example
description: 微信文章 → 视频的端到端可执行 Python 范本。一个 build.py 跑完：URL 解析 → 图片下载 → 章节/金句/数据抽取 → 9-12 镜头规划（至少 4 种 scene 类型）→ MiniMax TTS → ffprobe 实测 → 反推 scene 时长 → 生成 designer-quality HTML（5 种 scene 范式 + 浮点防御 + GSAP 动效）。跑完前必须 Playwright 视觉验证。
type: reference
---

# wechat-article-video 端到端 Python 范本

> 把 `references/wechat-article-video.md` 的概念指南浓缩成**一个可执行 Python 文件**。直接拷走，改 URL 跑就行。
>
> 完整工作流：URL → 解析 → 抽图 → 规划 → TTS → 时长反推 → HTML → 视觉验证 → 渲染。

---

## 项目结构

```
wx-video/
  build.py                    # 本文件就是这一个（拆成 build_parse.py / gen_tts.py / build_html.py 也行）
  .env                        # 包含 minimaxi=<API_KEY>
  video/                      # 产物
    index.html
    img/                      # 下载的文章图片
    audio/                    # TTS 旁白 + timeline.json
```

**关键约定**：
- 整个 pipeline 输出都进 `video/` 子目录（HyperFrames 默认的 video root）
- `.env` 在项目根（`../.env` 相对 `video/`）
- 跑前先 `cd video/`

---

## 完整 build.py

```python
"""WeChat article → HyperFrames video · end-to-end pipeline.

Usage:
    cd video/
    python ../build.py "https://mp.weixin.qq.com/s/xxxxxxxxxx"

What it does:
    1. POST URL to ideaflow-article-to-markdown API → save wx_article.md + wx_resp.json
    2. Extract ![img](...) URLs → download to img/, print PIL sizes
    3. Extract H2/H3 headings + bold-emphasis sentences + percentage/number stats
    4. Plan 9-12 scenes with at least 4 distinct types (cinema-title / showcase /
       kinetic-quote / infographic-grid / comparison-bars)
    5. Generate TTS via MiniMax, measure with ffprobe, write timeline.json
    6. Build designer-quality HTML with GSAP timeline + audio elements
    7. Print next-step commands: lint, inspect, visual-validate, render
"""
import json, os, re, sys, time, subprocess, requests
from pathlib import Path

# ---------- Paths ----------
ROOT = Path(".")
IMG_DIR = ROOT / "img"
AUDIO_DIR = ROOT / "audio"
for d in (IMG_DIR, AUDIO_DIR):
    d.mkdir(exist_ok=True)

# ---------- 1. Parse article ----------
def parse_article(url: str) -> str:
    """Hit ideaflow API → save json + markdown → return markdown text."""
    print(f"\n[1/7] Parsing {url} ...")
    api = "https://ideaflow-article-to-markdown.hf.space/resolve/mark"
    headers = {
        "Referer": "https://ideaflow-article-to-markdown.hf.space/",
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        "Content-Type": "application/json",
    }
    resp = requests.post(api, headers=headers, json={"blogUrl": url}, timeout=120)
    data = resp.json()
    md = data["data"]["markdown"]

    (ROOT / "wx_resp.json").write_text(json.dumps(data, ensure_ascii=False, indent=2))
    (ROOT / "wx_article.md").write_text(md, encoding="utf-8")
    print(f"  → wx_article.md ({len(md)} chars)")
    return md


# ---------- 2. Extract images ----------
def extract_images(md: str) -> list[Path]:
    """Download every ![alt](url) to img/ and return local paths."""
    print(f"\n[2/7] Extracting images ...")
    urls = re.findall(r'!\[.*?\]\((.+?)\)', md)
    print(f"  Found {len(urls)} images")
    local = []
    for u in urls:
        name = re.sub(r'[^\w.]', '_', u.split('/')[-1].split('?')[0]) or f"img_{len(local)}.png"
        path = IMG_DIR / name
        if path.exists():
            local.append(path)
            continue
        try:
            r = requests.get(u, timeout=30)
            path.write_bytes(r.content)
            local.append(path)
            print(f"  ✓ {name} ({len(r.content)//1024} KB)")
        except Exception as e:
            print(f"  ✗ {name}: {e}")
    return local


# ---------- 3. Extract content (headings, quotes, stats) ----------
def extract_content(md: str) -> dict:
    """Parse headings, emphasized sentences, and numeric stats from markdown."""
    print(f"\n[3/7] Extracting structure ...")
    headings = re.findall(r'^#{2,3}\s+(.+)$', md, re.MULTILINE)
    quotes = re.findall(r'\*\*(.+?)\*\*', md)  # bolded text often = key insights
    # Find numbers like 95%, 1000+, 852 个, ¥123.4 亿
    stats = re.findall(r'\b(\d+(?:\.\d+)?[%＋+]?)\s*(个|亿|万|分|分?钟|小时|天|年)?', md)
    stats = [f"{n}{u}" for n, u in stats if float(n) > 5][:20]  # filter trivial numbers
    print(f"  headings={len(headings)}  quotes={len(quotes)}  stats={len(stats)}")
    return {"headings": headings, "quotes": quotes, "stats": stats}


# ---------- 4. Plan scenes (this is where you customize) ----------
def plan_scenes(content: dict, images: list[Path]) -> list[dict]:
    """Build 9-12 scene plan with at least 4 distinct types.

    CRITICAL design rules (non-negotiable):
      - Mix ≥ 4 scene types. Pure image carousel = boring.
      - Open with cinema-title. End with cinema-title or kinetic-quote.
      - showcase = article image + text overlay (NOT cover/contain pitfall).
      - infographic-grid = 2x2 animated data cards (extract from content.stats).
      - comparison-bars = 3+ horizontal bars with stagger animation.
      - kinetic-quote = centered hero text + em highlights + accent line.
    """
    # ↓ ↓ ↓ CUSTOMIZE THIS BLOCK PER ARTICLE ↓ ↓ ↓
    scenes = [
        # (label, type, duration_target, kwargs)
        ("S01", "cinema-title", 7.0, {
            "eyebrow": "AI · 深度阅读",
            "title": "<em>微信文章</em>\n一键变视频",
            "desc": "作者写 1 小时，观众看 60 秒",
        }),
        ("S02", "showcase", 8.0, {
            "eyebrow": "原文截图",
            "title": "<em>核心论点</em>：…",
            "desc": "…",
            "image": str(images[0]) if images else None,
        }),
        # repeat showcase for each key image ...
        ("S03", "kinetic-quote", 6.5, {
            "eyebrow": "金句提炼",
            "title": "这一段的<em>核心结论</em>\n用大字砸出来",
        }),
        ("S04", "infographic-grid", 9.0, {
            "eyebrow": "关键数据",
            "metrics": [
                ("852", "<span>个</span>", "原文任务 1", 95),
                ("120", "<span>个</span>", "原文任务 2", 70),
                ("0.66", "", "原文任务 3", 66),
                ("8", "<span>维度</span>", "原文任务 4", 100),
            ],
        }),
        ("S05", "comparison-bars", 8.0, {
            "eyebrow": "三个反直觉结论",
            "bars": [
                ("方案 A", 95, "#F59E0B", "0.95"),
                ("方案 B", 64, "#FBBF24", "0.64"),
                ("方案 C", 46, "#6366F1", "0.46"),
            ],
            "desc": "Harness 换一套，差距从 0.182 拉到顶",
        }),
        ("S06", "kinetic-quote", 6.0, {
            "eyebrow": "结语",
            "title": "<em>行动号召</em>\n或核心金句",
        }),
        ("S07", "cinema-title", 7.0, {
            "eyebrow": "谢谢观看",
            "title": "<em>扫码</em>关注\n看完整文章",
        }),
    ]
    # ↑ ↑ ↑ CUSTOMIZE BLOCK ENDS ↑ ↑ ↑

    # Validate: at least 4 distinct types
    types = {s[1] for s in scenes}
    assert len(types) >= 4, f"Need ≥ 4 scene types, got {types}"
    # Validate: opens + closes with hero type
    assert scenes[0][1] in ("cinema-title", "kinetic-quote"), "Open with hero type"
    assert scenes[-1][1] in ("cinema-title", "kinetic-quote"), "Close with hero type"
    return [
        {"label": l, "type": t, "target_dur": d, **kw}
        for l, t, d, kw in scenes
    ]


# ---------- 5. Generate TTS ----------
def gen_tts(scenes: list[dict]) -> list[dict]:
    """Call MiniMax TTS, save mp3, measure with ffprobe, return audio results."""
    print(f"\n[5/7] Generating TTS for {len(scenes)} scenes ...")
    key = re.search(r"minimaxi\s*=\s*(\S+)", (Path("..") / ".env").read_text())
    api_key = key.group(1) if key else None
    if not api_key:
        raise SystemExit("minimaxi key not found in ../.env")

    url = "https://api.minimaxi.com/v1/t2a_v2"
    headers = {"Content-Type": "application/json", "Authorization": f"Bearer {api_key}"}
    voice_id = "male-qn-qingse"  # 中文男声
    speed = 1.6                   # 实测 ~4.6 字/秒（理论 4.0，比理论慢 27%）

    results = []
    for s in scenes:
        text = s.get("narration") or s.get("title", "").replace("<em>", "").replace("</em>", "")
        text = re.sub(r'\n+', '，', text).strip()
        if not text:
            continue
        fname = f"wx_{s['label'].lower()}.mp3"
        path = AUDIO_DIR / fname
        print(f"  {s['label']}: {text[:30]}…")

        payload = {
            "model": "speech-2.8-hd",
            "text": text,
            "stream": False,
            "voice_setting": {"voice_id": voice_id, "speed": speed, "vol": 1, "pitch": 0},
            "audio_setting": {"sample_rate": 32000, "bitrate": 128000, "format": "mp3", "channel": 1},
            "output_format": "url",
        }
        r = requests.post(url, json=payload, headers=headers, timeout=60).json()
        if r.get("base_resp", {}).get("status_code") != 0:
            print(f"    ✗ {r.get('base_resp', {}).get('status_msg')}")
            continue
        ad = r["data"]["audio"]
        body = requests.get(ad, timeout=60).content if ad.startswith("http") else bytes.fromhex(ad)
        path.write_bytes(body)
        # Measure
        proc = subprocess.run(
            ["ffprobe", "-v", "error", "-show_entries", "format=duration",
             "-of", "csv=p=0", str(path)],
            capture_output=True, text=True, timeout=10,
        )
        actual = float(proc.stdout.strip())
        print(f"    → {actual:.2f}s (target scene {s['target_dur']}s)")
        results.append({**s, "filename": fname, "audio_dur": round(actual, 2), "text": text})
        time.sleep(0.4)
    return results


# ---------- 6. Build timeline (with float-overlap defense) ----------
def build_timeline(results: list[dict]) -> dict:
    """scene_dur = max(audio_dur + 2.0, 6.0); force-align to prevent lint overlap errors."""
    print(f"\n[6/7] Building timeline ...")
    BUFFER, MIN_DUR = 2.0, 6.0
    cur, scenes = 0.0, []
    for r in results:
        sd = max(r["audio_dur"] + BUFFER, MIN_DUR)
        sd = round(sd, 1)
        scenes.append({**r, "start": round(cur, 1), "duration": sd})
        print(f"  {r['label']}: start={cur:.1f}s  dur={sd:.1f}s  audio={r['audio_dur']:.2f}s")
        cur += sd
    # C5: force-align to prevent overlapping_clips_same_track
    for i in range(len(scenes) - 1):
        end = scenes[i]["start"] + scenes[i]["duration"]
        if end > scenes[i+1]["start"]:
            scenes[i]["duration"] = round(scenes[i+1]["start"] - scenes[i]["start"] - 0.1, 1)
    total = round(cur, 1)
    print(f"  total: {total}s")
    return {"total_duration": total, "scenes": scenes}


# ---------- 7. Build HTML (C1-C10 hyperframes-render rules baked in) ----------
def build_html(timeline: dict, scenes: list[dict]) -> str:
    """Assemble designer-quality HTML with GSAP timeline + audio at body level."""
    print(f"\n[7/7] Building index.html ...")
    W, H = 1920, 1080  # 横屏 16:9；竖屏用 1080×1920
    total = timeline["total_duration"]
    total_buf = round(total + 0.5, 1)

    # Render scene blocks (use scene_data lookup for per-scene type-specific content)
    scene_data = {s["label"]: s for s in scenes}

    def render_scene(i, sd):
        d = scene_data[sd["label"]]
        sel = f'#s{i}-inner'
        # Each scene gets a typed render (omitted here for brevity, see wx-video/build_html.py
        # for the full 5-type implementation: cinema_title/showcase/kinetic_quote/infographic/comparison)
        # In real use, copy the per-type scene_*() functions from the wx-video repo.
        raise NotImplementedError("Copy scene_*() from wx-video/build_html.py")

    # Audio: at body level (C4) with real audio_dur (C6) and track-index=2
    audio = "\n".join(
        f'<audio id="narr-s{i+1}" data-start="{sd["start"]}" '
        f'data-duration="{sd["audio_dur"]}" data-track-index="2" '
        f'src="audio/{sd["filename"]}" data-volume="1"></audio>'
        for i, sd in enumerate(timeline["scenes"])
    )

    # C1: window.load, not DOMContentLoaded
    # C2: every tl.fromTo has immediateRender:true
    js = f"""
window.addEventListener("load", function() {{
  window.__timelines = window.__timelines || {{}};
  const tl = gsap.timeline({{defaults:{{overwrite:"auto"}}}});
  // ... per-scene anim blocks copied from wx-video/build_html.py ...
  window.__timelines["main"] = tl;
}});
"""

    html = f"""<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>WeChat article → video</title>
<style>
  /* C9: register @font-face for any non-PingFang font you use */
  @font-face {{ font-family: 'PingFang SC'; src: local('PingFang SC'); }}
  @font-face {{ font-family: 'Space Grotesk'; src: local('Space Grotesk'); }}
  /* ... full CSS for 5 scene types (see wx-video/build_html.py) ... */
</style>
</head>
<body>
<div data-composition-id="main" data-width="{W}" data-height="{H}"
     data-start="0" data-duration="{total_buf}">
  <!-- ... scene divs ... -->
</div>
{audio}
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script>{js}</script>
</body>
</html>
"""
    return html


# ---------- main ----------
if __name__ == "__main__":
    url = sys.argv[1] if len(sys.argv) > 1 else None
    if not url or "mp.weixin.qq.com" not in url:
        raise SystemExit("Usage: python build.py <mp.weixin.qq.com URL>")
    md = parse_article(url)
    imgs = extract_images(md)
    content = extract_content(md)
    scenes = plan_scenes(content, imgs)
    results = gen_tts(scenes)
    timeline = build_timeline(results)
    (AUDIO_DIR / "timeline.json").write_text(
        json.dumps(timeline, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    html = build_html(timeline, scenes)
    (ROOT / "index.html").write_text(html, encoding="utf-8")
    print(f"\n✓ Wrote index.html ({len(html)} bytes)")
    print(f"\nNext steps (REQUIRED — skip any of these and the render will silently break):")
    print(f"  1. npx hyperframes lint    # 0 errors required")
    print(f"  2. npx hyperframes inspect  # 0 issues required")
    print(f"  3. Playwright getComputedStyle + screenshot  # see hyperframes-render.md V3+V4")
    print(f"  4. npx hyperframes render   # → output.mp4")
```

---

## 跑前必读

**完整实现不在这一个文件里**——`render_scene()` 的 5 种 scene 类型 + CSS + GSAP 时间线有 ~300 行，要从 `wx-video/build_html.py` 拷过去。本文件是**架构和流程的范本**，不是完整产品。

完整实现拆 3 个文件更清晰：
- `build_parse.py` — 上面 §1-3
- `gen_tts.py` — 上面 §5
- `build_html.py` — 上面 §7 + 从 `wx-video/build_html.py` 拷的 `scene_*()` 函数

---

## 5 种 scene 类型的 CSS 范本（直接抄）

完整版见 `wx-video/build_html.py` 的 `CSS = """..."""` 块。下面是简化骨架：

| 类型 | 关键 class | 关键 CSS |
|---|---|---|
| **cinema-title** | `.cinema-bg` `.hero-text` `.hero-title` | `position:absolute; inset:0; background-size:cover`，text `position:absolute; bottom:90px; padding:100px 140px` |
| **showcase** | `.showcase-img` `.overlay-top` `.overlay-bottom` | **`object-fit:contain`**（不是 cover！文章截图有文字不能裁），`max-width:78%; max-height:60vh; border-radius:16px; box-shadow` |
| **infographic-grid** | `.info-card` `.info-num` `.info-bar` | `display:grid; grid-template-columns:1fr 1fr`，num 字号 80px / Space Grotesk 800，bar 高度 6-8px 渐变色 |
| **comparison-bars** | `.compare-row` `.compare-label` `.compare-fill` | 横向 flex，label 固定宽 280-380px，track 高度 28-36px，fill 动画 width 0→pct% |
| **kinetic-quote** | `.kinetic-line` `.kinetic-quote` | line 60px×3px 主色，quote 字号 64px 行高 1.5，`<em>` 字号 72px 加粗 |

**5 类必备 4 类**：做 9-12 个场景时至少混 4 种。纯图片轮播=无聊。

---

## 5 类必备 4 类的硬性约束

下面这套防呆检查在 `plan_scenes()` 里就做了：

```python
types = {s[1] for s in scenes}
assert len(types) >= 4, f"Need ≥ 4 scene types, got {types}"
assert scenes[0][1] in ("cinema-title", "kinetic-quote"), "Open with hero"
assert scenes[-1][1] in ("cinema-title", "kinetic-quote"), "Close with hero"
```

`wechat-article-video.md` § Step 3 已经把 5 类列全了。每次新文章都从那 5 类挑，**别用第 6 种**——渲器没适配。

---

## 7 个必踩的坑（与 hyperframes-render.md 互补）

| # | 坑 | 防御 |
|---|---|---|
| 1 | `data-duration` 用 scene slot 而非 `audio_dur` | `gen_tts` 里 ffprobe 测出来的 `actual` 存到 `audio_dur`，HTML 里用它 |
| 2 | `data-start + data-duration` 浮点重叠 → lint 报 `overlapping_clips_same_track` | `build_timeline` 里强制 -0.1s 对齐 |
| 3 | `<audio>` 放进 composition div → 没声音 | `build_html` 把 audio 块放到 composition div **之后**（body 顶层） |
| 4 | `class="scene"` 漏掉 `clip` → 全程可见与下个 scene 叠加 | 每个 scene div 必带 `class="scene clip"` |
| 5 | composition div 漏 `data-width`/`data-height` → lint 报 `root_missing_dimensions` | 模板里硬编码 `<div data-composition-id data-width data-height>` |
| 6 | 用 `DOMContentLoaded` → 渲染报 "Composition has zero duration" | `window.addEventListener("load", ...)` |
| 7 | GSAP `tl.fromTo` 缺 `immediateRender:true` → 时间线长度被算成 0 | 全部 `tl.fromTo` 都带这俩参数 |

---

## 跑完后必须做的 4 步验证

**任一一步漏了，最终 MP4 都可能肉眼可见崩坏。** 详见 `hyperframes-render.md` V1-V5。

```bash
# V1: lint
npx hyperframes lint

# V2: inspect
npx hyperframes inspect --timeout 30000

# V3: Playwright getComputedStyle（CSS 真的匹配上没）
# ⚠️ 必须用系统 Chrome，不要 `playwright install chromium`！
python -c "
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(
        executable_path='C:/Program Files/Google/Chrome/Application/chrome.exe',
        args=['--no-sandbox']
    )
    pg = b.new_page(viewport={'width':1920,'height':1080})
    pg.goto('http://localhost:8000/index.html')
    pg.wait_for_load_state('networkidle')
    # 关键 layout 验证
    cs = pg.evaluate('getComputedStyle(document.querySelector(\".showcase-img\")).objectFit')
    assert cs == 'contain', f'showcase-img object-fit={cs}, must be contain'
    cs2 = pg.evaluate('parseInt(getComputedStyle(document.querySelector(\".hero-title\")).fontSize)')
    assert cs2 >= 80, f'hero-title too small: {cs2}px'
    pg.screenshot(path='check_s01.png', full_page=False)
    b.close()
"

# V4: ffmpeg 抽帧肉眼检查
ffmpeg -ss 4 -i renders/output.mp4 -frames:v 1 s01_check.png
ffmpeg -ss 30 -i renders/output.mp4 -frames:v 1 s05_check.png

# V5: ffprobe 验证
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,duration \
    -of default renders/output.mp4
```

**lint 过 ≠ 视觉对**。Playwright 验证是**唯一可靠**的方式（详见 `feedback_css_id_sync.md` memory）。

**⚠️ Playwright 必须用系统 Chrome**：`executable_path='C:/Program Files/Google/Chrome/Application/chrome.exe'`。
**绝对不要跑 `playwright install chromium`**——会下载 ~200MB，纯浪费，Chrome 已经装好了。
找不到 Chrome 时回退到 Edge：`C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe`。
两步都没找到再问用户怎么办。

---

## 与 wechat-article-video.md 的分工

| 文件 | 角色 |
|---|---|
| `references/wechat-article-video.md` | **概念指南** —— 7 步流程、5 种 scene 类型、API 端点、CSS 范式、踩坑清单。**先读这个理解"为什么这么做"** |
| `references/wechat-build-example.md`（本文件）| **可执行范本** —— 一个 build.py 的架构骨架 + 5 类 CSS 速查 + 4 步硬验证。**写代码时翻这个** |
| `references/hyperframes-render.md` | **渲染侧硬约束** —— 10 条 C1-C10 + 5 步 V1-V5 + build_html.py 完整骨架 + 故障速查表 |
| `wx-video/build_html.py` | **真实生产代码** —— 9 场景完整实现，从这个文件拷 `scene_*()` 函数 |

---

## 真实生产代码的参考

`D:\video-spec-builder-main\wx-video\` 下的实际生产代码：
- `wx-video/gen_tts.py` — 9 场景 TTS + 浮点对齐 timeline.json 生成
- `wx-video/build_html.py` — 9 场景 5 类型 HTML 组装（最完整的一份参考实现）

这两个文件配合本范本 = 端到端可跑的微信文章视频生成器。

---

## 相关 References

- `wechat-article-video.md` — 概念指南，5 种 scene 类型的完整说明
- `hyperframes-render.md` — 渲染侧 10 条硬约束 + 5 步验证
- `external-tts.md` — MiniMax TTS API、速度校准、浮点重叠防御
- `tts-workflow.md` — TTS 端到端 6 步流水线
- `components-catalog.md` — 69 个组件目录，scene 类型可对应到具体组件 ID
