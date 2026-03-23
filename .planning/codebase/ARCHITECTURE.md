# Architecture

**Analysis Date:** 2026-03-23

## Pattern Overview

**Overall:** Multi-window Electron application with dual process architecture (main + renderer) using context isolation security model

**Key Characteristics:**
- Main window (dictation overlay) and control panel window (full UI) share React codebase via URL routing
- Process separation: Electron main process handles system integration, native modules, IPC routing; renderer process handles React UI
- Cross-cutting handlers in main process (audio, clipboard, hotkeys, database) exposed via secure IPC bridge
- Cloud-first transcription architecture with local model fallbacks (whisper.cpp, NVIDIA Parakeet)
- Event-driven meeting detection (process monitoring, microphone activity, calendar sync)
- Multi-provider AI reasoning pipeline (OpenAI, Anthropic, Google Gemini, local llama.cpp)

## Layers

**Main Process (Node.js + Native Binaries):**
- Purpose: Electron lifecycle management, system integration, IPC handler registration, background services
- Location: `main.js` at project root, handler implementations in `src/helpers/`
- Contains: Window managers, hotkey handlers, audio pipeline, database operations, clipboard integration, native process monitoring
- Depends on: Electron APIs, Node.js modules, native binaries (whisper.cpp, sherpa-onnx, llama.cpp)
- Used by: Renderer process via IPC bridge (preload.js)

**Preload Bridge (Security Boundary):**
- Purpose: Secure IPC interface between main and renderer processes (context isolation enabled)
- Location: `preload.js` at project root
- Contains: `electronAPI` object with exposed IPC invoke/emit/listen methods
- Depends on: Main process handlers, Electron contextBridge
- Used by: React renderer components via `window.electronAPI`

**Renderer Process (React Application):**
- Purpose: UI rendering, user interaction handling, state management, audio recording
- Location: `src/` directory
- Contains: React components, hooks, Zustand stores, service layer, utilities
- Depends on: Preload bridge (IPC), React libraries, i18next for localization
- Used by: Both windows (dictation overlay and control panel) via URL-based routing

**Audio Pipeline (Renderer + Main Process):**
- Purpose: Record audio, transcribe via local or cloud providers, handle streaming
- Location: `src/hooks/useAudioRecording.js` (recorder), `src/helpers/audioManager.js` (transcription), streaming providers in helpers
- Contains: MediaRecorder API wrapper, transcription orchestration, streaming clients (Deepgram, AssemblyAI, OpenAI Realtime)
- Depends on: FFmpeg (bundled), cloud APIs, local binaries (whisper.cpp, sherpa-onnx)
- Used by: App component for dictation flow

**AI Reasoning Layer:**
- Purpose: Process transcribed text through cloud/local AI models
- Location: `src/services/ReasoningService.ts`, `src/services/LocalReasoningService.ts`
- Contains: OpenAI Responses API client, Anthropic/Gemini via IPC bridge, local llama.cpp via fork/pipe
- Depends on: Model registry, API endpoints, secure caching, session management
- Used by: Audio manager post-transcription, notes processing

**Database Layer:**
- Purpose: Persist transcriptions, audio metadata, settings, notes, dictionary
- Location: `src/helpers/database.js` (main process), `preload.js` (IPC interface)
- Contains: better-sqlite3 operations (transcriptions, notes, audio metadata, dictionaries)
- Depends on: better-sqlite3 native module, Electron app userData path
- Used by: IPC handlers, audio processing, notes service

**State Management:**
- Purpose: Client-side application state (settings, transcriptions, notes)
- Location: `src/stores/`
- Contains: Zustand stores for settings, transcriptions, notes, actions
- Depends on: localStorage, IPC handlers for persistence
- Used by: React components, hooks, services

## Data Flow

**Dictation Recording Flow:**

1. User presses hotkey (registered via hotkeyManager in main process)
2. Main process emits "start-dictation" IPC event to renderer
3. Renderer's useHotkey hook receives event, updates app state to "recording"
4. App component begins MediaRecorder via useAudioRecording hook
5. Audio chunks collected in memory as user speaks
6. User releases hotkey → main process emits "stop-dictation" IPC event
7. Renderer collects audio chunks into Blob → converts to ArrayBuffer
8. Sends ArrayBuffer via IPC to main process audioManager
9. audioManager writes Blob to temporary file, invokes transcription provider:
   - Cloud (OpenAI, Groq, Mistral): HTTP POST to API endpoint
   - Streaming (Deepgram, AssemblyAI, OpenAI Realtime): WebSocket connection
   - Local (whisper.cpp, Parakeet): Spawn child process, pipe audio file
