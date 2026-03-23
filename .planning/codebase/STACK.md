# Technology Stack

**Analysis Date:** 2026-03-23

## Languages

**Primary:**
- TypeScript 5.9.3 - React components (`src/components/`, `src/hooks/`, `src/services/`)
- JavaScript (CommonJS) - Electron main process, helpers, scripts
- Swift - macOS native integrations (Globe key listener, mic listener, text monitor)
- C - Windows native integrations (key listener, mic listener)

**Secondary:**
- HTML/CSS - UI templates (Electron windows, forms)
- JSON - Configuration, localization files (`src/locales/{lang}/`)

## Runtime

**Environment:**
- Node.js 22 LTS (pinned in `.nvmrc` — do NOT regenerate `package-lock.json` with different major version)
- Electron 39 - Desktop framework

**Package Manager:**
- npm (Node Package Manager)
- Lockfile: `package-lock.json` (present, must be regenerated with Node 22)

## Frameworks

**Core:**
- React 19.1.0 - UI framework
- Vite 6.3.5 - Build tool and dev server (`npm run dev:renderer`)
- Electron 39.0.0 - Desktop application framework

**Desktop Integration:**
- electron-builder 26.4.0 - Multi-platform packaging and signing
- electron-updater 6.6.2 - Automatic app updates from GitHub releases

**UI/Styling:**
- Tailwind CSS 4.1.10 - Utility-first CSS framework
- shadcn/ui 0.9.5 - Reusable React components (buttons, dialogs, inputs, etc.)
- Radix UI (via shadcn) - Accessible component primitives
- PostCSS 8.5.6 - CSS transformation
- Autoprefixer 10.4.21 - Browser prefix generation
- Lucide React 0.518.0 - Icon library

**Internationalization:**
- i18next 25.8.4 - Translation engine
- react-i18next 15.7.4 - React integration for i18next
- 9 languages supported: en, es, fr, de, pt, it, ru, zh-CN, zh-TW (defined in `src/locales/`)

**Testing/Development:**
- ESLint 9.25.0 - Code linting (main process and src separate configs)
- Prettier 3.4.2 - Code formatting
- TypeScript 5.9.3 - Type checking (`npm run typecheck`)
- @types/react 19.1.2, @types/react-dom 19.1.2 - Type definitions
- concurrently 8.2.2 - Run multiple scripts simultaneously
- cross-env 10.0.0 - Cross-platform environment variables

## Key Dependencies

**Critical:**
- better-sqlite3 12.4.2 - Local SQLite database for transcription history (bundled in ASAR unpacked)
- ffmpeg-static 5.2.0 - Cross-platform FFmpeg binary (bundled in ASAR unpacked)
- electron-updater 6.6.2 - GitHub-based app updates
- zod 4.3.6 - Schema validation for type-safe config

**Audio Processing:**
- whisper.cpp (native binary) - Local speech-to-text (GGML format models)
- sherpa-onnx (native binary) - NVIDIA Parakeet transcription via ONNX runtime
- llama.cpp (native binary) - Local LLM inference for reasoning tasks

**Desktop:**
- dbus-next 0.10.2 - D-Bus IPC for GNOME/Hyprland Wayland global hotkeys
- ps-list 9.0.0 - Process list detection (for meeting apps on Windows/Linux)

**Data/Communication:**
- ws 8.19.0 - WebSocket client (for real-time transcription streaming)
- tar 7.4.3, unbzip2-stream 1.4.3, unzipper 0.12.3 - Archive extraction
- dotenv 16.3.1 - Environment variable loading

**Authentication/Database:**
- @neondatabase/auth 0.1.0-beta.21 - Neon Auth client
- @neondatabase/neon-js 0.1.0-beta.22 - Neon database JS client
- @vercel/blob 2.3.0 - Vercel Blob storage (for cloud features)

**State Management:**
- zustand 5.0.11 - Lightweight state management (useSettings, useHotkey)

**Utilities:**
- clsx 2.1.1 - Dynamic className merging
- tailwind-merge 3.3.1 - Merge Tailwind CSS classes intelligently
- class-variance-authority 0.7.1 - CSS class variants
- react-markdown 10.1.0 - Markdown rendering for notes
- object-assign 4.1.1 - Object.assign polyfill

## Configuration

**Environment:**
- `.env` file (in `.planning/codebase/` — contains app secrets)
- `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `MISTRAL_API_KEY` - AI provider keys
- `VITE_NEON_AUTH_URL` - Neon Auth endpoint (optional)
- `VITE_OPENWHISPR_API_URL` - OpenWhispr Cloud API endpoint (optional)
- `OPENWHISPR_LOG_LEVEL` - Debug logging (optional, values: debug, info, warn, error)
- Other Vite env vars: `VITE_*` prefix exposed to frontend, `OPENWHISPR_*` for app-specific config

**Build:**
- `vite.config.js` - Vite dev server and build config
- `electron-builder.json` - Multi-platform packaging configuration
- `electron-builder.dev.json` - Development build config (for testing, unsigned)
- `eslint.config.js` - ESLint rules for main process (CommonJS)
- `src/eslint.config.js` - ESLint rules for React/TypeScript code
- `tsconfig.json` (in `src/`) - TypeScript compiler options
- `.prettierrc` - Prettier formatting rules

**Platform Configs:**
- macOS: `resources/mac/entitlements.mac.plist` - App sandbox/permissions
- Windows: NSIS installer config in `electron-builder.json`
- Linux: AppImage, DEB, RPM targets with DEB/RPM-specific dependencies

## Platform Requirements

**Development:**
- macOS 11+ (for building macOS and native targets)
- Windows 10+ (for Windows build)
- Linux (Ubuntu 20.04+) for Linux build
- Xcode Command Line Tools (macOS) or Visual Studio Build Tools (Windows) for native compilation
- Node.js 22 (must match `.nvmrc` — CI enforces Node 22)

**Production:**
- macOS 11+ (Intel/Apple Silicon via universal build)
- Windows 10+ (x64)
- Linux (AppImage works on most distros, DEB for Debian/Ubuntu, RPM for RHEL/Fedora)
- System audio/microphone permissions required

**Native Binaries Bundled:**
- whisper-cpp (OpenAI Whisper in C++) - macOS, Windows, Linux
- sherpa-onnx (ONNX Runtime) - macOS, Windows, Linux for Parakeet transcription
- llama.cpp - macOS, Windows, Linux for local LLM inference
- ffmpeg-static - All platforms
- macos-globe-listener (Swift) - macOS only (compiled during build)
- macos-fast-paste (Swift) - macOS only (compiled during build)
- macos-mic-listener (Swift) - macOS only (event-driven CoreAudio)
- windows-key-listener (C) - Windows only (low-level keyboard hook)
- windows-mic-listener (C) - Windows only (WASAPI session monitoring)
- windows-fast-paste (C) - Windows only (native clipboard simulation)
- linux-fast-paste (C) - Linux only (XTest + Wayland paste simulation)
- nircmd.exe - Windows only (fallback clipboard tool)

---

*Stack analysis: 2026-03-23*
