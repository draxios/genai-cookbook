# ROLE & OBJECTIVE

You are a senior software engineer and solutions architect. Build a **local desktop application** named **“ForgeVideo Manager”** using **.NET 8 C# WinForms**. It must be a **workhorse** video organization and management system with **powerful FFmpeg encoding**, batch processing, metadata, tagging, file operations, and **deep Plex integration**. Deliver production-ready code and docs in a single response.

# HIGH-LEVEL REQUIREMENTS

1. **Platform**

   * .NET 8, C# 12, WinForms.
   * Windows 10+; x64.
   * No admin required to run.
   * Offline-first. All features must work without internet, except Plex calls (optional if user configures).

2. **Core Features**

   * **Library Management:** scan/import folders; index videos; generate thumbnails; compute file hashes; deduplicate detection; rename/move/copy/delete with safety checks; watch folders (FileSystemWatcher) for auto-import.
   * **Search & Filter:** full-text search across filename, path, tags, codec/container/bitrate/resolution/audio/subtitles; sort by any column; saved searches.
   * **Tagging & Metadata:** custom tags (key\:value and free-form); read technical metadata via **ffprobe**; store normalized metadata fields; allow editing of descriptive fields (title, year, collection, notes).
   * **FFmpeg Powerhouse:**

     * Built-in **FFmpeg & FFprobe** path detection; allow portable binaries in app folder or user-specified path.
     * **Preset system & script editor:** JSON-based templates for video/audio/subtitle/container; free-form CLI script editor with variable expansion; lint/validate; save/name/share presets; versioning of presets.
     * **Batch encoding queue** with priorities, concurrency limits, pause/resume/cancel, retry policy, progress parsing, ETA, per-job logs.
     * Hardware acceleration options (detect & expose **NVENC**, **QSV**, **AMF**, **VAAPI** where applicable; on Windows primarily NVENC/QSV).
     * Popular ready-made presets: H.264 (x264 & NVENC), H.265/HEVC (x265 & NVENC), AV1 (SVT-AV1 & NVENC AV1), copy/remux, audio transcode (AAC/AC3/EAC3/Opus), subtitle burn-in vs copy, container MKV/MP4/MOV.
     * Per-track mapping UI (map 0\:v, 0\:a:0, etc.), quality modes (CRF / CQ / target bitrate), 2-pass option, filters (scale, deinterlace, denoise, crop, rotate).
     * **Dry-run** preview of generated command line and estimated output.
   * **Plex Integration (optional):**

     * Configure PMS URL and **X-Plex-Token**.
     * Browse Plex libraries and sections; map local folders to Plex sections.
     * Trigger library scans/refreshes after file moves/transcodes; update media metadata where supported.
     * Option to export post-encode files to Plex-managed paths with naming templates (TV, Movies).
   * **Profiles & Automation:**

     * User profiles for presets, watch folders, and output rules (e.g., “When 4K source + HDR10, use HEVC NVENC preset with tone-map → SDR”).
     * Scheduled scans; auto-enqueue rules (e.g., new files in Watch\_4K → preset “HEVC-NVENC Balanced”).
   * **Safety & Audit:**

     * Confirmations for destructive actions; recycle bin option; conflict handling; per-job detailed logs.
     * Central activity log + rolling job logs; export diagnostics zip.

3. **Architecture & QUALITY**

   * **Clean architecture** layering:

     * `ForgeVideo.Core` (domain, entities, value objects, services interfaces)
     * `ForgeVideo.Infrastructure` (SQLite data access, FFmpeg/Plex adapters, file ops)
     * `ForgeVideo.WinForms` (UI)
     * `ForgeVideo.Tests` (unit tests; minimal UI tests where feasible)
   * Use **Microsoft.Data.Sqlite** for a local DB file (`forgevideo.db`) and Dapper or EF Core (pick one and implement).
   * Background services via **System.Threading.Channels** and hosted like workers (custom lightweight host).
   * Logging via **Microsoft.Extensions.Logging**; file sink (daily rolling).
   * Strong typing, cancellation tokens, async/await where appropriate.
   * Robust error handling; no silent failures.
   * Zero TODOs in final code; compile-ready.

# DATA MODEL (SUGGESTED)

* `MediaItem` (Id, Path, FileName, Directory, SizeBytes, Hash, Duration, VideoCodec, AudioCodec, Width, Height, FrameRate, Bitrate, Container, AudioChannels, AudioLanguage(s), SubtitleTracks, HDR/Color info, ScanDateUtc, LastSeenUtc, TagsJson, Notes)
* `Tag` (Id, Name, Type, Description) with M2M `MediaItemTag`
* `Preset` (Id, Name, Category, Description, Version, JsonDefinition, CreatedUtc, UpdatedUtc, IsDefault)
* `PresetVersion` (Id, PresetId, Version, JsonDefinition, Changelog, CreatedUtc)
* `EncodeJob` (Id, MediaItemId, PresetId nullable, CustomArgs, Status \[Queued|Running|Paused|Succeeded|Failed|Canceled], Priority, ConcurrencyGroup, OutputPath, StdoutLogPath, StderrLogPath, ProgressPercent, Eta, StartUtc, EndUtc, ReturnCode, ErrorMessage)
* `WatchFolder` (Id, Path, IncludePatterns, ExcludePatterns, AutoEnqueuePresetId nullable, OutputRuleId nullable, Enabled)
* `OutputRule` (Id, Name, PatternTemplate, DestinationRoot, Container, NamingTemplate e.g. `{Title} ({Year})/{Title} - S{Season:00}E{Episode:00}`, OverwritePolicy)
* `Setting` (Key, Value)
* `PlexConfig` (ServerUrl, Token, LibraryMappingsJson)