10. Transcription result returned to renderer via IPC ("transcription-complete" event)
11. Renderer updates UI with transcribed text
12. If reasoning enabled: send text to ReasoningService (cloud or local)
13. Reasoning result returned to renderer
14. Database saves transcription record (optional audio storage)
15. Clipboard injection: send final text via clipboard manager (IPC)

**Settings Persistence Flow:**

1. User changes setting in SettingsPage component
2. `useSettings` hook updates Zustand store (in-memory)
3. Hook triggers IPC call to sync setting to main process (`save-setting`)
4. Main process handler writes to better-sqlite3 (settings table)
5. Main process broadcasts "settings-changed" IPC event to all renderer windows
6. useSettings hook listens for this event, re-reads from database if needed
7. Settings survive app restarts via database persistence

**Meeting Detection Flow:**

1. MeetingDetectionEngine orchestrates three parallel sources:
   - Process detection: systemPreferences.subscribeWorkspaceNotification (macOS) or processListCache polling (Windows/Linux)
   - Microphone activity: native listeners (macos-mic-listener, windows-mic-listener) or pactl subscribe (Linux)
   - Calendar: GoogleCalendarManager syncs with 2-min interval + exponential backoff on failures
2. Each source emits "meeting-detected" with confidence level
3. Engine gates notifications during recording (tap-to-talk or push-to-talk)
4. Post-recording: 2.5s cooldown before showing queued notifications
5. Multiple signals coalesced: process > audio priority, one notification shown
6. Notification suppresses itself if overlapping with another recording

## State Management

**Zustand Stores:**
- `settingsStore.ts`: application settings (provider, model, API keys, hotkey, etc.)
- `transcriptionStore.ts`: current/recent transcription history
- `noteStore.ts`: notes CRUD operations
- `actionStore.ts`: pending user actions/confirmations

**localStorage:**
- Settings: persisted by settingsStore (custom encrypted handling for API keys)
- User preferences: language, theme, onboarding completion flag
- Cache: temporary data for in-session state

**better-sqlite3 Database (main process):**
- `transcriptions` table: text, processing method, timestamp, audio file reference
- `audio_files` table: stored audio with metadata and duration
- `notes` table: user-created notes with folders and tags
- `dictionaries` table: custom word lists for transcription hints
- `settings` table: persisted application configuration

## Key Abstractions

**WindowManager:**
- Purpose: Lifecycle management for main window (dictation overlay) and control panel
- Location: `src/helpers/windowManager.js`
- Pattern: Singleton managing window instances, communication, state sync
- Used by: main.js initialization, hotkey callbacks for window show/hide

**HotkeyManager:**
- Purpose: Unified hotkey registration across platforms
- Location: `src/helpers/hotkeyManager.js`
- Pattern: Abstracts platform differences (Electron globalShortcut, GNOME D-Bus, Hyprland hyprctl, Windows native listener)
- Used by: main.js, dictation flow startup

**AudioManager:**
- Purpose: Transcription orchestration, provider selection, streaming client management
- Location: `src/helpers/audioManager.js`
- Pattern: Manages cached API keys, handles provider-specific streaming setup, retry logic
- Responsibility: Selects transcription provider → opens stream/file → monitors status → delivers result
- Used by: IPC handler for "transcribe-audio" requests

**ReasoningService:**
- Purpose: AI processing for post-transcription text refinement
- Location: `src/services/ReasoningService.ts`
- Pattern: Abstract base (BaseReasoningService) with implementations for each provider
- Responsibility: Selects cloud provider or local llama.cpp → builds API request → retries on failure → caches results
- Used by: Audio manager, manual text processing

**DatabaseManager:**
- Purpose: SQLite operations with prepared statements
- Location: `src/helpers/database.js`
- Pattern: Singleton with lazy initialization, table existence checks
- Used by: IPC handlers for CRUD operations

**ClipboardManager:**
- Purpose: Cross-platform text paste (auto-injection into active application)
- Location: `src/helpers/clipboard.js`
- Pattern: Platform detection (macOS/Windows/Linux) → uses appropriate paste method
  - macOS: AppleScript `System Events` keystroke + accessibility check
  - Windows: PowerShell SendKeys or nircmd.exe
  - Linux: xdotool (X11) / wtype (Wayland) / ydotool (GNOME) with terminal detection
- Used by: IPC handler, post-transcription workflow

**MeetingDetectionEngine:**
- Purpose: Orchestrates multi-source meeting detection with notification gating
- Location: Not a single file; logic spread across helpers (processListCache, audioActivityDetector, googleCalendarManager)
- Pattern: Listens to events from multiple detectors → coalesces signals → gates during recording
- Used by: Main process for suppression logic during recording

