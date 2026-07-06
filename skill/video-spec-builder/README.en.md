<img width="2172" height="724" alt="ChatGPT Image May 16, 2026, 10_46_58 PM" src="https://github.com/user-attachments/assets/7820d93e-84b6-4e09-904c-9567c6595c57" />

**English** · [中文](README.md)

# video-spec-builder

[![License: MIT](https://img.shields.io/badge/License-MIT-brightgreen)](LICENSE) ![Agent Agnostic](https://img.shields.io/badge/Agent-Agnostic-blueviolet) [![skills.sh Compatible](https://img.shields.io/badge/skills.sh-Compatible-brightgreen)](https://skills.sh)

> A skill that works like a video director. You say "I want to make a video," and it grills you with questions until your idea is a script you can actually shoot.

I built this skill after realizing the hard part of making a video isn't the rendering. It's figuring out what you actually want.

You've got a vague idea in your head: a product video, a short for social, a company intro. But it's fuzzy. The moment you try to build it, the details get you — how long each shot runs, what's on screen, what comes first and what comes later. You probably haven't pinned them all down, and you might not even be able to put them into words.

video-spec-builder gets you through that part. Install it, then tell your AI "I want to make a video" inside Codex or Claude Code, and it takes over the conversation. It listens to your brief the way a director would, then keeps asking: Who's this for? How long? What's the one line people should walk away with? Which shot carries the weight? Anywhere you go vague, or skip something, it stops and pushes you to fill it in.

A few rounds of that, and the fuzzy idea becomes a `video-spec.md`: a shot-by-shot script, timed to the second, every shot written out. Hand that to HyperFrames and it renders into a real video.

It won't shoot the video for you, and it won't invent the idea. It does one thing: push you, and stay with you, until the idea is something you can actually build.

## What it helps with

The problem it solves is "I have an idea but I can't explain it." A few situations where it earns its keep:

- You know the feeling you want but can't describe the actual picture. It refuses words like "premium" or "high-impact" and keeps after you until you can describe real shots and real motion.
- You have an idea but never thought parts of it through. Maybe you've got the opening and the ending but not the middle. Maybe it never crossed your mind that a section could use captions, or that visuals can move to the beat of the music. It brings those up.
- You have plenty of raw material but no order to it. A script, selling points, a pile of assets — it helps you cut that into individual shots and put them in sequence.

In the end it writes all of it into a script: what each shot shows, how it's presented, how long it holds, how it cuts to the next one.

There are two ways to use it. With no script yet, it talks you through the whole thing from scratch and produces a `video-spec.md`. With a script already there and just one thing to change, you tell it what you want different; it asks enough to be sure, makes the change, and checks whether it knocked anything else loose.

## The workflow

It's two skills working in sequence. video-spec-builder sits upstream and turns your idea into a script. HyperFrames sits downstream and turns the script into video.

The main path is a 0-1 Socratic conversation. Two shortcuts ride alongside it:

- **Drop a WeChat article URL** (`mp.weixin.qq.com`) → it parses the article, extracts images and data, plans the shots, generates TTS narration, writes the HTML composition, and hands it to HyperFrames.
- **Ask for MiniMax TTS** → uses a cloud Chinese voice (much more natural than the built-in local TTS), with a "measure first, derive scene duration second" pipeline that puts narration into the video correctly.

```
                ┌──────────────────────┐
                │   You give it:        │
                │   idea / script / URL │
                └──────────────────────┘
                          │
        ┌─────────────────┼─────────────────────┐
        │                 │                     │
   vague idea         full script       WeChat article URL
        │                 │                     │
        ▼                 ▼                     ▼
   ┌────────┐         ┌────────┐         ┌──────────┐
   │ 0-1 chat │         │ iterate  │         │ wechat-  │
   │ Q&A + shots│       │ edit script │       │ article  │
   └────────┘         └────────┘         │ video flow│
        │                 │              └──────────┘
        ▼                 ▼                     │
   video-spec.md    video-spec.md (edited)      │
        │                 │                     │
        └─────────────────┼─────────────────────┘
                          │
                          ▼   /hyperframes
                 ┌────────────────────────┐
                 │       HyperFrames      │   renders from the script
                 └────────────────────────┘
                          │
                          ▼
                       finished video
```

So before you start, you'll want both skills installed.

## Install

I mostly use this skill in **Codex**, and after that **Claude Code**. Those are the two setups it works best in.

Before anything else, install two things: HyperFrames (the renderer, downstream) and video-spec-builder (this skill). Both go in through the `skills` CLI, one command each:

```bash
npx skills add heygen-com/hyperframes
npx skills add feicaiclub/video-spec-builder
```

Each command installs once and covers Codex, Claude Code, Cursor and the rest. You don't install separately for each tool.

Two scopes to know about. By default it installs into the current folder (project-level), so it only works in the project where you ran the command. If you make videos often, add `-g` to install globally, available everywhere:

```bash
npx skills add feicaiclub/video-spec-builder -g
```

Never used the `skills` CLI? Nothing to set up. `npx` pulls a copy just to run and leaves nothing behind. Needs Node 18 or newer.

## Using it

### Making a video from scratch

Once it's installed, just talk to your AI in plain language inside Codex or Claude Code:

```
I want to make a 3-minute product demo, posting it on YouTube
```

It takes over and starts asking. You don't need to track its internal steps; it just talks with you. First it pins down the basics: who it's for, where it's going, how long, the core message. Then it takes stock of the material you have. Then it settles the style and pacing, picks a visual theme, and finally uses reference videos and counter-examples to calibrate.

It's a real conversation, not a form to fill in. Answer vaguely and it digs; miss something and it fills it in. When you're done, it writes out `video-spec.md`.

### Changing a video you already have

If there's already a `video-spec.md` in the project, just say what you want:

```
Shot 3 is too fast, slow it down; swap the background music for something quieter
```

It checks what you're after, looks at whether the change touches other shots, then updates the script.

### Rendering it

Once the script is final, hand it to HyperFrames:

```
/hyperframes
```

> In Claude Code, besides triggering it by talking, you can also call it directly with `/video-spec-builder`.

### Generating a video from an article

Drop a WeChat public-account article link (must be on the `mp.weixin.qq.com` domain) and skip the script-writing step:

```
https://mp.weixin.qq.com/s/EWUB8PTltK0iM9zf6_dTBg turn this into a video
```

The skill switches into the WeChat-article video flow automatically: parse the article → extract images and data → plan 5+ scene types → generate TTS narration → derive scene duration from real audio length → write the HTML composition. End-to-end, no dialogue round-trips. Great for turning long-form posts into distributable videos.

You'll need a MiniMax API key (set `minimaxi=sk-...` in your `.env`). Without it, the skill falls back to local TTS — workable, but noticeably more mechanical in Chinese.

### Using MiniMax TTS for more natural Chinese narration

If you want cloud AI voiceover (much more natural than the local TTS), tell the skill during the conversation:

```
Use MiniMax for the voiceover, the clear male voice
```

Or set the TTS source to `MiniMax` in the §5 of `video-spec.md`, with a `voice_id` and `speed`. The skill then runs the 6-step pipeline: **generate audio → measure with ffprobe → derive scene duration (don't trust the 27% gap between the official rate and reality) → write HTML → lint → render**.

## What HyperFrames can and can't do

Worth spelling this out, because it decides whether your script is worth the paper it's on.

HyperFrames renders video from HTML. That one fact is the root of everything it can and can't do. If HTML, CSS, and code can draw it, HyperFrames can turn it into video. If HTML can't draw it, HyperFrames can't either.

What it's **good at** is text and layout work: title animation, captions, word-by-word highlighting, page layout, transitions, charts, UI mockups, geometric animation. Anything you can draw with code, it handles cleanly.

What it **can't do** — know this before you write the script, because however good the script is, if HyperFrames can't render it, the work is wasted:

- It can't draw illustrations. Hand-drawn characters, painterly visuals, cartoon figures — it can't produce those, and writing code won't get you there. Code draws shapes and charts, not artwork.
- It can't generate live-action footage. A real filmed shot, a person performing — it can't conjure that out of nothing.
- It can't generate photorealistic images.
- For voiceover, you have three paths: local TTS (Kokoro-82M, free, plain Chinese), **MiniMax cloud TTS** (speech-2.8-hd, needs an API key, much more natural Chinese), or a human recording (you record it or hire a voice actor). Cloud TTS is the right call when you want hands-off audio from a script; for the highest quality, a human is still the answer.
- It won't compose background music for you.

The short version: HyperFrames is an **assembly** tool, not a **creation** tool. It takes the material you've prepared — video clips, images, voiceover, music — cuts and composites it, adds text and motion, and puts together a finished video. Assembly is its job.

So here's the thing worth remembering: how good the video looks comes down to the material you feed it. Good material and HyperFrames assembles it sharply. Weak material and HyperFrames can't save it. Video clips, images, voiceover, music — these are worth preparing carefully up front. They decide the quality, not HyperFrames.

## What's new

### WeChat article URL → video

Drop a WeChat public-account article link (must be on the `mp.weixin.qq.com` domain) and the skill runs end-to-end without a 5-round dialogue. The flow:

1. Call a third-party API to convert the article to markdown
2. Extract images, headings, data points, and pull quotes from the body
3. Plan 9-12 shots across at least 4 of the 5 scene types (no image-only carousels)
4. Generate per-scene TTS, measure actual audio length with ffprobe, derive scene duration from that
5. Write a designer-quality HTML (images use `object-fit:contain` so they're never cropped, screen text gets shadows, motion is GSAP)
6. Lint and inspect, then hand to HyperFrames

Per-step details in `references/wechat-article-video.md`.

### MiniMax cloud TTS pipeline

Chinese AI voiceover has historically been painful: local TTS sounds mechanical, other cloud TTS services mean one-off integration. MiniMax's `speech-2.8-hd` is a solid choice for natural Chinese, callable over plain HTTP, no client to install.

This repository's TTS pipeline is designed around five rules:

1. **One audio file per scene** — never concatenate. Per-scene audio is what lets narration sync to visuals.
2. **Generate first, measure second** — the official speech rate is 27% faster than reality. Estimate-by-char-count will get you burned.
3. **`<audio>` lives at the top level of `<body>`** — inside the composition div, it's silently ignored, and the rendered video has no sound.
4. **`data-duration` must be explicit seconds** — `"auto"` plays in the browser but renders as silence.
5. **Guard against floating-point overlap** — adjacent audio clips on the same track can collide by sub-millisecond rounding; lint catches it. Fix: round `data-duration` down by 0.1s, or shift the next clip's `data-start` up by +0.01s.

Full spec in `references/external-tts.md`. End-to-end 6-step pipeline in `references/tts-workflow.md`.

### Visual validation (required)

Once you're at the `build_html.py` → render step, watch out: HyperFrames' `lint` and `inspect` **don't run a headless render**. `lint 0 errors + inspect 0 issues` is not the same as "the picture is right."

The classic silent failure is renaming HTML IDs without updating the matching CSS selectors — say, renaming `s01` → `s1` everywhere in the HTML but leaving 34 `#s01 .platform-cloud`-style selectors untouched. Result: all 5 scenes end up squashed into the middle ~30% of the canvas, and lint doesn't notice.

**So from this point on, every built HTML has to clear a 5-step validation:**

| # | Check | Tool | Pass condition |
|---|---|---|---|
| V1 | Structure | `npx hyperframes lint` | 0 errors |
| V2 | Timeline | `npx hyperframes inspect` | 0 issues |
| V3 | **CSS actually matches** | Playwright `getComputedStyle` | Key layout selectors actually take effect |
| V4 | **Manual frame check** | `ffmpeg -ss <N> -i output.mp4 -frames:v 1` | No overflow, layout matches intent |
| V5 | Media streams | `ffprobe` | Both video+audio present, duration error ≤ 0.5s |

V1+V2+V5 are structural (machine-judged). **V3+V4 are visual (Playwright + your eyes)**. Skipping V3+V4 is flying blind.

**The bar for "render succeeded" is V3 confirming `display: grid` is actually `grid`, font-size is actually ≥ 60px, and `object-fit: contain` is actually `contain`.** Full script and troubleshooting in the V1-V5 section of `references/hyperframes-render.md`.

## Visual themes

What a video looks like — colors, fonts, motion, transition style — is decided by a "theme." You either use one of HyperFrames' built-in presets, or write your own.

### The 8 HyperFrames presets

HyperFrames ships 8 themes. Name one and it's yours:

| Theme | Mood | Good for |
|---|---|---|
| Swiss Pulse | Precise, restrained, Swiss type | SaaS, data, dev tools, dashboards |
| Velvet Standard | Premium, timeless | Luxury, enterprise software, keynotes, investor decks |
| Deconstructed | Industrial, raw | Tech launches, security products, anything with a punk edge |
| Maximalist Type | Loud, kinetic | Big launches, milestone announcements, high-energy hype |
| Data Drift | Futuristic, immersive | AI products, ML platforms, frontier tech |
| Soft Signal | Intimate, warm | Wellness brands, personal stories, lifestyle products |
| Folk Frequency | Cultural, vivid | Consumer apps, food, community products |
| Shadow Cut | Dark, cinematic | Security products, dramatic reveals, serious storytelling |

Once you've picked one, write its name into `video-spec.md`.

### Writing your own

If none of the presets fit, write your own. HyperFrames has a few hard rules for custom themes, nothing complicated:

- A theme is a single `design.md` file, placed at the root of your video project. HyperFrames finds and reads it automatically when rendering.
- The format is fixed. A block of YAML up top for the design variables: colors, fonts, corner radius, spacing, motion. Below it, a set of fixed sections describing the design rules in prose: Overview, Colors, Typography, Elevation, Components, Do's and Don'ts.
- If your theme uses a font HyperFrames doesn't ship with, put the font's `.woff2` files in the project's `fonts/` folder yourself.

Drop a finished `design.md` into the video project root and the theme is live.

### A theme I made for you: Spec Mono

Writing a `design.md` from scratch takes some work, so I made one ahead of time and put it in this repo. It's called **Spec Mono**: pure black and white, the geometric, restrained, engineered look of SpaceX × Grok. It's done — use it as is.

<!-- placeholder: drop the Spec Mono preview image at spec-mono/preview.png, then uncomment the line below -->
<!-- ![Spec Mono preview](spec-mono/preview.png) -->

Download to see complete design [视频组件库 v2 · 硅谷暗色科技风.pdf](https://github.com/user-attachments/files/27866436/v2.pdf)
<img width="1020" height="1440" alt="视频组件库 v2 · 硅谷暗色科技风" src="https://github.com/user-attachments/assets/bef576da-73ba-4bad-a9c4-3c673e652eaa" />

The `spec-mono/` folder holds three files:

| File | What it is |
|---|---|
| `design.md` | the theme itself — this is what HyperFrames reads |
| `tokens.css` | a ready-made CSS file: color/font/spacing variables, plus styles for some decorative elements |
| `spec-mono-components.md` | the per-component spec for all 69 components under this theme |

To use it, copy `spec-mono/design.md` into your video project root and bring `tokens.css` along. It's already written to HyperFrames' format, so it renders right away.

> **Heads up:** the `design.md` tokens and `spec-mono-components.md` here are only a distilled, condensed extract. The complete theme design code is generated and downloaded from Claude Design. For the full implementation code, see the `Full Code/` folder.

## What's in this repo

```
video-spec-builder/
├── SKILL.md                  the skill's main file — the AI reads this first
├── README.md                 中文
├── README.en.md              English
├── LICENSE
├── references/               reference docs on questioning, shot breakdown, pacing — loaded as needed
│   ├── workflow-0-1.md
│   ├── workflow-iteration.md
│   ├── question-bank.md
│   ├── scene-breakdown.md
│   ├── components-catalog.md
│   ├── pacing-rules.md
│   ├── spec-rules.md
│   ├── dialogue-style.md
│   ├── external-tts.md            MiniMax cloud TTS API, rate calibration, float-overlap guard
│   ├── tts-workflow.md            6-step end-to-end pipeline (script → audio → timeline → HTML → lint → render)
│   ├── wechat-article-video.md    WeChat article URL → video — concept guide (API parse, image extraction, 5 scene types)
│   ├── wechat-build-example.md    WeChat article → video — executable Python template (one build.py runs the whole pipeline)
│   ├── hyperframes-render.md      Renderer-side 10 hard constraints (C1-C10) + 5-step validation (V1-V5) + troubleshooting
│   └── design-md-spec.md          Custom design.md YAML format spec (field reference + 6-section template)
├── templates/
│   └── video-spec-template.md    output template for video-spec.md
├── examples/
│   └── video-spec-spacex.md      a complete video-spec example
└── spec-mono/                    the bundled custom theme, Spec Mono
    ├── design.md
    ├── tokens.css
    └── spec-mono-components.md
```

## License

MIT
