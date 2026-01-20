# Long Vacation (XVID) Restoration Playground (macOS / Apple Silicon)

[Cover Image](images/cover.png)

A small, hacky-but-structured video restoration playground for older TV sources (e.g., XVID AVI).  
Goal: learn and document a practical workflow to improve perceived clarity while avoiding common pitfalls (interlacing artifacts, over-sharpening, “plastic faces”, temporal flicker).

This README summarizes what’s done so far and captures the rationale behind each step, so I can iterate later (especially when adding the AI pipeline).

---

## Environment

- **Machine**: Apple Silicon (M2 Max)
- **OS**: macOS Tahoe 26.2
- **Source example**: XVID AVI (MPEG-4 ASP), 720×540, DAR 4:3, ~29.97 fps

---

## Why This Workflow

Old TV/AVI sources often have:
- uncertain or missing interlacing metadata
- compression artifacts (blocking, ringing, mosquito noise)
- limited true detail (upscaling too aggressively can hallucinate or look “AI-ish”)
- temporal instability risks (frame-by-frame processing can flicker)

So the workflow is:
1) **Inspect** first (don’t guess).
2) **Confirm** interlacing behavior with both tools and eyes.
3) Build a **baseline** that is safe and measurable.
4) Only then move to **AI-based restoration** (planned).

---

## Step 0 — Install FFmpeg (Homebrew)

**Purpose:** Use FFmpeg/FFprobe for inspection, sampling, baseline filtering, and encoding.

