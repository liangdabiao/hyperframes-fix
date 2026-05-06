---
name: hyperframes
description: Create video compositions, animations, title cards, overlays, captions, voiceovers, audio-reactive visuals, and scene transitions in HyperFrames HTML. Use when asked to build any HTML-based video content, add captions or subtitles synced to audio, generate text-to-speech narration, integrate third-party TTS voiceover (MiniMax, OpenAI, etc.) with per-scene audio segmentation and sync, create audio-reactive animation (beat sync, glow, pulse driven by music), add animated text highlighting (marker sweeps, hand-drawn circles, burst lines, scribble, sketchout), or add transitions between scenes (crossfades, wipes, reveals, shader transitions). Covers composition authoring, timing, media, and the full video production workflow. Also triggers on WeChat article URLs (mp.weixin.qq.com) to auto-parse and produce video. For CLI commands (init, lint, preview, render, transcribe, tts) see the hyperframes-cli skill.
---

# HyperFrames

HTML is the source of truth for video. A composition is an HTML file with `data-*` attributes for timing, a GSAP timeline for animation, and CSS for appearance. The framework handles clip visibility, media playback, and timeline sync.

## Approach

### Discovery (exploratory requests only)

For open-ended requests ("make me a product launch video", "create something for our brand") where the user hasn't committed to a direction, understand intent before picking colors:

- **Audience** — who watches this? Developers? Executives? General consumers?
- **Platform** — where does it play? Social (15s), website hero, product demo, internal?
- **Priority** — what matters most? Motion quality? Content accuracy? Brand fidelity? Speed?
- **Variations** — does the user want options, or a single best shot?

For specific requests ("add a title card", "fix the timing on scene 3"), skip discovery.

For exploratory requests, consider offering 2-3 variations that differ meaningfully — not just color swaps, but different pacing, energy levels, or structural approaches. One safe/expected, one ambitious. Don't mandate this — it's a tool available when appropriate.

### Step 1: Design system

