# Glimpse — Product Requirements Document

**Version:** 0.1.0  
**Status:** Draft  
**Last Updated:** 2026-01-03

---

## Overview

Glimpse is a CLI screenshot tool designed for developers and AI agents. It enables scriptable, permission-controlled window capture on Wayland (Linux) with planned macOS support.

**Core principle:** Screenshots should be easy to script but impossible to abuse. Users maintain explicit control over what can be captured.

---

## Target Users

| User Type | Needs |
|-----------|-------|
| Developers | Scriptable screenshots for documentation, bug reports, automation |
| AI Agents | Programmatic screen capture for vision-based workflows |

---

## MVP Scope (v1.0)

### Platform Support

- **Linux:** wlroots-based compositors (Sway, Hyprland, etc.)
- Wayland protocols: `wlr-screencopy`, `wlr-foreign-toplevel-management`

### Core Features

#### 1. Window Capture

Capture a specific window by app ID, title, or both.

```bash
glimpse capture --app-id "firefox"
glimpse capture --title "GitHub*"
glimpse capture --app-id "alacritty" --title "nvim"
```

#### 2. Permission System

Two-tier permission model:

| Scenario | Behavior |
|----------|----------|
| Window in allowlist | Capture silently, return immediately |
| Window not in allowlist | Show approval GUI, wait for user decision |

**Approval GUI:**
- Displays captured image preview
- Two buttons: **Approve** / **Deny**
- Configurable timeout (default: 30s) — auto-deny on timeout

**Allowlist entry format:**
```toml
[[allow]]
app_id = "firefox"
title = "*"  # wildcard: allow all Firefox windows

[[allow]]
app_id = "alacritty"
title = "nvim*"  # only terminals with nvim in title
```

#### 3. Output Modes

| Flag | Behavior |
|------|----------|
| (default) | Copy PNG to clipboard via `wl-copy` |
| `--stdout` | Write PNG bytes to stdout |
| `--output <path>` | Write PNG to file |

#### 4. Window Listing

```bash
glimpse list
```

Output (JSON):
```json
[
  {"id": "1234", "app_id": "firefox", "title": "GitHub - Mozilla Firefox"},
  {"id": "1235", "app_id": "alacritty", "title": "~"}
]
```

---

## CLI Interface

```
glimpse <command> [options]

Commands:
  capture     Capture a window screenshot
  list        List available windows
  config      Manage allowlist

Options (capture):
  --app-id <id>       Filter by app ID (exact match)
  --title <pattern>   Filter by title (glob pattern)
  --output <path>     Write to file instead of clipboard
  --stdout            Write PNG to stdout
  --no-clipboard      Don't copy to clipboard (use with --output)
  --timeout <secs>    Approval timeout in seconds (default: 30)

Options (config):
  --add               Add current capture target to allowlist
  --remove            Remove entry from allowlist
  --list              Show current allowlist
```

---

## Configuration

**Location:** `~/.config/glimpse/config.toml`

```toml
[general]
timeout = 30  # seconds before auto-deny

[clipboard]
command = "wl-copy"  # clipboard command

[[allow]]
app_id = "firefox"
title = "*"

[[allow]]
app_id = "code"
title = "*"
```

**Allowlist file:** `~/.config/glimpse/allowlist.toml`

Separate file for easier programmatic management.

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | No matching window found |
| 3 | User denied capture |
| 4 | Timeout (user didn't respond) |
| 5 | Compositor connection failed |

---

## Technical Architecture

```
┌──────────────┐
│  glimpse     │
│  (CLI)       │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│  Core Library                                │
│  ┌─────────────┐  ┌─────────────────────┐   │
│  │ Permissions │  │ Backend Trait       │   │
│  │ (allowlist) │  │  - list_windows()   │   │
│  └─────────────┘  │  - capture()        │   │
│                   │  - get_app_id()     │   │
│  ┌─────────────┐  └──────────┬──────────┘   │
│  │ Approval UI │             │              │
│  │ (iced/egui) │  ┌──────────┴──────────┐   │
│  └─────────────┘  │ wlroots Backend     │   │
│                   │ (smithay-client-tk) │   │
│                   └─────────────────────┘   │
└──────────────────────────────────────────────┘
```

**Key crates:**
- `wayland-client` — Wayland protocol bindings
- `smithay-client-toolkit` — Higher-level wlroots helpers
- `iced` or `egui` — Approval GUI
- `image` + `png` — Image encoding
- `toml` — Config parsing
- `clap` — CLI parsing

---

## Non-Goals (v1.0)

- Multi-monitor support
- Region selection
- Screen recording
- Annotation/editing
- X11 support
- GNOME/KDE compositor support

---

## Future: macOS Support (v2.0)

### Platform APIs

| Feature | API |
|---------|-----|
| Window enumeration | `CGWindowListCopyWindowInfo` |
| Window capture | `CGWindowListCreateImage` |
| App identification | Bundle identifier |
| Clipboard | `NSPasteboard` |

### macOS-Specific Requirements

- **Screen Recording Permission:** macOS requires explicit permission grant on first use (system prompt). Glimpse should detect missing permission and guide user to System Preferences.
- **Multi-monitor:** Required for macOS (common setup). Capture should work across displays.
- **Retina displays:** Handle 2x scaling correctly in PNG output.

### Abstraction

```rust
trait Backend {
    fn list_windows(&self) -> Result<Vec<Window>, Error>;
    fn capture(&self, window_id: WindowId) -> Result<Image, Error>;
    fn window_info(&self, window_id: WindowId) -> Result<WindowInfo, Error>;
}

struct Window {
    id: WindowId,
    app_id: String,      // Wayland app_id or macOS bundle ID
    title: String,
    // platform-specific extensions via feature flags
}
```

### Timeline Consideration

Implement macOS after wlroots backend is stable and permission UX is validated.

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Capture latency (allowlisted) | < 200ms |
| Capture latency (with approval) | < 500ms + user time |
| Binary size | < 10MB |
| Memory usage | < 50MB during capture |

---

## Open Questions

1. **Glob vs regex for title matching?** Glob is simpler but less powerful.
2. **Should allowlist support time-based expiry?** e.g., "allow for 1 hour"
3. **Daemon mode for v1?** Currently scoped out, but may improve latency.
4. **GUI toolkit choice:** iced (pure Rust, cross-platform) vs egui (immediate mode, simpler)?

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-01-03 | Initial draft |
