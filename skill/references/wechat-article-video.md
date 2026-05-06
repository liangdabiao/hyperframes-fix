---
name: wechat-article-video
description: Parse WeChat public account articles (mp.weixin.qq.com) via API, extract content and images, then produce HyperFrames video compositions with cinematic design, image showcase (complete, no crop), text overlays, animated infographics, kinetic typography, and comparison charts. Covers API parsing, image downloading, narration generation, scene planning, designer-quality composition, and validation.
---

# WeChat Article → Video Workflow

Automated pipeline for converting a WeChat public account article into a HyperFrames video composition. Covers article parsing, image extraction, content analysis, scene design, TTS narration, and designer-quality HTML composition output.

## Trigger

When the user provides a URL starting with `https://mp.weixin.qq.com`, follow this workflow end-to-end.

## Step 1: Parse Article via API

Call the ideaflow API to convert the WeChat article to markdown:

```bash
curl -s "https://ideaflow-article-to-markdown.hf.space/resolve/mark" \
  -H "Referer: https://ideaflow-article-to-markdown.hf.space/" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -H "Content-Type: application/json" \
  -d '{"blogUrl":"<USER_URL>"}' \
  -o wx_resp.json
```

Parse the JSON response:

```python
import json, re, os
with open("wx_resp.json", "r", encoding="utf-8") as f:
    data = json.load(f)
md = data["data"]["markdown"]
```

Save the markdown for reference: `wx_article.md`.

## Step 2: Extract Images and Content Structure

From the parsed markdown, extract:

1. **Images**: all `![alt](url)` references — article screenshots/illustrations
2. **Headings**: `## ` or `### ` lines — section structure
3. **Key quotes**: impactful sentences for quote cards
4. **Data points**: statistics, percentages, comparisons for infographics

```python
imgs = re.findall(r'!\[.*?\]\((.+?)\)', md)
headings = re.findall(r'^#{2,3}\s+(.+)$', md, re.MULTILINE)
```

Check image dimensions with PIL to understand aspect ratios before planning scenes:

```python
from PIL import Image
for url in imgs:
    name = os.path.basename(url)
    im = Image.open(f"img/{name}")
    print(f"{name}: {im.size[0]}x{im.size[1]}")
```

## Step 3: Plan Scene Types

**CRITICAL: Never use pure image carousel.** Mix at least 4 different scene types. Think like a designer, not a developer.

### Design Principles (Non-Negotiable)

1. **Images must be shown COMPLETELY** — use `object-fit:contain`, never `object-fit:cover`. Article screenshots contain text that becomes unreadable when cropped.
2. **Text overlays ON images** — don't separate text and image into different areas. Overlay title text on top/bottom of the image with `text-shadow` for readability.
3. **Dynamic infographics** — use animated progress bars, counting numbers, comparison bar charts. Static numbers are boring; animate them.
4. **Cinematic feel** — dark backgrounds, subtle gradients, bottom-aligned hero text, pill tags, large shadows on image cards.
5. **No rigid grids** — vary the visual weight across scenes. Some scenes are image-heavy, some are text-heavy, some are data-heavy.

### Scene Type Catalog

| Type | Visual | Best for |
|:---|:---|:---|
| **cinema-title** | Pexels BG + bottom-aligned hero text + pill tags | Opening title, closing message |
| **showcase** | Complete image centered with rounded corners + shadow + text overlay top/bottom | Article screenshots, comparison images |
| **infographic-grid** | 2x2 or 1x3 grid of animated data cards with progress bars | Key metrics, stats, comparisons |
| **comparison-bars** | Horizontal bar chart with 3+ items, animated fill | Model benchmarks, score comparisons |
| **kinetic-quote** | Centered large text + accent line + em highlights | Key insights, conclusions, powerful quotes |

### Scene Planning Rules

1. **Open with cinema-title** — eyebrow tag + large title + description + pill tags
2. **Use showcase for article images** — most images should be this type, with overlay text
3. **Add 1-2 infographic scenes** — extract data from the article (percentages, scores, counts)
4. **Add 1 comparison scene** — if the article compares things (models, products, methods)
5. **End with kinetic-quote or cinema-title** — powerful closing statement
6. **Scene count**: 10-14 scenes for a typical article
7. **Never repeat same type consecutively**

