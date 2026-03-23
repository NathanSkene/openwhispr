# Roadmap: OpenWhispr Fork Consolidation

**Created:** 2026-03-23
**Granularity:** Coarse (3 phases)
**Coverage:** 19/19 requirements mapped

---

## Phases

- [ ] **Phase 1: Upstream Sync** - Merge 17 upstream commits, resolve conflicts, push fork up to date
- [ ] **Phase 2: Untangle and Fix** - Extract three WIP features into isolated branches, fix recording overlay bug and TypeScript errors
- [ ] **Phase 3: Validate and Submit** - Typecheck/lint/build each branch, confirm features work, submit 5 upstream PRs, clean up

---

## Phase Details

### Phase 1: Upstream Sync
**Goal**: The fork is current with upstream and Nathan's origin reflects local main
**Depends on**: Nothing (first phase)
**Requirements**: SYNC-01, SYNC-02
**Success Criteria** (what must be TRUE):
  1. `git log upstream/main` shows 0 commits ahead of local main (all 17 merged, conflicts resolved)
  2. `git push origin main` succeeds — fork on GitHub matches local main
  3. Installed app `/Applications/OpenWhispr Dev.app` still builds and functions after merge
**Plans:** 1 plan
Plans:
- [ ] 01-01-PLAN.md — Stash WIP, merge 17 upstream commits, push to origin, build and verify app

---

### Phase 2: Untangle and Fix
**Goal**: All WIP is off main — each feature lives on its own branch — and the recording overlay works correctly without bugs
**Depends on**: Phase 1
**Requirements**: BUG-01, BUG-02, ISO-01, ISO-02, ISO-03, ISO-04
**Success Criteria** (what must be TRUE):
  1. Main branch has zero uncommitted changes (all WIP extracted)
  2. Three feature branches exist: VAD gap compression, max dictation duration, recording overlay refinements — each with clean, focused commits
  3. Recording overlay appears during active recording (not just processing); the dictation panel blue indicator does not appear simultaneously
  4. `npm run typecheck` on main branch returns 0 errors
  5. PR #478 (cleanup prompt fix) reconciled — no conflicting changes across WIP and existing PR
**Plans**: TBD

---

### Phase 3: Validate and Submit
**Goal**: Every feature is verified working, passes CI checks, and has a submitted upstream PR
**Depends on**: Phase 2
**Requirements**: VAL-01, VAL-02, VAL-03, VAL-04, PR-01, PR-02, PR-03, PR-04, PR-05, CLN-01, CLN-02
**Success Criteria** (what must be TRUE):
  1. Each of the 5 feature branches runs `npm run typecheck` (0 errors) and `npm run lint` (0 errors)
  2. Each feature branch builds successfully via `npm run build:dev`
  3. Recording overlay + waveform, BYOK chunking, VAD compression, and max dictation duration all confirmed working in the built app
  4. Five PRs open on upstream repo: recording overlay, BYOK chunking, VAD, max duration, overlay refinements
  5. Stale merged branches deleted from origin; main branch has no uncommitted WIP
**Plans**: TBD

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Upstream Sync | 0/1 | Not started | - |
| 2. Untangle and Fix | 0/? | Not started | - |
| 3. Validate and Submit | 0/? | Not started | - |
