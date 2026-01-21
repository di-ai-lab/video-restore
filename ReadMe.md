# Long Vacation (XVID) Restoration Playground (for macOS / Apple Silicon)

![Cover Image](images/cover.png)

A practical, reproducible restoration workflow for older TV sources (e.g., XVID AVI).  
The focus is **real-world usability**: reduce compression artifacts, upscale to a sensible target (4:3 preserved), and leave room for an AI pipeline without making the result look “fake” or temporally unstable.

---

## Goals

1) **Improve perceived clarity** on a 4:3 TV source (XVID, 720×540, ~29.97 fps).  
2) **Avoid common failure modes**: combing/interlacing artifacts, over-sharpening, “plastic faces”, temporal flicker.  
3) Keep the workflow **iterative and measurable**: always test on a short clip and compare A/B outputs.  
4) Learn the building blocks so the same recipe generalizes to other legacy sources.

---

## Approach

Two-layer workflow:

### 1) Non-AI baseline (FFmpeg-only)
A “safe” reference that:
- checks interlacing properly (tools + eyes)
- applies conservative denoise/deblock
- upscales 2× to **1440×1080** (preserves 4:3 without stretching)

This baseline is valuable because it sets a reality check: if AI doesn’t clearly beat it (without artifacts), the AI settings are wrong.

### 2) AI pipeline (planned)
A VapourSynth + vs-mlrt pipeline (CoreML on Apple Silicon) to improve:
- **artifact removal** (denoise/deblock) before upscaling  
- **2× super-resolution** with better detail reconstruction  
- optional **temporal stabilization** to reduce flicker

> The AI pipeline section will evolve as experiments continue. The current README keeps the “contract” and design intent, while detailed commands live in the appendix.

---

## Environment

- **Machine**: Apple Silicon (M2 Max)
- **OS**: macOS Tahoe 26.2
- **Source example**: XVID AVI (MPEG-4 ASP), 720×540, DAR 4:3, ~29.97 fps

---

## Quickstart (Current: FFmpeg Baseline)

### Install FFmpeg
```bash
brew install ffmpeg
ffmpeg -version
ffprobe -version
````

### Produce a baseline output (test clip recommended)

```bash
ffmpeg -i clip.avi -vf "\
hqdn3d=1.5:1.5:6:6,\
deblock=filter=strong:block=8,\
scale=1440:1080:flags=lanczos" \
-c:v libx264 -crf 16 -preset slow -c:a copy \
clip_clean_scale.mp4
```

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

## Documentation

* **FFmpeg commands + step-by-step reasoning + outputs/logs**: `docs/ffmpeg-playbook.md`
* **AI pipeline plan (VapourSynth + vs-mlrt)**: `docs/ai-pipeline.md`

---

## Roadmap

* [ ] Wrap the FFmpeg commands into scripts under `scripts/` (avoid copy/paste)
* [ ] Create a dedicated Python venv (e.g., `vsr/`) and minimal VapourSynth read test
* [ ] Integrate vs-mlrt (CoreML backend)
* [ ] Implement model denoise/deblock → model SR (2×)
* [ ] Add evaluation harness: crops, motion scenes, subtitles, flicker checks

---

## License

Personal learning project / notes. No warranties.