# PRESET JSON SCHEMA (EXAMPLE)

```json
{
  "name": "HEVC NVENC Balanced 1080p",
  "container": "mkv",
  "video": {
    "encoder": "hevc_nvenc",
    "mode": "cq",
    "cq": 20,
    "preset": "p5",
    "profile": "main",
    "pix_fmt": "p010le",
    "scaler": "scale_cuda=1920:1080:format=p010",
    "filters": ["tonemap=reinhard:desat=0.2"]
  },
  "audio": [
    {"map": "0:a:0", "action": "transcode", "encoder": "aac", "bitrate": "192k", "channels": 2}
  ],
  "subtitles": {"mode": "copy_preferred", "burn_in": false},
  "mapping": {"video": "0:v:0", "audio": ["0:a:0"], "subs": "all"},
  "extraArgs": ["-movflags", "+faststart"],
  "twoPass": false,
  "hwaccel": "auto"
}
```

* The app must convert this JSON to a validated FFmpeg CLI string, with a **preview pane** and **lint diagnostics**.

# FFPROBE & FFMPEG INTEGRATION

* Detect ffmpeg/ffprobe path at first run:

  * Order: app folder `.\bin\ffmpeg.exe` → PATH → user supplied.
  * Validate versions; show capabilities (encoders/decoders/hwaccels).
* **FFprobe wrapper**: run `ffprobe -v error -print_format json -show_format -show_streams` and map to `MediaItem`.
* **FFmpeg progress**: spawn process with `-progress pipe:1 -nostats`; parse `out_time_ms`, `fps`, `speed`, `bitrate`; compute % based on duration; update job row; support cancel via `Process.Kill(entireProcessTree: true)`.

# UI REQUIREMENTS (WINFORMS)

* **Main Window** with:

  * Left: Library tree (folders, saved searches, tags, watch folders).
  * Center: Grid of MediaItems with columns and multi-select; context menu for file ops and “Add to Queue”.
  * Right: Details panel (metadata; quick tags; thumbnails; previews).
  * Bottom: **Encoding Queue** panel (jobs list, progress, start/pause/cancel, concurrency).
  * Top: Search bar (with advanced filter builder and saved searches dropdown).
* **Preset Manager** dialog:

  * List of presets + versions; import/export JSON; set defaults; duplicate; validate.
  * **Script Editor tab** with raw CLI editor (multi-line); token highlighting (e.g., `{Input}`, `{Output}`, `{Map:0:a:0}`); variable cheat-sheet; “Build Command” button shows resolved command.
* **Plex Settings** dialog:

  * Server URL, X-Plex-Token, test connection, library mapping (local path → Plex Section).
  * “Refresh Plex Library” button per section.
* **Watch Folders** dialog: add/remove; include/exclude patterns; auto-enqueue rules.
* **Options**: paths, concurrency, temp folder, post-encode actions (move, replace source, keep both), safety options.

# FILE OPERATIONS & NAMING

* Safe move/copy/delete with collision handling: options (skip, rename increment, overwrite if newer).
* After successful encode, apply **OutputRule** → place in destination tree; if Plex mapping exists, place accordingly and trigger section refresh.
* Hashing for dedupe (SHA-256); duplicate inspector view.

# PLEX INTEGRATION DETAILS

* Provide a `IPlexClient` with methods:

  * `GetLibrariesAsync()`, `GetSectionsAsync()`, `RefreshSectionAsync(sectionId)`, `GetServerInfoAsync()`.
* Store token securely in app settings (encrypted with DPAPI CurrentUser).
* Optional: Post-encoding webhook to Plex to refresh a specific path or section.

# EXTENSIBILITY

* **Plugin interface** (MEF or simple reflection) for:

  * Custom post-processing steps.
  * New preset providers.
  * Custom metadata providers.
* **Scripting hooks** (optional): run external command before/after encode with templated variables.

# PERFORMANCE & CONCURRENCY

* Channel-based job queue; worker pool with max N parallel encodes (user setting).
* Disk I/O throttling option to keep UI responsive.
* Avoid UI thread blocking; marshal updates via `SynchronizationContext`.

# TESTING

* Unit tests for: preset validation, ffprobe parsing, command line builder, progress parser, job state machine.
* Integration tests (where feasible) behind flags; mockable `IProcessRunner`.

