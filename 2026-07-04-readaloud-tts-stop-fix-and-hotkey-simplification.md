# ReadAloud TTS Windows — Mid-Chunk Stop Fix + Hotkey Simplification

## Date
2026-07-04

## Summary
Fixed the last major playback bug in ReadAloudTTS: mid-sentence stop was impossible because `winsound.PlaySound(None, 0)` from the daemon's main thread could not interrupt a blocking `PlaySound` call on the worker thread. Also simplified the hotkey model to a deliberate two-key interaction: `Home` to read, `F6` to stop.

## Issues Fixed

### 1. Mid-chunk stop impossible (the real stop bug)
- **Problem**: The daemon's `handle_stop()` runs on the main loop thread and calls `winsound.PlaySound(None, 0)` to cancel playback. But the worker thread is inside a blocking `winsound.PlaySound(wav_bytes, SND_MEMORY)` call — Windows pins the memory buffer and plays it to exhaustion, ignoring cross-thread cancel attempts. The stop request was received and acknowledged (`{"status": "ok", "message": "Stop requested"}`) but audio kept playing to the end of the chunk.
- **Investigation**: Three playback modes were tested:
  1. `SND_MEMORY` (blocking) — Windows pins the buffer, plays to exhaustion, cross-thread `PlaySound(None, 0)` is ignored.
  2. `SND_FILENAME` (blocking) — cross-thread `PlaySound(None, 0)` deadlocks or is silently ignored on many Windows builds.
  3. `SND_FILENAME + SND_ASYNC` — playback starts in the background, the worker thread polls `_stop_requested` every 30ms, and when stop is detected the **same thread** calls `PlaySound(None, 0)` to cancel. This is the only reliable pattern for interruptible winsound playback on Windows.
- **Solution**: Switched to `SND_FILENAME | SND_ASYNC`. Chunk WAV bytes are written to temp files under `tmp/playback-*/`, played asynchronously, and the worker thread polls `_stop_requested` every 30ms. Chunk duration is computed from sample count + sample rate so the poll loop exits on natural completion (no infinite waits). When stop is requested, the same thread that owns the playback session calls `PlaySound(None, 0)` — which works reliably. Temp files are cleaned up in a `finally` block.
- **Missing imports also fixed**: `shutil` (for `rmtree` in the cleanup block) and `TMP_DIR` (from `speak.py`, used as the temp-file parent directory).

### 2. Hotkey model simplified
- **Problem**: The repo shipped with `Ctrl+Right-click` (read) and `Ctrl+Alt+Space` (stop) only — underselling the tool's convenience. An earlier commit added `Home`, `F6`, `F12` (read) and `F8` (stop), but that was too many keys and the model was hard to remember.
- **Solution**: Reduced to a deliberate two-key model:
  - `Home` = read the current text selection (the main player)
  - `F6` = stop speech immediately, mid-sentence
  - `Ctrl+Right-click` = alternative read gesture (kept for mouse-driven workflows)
- Removed `F8`, `F12`, `Ctrl+Alt+Space`, and `Ctrl+AppsKey` bindings. The interaction is now: **one key to read, one key to stop.**
- README updated: new hotkeys table, one-second pitch ("Press `Home`, it speaks. Press `F6` to stop."), "Why these hotkeys" section explaining the minimal design, and a note for Logitech MX Keys / compact keyboard users about `Fn+Esc` to toggle the function-key lock so F6 sends a real F-key without holding Fn.

## Files Modified
- `src/speak_server.py` — async file-based playback with same-thread stop polling; added `shutil` and `TMP_DIR` imports
- `src/ReadAloudTTS.ahk` — simplified hotkey bindings to Home (read) + F6 (stop) + Ctrl+Right-click (read)
- `README.md` — updated hotkeys table, one-second pitch, "Why these hotkeys" section, Fn+Esc note, usage steps

## Verification
- Operator tested live: select text → press `Home` → it speaks; press `F6` mid-sentence → it stops immediately
- Operator confirmed: pressing `Home` during active playback cleanly interrupts and restarts with the new selection — responsive, no overlap
- Daemon log confirms: stop request received, `_stop_requested` flag set, worker thread poll loop exits, `PlaySound(None, 0)` cancels async playback on the same thread
- No `NameError` or crash in worker thread after import fixes

## Repository Status
- All changes committed and pushed to `LE-VAI/read-aloud-tts-windows` on `main`
- Commit: `55931c4` — "Fix: mid-chunk stop + simplify hotkeys to Home (read) / F6 (stop)"
- No personal data in any file or commit message (verified via pre-commit hygiene scan)

## Lessons Reinforced
- **winsound is not thread-safe the way you'd expect**: `PlaySound(None, 0)` from a second thread does not interrupt a blocking `PlaySound` on the first thread. The async + same-thread-poll pattern is the answer. This took three failed attempts to understand.
- **Pre-commit hygiene scan is non-negotiable**: every public-facing file is scanned for usernames, local paths, and machine-identifiable details before commit. Established after a previous build-log entry leaked local filesystem paths and required a history rewrite + force-push to scrub.
- **Verify before claiming**: no fix is announced as "working" until it has been tested end-to-end by the operator pressing the actual keys and confirming the expected behavior.

## Next Steps
- Visual identity refresh in progress (premium SaaS-level assets with VAI-yellow branding)
- Continue monitoring for edge cases in the stop/restart flow