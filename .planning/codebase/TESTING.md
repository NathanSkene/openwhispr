# Testing Patterns

**Analysis Date:** 2026-03-23

## Test Framework

**Status:** NOT IMPLEMENTED

The OpenWhispr codebase currently has **no automated test framework** installed or configured. There are no test files in the `src/` directory, and no testing dependencies in `package.json`.

**Testing decision:** The project appears to rely on manual testing and integration testing via the built Electron app.

## Recommended Test Setup (If Adding Tests)

Should the project add automated tests, the recommended approach would be:

**Test Framework:** Jest or Vitest (Vitest preferred for Vite integration)

**Test Types:** Three-tier approach recommended
- Unit tests (utility functions, hooks, stores)
- Component tests (React components with react-testing-library)
- E2E tests (Electron app with Spectron or Playwright)

**Configuration:** If implemented, create:
- `vitest.config.ts` or `jest.config.js` at project root
- Test files colocated with source: `*.test.ts`, `*.spec.ts`
- Setup file for environment mocking (`vitest.setup.ts`)

## Current Test Coverage

**Untested areas (entire codebase):**

| Area | Files | Notes |
|------|-------|-------|
| Audio Recording | `src/hooks/useAudioRecording.js`, `src/helpers/audioManager.js` | MediaRecorder API, audio processing |
| Transcription | `src/helpers/audioManager.js` | Local whisper.cpp, cloud APIs (OpenAI, Groq, Mistral) |
| AI Reasoning | `src/services/ReasoningService.ts` | Multi-provider AI processing (OpenAI, Anthropic, Gemini) |
| Hotkey Management | `src/helpers/hotkeyManager.js` | Global hotkey registration, platform-specific (macOS, Windows, Linux) |
| Settings Store | `src/stores/settingsStore.ts` | Zustand store, localStorage persistence |
| Hooks | `src/hooks/use*.ts` (27 custom hooks) | State management, IPC bridge, localStorage |
| Components | `src/components/*.tsx` (41 components) | React UI, especially complex: SettingsPage, OnboardingFlow, ReasoningModelSelector |
| IPC Bridge | `preload.js`, `src/helpers/ipcHandlers.js` | Electron IPC channel definitions and handlers |
| i18n System | `src/i18n.ts`, translation JSON files | Internationalization logic and completeness |
| Utilities | `src/utils/*.ts` (25+ utilities) | Validators, formatters, language support, logger |

## Code Quality Practices

**Manual Testing Checklist (from CLAUDE.md):**

The project defines a comprehensive manual testing checklist:

```
- [ ] Test both local and cloud processing modes
- [ ] Verify hotkey works globally
- [ ] Check clipboard pasting on all platforms
- [ ] Test with different audio input devices
- [ ] Verify whisper.cpp binary detection
- [ ] Test all Whisper models
- [ ] Check agent naming functionality
- [ ] Test custom dictionary with uncommon words
- [ ] Verify Windows Push-to-Talk with compound hotkeys
- [ ] Test GNOME Wayland hotkeys (if on GNOME + Wayland)
- [ ] Test Hyprland Wayland hotkeys (if on Hyprland + Wayland)
- [ ] Verify activation mode selector hidden on GNOME Wayland
- [ ] Verify meeting detection works with event-driven mode
- [ ] Test meeting notification suppression during recording
- [ ] Test post-recording cooldown
```

This checklist should be followed before any release.

## Code Quality Checks

**Available npm scripts:**

```bash
npm run lint              # Run ESLint
npm run format:check      # Verify Prettier formatting
npm run typecheck         # TypeScript type checking
npm run quality-check     # Run format:check + typecheck (recommended)
npm run i18n:check        # Verify all translations are complete
```

**CI/CD Quality Gates:**
- ESLint: all source files pass linting (`.js`, `.ts`, `.tsx`)
- Prettier: all code formatted to 100-char line width
- TypeScript: no type errors (strict: false but requires valid types)
- i18n: all translation keys present in all 10 language files

## Test Infrastructure Needs

**If implementing automated tests, these would be critical:**

### Unit Test Gaps