# SECURITY & PRIVACY

* No telemetry.
* PII-free logs.
* Token encryption (Plex).
* Safe defaults (no overwrite unless explicit).

# INSTALL & PACKAGING

* Build: Release | Any CPU x64.
* Single-folder deployment using `PublishSingleFile=false` (FFmpeg is separate).
* Provide **MSIX or Inno Setup** script; include requirement to place `ffmpeg.exe` & `ffprobe.exe` in `.\bin` or configure path.

# CONFIGURATION

* `appsettings.json` (or user settings) for paths, concurrency, rules, Plex config.
* Export/Import settings to JSON.

# DEVELOPER EXPERIENCE

* Thorough **README.md** with screenshots (placeholder images), getting started, prerequisites, FFmpeg notes, Plex notes.
* **MIGRATIONS** for SQLite schema (if EF Core) or SQL bootstrap script (if Dapper).
* Sample data seeder for demo mode.

# OUTPUT EXPECTATIONS (VERY IMPORTANT)

When you respond, produce **all of the following**, clearly separated with headings and file paths:

1. **Architecture Overview**: diagrams (ASCII ok) and explanation.
2. **Solution layout** (tree) and **all** project files:

   * `ForgeVideo.sln`
   * `/ForgeVideo.Core/*.cs`
   * `/ForgeVideo.Infrastructure/*.cs`
   * `/ForgeVideo.WinForms/*.cs` (Forms designers included), `.resx`, and `Program.cs`
   * `/ForgeVideo.Tests/*.cs`
   * `.csproj` files with proper references and NuGet packages (only essential: Microsoft.Data.Sqlite + chosen ORM/logging).
3. **DB schema** SQL (and EF migrations if using EF).
4. **FFprobe models & mapper** + **FFmpeg command builder** with preset JSON parser/validator.
5. **Process runner** with progress parsing, cancellation, retries, and logs.
6. **UI code** for all forms/dialogs and controls, wired to services.
7. **Preset samples**: at least 8 JSON presets (x264/x265/AV1; software & NVENC; remux; audio-only).
8. **Watch folder** example config.
9. **Plex client** implementation (HTTP calls with token header/query), plus a stubbed mock for tests.
10. **Logging** setup writing to `Logs\*.log`, job logs per run.
11. **Installer** script (MSIX or Inno Setup) and build instructions.
12. **README.md** with step-by-step setup, screenshots placeholders, and troubleshooting.
13. **LICENSE** (MIT).
14. **Acceptance Test Checklist** mapping each feature to a testable step.
15. **Future Work** section (HDR tone-mapping presets, AV1 tuning, per-profile auto-crop, queue auto-balancing).

# CODING CONSTRAINTS

* No placeholders like “TODO”; deliver compilable, runnable code.
* Avoid heavy third-party UI deps; stick to stock WinForms controls (optional: System.Drawing imaging).
* Use dependency injection via `Microsoft.Extensions.DependencyInjection` to wire services in `Program.cs`.
* All long-running work off the UI thread; WinForms `Invoke` for UI updates.
* Be explicit in error messages; show actionable hints (e.g., if ffmpeg not found).

# COMMAND LINE GENERATION RULES

* Replace tokens:

  * `{Input}`, `{Output}`, `{OutputDir}`, `{Container}`, `{Map:...}`, `{CRF}`, `{CQ}`, `{Bitrate}`, `{Width}`, `{Height}`, `{Filters}`, `{ExtraArgs}`
* Always include `-y` only if overwrite is chosen; otherwise omit.
* For MP4, add `-movflags +faststart` when appropriate.
* If 2-pass is selected, produce two commands with proper `-passlogfile` handling.
* If HW encoder selected, reflect correct flags (e.g., `-hwaccel cuda` / `-init_hw_device` when needed).
* Validate combinations (e.g., NVENC + p010le + hevc\_nvenc profile).

# UX POLISH

* Keyboard shortcuts for common actions.
* Drag-and-drop files to enqueue.
* Status bar with current workers, CPU/GPU usage (basic WMI), disk free space.
* Theming (light/dark).

# ACCEPTANCE CRITERIA (EXCERPT)

* Can import a folder, index files, and search “codec\:hevc AND width>=1920”.
* Can create a preset via JSON, validate, preview CLI, and **successfully encode** a sample file with live progress.
* Can enqueue 10 files with NVENC preset and see concurrent jobs (limit user-defined).
* After encode, files move to Plex library path and Plex section refresh is triggered if configured.
* Logs are written; app remains responsive; cancel works mid-encode and cleans up temp files.

# DELIVERY FORMAT

* Provide the **entire codebase** inline with headings and file paths.
* Ensure the solution **builds** under .NET 8 SDK with standard `dotnet build`.
* Include any helpful scripts (e.g., `tools/ffmpeg-detect.md`, `scripts/post-install.ps1`).

---

If anything is ambiguous, make reasonable assumptions and **document them** in the README. Do not omit any of the files requested above. Deliver production-ready code and documentation.
