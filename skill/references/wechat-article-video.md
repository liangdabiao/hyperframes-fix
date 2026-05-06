---
name: wechat-article-video
description: Parse WeChat public account articles (mp.weixin.qq.com) via API, extract content and images, then produce HyperFrames video compositions with mixed scene types (text cards, fullscreen images, split layouts, stat cards). Covers API parsing, image downloading, narration generation, scene planning, and HTML composition authoring.
---

# WeChat Article → Video Workflow

Automated pipeline for converting a WeChat public account article into a HyperFrames video composition. Covers article parsing, image extraction, content analysis, scene design, TTS narration, and full HTML composition output.

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

1. **Images**: all `![alt](url)` references — these are the article's screenshots/illustrations
2. **Headings**: `## ` or `### ` lines — these define the article's section structure
3. **Key quotes**: impactful sentences for quote cards
4. **Data points**: statistics, percentages, comparisons for stat cards

```python
imgs = re.findall(r'!\[.*?\]\((.+?)\)', md)
headings = re.findall(r'^#{2,3}\s+(.+)$', md, re.MULTILINE)
```

## Step 3: Plan Scene Types

**CRITICAL: Never use pure image carousel.** Mix at least 3-4 different scene types for visual variety. The scene type determines the visual treatment.

### Scene Type Catalog

| Type | Visual | Best for |
|:---|:---|:---|
| **text-card** | Pexels BG photo + gradient overlay + centered large text | Title, section intro, key message, outro |
| **img-full** | Fullscreen screenshot with bottom caption bar | Article screenshots, comparison images, data tables |
| **split-left** | Left 55% text, right 45% image with gradient fade | Explaining an image, context + evidence |
| **split-right** | Left 45% image with gradient fade, right 55% text | Image showcase + explanation |
| **quote-card** | Dark BG + left accent border + large quote text | Impactful quotes, user complaints, key insights |
| **stat-card** | Dark BG +超大数字 + label | Data comparisons, hit rates, percentages |

### Scene Planning Rules

1. **Open with text-card** — title + subtitle + tag, Pexels background
2. **Alternate scene types** — never put 2 same-type scenes back to back
3. **Match content to type**:
   - Screenshots/comparisons → `img-full`
   - Section transitions / key points → `text-card`
   - Data/stats → `stat-card`
   - Image needing explanation → `split-left` or `split-right`
   - Impactful quotes → `quote-card`
4. **End with text-card** — closing message, Pexels background (same as opening)
5. **Scene count**: typically 10-16 scenes for a standard article

### Example Scene Sequence

```
text-card (title) → quote-card (problem) → img-full (screenshot)
→ img-full (screenshot) → text-card (solution intro) → split-left (feature + image)
→ stat-card (key metric) → img-full (comparison) → split-right (analysis + image)
→ text-card (section header) → img-full (result) → split-left (workflow + image)
→ img-full (architecture) → text-card (outro)
```

## Step 4: Write Narration

Split article content into narration segments matching planned scenes:

- **Title scene**: 1-2 sentences summarizing the article topic
- **Content scenes**: Extract key points from the article text, paraphrase concisely
- **Outro**: Call-to-action or closing thought
- **Text density per scene**: follow tts-workflow.md guidelines (60-70% of scene duration at TTS speed 1.6)

Generate audio via MiniMax TTS API (see `external-tts.md` for full details).

## Step 5: Calculate Timing

**Total duration = max(last scene end time, last audio end time)**

This is critical — audio often runs longer than visuals. Never cut audio short. If audio exceeds image rotation, add an outro text-card scene to fill the gap.

```
Scene duration: typically 6-8 seconds
Transition gap: 0.4s between scenes (0.8s for major transitions like title→content)
Audio buffer: +2.0s per scene recommended
```

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

### CSS Patterns for Each Scene Type

**text-card** (centered text on photo background):
```css
.text-card{background-size:cover;background-position:center}
.text-card::after{content:'';position:absolute;inset:0;
  background:linear-gradient(135deg,rgba(10,10,15,0.88) 0%,rgba(10,10,15,0.65) 100%);z-index:1}
.text-card .tc-inner{position:relative;z-index:5;display:flex;flex-direction:column;
  justify-content:center;align-items:center;width:100%;height:100%;text-align:center;
  padding:100px 120px 80px}
.tc-huge{font-size:76px;font-weight:900;line-height:1.15;letter-spacing:-2px;max-width:1400px}
.tc-sub{font-size:32px;color:#7A7A8A;font-weight:300;margin-top:20px;max-width:1200px}
```

**img-full** (fullscreen image with caption):
```css
.img-full .img-wrap{position:absolute;top:0;left:0;width:100%;height:100%;z-index:1;overflow:hidden}
.img-full .img-wrap img{width:100%;height:100%;object-fit:cover;display:block}
.img-full .img-info{position:absolute;bottom:0;left:0;right:0;padding:32px 64px;z-index:8;
  background:linear-gradient(transparent,rgba(0,0,0,0.82));pointer-events:none}
```