**Store Testing (`settingsStore.ts`):**
- Store initialization from localStorage
- State mutations (setUseLocalWhisper, setWhisperModel, etc.)
- Settings persistence across app restarts
- Language migration (zh → zh-CN)

**Hook Testing (27 custom hooks):**
- `useSettings` - initialization, IPC sync, dictionary updates
- `useAudioRecording` - MediaRecorder lifecycle
- `useHotkey` - hotkey registration/deregistration
- `useLocalStorage` - serialization/deserialization
- `useAuth` - authentication state management
- `useDialogs` - dialog lifecycle

**Utility Testing:**
- `logger.ts` - log level resolution, fallback to console
- `hotkeyValidator.ts` - hotkey format validation, reserved key detection
- `languageSupport.ts` - language code normalization, model compatibility
- `audioDeviceUtils.ts` - audio device detection and filtering
- `urlUtils.ts` - URL validation, protocol checking

### Component Test Gaps

**Critical Components (High Complexity):**
- `SettingsPage.tsx` (162KB) - 8 settings sections, hundreds of controls
- `OnboardingFlow.tsx` (33KB) - 8-step wizard with state management
- `ReasoningModelSelector.tsx` (36KB) - AI model selection with provider logic
- `TranscriptionModelPicker.tsx` (35KB) - Model picker with download UI
- `ControlPanel.tsx` (27KB) - Main panel layout and routing

**Basic Components (Simpler):**
- `AgentOverlay.tsx` - Agent mode UI
- `ErrorBoundary.tsx` - Error handling wrapper
- `MeetingNotificationOverlay.tsx` - Meeting detection UI
- All `src/components/ui/` components (shadcn/ui wrappers)

### Integration Test Gaps

**Audio Pipeline:**
- Recording → MediaRecorder → whisper.cpp → Clipboard
- Recording → cloud transcription (OpenAI, Groq, Mistral)
- Streaming transcription (Deepgram, AssemblyAI, OpenAI Realtime)

**Settings Sync:**
- localStorage → Zustand → IPC → Main process
- Model selection → Startup pre-warming
- API key changes → Cache invalidation

**IPC Communication:**
- Renderer → Main process (hotkey, file access, etc.)
- Main → Renderer (transcription results, meeting notifications)
- Preload script sandboxing

### E2E Test Gaps (Electron)

**Hotkey Functionality:**
- Global hotkey registration across platforms
- Push-to-talk vs tap-to-talk modes
- Fallback to alternative hotkeys
- GNOME/Hyprland Wayland D-Bus integration

**Meeting Detection:**
- Event-driven meeting app detection (macOS systemPreferences)
- Microphone activity monitoring
- Google Calendar sync with exponential backoff
- Notification suppression during recording

**Audio Features:**
- Multiple microphone selection
- Audio device hot-plugging
- Clipboard pasting on macOS/Windows/Linux
- Recording overlay with waveform visualization

## Common Testing Patterns (If Tests Were Added)

### Hook Testing Pattern

Would follow react-testing-library `renderHook` approach:

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useSettings } from '../src/hooks/useSettings';

describe('useSettings', () => {
  it('initializes settings from storage', async () => {
    const { result } = renderHook(() => useSettings());

    await waitFor(() => {
      expect(result.current.useLocalWhisper).toBeDefined();
    });
  });

  it('persists setting changes to storage', async () => {
    const { result } = renderHook(() => useSettings());

    act(() => {
      result.current.setUseLocalWhisper(true);
    });

    expect(localStorage.getItem('useLocalWhisper')).toBe('true');
  });
});
```

### Component Testing Pattern

Would use react-testing-library with user events:

```typescript
import { render, screen, userEvent } from '@testing-library/react';
import { SettingsPage } from '../src/components/SettingsPage';

