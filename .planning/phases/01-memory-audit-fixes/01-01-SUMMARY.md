---
plan: 01-01
phase: "01"
subsystem: memory-management
status: complete
wave: 1
tags: [memory-safety, destructor, smart-pointer, hunspell, qt-parent]
dependency_graph:
  requires: []
  provides: [safe-mpRoomDB-ownership, hunspell-destroy-fix]
  affects: [TMap, mudlet-dtor]
tech_stack:
  added: [std::unique_ptr<TRoomDB>]
  patterns: [RAII, smart-pointer-migration]
key_files:
  created: []
  modified:
    - src/mudlet.cpp
    - src/TMap.h
    - src/TMap.cpp
    - src/RoomContextMenuHandler.cpp
    - src/RoomMoveDragHandler.cpp
    - src/XMLimport.cpp
decisions:
  - Use .get() at TArea/TRoom construction sites rather than changing their constructors to accept unique_ptr (non-owning callers)
  - Use std::move(pNewRoomDB) into mpRoomDB in readJsonMapFile swap pattern (eliminates separate delete of old value)
metrics:
  duration_seconds: 1030
  completed_date: "2026-03-24"
  tasks_completed: 2
  files_modified: 6
---

# Phase 01 Plan 01: Memory Safety Fixes - Hunspell Leak and TMap::mpRoomDB Summary

**One-liner:** Fixed Hunspell_destroy leak in mudlet dtor, removed redundant QTimer delete, and migrated TMap::mpRoomDB from raw TRoomDB* to std::unique_ptr<TRoomDB>.

## Tasks Completed

1. **Fixed Hunspell shared dictionary leak** — Added `Hunspell_destroy(mpHunspell_sharedDictionary)` before nulling the pointer in `mudlet::~mudlet()`. The handle was previously leaked on every shutdown.

2. **Removed redundant mpTimerReplay delete** — Removed `delete(mpTimerReplay); mpTimerReplay = nullptr;` from `mudlet::~mudlet()`. The timer has Qt parent `this` and is wrapped in a `QPointer<QTimer>`, so it is auto-deleted by Qt's parent-child ownership system.

3. **Migrated TMap::mpRoomDB to std::unique_ptr** — Changed declaration to `std::unique_ptr<TRoomDB> mpRoomDB`, added `#include <memory>` to TMap.h, updated ctor to use `std::make_unique<TRoomDB>(this)`, removed explicit `delete mpRoomDB` from destructor. Updated all construction sites that passed `mpRoomDB` directly (requiring a raw `TRoomDB*`) to use `.get()`.

## Files Changed

- `src/mudlet.cpp` — `Hunspell_destroy` added, `delete mpTimerReplay` + null removed
- `src/TMap.h` — `#include <memory>` added, `mpRoomDB` changed to `std::unique_ptr<TRoomDB>`
- `src/TMap.cpp` — `make_unique` in ctor, `delete mpRoomDB` removed from dtor, `readJsonMapFile` swap pattern updated to `std::move`, `.get()` added at `new TArea`/`new TRoom` call sites
- `src/RoomContextMenuHandler.cpp` — `.get()` added where `auto* roomDatabase = mpRoomDB`
- `src/RoomMoveDragHandler.cpp` — `.get()` added where `auto* roomDb = mpRoomDB`
- `src/XMLimport.cpp` — `.get()` added where `new TRoom(mpRoomDB)`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed auto* pointer assignments in RoomContextMenuHandler and RoomMoveDragHandler**
- **Found during:** Task 2 build verification
- **Issue:** `auto* roomDatabase = mpRoomDB` and `auto* roomDb = mpRoomDB` no longer compile once mpRoomDB is unique_ptr — `auto*` deduces to a pointer, incompatible with unique_ptr
- **Fix:** Added `.get()` to both assignment sites
- **Files modified:** src/RoomContextMenuHandler.cpp, src/RoomMoveDragHandler.cpp
- **Commit:** 28dcf96d4

**2. [Rule 1 - Bug] Fixed direct mpRoomDB usage in TMap.cpp and XMLimport.cpp**
- **Found during:** Task 2 build verification (second compile pass)
- **Issue:** `new TArea(this, mpRoomDB)` (3 sites in TMap.cpp) and `new TRoom(mpRoomDB)` (TMap.cpp + XMLimport.cpp) pass the unique_ptr where raw TRoomDB* is expected
- **Fix:** Added `.get()` at each construction site; also migrated local `pNewRoomDB` in `readJsonMapFile` to `std::make_unique` and used `std::move` for the swap, eliminating the manual `delete pNewRoomDB`
- **Files modified:** src/TMap.cpp, src/XMLimport.cpp
- **Commit:** 28dcf96d4

## Build Status

PASS — 80 targets built with no errors. Only pre-existing linker warnings about macOS SDK version mismatches (not introduced by these changes). Note: TTrigger.cpp has a pre-existing unrelated compile error (`unique_ptr<TColorTable>` copy constructor issue) that existed before this plan.

## Known Stubs

None.