### Example Scene Sequence

```
cinema-title → showcase (overview) → showcase (benchmark) → showcase (test result)
→ showcase (detail) → infographic-grid (4 key metrics) → comparison-bars (model scores)
→ kinetic-quote (key insight) → cinema-title (outro)
```

## Step 4: Write Narration

Split article content into narration segments matching planned scenes:

- **Title scene**: 1-2 sentences summarizing the article topic
- **Content scenes**: Extract key points, paraphrase concisely
- **Outro**: Call-to-action or closing thought
- **Text density**: follow tts-workflow.md guidelines (60-70% of scene duration at TTS speed 1.6)

Generate audio via MiniMax TTS API (see `external-tts.md` for full details).

## Step 5: Calculate Timing

**CRITICAL: Total duration MUST be ≥ last audio end time.** This is the #1 source of black-screen bugs.

```python
# After all audio generated:
last_audio_end = max(audio_start + audio_dur for all audio clips)
last_scene_end = max(scene_start + scene_dur for all scenes)
TOTAL_DUR = max(last_audio_end, last_scene_end) + 1.0  # +1s buffer
```

If audio exceeds visuals by a lot, extend the outro scene duration to fill the gap.

```
Scene duration: typically 7-10 seconds per showcase, 9-12 for infographic
Transition gap: 0.4s between scenes (0.8s blur dissolve for title→first content)
```

**Audio track overlap bug**: Floating-point rounding causes `overlapping_clips_same_track` errors. Fix by adding +0.01s to the start time of the later clip when two clips share a boundary (e.g., `63.15` → `63.16`).

## Step 6: Build HTML Composition

### Project Structure

```
wx-video/
  build.py              # Python build script (parsing + TTS + HTML generation)
  video/
    index.html          # Final HyperFrames composition
    img/                # Downloaded article images
    audio/              # Generated TTS narration files
```

### CSS Patterns

**cinema-title** (cinematic hero with bottom-aligned text):
```css
.cinema-bg{position:absolute;inset:0;background-size:cover;background-position:center}
.cinema-bg::after{content:'';position:absolute;inset:0;
  background:linear-gradient(180deg,rgba(6,6,10,0.75) 0%,rgba(6,6,10,0.25) 50%,rgba(6,6,10,0.85) 100%)}
.hero-text{position:relative;z-index:5;display:flex;flex-direction:column;
  justify-content:flex-end;width:100%;height:100%;padding:100px 140px 90px}
.hero-eyebrow{font-size:18px;color:rgba(255,255,255,0.5);font-family:'Space Grotesk',monospace;
  letter-spacing:6px;text-transform:uppercase;margin-bottom:20px}
.hero-title{font-size:90px;font-weight:900;line-height:1.0;letter-spacing:-3px}
.hero-title em{font-style:normal;color:#F59E0B}
.hero-desc{font-size:26px;color:rgba(255,255,255,0.55);font-weight:300;margin-top:24px;max-width:900px}
.hero-pills{display:flex;gap:12px;margin-top:32px}
.pill{padding:8px 24px;border-radius:100px;font-size:18px;font-weight:600;
  border:1px solid rgba(255,255,255,0.15);color:rgba(255,255,255,0.7)}
.pill-hot{background:#F59E0B;color:#000;border-color:#F59E0B}
```

