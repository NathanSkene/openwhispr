# Coding Conventions

**Analysis Date:** 2026-03-23

## Naming Patterns

**Files:**
- React components (TSX): PascalCase with .tsx extension (e.g., `SettingsPage.tsx`, `OnboardingFlow.tsx`)
- Utilities/helpers (TS): camelCase with .ts extension (e.g., `logger.ts`, `hotkeyValidator.ts`)
- Hooks (TS): camelCase with `use` prefix (e.g., `useSettings.ts`, `useHotkey.ts`)
- Stores (TS): camelCase ending with `Store` (e.g., `settingsStore.ts`, `noteStore.ts`)
- Main process files (JS): camelCase or descriptive (e.g., `main.js`, `ipcHandlers.js`)
- Config files (TS/JSON): camelCase or UPPER_CASE for constants (e.g., `InferenceConfig.ts`, `constants.ts`)

**Functions:**
- camelCase for all functions: `getLogLevel()`, `normalizeHotkey()`, `processAudioBuffer()`
- Prefixes for common patterns:
  - Getters: `get*` (e.g., `getRecordingErrorTitle()`)
  - Setters: `set*` (e.g., `setUiLanguage()`)
  - Validators: `validate*` or `is*` (e.g., `validateLanguageForModel()`, `isValidApiKey()`)
  - Handlers: `on*` or `handle*` (e.g., `onDictationRealtimePartial`, `handleCopyTranscription()`)

**Variables:**
- camelCase: `useLocalWhisper`, `whisperModel`, `mediaRecorder`
- Constants: UPPER_SNAKE_CASE at module level (e.g., `SHORT_CLIP_DURATION_SECONDS`, `PLACEHOLDER_KEYS`)
- Boolean prefixes: `is*`, `has*`, `use*` (e.g., `isRecording`, `hasInitialized`, `useLocalWhisper`)
- Arrays: plural form (e.g., `audioChunks`, `customDictionary`, `gcalAccounts`)

**Types/Interfaces:**
- PascalCase (e.g., `TranscriptionSettings`, `HotkeySettings`, `LocalTranscriptionProvider`)
- Suffix with `Settings` for settings interfaces
- Suffix with `Props` for component prop interfaces
- Suffix with `State` for state objects
- Suffix with `Error` for error types

## Code Style

**Formatting:**
- Prettier v3.4.2 configured in `.prettierrc`
- Semi-colons: enabled (semi: true)
- Single quotes: disabled (singleQuote: false) — use double quotes
- Tab width: 2 spaces
- Print width: 100 characters
- Trailing comma: ES5 (includes objects/arrays, but not parameters)
- JSX single quote: disabled — use double quotes
- Arrow function parens: always (e.g., `(x) => x` not `x => x`)
- Bracket spacing: enabled

**Command to format:**
```bash
npm run format:js          # eslint fix + prettier
npm run format:check       # verify formatting
```

**Linting:**
- ESLint 9.25.0 with separate configs for main process and renderer
- Main process (`.js` in root): `eslint.config.js` — CommonJS, Node globals, relaxed rules
- Renderer (`.ts/.tsx` in src): `src/eslint.config.js` — ES modules, browser globals, React rules

**Key ESLint rules:**
- `no-unused-vars`: warn (ignores pattern-prefixed: `_`, uppercase, `event`, `err`, `error`)
- `no-console`: off (allowed in main and renderer)
- `no-empty`: error except for empty catch blocks
- `no-constant-condition`: error but allows loops
- `no-control-regex`: off
- `no-useless-catch`: off
- `react-refresh/only-export-components`: warn (allows constants alongside components)
- React hooks rules enabled via `eslint-plugin-react-hooks`

**TypeScript:**
- Target: ES2022
- Module: ESNext
- JSX: react-jsx (automatic JSX transform)
- Strict mode: disabled (`strict: false`)
- Module resolution: bundler (for Vite)
- Path alias: `@/*` resolves to src directory
- Check JS: disabled
- Isolated modules: enabled

## Import Organization

**Order:**
1. React and framework imports
2. Third-party library imports (Radix, Lucide, Zustand, etc.)
3. Internal type imports (from `types/`)
4. Internal hook imports (from `hooks/`)
5. Internal store imports (from `stores/`)
6. Internal component/service imports
7. Internal utility imports (from `utils/`, `lib/`, `config/`)
8. CSS/style imports

