# VapourSynth AI Pipeline (Apple Silicon + CoreML) — Setup & Runbook

This repo runs an ML-based video restoration pipeline on **Apple Silicon macOS** using:

- **VapourSynth** (`.vpy` scripts)
- **vs-mlrt / vsort** plugin (ONNX Runtime backend + **CoreMLExecutionProvider**)
- Models under `third_party/vs-mlrt/models/`

**Goals**
- Better artifact removal + detail reconstruction via ML models
- Keep results **natural** and **temporally stable**
- Prioritize **stability/quality**, then speed
- **No CUDA** (CoreML / CPU only)

---

## 1) Requirements

### Hardware / OS
- Apple Silicon Mac (M1/M2/M3). Tested on **M2 Max, 32GB**.

### Core tools
- Homebrew
- VapourSynth (includes `vspipe`)
- One source plugin (e.g. `ffms2`)
- ffmpeg (encoding)

---

## 2) Repo Layout (current)

```

video-restore/
    scripts/
        coreml_smoke.vpy
        real_esrgan_coreml.vpy
    third_party/
        vs-mlrt/
            plugins/
                libvsort.dylib
                deps/
                    libonnxruntime.1.23.2.dylib
        models/
            RealESRGANv2/
                Ani4Kv2-G6i2-UltraCompact.onnx
        vsmlrt.py
    inputs/
        clip.avi
    outputs/
````

> The VapourSynth plugin is `libvsort.dylib`.  
> `libonnxruntime*.dylib` is a dependency library used by the plugin.
> Those two were built manually for this practice.
---

## 3) Clean Installation (Homebrew)

### 3.1 Install Homebrew (if needed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 3.2 Install Python + VapourSynth + ffmpeg

> Recommended: use a consistent Homebrew Python (example uses 3.14).

```bash
brew update
brew install python@3.14
brew install vapoursynth
brew install ffmpeg
```

Verify:

```bash
/opt/homebrew/opt/python@3.14/bin/python3.14 -V
which vspipe
vspipe --version
```

Expected:

* `vspipe` at `/opt/homebrew/bin/vspipe` (or on PATH)

### 3.4 Install a source plugin (ffms2)

This repo uses `core.ffms2.Source(...)`.

```bash
brew install vapoursynth-ffms2
```

Verify `ffms2` is available:

```bash
vspipe --info - <<'PY'
import vapoursynth as vs
from vapoursynth import core
print("ffms2 available:", hasattr(core, "ffms2"))
PY
```
### 3.5 Model Download

Check releases from https://github.com/AmusementClub/vs-mlrt/tree/master to download the most recent models.

---

## 4) Plugin Path Setup (use repo plugin bundle)

### 4.1 Export `VAPOURSYNTH_PLUGIN_PATH`

From the repo root:

```bash
cd /path/to/video-restore
export VAPOURSYNTH_PLUGIN_PATH="$(pwd)/third_party/vs-mlrt/plugins"
echo $VAPOURSYNTH_PLUGIN_PATH
```

> This prevents accidentally loading a system/brew `libvsort.dylib`

---

## 5) Smoke Test (CoreML + SR model loads)

Create `scripts/coreml_smoke.vpy`:

```python
import vapoursynth as vs
from vapoursynth import core
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent

# Optional: explicit plugin load (useful if you don't want to rely on env var)
# PLUG = ROOT / "third_party" / "vs-mlrt" / "plugins" / "libvsort.dylib"
# core.std.LoadPlugin(path=str(PLUG))

# SR models usually require 3-plane RGB float input
clip = core.std.BlankClip(width=640, height=360, format=vs.RGBS, length=120)

MODEL = str(
    ROOT / "third_party" / "vs-mlrt" / "models" / "RealESRGANv2" / "Ani4Kv2-G6i2-UltraCompact.onnx"
)

out = core.ort.Model(
    clip, MODEL,
    provider="COREML",
    ml_program=1,     # 1=MLProgram; try 0 if CoreML coverage is low
    fp16=True,
    num_streams=1,
    verbosity=2,
    builtin=False,
)

out.set_output()
```

Run:

```bash
vspipe --info scripts/coreml_smoke.vpy
```

Expected:

* No `unknown provider COREML`
* Output clip prints successfully (RGBS float)
* Resolution typically doubles if the model is x2 SR (e.g. 640×360 → 1280×720)

You may see warnings about node assignment; that’s normal.

---

## 6) Real Video SR → Encode

### 6.1 vpy script (read → RGBS → SR → YUV for y4m)

Create `scripts/real_esrgan_coreml.vpy`:

```python
import vapoursynth as vs
from vapoursynth import core
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent

src = core.ffms2.Source(str(ROOT / "inputs" / "clip.avi"))

# Convert to RGB float for the model
rgb = core.resize.Bicubic(src, format=vs.RGBS, matrix_in_s="709")

MODEL = str(
    ROOT / "third_party" / "vs-mlrt" / "models" / "RealESRGANv2" / "Ani4Kv2-G6i2-UltraCompact.onnx"
)

out = core.ort.Model(
    rgb, MODEL,
    provider="COREML",
    ml_program=1,
    fp16=True,
    num_streams=1,
    tilesize=[512, 512],   # recommended for >1080p / 4K stability
    overlap=[32, 32],
    verbosity=2,
    builtin=False,
)

# y4m requires YUV or Gray (not RGB), so convert before piping to ffmpeg
yuv = core.resize.Bicubic(out, format=vs.YUV420P10, matrix_s="709")
yuv.set_output()
```

### 6.2 Encode with FFmpeg

#### Option A: HEVC VideoToolbox (hardware) + software fallback

```bash
mkdir -p outputs

vspipe -c y4m scripts/real_esrgan_coreml.vpy - | \
ffmpeg -f yuv4mpegpipe -i - \
  -c:v hevc_videotoolbox -allow_sw 1 \
  -pix_fmt yuv420p10le -b:v 12000k \
  outputs/coreml_test.mp4
```

#### Option B: Software x265 (most robust)

```bash
vspipe -c y4m scripts/real_esrgan_coreml.vpy - | \
ffmpeg -f yuv4mpegpipe -i - \
  -c:v libx265 -crf 18 -preset medium -pix_fmt yuv420p10le \
  outputs/coreml_test_x265.mp4
```

---

## 7) Common Pitfalls & Fixes

### `operator(): unknown provider COREML`

* Cause: `libvsort.dylib` was built without CoreML support or the wrong plugin is loaded.
* Fix: ensure `VAPOURSYNTH_PLUGIN_PATH` points to the repo plugin directory and the plugin is CoreML-enabled.

### `operator(): expects 3 input planes`

* Cause: model expects RGB (3 planes), but input is Gray/YUV.
* Fix: convert to `vs.RGBS` before `core.ort.Model(...)`.

### `Error: can only apply y4m headers to YUV and Gray...`

* Cause: you are piping RGB to `vspipe -c y4m`.
* Fix: convert output to `YUV420P10` or `YUV420P8` inside the `.vpy`.

### `Warning: No entry point found in libonnxruntime...`

* Cause: VapourSynth is scanning ORT dylib as a plugin.
* Fix: move ORT dylibs into `plugins/deps/` and add rpath (Section 4.2).

### `hevc_videotoolbox: cannot create compression session: -12908`

* Cause: hardware encoder busy or unsupported format combo.
* Fix: add `-allow_sw 1`, keep a consistent 10-bit pipeline (`yuv420p10le`), or use software encode (`libx265`).

---

## 8) Performance Knobs (Apple Silicon)

Tune in this order:

1. CoreML mode:

* `ml_program=1` (MLProgram)
* try `ml_program=0` if CoreML coverage is low

2. Precision:

* `fp16=True` (recommended)
* try `fp16=False` if needed

3. Tiling (important for >1080p / 4K):

* `tilesize=[512,512]` → `[640,640]` → `[768,768]`
* `overlap=[16,16]` or `[32,32]`

> ORT may print “supported by CoreML: X nodes”.
> Low coverage means many ops fall back to CPU → slower.

---

## 9) Quick Start (copy/paste)

```bash
cd /path/to/video-restore
export VAPOURSYNTH_PLUGIN_PATH="$(pwd)/third_party/vs-mlrt/plugins"

# smoke test
vspipe --info scripts/coreml_smoke.vpy

# encode
mkdir -p outputs
vspipe -c y4m scripts/real_esrgan_coreml.vpy - | \
ffmpeg -f yuv4mpegpipe -i - \
  -c:v hevc_videotoolbox -allow_sw 1 \
  -pix_fmt yuv420p10le -b:v 12000k \
  outputs/coreml_test.mp4
```
