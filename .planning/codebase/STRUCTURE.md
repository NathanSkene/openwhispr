# Codebase Structure

**Analysis Date:** 2026-03-23

## Directory Layout

```
openwhispr/
├── main.js                          # Electron main process entry point
├── preload.js                       # Context isolation security bridge
├── package.json                     # Dependencies and build scripts
├── vite.config.mjs                  # Vite config (in src/ directory)
├── electron-builder.json            # Release build config
├── electron-builder.dev.json        # Dev build config
│
├── src/                             # React application source code
│   ├── main.jsx                     # React entry point (Vite)
│   ├── index.html                   # HTML template
│   ├── App.jsx                      # Root component (dictation overlay)
│   ├── index.css                    # Tailwind + global styles
│   │
│   ├── components/                  # React components
│   │   ├── ControlPanel.tsx         # Full settings/history/notes UI
│   │   ├── SettingsPage.tsx         # Settings interface (162KB)
│   │   ├── OnboardingFlow.tsx       # 8-step first-time setup
│   │   ├── HistoryView.tsx          # Transcription history view
│   │   ├── NotesView.tsx            # Notes CRUD interface
│   │   ├── RecordingOverlay.tsx     # Real-time audio waveform
│   │   ├── MeetingNotificationOverlay.tsx
│   │   ├── AgentOverlay.tsx         # AI agent response display
│   │   ├── ReasoningModelSelector.tsx
│   │   ├── ui/                      # shadcn/ui + custom components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Dialog.tsx
│   │   │   ├── Dropdown.tsx
│   │   │   ├── Tabs.tsx
│   │   │   ├── Toast.tsx
│   │   │   └── ...
│   │   ├── settings/                # Settings sub-panels
│   │   ├── notes/                   # Notes-related components
│   │   │   ├── DictationWidget.tsx
│   │   │   └── NoteEditor.tsx
│   │   └── agent/                   # Agent-specific components
│   │
│   ├── hooks/                       # Custom React hooks
│   │   ├── useAudioRecording.js     # MediaRecorder wrapper
│   │   ├── useSettings.ts           # Zustand settings store integration
│   │   ├── useHotkey.js             # Hotkey state management
│   │   ├── useMeetingTranscription.ts
│   │   ├── useNoteRecording.ts
│   │   ├── usePermissions.ts        # System permission checks
│   │   ├── useLocalModels.ts        # Local model management
│   │   ├── useModelDownload.ts      # Model download progress
│   │   ├── useClipboard.ts          # Clipboard operation hook
│   │   └── ...
│   │
│   ├── stores/                      # Zustand state management
│   │   ├── settingsStore.ts         # Application settings (33KB)
│   │   ├── transcriptionStore.ts    # Transcription history
│   │   ├── noteStore.ts             # Notes CRUD
│   │   └── actionStore.ts           # User action confirmations
│   │
│   ├── services/                    # Business logic layer
│   │   ├── ReasoningService.ts      # AI processing (42KB)
│   │   ├── BaseReasoningService.ts  # Abstract base class
│   │   ├── LocalReasoningService.ts # llama.cpp integration
│   │   ├── NotesService.ts          # Notes operations
│   │   └── localReasoningBridge.js  # Fork/pipe to llama.cpp
│   │
│   ├── helpers/                     # Main process functionality (called via IPC)
│   │   ├── ipcHandlers.js           # IPC handler registration (145KB)
│   │   ├── audioManager.js          # Transcription orchestration (94KB)
│   │   ├── database.js              # SQLite operations (44KB)
│   │   ├── clipboard.js             # Cross-platform text injection (55KB)
│   │   ├── hotkeyManager.js         # Global hotkey registration (26KB)
│   │   ├── windowManager.js         # Window lifecycle
│   │   ├── windowConfig.js          # Window configuration constants
│   │   ├── dragManager.js           # Window dragging overlay
│   │   ├── googleCalendarManager.js # Google Calendar sync
│   │   ├── whisper.js               # whisper.cpp binary wrapper
│   │   ├── parakeet.js              # Parakeet model management
│   │   ├── parakeetServer.js        # sherpa-onnx CLI wrapper
│   │   ├── llamaServer.js           # llama.cpp server wrapper
│   │   ├── meetingDetectionEngine.js
│   │   ├── meetingProcessDetector.js # Process-based meeting detection
│   │   ├── audioActivityDetector.js # Microphone-based detection
│   │   ├── processListCache.js      # Shared process list cache
│   │   ├── gnomeShortcut.js         # GNOME Wayland hotkeys (9KB)
│   │   ├── hyprlandShortcut.js      # Hyprland Wayland hotkeys (9KB)
│   │   ├── globeKeyManager.js       # macOS Globe key detection
│   │   ├── windowsKeyManager.js     # Windows key listener binary
│   │   ├── assemblyAiStreaming.js   # AssemblyAI streaming client
│   │   ├── deepgramStreaming.js     # Deepgram streaming client
│   │   ├── openaiRealtimeStreaming.js
│   │   ├── audioStorage.js          # Audio file persistence
│   │   ├── audioActivityDetector.js # Microphone activity detection
│   │   ├── ffmpegUtils.js           # Audio format conversion
│   │   ├── environment.js           # Environment variable management
│   │   ├── debugLogger.js           # Debug logging with file output
│   │   ├── devServerManager.js      # Vite dev server integration
│   │   ├── downloadUtils.js         # GitHub release downloads
│   │   └── ...
│   │
│   ├── utils/                       # Utility functions
│   │   ├── hotkeys.ts               # Hotkey formatting/parsing
│   │   ├── languages.ts             # Language code support
│   │   ├── retry.ts                 # Exponential backoff retry logic
│   │   ├── logger.ts                # TypeScript logging wrapper
│   │   ├── SecureCache.ts           # In-memory API key cache
│   │   ├── urlUtils.ts              # URL validation/normalization
│   │   ├── audioDeviceUtils.ts      # Audio device detection
│   │   ├── systemAudio.ts           # System audio stream access
│   │   ├── languageSupport.ts       # Language validation per model
│   │   └── ...
│   │
│   ├── models/                      # AI model configuration
│   │   ├── ModelRegistry.ts         # TypeScript wrapper
│   │   ├── modelRegistryData.json   # Single source of truth
│   │   │   ├── cloudProviders: OpenAI, Anthropic, Google Gemini
│   │   │   └── localProviders: GGUF models via llama.cpp
│   │   └── ...
│   │
│   ├── config/                      # Application configuration
│   │   ├── constants.ts             # API endpoints, token limits
│   │   ├── InferenceConfig.ts       # Model inference settings
│   │   ├── prompts.ts               # System prompts for reasoning
│   │   ├── promptData.json          # Prompt templates (5.4KB)
│   │   ├── languageRegistry.json    # 58 supported languages (24KB)
│   │   └── aiProvidersConfig.ts     # Derives AI modes from registry
│   │
│   ├── lib/                         # Third-party integrations
│   │   └── neonAuth.ts              # Google Calendar OAuth
│   │
│   ├── locales/                     # i18n translation files
│   │   ├── en/translation.json
│   │   ├── es/translation.json
│   │   ├── fr/translation.json
│   │   ├── de/translation.json
│   │   ├── pt/translation.json
│   │   ├── it/translation.json
│   │   ├── ru/translation.json
│   │   ├── zh-CN/translation.json
│   │   └── zh-TW/translation.json
│   │
│   ├── assets/                      # Static resources
│   │   ├── icons/
│   │   │   ├── providers/           # Provider-specific icons
│   │   │   └── ...
│   │   └── fonts/
│   │
│   ├── types/                       # TypeScript type definitions
│   │   ├── electron.d.ts            # electronAPI typing
│   │   └── ...
│   │
│   └── dist/                        # Build output (generated)
│       ├── index.html
│       ├── assets/
│       │   ├── index-*.js
│       │   ├── *.css
│       │   └── vendor-*.js
│       └── runtime-env.json         # Built-in API endpoints
│
├── resources/                       # Native binaries and platform-specific code
│   ├── bin/                         # Bundled binary executables
│   │   ├── whisper-cpp-mac-arm64    # Local transcription (arm64)
│   │   ├── whisper-cpp-mac-x64      # Local transcription (x64)
│   │   ├── sherpa-onnx-*            # NVIDIA Parakeet runtime
│   │   ├── macos-mic-listener       # CoreAudio mic detection (macOS)
│   │   ├── macos-fast-paste         # AppleScript optimized paste (macOS)
│   │   ├── windows-key-listener.exe # Low-level keyboard hook (Windows)
│   │   ├── windows-mic-listener.exe # WASAPI mic detection (Windows)
│   │   ├── windows-fast-paste.exe   # PowerShell paste helper (Windows)
│   │   ├── linux-fast-paste         # XTest paste helper (Linux)
│   │   └── nircmd.exe               # Windows utility commands
│   │
│   ├── mac/                         # macOS-specific resources
│   │   ├── entitlements.mac.plist   # Codesigning permissions
│   │   └── background.png           # DMG background
│   │
│   ├── linux/                       # Linux-specific resources
│   │   ├── open-whispr.desktop      # Desktop entry file
│   │   └── ...
│   │
│   ├── nsis/                        # Windows installer config (NSIS)
│   │   └── installer.nsi
│   │
│   └── .swift-module-cache/         # Cached compiled Swift modules
│
├── scripts/                         # Build and development scripts
│   ├── build-*.js                   # Native module compilation
│   │   ├── build-globe-listener.js  # macOS Globe key detector
│   │   ├── build-macos-mic-listener.js
│   │   ├── build-windows-key-listener.js
│   │   ├── build-macos-fast-paste.js
│   │   ├── build-windows-fast-paste.js
│   │   ├── build-linux-fast-paste.js
│   │   ├── build-text-monitor.js
│   │   └── build-media-remote.js
│   │
│   ├── download-*.js                # Binary/model downloads from GitHub/HuggingFace
│   │   ├── download-whisper-cpp.js
│   │   ├── download-llama-server.js
│   │   ├── download-sherpa-onnx.js
│   │   ├── download-nircmd.js
│   │   ├── download-windows-key-listener.js
│   │   ├── download-windows-mic-listener.js
│   │   └── download-windows-fast-paste.js
│   │
│   ├── run-electron.js              # Dev server startup helper
│   ├── check-i18n.js                # Validate i18n keys
│   ├── lib/download-utils.js        # Shared download/extract utilities
│   └── ...
│
├── .github/workflows/               # CI/CD pipelines
│   ├── build-windows-key-listener.yml  # Auto-build Windows binary
│   ├── build-windows-mic-listener.yml
│   └── ...
│
├── .planning/codebase/              # GSD analysis documents
│   ├── ARCHITECTURE.md              # This file
│   ├── STRUCTURE.md                 # Directory layout and conventions
│   └── ...
│
└── .claude/skills/openwhispr/       # Nathan's skill docs
    ├── SKILL.md                     # Workflow documentation
    └── ...
```