**Example from `SettingsPage.tsx`:**
```typescript
// 1. React framework
import React, { useState, useCallback, useEffect, useRef } from "react";
import { useTranslation } from "react-i18next";

// 2. Third-party UI
import { Button } from "./ui/button";
import { Input } from "./ui/input";
import { Badge } from "./ui/badge";
import { RefreshCw, Download, Mic, Shield } from "lucide-react";

// 3. Internal types
import type { LocalTranscriptionProvider } from "../types/electron";

// 4. Internal hooks
import { useAuth } from "../hooks/useAuth";
import { useSettings } from "../hooks/useSettings";
import { useDialogs } from "../hooks/useDialogs";

// 5. Internal stores
import { useSettingsStore } from "../stores/settingsStore";

// 6. Internal components
import MicPermissionWarning from "./ui/MicPermissionWarning";
import TranscriptionModelPicker from "./TranscriptionModelPicker";

// 7. Internal utilities
import { useAgentName } from "../utils/agentName";
import logger from "../utils/logger";
```

**Path aliases:**
- `@/*` maps to src directory (defined in `src/tsconfig.json`)
- Generally imports use relative paths for clarity

## Error Handling

**Patterns:**
- Use try-catch with typed errors (Error objects with message property)
- Always log errors with context using `logger.error()` or `logger.warn()`
- For async operations, chain `.catch()` with error logging
- User-facing errors: use `toast()` UI notifications
- Network errors: validate response and handle gracefully
- Fallback mechanisms for critical features (e.g., whisper.cpp → OpenAI, native hotkey → polling)

**Example from logger.ts:**
```typescript
try {
  const level = normalizeLevel(await window.electronAPI.getLogLevel());
  if (level) {
    cachedLevel = level;
    return level;
  }
} catch {
  // Fall back to default level
}
cachedLevel = defaultLevel;
return cachedLevel;
```

**Example from audioManager.js:**
```typescript
.catch((err) =>
  logger.warn(
    "Failed to initialize settings store",
    { error: (err as Error).message },
    "settings"
  )
);
```

**IPC error handling:**
- Main process handlers wrap Electron IPC calls in try-catch
- Errors returned as `{ error: string }` or thrown as Error objects
- Renderer catches and logs errors before showing UI

**Validation errors:**
- Return error codes and messages (not exceptions) for user input validation
- Example from `hotkeyValidator.ts`: returns `{ isValid: boolean, errorCode?: ValidationErrorCode }`

## Logging

**Framework:** Custom `logger` object (file: `src/utils/logger.ts`)

**Log levels:** trace, debug, info, warn, error, fatal

**Patterns:**
- `logger.debug(message, metadata?, scope)` for development/diagnostic info
- `logger.info(message, metadata?, scope)` for general information
- `logger.warn(message, metadata?, scope)` for non-critical issues
- `logger.error(message, metadata?, scope)` for failures
- `logger.logReasoning(stage, details)` for AI reasoning pipeline debugging

**Scope parameter:** Optional string identifying log source (e.g., `"settings"`, `"reasoning"`, `"meeting-detection"`)

**Metadata object:** Include structured data (e.g., `{ error: err.message, provider: "openai" }`)

**IPC bridge:** Logger sends to main process when available, falls back to console

**Default level:** info (debug logs hidden unless `OPENWHISPR_LOG_LEVEL=debug` or `--log-level=debug`)

## Comments

**When to comment:**
- Complex algorithms or non-obvious logic (e.g., platform-specific hotkey format conversions)
- Workarounds or hacks with explanation of why they exist
- Disabled code with reason for disablement (prefer deletion over commented code)
- Important integration points (e.g., IPC channel documentation)
- TypeScript type narrowing or casting rationale

**JSDoc/TSDoc:**
- Required for: exported functions, hook signatures, service methods
- Optional for: internal utilities or well-named functions
- Format: Use `/** ... */` block style above definition
- Include `@param`, `@returns`, `@throws` where relevant

**Example:**
```typescript
/**
 * Normalize UI language code with migration support.
 * @param lang Language code (e.g., "en", "zh-CN")
 * @returns Normalized language code (e.g., "zh" migrated to "zh-CN")
 */
export const normalizeUiLanguage = (lang: string): string => {
  // ...
};
```

## Function Design

**Size:**
- Target: 40-80 lines per function (including comments)
- Maximum: 150 lines (indicates refactoring candidate)
- Smaller functions preferred for hooks and handlers

**Parameters:**
- Maximum 3 positional parameters (use object destructuring for >3)
- Use TypeScript interfaces for complex parameter shapes
- Prefer named parameters via object destructuring: `function process({ input, options }) {}`

**Return values:**
- Single return type per function (avoid overloaded returns)
- Use union types for multiple valid returns: `() => string | null`
- For errors, return error tuple: `[value, error]` OR throw exception
- Async functions always return Promise<T>

**Side effects:**
- Minimize in utility functions (pure functions preferred)
- Clearly document side effects in JSDoc
- Use hooks/stores for component state management, not closures

## Module Design

