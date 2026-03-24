---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: Milestone complete
last_updated: "2026-03-24T21:36:34.764Z"
progress:
  total_phases: 2
  completed_phases: 2
  total_plans: 5
  completed_plans: 5
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-24)

**Core value:** Every heap allocation owned by Qt parent-child or smart pointer
**Current focus:** Phase 02 — continue-memory-safety

## Current Status

- Phase 0 (cherry-picks): Complete ✓
- Phase 1 (audit & fixes): Complete ✓ (all 2 plans executed, 21/21 tests pass)
- Phase 2: In Progress — 02-01 complete (Host mMMCPServer/mStopWatchMap/profileShortcuts → unique_ptr), 02-02 complete (Discord mpHandlers/mPresencePtrs → unique_ptr + std::map), 02-03 complete (VarUnit::base + Updater::feed → unique_ptr)

## Decisions

- Use `.get()` at TArea/TRoom construction sites rather than changing constructors — callers are non-owning
- Use `std::move` in readJsonMapFile swap pattern — eliminates separate `delete` of old TRoomDB
- [Phase 02-continue-memory-safety]: Use std::map instead of QMap for unique_ptr values — Qt COW requires copyable values, making QMap incompatible with move-only types
- [Phase 02-continue-memory-safety]: Use std::map instead of QMap for unique_ptr values — Qt COW requires copyable values, making QMap incompatible with move-only types (confirmed in Discord migration)
- [Phase 02-continue-memory-safety]: Use base.reset(pVariable) in setBase() rather than assignment operator — explicitly shows ownership transfer

## Session Notes

- Branch: fix-memory-leaks
- Upstream leak fix commits cherry-picked successfully (6 commits)
- Build verified clean after cherry-picks
- Codebase map available in .planning/codebase/
- Phase 1 complete: 01-01 (Hunspell_destroy, TMap::mpRoomDB→unique_ptr) + 01-02 (TTrigger unique_ptr migrations)
- ctest: 21/21 tests passed (2026-03-25)
- Phase 2 plan 02-01 complete: mMMCPServer delete removed, mStopWatchMap + profileShortcuts → std::map + unique_ptr (2026-03-24)
- Phase 2 plan 02-02 complete: Discord mpHandlers → unique_ptr, mPresencePtrs → std::map + unique_ptr (2026-03-24)
- Phase 2 plan 02-03 complete: VarUnit::base → unique_ptr, Updater::feed → unique_ptr, feed.get() at all Qt API boundaries (2026-03-25)

## Accumulated Context

### Roadmap Evolution

- Phase 2 added: Continue memory safety

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260416-gx1 | resolve the conflicts and complete the rebase | 2026-04-16 | 84e60da50 | [260416-gx1-resolve-the-conflicts-and-complete-the-r](./quick/260416-gx1-resolve-the-conflicts-and-complete-the-r/) |

---
*Last updated: 2026-04-16 — Completed quick task 260416-gx1: resolve the conflicts and complete the rebase*