## Directory Purposes

**`src/`:**
- Purpose: React application source code (dictation overlay and control panel)
- Contains: Components, hooks, stores, services, utilities, configuration
- Key files: `App.jsx` (root), `ControlPanel.tsx` (settings), `SettingsPage.tsx` (162KB settings UI)

**`src/components/`:**
- Purpose: Reusable React components
- Contains: UI components (shadcn/ui), page views, modals, overlays
- Key files: `ControlPanel.tsx`, `SettingsPage.tsx`, `OnboardingFlow.tsx`, `HistoryView.tsx`

**`src/hooks/`:**
- Purpose: Custom React hooks for stateful logic
- Contains: Audio recording, settings, permissions, downloads, local models
- Key files: `useSettings.ts` (store integration), `useAudioRecording.js` (MediaRecorder)

**`src/stores/`:**
- Purpose: Zustand state management (replaces Redux/Context)
- Contains: Global app state (settings, transcriptions, notes)
- Key files: `settingsStore.ts` (33KB), `transcriptionStore.ts`, `noteStore.ts`

**`src/services/`:**
- Purpose: Business logic layer (transcription, reasoning, notes)
- Contains: API clients, model selection, retry logic
- Key files: `ReasoningService.ts` (42KB), `NotesService.ts`, `LocalReasoningService.ts`

