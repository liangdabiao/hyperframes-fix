# News Flash Image Workflow — Pexels API Integration

Workflow for sourcing background images for the News Flash (信息流快消) visual style. Uses Pexels API for high-quality, free-to-use photographic backgrounds.

## API Setup

**Endpoint:** `GET https://api.pexels.com/v1/search`

**Auth:** `Authorization: <API_KEY>` header

**Key params:**
- `query` — search terms (English)
- `per_page` — results per request (1-3 recommended)
- `orientation` — `"portrait"` for 1080x1920, `"landscape"` for 1920x1080

**Image URL formats:**
- `src.large2x` — best quality, `?auto=compress&cs_tinysrgb&w=1260&h=2240&dpr=1` for portrait
- `src.original` — uncompressed, may be very large

## Workflow

### Step 1: Plan image categories by scene theme

Map each scene to a visual theme before searching:

| Scene content | Search query | Typical image |
|---|---|---|
| Title / Market overview | `stock market trading screen` | Digital charts, trading floors |
| Tech / AI | `artificial intelligence technology` | Robots, circuit boards, digital networks |
| Semiconductor | `microchip semiconductor` | Circuit board close-ups |
| Energy / Battery | `electric vehicle battery` | EVs, charging, batteries |
| Mining / Minerals | `mining minerals rocks` | Open-pit mines, mineral deposits |
| Solar / Renewable | `solar panel energy` | Solar farms, panels in sunlight |
| Financial data | `business finance chart growth` | Charts, data analysis |
| Celebration / Closing | `celebration fireworks night` | Fireworks, festive scenes |
| Risk / Warning | `stock market crash` | Red screens, dramatic charts |

### Step 2: Batch search Pexels

Search 6-8 categories in parallel (API rate limit allows this). Use `per_page=3` for options.

```bash
curl -s -H "Authorization: <KEY>" \
  "https://api.pexels.com/v1/search?query=stock+market+trading&per_page=3&orientation=portrait" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);[print(p['id'],p['src']['large2x'],p['alt']) for p in d.get('photos',[])]"
```

### Step 3: Select and assign images

Choose images that:
1. **Match the scene theme** — financial data → chart images
2. **Have dark or muted tones** — bright images need heavier overlay
3. **Have good contrast regions** — ensure text readability areas exist
4. **Vary across scenes** — avoid using the same image for adjacent scenes (reusing for distant scenes is fine)

### Step 4: Reference images in CSS

Use as `background-image` on a full-bleed positioned div:

```css
.bg-img{position:absolute;top:0;left:0;width:100%;height:100%;
  background-size:cover;background-position:center;z-index:0}
#scene5 .bg-img{background-image:url('https://images.pexels.com/photos/6477216/pexels-photo-6477216.jpeg?auto=compress&cs_tinysrgb&w=1260&h=2240&dpr=1')}
```

**Always use `crossorigin="anonymous"` on `<img>` tags** (not needed for CSS backgrounds).

### Step 5: Add overlay for text readability

Dark overlay on top of the image, below the content:

```html
<div class="bg-img"></div>
<div class="overlay ov-heavy"></div>  <!-- rgba(0,0,0,0.72) -->
<div class="sc">...</div>            <!-- content at z-index:2 -->
```

Overlay darkness guide:
- **70-75%** (`rgba(0,0,0,0.72-0.75)`) — for data-heavy scenes with many text elements
- **55-65%** (`rgba(0,0,0,0.55-0.65)`) — for hero/title scenes where image atmosphere matters more
- **Gradient** — for transitional scenes, use `linear-gradient` to blend image edges

## CSS Architecture

```css
/* Base layers */
.scene{position:absolute;top:0;left:0;width:1080px;height:1920px;overflow:hidden}
.bg-img{position:absolute;top:0;left:0;width:100%;height:100%;background-size:cover;background-position:center;z-index:0}
.overlay{position:absolute;top:0;left:0;width:100%;height:100%;z-index:1}
.ov-heavy{background:rgba(0,0,0,0.72)}
.ov-med{background:rgba(0,0,0,0.6)}
.ov-light{background:linear-gradient(180deg,rgba(0,0,0,0.5) 0%,rgba(0,0,0,0.8) 100%)}
.ov-grad-r{background:linear-gradient(135deg,rgba(0,0,0,0.3) 0%,rgba(0,0,0,0.85) 100%)}
.sc{position:relative;z-index:2;display:flex;flex-direction:column;width:100%;height:100%}
```

## Three-Segment Layout Pattern

The signature News Flash scene structure stacks three zones vertically:

```html
<div class="sc">
  <!-- HEAD: Red/Yellow/Green colored banner -->
  <div class="scene-head" style="background:#DC2626">
    <div class="head-title" style="font-size:48px;font-weight:900">BREAKING HEADLINE</div>
  </div>

  <!-- MID: Secondary highlights in pills/tags -->
  <div class="scene-mid" style="background:#F59E0B">
    <span class="pill" style="background:rgba(0,0,0,0.85);padding:12px 24px">Tag 1</span>
    <span class="pill" style="background:rgba(0,0,0,0.85);padding:12px 24px">Tag 2</span>
  </div>

  <!-- BODY: flex:1 data area with dark background -->
  <div class="scene-body" style="flex:1;background:rgba(0,0,0,0.55)">
    <div class="data-card">...</div>
  </div>
</div>
```

**Sizing caution:** The three segments must fit within the scene height (1920px portrait). If content overflows:
- Reduce padding (40px → 32px on banners)
- Reduce font sizes slightly (74px → 68px on titles)
- Use `flex:1` on the body section so it absorbs remaining space
- Mark with `data-layout-allow-overflow` or `data-layout-ignore` if minor overflow is acceptable (clipped by `overflow:hidden` on scene)

## Inspect Overflow Handling

The `hyperframes inspect` tool measures ALL scenes in the DOM, including hidden ones (opacity:0). For scenes that are intentionally packed tight with `overflow:hidden`:

- `data-layout-allow-overflow` — marks overflow as intentional on the element
- `data-layout-ignore` — skips the element entirely from inspection (use on decorative elements and scenes only measured when hidden)

For the three-segment layout specifically, if the `.sc` reports overflow at timestamps where that scene isn't visible, it's safe to add `data-layout-ignore` to the scene div.