**ModelRegistry:**
- Purpose: Single source of truth for all transcription and reasoning models
- Location: `src/models/ModelRegistry.ts` (derives from `modelRegistryData.json`)
- Pattern: Centralized model definitions with cloud provider enums, local model URLs, prompt templates
- Used by: Audio manager (transcription), ReasoningService (reasoning), UI components (model selection)

## Entry Points

**Application Entry (Main Process):**
- Location: `main.js`
- Triggers: User launches app or `npm start` / `npm dev`
- Responsibilities:
  - Environment setup (channel detection, platform-specific Electron switches)
  - Manager initialization (audio, database, clipboard, hotkeys, windows, calendar, etc.)
  - IPC handler registration via IPCHandlers class
  - Window creation and lifecycle
  - Tray icon setup
  - Auto-update initialization

**Renderer Entry (React Application):**
- Location: `src/main.jsx` (entry point for Vite build)
- Renders into: `src/index.html` (includes root div#root)
- App component: `src/App.jsx`
- Triggers: Electron loads file:// URL after main.js creates window
- Responsibilities:
  - Root React component managing dictation overlay UI
  - Hotkey state management
  - Audio recording orchestration
  - Toast notifications
  - Window drag behavior

**Control Panel Entry (Same React App, Different Route):**
- Location: Same Vite build as dictation overlay
- Route detection: Window URL parameter or `location.pathname`
- Components: ControlPanel.tsx for full settings/history/notes UI
- Responsibilities: Settings management, transcription history, note creation, model management

**Preload Process:**
- Location: `preload.js`
- Triggers: Electron loads preload script before renderer process sandbox
- Responsibilities: Expose safe IPC methods on `window.electronAPI` via contextBridge

## Error Handling

**Strategy:** Multi-layered with user-facing toast notifications and detailed debug logs

**Patterns:**

- **Transcription Failures:**
  - Local whisper.cpp: retry 3x with exponential backoff, fall back to cloud provider
  - Cloud API: retry via `withRetry()` utility, show "transcription failed" toast
  - Audio format issues: log and skip, show "unable to process audio" notification
  - Network timeouts: 10s socket timeout on all HTTP requests, fail with user message

- **API Authentication:**
  - Missing API key: show "API key required" toast, open settings
  - Invalid key format: cache miss forces re-validation on next request
  - Session expiration (calendar sync): exponential backoff prevents hammering API

- **Database Errors:**
  - Table creation failures: app crashes with error dialog (unrecoverable)
  - Row insertion fails: log and show "failed to save" toast, allow retry
  - Constraint violations: silently skip duplicate entries

- **Hotkey Registration:**
  - Default hotkey (backtick) unavailable: fall back to F8, notify user
  - GNOME Wayland shortcuts fail: fallback to Electron globalShortcut
  - Windows key listener binary missing: fall back to Electron globalShortcut

- **Window Management:**
  - Main window destroyed unexpectedly: recreate on next hotkey press
  - Overlay transparency not supported (Linux + old GPU): warn and use opaque window

## Cross-Cutting Concerns

**Logging:**
- Framework: Custom `debugLogger.js` with file output to app userData/logs
- Trigger: `OPENWHISPR_LOG_LEVEL=debug` environment variable
- Verbosity: Logs hotkey registration, FFmpeg resolution, audio pipeline stages, reasoning requests/responses
- Pattern: Use `logger.log()` in main process, `logger` in renderer (TypeScript version)

**Validation:**
- Audio format: Check MIME type, duration > 0.1s before transcription
- API keys: Validate format (not empty, not placeholder strings) before API call
- Settings: Type-check in settingsStore before persistence
- Language code: Validate against language registry before passing to models

**Authentication:**
- Cloud APIs: Store API keys in Zustand store (optional encryption planned), cache in-memory
- Calendar OAuth: Google Calendar token via neonAuth library, 10s socket timeout, exponential backoff
- Session persistence: API keys lost on app restart (not persisted to disk for now)
- Custom endpoints: Support custom OpenAI-compatible base URLs via settings

**Internationalization:**
- Framework: react-i18next v15 with i18next v25
- Keys: Nested structure (e.g., `notes.editor.placeholder`)
- Languages: en, es, fr, de, pt, it, ru, zh-CN, zh-TW (9 total)
- Pattern: `const { t } = useTranslation()` in components, t("key.path") for lookups
- Interpolation: `{{variable}}` syntax for dynamic values
- UI strings: REQUIRED in all 9 language files; never hardcoded in components

---

*Architecture analysis: 2026-03-23*