**`src/helpers/`:**
- Purpose: Main process functionality (called via IPC from renderer)
- Contains: Audio pipeline, database, clipboard, hotkeys, window management
- Key files: `ipcHandlers.js` (145KB, all handler registration), `audioManager.js` (94KB)

**`src/utils/`:**
- Purpose: Pure utility functions (no side effects)
- Contains: Language support, retry logic, logger, URL validation, audio devices
- Key files: `retry.ts` (exponential backoff), `languages.ts` (58 languages)

**`src/config/`:**
- Purpose: Application-wide configuration constants
- Contains: API endpoints, token limits, model registry, prompts
- Key files: `constants.ts`, `promptData.json`, `languageRegistry.json`

**`src/models/`:**
- Purpose: AI model definitions and registry
- Contains: CloudProvider/LocalProvider enums, model URLs, prompt templates
- Key files: `ModelRegistry.ts` (wrapper), `modelRegistryData.json` (source of truth)

**`src/locales/`:**
- Purpose: i18n translation files (react-i18next)
- Contains: JSON translation keys for 9 languages
- Key files: `en/translation.json` (base), es/fr/de/pt/it/ru/zh-CN/zh-TW variants

**`resources/`:**
- Purpose: Native binaries, platform-specific code, installer configs
- Contains: whisper.cpp, sherpa-onnx, llama.cpp, native listeners, DMG background
- Key files: `bin/whisper-cpp-*` (bundled), `mac/entitlements.mac.plist`