describe('SettingsPage', () => {
  it('renders transcription section', () => {
    render(<SettingsPage activeSection="transcription" />);
    expect(screen.getByText(/transcription/i)).toBeInTheDocument();
  });

  it('allows user to toggle local whisper', async () => {
    render(<SettingsPage />);
    const toggle = screen.getByRole('checkbox', { name: /local whisper/i });

    await userEvent.click(toggle);
    expect(toggle).toBeChecked();
  });
});
```

### Mocking IPC Pattern

Would mock `window.electronAPI`:

```typescript
// In test setup (vitest.setup.ts)
global.window.electronAPI = {
  getSettings: vi.fn().mockResolvedValue({ useLocalWhisper: true }),
  saveSettings: vi.fn().mockResolvedValue(void 0),
  onDictionaryUpdated: vi.fn((cb) => {
    // Return unsubscribe function
    return () => {};
  }),
};
```

### Mocking External APIs

Would mock fetch/axios for API tests:

```typescript
vi.mock('../src/services/ReasoningService', () => ({
  default: {
    process: vi.fn().mockResolvedValue('Processed text'),
  },
}));

describe('Audio transcription with OpenAI', () => {
  it('sends audio to OpenAI and returns text', async () => {
    const result = await transcribeWithOpenAI(audioBuffer);
    expect(result).toBe('Transcribed text');
  });
});
```

## Type Safety

**TypeScript Configuration:**
- Strict mode: disabled (`strict: false`) to allow `any` types when necessary
- Allow JS: enabled (mixing JS and TS acceptable)
- Check JS: disabled (don't enforce types in plain JS files)

**Type Coverage:**
- All React components have TypeScript props interfaces
- Zustand stores fully typed with `interface StateType`
- Services and utilities have return type annotations
- No explicit `any` types without `// @ts-ignore` comment

**Type Checking:**
```bash
npm run typecheck          # Check all TS without emitting
cd src && tsc --noEmit     # Explicit check command
```

## Code Quality Enforcement

**Pre-commit Hooks:** Not currently configured (relies on CI)

**CI Workflows (`.github/workflows/`):**
- Linting and formatting checks
- TypeScript compilation
- i18n completeness checks
- Build verification

**Manual verification before release:**
```bash
npm run quality-check      # format:check + typecheck
npm run i18n:check         # Translation completeness
npm run build              # Full build verification
```

## Areas Requiring Manual Testing

**Platform-Specific Features:**

| Feature | macOS | Windows | Linux |
|---------|-------|---------|-------|
| Global hotkeys | systemPreferences | windows-key-listener.exe | XTest binary / D-Bus |
| Clipboard paste | AppleScript | PowerShell SendKeys | xdotool/wtype/ydotool |
| Microphone detection | CoreAudio listener | WASAPI listener | pactl subscribe |
| Wayland support | N/A | N/A | GNOME/Hyprland/Sway |
| Push-to-talk | Not supported | Native support | Not supported |

**Multi-Provider Testing:**

| Provider | Types | Coverage |
|----------|-------|----------|
| Transcription | OpenAI, Groq, Mistral, custom | Streaming (Deepgram, AssemblyAI), Realtime (OpenAI) |
| Reasoning | OpenAI, Anthropic, Google Gemini, local | GPT-5, Claude Sonnet/Haiku/Opus, Gemini Pro/Flash |
| Meeting detection | macOS (process), Windows/Linux (process), Microphone (all), Calendar (all) | 4 independent sources |

## Risk Areas (Should Have Tests)

**High-Risk Components:** Recommend priority for test coverage if framework is added

| Component | Risk | Reason |
|-----------|------|--------|
| `audioManager.js` (400+ LOC) | High | Complex audio pipeline with multiple providers |
| `SettingsPage.tsx` (162KB!) | High | Largest component, many interdependent settings |
| `hotkeyManager.js` | High | Platform-specific code, silent failures possible |
| `ipcHandlers.js` | High | Security boundary, data bridge between processes |
| `ReasoningService.ts` | High | Multi-provider AI integration, complex logic |
| `settingsStore.ts` | Medium | Central state, persistence critical |
| `useSettings.ts` | Medium | Initialization and sync logic |
| IPC preload bridge | Critical | Security-sensitive code |

## Technical Debt

**Testing-related issues:**
- No test infrastructure (framework, runners, fixtures)
- No CI/CD test automation
- 100+ untested source files
- Platform-specific code untestable without physical hardware
- Electron window management hard to test without E2E framework

**Mitigation:**
- Continue manual testing checklist before releases
- Increase test coverage incrementally
- Use Vitest for new feature development
- Consider Playwright for E2E testing of Electron app

---

*Testing analysis: 2026-03-23*
