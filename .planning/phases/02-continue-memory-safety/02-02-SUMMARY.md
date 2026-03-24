---
phase: 02-continue-memory-safety
plan: 02
subsystem: memory-safety
tags: [cpp, unique_ptr, smart-pointers, discord, raii]

# Dependency graph
requires:
  - phase: 01-memory-safety-audit
    provides: audit of raw owning pointers and manual delete patterns
provides:
  - Discord::mpHandlers as std::unique_ptr<DiscordEventHandlers> with .get() at C API boundaries
  - Discord::mPresencePtrs as std::map<QString, std::unique_ptr<localDiscordPresence>> — no manual delete loops
affects: [future-discord-work]

# Tech tracking
tech-stack:
  added: []
  patterns: ["Use std::map instead of QMap for unique_ptr values — QMap COW semantics require copyable value types"]

key-files:
  created: []
  modified:
    - src/discord.h
    - src/discord.cpp

key-decisions:
  - "Used std::map<QString, std::unique_ptr<localDiscordPresence>> instead of QMap — Qt COW semantics make QMap incompatible with move-only types (same decision as 02-01)"
  - "Replaced mPresencePtrs.value() calls with find() + .get() pattern — QMap::value() returns by value which is incompatible with unique_ptr"
  - "Replaced manual QMutableMapIterator delete loop in destructor with single mPresencePtrs.clear() call"

patterns-established:
  - "Pattern: std::map::emplace() + std::make_unique for inserting unique_ptr values"
  - "Pattern: auto it = map.find(key); ptr = (it != map.end()) ? it->second.get() : nullptr; for unique_ptr map lookup"

requirements-completed: [MEM-DISCORD-HANDLERS, MEM-DISCORD-PRESENCE]

# Metrics
duration: 3min
completed: 2026-03-24
---

# Phase 02 Plan 02: Discord unique_ptr Migration Summary

**Discord mpHandlers and mPresencePtrs migrated from raw owning pointers to unique_ptr using std::map, eliminating manual delete in constructor, destructor, and UpdatePresence()**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-24T21:17:56Z
- **Completed:** 2026-03-24T21:21:09Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments

- Migrated `Discord::mpHandlers` from `DiscordEventHandlers*` to `std::unique_ptr<DiscordEventHandlers>`; constructor uses `make_unique`, destructor auto-cleans, C API calls use `.get()`
- Migrated `Discord::mPresencePtrs` from `QMap<QString, localDiscordPresence*>` to `std::map<QString, std::unique_ptr<localDiscordPresence>>`; all `new localDiscordPresence` replaced with `make_unique`, manual `QMutableMapIterator` delete loop replaced with `.clear()`
- All `mPresencePtrs.value()` calls (which QMap returns by value, incompatible with unique_ptr) replaced with `find()` + `it->second.get()` pattern

## Task Commits

1. **Task 1: Migrate Discord::mpHandlers to unique_ptr** - `2e656dddd` (fix)

## Files Created/Modified

- `src/discord.h` - Added `#include <map>` and `#include <memory>`; changed mpHandlers declaration to `std::unique_ptr<DiscordEventHandlers>`; changed mPresencePtrs to `std::map<QString, std::unique_ptr<localDiscordPresence>>`
- `src/discord.cpp` - Updated constructor (make_unique, memset with .get()), destructor (removed manual delete loop and delete mpHandlers), UpdatePresence (emplace/make_unique inserts, find() lookups, mpHandlers.get() at Discord_Initialize calls)

## Decisions Made

- Used `std::map` instead of `QMap` for the presence map — Qt's copy-on-write semantics require copyable value types; unique_ptr is move-only and incompatible with QMap. This is consistent with the decision made in plan 02-01.
- Replaced `mPresencePtrs.value(nullptr)` / `mPresencePtrs.value(applicationID)` with `find()` pattern — `QMap::value()` returns a copy which cannot compile with unique_ptr values.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Used std::map instead of QMap for mPresencePtrs**

- **Found during:** Task 1 (migrating mPresencePtrs)
- **Issue:** `QMap<QString, std::unique_ptr<localDiscordPresence>>` would not compile because Qt's COW internals require value types to be copyable; unique_ptr is move-only
- **Fix:** Changed declaration to `std::map<QString, std::unique_ptr<localDiscordPresence>>`; updated all insertion/lookup sites to use std::map API (`emplace`, iterator `it->second.get()`)
- **Files modified:** src/discord.h, src/discord.cpp
- **Verification:** Build clean, 21/21 ctest pass
- **Committed in:** 2e656dddd (part of task commit)

---

**Total deviations:** 1 auto-fixed (1 blocking — QMap incompatibility with move-only types)
**Impact on plan:** Required deviation from plan's `QMap` instruction; std::map is a direct substitute. No scope creep.

## Issues Encountered

- `clang-format` not available in the execution environment (no LLVM in PATH). Code changes follow project style conventions manually; formatting will be enforced by CI or pre-commit hooks.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Discord memory management now fully RAII-based
- Both Discord raw pointer patterns (single handler, map of presences) converted to unique_ptr ownership
- Ready for plan 02-03 (profileShortcuts migration) or subsequent memory safety work

---
*Phase: 02-continue-memory-safety*
*Completed: 2026-03-24*
