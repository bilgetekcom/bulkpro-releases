# BulkPro Changelog

All notable changes to BulkPro Enterprise Tool Suite are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [0.1.44] — 2026-07-30

### Fixed

- **Ilk kurulumda bootstrap zinciri sessizce bozuluyordu — uc birbirine bagli hata.**
  1. Medya Indirici: `external/` binary'leri kurulumla gelmiyor, ilk-acilis
     ffmpeg fetch'i `run.bat`in hizli yolu tarafindan atlaniyordu. Video+ses
     birlestirme gerektiren her indirme "ffmpeg is not installed" ile
     dusuyordu. `run.bat` artik ffprobe yoksa hizli yolu atlar; indirme
     motoru ffmpeg'i son bir kez daha getirmeyi dener, gercekten yoksa
     agsiz ve anlasilir bir hatayla hizlica durur.
  2. `run.bat` Unix (LF) satir sonlariyla kayitliydi. `cmd.exe`'nin
     GOTO/etiket konumlandirmasi CRLF varsayar; LF-only dosyada ilk
     kurulumu tetikleyen HER `goto` (yani her sifir makine) komutlarin
     rastgele kelimelere bolunup calistirilmaya calisilmasina yol
     aciyordu. CRLF'e cevrildi.
  3. `bootstrap.py`, eski `.venv` klasorlerini temizlerken
     `taskkill /F /IM python.exe` calistiriyordu — SISTEMDEKI HER
     python.exe surecini (kendisi dahil, kullanicinin alakasiz diger
     Python isleri dahil) anlik olarak sonlandiriyordu. Kaldirildi;
     mevcut retry+read-only-temizleme mekanizmasina birakildi.
  Regresyon testi: `tests/test_media_ffmpeg_resolution.py`. Zincir, sifir
  bir makinede gercek `run.bat` girisinden uctan uca dogrulandi (pip
  kurulum -> ffmpeg fetch -> model senkron -> gercek video indirme).

---

## [0.1.43] — 2026-07-26

### Fixed

- **Video görevleri kurulu uygulamada "Permission denied" ile ölüyordu.**
  MoviePy geçici ses dosyasının adını yalnızca çıktının dosya adından türetiyor
  (`VideoClip.write_videofile`), dolayısıyla dosya process'in **çalışma
  dizinine** yazılıyordu. Kurulu uygulama `C:\Program Files\BulkPRO` cwd'siyle
  koştuğu için sesi olan her video görevi (kırpma, dönüştürme, sıkıştırma,
  yeniden boyutlandırma, hız, filigran, ses master, intro/outro) tek kare bile
  encode etmeden düşüyordu. Geçici dosya artık çıktı klasörüne sabitleniyor.
  Yeni ortak yardımcı: `apps/video_engine/src/core/tasks/video_writer.py`.
- **Kırpma aracının "böl" modu 3. parçadan sonra patlıyordu.** Alt-klip
  `close()` edilince ana klibin paylaşılan ffmpeg okuyucusu da kapanıyor,
  sonraki parçalar `'NoneType' object has no attribute 'stdout'` veriyordu.
- **WEBM ve WMV çıktısı hiç çalışmıyordu.** `libopus` ve `wmav2` MoviePy'ın
  codec→container tablosunda yok, bu yüzden yazma daha başlamadan
  `ValueError` fırlatıyordu; libopus ayrıca MoviePy'ın varsayılanı olan
  44.1 kHz'i reddediyor. Her ikisi de eşlendi (WEBM sesi 48 kHz'e sabitlendi).

Regresyon testleri: `tests/test_video_writer.py`.

---

## [0.1.30] — 2026-06-10 (hotfix)

### Fixed

- **Models klasörü artık LOCALAPPDATA altında**, eskiden Program Files altında
  oluşturulmaya çalışılıyordu ve UAC izni olmayan kullanıcı oturumlarında
  `PermissionError [WinError 5]` ile bootstrap kırılıyordu. Ayrıca aktif
  BulkPro instance'i ile pip install arasındaki dosya kilitleme yarışını
  azaltmak için bootstrap önce mevcut process'leri kontrol ediyor.
  Etkilenen path: `MODELS_DIR` → `%LOCALAPPDATA%\BulkPro\models`.