**showcase** (complete image + text overlay — USE THIS for article images):
```css
.showcase{position:relative;width:100%;height:100%;display:flex;
  align-items:center;justify-content:center}
.showcase-img{max-width:88%;max-height:82vh;border-radius:16px;
  box-shadow:0 40px 120px rgba(0,0,0,0.6);
  object-fit:contain;display:block;position:relative;z-index:2}
/* IMPORTANT: object-fit:contain — never cover — images must not be cropped */
.overlay-top{position:absolute;top:80px;left:140px;right:140px;z-index:10}
.overlay-top h2{font-size:48px;font-weight:800;line-height:1.2;letter-spacing:-1px;
  text-shadow:0 4px 30px rgba(0,0,0,0.8)}
.overlay-top h2 em{font-style:normal;color:#F59E0B}
.overlay-bottom{position:absolute;bottom:80px;left:140px;right:140px;z-index:10;
  display:flex;justify-content:space-between;align-items:flex-end}
.overlay-bottom p{font-size:22px;color:rgba(255,255,255,0.6);text-shadow:0 2px 20px rgba(0,0,0,0.9)}
.overlay-badge{padding:10px 28px;border-radius:12px;font-size:18px;font-weight:700;
  background:rgba(245,158,11,0.9);color:#000}
```

**infographic-grid** (2x2 animated data cards):
```css
.info-bg{position:absolute;inset:0;background:linear-gradient(160deg,#0a0a14,#12121f,#0a0a14)}
.info-grid{position:relative;z-index:5;width:100%;height:100%;display:grid;
  grid-template-columns:1fr 1fr;gap:0;padding:80px 100px}
.info-card{display:flex;flex-direction:column;justify-content:center;padding:40px 50px;
  border:1px solid rgba(255,255,255,0.06);position:relative}
.info-card::before{content:'';position:absolute;top:0;left:0;width:4px;height:60px;
  background:#F59E0B;border-radius:0 4px 4px 0}
.info-num{font-size:80px;font-weight:800;color:#F59E0B;font-family:'Space Grotesk',sans-serif;
  line-height:1}
.info-num span{font-size:32px;color:rgba(255,255,255,0.4);font-weight:400}
.info-bar-wrap{width:100%;height:8px;background:rgba(255,255,255,0.08);border-radius:4px;
  margin-top:16px;overflow:hidden}
.info-bar{height:100%;background:linear-gradient(90deg,#F59E0B,#FBBF24);
  border-radius:4px;width:0}  /* animated from 0 to target % */
```

**comparison-bars** (horizontal bar chart):
```css
.compare-row{display:flex;align-items:center;gap:24px}
.compare-label{width:280px;font-size:20px;color:rgba(255,255,255,0.6);text-align:right}
.compare-track{flex:1;height:28px;background:rgba(255,255,255,0.05);border-radius:6px;overflow:hidden}
.compare-fill{height:100%;border-radius:6px;display:flex;align-items:center;padding-left:12px;
  font-size:14px;font-weight:700;font-family:'Space Grotesk',monospace;color:rgba(0,0,0,0.8)}
.fill-lite{background:linear-gradient(90deg,#F59E0B,#FBBF24)}
.fill-pro{background:linear-gradient(90deg,#10B981,#34D399)}
.fill-gemini{background:linear-gradient(90deg,#6366F1,#818CF8)}
```

**kinetic-quote** (centered kinetic typography):
```css
.kinetic-bg{position:absolute;inset:0;
  background:radial-gradient(ellipse at 30% 50%,rgba(245,158,11,0.08),transparent 60%),#06060A}
.kinetic-content{position:relative;z-index:5;width:100%;height:100%;display:flex;
  flex-direction:column;justify-content:center;align-items:center;padding:120px 160px;text-align:center}
.kinetic-quote{font-size:42px;font-weight:300;color:rgba(255,255,255,0.85);line-height:1.8;
  max-width:1400px}
.kinetic-quote em{font-style:normal;font-weight:700;color:#F59E0B}
.kinetic-line{width:60px;height:3px;background:#F59E0B;border-radius:2px}
```

### Animation Patterns

**Showcase entrance** (image pops in + text overlays):
```javascript
tl.fromTo("#s2 .showcase-img",{opacity:0,scale:0.92,y:30},
  {opacity:1,scale:1,y:0,duration:0.7,ease:"power3.out",overwrite:"auto",immediateRender:true},9.10);
tl.fromTo("#s2 .overlay-top h2",{y:40,opacity:0},{y:0,opacity:1,duration:0.5,ease:"power2.out",
  overwrite:"auto",immediateRender:true},9.30);
tl.fromTo("#s2 .overlay-bottom",{y:20,opacity:0},{y:0,opacity:1,duration:0.4,ease:"expo.out",
  overwrite:"auto",immediateRender:true},9.45);
```

