---
phase: 02-continue-memory-safety
plan: 03
subsystem: memory
tags: [unique_ptr, raii, cpp20, VarUnit, Updater, dblsqd]

# Dependency graph
requires:
  - phase: 02-continue-memory-safety
    provides: unique_ptr migration pattern established in 02-01 and 02-02
provides:
  - std::unique_ptr<TVar> ownership for VarUnit::base
  - std::unique_ptr<dblsqd::Feed> ownership for Updater::feed
  - No manual delete in VarUnit or Updater destructors
affects: [future memory safety work, any code interacting with VarUnit or Updater]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "unique_ptr for non-Qt heap allocations in header declarations"
    - "= default destructor when unique_ptr members handle cleanup"
    - "feed.get() at API boundaries (Qt connect, constructors requiring raw pointer)"

key-files:
  created: []
  modified:
    - src/VarUnit.h
    - src/VarUnit.cpp
    - src/updater.h
    - src/updater.cpp

key-decisions:
  - "Use base.reset(pVariable) in setBase() rather than assignment operator — explicitly shows ownership transfer"
  - "dblsqd::Feed has no QObject parent parameter, so unique_ptr is correct (confirmed by checking 3rdparty/dblsqd/dblsqd/feed.h)"
  - "Default destructor (= default) used when unique_ptr handles all cleanup — avoids empty destructor bodies"

patterns-established:
  - "Pattern: pointer comparisons with unique_ptr require .get() — e.g. var == base.get()"
  - "Pattern: Qt connect() and KDToolBox::connectSingleShot() require .get() for unique_ptr senders/receivers"

requirements-completed: [MEM-VARUNIT-BASE, MEM-UPDATER-FEED]

# Metrics
duration: 7min
completed: 2026-03-25
---

# Phase 02 Plan 03: VarUnit::base and Updater::feed unique_ptr Migration Summary

**VarUnit::base and Updater::feed migrated to unique_ptr — manual delete removed from both destructors, feed.get() used at all Qt API boundaries**

## Performance

- **Duration:** 7 min
- **Started:** 2026-03-25T21:23:23Z
- **Completed:** 2026-03-25T21:30:06Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- VarUnit::base changed from raw owning TVar* to std::unique_ptr<TVar> — destructor and clear() no longer need manual delete
- Updater::feed changed from raw owning dblsqd::Feed* to std::unique_ptr<dblsqd::Feed> — destructor no longer calls delete(feed)
- All 14 Qt connect/KDToolBox::connectSingleShot calls in updater.cpp updated to use feed.get()
- All 3 dblsqd::UpdateDialog construction sites updated to use feed.get()
- 21/21 ctest tests pass after both changes

## Task Commits

Each task was committed atomically:

1. **Task 1: Migrate VarUnit::base to unique_ptr** - `e57618f49` (fix)
2. **Task 2: Migrate Updater::feed to unique_ptr** - `cbd0ab310` (fix)

**Plan metadata:** (docs commit below)

## Files Created/Modified

- `src/VarUnit.h` - Added #include <memory>, changed TVar* base to std::unique_ptr<TVar> base
- `src/VarUnit.cpp` - Removed delete base from destructor (= default), base.reset() in clear(), base.reset(pVariable) in setBase(), base.get() in getBase() and pointer comparisons
- `src/updater.h` - Added #include <memory>, changed dblsqd::Feed* feed to std::unique_ptr<dblsqd::Feed> feed
- `src/updater.cpp` - make_unique<dblsqd::Feed>() in constructor, removed delete(feed) from destructor (= default), feed.get() at all connect/connectSingleShot/UpdateDialog sites

## Decisions Made

- Used `= default` for both destructors since unique_ptr members handle all cleanup — cleaner than an empty body
- dblsqd::Feed confirmed to have no QObject parent parameter (checked feed.h in 3rdparty/dblsqd/) — unique_ptr is the correct ownership pattern
- Pointer comparisons in VarUnit::varName() comparing TVar* to base required updating to `base.get()` (rule: unique_ptr is not implicitly comparable to raw pointer)

## Deviations from Plan

None - plan executed exactly as written.

The one additional change not explicitly listed was fixing pointer comparisons in `varName()` (`var == base` → `var == base.get()`), but this was covered by the plan's instruction to "search all of VarUnit.cpp for any remaining raw usage of base that would break with unique_ptr."

## Issues Encountered

- clang-format not available at `$(brew --prefix llvm)/bin/clang-format` (llvm not installed via homebrew). Used CommandLineTools version at `/Library/Developer/CommandLineTools/usr/bin/clang-format` successfully.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Both VarUnit and Updater now use RAII ownership — no manual delete in either class
- 21/21 tests pass
- Ready for plan 02-04 (profileShortcuts migration)

## Self-Check: PASSED

- SUMMARY.md: FOUND at .planning/phases/02-continue-memory-safety/02-03-SUMMARY.md
- VarUnit.h: FOUND, std::unique_ptr<TVar> base present
- VarUnit.cpp: FOUND, no delete base, base.reset() present
- updater.h: FOUND, std::unique_ptr<dblsqd::Feed> feed present
- updater.cpp: FOUND, no delete(feed), feed.get() at 14 sites, make_unique present
- Commit e57618f49: FOUND (Task 1 — VarUnit::base unique_ptr)
- Commit cbd0ab310: FOUND (Task 2 — Updater::feed unique_ptr)
- Commit d7d073758: FOUND (docs — metadata)

---
*Phase: 02-continue-memory-safety*
*Completed: 2026-03-25*
