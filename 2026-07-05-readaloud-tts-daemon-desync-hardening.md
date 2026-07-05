# ReadAloud TTS Windows - Daemon Desync Hardening

## Date
2026-07-05

## Summary
Fixed the overnight issue where `Home` stops playing speech after sleep, hibernate, reboot, or an AutoHotkey reload. The root cause was an orphaned-daemon desync between the AHK overlay and the long-lived Piper TTS daemon. Shipped a three-layer hardening (self-healing daemon, smart AHK startup with orphan recovery, and a one-click user-facing refresh helper) plus documentation to the public repo.

## Root Cause
The daemon (`speak_server.py`) writes a `tmp/daemon_ready` marker file when it finishes loading the Piper voice model. AHK checks this marker before speaking. On every AHK startup, AHK **blindly deleted** the marker — even when the daemon was still alive holding its single-instance Windows kernel mutex (`Global\ReadAloudTTS_speak_server_singleton`). AHK then spawned a competitor process, which exited immediately with `"Another speak_server daemon is already running; exiting"` (the mutex guard in `speak_server.py:429`). No marker was ever created, so every subsequent `Home` press silently did nothing. The daemon was alive and responsive (it answered `ping` requests) but orphaned from AHK.

This affected any user after sleep, reboot, or AHK reload when a daemon from a previous session was still running.

## Issues Fixed

### 1. Daemon does not self-heal its readiness marker
- **Problem**: Once anything deleted `tmp/daemon_ready` (AHK restart, cleanup script, transient FS issue), the daemon never recreated it — even though the daemon was still alive and ready. AHK permanently lost track of it.
- **Solution**: Added a self-heal check to the daemon's main poll loop in `speak_server.py`. Every 20ms, after handling any request, the daemon checks whether `daemon_ready` exists and recreates it if missing. Logged as `"daemon_ready marker was missing; restored"`. The marker now always reflects the daemon's actual liveness.

### 2. AHK blind-deletes the marker on startup
- **Problem**: `ReadAloudTTS.ahk` line 32 ran `try FileDelete DaemonReadyPath` unconditionally at every startup, orphaning any live daemon.
- **Solution**: Replaced with `PruneStaleDaemon()`, which pings the daemon first. If the daemon responds, the marker is kept (the daemon is alive). Only if the ping times out is the marker removed. This prevents the desync at the AHK side.

### 3. StartDaemon has no orphan recovery
- **Problem**: When a fresh daemon spawn failed (because an orphan held the mutex), `StartDaemon` just waited 10s for a marker that never appeared, then gave up silently.
- **Solution**: `StartDaemon` now detects the spawn failure and recovers automatically: sends a `quit` to the orphan via the request-file IPC, waits 800ms for the mutex to release, and retries the spawn once. Emits tray notifications (`"Recovering stuck TTS daemon..."` / `"TTS daemon recovered"`). Same orphan-recovery logic applied to the installed `ReadAloudTTS_WORKING.ahk` `EnsureDaemonRunning`.

### 4. StopDaemon only sends quit if marker exists
- **Problem**: `StopDaemon` sent a `quit` request only when `IsDaemonReady()` was true. If the marker was missing but the daemon was alive, the quit was skipped and the daemon was left running.
- **Solution**: `StopDaemon` now always sends a quit (via the new `SendDaemonQuit` helper). The quit IPC works across process elevation — unlike `taskkill`, which returns "Access is denied" if the daemon was started in a different/elevated context. The PID force-kill remains as a fallback.

### 5. No user-facing recovery tool
- **Problem**: When Home went silent, users had no way to recover without technical knowledge (editing files, killing processes, running commands). The tray "Restart Daemon" item existed but force-killed by PID, which fails on elevated/stale daemons.
- **Solution**: Created `refresh-readaloud.cmd` (double-clickable) and `refresh-readaloud.ps1`. The script sends a graceful quit to any stuck daemon, waits for the mutex to release, launches a fresh daemon, and verifies readiness with clear user-facing output. Works regardless of process elevation. Safe to run at any time, even when the daemon is already healthy. `install.ps1` now deploys both files to the install folder.

## Files Modified

### Source repo (`D:\1ATLAS\read-aloud-tts-windows`)
- `src/speak_server.py` — self-healing marker in main loop
- `src/ReadAloudTTS.ahk` — `PruneStaleDaemon`, `SendDaemonQuit`, orphan-recovery `StartDaemon`, robust `StopDaemon`, `RestartDaemon` toast update
- `refresh-readaloud.ps1` — **new**, one-click recovery script
- `refresh-readaloud.cmd` — **new**, double-click launcher
- `install.ps1` — deploys refresh helper to install dir
- `scripts/smoke-test.ps1` — validates new files exist + PowerShell parses
- `docs/TROUBLESHOOTING.md` — "Home does nothing" section with refresh instructions
- `CHANGELOG.md` — 0.7.0 entry with root-cause background
- `README.md` — troubleshooting section mentions refresh helper

### Install dir (`%LOCALAPPDATA%\ReadAloudTTS`)
- `speak_server.py` — synced from source, active in running daemon
- `ReadAloudTTS_WORKING.ahk` — smart `EnsureDaemonRunning` with orphan recovery, active after AHK reload
- `refresh-readaloud.ps1` + `refresh-readaloud.cmd` — deployed and tested

## Verification

### Layer 1: Self-healing marker
- Deleted `tmp/daemon_ready` manually while daemon was running.
- Marker reappeared in **20ms** (one poll cycle).
- Daemon log confirmed: `"daemon_ready marker was missing; restored"`.
- Repeated on a refresh-launched daemon — same result.

### Layer 2: Smart AHK startup (orphan handling)
- Reloaded AHK (`#SingleInstance Force`) while daemon PID 33796 was alive.
- New AHK (PID 48092) came up; daemon **stayed alive, not orphaned**.
- Daemon log showed **no** `"already running"` exit — the new `EnsureDaemonRunning` pinged the daemon, got `pong`, kept the marker, and returned immediately. This is the exact scenario that would have broken before the fix.

### Layer 3: Refresh helper
- Ran `refresh-readaloud.ps1` against a live daemon.
- Output: quit acknowledged → fresh daemon started → ready in **2s**.
- User-facing output showed `Home = read selected text` / `F6 = stop speech` / daemon PID / last log line.

### Syntax validation
- `speak_server.py` — `py_compile` PASS (source + installed).
- `refresh-readaloud.ps1` — PowerShell parser PASS, no errors.
- `smoke-test.ps1` — all structural checks PASS (required files, syntax, .gitignore, no forbidden artifacts). Only the git-clean check failed, expected during development.

## Repository Status
- **Committed and pushed** to `LE-VAI/read-aloud-tts-windows` on `main`.
- Commit: `fcc19b1` — 9 files changed, 241 insertions, 16 deletions.
- Remote confirmed in sync: `## main...origin/main`.
- Minor cosmetic note: commit subject has a leading UTF-8 BOM from PowerShell 5.1's `Set-Content -Encoding utf8`. Renders fine on GitHub; not worth a force-push.

## Next Steps
- The AHK-side orphan recovery (`PruneStaleDaemon`, `StartDaemon` rewrite) is in the canonical `src/ReadAloudTTS.ahk` but the installed `ReadAloudTTS_WORKING.ahk` is a separate file with different hotkey bindings and no highlight overlay. Consider converging the installed version onto the canonical source so future fixes propagate automatically via `install.ps1` instead of requiring manual sync to two files.
- Monitor for any new desync edge cases after sleep/hibernate over the coming days.
- Consider adding a `refresh-readaloud` desktop shortcut during install for maximum discoverability.
