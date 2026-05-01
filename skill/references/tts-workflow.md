# TTS → HTML Workflow

End-to-end workflow for producing a voiceover video: from script to rendered MP4. Covers narration writing, audio generation, timeline calculation, composition wiring, and validation.

## Overview

```
Script (.md)
    ↓  1. Split into scenes
Narration segments
    ↓  2. Generate TTS audio
Audio files (mp3)
    ↓  3. Measure & build timeline
timeline.json
    ↓  4. Build HTML composition
index.html + audio files
    ↓  5. Lint & Inspect
0 errors
    ↓  6. Render
output.mp4 (video + audio)
```

## Step 1: Script → Scene Narration

Read the source script and split it into one narration segment per scene.

**Rules:**
- Every scene gets narration, including title and closing
- Segment text should be 60-70% of the target scene duration
- Scene duration: typically 6-15 seconds each, longer for data-heavy scenes

**Chinese TTS sizing** (MiniMax speech-2.8-hd, speed=1.6):

| Scene duration | Max chars (60%) | Max chars (70%) |
|:---|:---|:---|
| 6s | 17 | 19 |
| 8s | 22 | 26 |
| 10s | 28 | 32 |
| 12s | 33 | 39 |

**English TTS sizing** (~2.5 words/sec):

| Scene duration | Max words (60%) | Max words (70%) |
|:---|:---|:---|
| 6s | 9 | 10 |
| 8s | 12 | 14 |
| 12s | 18 | 21 |

Output: a list of `(label, text)` pairs, one per scene.

## Step 2: Generate Audio

Call the TTS API for each segment. See `external-tts.md` for API details.

```python
import requests, os, subprocess, json, time

API_KEY = "your-key"
API_URL = "https://api.minimaxi.com/v1/t2a_v2"
AUDIO_DIR = "audio/project"
os.makedirs(AUDIO_DIR, exist_ok=True)
headers = {"Content-Type": "application/json", "Authorization": f"Bearer {API_KEY}"}

narrations = [
    ("S01", "Scene 1 narration text here..."),
    ("S02", "Scene 2 narration text here..."),
    # ... one per scene
]

speed = 1.6
results = []

for label, text in narrations:
    filename = f"pd_{label.lower()}.mp3"
    print(f"Generating {label} ({len(text)} chars)...")

    payload = {
        "model": "speech-2.8-hd",
        "text": text,
        "stream": False,
        "voice_setting": {"voice_id": "male-qn-qingse", "speed": speed, "vol": 1, "pitch": 0},
        "audio_setting": {"sample_rate": 32000, "bitrate": 128000, "format": "mp3", "channel": 1},
        "output_format": "url",
    }

    resp = requests.post(API_URL, json=payload, headers=headers, timeout=60)
    rj = resp.json()
    if rj.get("base_resp", {}).get("status_code") != 0:
        print(f"  ERROR: {rj['base_resp']['status_msg']}")
        results.append({"label": label, "audio_dur": 0})
        continue

    # Download audio
    audio_data = rj["data"]["audio"]
    filepath = os.path.join(AUDIO_DIR, filename)
    if audio_data.startswith("http"):
        ar = requests.get(audio_data, timeout=60)
        with open(filepath, "wb") as f:
            f.write(ar.content)
    else:
        with open(filepath, "wb") as f:
            f.write(bytes.fromhex(audio_data))

    # Measure actual duration with ffprobe
    try:
        r = subprocess.run(
            ["ffprobe", "-v", "error", "-show_entries", "format=duration", "-of", "csv=p=0", filepath],
            capture_output=True, text=True, timeout=10
        )
        actual_dur = float(r.stdout.strip())
    except:
        actual_dur = rj["extra_info"]["audio_length"] / 1000

    results.append({"label": label, "filename": filename, "audio_dur": actual_dur})
    print(f"  Audio: {actual_dur:.2f}s")
    time.sleep(0.3)  # Rate limit
```

## Step 3: Build Timeline

Calculate scene timing from actual audio durations:

```python
BUFFER = 2.0  # seconds after audio finishes
current_time = 0
scenes = []

for item in results:
    scene_dur = max(item["audio_dur"] + BUFFER, 6.0)
    scenes.append({
        "label": item["label"],
        "filename": item["filename"],
        "start": round(current_time, 1),
        "duration": round(scene_dur, 1),
        "audio_dur": round(item["audio_dur"], 2),
    })
    print(f"  {item['label']}: start={current_time:.1f}s  dur={scene_dur:.1f}s  audio={item['audio_dur']:.2f}s")
    current_time += scene_dur

total = round(current_time, 1)
print(f"\nTotal: {total:.1f}s ({total/60:.1f}min)")

# Force-align to prevent overlap
for i in range(len(scenes) - 1):
    end = scenes[i]["start"] + scenes[i]["duration"]
    if end > scenes[i+1]["start"]:
        scenes[i]["duration"] = round(scenes[i+1]["start"] - scenes[i]["start"] - 0.1, 1)

with open(f"{AUDIO_DIR}/timeline.json", "w", encoding="utf-8") as f:
    json.dump(scenes, f, ensure_ascii=False, indent=2)
```

**Key formula:** `scene_duration = max(audio_duration + 2.0, 6.0)`

The 2-second buffer ensures narration finishes naturally before the scene transitions. Never trim audio to fit — shorten text instead.

## Step 4: Wire Audio into Composition

Each audio file becomes a `<audio>` element in `<body>`, outside the composition div:

```html
<body>
  <div data-composition-id="main" data-width="1080" data-height="1920"
       data-start="0" data-duration="189">

    <!-- Scene 1 -->
    <div id="scene1" class="scene">...</div>
    <!-- Scene 2 -->
    <div id="scene2" class="scene" style="opacity:0">...</div>
    <!-- ... -->
  </div>

  <!-- Audio clips OUTSIDE the composition div -->
  <audio id="narr-s01" data-start="0" data-duration="8.4" data-track-index="2"
         src="audio/project/pd_s01.mp3" data-volume="1"></audio>
  <audio id="narr-s02" data-start="8.4" data-duration="14.7" data-track-index="2"
         src="audio/project/pd_s02.mp3" data-volume="1"></audio>
  <!-- ... -->
</body>
```

**Critical attributes:**
- `data-start` — must match scene start time exactly
- `data-duration` — explicit seconds, **never `"auto"`**
- `data-track-index="2"` — all narration on the same track
- `src` — path relative to the HTML file (use `../` if HTML is in a subdirectory)

**No overlaps:** `data-start + data-duration <= next clip's data-start`. If the lint reports `overlapping_clips_same_track`, the `data-duration` values may have floating-point precision issues — round down by 0.1s.

## Step 5: Lint & Inspect

```bash
npx hyperframes lint          # Must be 0 errors
npx hyperframes inspect --at 2,10,45,100 --no-contrast --timeout 30000
```

Common lint fixes:
- `overlapping_clips_same_track` → reduce `data-duration` by 0.1s
- `composition_file_too_large` → warning only, acceptable for single-file compositions

## Step 6: Render & Verify

```bash
npx hyperframes render
ffprobe renders/output.mp4    # Must show both video AND audio streams
```

## Troubleshooting

| Problem | Cause | Fix |
|:---|:---|:---|
| Rendered video has no audio | `<audio>` inside composition div, or `data-duration="auto"` | Move outside div, use explicit seconds |
| Audio doesn't match scenes | Single continuous file | Split into per-scene segments |
| `overlapping_clips_same_track` | Audio durations overlap on same track | Round down `data-duration`, verify timeline alignment |
| Narration cut off mid-sentence | Audio trimmed, or text too long | Shorten text, regenerate, never trim audio |
| Scenes feel rushed | Too many words per scene | Reduce text to 60-70% capacity |

## Related References

- `external-tts.md` — MiniMax API details, speed calibration, overlap prevention
- `narration.md` — Script writing: pacing, tone, structure, opening lines
- `tts.md` — Built-in local TTS (Kokoro-82M)
- `news-flash-images.md` — Pexels background images for News Flash style