```bash
brew install ffmpeg
ffmpeg -version
ffprobe -version
````

---

## Step 1 — Inspect Source Metadata

**Purpose:** Understand resolution, aspect ratio, framerate, codec, and audio.
This informs how to upscale correctly (e.g., preserve 4:3) and whether to suspect interlacing/telecine.

```bash
ffprobe -hide_banner -select_streams v:0 -show_streams "input.avi" \
| grep -E "width|height|sample_aspect_ratio|display_aspect_ratio|field_order|r_frame_rate|avg_frame_rate|codec_name"
```

**Typical output / expectations (for this project):**

* `720x540`, `SAR 1:1`, `DAR 4:3`
* `29.97 fps`
* `field_order=unknown` (common for AVI/XVID; not definitive)

---

## Step 2 — Create a Short Test Clip (30–60s)

**Purpose:** Iterate quickly and do A/B testing without spending time on full-length processing.

```bash
ffmpeg -ss 00:10:00 -t 00:00:30 -i "input.avi" -c copy clip.avi
```

**Tips for selecting a clip:**

* include **faces**, **motion** (walking, camera pan), and **subtitles**
* include **dark scenes** if possible (banding/blocking is easiest to see)

---

## Step 3 — Check Interlacing (Tool + Eyes)

### 3.1 Detect with `idet`

**Purpose:** Determine if the content is interlaced or progressive.
Interlaced video can create “combing” artifacts if treated as progressive (and filters/models can amplify it).

```bash
ffmpeg -i clip.avi -vf idet -an -f null - 2>&1 | tail -n 60
```

**How to interpret:**

* `Multi frame detection: Progressive` dominant → likely progressive overall
* `TFF/BFF` dominant → likely interlaced
* `Undetermined` can be high in single-frame detection for noisy/compressed sources

**Project finding so far:** `Multi frame detection` strongly favored **progressive** on the test clip.

### 3.2 Visual Check (Combing)

**Purpose:** Confirm what the metrics suggest.
Even if metadata is unknown, **combing** in motion edges is the telltale sign.

```bash
ffmpeg -ss 00:10:05 -i clip.avi -frames:v 1 frame.png
open frame.png
```

**Look for:**

* horizontal “teeth” on edges (arms, faces, subtitles) during motion

[interlace_check](images/interlace_check.png)

**Project finding so far:** combing (if present) is not obvious.

---

## Step 4 — Deinterlace A/B Test (Only If Needed)

**Purpose:** If uncertain, create two outputs and compare.
If deinterlacing doesn’t help, don’t keep it (it can soften detail).

### A) Treat as progressive

```bash
ffmpeg -i clip.avi -vf "scale=1440:1080" \
-c:v libx264 -crf 16 -preset slow -c:a copy \
A_prog.mp4
```

### B) Deinterlace only when needed (safer)

```bash
ffmpeg -i clip.avi -vf "bwdif=mode=send_frame:parity=auto:deint=interlaced,scale=1440:1080" \
-c:v libx264 -crf 16 -preset slow -c:a copy \
B_bwdif_auto.mp4
```

**Project finding so far:** A and B looked **very similar**, so the working assumption is **progressive** processing.

---

## Step 5 — 2× Upscale (Preserve 4:3)

**Purpose:** Increase viewing resolution without distorting aspect ratio.
For a 4:3 source:

* 1080-height target is naturally **1440×1080** (still 4:3)

```bash
ffmpeg -i clip.avi -vf "scale=1440:1080:flags=lanczos" \
-c:v libx264 -crf 16 -preset slow -c:a copy \
clip_up2x_1440x1080.mp4
```

**Notes:**

* Avoid forcing 1920×1080 unless you intentionally add pillarbox bars (otherwise faces get stretched).
* Start with 2×. Jumping directly to 4K can emphasize artifacts or hallucinate detail.

---

## Step 6 — Baseline “Safe” Restoration (FFmpeg-only)

**Purpose:** Establish a non-AI baseline:

* reduce compression noise (temporal/spatial)
* reduce blocking/ringing
* then upscale

This gives a reference point to measure how much AI actually improves (vs just “sharper but fake”).

```bash
ffmpeg -i clip.avi -vf "\
hqdn3d=1.5:1.5:6:6,\
deblock=filter=strong:block=8,\
scale=1440:1080:flags=lanczos" \
-c:v libx264 -crf 16 -preset slow -c:a copy \
clip_clean_scale.mp4
```

**What to look for:**

* fewer macroblocks / mosquito noise around edges
* subtitles edges cleaner
* faces still natural (no waxy skin)
* minimal added flicker

**Project finding so far:** baseline looked *slightly more refined*, but not “clear enough”, suggesting AI restoration/super-resolution is needed next.

---

## Planned Next Step — AI Pipeline (VapourSynth + vs-mlrt)

This section is intentionally a placeholder; it will be expanded as the work continues.

### Why VapourSynth

**Purpose:**

* write a reproducible, modular pipeline in Python (`.vpy`)
* avoid manual frame extraction/merge
* better control over temporal behavior (reduce flicker)
* easier A/B testing and parameter iteration

### Why vs-mlrt (CoreML backend on Apple Silicon)

**Purpose:**

* run denoise/deblock and super-resolution models efficiently on Apple Silicon
* keep the pipeline scriptable and iterative

### Pipeline Idea (High Level)

1. Source read-in
2. (Optional) deinterlace only if needed
3. Denoise / deblock (model-based)
4. 2× super-resolution (model-based) to 1440×1080
5. Optional mild sharpening / temporal stabilization
6. Encode to final deliverable

---

## Practical Notes / Learnings So Far

* `field_order=unknown` is common for AVI/XVID; it doesn’t mean it’s interlaced.
* Use both **idet** and **visual inspection**; if deinterlacing A/B is identical, don’t keep deinterlacing.
* For 4:3 sources, **1440×1080** is a clean “1080-ish” target without stretching.
* Establishing a **FFmpeg-only baseline** is valuable to avoid chasing “AI sharpness” that looks fake.

---

## Next TODOs

* [ ] Set up a dedicated Python venv (e.g., `vsr/`) for VapourSynth + ML tooling
* [ ] Minimal VapourSynth read test (`vspipe`) on the source AVI
* [ ] Integrate vs-mlrt CoreML backend
* [ ] Implement: model denoise/deblock → model SR (2×)
* [ ] Add evaluation snippets (side-by-side crops, motion scenes, subtitles checks)
* [ ] Decide final encode strategy (H.264 vs HEVC) for size vs quality

---

## License

Personal learning project / notes. No warranties.