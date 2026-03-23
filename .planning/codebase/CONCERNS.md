# Codebase Concerns

**Analysis Date:** 2026-03-23

## Tech Debt

### 1. Monolithic IPC Handler

**Issue:** `src/helpers/ipcHandlers.js` contains 4,140 lines of code with highly coupled responsibilities spanning transcription, streaming, reasoning, model management, and state synchronization.

**Files:** `src/helpers/ipcHandlers.js`

**Impact:**
- Difficult to test individual handlers without full initialization
- Changes to one feature risk breaking others
- Long initialization and context switching time
- No clear separation of concerns

**Fix approach:**
- Split into domain-specific handler modules: `ipcHandlersTranscription.js`, `ipcHandlersStreaming.js`, `ipcHandlersModels.js`, `ipcHandlersState.js`
- Create handler registry pattern in main IPC setup
- Extract common error handling into utility

### 2. Audio Manager Complexity

**Issue:** `src/helpers/audioManager.js` is 2,796 lines and handles recording, streaming, VAD, silence detection, platform-specific audio, and pipeline orchestration simultaneously.

**Files:** `src/helpers/audioManager.js`

**Impact:**
- Difficult to isolate audio bugs
- Multiple async flows (recording, streaming, fallbacks) increase race conditions
- Silence detection setup failures silently fall through (line 374 catches error, sets fallback value)
- VAD processor integration not well documented

**Fix approach:**
- Extract audio I/O into `AudioInputManager.js`
- Extract streaming logic into `StreamingPipeline.js`
- Extract silence/VAD detection into `AudioAnalysis.js`
- Create AudioPipeline orchestrator that composes these modules

### 3. Platform-Specific Clipboard Implementation

**Issue:** `src/helpers/clipboard.js` (1,690 lines) handles 6+ different clipboard strategies (AppleScript, PowerShell, nircmd, xdotool, wtype, ydotool) with complex fallback chains and platform detection.

**Files:** `src/helpers/clipboard.js`

**Impact:**
- Risk of silent fallback failures (e.g., ydotool daemon not running, XWayland not available)
- No standardized error reporting per strategy (some throw, some return false)
- Difficulty testing all platform combinations
- Cache management adds state that could become stale

**Fix approach:**
- Create platform-specific clipboard modules: `clipboardDarwin.js`, `clipboardWindows.js`, `clipboardLinux.js`
- Implement consistent StrategyResult type with success/failure/fallback status
- Expose strategy selection in debug logs
- Add health check command to verify available strategies at startup

### 4. Native Binary Management

**Issue:** Multiple native binaries (whisper-cpp, sherpa-onnx, globe-listener, key-listener, mic-listener, text-monitor, media-remote) with separate build/download scripts and platform-specific compilation.

**Files:** `scripts/build-*.js`, `scripts/download-*.js`, multiple compilation flags in `main.js` and helpers

**Impact:**
- No central registry of which binaries are required on which platforms
- Build failures don't block app startup (failures are "non-fatal")
- `resources/bin/` directory grows unbounded; no cleanup of old versions
- Startup delays waiting for binary initialization in parallel
- Missing binaries degrade functionality silently

**Fix approach:**
- Create `BinaryRegistry.json` with platform/architecture/binary mapping, version, and fallback behavior
- Implement BinaryManager class that validates all required binaries at startup
- Create cleanup mechanism for old binary versions
- Expose binary status in app health check

### 5. Uncoordinated Startup Sequence

**Issue:** Multiple managers initialize in parallel with implicit dependencies (hotkeyManager depends on globeKeyManager, meetingDetectionEngine depends on audioActivityDetector and googleCalendarManager). No startup validation.

**Files:** `main.js` (lines 600-1000)

**Impact:**
- Race conditions if dependent managers not ready (e.g., hotkey handler called before globeKeyManager initialized)
- Failures in non-critical services block app launch (auth bridge, dev server)
- No health check or status reporting at startup
- Difficult to debug initialization order issues

**Fix approach:**
- Create AppStartupOrchestrator with explicit dependency graph
- Implement startup phases (init → validate → connect → ready)
- Return startup report with successes/failures/degradations
- Auto-disable dependent services if dependencies fail

## Known Bugs

### 1. Streaming Token Cache Not Validated

**Problem:** AssemblyAI and Deepgram streaming tokens are cached but not validated before use (lines 3564, 3809 in ipcHandlers.js). If token expires during use, pipeline fails without graceful re-authentication.

**Files:** `src/helpers/ipcHandlers.js` (lines 3540-3570, 3795-3830)

**Trigger:** Long-running streaming sessions (>1 hour) where token TTL expires

