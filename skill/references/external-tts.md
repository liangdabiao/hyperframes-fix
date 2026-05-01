# External TTS Integration

Add third-party TTS voiceover to a HyperFrames video. Covers audio generation, per-scene segmentation, text sizing, and composition wiring.

## Core Principles

1. **Per-scene audio** — one audio file per narrated scene, each with its own `<audio>` element. Never use a single continuous file.
2. **Every scene narrated** — all scenes including the title card should have voiceover. No silent scenes unless intentional.
3. **Audio finishes before transition** — narration must complete naturally. Never trim audio to fit; shorten text instead.
4. **Explicit `data-duration`** — always use seconds, never `"auto"`.
5. **Audio outside composition div** — `<audio>` elements are top-level clips in `<body>`.

## Step 1: Map narration to scenes

For each scene that needs narration, note:
- `data-start` time (when the scene begins)
- Scene duration (gap until next scene starts)
- Narration text

Every scene should have narration, including the title card. No scene should be silent unless intentionally designed that way.

### Text sizing (Chinese TTS)

Chinese speech rate scales linearly with `speed` setting (MiniMax speech-2.8-hd measured):

| Speed | Rate | Effective chars/sec (target 60%) | Effective chars/sec (target 70%) |
|:---|::---|:---|:---|
| 1.0 | ~4.0 chars/sec | 2.4 | 2.8 |
| 1.2 | ~4.8 chars/sec | 2.9 | 3.4 |
| **1.6 (default)** | ~6.4 chars/sec | 3.8 | 4.5 |
| 2.0 | ~8.0 chars/sec | 4.8 | 5.6 |

Target **60-70% of scene duration** — this leaves enough buffer so the voice always finishes before the scene transitions.

**Rule: never trim audio.** If the generated audio exceeds the scene duration, shorten the narration text and regenerate. Trimming cuts off mid-sentence speech and sounds broken.

| Scene duration | speed 1.0 (60/70%) | speed 1.6 (60/70%) | speed 2.0 (60/70%) |
|:---|:---|:---|:---|
| 5s | 12 / 14 | 19 / 22 | 24 / 28 |
| 6s | 14 / 17 | 23 / 27 | 29 / 34 |
| 7s | 17 / 20 | 27 / 32 | 34 / 39 |
| 8s | 19 / 22 | 31 / 36 | 38 / 45 |
| 9s | 22 / 25 | 35 / 40 | 43 / 50 |
| 12s | 29 / 34 | 46 / 54 | 58 / 67 |

After first generation, **always measure actual audio duration** with `ffprobe`. If audio > scene duration, reduce text and regenerate. Iterate until every segment fits.

### Speed calibration (实测校准)

**重要**: MiniMax 官方文档的语速数据与实际存在偏差。实测 MiniMax speech-2.8-hd 中文播报:

| speed 参数 | 官方理论语速 (字/秒) | 实测语速 (字/秒) | 偏差 |
|:---|:---|:---|:---|
| 1.0 | ~4.0 | ~3.2 | -20% |
| 1.6 | ~6.4 | ~4.6 | -28% |
| 2.0 | ~8.0 | ~5.8 | -27% |

**结论**: 实际语速约为理论值的 72-80%。如果按理论值计算文本长度，音频大概率会超出场景时长。

**推荐做法 — 先生成后量测**:
1. 先按理论值 60% 目标编写文本
2. 调用 API 生成所有音频
3. 用 `ffprobe` 测量每段实际时长
4. 按实际时长反向设定场景 duration:

```python
scene_duration = max(actual_audio_dur + 2.0, 6.0)  # 2秒 buffer，最短6秒
```

这种方式比反复调整文本更高效，一次生成即可得到正确的 timeline。

### Text sizing (English TTS)

~2.5 words per second (see `narration.md`). Target 60-70% of scene duration.

| Scene duration | Max words (60%) | Max words (70%) |
|:---|:---|:---|
| 5s | 7 | 9 |
| 7s | 10 | 12 |
| 9s | 13 | 16 |
| 12s | 18 | 21 |

## Step 2: Generate audio per scene

Call the TTS API once per scene segment. Example for MiniMax HTTP API:

```python
segments = [
    ("S1",  "s01.mp3", 6.0, "摆摊三天，亏明白了。深圳清湖市集血泪经验大公开。"),
    ("S2",  "s02.mp3", 5.0, "从来没做过市集，想体验线下真实流量，七个品类去深圳清湖摆摊。"),
    # ...one per scene, text sized for chosen speed
]

speed = 1.6  # Default. Adjust only if user requests a different pace.

for label, filename, scene_dur, text in segments:
    payload = {
        "model": "speech-2.8-hd",
        "text": text,
        "stream": False,
        "voice_setting": {"voice_id": "male-qn-qingse", "speed": speed, "vol": 1, "pitch": 0},
        "audio_setting": {"sample_rate": 32000, "bitrate": 128000, "format": "mp3", "channel": 1},
        "output_format": "url",
    }
    resp = requests.post("https://api.minimaxi.com/v1/t2a_v2", json=payload, headers=headers)
    audio_ms = resp["extra_info"]["audio_length"]
    if audio_ms / 1000 > scene_dur:
        # Text too long for this speed. Shorten text and regenerate.
```

