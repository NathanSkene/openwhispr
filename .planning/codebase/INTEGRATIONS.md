# External Integrations

**Analysis Date:** 2026-03-23

## APIs & External Services

**AI/LLM Providers:**
- OpenAI - Chat completions and Whisper transcription
  - SDK/Client: Native `fetch()` via custom `ReasoningService`
  - Endpoints: `/responses` (Responses API, fallback to `/chat/completions`)
  - Auth: `OPENAI_API_KEY` environment variable
  - Models: GPT-5.2, GPT-5 Mini, GPT-4.1, GPT-4.1 Mini via `modelRegistryData.json`

- Anthropic (Claude) - LLM reasoning and text processing
  - SDK/Client: Custom fetch via IPC handler (no direct CORS)
  - Endpoint: `https://api.anthropic.com/v1/messages`
  - Auth: `ANTHROPIC_API_KEY` environment variable
  - Header: `anthropic-version: 2023-06-01`
  - Models: Claude Opus 4.6, Claude Sonnet 4.6, Claude Haiku 4.5

- Google Gemini - LLM reasoning
  - SDK/Client: Native `fetch()` via `ReasoningService`
  - Endpoint: `https://generativelanguage.googleapis.com/v1beta`
  - Auth: `GEMINI_API_KEY` environment variable
  - Models: Gemini 3.1 Pro, Gemini 3 Flash, Gemini 2.5 Flash Lite

- Groq - Ultra-fast cloud inference
  - SDK/Client: Custom endpoint configuration
  - Endpoint: `https://api.groq.com/openai/v1` (OpenAI-compatible)
  - Auth: `GROQ_API_KEY` environment variable

- Mistral - AI inference
  - SDK/Client: Custom endpoint configuration
  - Endpoint: `https://api.mistral.ai/v1`
  - Auth: `MISTRAL_API_KEY` environment variable

**Transcription Services:**
- OpenAI Whisper API - Cloud transcription (fallback to local whisper.cpp)
  - SDK/Client: Native fetch
  - Endpoint: Configurable via `OPENWHISPR_TRANSCRIPTION_BASE_URL` or defaults to OpenAI `/audio/transcriptions`
  - Auth: `OPENAI_API_KEY`
  - Files: Audio blobs uploaded as multipart form data

- OpenWhispr Cloud API (optional) - Custom transcription backend
  - Endpoint: Configured via `VITE_OPENWHISPR_API_URL`
  - Purpose: Alternative to OpenAI for transcription
  - Handler: `meeting-transcribe-chain` IPC command (file path in `src/helpers/ipcHandlers.js`)

**Calendar Integration:**
- Google Calendar - Meeting detection for auto-pause recording
  - OAuth 2.0 flow via `googleCalendarOAuth.js`
  - Endpoint: `https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events`
  - Auth: OAuth token (user-signed, stored locally)
  - Scope: `calendar.readonly` (read-only calendar access)
  - Purpose: Detect imminent meetings and active calendar events
  - Sync strategy: Exponential backoff (2min → 4min → 8min → cap 30min on failures)
  - Socket timeout: 10 seconds per request

**GitHub Integration:**
- Release Downloads - Multi-binary distribution
  - Endpoints:
    - `https://github.com/OpenWhispr/openwhispr/releases` - App updates
    - `https://github.com/OpenWhispr/whisper.cpp/releases` - Whisper binaries
    - `https://api.github.com/repos/ggerganov/llama.cpp/releases/latest` - llama.cpp releases
  - Auth: Optional `GITHUB_TOKEN` env var for higher rate limits
  - Purpose: Download native binaries (whisper-cpp, llama-server, sherpa-onnx, etc.)

**Update Service:**
- Electron-updater - Automatic app updates from GitHub
  - Provider: GitHub (configured in `electron-builder.json`)
  - Repo: `OpenWhispr/openwhispr`
  - Release type: draft (staging before public release)
  - Used by: `windowManager.js` to check and apply updates

## Data Storage

**Databases:**
- SQLite (better-sqlite3) - Local transcription history
  - Connection: `~/.openwhispr/transcriptions.db` (or app data dir)
  - Client: better-sqlite3 (bundled in ASAR unpacked)
  - Schema: Single `transcriptions` table with columns:
    - `id` (INTEGER PRIMARY KEY)
    - `timestamp` (DATETIME)
    - `original_text` (TEXT)
    - `processed_text` (TEXT)
    - `is_processed` (BOOLEAN)
    - `processing_method` (TEXT)
    - `agent_name` (TEXT)
    - `error` (TEXT)

- Neon PostgreSQL (optional) - Account features and cloud sync
  - Client: `@neondatabase/neon-js` (0.1.0-beta.22)
  - Auth: `@neondatabase/auth` client (via `src/lib/neonAuth.ts`)
  - Purpose: Optional cloud features (user accounts, settings sync)
  - Endpoint: `VITE_NEON_AUTH_URL` environment variable

**File Storage:**
- Local filesystem only for primary functionality
  - Models cached: `~/.cache/openwhispr/whisper-models/`, `parakeet-models/`
  - Settings: `~/Library/Application Support/open-whispr/` (macOS), `%APPDATA%/open-whispr/` (Windows), `~/.config/open-whispr/` (Linux)
  - Audio temporary files: System temp directory (auto-cleaned)

- Vercel Blob (optional) - Cloud file storage
  - SDK: `@vercel/blob` (2.3.0)
  - Purpose: Potential cloud storage for audio files
  - Auth: Vercel API key (not currently exposed in .env.example)