**Workaround:** Restart application to get fresh token

### 2. Silence Detection Setup Swallowed

**Problem:** Silence detection failure in `audioManager.js` line 374 catches error and silently sets `_peakRms = 1` (assume speech). User never knows detection failed, may affect transcription accuracy.

**Files:** `src/helpers/audioManager.js` (lines 373-375)

**Trigger:** AudioContext setup failure (out of memory, audio device unavailable)

**Workaround:** Check debug logs; restart app if audio is unresponsive

### 3. Stale Streaming Connection Cleanup

**Problem:** `ipcHandlers.js` lines 3543-3547 document cleaning up "stale active connection (shouldn't happen normally)" before starting new stream, but provides no mechanism to detect what caused staleness or prevent it.

**Files:** `src/helpers/ipcHandlers.js` (lines 3543-3547, 3798-3810)

**Trigger:** User rapidly toggles streaming on/off, or network disconnect during streaming

**Workaround:** User must manually stop and restart streaming

## Security Considerations

### 1. API Key Persistence in .env File

**Risk:** OpenAI, Anthropic, Gemini, and third-party API keys stored in plaintext in `.env` file in app data directory.

**Files:** `src/helpers/environment.js`, `src/helpers/ipcHandlers.js` (lines 2950-2980 saveAllKeysToEnvFile)

**Current mitigation:** `.env` file is in user-owned directory with restricted permissions; still vulnerable to:
- Malicious apps with user-level access
- Disk recovery after deletion
- Accidental exposure if user backs up app data

**Recommendations:**
- Move keys to system keychain/credential manager (macOS Keychain, Windows Credential Manager, Linux Secret Service)
- Keep plaintext fallback only for development mode
- Add warning in UI if keys stored plaintext

### 2. IPC Surface Exposed to Renderer

**Risk:** Renderer process can call any IPC handler registered in `ipcHandlers.js` without authentication or rate limiting.

**Files:** `src/helpers/ipcHandlers.js` (entire file), `preload.js`

**Current mitigation:** Context isolation enabled; preload only exposes intended APIs

**Remaining risk:** If renderer is compromised, attacker can:
- Access database with transcription history
- Trigger transcription without user knowledge (using saved API keys)
- Modify application settings (hotkeys, models, providers)
- Potentially write files via file dialogs

**Recommendations:**
- Implement IPC rate limiting per handler
- Add authentication for sensitive operations (clear API keys, change providers)
- Log all sensitive IPC calls with timestamps/sources

### 3. Unvalidated External URLs in OAuth Flows

**Risk:** OAuth redirect URLs (`main.js` lines 349-360) accept any `code` parameter without CSRF validation. If attacker controls deep link, could inject authorization code.

**Files:** `main.js` (lines 349-360, auth bridge setup)

**Current mitigation:** OAuth code must match exact user account; replay attacks unlikely

**Recommendations:**
- Implement state parameter validation for OAuth flows
- Require nonce matching in callback

## Performance Bottlenecks

### 1. Polling-Based Audio Activity Detection on Linux

**Problem:** If `pactl subscribe` unavailable or fails, falls back to polling microphone device list every 3-15 seconds. On busy systems, `ps-list` can block event loop for 100-300ms.

**Files:** `src/helpers/audioActivityDetector.js` (lines 30-100)

**Cause:** Event-driven approach requires native binary (not available in all Wayland compositors)

**Impact:** 100-300ms blocking on every poll cycle; noticeable lag in responsiveness

**Improvement path:**
- Profile `ps-list` performance on loaded systems
- Implement worker thread for process list scanning if >100ms blocking detected
- Add cache with debounce to avoid redundant scans

### 2. Google Calendar Sync Exponential Backoff

**Problem:** On network failure, calendar sync backs off to 30-minute intervals. Imminent meeting within 5 minutes won't be detected during outage.

**Files:** `src/helpers/googleCalendarManager.js` (lines 250-300)

**Impact:** During network issues, meeting detection becomes unreliable