**After generation, verify every segment:**

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 audio/s02.mp3
# Must be <= scene duration. If not, shorten text and re-call API.
```

## Step 3: Wire into composition

### Placement (critical)

`<audio>` elements go in `<body>`, **outside** the `<div data-composition-id="...">`. They are top-level clip elements alongside `<video>`, `<img>`, and sub-compositions — NOT children of the composition div.

```html
<body>
  <div data-composition-id="main" data-width="1920" data-height="1080" data-start="0" data-duration="85">
    <!-- scenes... -->
  </div>

  <!-- Audio clips go HERE, outside the composition div -->
  <audio id="narr-s2" data-start="6" data-duration="5" data-track-index="2" src="audio/s02.mp3" data-volume="1"></audio>
  <audio id="narr-s3" data-start="11" data-duration="8" data-track-index="2" src="audio/s03.mp3" data-volume="1"></audio>
</body>
```

### Attributes

| Attribute | Required | Notes |
|:---|:---|:---|
| `data-start` | Yes | Must match the scene's start time exactly |
| `data-duration` | Yes | Use explicit seconds matching the scene window. **Do NOT use `"auto"`** — the renderer detects the audio but does not encode it into the video |
| `data-track-index` | Yes | Use `"2"` for narration. All narration clips share the same track |
| `src` | Yes | Path to audio file |
| `data-volume` | No | 0-1, default 1 |
| `id` | Yes | Unique per clip |

### No overlaps on same track

All narration clips on `data-track-index="2"` must have non-overlapping time windows (`data-start + data-duration <= next clip's data-start`). If they overlap, the linter errors with `overlapping_clips_same_track`. This is why text must be sized correctly (Step 1) rather than trimming audio.

**常见原因与修复:**

| 原因 | 示例 | 修复 |
|:---|:---|:---|
| 理论语速偏差导致音频超长 | speed=1.6 理论 6.4字/秒，实际 4.6 | 用 ffprobe 实测，按实际时长设定 |
| 浮点精度累积 | 85.7 + 11.2 = 96.9 > 下一片段 96.8 | `data-duration` 向下取整 0.1s |
| timeline.json 四舍五入 | round(96.85, 1) = 96.9，但下一片段 start=96.8 | 手动对齐，确保 strict `<` |

**修复模式:**
```python
# 生成 timeline 后，强制对齐消除重叠
for i in range(len(scenes) - 1):
    end = scenes[i]["start"] + scenes[i]["duration"]
    if end > scenes[i+1]["start"]:
        scenes[i]["duration"] = scenes[i+1]["start"] - scenes[i]["start"] - 0.01
```

**HTML 中的 data-duration 也需同步调整**，否则 lint 仍然报错。

## Step 4: Verify

```bash
npx hyperframes lint          # Must be 0 errors (overlapping_clips is an error)
npx hyperframes render
ffprobe renders/output.mp4    # Must show both video AND audio streams
```

## Troubleshooting

| Symptom | Cause | Fix |
|:---|:---|:---|
| Rendered video has no audio stream | `<audio>` inside composition div, or `data-duration="auto"` | Move outside composition div, use explicit duration |
| Narration doesn't match scenes | Single continuous audio file | Split into per-scene segments |
| `overlapping_clips_same_track` error | Audio duration exceeds scene window | Shorten narration text and regenerate — never trim |
| Speech gets cut off mid-sentence | Audio was trimmed, or text too long for scene | Shorten text, regenerate audio, use untrimmed file |
| Audio plays in browser but no sound in rendered video | `data-duration="auto"` on audio element | Change to explicit seconds |
| Scenes feel rushed with narration | Too many words per scene | Reduce text to 60-70% of scene capacity |

## API Reference: MiniMax TTS (HTTP)

Endpoint: `POST https://api.minimaxi.com/v1/t2a_v2`

Key parameters:
- `model`: `speech-2.8-hd` (best quality), `speech-2.8-turbo` (fastest)
- `text`: up to 10,000 chars; use `<#x.x#>` for pauses (seconds)
- `voice_setting.speed`: 0.5-2.0 (default 1.0). **Affects max text per scene** — see sizing table above. Default use 1.6 (fast but clear). 2.0 is very fast and may sound robotic
- `voice_setting.voice_id`: e.g. `male-qn-qingse`, `female-shaonv`
- `audio_setting.format`: `mp3`, `wav`, `flac`, `pcm`
- `output_format`: `"url"` (returns download URL in `data.audio` field) or `"hex"` (default)
- Auth: `Authorization: Bearer <api_key>`

Note: `output_format: "url"` puts the download URL in the `data.audio` field (not a hex string). Check with `audio_val.startswith("http")`.
