# AkshunsPerMinute — Product Specification

**Version:** 1.0  
**Date:** 2026-03-10  
**Status:** Ready for development

---

## Table of Contents

1. [Overview](#1-overview)
2. [Tech Stack](#2-tech-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Tracking Engine](#4-core-tracking-engine)
5. [System Tray](#5-system-tray)
6. [OBS Overlay Window](#6-obs-overlay-window)
7. [Overlay Widgets](#7-overlay-widgets)
8. [Settings Window](#8-settings-window)
9. [Tab: Overlay](#9-tab-overlay)
10. [Tab: Appearance & Theming](#10-tab-appearance--theming)
11. [Tab: Tracking](#11-tab-tracking)
12. [Tab: Sessions](#12-tab-sessions)
13. [Tab: General](#13-tab-general)
14. [Session Lifecycle](#14-session-lifecycle)
15. [Data Storage](#15-data-storage)
16. [Error Handling & Recovery](#16-error-handling--recovery)
17. [First-Run Experience](#17-first-run-experience)
18. [Build & Distribution](#18-build--distribution)
19. [Project File Structure](#19-project-file-structure)
20. [Coding Standards](#20-coding-standards)

---

## 1. Overview

### 1.1 Purpose

**AkshunsPerMinute** is a Windows desktop utility that tracks and displays the user's real-time APM (akshuns per minute) — a count of all keyboard keypresses, mouse button clicks, and scroll wheel ticks in a rolling time window.

### 1.2 Target User

**Primary user:** Streamers who broadcast on platforms such as Twitch or YouTube and want to display their live APM as an on-screen stat in their OBS scene.

### 1.3 Primary Use Case

The streamer runs AkshunsPerMinute alongside their game or application. The OBS overlay window (titled `AkshunsPerMinuteOverlay`) is captured in OBS as a **Window Capture** source. A chroma key filter is applied in OBS to remove the background colour, leaving only the styled widgets floating over the stream scene. The streamer customises the look via the settings window without disrupting their live scene.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.10+ |
| UI framework | [Dear PyGui](https://dearpygui.com/) — GPU-accelerated immediate-mode GUI |
| Input capture | [`pynput`](https://pypi.org/project/pynput/) — global keyboard & mouse listener |
| System tray | [`pystray`](https://pypi.org/project/pystray/) + [`Pillow`](https://pypi.org/project/Pillow/) for dynamic icon rendering |
| Session storage | SQLite 3 (via `sqlite3` stdlib) |
| Settings / themes | JSON (via `json` stdlib) |
| Logging | `logging` stdlib, rotating file handler |
| Build | [PyInstaller](https://pyinstaller.org/) `--onefile --windowed` |

---

## 3. Architecture Overview

### 3.1 Process Model

The application runs as a **single process** with multiple threads:

| Thread | Responsibility |
|---|---|
| **Main thread** | Dear PyGui event loop (settings window + overlay window) |
| **Tracker thread** | `pynput` listeners for keyboard and mouse input |
| **Tray thread** | `pystray` icon loop |
| **Persistence thread** | Periodic flushing of APM samples to SQLite |

All inter-thread communication uses thread-safe queues (`queue.Queue`) or atomic Python primitives. The tracker thread enqueues timestamped events; the main thread dequeues them on each render frame.

### 3.2 Single-Instance Enforcement

On launch the app acquires a named Windows mutex (`AkshunsPerMinute_SingleInstance`). If the mutex is already held, the new process sends a signal to the existing instance to bring its settings window to the foreground, then exits immediately.

### 3.3 Key Module Boundaries

```
main.py              Entry point, single-instance check, wires modules together
tracker.py           Input listener, rolling window, APM calculation, session data
overlay.py           Dear PyGui overlay window (OBS capture target)
settings_window.py   Dear PyGui settings window (5-tab UI)
tray.py              pystray tray icon, dynamic rendering, context menu
theme_engine.py      Theme loading, saving, import/export, built-in theme registry
session_store.py     SQLite read/write for session history and APM samples
settings_store.py    JSON read/write for user settings and preferences
hotkey_manager.py    Global hotkey registration, conflict detection
logger.py            Centralised rotating log file setup
```

---

## 4. Core Tracking Engine

### 4.1 What Counts as an Akshun

Each of the following is configurable (can be toggled independently by the user):

| Input | Default | Notes |
|---|---|---|
| Any keyboard keypress | ✅ enabled | Key-repeat events (holding a key) count only the **initial press**, not subsequent repeats |
| Mouse button click (left / right / middle) | ✅ enabled | Down event only |
| Scroll wheel tick | ✅ enabled | Each discrete tick = 1 akshun |
| Mouse movement | ✗ never | Mouse position changes are never counted |

### 4.2 Key Exclusion List

The user may specify individual keys to exclude from counting (e.g., standalone Shift, Ctrl, Alt, Win). The exclusion list is configured in the Tracking tab and stored in `settings.json`.

### 4.3 APM Calculation — Rolling Window

- APM is computed over a **user-configurable rolling window** (default: 60 seconds).  
- The displayed value equals the number of akshuns whose timestamp falls within the past `window_duration` seconds, updated on every overlay render frame.  
- **Before the window has fully elapsed** (i.e., the app or session has been running for less than `window_duration` seconds): APM is displayed as a **scaled partial-window value** — `(akshuns_so_far / elapsed_seconds) * 60`. This gives a meaningful live reading from the first keypress rather than displaying zero or a placeholder.

### 4.4 APM Display Format

- **Default:** Integer (`247`)  
- **Optional:** Configurable decimal precision via a format string in settings (e.g., `"{:.1f}"` → `247.3`)

### 4.5 Overlay Refresh Rate

The overlay redraws on every Dear PyGui frame (targeting 60 fps). The displayed APM value is recalculated on each frame using the current rolling window. This ensures the number updates immediately after any new input event rather than on a coarse timer.

---

## 5. System Tray

### 5.1 Tray Icon

- The tray icon is a **dynamically rendered image** generated with Pillow on every update.
- The current APM value is drawn as text directly onto the icon image (16×16 or 32×32 px, high-DPI-aware).
- The icon refreshes whenever the displayed APM value changes (driven by the main render loop).

### 5.2 Tooltip

- Hovering the tray icon displays a tooltip.
- **Default content:** Current APM + session peak APM.
- **Configurable:** The user selects which stats appear in the tooltip via the General tab.

### 5.3 Left-Click Action

- **Default:** Open the settings window.
- **Configurable:** The user may reassign left-click to any of the following in the General tab:
  - Open settings window
  - Toggle overlay visibility
  - Start / stop session
  - Reset session

### 5.4 Right-Click Context Menu

Fixed menu structure (not configurable):

```
Settings
─────────────
Toggle Overlay
─────────────
Start Session
Stop Session
Reset Session
─────────────
Quit
```

---

## 6. OBS Overlay Window

### 6.1 Window Properties

| Property | Value |
|---|---|
| Window title | `AkshunsPerMinuteOverlay` (hard-coded; used by streamers in OBS Window Capture) |
| Taskbar visibility | Hidden — does **not** appear in the Windows taskbar |
| Always on top | No requirement (OBS captures it as a window source regardless of z-order) |
| DPI awareness | Fully DPI-aware (`SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)`) |
| Background colour | User-configurable solid colour (default: chroma-key green `#00FF00`) |
| Transparency | Window background colour is solid (not OS-level transparency), enabling OBS chroma key |

### 6.2 OBS Integration Workflow

1. User launches AkshunsPerMinute — the overlay window appears.
2. In OBS, user adds a **Window Capture** source and selects `AkshunsPerMinuteOverlay`.
3. User applies a **Chroma Key** filter in OBS using the configured background colour.
4. The styled widgets appear floating over the scene.

### 6.3 Canvas & Auto-Resize

- The overlay window **auto-resizes** to tightly fit the bounding box of all visible widgets.
- A **"Lock canvas size"** toggle in the Overlay tab freezes the window dimensions, preventing OBS scene layout shifts when widgets are toggled or resized. When locked, a user-defined fixed width × height is used and widgets may be clipped if they exceed the bounds.
- The window position is **persisted** between sessions (stored in `settings.json`).
- If the last-known position is on a monitor that is no longer connected, the window snaps to the top-left of the primary display.

### 6.4 Overlay vs. Settings Preview

The overlay window (OBS capture target) updates **only when the user clicks Save/Apply** in the settings window. A live preview panel within the settings window reflects changes in real time without touching the overlay.

---

## 7. Overlay Widgets

Each widget is an independent, fully configurable component. Widgets are positioned on a 2D drag-and-drop canvas and reproduced exactly in the OBS overlay window.

### 7.1 Widget: Live APM

Displays the current rolling-window APM value.

| Property | Detail |
|---|---|
| Value | Rolling window APM (see §4.3) |
| Format | Configurable format string (default: integer) |
| Animation | **Configurable:** smooth lerp to new value over N ms, or instant snap |

### 7.2 Widget: Label

A static text label (e.g., `"APM"`, `"Akshuns/min"`). Fully styleable. Can be positioned independently of the APM number.

### 7.3 Widget: Peak APM

Displays one or both of:
- **Session peak** — highest APM recorded since the current session started
- **All-time peak** — highest APM ever recorded across all sessions (persisted in SQLite)

Display format example: `"Peak: 312"` / `"All-time: 489"`. The user configures which value(s) to show and the label prefix string.

### 7.4 Widget: Average APM

Displays the average APM over one or more user-selected time windows. Available windows:

| Window | Description |
|---|---|
| Last 1 minute | Rolling average over the past 60 s |
| Last 5 minutes | Rolling average over the past 5 min |
| Last 15 minutes | Rolling average over the past 15 min |
| Full session | Total akshuns / total session elapsed time |

The user may enable any combination of these; each enabled window is rendered as a separate line in the widget (or as separate widgets — configurable layout).

### 7.5 Widget: Sparkline

A mini time-series graph of APM history. Two user-configurable modes:

| Mode | Default | User-Configurable Range |
|---|---|---|
| Short | Last 60 s, sampled every 1 s | Sample rate: 1–10 s; history: 10 s – 5 min |
| Long | Last 10 min, sampled every 10 s | Sample rate: 5–60 s; history: 1 min – 60 min |

A toggle button on the widget (or a configurable hotkey) switches between modes at runtime.

Configurable visual properties: line colour, fill/gradient colour, background colour, line thickness, axis labels on/off.

### 7.6 Widget Common Properties

Every widget shares the following configurable style properties:

| Property | Notes |
|---|---|
| Position (x, y) | Set via drag-and-drop canvas; fine-tuned via numeric input fields for pixel-perfect alignment |
| Size (w, h) | Explicit dimensions or auto-fit to content |
| Font family | Any system font or bundled font |
| Font size | Points, configurable per widget |
| Font weight / style | Bold, italic, normal |
| Text colour | RGBA |
| Background colour | RGBA (can be transparent to inherit overlay background) |
| Border | Thickness, colour, radius |
| Drop shadow | Offset (x, y), blur, colour |
| Padding | Per-side (top, right, bottom, left) |
| Opacity | 0–100% |
| Animation | Smooth value transition duration (ms); disable for instant snap |
| Visibility | Toggled on/off independently |

---

## 8. Settings Window

### 8.1 Access

Opened via:
- System tray right-click → **Settings**
- System tray left-click (default action, configurable)
- Global hotkey (configurable)

### 8.2 Window Behaviour

- **Single instance:** Only one settings window can exist at a time. Triggering "open settings" when already open brings the window to the foreground.
- **Resizable:** Yes.
- **Live preview panel:** Embedded within the Overlay and Appearance tabs; shows a real-time preview of the overlay layout and theme as the user makes changes.
- **Save / Apply button:** Pushes current settings to disk and updates the OBS overlay window.
- **Cancel / Discard button:** Reverts unsaved changes in the UI.

### 8.3 Tab Structure

| Tab | Purpose |
|---|---|
| **Overlay** | Widget enable/disable, drag-and-drop canvas, per-widget config |
| **Appearance** | Theme picker, theme editor, import/export |
| **Tracking** | Input type toggles, key exclusions, rolling window, idle detection |
| **Sessions** | Session history browser, lifetime stats, export, delete |
| **General** | Global hotkeys, tray icon behaviour, display format, single-instance |

---

## 9. Tab: Overlay

### 9.1 Widget Panel

- A list of all available widgets (Live APM, Label, Peak APM, Average APM, Sparkline) with a toggle to enable/disable each.
- Selecting a widget in the list opens its per-widget property editor in a side panel.

### 9.2 Drag-and-Drop Canvas

- A scaled preview canvas reproducing the overlay dimensions.
- Widgets appear on the canvas and can be dragged to any position.
- **Pixel-precise override:** Each widget's position and size are also exposed as numeric spin-boxes (x, y, w, h in pixels) for exact placement.
- The canvas background colour reflects the current chroma key colour.

### 9.3 Canvas Size Controls

- **Auto-resize (default):** Overlay window resizes to the bounding box of all visible widgets with configurable padding.
- **Lock canvas size:** User sets a fixed W × H in pixels. The canvas in OBS remains stable regardless of widget changes.

### 9.4 OBS Tips Panel

A collapsible info panel within the Overlay tab displaying step-by-step instructions for adding the overlay to OBS (Window Capture → Chroma Key filter), including the exact window title to search for.

---

## 10. Tab: Appearance & Theming

### 10.1 Theme Picker

- Displays all available themes: built-in themes + user-saved themes.
- Selecting a theme applies it to the live preview instantly.
- Built-in themes cannot be deleted or overwritten; they can be duplicated as a starting point for a custom theme.

### 10.2 Built-In Themes

Dozens of pre-built themes covering popular streamer aesthetics. Indicative categories:

| Category | Examples |
|---|---|
| Dark / minimal | Dark Minimal, Charcoal, Obsidian, Slate |
| Neon / gaming | Neon Green, Cyber Pink, Electric Blue, Acid Yellow |
| Retro / pixel | Retro CRT, Pixel 8-bit, Arcade |
| Clean / light | Clean White, Paper, Soft Grey |
| Gradient / glow | Sunset Gradient, Aurora, Holographic |
| Esports branded | Team Red, Team Blue, Gold Champion |
| Transparent / ghost | Ghost (no background, white text), Shadow Overlay |

Each built-in theme defines values for all widget common properties (§7.6) across all widget types.

### 10.3 Theme Editor

Full property editor for the active theme. Properties are grouped by widget type (global defaults + per-widget overrides). All properties from §7.6 are editable. Changes are reflected in the live preview panel in real time.

### 10.4 User-Saved Themes

- Users can save the current editor state as a named theme.
- Named themes appear in the theme picker alongside built-in themes.
- Users can rename or delete their saved themes.

### 10.5 Theme File Format

Themes are stored as `.theme` files (JSON under the hood, `.theme` extension for discoverability).

```json
{
  "name": "My Custom Theme",
  "version": 1,
  "author": "optional string",
  "widgets": {
    "live_apm": { "font_family": "...", "font_size": 72, "text_color": "#FFFFFF", ... },
    "label":    { ... },
    "peak_apm": { ... },
    "avg_apm":  { ... },
    "sparkline": { "line_color": "#00FF88", "fill_color": "#00FF8833", ... }
  }
}
```

### 10.6 Theme Import / Export

- **Export:** Saves the selected theme to a `.theme` file at a user-chosen path.
- **Import:** Loads a `.theme` file; validates the schema before applying. Invalid files show a descriptive error. Imported themes are saved into the user themes directory.

---

## 11. Tab: Tracking

All settings in this tab take effect immediately (no Save/Apply required for tracking behaviour, though settings are persisted to `settings.json` on change).

| Setting | Type | Default | Notes |
|---|---|---|---|
| Count keyboard keypresses | Toggle | On | Excludes key-repeat events; only initial press |
| Count mouse button clicks | Toggle | On | Left, right, middle |
| Count scroll wheel ticks | Toggle | On | Each discrete tick |
| Key exclusion list | Multi-select key picker | Empty | e.g., Shift, Ctrl, Alt, Win alone |
| Rolling window duration | Numeric (seconds) | 60 | Min: 5 s, Max: 300 s |
| Idle detection threshold | Numeric (seconds) | 30 | No-input duration before idle state triggers |
| Idle detection action | Dropdown | Disabled | Options: Disabled / Pause session timer |

> **Note:** Idle detection does **not** auto-split or auto-end sessions (§14). When set to "Pause session timer", the elapsed session time excludes idle periods, keeping average APM accurate.

---

## 12. Tab: Sessions

### 12.1 Lifetime Stats Banner

Displayed at the top of the Sessions tab at all times:

- All-time peak APM (and the date it was set)
- Total number of sessions
- Total tracked time (summed across all sessions)
- Overall average APM (total akshuns / total tracked time)

### 12.2 Session List

A scrollable table of all recorded sessions with columns:

| Column | Notes |
|---|---|
| Name | User-assigned label, or auto-generated timestamp |
| Date & start time | ISO 8601 local time |
| Duration | HH:MM:SS |
| Session peak APM | |
| Session average APM | |

Clicking a row opens the Session Detail view.

### 12.3 Session Detail View

- Full APM-over-time line graph for the selected session (x-axis: elapsed time, y-axis: APM).
- Summary stats: peak, average, total akshuns, duration.
- Rename button to edit the session label.
- Export button (exports this session to CSV).
- Delete button (with confirmation dialog).

### 12.4 Bulk Actions

- **Export All:** Exports all sessions to a single CSV file.
- **Delete All / Clear History:** Confirmation dialog before wiping the SQLite database.

### 12.5 CSV Export Format

```csv
session_id,session_name,timestamp,elapsed_seconds,apm
abc123,Monday Stream,2026-03-10T14:00:00,0,0
abc123,Monday Stream,2026-03-10T14:00:01,1,60
...
```

---

## 13. Tab: General

### 13.1 Global Hotkeys

All 9 actions are bindable to any key combination. The user clicks a "Record" button then presses the desired key combo.

| Action | Default Hotkey |
|---|---|
| Toggle overlay visibility | *(none)* |
| Start session | *(none)* |
| Stop session | *(none)* |
| Reset session | *(none)* |
| Open settings window | *(none)* |
| Cycle theme | *(none)* |
| Toggle tray display mode | *(none)* |
| Increase overlay opacity | *(none)* |
| Decrease overlay opacity | *(none)* |

No default hotkeys are assigned out of the box to avoid conflicts with games and other apps.

**Conflict detection:** At bind time, the app checks whether the key combination is already registered by another app (via `RegisterHotKey` Win32 API trial registration). If a conflict is detected, an inline warning message is shown immediately, displaying the exact conflict details and inviting the user to choose a different combo or clear the existing binding.

### 13.2 Tray Icon Behaviour

- **Left-click action:** Dropdown selector (Open Settings / Toggle Overlay / Start|Stop Session / Reset Session). Default: Open Settings.
- **Tooltip content:** Multi-select checkboxes for which stats appear (current APM, session peak, all-time peak, session duration).

### 13.3 APM Display Format

- **Format string:** Text input accepting a Python-style format string applied to the APM float value before display (e.g., `"{:.0f}"` for integer, `"{:.1f}"` for one decimal place).
- Applies to: overlay Live APM widget, tray icon, tray tooltip.

### 13.4 Number Animation

- **Animation mode:** Dropdown — Smooth (lerp) / Instant snap.
- **Smooth duration:** Numeric input (ms). Default: 150 ms.

---

## 14. Session Lifecycle

### 14.1 Session Start

A new session begins automatically when the app launches. The user may also manually start a new session at any time via:
- The "Start Session" button in the tray context menu
- The global hotkey for "Start session"

If a session is already running when a manual start is triggered, a dialog prompts:
> **"A session is already in progress. Save it and start a new one, or continue with the current session?"**
> `[ Save & Start New ]` `[ Continue Current ]` `[ Cancel ]`

### 14.2 Session End

A session ends when:
- The app quits (see §14.4)
- The user clicks "Stop Session" (tray menu, hotkey, or Sessions tab)

### 14.3 Session Naming

- **Auto-name:** `"Session – YYYY-MM-DD HH:MM"` using the local start time.
- **User rename:** Editable at any time via the Sessions tab detail view or via a right-click on the session row.

### 14.4 Quit Confirmation

If a session is active when the user selects Quit, a confirmation dialog is shown:
> **"A session is in progress. It will be saved before quitting."**
> `[ Save & Quit ]` `[ Cancel ]`

### 14.5 Reset Session

Clears all stats for the current session (akshuns, APM history, session peak, session elapsed time) and begins a fresh session with the same name + a " (reset)" suffix. A confirmation dialog is shown first:
> **"Reset the current session? All stats for this session will be cleared."**
> `[ Reset ]` `[ Cancel ]`

---

## 15. Data Storage

### 15.1 Directory Layout

```
%APPDATA%\AkshunsPerMinute\
├── settings.json          # All user preferences (see §15.2)
├── history.db             # SQLite session history (see §15.3)
├── themes\                # User-created and imported .theme files
│   └── *.theme
├── backups\               # Automatic pre-write backups
│   ├── settings.YYYYMMDD_HHMMSS.json
│   └── history.YYYYMMDD_HHMMSS.db
└── logs\
    └── akshunsperminute.log   # Rotating log (see §16.4)
```

### 15.2 settings.json Structure (top-level keys)

```json
{
  "version": 1,
  "overlay": {
    "canvas_locked": false,
    "canvas_width": 300,
    "canvas_height": 200,
    "chroma_key_color": "#00FF00",
    "position": { "x": 100, "y": 100 },
    "widgets": { ... }
  },
  "appearance": {
    "active_theme": "Dark Minimal",
    "user_themes_dir": "%APPDATA%\\AkshunsPerMinute\\themes"
  },
  "tracking": {
    "count_keyboard": true,
    "count_mouse_clicks": true,
    "count_scroll": true,
    "excluded_keys": [],
    "rolling_window_seconds": 60,
    "idle_threshold_seconds": 30,
    "idle_action": "disabled"
  },
  "general": {
    "hotkeys": { ... },
    "tray_left_click_action": "open_settings",
    "tray_tooltip_stats": ["current_apm", "session_peak"],
    "apm_format_string": "{:.0f}",
    "animation_mode": "smooth",
    "animation_duration_ms": 150
  }
}
```

### 15.3 SQLite Schema

```sql
CREATE TABLE sessions (
    id          TEXT PRIMARY KEY,   -- UUID
    name        TEXT NOT NULL,
    started_at  TEXT NOT NULL,      -- ISO 8601
    ended_at    TEXT,               -- NULL if in progress
    peak_apm    REAL,
    total_akshuns INTEGER,
    notes       TEXT
);

CREATE TABLE apm_samples (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    sampled_at  TEXT NOT NULL,      -- ISO 8601
    elapsed_s   INTEGER NOT NULL,   -- seconds since session start
    apm         REAL NOT NULL
);

CREATE INDEX idx_apm_samples_session ON apm_samples(session_id, elapsed_s);
```

APM samples are written once per second during an active session (flushed in batches by the persistence thread).

### 15.4 Automatic Backups

Before any write to `settings.json` or `history.db`, a timestamped copy is written to the `backups\` directory. Backups older than 30 days are pruned on startup to limit disk usage.

---

## 16. Error Handling & Recovery

### 16.1 Crash Recovery

On every launch, the app checks whether a session in `history.db` has a non-null `started_at` but a null `ended_at` (indicating a crash mid-session). If found:
- The session is marked as ended at the last recorded `apm_samples` timestamp.
- A toast notification informs the user: *"Previous session recovered."*
- The recovered session appears in the Sessions browser with its partial data intact.

### 16.2 Corrupt Data

| File | Handling |
|---|---|
| `settings.json` corrupted | Warn the user with a dialog. Offer: load defaults / restore latest backup. Do not silently discard. |
| `history.db` corrupted | Warn the user with a dialog. Offer: restore latest backup / start fresh (data loss acknowledged). |

### 16.3 Global Hotkey Conflicts

At bind time:
1. The app attempts `RegisterHotKey` for the requested combo.
2. If it fails (combo already taken), an inline error message is displayed immediately below the hotkey field, stating: *"This key combination is in use by another application. Please choose a different combo."*
3. The previous valid binding (or no binding) is preserved.

### 16.4 Multi-Monitor Snap-Back

On startup, the app reads the saved overlay position from `settings.json` and checks whether it falls within the bounds of any currently connected monitor (via `EnumDisplayMonitors`). If not, the overlay is repositioned to the top-left corner of the primary display.

### 16.5 Application Logging

- Log file: `%APPDATA%\AkshunsPerMinute\logs\akshunsperminute.log`
- Rotating file handler: 5 MB per file, 3 backups retained.
- Log levels: DEBUG (file), WARNING+ (stderr in dev builds).
- All unhandled exceptions are caught at the top level, logged with full traceback, and a user-friendly error dialog is shown before the app exits cleanly.

---

## 17. First-Run Experience

On first launch (detected by absence of `settings.json`):

1. **Welcome dialog** — a small, professional modal introduces the app in 2–3 sentences: what it does, that it lives in the system tray, and where to find Settings.
2. **Default settings** applied: Dark Minimal theme, all input types enabled, 60-second rolling window, no hotkeys assigned.
3. **Overlay window** appears immediately in the top-right of the primary display, showing live APM in the default theme.
4. The welcome dialog includes a one-line OBS setup tip with a link to the OBS Tips panel in the Overlay tab.
5. No lengthy wizard or multi-step onboarding — the app is usable within seconds of first launch.

---

## 18. Build & Distribution

### 18.1 Executable

```bash
pyinstaller \
  --onefile \
  --windowed \
  --name AkshunsPerMinute \
  --icon assets/icon.ico \
  main.py
```

Output: `dist/AkshunsPerMinute.exe` — single portable binary, no installer required.

### 18.2 Dependencies (`requirements.txt`)

```
dearpygui>=1.11
pynput>=1.7
pystray>=0.19
Pillow>=10.0
pyinstaller>=6.0
```

### 18.3 No Installer

Distribution is portable-only. Users place `AkshunsPerMinute.exe` anywhere and double-click. Data is stored in `%APPDATA%\AkshunsPerMinute\` regardless of where the `.exe` lives.

---

## 19. Project File Structure

```
AkshunsPerMinute/
├── main.py                  # Entry point; single-instance check; wires all modules
├── tracker.py               # Input listener, rolling window, APM engine
├── overlay.py               # Dear PyGui OBS overlay window
├── settings_window.py       # Dear PyGui settings window (5 tabs)
├── tray.py                  # pystray tray icon, dynamic icon rendering, context menu
├── theme_engine.py          # Theme loading, built-in registry, import/export
├── session_store.py         # SQLite CRUD for sessions and apm_samples
├── settings_store.py        # JSON read/write for settings.json
├── hotkey_manager.py        # Global hotkey registration and conflict detection
├── logger.py                # Rotating log file setup
├── models.py                # Shared dataclasses / typed models (Session, ApmSample, Theme, Settings)
├── assets/
│   ├── icon.ico             # App icon (used by tray and PyInstaller)
│   └── themes/              # Built-in .theme files (read-only, bundled)
│       ├── dark_minimal.theme
│       ├── neon_green.theme
│       └── ...              # (dozens of built-in themes)
├── requirements.txt
├── README.md
├── references/
│   └── spec.md              # This document
└── .cursor/
    ├── code-style.mdc
    ├── workflow.mdc
    └── testing.mdc
```

---

## 20. Coding Standards

The project follows the conventions defined in `.cursor/code-style.mdc`:

- **Python 3.10+** with full type annotations on all function signatures.
- **PEP 8** formatting; 4-space indentation; 100-character line limit.
- **`black`** + **`isort`** as canonical formatters.
- **Pydantic** or `dataclass` for all internal typed models — no raw `dict` passing between module boundaries.
- Typed exceptions for all service-layer errors (e.g., `StorageError`, `HotkeyConflictError`).
- Docstrings on all public functions and classes.
- Comments only for non-obvious logic, design decisions, contracts, and edge cases — never narrating obvious code.
- `# ASSUMPTION:` markers at any decision point where an assumption was made.

---

*End of specification.*