**Improvement path:**
- Implement smarter backoff: fail-fast on timeout (don't wait full 30min), use calendar local cache while syncing
- Add manual "refresh now" button in UI
- Log backoff state in debug logs

### 3. Parallel Binary Compilation at Startup

**Problem:** `npm run dev` compiles 8 native binaries in parallel (prestart hook) even when unchanged. Full compilation takes 30-60 seconds on first run.

**Files:** `package.json` (line 20: prestart script)

**Impact:** Development cycle slow; blocks `npm start` until all binaries ready

**Improvement path:**
- Implement binary caching: skip compilation if source unchanged
- Create "quick-start" command that skips native compilation for testing React-only changes
- Move binaries to on-demand download (lazy init when actually needed)

## Fragile Areas

### 1. Hotkey Manager with Platform-Specific Implementations

**Files:** `src/helpers/hotkeyManager.js`, `src/helpers/globeKeyManager.js`, `src/helpers/windowsKeyManager.js`, `src/helpers/gnomeShortcut.js`, `src/helpers/hyprlandShortcut.js`

**Why fragile:**
- 5+ different implementations with different error modes
- If Electron globalShortcut fails, falls back to platform-specific code
- If platform-specific code fails (e.g., native binary missing), app can't record
- No unified error handling or fallback strategy

**Safe modification:**
- Always test on all 3 platforms (macOS, Windows, Linux) when changing hotkey logic
- Test failure scenarios: native binary missing, globalShortcut unavailable, Wayland session
- Add debug log at every fallback point showing which strategy is active

**Test coverage:** Manual testing required; no automated tests for platform-specific hotkey behavior

### 2. Audio Pipeline with Multiple Fallback Chains

**Files:** `src/helpers/audioManager.js`, `src/helpers/whisper.js`, `src/helpers/parakeet.js`, `src/helpers/ipcHandlers.js`

**Why fragile:**
- Silence detection → default to `_peakRms = 1` if it fails (silent failure)
- Transcription provider → whisper.cpp → cloud API fallback chain not always triggered correctly
- Streaming provider → token cache → re-auth → fresh token chain can get stuck if any step fails
- No state machine to track which fallback is active

**Safe modification:**
- Update debug logs when adding new fallbacks
- Test each fallback path individually (e.g., disable FFmpeg, verify whisper still works)
- Verify error messages are user-facing, not swallowed

**Test coverage:** No automated tests for fallback chains; manual verification required

### 3. Database State with Async IPC Handlers

**Files:** `src/helpers/database.js`, `src/helpers/ipcHandlers.js` (database calls)

**Why fragile:**
- No transaction boundaries in multi-step operations (e.g., save transcription + update metadata)
- Race conditions if two IPC handlers access same row (read-modify-write not atomic)
- Database locked errors not handled consistently

**Safe modification:**
- Always use transactions for multi-step db operations
- Wrap database calls with lock mechanism if multiple handlers touch same tables
- Test database with high concurrency (many transcriptions simultaneously)

**Test coverage:** No automated database tests; manual verification of concurrent updates required

## Scaling Limits

### 1. In-Memory Streaming State

**Resource:** Streaming connections stored in-memory in `DeepgramStreaming`, `AssemblyAiStreaming` classes with unbounded buffers.

**Current capacity:** Single streaming session with ~100KB/sec audio data

**Limit:** Memory fills if client disconnects without cleanup; buffer grows unbounded if no frame terminator received

**Scaling path:**
- Implement maximum buffer size with overflow handling
- Add connection timeout to close stale streams
- Monitor memory usage per streaming session

### 2. SQLite Database Without Vacuuming

**Resource:** `database.js` inserts transcriptions without periodic `VACUUM` command.

**Current capacity:** ~100K transcriptions at normal usage (~10/day)

**Limit:** Database file grows indefinitely; deletion doesn't reclaim space; query performance degrades after database reaches >100MB

**Scaling path:**
- Schedule monthly `VACUUM` operation
- Implement retention policy (archive old transcriptions after 1 year)
- Add `--optimize` command to shrink database

### 3. Meeting Detection Notification Queue

**Resource:** `meetingDetectionEngine.js` queues notifications in-memory if user recording (line 74).

**Current capacity:** Queue unbounded; each notification stored in `activeDetections` Map

**Limit:** If user records continuously for hours and multiple detections trigger, queue grows without bound

**Scaling path:**
- Set maximum queue size (e.g., 50 pending notifications)
- Drop oldest notifications if queue overflows
- Store to database for later review if space limited

## Dependencies at Risk

### 1. electron-updater Versioning

**Risk:** `electron-updater` v6.6.2 has breaking changes in newer Electron versions; may not work with Electron 40+.

**Files:** `package.json` (line 122)

**Impact:** Auto-update may break in future Electron releases

**Migration plan:**
- Monitor electron-updater releases for Electron 40+ support
- Test update mechanism with upcoming Electron versions before updating
- Keep update fallback mechanism (manual download link)

### 2. better-sqlite3 Native Dependency

**Risk:** `better-sqlite3` v12.4.2 is native binding; requires recompilation for each Node/Electron version.

**Files:** `package.json` (line 117), build scripts

**Impact:**
- Cannot use with Node version other than 22 (package.json line 8)
- CI will fail if Node version drifts
- Compilation failures block app build

**Migration plan:**
- Monitor for `better-sqlite3` updates that support broader Node versions
- Consider fallback to `sql.js` (pure JS) if compilation becomes blocker
- Document Node 22 requirement clearly for contributors

### 3. dbus-next for GNOME/Wayland Support

**Risk:** `dbus-next` v0.10.2 is new dependency for GNOME Wayland support; no fallback if module fails to import.

**Files:** `package.json` (line 120), `src/helpers/gnomeShortcut.js`, `src/helpers/hyprlandShortcut.js`

**Impact:** If D-Bus unavailable, GNOME hotkey registration fails silently and app falls back to regular hotkey (may not work on Wayland)

**Migration plan:**
- Catch import errors and disable D-Bus features gracefully
- Test on KDE Wayland (different D-Bus surface)
- Add feature flag to disable GNOME shortcuts for troubleshooting

## Missing Critical Features

### 1. Automatic Hotkey Conflict Detection

**Problem:** No mechanism to detect if chosen hotkey conflicts with system shortcuts before registration. App registers conflicting hotkey, hotkey silently fails at runtime.

**Files:** `src/helpers/hotkeyManager.js`

**Blocks:** Power users who customize hotkeys; no feedback when choice doesn't work

**Implementation:** Pre-check against known system hotkeys on each platform (see hotkeyManager.js lines 225-230 for conflict detection, but only for Electron-level conflicts)

### 2. Streaming Error Recovery Without User Intervention

**Problem:** If streaming connection drops mid-transcription, user must manually stop and restart streaming. No automatic retry or recovery.

**Files:** `src/helpers/ipcHandlers.js` (streaming handlers), `src/helpers/deepgramStreaming.js`, `src/helpers/assemblyAiStreaming.js`

**Blocks:** Long-form dictation over unreliable networks

**Implementation:** Implement automatic reconnection with exponential backoff; buffer audio during disconnect; resume on reconnect

### 3. Hotkey Registration Status Dashboard

**Problem:** No way for user to see which hotkey is active, what mode is enabled (tap vs push-to-talk), or why hotkey isn't working without consulting debug logs.

**Files:** `src/components/SettingsPage.tsx`

**Blocks:** Troubleshooting hotkey issues

**Implementation:** Add status section in Settings showing active hotkey, activation mode, availability status

## Test Coverage Gaps

### 1. Streaming Provider Fallback

**What's not tested:** Streaming fails over from one provider to another; token cache misses; authentication failures

**Files:** `src/helpers/ipcHandlers.js` (streaming handlers), `src/helpers/audioManager.js` (provider selection)

**Risk:** Streaming falls back to wrong provider or doesn't fall back at all

**Priority:** HIGH - User-facing failure mode

### 2. Audio Pipeline Error Handling

**What's not tested:**
- FFmpeg unavailable → fallback to system audio works
- Whisper.cpp binary missing → cloud API fallback triggers
- All three transcription providers fail → user gets meaningful error

**Files:** `src/helpers/audioManager.js`, `src/helpers/whisper.js`

**Risk:** Transcription fails with cryptic error instead of attempting fallback

**Priority:** HIGH - Core feature

### 3. Platform-Specific Clipboard (All Platforms)

**What's not tested:**
- Linux with no paste utility available → graceful fallback
- macOS accessibility denied → user prompted correctly
- Windows nircmd missing → PowerShell fallback works
- All strategies attempted and logged

**Files:** `src/helpers/clipboard.js`

**Risk:** Clipboard paste silently fails; user thinks transcription is broken

**Priority:** MEDIUM - Important feature but not critical

### 4. Hotkey Conflict Resolution (All Platforms)

**What's not tested:**
- System reserved hotkey chosen → app suggests alternative
- Platform-specific hotkey format → correct registration on each platform
- Wayland vs X11 → both paths work

**Files:** `src/helpers/hotkeyManager.js`, platform-specific managers

**Risk:** Hotkey doesn't work; hard to diagnose

**Priority:** MEDIUM - Important for usability

### 5. Meeting Detection Under Load

**What's not tested:**
- Multiple simultaneous detections → notification coalescing works
- Rapid on/off toggles → detection state cleaned up correctly
- Long recording sessions → no memory leaks from queued notifications

**Files:** `src/helpers/meetingDetectionEngine.js`, `src/helpers/audioActivityDetector.js`

**Risk:** Meeting notifications become unreliable or app crashes under heavy usage

**Priority:** MEDIUM - Affects reliability

---

*Concerns audit: 2026-03-23*