---

## [0.1.29] — 2026-06-10

Bu sürüm kapsamlı bir denetim turuyla ortaya çıkan **64 bulgunun
tamamını** kapatıyor: lisans bypass zincirleri, güvenlik açıkları, prod
bug'lar, dead code, test kalitesi.

### Security

- **Lisans bypass zinciri kapatıldı.** `shared/usage_tracker.py` üç ayrı
  zayıflığa sahipti: (1) thread'ler arası race condition sayaç kaybına
  yol açıyordu; (2) `PROMO_MODE` modül-seviyeli bir bool'du ve tek satır
  attribute patch ile tüm ücretli modüller ücretsiz açılıyordu; (3)
  `_save_stats_data` `OSError`'i sessizce yutuyordu — disk doluysa
  kullanıcı sınırsız işlem yapabiliyordu. Şimdi: `threading.RLock`,
  promo kararı imzalı claim payload'ından okuyor, disk fail flag'i
  UI'a yansıyor.
- **IPC port 58342 artık kimlik doğruluyor.** Eskiden aynı kullanıcı
  oturumundaki herhangi bir proses (browser eklentisi, malware) JSON
  komut post'layarak BulkPro'ya kullanıcı dosyaları üzerinde işlem
  yaptırabiliyordu. Hub boot'ta `secrets.token_urlsafe(32)` ile token
  üretiliyor, `%LOCALAPPDATA%\BulkPro\.ipc_token`'a yazılıyor; her
  mesajda HMAC compare_digest ile doğrulanıyor. Path'ler `realpath` +
  user-home zorunluluğu ile normalize ediliyor.
- **`BULKPRO_LOCK_TOKEN` artık random.** Eskiden source'ta sabit string
  vardı (`"BULKPRO_SECURE_2026"`); saldırgan sub-app main.py'larını
  doğrudan çalıştırıp hub bypass yapabilirdi. Şimdi hub boot'ta üretilip
  env üzerinden sub-process'lere miras geçiyor.
- **Server CVE'leri:** `jinja2>=3.1.6` (CVE-2025-27516 sandbox escape),
  `python-multipart>=0.0.18` (CVE-2024-53981 DoS), `cryptography>=44.0.0`
  (CVE-2024-26130), `requests==2.32.3` (CVE-2024-35195 verify bypass).
- **Server admin web panel'inde CSRF koruması** (double-submit cookie
  pattern) ve **heartbeat rate-limit** (60sn / 10 hit per instance).
- **Updater integrity strict:** checksum yoksa installer reddediliyor;
  eskiden silent WARNING + install ediyordu.
- **Server signer key path strict:** `LICENSE_PRIV_KEY_PATH` env yoksa
  FATAL; default Desktop fallback kaldırıldı.
- **License key üretici script'ler installer'dan çıkarıldı**
  (`scripts/_gen_license_*.py`, `debug_*.py`, `remove_itc_*.py`).
- **Server/client token format hizalandı:** istemci artık 2-part
  *ve* 3-part token'ı kabul ediyor; self-hosted lisans sunucusu
  artık çalışıyor.

### Fixed

- **Cross-app import collision 4 app'te daha vardı.** 0.1.28'de Excel'de
  fix'lediğimiz patolojinin aynısı PDF / Image / Audio / Video
  worker'larında da yaşıyordu: `Excel → Image → Excel` kullanım
  döngüsünde sibling app'in `core.utils`'i çekiliyor, worker thread
  sessizce hata fırlatıp "yükleniyor..." durumunda asılı kalıyordu.
  `_evict_cross_app_pollution` + `submodule_search_locations` pattern'i
  5/5 app'e port edildi.
