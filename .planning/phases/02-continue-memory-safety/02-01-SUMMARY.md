---
phase: 02-continue-memory-safety
plan: 01
subsystem: memory-safety
tags: [cpp, unique_ptr, smart-pointers, QMap, stdmap, Host, stopWatch, QKeySequence]

# Dependency graph
requires: []
provides:
  - Host::mMMCPServer delete removed — Qt parent-child handles destruction
  - mStopWatchMap migrated to std::map<int, std::unique_ptr<stopWatch>>
  - profileShortcuts migrated to std::map<QString, std::unique_ptr<QKeySequence>>
  - No manual delete of stopWatch or QKeySequence anywhere in codebase
affects: [future Host changes, XMLimport, XMLexport, dlgProfilePreferences, mudlet, TDetachedWindow]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - std::map<K, std::unique_ptr<V>> for non-copyable value ownership (avoids Qt COW issue)
    - operator[] + std::move for inserting move-only values into std::map
    - find/end pattern replaces QMap::value() for safe raw pointer access from unique_ptr map

key-files:
  created: []
  modified:
    - src/Host.h
    - src/Host.cpp
    - src/XMLimport.cpp
    - src/XMLexport.cpp
    - src/dlgProfilePreferences.cpp
    - src/TDetachedWindow.cpp
    - src/mudlet.cpp

key-decisions:
  - "Use std::map instead of QMap for unique_ptr values — Qt's COW detach copies the underlying std::map, requiring copyable values, making QMap<K, unique_ptr<V>> uncompilable"
  - "getStopWatch(id) inline method updated to use std::map::find + .get() — callers inside Host.cpp use this helper directly"
  - "XMLimport readStopWatchMap — use make_unique, insert via operator[] + std::move"
  - "XMLimport readProfileShortcut — use stack QKeySequence + swap, eliminating separate heap allocation and manual delete"

patterns-established:
  - "std::map preferred over QMap when values are move-only (unique_ptr)"
  - "find/end pattern for safe unique_ptr value access: auto it = map.find(key); if (it != map.end()) { use it->second.get(); }"

requirements-completed:
  - MEM-HOST-MMCP
  - MEM-HOST-STOPWATCH
  - MEM-HOST-PROFILESHORTCUTS

# Metrics
duration: 15min
completed: 2026-03-24
---

# Phase 02 Plan 01: Host Memory Safety Summary

**Eliminated all manual delete calls for MMCPServer, stopWatch, and QKeySequence in Host by migrating to std::map + unique_ptr ownership with RAII cleanup**

## Performance

- **Duration:** 15 min
- **Started:** 2026-03-24T21:00:26Z
- **Completed:** 2026-03-24T21:15:51Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments

- Removed redundant `delete mMMCPServer` from Host destructor — Qt parent-child already handles QObject cleanup
- Migrated `mStopWatchMap` from `QMap<int, stopWatch*>` to `std::map<int, std::unique_ptr<stopWatch>>` — zero manual deletes remaining
- Migrated `profileShortcuts` from `QMap<QString, QKeySequence*>` to `std::map<QString, std::unique_ptr<QKeySequence>>` — qDeleteAll removed
- Updated all seven caller files to use find/end + .get() pattern instead of .value()
- Build clean, 21/21 tests pass

## Task Commits

Each task was committed atomically:

1. **Task 1: Remove redundant mMMCPServer delete and migrate mStopWatchMap to unique_ptr** - `d9f3813b7` (feat)
2. **Task 2: Migrate Host::profileShortcuts to unique_ptr** - `0904c8d23` (feat)

## Files Created/Modified

- `src/Host.h` - mStopWatchMap changed to std::map<int, unique_ptr<stopWatch>>; profileShortcuts changed to std::map<QString, unique_ptr<QKeySequence>>; getStopWatch() updated to use std::map::find
- `src/Host.cpp` - Removed delete mMMCPServer, qDeleteAll calls; createStopWatch uses make_unique + operator[]; destroyStopWatch uses find+erase; all mStopWatchMap.value() replaced with getStopWatch()
- `src/XMLimport.cpp` - readStopWatchMap uses make_unique; readProfileShortcut uses stack QKeySequence + swap
- `src/XMLexport.cpp` - profileShortcuts export uses find/end + .get()
- `src/dlgProfilePreferences.cpp` - Two profileShortcuts.value() sites converted to find/end pattern
- `src/TDetachedWindow.cpp` - profileShortcuts.value() converted to find/end + const raw pointer
- `src/mudlet.cpp` - Two profileShortcuts.value() sites converted to find/end + .get()

## Decisions Made

- Used `std::map` instead of `QMap` for both migrated containers: Qt's `QMap` uses copy-on-write via `QSharedData`, and the internal `detach()` method copies the underlying `std::map<K, V>`, which requires `V` to be copyable. Since `std::unique_ptr<T>` is move-only, `QMap<K, std::unique_ptr<T>>` fails to compile whenever any QMap method triggers a detach.
- Used `operator[]` + `std::move` for insertion into std::map (simpler than emplace for this case).
- Removed the `std::sort` in `findStopWatchId` and `setStopWatchName(QString, QString)` — `std::map` is already sorted by key, so sorting the extracted key list was unnecessary (and removed).

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Switched from QMap to std::map for unique_ptr containers**

- **Found during:** Task 1 (mStopWatchMap migration)
- **Issue:** Qt's `QMap` has copy-on-write internals — `detach()` copies the underlying `std::map`, requiring `V` to be copyable. `std::unique_ptr<T>` is move-only. `QMap<int, std::unique_ptr<stopWatch>>` fails to compile at any insertion or access that triggers detach (including `insert()` and `operator[]`).
- **Fix:** Changed type to `std::map<int, std::unique_ptr<stopWatch>>` (Task 1) and `std::map<QString, std::unique_ptr<QKeySequence>>` (Task 2). Updated all usages from Qt iterator API (`.constBegin()`, `.constEnd()`, `.isEmpty()`, `.contains()`, `.take()`, `.keys()`, `.value()`) to std::map equivalents (`cbegin()`, `cend()`, `.empty()`, `.count()`, erase-from-iterator, manual key list, `find()/.end()/.get()`).
- **Files modified:** src/Host.h, src/Host.cpp, and all seven caller files
- **Verification:** Build compiles cleanly; 21/21 tests pass
- **Committed in:** d9f3813b7 and 0904c8d23

---

**Total deviations:** 1 auto-fixed (Rule 3 — blocking compilation issue)
**Impact on plan:** The fix is necessary for correctness. The container change achieves the same ownership semantics the plan intended. All semantic outcomes from the plan's must_haves and artifacts are satisfied with std::map instead of QMap.

## Issues Encountered

- Qt6's `QMap` cannot store `std::unique_ptr` values due to COW architecture requiring copyable values. Resolved by switching to `std::map` which has no copy requirement and supports move-only values natively.

## Next Phase Readiness

- Host memory ownership is clean for mMMCPServer, mStopWatchMap, and profileShortcuts
- Ready to proceed to phase 02's next plans (remaining memory safety items)
- No blockers

---
*Phase: 02-continue-memory-safety*
*Completed: 2026-03-24*
