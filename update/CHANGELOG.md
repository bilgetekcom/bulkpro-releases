# BulkPro Changelog

All notable changes to BulkPro Enterprise Tool Suite are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [0.1.15] — 2026-06-08

### Performance

- **Excel Studio opens ~8× faster** (7.4 s → 0.9 s on a cold start). The 8 tool
  tabs were being constructed eagerly at module load along with their pandas /
  openpyxl import chains; now each tab is materialised only when first selected.
- **Productivity Suite opens ~40× faster** (4.2 s → 0.1 s). The 12 cross-app
  pages were being imported via `safe_import_page` at module load; they now
  load on the first click of their dashboard tile.
- PDF Studio is also a touch quicker (1.1 s → 0.6 s) as a side effect.

### Fixed

- **Hidden console window flashes** when modules spawn background tools
  (ffmpeg, tesseract, powercfg, etc.). A global `subprocess.Popen` wrapper now
  applies `CREATE_NO_WINDOW` to every spawn — no more cmd windows blinking
  during normal use.
- **Update feed signature verification.** The `version.json` published to
  GitHub Pages was being normalised from CRLF to LF during `git push`, so the
  bytes the updater verified no longer matched the bytes that had been signed
  locally — surfacing as "Update check failed" in Settings. The deploy script
  now writes the feed in LF, self-verifies the signature before publishing,
  and pins the file as binary via `.gitattributes`.

---

## [0.1.14] — 2026-06-08

### License & Cleanup

This release is a major slim-down. After auditing every AI model BulkPro ships,
two had non-commercial licenses (Coqui XTTS-v2 CPML; Meta NLLB-200 CC-BY-NC-4.0)
that we shouldn't have been bundling in a paid product. Both — and the tools
that used them — are gone. While in there, we also ripped out dead code, zombie
dependencies, the unused GPU PyTorch component, and a marginal subtitle tool
whose model footprint didn't justify the value to users.

### Removed

- **Video AI Dubbing Studio** (`video_engine.dubber`) — XTTS-v2 voice cloning
  was CPML-licensed; not safe for commercial distribution.
- **Subtitle translation** in the Subtitle Generator — NLLB-200 model weights
  are CC-BY-NC-4.0 (Meta).
- **AI Subtitle Generator** (`video_engine.subtitler`) — strategic decision:
  the per-tool Whisper model download (75 MB – 3 GB) wasn't justified by the
  fraction of users that reached for subtitles.
- **PyTorch CUDA component** (`ai_gpu`, ~2.55 GB) — CPU-only AI moving forward.
- **MediaPipe component** (`ai_video`) — turned out the jump-cut AI mode
  actually uses Whisper, not MediaPipe; MediaPipe was only imported by a long-
  dead `smart_social_crop()` static method with zero callers.
- **Zombie deps**: `pyannote.audio`, `spleeter`, the deprecated `VideoEngine`
  facade class (12 static methods, all unreachable).

### Changed

- Video Studio now has **13 tools** (was 15 — subtitler + dubber removed).
- `requirements/manifest.json` ships with **5 components** (was 7): `core`,
  `ai_base`, `ai_bg_remover`, `ai_ocr`, `ai_transcription`.
- `video_engine.jump_cut` AI mode moved under `ai_transcription` (its real
  dependency was always Whisper).
- BulkPro ships with **zero non-commercial-licensed model weights**. Every
  remaining model is MIT or Apache-2.0 (Whisper, rembg / u2net family,
  EasyOCR, Tesseract).

### Documentation

- New `docs/REMOVED_FEATURES.md`: central plain-language changelog of every
  removed feature with rationale and a website-copy audit checklist.
- Older internal reports (E2E, PERF, PROD_READINESS) got a "historical
  snapshot" banner that points at REMOVED_FEATURES.md.
- 6 app-local stale `requirements.txt`, ~15 K lines of one-shot i18n-audit /
  refactor scripts, and the legacy `video_lab` TR locale block — all gone.

### Notes

- Net diff across cleanup phases: ~19 K lines removed, mostly stale.
- `shared/updater.py` is unchanged this release, so installed copies of
  v0.1.13 will see this update through the in-app feed normally.

---

## [0.1.13] — 2026-06-07

### Fixed