- **Audio Transcriber + Video Jump Cut iptal/çıkış sıkışması.**
  Whisper'ın `model.transcribe()` ve ffmpeg/moviepy çağrıları Python
  kontrolüne dönmüyor; cooperative `_is_cancelled` flag'i hiçbir zaman
  görmedikleri için Cancel butonu "cancelling..." durumunda asılı
  kalıyordu, Exit de çalışmıyordu. Şimdi: `cancel()` 1 sn nezaket + 
  `QThread.terminate()`; hub `closeEvent` running worker'ı zorla 
  durduruyor; tray "Exit" artık minimize'a düşmek yerine pencereyi 
  gerçekten kapatıyor.
- **PDF Tools resource leak ve robust page range.** `extract_text`
  task'ında exception path'te dosya leak'i (`open()` without `with`),
  6 yerde `map(int, part.split("-"))` "1-2-3" girince patlıyordu, 4
  bare `except: pass` `KeyboardInterrupt` yutuyordu. Tüm parser'lar
  artık robust (geçersiz parça skip + KI yutmuyor) ve dosya kapanışı
  garanti.
- **`shared/ui/components/file_card.py`** parantez bug'ı
  (`QLabel(str(index, self))`) — 0.1.28'in sonunda fix'lendiydi,
  bu sürümle release'e dahil edildi.

### Changed

- **FFmpeg LICENSE bilgisi About dialog'unda.** GPL build kullanıldığı
  için credit notu eklendi; `external/ffmpeg/LICENSE` dosyası "Open"
  butonu ile gösteriliyor.
- **PDF Merge / Split / Compress + Excel To-PDF UI'leri**
  `FileQueueList`'ten `FileBatchPanel`'e migrate edildi; drag-drop ve
  queue davranışı diğer 60+ tool ile artık tutarlı.
- **3 raw `QComboBox` → shared `ComboBox`** (video jump_cut, data_engine
  archive, intel_suite media downloader).
- **Worker signal sözleşmesi standardize edildi:** `SecurityWorker`,
  `_BulkWorker`, `_BreachWorker` artık `Signal(bool, str)` kullanıyor;
  rich payload `worker.result`'a taşındı.
- **`model_manager` worker iptal flag'i `threading.Event`'e geçti.**
  Backward-compat property korundu.
- **i18n:** file_search / data_engine / sys_tools / text_tools içindeki
  ~28 hardcoded string `tr()` ile sarıldı (EN + TR çevirileri eklendi).

### Removed (dead code)

