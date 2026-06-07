# BulkPro Changelog

All notable changes to BulkPro Enterprise Tool Suite are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

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