**Exports:**
- Named exports preferred for all exports (aids tree-shaking and refactoring)
- Default exports only for React components (convention in src)
- Barrel files use named re-exports: `export { FeatureA } from "./a"`

**Barrel files:**
- Use when grouping related utilities or components
- Location: `src/components/ui/` has barrel re-exports
- Keep barrel files simple (just re-exports, no logic)

**File organization:**
- One main export per file (e.g., `AudioManager` class in `audioManager.js`)
- Related utilities in same file if <200 LOC
- Move to separate file if growth exceeds 200 LOC

## Store Patterns (Zustand)

**Location:** `src/stores/*.ts`

**Structure:**
- Create interface for state: `interface SettingsState { ... }`
- Include both state properties and action methods in same interface
- Use `create<StateType>()` for type safety
- Initialize store with state object + action methods

**Example from settingsStore.ts:**
```typescript
export interface SettingsState
  extends TranscriptionSettings,
    ReasoningSettings,
    HotkeySettings {
  setUseLocalWhisper: (value: boolean) => void;
  setWhisperModel: (value: string) => void;
  // ...
}
```

**Actions:**
- Prefix with `set` for state mutations
- Prefix with `add`/`remove` for collection operations
- Return void (mutations are in-place)

## Hook Patterns

**Location:** `src/hooks/use*.ts`

**Naming:** Always start with `use` prefix

**Structure:**
- Custom hooks wrap Zustand stores or provide computed state
- Include setup effects (initialization, cleanup)
- Always return hooks from functional components

**Example patterns:**
- `useSettings()` - wraps settings store + initializes from IPC
- `useHotkey()` - manages hotkey registration and state
- `useLocalStorage()` - localStorage wrapper with serialization
- `useAuth()` - authentication state and methods

## React Component Patterns

**Functional components only** (no class components)

**File naming:**
- PascalCase for component files: `SettingsPage.tsx`
- Exported component name matches filename

**Props interface:**
- Define interface at top of file: `interface SettingsPageProps { ... }`
- Suffix with `Props`
- Use destructuring in component signature

**Example:**
```typescript
interface SettingsPageProps {
  activeSection?: SettingsSectionType;
}

export default function SettingsPage({ activeSection }: SettingsPageProps) {
  // ...
}
```

**Hook usage order (Rules of Hooks):**
1. State hooks (useState)
2. Callback hooks (useCallback)
3. Effect hooks (useEffect)
4. Custom hooks
5. Conditional hooks in sub-functions only

**Styling:**
- Tailwind CSS classes in `className` prop
- Use Tailwind utilities from config (src/index.css)
- Component-specific CSS in `.tsx` adjacent `.css` file only if needed
- Example: `RecordingOverlay.tsx` + `RecordingOverlay.css`

## Constants and Configuration

**Location:**
- `src/config/constants.ts` for app-wide constants
- `src/config/InferenceConfig.ts` for AI model configuration
- `src/models/modelRegistryData.json` for all AI models (single source of truth)

**Naming:** UPPER_SNAKE_CASE with clear scope prefix

**Example from audioManager.js:**
```javascript
const SHORT_CLIP_DURATION_SECONDS = 2.5;
const REASONING_CACHE_TTL = 30000; // 30 seconds
const REALTIME_MODELS = new Set(["gpt-4o-mini-transcribe", "gpt-4o-transcribe"]);
const PLACEHOLDER_KEYS = {
  openai: "your_openai_api_key_here",
  groq: "your_groq_api_key_here",
};
```

## Internationalization (i18n)

**Framework:** react-i18next v15 with i18next v25

**Translation files:** `src/locales/{lang}/translation.json`

**Supported languages:** en, es, fr, de, pt, it, ru, ja, zh-CN, zh-TW (10 total)

**Usage in components:**
```typescript
import { useTranslation } from "react-i18next";

const { t } = useTranslation();
// Simple key
t("notes.list.title")
// With interpolation
t("notes.upload.using", { model: "Whisper" })
```

**Key organization:** Hierarchical by feature area (e.g., `notes.editor.*`, `referral.toasts.*`)

**NEVER translate:**
- Brand names (OpenWhispr, Pro)
- Technical terms (Markdown, Signal ID)
- Format names (MP3, WAV)
- AI system prompts

**Translation requirements:**
- Every new UI string must have key in ALL 10 language files
- Use `useTranslation()` hook in components, never hardcode UI text
- Use `{{variable}}` syntax for interpolation

## Internationalization Implementation

**Files:**
- `src/i18n.ts` - i18next configuration and language utilities
- `src/locales/` - translation JSON files organized by language
- `scripts/check-i18n.js` - CI check for missing translations

**Verification:**
```bash
npm run i18n:check  # Verify all translations complete
```

---

*Convention analysis: 2026-03-23*
