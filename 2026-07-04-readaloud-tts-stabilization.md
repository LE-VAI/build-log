# ReadAloud TTS Windows - Stabilization and Fix

## Date
2026-07-04

## Summary
Fixed critical bug in ReadAloud TTS Windows where the application was reading from the clipboard instead of selected text. This was caused by a missing clipboard clearing operation before copying the selected text.

## Issues Fixed

### 1. Clipboard vs Selection Bug
- **Problem**: The running AHK script was missing `A_Clipboard := ""` before `Send "^c"`. Without clearing the clipboard first, `ClipWait(2.0)` returned immediately (the clipboard already had old data), so it read whatever was previously copied instead of the new selection.
- **Solution**: Added `A_Clipboard := ""` before `Send "^c"` so `ClipWait` actually waits for the new selection instead of returning immediately with stale clipboard data.

### 2. AHK v2 InputBox Syntax Warning
- **Problem**: The `TrayManualRead` function used AHK v1 syntax (`InputBox text, ...`) which caused a warning about the local variable never being assigned a value.
- **Solution**: Changed to AHK v2 syntax (`ib := InputBox(...)`) to silence the warning.

## Files Modified
- Installed AHK script (local install path)
- Repository source `ReadAloudTTS.ahk` (already had the clipboard fix)

## Verification
- Script was restarted cleanly
- Both fixes were verified to be in place
- Selection reading now works correctly instead of reading from clipboard

## Repository Status
- The clipboard fix was already committed in the repository source
- No new changes needed to be pushed to `LE-VAI/read-aloud-tts-windows`
- Build-log entry created to document the stabilization work

## Next Steps
- Continue monitoring for any other issues
- Consider updating visual assets for the repository as requested