If `design.md` or `DESIGN.md` exists in the project, read it first (check both casings — they're different files on Linux). It's the source of truth for brand colors, fonts, and constraints. Use its exact values — don't invent colors or substitute fonts. Any format works (YAML frontmatter, prose, tables — just extract the values).

If it names fonts you can't find locally (no `fonts/` directory with `.woff2` files, not a built-in font), warn the user before writing HTML: "design.md specifies [font name] but no font files found. Please add .woff2 files to `fonts/` or I'll fall back to [closest built-in alternative]."

If no `design.md` exists, offer the user a choice:

1. **User named a style or mood?** → Read [visual-styles.md](./visual-styles.md) for the 9 named presets (including News Flash 信息流快消). Pick the closest match.
2. **Want to browse options visually?** → Run the design picker: read [references/design-picker.md](references/design-picker.md) for the full workflow. This serves a visual picker page. The user configures mood, palette, typography, and motion in the browser, then copies the generated design.md and pastes it back into the conversation.
3. **Want to skip and go fast?** → Ask: mood, light or dark, any brand colors/fonts? Then pick a palette from [house-style.md](./house-style.md).

**design.md defines the brand. It does not define video composition rules.** Those come from [references/video-composition.md](references/video-composition.md) and [house-style.md](./house-style.md). Use brand colors at video-appropriate scale — not at web-UI opacity.

### Step 2: Prompt expansion

Always run on every composition (except single-scene pieces and trivial edits). This step grounds the user's intent against `design.md` and `house-style.md` and produces a consistent intermediate that every downstream agent reads the same way.

Read [references/prompt-expansion.md](references/prompt-expansion.md) for the full process and output format.

### Step 3: Plan

Before writing HTML, think at a high level:

1. **What** — what should the viewer experience? Identify the narrative arc, key moments, and emotional beats.
2. **Structure** — how many compositions, which are sub-compositions vs inline, what tracks carry what (video, audio, overlays, captions).
3. **Rhythm** — declare your scene rhythm before implementing. Which scenes are quick hits, which are holds, where do shaders land, where does energy peak. Name the pattern: fast-fast-SLOW-fast-SHADER-hold. Read [references/beat-direction.md](references/beat-direction.md) for rhythm templates.
4. **Timing** — which clips drive the duration, where do transitions land, what's the pacing.
5. **Layout** — build the end-state first. See "Layout Before Animation" below.
6. **Animate** — then add motion using the rules below.

**Build what was asked.** A request for "a title card" is not a request for "a title card + 3 supporting scenes + ambient music + captions." Every scene, every element, every tween should earn its place. If additional scenes or elements would genuinely improve the piece, propose them — don't add them.

For small edits (fix a color, adjust timing, add one element), skip straight to the rules.

<HARD-GATE>
Before writing ANY composition HTML — verify you have a visual identity from Step 1. If you're reaching for `#333`, `#3b82f6`, or `Roboto`, you skipped it.
</HARD-GATE>

## Layout Before Animation

Position every element where it should be at its **most visible moment** — the frame where it's fully entered, correctly placed, and not yet exiting. Write this as static HTML+CSS first. No GSAP yet.

**Why this matters:** If you position elements at their animated start state (offscreen, scaled to 0, opacity 0) and tween them to where you think they should land, you're guessing the final layout. Overlaps are invisible until the video renders. By building the end state first, you can see and fix layout problems before adding any motion.

### The process

1. **Identify the hero frame** for each scene — the moment when the most elements are simultaneously visible. This is the layout you build.
2. **Write static CSS** for that frame. The `.scene-content` (or `.sc`) container MUST fill the full scene using `width: 100%; height: 100%; padding: Npx;` with `display: flex; flex-direction: column; gap: Npx; box-sizing: border-box`. Use padding to push content inward — NEVER `position: absolute; top: Npx` on a content container. Absolute-positioned content containers overflow when content is taller than the remaining space. Reserve `position: absolute` for decoratives only. **Minimum padding: top/bottom 80px, left/right 48px.** When a brand bar (`position:absolute; top:0`) is present, the content container must have `padding-top: 80px` or more to prevent text from being covered. Content must never touch any edge of the canvas.
3. **Add entrances with `gsap.from()`** — animate FROM offscreen/invisible TO the CSS position. The CSS position is the ground truth; the tween describes the journey to get there. (In sub-compositions loaded via `data-composition-src`, prefer `gsap.fromTo()` — see load-bearing GSAP rules in [references/motion-principles.md](references/motion-principles.md).)
4. **Add exits with `gsap.to()`** — animate TO offscreen/invisible FROM the CSS position.

### Example

```css
/* scene-content fills the scene, padding positions content — never touch edges */
.scene-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 80px 48px;  /* minimum: top/bottom 80px, left/right 48px */
  gap: 24px;
  box-sizing: border-box;
}
.title {
  font-size: 120px;
}
.subtitle {
  font-size: 42px;
}
/* Container fills any scene size (1920x1080, 1080x1920, etc).
   Padding positions content. Flex + gap handles spacing. */
```

**WRONG — hardcoded dimensions and absolute positioning:**

```css
.scene-content {
  position: absolute;
  top: 200px;
  left: 160px;
  width: 1920px;
  height: 1080px;
  display: flex; /* ... */
}
```

```js
// Step 3: Animate INTO those positions
tl.from(".title", { y: 60, opacity: 0, duration: 0.6, ease: "power3.out" }, 0);
tl.from(".subtitle", { y: 40, opacity: 0, duration: 0.5, ease: "power3.out" }, 0.2);
tl.from(".logo", { scale: 0.8, opacity: 0, duration: 0.4, ease: "power2.out" }, 0.3);

// Step 4: Animate OUT from those positions
tl.to(".title", { y: -40, opacity: 0, duration: 0.4, ease: "power2.in" }, 3);
tl.to(".subtitle", { y: -30, opacity: 0, duration: 0.3, ease: "power2.in" }, 3.1);
tl.to(".logo", { scale: 0.9, opacity: 0, duration: 0.3, ease: "power2.in" }, 3.2);
```

### When elements share space across time

If element A exits before element B enters in the same area, both should have correct CSS positions for their respective hero frames. The timeline ordering guarantees they never visually coexist — but if you skip the layout step, you won't catch the case where they accidentally overlap due to a timing error.

### What counts as intentional overlap

Layered effects (glow behind text, shadow elements, background patterns) and z-stacked designs (card stacks, depth layers) are intentional. The layout step is about catching **unintentional** overlap — two headlines landing on top of each other, a stat covering a label, content bleeding off-frame.

## Data Attributes

### All Clips

| Attribute          | Required                          | Values                                                 |
| ------------------ | --------------------------------- | ------------------------------------------------------ |
| `id`               | Yes                               | Unique identifier                                      |
| `data-start`       | Yes                               | Seconds or clip ID reference (`"el-1"`, `"intro + 2"`) |
| `data-duration`    | Required for img/div/compositions | Seconds. Video/audio defaults to media duration.       |
| `data-track-index` | Yes                               | Integer. Same-track clips cannot overlap.              |
| `data-media-start` | No                                | Trim offset into source (seconds)                      |
| `data-volume`      | No                                | 0-1 (default 1)                                        |

`data-track-index` does **not** affect visual layering — use CSS `z-index`.

### Composition Clips

| Attribute                    | Required | Values                                       |
| ---------------------------- | -------- | -------------------------------------------- |
| `data-composition-id`        | Yes      | Unique composition ID                        |
| `data-start`                 | Yes      | Start time (root composition: use `"0"`)     |
| `data-duration`              | Yes      | Takes precedence over GSAP timeline duration |
| `data-width` / `data-height` | Yes      | Pixel dimensions (1920x1080 or 1080x1920)    |
| `data-composition-src`       | No       | Path to external HTML file                   |

## Composition Structure

Sub-compositions loaded via `data-composition-src` use a `<template>` wrapper. **Standalone compositions (the main index.html) do NOT use `<template>`** — they put the `data-composition-id` div directly in `<body>`. Using `<template>` on a standalone file hides all content from the browser and breaks rendering.

**CRITICAL: Always include `data-duration` on the root composition div.** Without it, the framework doesn't know the total video length and will either use the GSAP timeline duration (which only covers animations, not scene holds) or truncate the video prematurely. The correct duration is the sum of the last scene's `data-start + data-duration`.

```html
<!-- WRONG — no data-duration, video gets cut off -->
<div data-composition-id="main" data-width="1080" data-height="1920" data-start="0">

<!-- CORRECT — explicit total duration -->
<div data-composition-id="main" data-width="1080" data-height="1920" data-start="0" data-duration="162.9">
```

Sub-composition structure:

```html
<template id="my-comp-template">
  <div data-composition-id="my-comp" data-width="1920" data-height="1080">
    <!-- content -->
    <style>
      [data-composition-id="my-comp"] {
        /* scoped styles */
      }
    </style>
    <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
    <script>
      window.__timelines = window.__timelines || {};
      const tl = gsap.timeline({ paused: true });
      // tweens...
      window.__timelines["my-comp"] = tl;
    </script>
  </div>
</template>
```

Load in root: `<div id="el-1" data-composition-id="my-comp" data-composition-src="compositions/my-comp.html" data-start="0" data-duration="10" data-track-index="1"></div>`

## Video and Audio

Video must be `muted playsinline`. Audio is always a separate `<audio>` element:

```html
<video
  id="el-v"
  data-start="0"
  data-duration="30"
  data-track-index="0"
  src="video.mp4"
  muted
  playsinline
></video>
<audio
  id="el-a"
  data-start="0"
  data-duration="30"
  data-track-index="2"
  src="video.mp4"
  data-volume="1"
></audio>
```

## Timeline Contract

- All timelines start `{ paused: true }` — the player controls playback
- Register every timeline: `window.__timelines["<composition-id>"] = tl`
- Framework auto-nests sub-timelines — do NOT manually add them
- Duration comes from `data-duration`, not from GSAP timeline length
- Never create empty tweens to set duration

## Rules (Non-Negotiable)

**Deterministic:** No `Math.random()`, `Date.now()`, or time-based logic. Use a seeded PRNG if you need pseudo-random values (e.g. mulberry32).

**GSAP:** Only animate visual properties (`opacity`, `x`, `y`, `scale`, `rotation`, `color`, `backgroundColor`, `borderRadius`, transforms). Do NOT animate `visibility`, `display`, or call `video.play()`/`audio.play()`.

**Animation conflicts:** Never animate the same property on the same element from multiple timelines simultaneously.

**No `repeat: -1`:** Infinite-repeat timelines break the capture engine. Calculate the exact repeat count from composition duration: `repeat: Math.ceil(duration / cycleDuration) - 1`.

**Synchronous timeline construction:** Never build timelines inside `async`/`await`, `setTimeout`, or Promises. The capture engine reads `window.__timelines` synchronously after page load. Fonts are embedded by the compiler, so they're available immediately — no need to wait for font loading.

**Never do:**

1. Forget `window.__timelines` registration
2. Use video for audio — always muted video + separate `<audio>`
3. Nest video inside a timed div — use a non-timed wrapper
4. Use `data-layer` (use `data-track-index`) or `data-end` (use `data-duration`)
5. Animate video element dimensions — animate a wrapper div
6. Call play/pause/seek on media — framework owns playback
7. Create a top-level container without `data-composition-id`
8. Use `repeat: -1` on any timeline or tween — always finite repeats
9. Build timelines asynchronously (inside `async`, `setTimeout`, `Promise`)
10. Use `gsap.set()` on clip elements from later scenes — they don't exist in the DOM at page load. Use `tl.set(selector, vars, timePosition)` inside the timeline at or after the clip's `data-start` time instead.
11. Use `<br>` in content text — forced line breaks don't account for actual rendered font width. Text that wraps naturally + a `<br>` produces an extra unwanted break, causing overlap. Let text wrap via `max-width` instead. Exception: short display titles where each word is deliberately on its own line (e.g., "THE\nIMMORTAL\nGAME" at 130px).

## Scene Transitions (Non-Negotiable)

Every multi-scene composition MUST follow ALL of these rules. Violating any one of them is a broken composition.

1. **ALWAYS use transitions between scenes.** No jump cuts. No exceptions.
2. **ALWAYS use entrance animations on every scene.** Every element animates IN via `gsap.from()`. No element may appear fully-formed. If a scene has 5 elements, it needs 5 entrance tweens.
3. **NEVER use exit animations** except on the final scene. This means: NO `gsap.to()` that animates opacity to 0, y offscreen, scale to 0, or any other "out" animation before a transition fires. The transition IS the exit. The outgoing scene's content MUST be fully visible at the moment the transition starts.
4. **Final scene only:** The last scene may fade elements out (e.g., fade to black). This is the ONLY scene where `gsap.to(..., { opacity: 0 })` is allowed.

**WRONG — exit animation before transition:**

```js
// BANNED — this empties the scene before the transition can use it
tl.to("#s1-title", { opacity: 0, y: -40, duration: 0.4 }, 6.5);
tl.to("#s1-subtitle", { opacity: 0, duration: 0.3 }, 6.7);
// transition fires on empty frame
```

**RIGHT — entrance only, transition handles exit:**

```js
// Scene 1 entrance animations
tl.from("#s1-title", { y: 50, opacity: 0, duration: 0.7, ease: "power3.out" }, 0.3);
tl.from("#s1-subtitle", { y: 30, opacity: 0, duration: 0.5, ease: "power2.out" }, 0.6);
// NO exit tweens — transition at 7.2s handles the scene change
// Scene 2 entrance animations
tl.from("#s2-heading", { x: -40, opacity: 0, duration: 0.6, ease: "expo.out" }, 8.0);
```

## Animation Guardrails

- Offset first animation 0.1-0.3s (not t=0)
- Vary eases across entrance tweens — use at least 3 different eases per scene
- Don't repeat an entrance pattern within a scene
- Avoid full-screen linear gradients on dark backgrounds (H.264 banding — use radial or solid + localized glow)
- Font size minimums for **portrait 1080×1920**: card titles 56px+, card body 44px+, tags 44px+, scene headers 56px+, highlights 60px+, infographic labels 42px+. For landscape, scale these down by ~30% (titles 40px+, body 32px+).
- `font-variant-numeric: tabular-nums` on number columns

If no `design.md` exists, follow [house-style.md](./house-style.md) for aesthetic defaults.

## Common Pitfalls

### GSAP `from()` immediateRender conflict (CRITICAL)

**Never use `gsap.from()` on overlapping selectors.** `from()` defaults to `immediateRender: true`, which immediately sets all matched elements to the "from" values (e.g., `opacity: 0`). When a second `from()` targets a subset of those same elements, it captures the already-invisible state as its end state — so the element animates from invisible to invisible and permanently disappears.

```js
// BUG — .card:nth-child(2) captures invisible state as end state, card stays hidden
tl.from("#s06 .card", {y: 50, opacity: 0}, 48.8);
tl.from("#s06 .card:nth-child(2)", {y: 50, opacity: 0}, 49.2);

// FIX — fromTo with explicit end values + overwrite + immediateRender prevents conflict and flicker
tl.fromTo("#s06 .card:nth-child(1)", {y: 50, opacity: 0}, {y: 0, opacity: 1, duration: 0.6, overwrite: "auto", immediateRender: true}, 48.8);
tl.fromTo("#s06 .card:nth-child(2)", {y: 50, opacity: 0}, {y: 0, opacity: 1, duration: 0.6, overwrite: "auto", immediateRender: true}, 49.2);
```

**Rule:** When animating multiple child elements with staggered timing, use `fromTo()` with `overwrite: "auto"`, `immediateRender: true`, and individual `:nth-child()` selectors. Never mix a bulk parent selector (`.card`) with a child selector (`.card:nth-child(2)`) using `from()`. The `immediateRender: true` is required because `fromTo()` defaults to `immediateRender: false` — without it, elements flash visible for one frame before the animation starts.

### Background image gradient overlay opacity

When using `::after` pseudo-elements as gradient overlays on background images, keep alpha values between **0.40 and 0.60**. Values above 0.80 completely wash out the background photo, making it invisible. This is especially important for the News Flash 信息流快消 style which relies on Pexels background images.

```css
/* TOO DARK — background image invisible */
.scene-bg::after { background: linear-gradient(180deg, rgba(0,0,0,0.85), rgba(0,0,0,0.75)); }

/* CORRECT — image visible through overlay */
.scene-bg::after { background: linear-gradient(180deg, rgba(0,0,0,0.55), rgba(0,0,0,0.45)); }
```

### Track overlap from rounded timings

When deriving `data-start` and `data-duration` from a timeline (e.g., TTS generation output), rounding can cause adjacent clips to overlap by 0.1s on the same track, triggering `overlapping_clips_same_track` lint errors. Always verify that each clip's end time (`data-start + data-duration`) does not exceed the next clip's `data-start` on the same `data-track-index`. Use the exact values from `timeline.json`, or chain durations so each start equals the previous end.

### Scene count limit — max ~12 scenes per composition

**Never exceed ~12 scenes.** The capture engine cannot handle 20+ `position:absolute` scenes with `filter:blur` transitions and `backdrop-filter` effects simultaneously. Tested: 24 sub-scenes (one per image) causes "不能播放" (can't play). Proven working: 10-12 scenes.

**For multi-image chapters, use ONE scene per chapter with `tl.set()` src-swap to rotate images:**
```js
tl.to("#s1-img",{opacity:0,duration:.25,ease:"power2.in"},10.69);
tl.set("#s1-img",{attr:{src:"img/ch01_02.jpg"}},10.94);
tl.to("#s1-img",{opacity:1,duration:.25,ease:"power2.out"},10.94);
```

**Do NOT create sub-scenes per image.** Each sub-scene adds another full `position:absolute` layer with blur transitions, quickly exceeding the engine's capacity.

### position:absolute only on approved elements

Only `.scene`, `.brand-bar`, and `.sc-bg` may use `position:absolute`. Content elements (`.sc`, cards, images, text containers) must use flexbox layout. Stacking `<img>` elements with `position:absolute` inside a scene breaks playback.

### Font sizes too small for portrait video (INFO-FLASH style)

The skill's original minimums ("60px+ headlines, 20px+ body") were written for landscape 1920×1080 and are dangerously small for portrait 1080×1920. In practice, 32px card titles and 26px body text are unreadable on phone-sized output. Users reported "字还是很小" twice before sizes were large enough.

**Portrait 1080×1920 minimums:**
- Card titles (`.card-t`): **56px+** (was 24→32→48, still too small)
- Card body (`.card-b`): **44px+** (was 20→26→38, still too small)
- Tags (`.tag`): **44px+**
- Scene headers (`.ht`): **56px+**
- Highlight text: **60px+**
- Infographic labels: **42px+**
- Card padding: **52px 48px+** (was 30px 28px→44px 40px)
- Container padding (`.sc`): **110px 60px 90px 60px+** (was 80px 48px→100px 56px)

**Why it keeps happening:** The default web-developer instinct is to use small, dense type. Video is viewed at arm's length on a phone — every element needs to be billboard-scale. If a user says "字很小", increase ALL sizes by 40-50%, not 10-20%. This has now happened THREE times across sessions. Use the minimums above as absolute floor, never start lower.

## Typography and Assets

- **Built-in fonts:** Write the `font-family` you want in CSS — the compiler embeds supported fonts automatically.
- **Custom fonts:** If design.md names a font that isn't built-in, the user must provide `.woff2` files in a `fonts/` directory. If missing, warn before writing HTML. When files exist, add `@font-face` declarations pointing to the local files.
- Add `crossorigin="anonymous"` to external media
- For dynamic text overflow, use `window.__hyperframes.fitTextFontSize(text, { maxWidth, fontFamily, fontWeight })`
- All files live at the project root alongside `index.html`; sub-compositions use `../`

## Editing Existing Compositions

- **Read actual files, don't guess.** When editing, extending, or creating companion compositions, read the existing source. Don't reconstruct hex codes from memory. Don't guess GSAP easing patterns. The composition IS the spec — extract exact values from it.
- Match existing fonts, colors, animation patterns from what you read
- Only change what was requested
- Preserve timing of unrelated clips

## Output Checklist

**Fast (run immediately, block on results):**

- [ ] `npx hyperframes lint` and `npx hyperframes validate` both pass
- [ ] Design adherence verified if design.md exists

**Slow (run in parallel while presenting the preview to the user):**

- [ ] `npx hyperframes inspect` passes, or every reported overflow is intentionally marked
- [ ] Contrast warnings addressed (see Quality Checks below)
- [ ] Animation choreography verified (see Quality Checks below)

## Quality Checks

### Visual Inspect

`hyperframes inspect` runs the composition in headless Chrome, seeks through the timeline, and maps visual layout issues with timestamps, selectors, bounding boxes, and fix hints. Run it after `lint` and `validate`:

```bash
npx hyperframes inspect
npx hyperframes inspect --json
```

Failures usually mean text is spilling out of a bubble/card, a fixed-size label is clipping dynamic copy, or text has moved off the canvas. Fix by increasing container size or padding, reducing font size or letter spacing, adding a real `max-width` so text wraps inside the container, or using `window.__hyperframes.fitTextFontSize(...)` for dynamic copy.

Use `--samples 15` for dense videos and `--at 1.5,4,7.25` for specific hero frames. Repeated static issues are collapsed by default to avoid flooding agent context. If overflow is intentional for an entrance/exit animation, mark the element or ancestor with `data-layout-allow-overflow`. If a decorative element should never be audited, mark it with `data-layout-ignore`.

### Inspect Strategy

**Timestamp selection matters.** The inspector measures ALL elements in the DOM, including hidden scenes (opacity:0). A timestamp like `t=0.5` will report overflow from every scene stacked in the document, even though only one is visible. This floods results with false positives.

**Use `--at` to sample only at visible timestamps:**

```bash
# Calculate when each scene is visible, then sample at those times
npx hyperframes inspect --at 2,10,45,100,150 --no-contrast --timeout 30000
```

**Windows timeout.** The default 10s navigation timeout is too short on Windows. Always specify `--timeout 30000` (30s).

**Batch vs individual.** More than 8 timestamps may hit timeout. Run in smaller batches:

```bash
npx hyperframes inspect --at 2,10,45 --no-contrast --timeout 30000
npx hyperframes inspect --at 100,150,175 --no-contrast --timeout 30000
```

**Hidden-scene false positives.** If an error references a scene that isn't active at the sampled timestamp (e.g., scene4 text overflow at t=2 when scene1 is playing), the error is cosmetic — the content is clipped by `overflow:hidden` and never visible. Options:
- **Ignore** — the error won't affect rendered output
- **`data-layout-ignore`** — add to the hidden scene's element to skip inspection entirely

**Error severity:**
- `error` — real visual issue, must fix
- `warning` — container overflow, usually fixable
- `info` — canvas overflow on hidden/offscreen content, usually safe to ignore

`hyperframes layout` is the compatibility alias for the same check.

### Contrast

`hyperframes validate` runs a WCAG contrast audit by default. It seeks to 5 timestamps, screenshots the page, samples background pixels behind every text element, and computes contrast ratios. Failures appear as warnings:

```
⚠ WCAG AA contrast warnings (3):
  · .subtitle "secondary text" — 2.67:1 (need 4.5:1, t=5.3s)
```

If warnings appear:

- On dark backgrounds: brighten the failing color until it clears 4.5:1 (normal text) or 3:1 (large text, 24px+ or 19px+ bold)
- On light backgrounds: darken it
- Stay within the palette family — don't invent a new color, adjust the existing one
- Re-run `hyperframes validate` until clean

Use `--no-contrast` to skip if iterating rapidly and you'll check later.

### Design Adherence

If a `design.md` exists, verify the composition follows it after authoring. Read the HTML and check:

1. **Colors** — every hex value in the composition appears in design.md's palette section (however the user labeled it: Colors, Palette, Theme, etc.). Flag any invented colors.
2. **Typography** — font families and weights match design.md's type spec. No substitutions.
3. **Corners** — border-radius values match the declared corner style, if specified.
4. **Spacing** — padding and gap values fall within the declared density range, if specified.
5. **Depth** — shadow usage matches the declared depth level, if specified (flat = none, subtle = light, layered = glows).
6. **Avoidance rules** — if design.md has a section listing things to avoid (commonly "What NOT to Do", "Don'ts", "Anti-patterns", or "Do's and Don'ts"), verify none are present.

Report violations as a checklist. Fix each one before serving.

If no `design.md` exists (house-style-only path), verify:

1. **Palette consistency** — the same bg, fg, and accent colors are used across all scenes. No per-scene color invention.
2. **No lazy defaults** — check the composition against house-style.md's "Lazy Defaults to Question" list. If any appear, they must be a deliberate choice for the content, not a default.

### Animation Map

After authoring animations, run the animation map to verify choreography:

```bash
node skills/hyperframes/scripts/animation-map.mjs <composition-dir> \
  --out <composition-dir>/.hyperframes/anim-map
```

Outputs a single `animation-map.json` with:

- **Per-tween summaries**: `"#card1 animates opacity+y over 0.50s. moves 23px up. fades in. ends at (120, 200)"`
- **ASCII timeline**: Gantt chart of all tweens across the composition duration
- **Stagger detection**: reports actual intervals (`"3 elements stagger at 120ms"`)
- **Dead zones**: periods over 1s with no animation — intentional hold or missing entrance?
- **Element lifecycles**: first/last animation time, final visibility
- **Scene snapshots**: visible element state at 5 key timestamps
- **Flags**: `offscreen`, `collision`, `invisible`, `paced-fast` (under 0.2s), `paced-slow` (over 2s)

Read the JSON. Scan summaries for anything unexpected. Check every flag — fix or justify. Verify the timeline shows the intended choreography rhythm. Re-run after fixes.

Skip on small edits (fixing a color, adjusting one duration). Run on new compositions and significant animation changes.

---

## References (loaded on demand)

- **[references/captions.md](references/captions.md)** — Captions, subtitles, lyrics, karaoke synced to audio. Tone-adaptive style detection, per-word styling, text overflow prevention, caption exit guarantees, word grouping. Read when adding any text synced to audio timing.
- **[references/tts.md](references/tts.md)** — Text-to-speech with Kokoro-82M. Voice selection, speed tuning, TTS+captions workflow. Read when generating narration or voiceover using the built-in local TTS engine.
- **[references/external-tts.md](references/external-tts.md)** — Third-party TTS integration (MiniMax, OpenAI, etc.) with per-scene audio segmentation, speed calibration, overlap prevention, and composition wiring. Read when adding voiceover from an external TTS API to a multi-scene composition.
- **[references/tts-workflow.md](references/tts-workflow.md)** — End-to-end TTS→HTML workflow: script splitting, audio generation, timeline calculation, composition wiring, lint/render pipeline. Read when producing a full voiceover video from scratch.
- **[references/wechat-article-video.md](references/wechat-article-video.md)** — WeChat public account article (mp.weixin.qq.com) → HyperFrames video pipeline. API parsing, image extraction, mixed scene types (text-card, img-full, split, stat, quote), narration generation, timing, and full composition authoring. Read when the user provides a WeChat article URL.
- **[references/audio-reactive.md](references/audio-reactive.md)** — Audio-reactive animation: map frequency bands and amplitude to GSAP properties. Read when visuals should respond to music, voice, or sound.
- **[references/css-patterns.md](references/css-patterns.md)** — CSS+GSAP marker highlighting: highlight, circle, burst, scribble, sketchout. Deterministic, fully seekable. Read when adding visual emphasis to text.
- **[references/video-composition.md](references/video-composition.md)** — Video-medium rules: density, color presence, scale, frame composition, design.md as brand not layout. **Always read** — these override web instincts.
- **[references/beat-direction.md](references/beat-direction.md)** — Beat planning: concept, mood, choreography verbs, rhythm templates, transition decisions, depth layers. **Always read for multi-scene compositions.**
- **[references/typography.md](references/typography.md)** — Typography: font pairing, OpenType features, dark-background adjustments, font discovery script. **Always read** — every composition has text.
- **[references/motion-principles.md](references/motion-principles.md)** — Motion design principles, image motion treatment, load-bearing GSAP rules. **Always read** — every composition has motion.
- **[references/techniques.md](references/techniques.md)** — 11 visual techniques with code patterns: SVG drawing, Canvas 2D, CSS 3D, kinetic type, Lottie, video compositing, typing effect, variable fonts, MotionPath, velocity transitions, audio-reactive. Read when planning techniques per beat.
- **[references/narration.md](references/narration.md)** — Pacing, tone, script structure, number pronunciation, opening line patterns. Read when the composition includes voiceover or TTS.
- **[references/design-picker.md](references/design-picker.md)** — Create a design.md via visual picker. Read when no design.md exists and the user wants to create one.
- **[visual-styles.md](visual-styles.md)** — 9 named visual styles with hex palettes, GSAP easing signatures, and shader pairings. Includes News Flash (信息流快消) for urgent, data-heavy financial/news content with Pexels background images. Read when user names a style or when generating design.md.
- **[news-flash-images.md](references/news-flash-images.md)** — Pexels API integration workflow for News Flash style: search queries by scene theme, image selection criteria, overlay darkness guide, three-segment layout sizing, and inspect overflow handling.
- **[house-style.md](house-style.md)** — Default motion, sizing, and color palettes when no design.md is specified.
- **[patterns.md](patterns.md)** — PiP, title cards, slide show patterns.
- **[data-in-motion.md](data-in-motion.md)** — Data, stats, and infographic patterns.
- **[references/transcript-guide.md](references/transcript-guide.md)** — Transcription commands, whisper models, external APIs, troubleshooting.
- **[references/dynamic-techniques.md](references/dynamic-techniques.md)** — Dynamic caption animation techniques (karaoke, clip-path, slam, scatter, elastic, 3D).

- **[references/transitions.md](references/transitions.md)** — Scene transitions: crossfades, wipes, reveals, shader transitions. Energy/mood selection, CSS vs WebGL guidance. **Always read for multi-scene compositions** — scenes without transitions feel like jump cuts.
  - [transitions/catalog.md](references/transitions/catalog.md) — Hard rules, scene template, and routing to per-type implementation code.
  - Shader transitions are in `@hyperframes/shader-transitions` (`packages/shader-transitions/`) — read package source, not skill files.

GSAP patterns and effects are in the `/gsap` skill.
