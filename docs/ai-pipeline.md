# AI Pipeline (Planned): VapourSynth + vs-mlrt (CoreML on Apple Silicon)

This document is a **living design** for the AI-based restoration pipeline.
The goal is to go beyond the FFmpeg-only baseline by improving artifact removal and reconstructing details via ML models, while keeping results natural and temporally stable.

---

## Why VapourSynth

* Scriptable pipeline in Python (`.vpy`)
* Avoids manual “extract frames → run model → reassemble” overhead
* Easier A/B testing and incremental iteration
* Better control of temporal behavior (reduce flicker)

---

## Why vs-mlrt (CoreML backend)

* Efficient execution on Apple Silicon
* Convenient model-based filters for denoise / SR
* Fits a modular pipeline design

---

## Pipeline Idea (High Level)

1. Source read-in
2. Optional deinterlace only if needed
3. ML artifact removal (denoise / deblock)
4. ML 2× super-resolution to 1440×1080
5. Optional temporal stabilization / mild sharpening
6. Encode to final deliverable

---

## Notes / Guardrails

* Prefer “natural” settings over aggressive sharpening (avoid plastic faces).
* Validate on short clips with motion + subtitles.
* Watch for flicker (frame-to-frame texture instability).
* Keep the FFmpeg-only baseline as a sanity check.

---

## TODO

* [ ] Confirm VapourSynth install on macOS Tahoe 26.2 (Apple Silicon)
* [ ] Verify source reading plugins (ffms2 / lwlibavsource)
* [ ] Integrate vs-mlrt (CoreML)
* [ ] Decide models and defaults for: denoise/deblock + 2× SR
* [ ] Add reproducible scripts and command wrappers under `scripts/`