**`resources/bin/`:**
- Purpose: Bundled executable binaries (unpacked from ASAR at runtime)
- Contains: Cross-platform transcription/paste/hotkey binaries
- Platform-specific: Each platform gets its own set (macOS arm64/x64, Windows x64, Linux x64)

**`scripts/`:**
- Purpose: Build automation and setup
- Contains: Native module compilation, binary downloads from GitHub/HuggingFace
- Key files: `build-globe-listener.js`, `download-whisper-cpp.js`, `run-electron.js`

**`.planning/codebase/`:**
- Purpose: GSD analysis documents (auto-generated)
- Contains: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md
- Used by: `/gsd:plan-phase` and `/gsd:execute-phase` commands

## Key File Locations

**Entry Points:**
- `main.js`: Electron main process initialization
- `preload.js`: Context isolation security bridge
- `src/main.jsx`: Vite React entry (bundles both windows)
- `src/index.html`: HTML template loaded by Electron

**Configuration:**
- `package.json`: Dependencies, build scripts
- `electron-builder.json`: Release build (notarization, signing, install paths)
- `electron-builder.dev.json`: Dev build (codesigning disabled)
- `src/vite.config.mjs`: Vite build configuration
- `src/config/constants.ts`: API endpoints and token limits

**Core Logic:**
- `src/helpers/ipcHandlers.js`: All IPC handler implementations (145KB)
- `src/helpers/audioManager.js`: Transcription orchestration (94KB)
- `src/services/ReasoningService.ts`: AI processing pipeline (42KB)
- `src/stores/settingsStore.ts`: Global settings state (33KB)

**Testing:**
- `src/helpers/database.js`: SQLite schema and operations
- `src/components/SettingsPage.tsx`: Largest component (162KB)
- `src/hooks/useMeetingTranscription.ts`: Meeting-specific transcription
- `src/hooks/usePermissions.ts`: System permission management

## Naming Conventions