**Caching:**
- None - All caching handled in-memory via Zustand store (`settingsStore.ts`)
- API Key Cache: SecureCache with 1-hour TTL (defined in `ReasoningService`)
- Model availability: CACHE_CONFIG.AVAILABILITY_CHECK_TTL = 30 seconds

## Authentication & Identity

**Auth Provider:**
- Neon Auth (optional) - User authentication for cloud features
  - Implementation: OAuth-based via `@neondatabase/auth`
  - File: `src/lib/neonAuth.ts`
  - Session management: Grace period (60 seconds) for short network failures
  - Storage: localStorage keys `openwhispr:lastSignInTime` and `isSignedIn`
  - Social provider: Google (via Neon Auth)

- Google OAuth 2.0 - Calendar access
  - Implementation: `googleCalendarOAuth.js`
  - Flow: Authorization code flow (desktop callback URL)
  - Scopes: `calendar.readonly`
  - Storage: OAuth token in `@neondatabase/neon-js` session

- Custom API Keys - AI provider authentication
  - Implementation: Environment variables or settings storage
  - Keys stored: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `MISTRAL_API_KEY`
  - Security: Keys sanitized, not logged, 1-hour TTL in memory cache

## Monitoring & Observability

**Error Tracking:**
- None detected - App logs to debug files only

**Logs:**
- Local file logging via `debugLogger.js`
- Output: Platform-specific app data directory (system-specific paths)
- Level: Configurable via `OPENWHISPR_LOG_LEVEL` env var (debug, info, warn, error)
- Reasoning pipeline: Detailed stage-by-stage logging in `logger.logReasoning()`

## CI/CD & Deployment

**Hosting:**
- GitHub Releases - Primary distribution
- electron-updater auto-downloads and installs from GitHub

**CI Pipeline:**
- GitHub Actions workflows (in `.github/workflows/`)
- Workflows: Build (macOS, Windows, Linux), native binary builds, signed distribution
- Triggers: Push to main, release creation

**Code Signing:**
- macOS: Developer certificate (HeroTools Inc.)
  - Identity: `HeroTools Inc. (9R85XFMH59)`
  - Notarization: Enabled (`@electron/notarize`)
  - Entitlements: `resources/mac/entitlements.mac.plist`

- Windows: Optional code signing (via Electron Builder)
  - NSIS installer configuration

- Linux: No code signing (not applicable for AppImage/DEB/RPM)

## Environment Configuration

**Required env vars:**
- `OPENAI_API_KEY` - OpenAI access (for Whisper API and GPT models)

**Optional env vars for AI providers:**
- `ANTHROPIC_API_KEY` - Anthropic Claude access
- `GEMINI_API_KEY` - Google Gemini access
- `GROQ_API_KEY` - Groq ultra-fast inference
- `MISTRAL_API_KEY` - Mistral AI access

**Optional env vars for cloud features:**
- `VITE_NEON_AUTH_URL` - Neon Auth endpoint
- `VITE_OPENWHISPR_API_URL` - OpenWhispr Cloud API endpoint
- `NEON_AUTH_URL` - Server-side Neon Auth URL (main process)
- `GOOGLE_CALENDAR_CLIENT_ID` - Google OAuth client ID
- `GOOGLE_CALENDAR_CLIENT_SECRET` - Google OAuth client secret
- `VITE_OPENWHISPR_OAUTH_CALLBACK_URL` - OAuth callback URL for desktop app

**Configuration env vars:**
- `OPENWHISPR_LOG_LEVEL` - Debug logging level
- `OPENWHISPR_CHANNEL` - Release channel (default: "production")
- `OPENWHISPR_TRANSCRIPTION_BASE_URL` - Custom transcription endpoint
- `WHISPER_BASE_URL` - Alternative transcription endpoint
- `OPENAI_BASE_URL` - Custom OpenAI-compatible endpoint
- `OPENWHISPR_OPENAI_BASE_URL` - OpenWhispr-specific OpenAI endpoint override

**Secrets location:**
- `.env` file (in project root, git-ignored)
- localStorage (browser-only settings, no secrets persisted)
- System keychain (future - not currently used)

## Webhooks & Callbacks

**Incoming:**
- None detected - App is event-driven, not webhook-based

**Outgoing:**
- Google Calendar Auth callback: Desktop OAuth callback URL (`VITE_OPENWHISPR_OAUTH_CALLBACK_URL`)
  - Default: `http://localhost:27149/auth/callback` (port configurable)
  - Used for: Returning OAuth authorization code from Google

## Cross-Origin & CORS Handling

**CORS-Affected APIs:**
- OpenAI, Anthropic, Gemini - Renderer process direct fetch calls
  - Handled via: Custom headers, IPC bridge (Anthropic), CORS configuration

- Anthropic API - Routes through IPC handler in main process to avoid CORS
  - Handler: `anthropic-reasoning` IPC (defined in `ipcHandlers.js`)

**Electron Security Context:**
- Context isolation enabled in preload.js
- IPC bridge: `window.api.*` methods defined in preload.js
- Safe API surface: Minimal IPC exposure to prevent security issues

## API Rate Limiting & Resilience

**Retry Strategy:**
- Exponential backoff: 1s → 2s → 4s → 10s (RETRY_CONFIG.MAX_RETRIES: 3)
- OpenAI endpoint fallback: Tries `/responses` first, falls back to `/chat/completions`
- Google Calendar: Exponential backoff with 30-minute cap on consecutive failures

**Request Timeouts:**
- Google Calendar: 10-second socket timeout per request
- Inference: 30-second timeout (configurable via InferenceConfig)
- API requests: Standard fetch timeout via AbortController

---

*Integration audit: 2026-03-23*