- **36 ölü dosya silindi** (~138 KB):
  - `apps/data_engine/src/engine/` zinciri (engine.py, discovery_engine,
    archiver, deduplicator, windows_search, regex_library, _purchase_helper)
  - `apps/data_engine/src/ui/dedupe_page.py` (sanitizer'a migrate edilmişti)
  - `apps/file_search/src/core/` paralel paket (paralel `engine/` aktif)
  - `audio_engine` / `video_engine` zombi tools (audio_tools, audio_lab,
    video_tools, video_lab + `_style.py`'lar) — DeprecationWarning raise
    ediyorlardı, kullanılmıyorlardı
  - `text_tools/src/engine/text_engine.py`
  - `shared/help_manager.py`
  - 8 eski one-shot script (debug_fitz, remove_itc_branding,
    cleanup_workspace, fetch_tesseract_direct, fetch_deps, setup_app,
    build_app, check_deps)
  - `_bulkpro_runtime.log` (repo'ya commit edilmiş runtime log)
- **5 ölü asset**: boş `data/` dizini + 3 referans verilmeyen locale
  JSON (`apps.json`, `media_engine.json`, `tools.json`).
- **30+ anlamlı ölü import** (Qt/QLabel/QColor/FormGrid/t vb.).

### Tests

- **133 yeni koruma testi:** shared widget smoke (34), tab integrity (64
  — her app'in her sekmesinin doğru tool class'ını döndürdüğünü garanti
  eden regression koruyucu), 18 yeni functional test (9 audio + 9 video
  araç için), 9 edge case (unicode dosya adı, 0-sayfa, paralel worker,
  read-only output, yanlış parola, 100-dosya batch, vb.).
- **Semantic validator** (`tests/semantic_validator.py`): test çıktılarını
  pypdf/openpyxl/PIL/ffprobe ile açıp **gerçekten doğru işlem yapılmış
  mı** diye doğruluyor (merge page count sum, jump_cut süre kısalması,
  stereo→mono boyut yarıya inmesi vb.). 181/0/0 ok.
- **Tool refactor borcu:** 5 app'in main.py'ındaki `_TAB_ENTRIES`
  sırasının test beklentileriyle senkron kaldığını `test_tab_integrity`
  garanti ediyor; tab bir yere kayarsa anında fail veriyor (0.1.28'deki
  "FileCard bug'ın testlerden niye kaçtığı" hikayesinin tekrarlanmaması
  için).
- **Mock zayıflıkları:** `QMessageBox.critical` artık `pytest.fail`
  çağırıyor (kritik pop-up sessizce yutulmuyor), `pytesseract` mock'u
  input-aware, `ocrmypdf` mock'u OCR metadata stamp ekliyor.
- **STRUCTURAL skip → fail (37 dönüşüm):** modül silinirse "modül yok →
  pytest.skip" yerine "modül yok → assertion fail" — silent green
  riski kapatıldı.
- **Heavy task timeout (`_HEAVY_TASKS`):** ocr/transcribe/diarize/jump_cut
  vb. için varsayılan 40s yerine 300s; UI-test framework `start_grace`
  false-negative'lerini yakalama opsiyonu (`require_busy=True`).

### Test results

596 test pass / 4 skip (Whisper / pyannote env-dep) / **0 fail**.
Semantic validator: 181 dosya / 0 warn / 0 fail.

---

## [0.1.28] — 2026-06-09

### Changed

- **Tools reordered for muscle-memory consistency.** Converter is now the
  first tile in every module that has one (PDF, Excel, Image, Video,
  Audio). Within each module the rest is sorted by guessed-frequency —
  high-traffic operations like merge / split / compress / cutter / resize
  / extract above niche or workflow-specific ones, with AI-gated tools
  (OCR, jump-cut, transcription, diarization, noise reduction, bg
  remover, upscaler) pushed to the bottom so users don't trip over an
  install prompt while reaching for an everyday utility.
- Productivity Suite reorders too: Smart Search → Archive → Hash →
  Password → Calculation Station → Code Formatter → Text Analyzer →
  Media Downloader → Barcode → Color Picker → Metadata Audit → OCR.
- System Center: Disk Cleaner → Power Manager → Always On Top → Focus
  Mode → Smart Reminder.

---

## [0.1.27] — 2026-06-08

### Fixed

- **First-launch splash no longer mangles Turkish characters.** PowerShell
  5.1 falls back to the system ANSI code page (Windows-1254 on a Turkish
  locale) when a `.ps1` file lacks a UTF-8 byte-order mark — `Geçen süre`
  was being read as `Geçen süre`, `bileşenler` as `bileşenler`, etc.
  `scripts/install_splash.ps1` is now saved with a UTF-8 BOM and the
  hint / message strings use proper Turkish diacritics throughout.

---

## [0.1.26] — 2026-06-08

### Changed

- **Installing an optional AI component now offers to restart BulkPRO**
  so the freshly-installed wheels are actually picked up. Before this,
  `pip` would finish, the dialog would close, and the user was left
  staring at a tool that should work but didn't — because the running
  interpreter's import state is fixed at startup and can't see new
  packages until the process restarts. The user now gets a "Restart
  now?" prompt (Yes = `os.execl` clean restart, No = silent, take
  effect on next manual launch).

---

## [0.1.25] — 2026-06-08

### Fixed

- **Excel Studio tabs no longer break after navigating to another module
  and coming back** (`Could not load this tool: No module named
  'tools.cleaner'`). The Hub's `load_app_module` evicts `sys.modules['tools']`
  every time you switch apps and rebinds it to whichever sibling you
  visited; the lazy importer in `excel_tools` then resolved `tools.cleaner`
  against e.g. video_engine's tools folder and failed. The importer now
  evicts any cross-app `tools.*` / `core.*` cache before adding our `src/`
  back to `sys.path`.
- **Text Analyzer no longer writes a crash log when content changes.** The
  worker reference held a dangling Python wrapper of a Qt `NLPWorker` whose
  C++ object had already been deleted (`deleteLater` chained from the
  previous run's `finished` signal). Calling `isRunning()` on the stale
  ref raised `RuntimeError: libshiboken: Internal C++ object (NLPWorker)
  already deleted` → caught by the global excepthook → user saw the
  crash-log dialog. Same fix applied as v0.1.19's UpdateManager: guard
  the access with `try/except RuntimeError` and drop the dead ref.
- **Password Generator's leak-check now works out of the box.** The
  `allow_network_breach` setting defaulted to `False`, so the button
  silently fell back to "needs internet or local DB" even with a working
  connection. Flipped the default to `True` (HIBP uses k-anonymity — only
  the first 5 SHA-1 hex chars of the password leave the device) and
  auto-migrates existing configs that had the legacy `False` with no
  local DB path. Users who specifically want the API disabled can untick
  the checkbox in Settings.

---

## [0.1.24] — 2026-06-08

### Fixed

- **The "pythonw" mini-window flashes are GONE for real this time.** The
  v0.1.22 attempt parented the Hub's `ToolCard` / `ToolSubCard` /
  `ModulePanel` children, but the residual flashes turned out to be
  coming from the shared UI library that every tool uses. The v0.1.23
  window tracer pointed straight at `shared/ui/components/header.py`,
  `section.py`, `drop_zone.py`, `file_card.py`, `form_row.py`,
  `output_bar.py`, `page_range.py`, `slider.py`, `color_picker.py` and
  `file_batch.py` — every QLabel / QWidget / QFrame in those classes was
  being created without a parent. With ~75 tools each instantiating
  several of these widgets on first activation, every first open
  multiplied into a burst of brief top-level "pythonw" windows that DWM
  briefly composited. Each child now gets `self` (the shared component)
  passed in its constructor — full all-modules trace went from 70+
  top-level widget show events down to 0.

---

## [0.1.23] — 2026-06-08

### Diagnostics

- **Qt-level window tracer.** When `BULKPRO_TRACE_WINDOWS` is set to a
  writable path, every top-level QWidget Show / Hide / Activate event is
  logged with timestamp, class name, window title, flags, and size. The
  v0.1.22 lazy-stub parenting fix didn't fully eliminate the "pythonw"
  mini-window flashes; this tracer logs the exact Qt widget class that's
  becoming top-level so we can fix it surgically rather than mechanically
  refactor every tool class.

---

## [0.1.22] — 2026-06-08

### Fixed

- **The "pythonw" mini-windows that flashed during first module / tool
  open are now gone.** The bug was NOT a child process — it was Qt
  widgets. Every lazy-load placeholder (`stub = QWidget()`,
  `QLabel("")`, etc.) was being created without a parent. A parent-less
  Qt widget is technically a top-level window, so DWM briefly added it
  to the window list with the default title (the executable name,
  `pythonw`) before our `addTab` / `addWidget` call reparented it. The
  flash was that brief DWM-managed appearance.
- Fixed in pdf_tools, excel_tools, image_pro, audio_engine,
  video_engine, intelligence_suite and sys_tools — every lazy
  placeholder now passes its parent (`self.tabs` / `self.stack`) in the
  constructor so it's never momentarily top-level. This is why a user's
  trace.log was empty: the wrapper was correctly catching every
  `subprocess.Popen` / `_winapi.CreateProcess` call (only the moviepy
  startup `ver` chain). The flash was Qt widgets, not subprocesses, and
  the trace was telling us so.

---

## [0.1.21] — 2026-06-08

### Fixed

- **Productivity Suite's OCR tile (and any other AI-gated page) no longer
  shows a misleading "Akıllı Arama yüklenirken hata" / "Smart Search
  loading error".** When `intelligence_suite/_ensure_slot_built`
  instantiated a page decorated with `@require_component`, the
  `ComponentMissingError` raised by the decorator was caught by a generic
  `except Exception` that surfaced a label with the smart-search error
  key — confusing because the user clicked an OCR tile. The slot now
  catches `ComponentMissingError` separately and shows the proper
  "🧩 Optional AI Component Required" placeholder with an Install Now
  button (same UX the audio / image / video studios already had).
- Truly unexpected exceptions now bubble up to a generic
  `tab_build_fail` label that includes the actual exception text,
  instead of being hidden behind a smart-search-specific message.

---

## [0.1.20] — 2026-06-08

### Diagnostics

- **Process-spawn tracer for chasing residual console flashes.** When
  `BULKPRO_TRACE_SPAWNS` is set to a writable path, every `subprocess.Popen`
  and `_winapi.CreateProcess` call is logged with timestamp, pre-OR
  `creationflags`, command line, and a six-frame caller stack. Off when the
  env var is unset — zero overhead in normal runs. Used to isolate which
  spawn (if any) is reaching the screen even with Layers 1+2+3 active.

---

## [0.1.19] — 2026-06-08

### Fixed

- **"Check for updates" no longer surfaces `libshiboken: Internal C++ object
  (UpdateChecker) already deleted`.** When the previous check's `QThread`
  was destroyed by Qt, `UpdateManager` kept a dangling Python reference to
  it; the next call's `self._checker.isRunning()` then raised through
  shiboken and the Settings UI displayed it as the failure reason. The
  manager now drops both the C++ object and the Python ref when a checker
  / downloader finishes, and guards every stale-ref access with a
  `try/except RuntimeError` fallback.
- **Excel Studio tabs open again.** v0.1.15's lazy-import refactor moved the
  tool imports out of `main.py`'s top level, but the Hub's `load_app_module`
  removes the app's `src/` from `sys.path` as soon as the import returns —
  so the deferred `__import__('tools.X')` later threw
  `ModuleNotFoundError: No module named 'tools'` and the tab's stub stayed
  empty. The lazy loader now re-adds `src/` for the duration of each tool
  import.

---

## [0.1.18] — 2026-06-08

### Fixed

- **pythonw worker windows no longer flash on first module / tool open.**
  Tools that spin up `multiprocessing.Process` or `Pool` workers call
  `_winapi.CreateProcess` directly, bypassing the v0.1.15 / v0.1.17
  `subprocess.Popen` wrappers. The patch now also wraps the syscall itself
  to OR-in `CREATE_NO_WINDOW`, catching every Windows process spawn route
  used by stdlib code.
- **Update check now shows the actual error** when it fails. v0.1.17 just
  said "Check failed" with no hint of why (signature mismatch? network?
  SSL trust?) — the tab now appends the underlying message and stores the
  full text in the tooltip so it can be copied for support.

---

## [0.1.17] — 2026-06-08

### Fixed

- **Console window flashes on first module open are now actually gone.**
  v0.1.15's subprocess wrapper added `CREATE_NO_WINDOW`, which is enough for
  most callers but still leaked a brief cmd.exe flash on some Windows builds
  when transitive imports (`moviepy` → `imageio_ffmpeg` → stdlib
  `platform.win32_ver()` → `subprocess.check_output("ver", shell=True)`)
  fired during Video Studio's first load. The wrapper now also attaches a
  `STARTUPINFO` with `SW_HIDE` as a second layer. Callers that explicitly
  want their child to show a window (custom `STARTUPINFO` with a non-zero
  `wShowWindow`) are left alone.

---

## [0.1.16] — 2026-06-08

### Installer

- **Removed phantom AI components** (`ai_video`, `ai_gpu`) from the installer
  component picker. Both were dropped back in v0.1.14 but the Inno Setup
  script still listed them, so users saw — and could check — install options
  for tools that no longer ship.
- **Desktop shortcut is now pre-selected** by default during install.
- Video Jump-Cut now correctly declares its `ai_transcription` requirement
  (it was tagged `ai_video`, an alias that no longer existed, so the install
  prompt for the Whisper component never fired even though the tool needs it).

### Fixed

- **First-launch splash bar now animates.** The Marquee-style ProgressBar
  needed `Application.EnableVisualStyles()` to be called before any control
  was created; without it the bar rendered in classic Windows style and
  stayed frozen, making the install look hung. Added an elapsed-time
  counter below the bar as a second "alive" signal in case the OS theme
  ever refuses the marquee animation.

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