**Files:**
- Components: PascalCase.tsx (e.g., `ControlPanel.tsx`, `OnboardingFlow.tsx`)
- Hooks: camelCase.ts with `use` prefix (e.g., `useSettings.ts`, `useAudioRecording.js`)
- Utilities: camelCase.ts (e.g., `languages.ts`, `retry.ts`)
- Helpers (main process): camelCase.js (e.g., `audioManager.js`, `database.js`)
- Stores: camelCase.ts with `Store` suffix (e.g., `settingsStore.ts`)
- Services: PascalCase.ts (e.g., `ReasoningService.ts`)
- Types: camelCase.d.ts or inline in files (no separate types directory structure)

**Directories:**
- Plural for collections (e.g., `components/`, `hooks/`, `utils/`)
- camelCase for feature groupings (e.g., `src/components/settings/`, `src/components/notes/`)
- Lowercase for assets (e.g., `assets/icons/`, `locales/en/`)

**Functions:**
- React components: PascalCase (e.g., `export default function App()`)
- Hooks: camelCase with `use` prefix (e.g., `export function useSettings()`)
- Utility functions: camelCase (e.g., `export function formatHotkey()`)
- IPC handlers: kebab-case channel names (e.g., `"start-dictation"`, `"db-save-transcription"`)

**Variables & Constants:**
- State: camelCase (e.g., `const [isRecording, setIsRecording]`)
- Constants: UPPER_SNAKE_CASE (e.g., `const DEFAULT_TIMEOUT_MS = 5000`)
- Environment: UPPER_SNAKE_CASE (e.g., `process.env.NODE_ENV`)

## Where to Add New Code

**New Transcription Provider:**
- Streaming client: `src/helpers/{providerName}Streaming.js`
- Integration in audioManager: `src/helpers/audioManager.js` (add to STREAMING_PROVIDERS map)
- Config: `src/config/constants.ts` (add endpoint)
- Model registry: `src/models/modelRegistryData.json` (add to cloudProviders)

**New UI Feature:**
- Component: `src/components/{FeatureName}.tsx`
- Hook if stateful: `src/hooks/use{FeatureName}.ts`
- Store integration: `src/stores/` (new store or extend existing)
- Translation keys: Add to all 9 language files in `src/locales/{lang}/translation.json`
- Tests: Colocate as `{FeatureName}.test.tsx` (pattern not yet implemented; add alongside component)

**New IPC Handler:**
- Handler implementation: `src/helpers/ipcHandlers.js` (add to IPCHandlers class)
- Preload exposure: `preload.js` (add to electronAPI object)
- Renderer caller: Wrap in hook (`src/hooks/use*.ts`) or call directly from component

**New System Integration (Windows/macOS/Linux):**
- Binary/script: `scripts/build-{name}.js` (compile native module)
- Build step: `package.json` scripts (add to `compile:native` chain)
- Main process: Add manager class in `src/helpers/` or integration in relevant manager
- IPC if needed: Add handler in `ipcHandlers.js`

**Utilities:**
- Pure functions: `src/utils/{category}.ts` (group by domain, e.g., `languages.ts`)
- Logger: Use `logger` from `src/utils/logger.ts`
- Retry logic: Use `withRetry()` from `src/utils/retry.ts`

**Localization:**
- Translation key: Add to ALL 9 language files in `src/locales/`
- Never hardcode UI strings in components
- Format: `t("feature.component.key", { variable })`

## Special Directories

**`src/dist/` (Build Output):**
- Purpose: Generated by Vite build
- Contents: Bundled JavaScript, CSS, HTML, assets
- Generated: `npm run build:renderer`
- Committed: No (git-ignored)

**`resources/bin/.swift-module-cache/`:**
- Purpose: Cached compiled Swift modules for native listeners
- Generated: First run of `compile:native` script
- Committed: No (git-ignored)

**`.planning/codebase/`:**
- Purpose: GSD-generated analysis documents
- Contents: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md
- Generated: `/gsd:map-codebase` command
- Committed: Yes (for visibility in CI/PR reviews)

**`node_modules/`:**
- Purpose: NPM dependencies
- Installed: `npm install` or CI `npm ci`
- Committed: No (git-ignored)

---

*Structure analysis: 2026-03-23*
