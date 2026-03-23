# OpenWhispr Fork Consolidation

## What This Is

Consolidating Nathan's OpenWhispr fork: untangling interleaved features from uncommitted WIP on main, fixing bugs (recording overlay not showing during recording, duplicate indicators), syncing with upstream, and submitting each feature as a clean PR to the upstream repo.

## Core Value

Every feature is validated, isolated on its own branch, and submitted as a clean upstream PR — with the installed app working correctly throughout.

## Requirements

### Validated

- ✓ Recording overlay with real-time audio waveform — existing (commit 02c236e)
- ✓ Client-side chunking for BYOK large file transcription — existing (commit 85aef93)
- ✓ Bulk dictionary import via paste/file upload — existing (PR #461 open)
- ✓ Cleanup prompt fix for command-like transcriptions — existing (PR #478 open)
- ✓ Dev build infrastructure (build:dev, electron-builder.dev.json) — existing (working)

### Active

- [ ] Sync with upstream (17 commits behind)
- [ ] Fix recording overlay bug: overlay not showing during active recording, only during processing
- [ ] Fix duplicate recording indicator: "blue thing" (dictation panel) appears alongside the centered recording overlay — only one should show
- [ ] Untangle uncommitted WIP into isolated feature branches (VAD, max duration, overlay refinements)
- [ ] Fix TypeScript errors (11 errors from incomplete IPC/preload bindings)
- [ ] Validate each feature independently (typecheck, lint, build)
- [ ] Submit clean PRs to upstream for: recording overlay, BYOK chunking, VAD compression, max dictation duration, overlay refinements
- [ ] Reconcile prompt hardening WIP with existing PR #478
- [ ] Clean up stale merged branches (feat/recording-overlay, feature/byok-client-side-chunking)
- [ ] Push origin/main to match local main

### Out of Scope

- New features not already in progress — this is consolidation only
- Upstream features or other contributors' PRs — not our concern
- Refactoring OpenWhispr codebase beyond what's needed for our PRs
- Meeting detection or calendar features — not touched in this work

## Context

### Repository State
- **Origin** (Nathan's fork): `git@github.com:NathanSkene/openwhispr.git`
- **Upstream**: `https://github.com/OpenWhispr/openwhispr.git`
- Local main: 4 commits ahead of origin/main, 17 behind upstream/main
- 16 modified files + 2 new files uncommitted on main
- 2 open PRs: #461 (bulk dictionary), #478 (cleanup prompt fix)

### Uncommitted WIP Details

Three features interleaved in uncommitted changes:

**VAD Gap Compression** — `vadProcessor.js` (new), `audioManager.js`, `settingsStore.ts`, `useSettings.ts`, `SettingsPage.tsx`
- Energy-based silence detection, compresses gaps >1.5s to 500ms max
- Prevents Whisper cloud hallucinations ("Thank you for watching!")
- Only applies to cloud transcription, not local whisper

**Max Dictation Duration** — `settingsStore.ts`, `useSettings.ts`, `SettingsPage.tsx`, `DictationWidget.tsx`, `translation.json`
- Auto-stop at configurable limit (default 5 min)
- Amber warning at 60s before limit
- Options: off, 2, 5, 10, 15, 30 min

**Recording Overlay Refinements** — `RecordingOverlay.css`, `windowConfig.js`, `windowManager.js`, `index.css`
- Shrunk from 320x64 to 160x32, repositioned to bottom-center
- Multi-monitor cursor tracking
- Dark theme in light mode for consistency

### Recording Indicator Bug
Nathan reports: the centered recording overlay (the "nice thing we built") only appears during processing, not during active recording. The dictation panel ("blue thing in bottom right") still appears. The overlay show/hide logic in `windowManager.js` looks correct (lines 425-428 toggle on `sendToggleDictation`), so the bug may be in timing or the `_isDictatingToggle` state management.

### Quality Status
- typecheck: 11 errors (missing preload.js IPC bindings, missing `onAudioLevel` prop, store type gaps)
- lint: 0 errors, 24 warnings (pre-existing)
- Build: last build produced working `/Applications/OpenWhispr Dev.app`

### Build Workflow
Nathan uses `/openwhispr build` skill: kill app → `npm run build:dev` → codesign → install to `/Applications/OpenWhispr Dev.app`. Settings persist in `~/Library/Application Support/open-whispr/`.

## Constraints

- **Node version**: Must use Node 22 (pinned in `.nvmrc`) — do NOT regenerate package-lock.json
- **Daily driver**: Nathan uses this app daily — builds must be stable
- **Upstream compatibility**: PRs must be clean, independent, and pass upstream CI
- **IPC contract**: Any new renderer features need preload.js + ipcHandlers.js updates
- **i18n**: All UI strings must use translation keys in all 9 language files
- **No npm start**: Use `/openwhispr build` or `npm run dev` only

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Separate features into individual PRs | Upstream maintainers prefer focused PRs | — Pending |
| Sync upstream before branching | Clean base for all feature branches | — Pending |
| Fix bugs before submitting overlay PR | Don't PR broken features | — Pending |
| Keep dev build infrastructure local | build:dev/electron-builder.dev.json is fork-specific | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-23 after initialization*