**split-left** (text left, image right):
```css
.split-left .sp-img{position:absolute;top:0;right:0;width:55%;height:100%;overflow:hidden}
.split-left .sp-img img{width:100%;height:100%;object-fit:cover;display:block}
.split-left .sp-img::after{content:'';position:absolute;inset:0;
  background:linear-gradient(270deg,rgba(10,10,15,0.70) 0%,transparent 100%)}
.split-left .sp-text{position:absolute;top:0;left:0;width:55%;height:100%;z-index:5;
  display:flex;flex-direction:column;justify-content:center;padding:80px 60px 80px 100px;gap:20px}
```

**split-right** (image left, text right):
```css
.split-right .sp-img{position:absolute;top:0;left:0;width:55%;height:100%;overflow:hidden}
.split-right .sp-text{position:absolute;top:0;right:0;width:55%;height:100%;z-index:5;
  display:flex;flex-direction:column;justify-content:center;padding:80px 100px 80px 60px;
  gap:20px;text-align:right}
```

**stat-card** (big number):
```css
.stat-card .st-inner{position:relative;z-index:5;display:flex;flex-direction:column;
  justify-content:center;align-items:center;width:100%;height:100%;text-align:center;
  padding:100px 120px}
.st-num{font-size:120px;font-weight:900;color:#A855F7;line-height:1;font-family:'Space Grotesk',sans-serif}
```

### Animation Patterns

Each scene type has its own entrance animation:

- **text-card**: title slides up `y:60→0`, subtitle follows, tag last — staggered power3.out
- **img-full**: image fades in with subtle scale `scale:1.05→1`, caption slides up
- **split-left**: image slides from right with scale, text slides from left `x:-40→0`
- **split-right**: mirror of split-left
- **stat-card**: number scales up with `back.out(1.4)` bounce, label follows
- **quote-card**: quote slides from left with opacity, subtitle follows

Transitions between scenes: 0.2s black fade (quick cuts) or 0.4s blur dissolve (major transitions).

### Timeline Construction

```javascript
// Scene entrance pattern (fromTo + overwrite:auto + immediateRender:true)
tl.fromTo("#s1",{opacity:0},{opacity:1,duration:0.05,overwrite:"auto",immediateRender:true},0.00);
tl.fromTo("#s1 .tc-huge",{y:60,opacity:0},{y:0,opacity:1,duration:0.7,ease:"power3.out",overwrite:"auto",immediateRender:true},0.30);

// Quick transition between scenes
tl.fromTo("#t1",{opacity:0},{opacity:1,duration:0.20,overwrite:"auto",immediateRender:true},8.00);
tl.to("#t1",{opacity:0,duration:0.20,overwrite:"auto"},8.20);
```

### Brand Bar (Required)

Every scene must include the brand bar:
```html
<div class="brand-bar"><span>github.com/liangdabiao</span></div>
```

### Data Attributes

Every timed element needs:
- `class="clip"` — required on all scenes and transitions for proper visibility control
- `data-start`, `data-duration`, `data-track-index` — on scenes
- Transitions on `data-track-index="15"`

## Step 7: Validate

```bash
npx hyperframes lint <video-dir>        # 0 errors required
npx hyperframes inspect <video-dir>      # 0 errors, warnings acceptable
```

Common issues:
- `timed_element_missing_clip_class` → add `class="clip"` to all timed scenes/transitions
- `container_overflow` on img-full images → normal for `object-fit:cover`, safe to ignore
- `composition_file_too_large` → warning only for single-file compositions

## Build Script Template

Create a Python build script that automates the entire pipeline:

```python
import os, re, requests, html as h
from concurrent.futures import ThreadPoolExecutor, as_completed

# 1. Parse article via API
# 2. Extract images and content
# 3. Download images
# 4. Generate TTS narration
# 5. Calculate timing
# 6. Build HTML with mixed scene types
# 7. Output video/index.html
```

Key build script considerations:
- Use `requests` for both API calls and image downloads
- Parallel image downloads with `ThreadPoolExecutor(max_workers=5)`
- Skip re-downloading existing files (check size > 1000 bytes)
- TTS: use MiniMax API with speed=1.6, male-qn-qingse voice
- Audio gap: 0.01s between sequential audio clips on same track
- Escape all user content with `html.escape()` before inserting into HTML

## Common Pitfalls

| Problem | Cause | Fix |
|:---|:---|:---|
| Video cuts off early | Total duration doesn't cover all audio | Calculate `max(last_scene_end, last_audio_end)` |
| Pure image carousel boring | Only one scene type used | Mix 3-4 scene types, alternate |
| Audio truncated | Video duration < audio end time | Add outro scene to fill gap |
| Images not loading | URL path wrong or download failed | Verify all images downloaded, check paths |
| Text too small on landscape | Using portrait font sizes | Scale down ~30% for 1920x1080 |
| Scene jump cuts | No transition between scenes | Add fade transitions between every scene |
| Lint clip warnings | Missing `class="clip"` | Add to all timed scenes and transitions |

## Related References

- `tts-workflow.md` — Full TTS pipeline (audio generation, timeline calculation, wiring)
- `external-tts.md` — MiniMax TTS API details
- `news-flash-images.md` — Pexels background image selection
- `video-composition.md` — Layout and composition rules
- `beat-direction.md` — Scene rhythm planning
- `transitions.md` — Scene transition patterns
