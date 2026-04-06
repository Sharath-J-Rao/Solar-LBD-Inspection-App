# FLIR Thermal Analysis Application

> **Version:** 3.30.25.20 · **Status:** Production Ready  
> **Architecture:** Hybrid Web + Desktop (FastAPI + Tkinter)

A production-grade thermal imaging analysis platform built for FLIR R-JPEG cameras. Automates component detection, temperature extraction, QC review, and DOCX/PDF report generation — accessible via a web browser or a native desktop GUI.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Running the Application](#running-the-application)
- [Runtime Modes](#runtime-modes)
- [Folder & Output Structure](#folder--output-structure)
- [Configuration](#configuration)
- [Web API Reference](#web-api-reference)
- [Module / Section Map](#module--section-map)
- [Performance Notes](#performance-notes)
- [Known Bugs & Fix Registry](#known-bugs--fix-registry)
- [Contributing & Versioning](#contributing--versioning)
- [License](#license)

---

## Overview

This application processes batches of FLIR radiometric JPEG (R-JPEG) thermal images through a fully automated pipeline:

1. **Thermal data extraction** — raw DN → calibrated temperature using the FLIR SDK two-path atmospheric model (matches FLIR Thermal Studio to ±0.1 °C).
2. **Component detection** — YOLOv8-based object detection identifies electrical components in each image.
3. **Spot temperature measurement** — adaptive hot-spot spotting with IEC/ASTM threshold grading.
4. **QC review** — browser-based or desktop canvas viewer for manual annotation correction.
5. **Report generation** — per-component DOCX + PDF reports from a Word template, with full image embedding and structured metadata tables.
6. **Verification export** — structured CSV extraction from generated DOCX files for cross-checking.

---

## Key Features

| Category | Feature |
|---|---|
| **Detection** | YOLOv8 inference with configurable confidence thresholds, IoU NMS, geometric area/aspect-ratio filtering |
| **Temperature** | `flyr`-based Planck inversion (primary), `flirpy` raw-array fallback; fixed atmospheric transmission formula |
| **Metadata** | Persistent `exiftool` singleton (~1–5 ms/call vs ~100–200 ms cold-start); full FLIR MakerNotes extraction |
| **Preprocessing** | Variance-adaptive CLAHE + optional unsharp sharpening; thread-local objects for thread safety |
| **Deduplication** | SHA-256 content hashing tracks processed images across sessions; `not_marked/` auto-cleanup |
| **QC Viewer** | Point & rectangle annotation modes; manual temperature editing; spot spatial ID assignment |
| **Report generation** | Template pre-loaded into memory (zero disk I/O per row); image cache; `docx2pdf` via Word COM |
| **Web UI** | FastAPI REST API + browser frontend; live job progress polling; directory browser; cancel support |
| **Desktop UI** | Full Tkinter GUI with batch processing, settings, training, and manual correction panels |
| **Headless mode** | Import `CoreEngine.*` directly; no display required |
| **GPU support** | CUDA auto-detected via PyTorch; falls back to CPU gracefully |
| **Multi-user safety** | `filelock`-based file locking; per-job isolated working directories; WAL audit logging |
| **Compliance** | ASTM E1934 threshold grading; IEC 62304 SOUP register; NIST-traceable unit test scaffold |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  StartupController                   │
│  Detects runtime mode → starts appropriate interface │
└────────────┬──────────────────────┬─────────────────┘
             │                      │
     ┌───────▼──────┐      ┌────────▼────────┐
     │  FastAPI +   │      │  Tkinter MainApp │
     │  Web Browser │      │  (Desktop GUI)   │
     └───────┬──────┘      └────────┬─────────┘
             │                      │
             └──────────┬───────────┘
                        │
               ┌────────▼────────┐
               │   CoreEngine    │  ← Headless API
               │  (Section 16)   │
               └────────┬────────┘
                        │
       ┌────────────────┼────────────────┐
       │                │                │
  ┌────▼────┐   ┌───────▼──────┐  ┌─────▼──────┐
  │  YOLO   │   │  Thermal     │  │  Report    │
  │Detection│   │  Extraction  │  │ Generator  │
  └─────────┘   └──────────────┘  └────────────┘
```

The application ships as a **single Python file** paired with a **companion HTML file** (`flir_web_ui.html`) for the browser front-end. Both files must always be deployed together.

---

## Prerequisites

### System Requirements

- **Python:** 3.10+
- **OS:** Windows 10/11 (primary target), Linux (headless/web mode), macOS (limited)
- **exiftool:** Must be installed and on `PATH` — [https://exiftool.org](https://exiftool.org)
- **GPU (optional):** CUDA-capable NVIDIA GPU for faster YOLO inference
- **Microsoft Word (optional):** Required for DOCX → PDF conversion via `docx2pdf` (Windows COM)

### Python Dependencies

#### Core (required)

```
opencv-python
numpy
Pillow
ultralytics       # YOLOv8
python-docx
```

#### Recommended

```
flyr              # Studio-accurate temperature conversion (matches FLIR Thermal Studio ±0.1°C)
flirpy            # Raw thermal array fallback; broad camera model support
torch             # Explicit GPU control and CUDA detection
scipy             # Bad-pixel correction and advanced glint detection
PyExifTool        # Persistent exiftool singleton (5-10× faster than subprocess)
filelock          # Multi-user file locking safety
PyYAML            # data.yaml validation for YOLO training
```

#### Web Mode (optional)

```
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
pydantic>=2.0.0
python-multipart>=0.0.9
```

#### Report Generation

```
docx2pdf          # Word COM PDF export (Windows only)
```

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-org/flir-thermal-app.git
cd flir-thermal-app

# 2. Create a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate           # Windows

# 3. Install core dependencies
pip install opencv-python numpy Pillow ultralytics python-docx

# 4. Install recommended extras
pip install flyr flirpy torch scipy PyExifTool filelock PyYAML

# 5. Install web mode extras (optional)
pip install "fastapi>=0.110.0" "uvicorn[standard]>=0.29.0" "pydantic>=2.0.0" python-multipart

# 6. Install report generation (Windows only)
pip install docx2pdf

# 7. Verify exiftool is available
exiftool -ver
```

---

## Running the Application

```bash
# Default — auto-detects best mode (hybrid web+desktop on Windows with display)
python flir_thermal_app_v3_30_25_21.py

# Explicit web mode (browser UI on http://127.0.0.1:8000)
FLIR_RUNTIME=web python flir_thermal_app_v3_30_25_21.py

# Custom host/port
FLIR_PORT=9000 FLIR_HOST=0.0.0.0 python flir_thermal_app_v3_30_25_21.py

# Explicit desktop mode
FLIR_RUNTIME=local python flir_thermal_app_v3_30_25_21.py

# Headless / server mode (import CoreEngine directly)
FLIR_RUNTIME=headless python flir_thermal_app_v3_30_25_21.py
```

---

## Runtime Modes

The mode is determined at startup in this priority order:

| Priority | Condition | Mode |
|---|---|---|
| 1 | `FLIR_RUNTIME` env var set | Explicit override |
| 2 | FastAPI/uvicorn already in `sys.modules` | `web` |
| 3 | Linux + no `$DISPLAY` / `$WAYLAND_DISPLAY` | `headless` |
| 4 | Tkinter display probe succeeds | `local` (desktop) |
| 5 | Fallback | `local` |

### Mode Behaviour

- **`local`** — Full Tkinter desktop GUI. No web server started.
- **`web`** — FastAPI server started; browser opened automatically; Tkinter available as fallback if web server stops.
- **`headless`** — No GUI, no web server. Import and call `CoreEngine.*` programmatically.

---

## Folder & Output Structure

After a batch detection run the application creates the following structure under your chosen output directory:

```
<output_dir>/
├── annotated/
│   ├── original_ir/          # Clean CLAHE thermal images (no overlays)
│   ├── annotated/            # Annotated images with spot overlays
│   ├── detection_jsons/      # Per-image JSON sidecars (spots, metadata, Planck params)
│   └── verification_report_<timestamp>.csv
├── visual_annotated/
│   ├── visual/               # Clean visual (RGB) copies
│   └── annotated/            # Visual images with overlays
├── not_marked/               # Images with insufficient detections
│   └── jsons/                # Partial JSON for not_marked images
├── reports/                  # Generated DOCX + PDF reports
├── qc_annotated/             # QC-corrected annotated images
└── processed_images_hash.json  # SHA-256 deduplication registry
```

---

## Configuration

Application settings are stored in a `.ini`-style config file loaded at startup. Key sections:

| Setting | Description | Default |
|---|---|---|
| `NMS_IOU_Threshold` | IoU threshold for non-maximum suppression | `0.55` |
| `ConfidenceThreshold` | Minimum YOLO detection confidence | configurable |
| `SpotBoxSize` | Pixel size of temperature sampling box | configurable |
| `UseOverride` (ATMOSPHERE) | Apply manual atmospheric parameter overrides | `False` |
| `TempUnit` | Display unit for temperatures (`C` or `F`) | `C` |

Environment variable overrides:

| Variable | Purpose |
|---|---|
| `FLIR_RUNTIME` | Force runtime mode (`web`, `local`, `headless`) |
| `FLIR_PORT` | Web server port (default: `8000`) |
| `FLIR_HOST` | Web server bind address (default: `127.0.0.1`) |
| `FLIR_JOBS_DIR` | Base directory for per-job isolated working directories |

---

## Web API Reference

Base URL: `http://<host>:<port>/api/v1`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Server liveness check |
| `GET` | `/version` | Application version |
| `POST` | `/shutdown` | Graceful server shutdown |
| `POST` | `/detect/batch` | Start a batch detection job |
| `GET` | `/detect/status/{job_id}` | Poll job status and progress |
| `DELETE` | `/detect/cancel/{job_id}` | Cancel a running job |
| `POST` | `/detect/manual-correction` | Trigger manual correction pipeline |
| `POST` | `/detect/preview` | Preview YOLO input for an image |
| `GET` | `/qc/images` | List images available for QC review |
| `GET` | `/qc/image/{stem}` | Get QC state for a specific image |
| `POST` | `/qc/save` | Save QC spot temperature edits |
| `POST` | `/report/generate` | Generate DOCX/PDF reports |
| `GET` | `/report/performance` | Report generation performance stats |
| `GET` | `/browse/roots` | List filesystem root directories |
| `POST` | `/browse/directory` | List contents of a directory |

---

## Module / Section Map

The application is structured into 19 sections within the single-file codebase:

| Section | Content |
|---|---|
| 01 | Runtime mode detection & optional web dependency probing |
| 02 | Logging configuration |
| 03 | Stdlib & third-party imports |
| 04 | Global configuration constants |
| 05 | Persistent exiftool singleton |
| 06 | Thermal physics — Planck inversion, atmospheric model, raw-to-temperature |
| 07 | FLIR metadata extraction (exiftool / flirpy / flyr) |
| 08 | Image preprocessing — CLAHE, bad-pixel correction, glint detection |
| 09 | YOLO inference pipeline — NMS, geometric filtering, spatial ID assignment |
| 10 | Spot temperature extraction — adaptive hot-spot detection |
| 11 | Data persistence — CSV, JSON sidecars, annotation quality metadata |
| 12 | `ThermalEditor` — canvas-based manual annotation widget |
| 13 | `QCViewer` — QC review panel with spot editing |
| 14 | Report generation — template population, image embedding, DOCX/PDF export |
| 15 | `MainApp` — Tkinter desktop GUI (settings, batch, training, manual correction) |
| 16 | `CoreEngine` — headless programmatic API |
| 17 | FastAPI layer — REST routes, job manager, request/response models |
| 18 | `StartupController` — mode dispatch, health polling, browser open |
| 19 | Entrypoint — `__main__` dispatch |

---

## Performance Notes

Report generation was optimised from ~60 minutes → ~5 minutes for 240 reports:

| Optimisation | Saving |
|---|---|
| **PERF-001** Template pre-load into `bytes` — `Document(BytesIO(...))` eliminates per-row disk reads | ~1.6 min |
| **PERF-002** Image cache — identical source images decoded/resized only once per batch | variable |
| **PERF-003** *(reverted in v3.30.25.20)* Batch PDF conversion reverted to per-file `docx2pdf` via Word COM (LibreOffice JVM approach removed) | — |

Metadata extraction is accelerated by a persistent `exiftool` singleton (~1–5 ms/call vs ~80–200 ms cold-start subprocess). The singleton is thread-safe via a module-level lock and is torn down cleanly at exit via `atexit`.

---

## Known Bugs & Fix Registry

The codebase maintains a detailed inline change registry. Major fix categories:

| ID | Area | Summary |
|---|---|---|
| `BUG-REFL-TEMP-001/002` | Temperature | FLIR factory 0 K sentinel for `ReflectedApparentTemperature` now replaced with 20.0 °C default |
| `BUG-SGM-RANGE-001` | Temperature | Range Max/Min Kelvin→Celsius double-conversion and missing safety clamps fixed |
| `BUG-NEGTEMP-001` | Temperature | Genuine negative reflected temperatures (e.g. −6 °C) no longer clamped as sentinel values |
| `BUG-UNIT-001` | Reports | Range Max/Min and Refl. Temp now correctly converted to °F when `TempUnit=F` |
| `BUG-NEGZERO-001` | Reports | `−0.00` display artifact eliminated via `_fmt_celsius()` clamp helper |
| `BUG-FILENAME-001` | Reports | `Report_` prefix removed from DOCX/PDF output filenames |
| `BUG-VERIFY-001` | Reports | Verification CSV completely rewritten to produce one structured row per report |
| `BUG-DUP-001..004` | Pipeline | SHA-256 deduplication; `not_marked/` auto-cleanup after batch |
| `BUG-MC-001/002` | Pipeline | Manual corrections now write all four output folders (`original_ir/`, `visual/`, `annotated/`, `qc_annotated/`) |
| `BUG-PB-01..16` | CoreEngine | Full `CoreEngine.process_batch` rewrite — 38-column CSV, JSON sidecars, live progress, cancel support, output folder parity |
| `BUG-INFERENCE-001..008` | CoreEngine | CoreEngine inference parity with Tkinter pipeline — NMS, geometric filter, spatial IDs, adaptive CLAHE, atmosphere overrides |
| `FIX-MANUALCORR-001` | QCViewer | No spot limit in `ThermalEditor`; full four-folder image output on QC save |

---

## Contributing & Versioning

The version is defined in a single constant:

```python
APP_VERSION = "3.30.25.20"  # ← change this one line to bump the version
```

This propagates automatically to window titles, log banners, JSON annotation fields, CSV headers, and the About dialog.

The companion HTML file `flir_web_ui.html` **must always be committed and delivered alongside** any change to the Python file. Never commit one without the other.

---

## License

*License information not specified in source. Contact the repository owner.*

---

*Generated from source `flir_thermal_app_v3_30_25_21.py` — v3.30.25.20*