- **Updater feed rejected**: `shared/updater.py` allow-list was missing `bilgetekcom.github.io`, so v0.1.11 / v0.1.12 builds refused to follow their own update feed ("Untrusted feed host" in logs). Added the host; the in-app update banner now actually surfaces new releases.
- **Config "Permission denied"**: `config.json` lived under `C:\Program Files\BulkPRO\` — every save attempt failed with `[Errno 13]` for non-admin users, silently dropping language / theme / AI device preferences. Canonical config now lives at `%LOCALAPPDATA%\BulkPro\config.json`. Existing installs still seed from the install-dir copy on first launch.
- **Self-Healing false-FATAL**: a `/COMPONENTS=core` install has no bundled `external/` directory, but `external` was on the FATAL `REQUIRED_DIRS` list — Self-Healing flagged perfectly healthy installs as "install incomplete". Moved to `OPTIONAL_DIRS` and reports as a `WARN` instead. The `ffmpeg` system-PATH fallback that was already in place (`_fix_imageio_env`) works as intended.

### Observability

- **Bootstrap log file**: pythonw.exe sends stdout to NUL, so the silent-launch hardening from v0.1.12 also silenced `[BOOTSTRAP] ...` lines. They now tee into `%LOCALAPPDATA%\BulkPro\logs\bootstrap.log` along with full pip / model-manager subprocess output. The fatal-error MessageBox shows the exact path so users can grab it without spelunking.

---

## [0.1.12] — 2026-06-07

### UX

- **Silent first-launch setup**: the slow-path bootstrap (clone Python, pip-install core deps, AI components, model sync) no longer flashes a black CMD window. `run.bat` now launches `bootstrap.py` via `pythonw.exe`, and every internal `pip`/`taskkill` subprocess uses `CREATE_NO_WINDOW`.
- **Install splash dialog**: while bootstrap runs, a native Windows.Forms progress window (PowerShell-based, no extra dependency) shows "Eksik bilesenler indiriliyor..." with a marquee progress bar. It auto-closes when bootstrap finishes — or if bootstrap crashes (parent-PID watchdog).
- **Fatal-error MessageBox**: failures inside `bootstrap.py` now surface via a native MessageBox (ctypes / user32) so pythonw users see a real error instead of a silent exit.

---

## [0.1.11] — 2026-06-07

### Maintenance

- **Deprecated tools removed**: `data_engine`, `text_tools`, and `file_search` modules are no longer shipped. Docs, tests, and Hub `APP_COLORS` entries cleaned up (~10 k LOC removed).
- **Stress harness**: New `stress/` directory provides a multi-engine load-test rig (fixture generator, per-engine harness, report aggregator) for audio, excel, image, pdf, and video pipelines.

### Auto-Update Fixes

- **Feed pipeline**: GitHub Pages feed is now served from the public `bulkpro-releases` repo (the source `bulkpro` repo is private and cannot host Pages). In-app "Check for updates" now resolves correctly.

---

## [0.1.10] — 2026-06-06

### Improvements

- **Hub stability**: `_resolve_task_type` module snapshot fix + 5 tech-debt cleanups.

---

## [0.1.1] — 2026-05-26

### 🔧 Improvements & Bug Fixes

#### Core & Architecture
- **Self-Healing System**: Added automatic startup recovery and package metadata fixing (resolving imageio/soundfile/moviepy initialization errors).
- **Drag-and-Drop**: Added recursive drag-and-drop file/folder support across all application modules.

#### Audio Engine
- **Speaker Diarization**: Refactored clustering using Z-score standardization and Silhouette score optimization to prevent speaker count hallucinations. Fixed Unicode logging error under Windows CP1254 locale.
- **Silence Remover**: Synchronized default parameters with Jump-Cut settings for better consistency.

#### Image Engine
- **Text Overlay**: Refactored the text overlay and layout modules to fix scaling and text sizes.
- **Canvas Layout**: Improved canvas settings and positioning features.

---

## [0.1.0] — 2026-05-20

### 🎉 Initial Release

#### Hub & Core
- Dynamic App Discovery: JSON-based scanning of `apps/` directory
- Isolated Runtime Architecture: per-module venv management
- Universal Launcher: centralized UI with app-specific interpreters
- System Tray Integration: quick access from Windows taskbar
- Global Dashboard: tool grid with search and category navigation
- Sidebar Navigation: icon-based quick access to all modules
- Frameless custom window: drag, maximize/restore, minimize

#### Auto-Update System
- GitHub Pages–based version feed (`update/version.json`)
- Background update checker with 24-hour interval
- In-app update banner with download and changelog links
- Per-version dismiss memory

#### PDF Tools
- Merge, Split, Security, Numbering, Watermark, Rotate
- Metadata Editor, Compress, Convert (PDF↔Word/Excel/JPG)
- Repair, Page Organizer, Redact, OCR

#### Excel & Data Tools
- Merge, Split, Data Cleaning, Duplicate Remover
- Smart Matcher (VLOOKUP alternative), Formatter, Regex Extractor, Transpose

#### Image Engine
- Format Converter, Resize, AI Background Removal, AI Upscale
- Smart Trim, EXIF Manager

#### Audio Lab
- Noise Cleaner, Normalizer (EBU R128), Auto-Silence Trimmer
- AI Transcription (Whisper), Channel Converter, Audio Master

#### Video Lab
- Smart Compression, Social Media Adapter, Auto-Subtitle (Whisper v3)
- Frame Extraction, Speed Controller, AI Dubbing Studio
- Jump-Cut Remover, WebM/VP9 Support, Intro/Outro, Watermarking

#### System Tools
- Disk & Log Cleaner, Focus Mode, Privacy Shield
- Power Manager, Barcode/QR Station, Desktop Hide, Unit Converter, Color Picker

#### Text Tools
- Code Formatter (JSON/SQL/XML), Hash Generator, Password Generator, Text Analyzer

#### Advanced Power Suite
- Media Downloader (yt-dlp), PII Intelligence, Metadata Audit

#### Security & Infrastructure
- Ed25519 hardware-locked license system
- Internationalization (TR/EN/DE/FR)
- Per-feature usage tracking and probabilistic degradation
- Windows Explorer right-click context menu integration

---

*For full documentation see [DOCUMENTATION.md](../docs/DOCUMENTATION.md)*