**Infographic animation** (numbers bounce in + bars grow):
```javascript
tl.fromTo("#inf-1",{opacity:0,scale:0.7},{opacity:1,scale:1,duration:0.5,
  ease:"back.out(1.6)",overwrite:"auto",immediateRender:true},65.50);
tl.to("#bar-1",{width:"95%",duration:1.2,ease:"power3.out",overwrite:"auto"},66.30);
```

**Comparison bars** (bars grow with stagger):
```javascript
tl.to("#cb-1a",{width:"87.7%",duration:0.8,ease:"power3.out",overwrite:"auto"},75.20);
tl.to("#cb-1b",{width:"89.5%",duration:0.8,ease:"power3.out",overwrite:"auto"},75.30);
```

**Kinetic quote** (line draws + text fades in + em highlights stagger):
```javascript
tl.fromTo("#k-line",{width:0},{width:60,duration:0.4,ease:"power2.out",
  overwrite:"auto",immediateRender:true},85.30);
tl.fromTo("#k-quote",{opacity:0,y:40},{opacity:1,y:0,duration:0.8,ease:"power3.out",
  overwrite:"auto",immediateRender:true},85.50);
tl.fromTo("#k-quote em",{opacity:0},{opacity:1,duration:0.5,stagger:0.15,overwrite:"auto"},86.00);
```

**Hero title entrance** (staggered bottom-up):
```javascript
tl.fromTo(".hero-eyebrow",{opacity:0,y:15},{opacity:1,y:0,duration:0.5,ease:"power2.out"},0.40);
tl.fromTo(".hero-title",{opacity:0,y:50},{opacity:1,y:0,duration:0.8,ease:"power3.out"},0.60);
tl.from(".pill",{opacity:0,y:15,duration:0.4,ease:"power2.out",stagger:0.08},1.20);
```

Transitions: 0.2s black fade for quick scene changes, 0.4s blur dissolve for title→first content.

### Brand Bar (Required)

Every scene must include:
```html
<div class="brand-bar"><span>github.com/liangdabiao</span></div>
```

### Data Attributes

Every timed element needs:
- `class="clip"` — required on all scenes and transitions
- `data-start`, `data-duration`, `data-track-index` — on scenes
- Transitions on `data-track-index="15"`

## Step 7: Validate

```bash
npx hyperframes lint <video-dir>        # 0 errors required
npx hyperframes inspect <video-dir> --timeout 30000  # 0 errors
```

## Common Pitfalls

| Problem | Cause | Fix |
|:---|:---|:---|
| **Last 25% black screen** | `data-duration` < last audio end time | `TOTAL_DUR = max(last_audio_end, last_scene_end) + 1` |
| **Images cropped / text unreadable** | `object-fit:cover` | Always use `object-fit:contain` on article images |
| **Pure image carousel boring** | Only one scene type | Mix showcase + infographic + comparison + kinetic |
| **Audio overlapping lint error** | Float precision at clip boundaries | Add +0.01s to later clip's `data-start` |
| **Rigid / template-looking layout** | Fixed grid, no visual variety | Vary scene types, use overlays, add animated data |
| **No data in video** | Article has stats but they're not visualized | Extract numbers → infographic-grid + comparison-bars |
| **Text too small** | Using web sizes on video canvas | Landscape 1920x1080: titles 48-90px, body 22-32px |
| **Scene jump cuts** | Missing transitions | Add fade transition between every scene |

## Related References

- `tts-workflow.md` — Full TTS pipeline (audio generation, timeline calculation, wiring)
- `external-tts.md` — MiniMax TTS API details
- `news-flash-images.md` — Pexels background image selection
- `video-composition.md` — Layout and composition rules
- `beat-direction.md` — Scene rhythm planning
- `transitions.md` — Scene transition patterns
