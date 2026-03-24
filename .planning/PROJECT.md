# Mudlet Memory Safety

## What This Is

A systematic audit and fix of memory management across the Mudlet codebase. Mudlet is a cross-platform MUD client (Qt6/C++20) with years of accumulated code where raw pointers, manual allocation, and inconsistent widget lifecycle patterns have introduced memory leaks and unsafe memory usage. This project addresses those issues systematically.

## Core Value

Every heap allocation is either owned by Qt's parent-child system or a C++ smart pointer — no raw owning pointers that require manual deletion.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

- ✓ Lua state not closed before re-init on profile reset — fixed (Phase 0 cherry-picks)
- ✓ Unit::uninstall() called unregister() instead of delete — fixed (Phase 0 cherry-picks)
- ✓ tempComplexRegexTrigger leaked old trigger — fixed (Phase 0 cherry-picks)
- ✓ resetMainConsole used close() on widgets without WA_DeleteOnClose — fixed (Phase 0 cherry-picks)
- ✓ stopWatch objects not freed in ~Host() — fixed (Phase 0 cherry-picks)
- ✓ mMMCPServer + Discord map dangling pointers — fixed (Phase 0 cherry-picks)
- ✓ Hunspell leak + mpTimerReplay + mpRoomDB raw delete — fixed (Phase 1)
- ✓ TTrigger::mColorPatternList raw owning pointers — migrated to unique_ptr (Phase 1)
- ✓ Host::mStopWatchMap + profileShortcuts raw owning maps — migrated to std::map<K, unique_ptr<V>> (Phase 2)
- ✓ Discord::mpHandlers + mPresencePtrs raw pointers — migrated to unique_ptr (Phase 2)
- ✓ VarUnit::base + Updater::feed raw owning pointers — migrated to unique_ptr (Phase 2)

### Active

- [ ] Comprehensive audit of remaining raw owning pointers in codebase
- [ ] Fix unsafe memory patterns found in audit
- [ ] Prefer Qt parent-child ownership for QObject-derived types
- [ ] Prefer smart pointers (unique_ptr/shared_ptr) for non-Qt types

### Out of Scope

- Performance optimizations unrelated to memory — separate concern
- Refactoring non-memory-related code — separate concern
- Adding new features — separate concern

## Context

Mudlet is single-threaded (main thread for all profiles, triggers, Lua engine). Networking handled by Qt in background. Key classes: Host, TMainConsole, TLuaInterpreter, ctelnet, TMap. Recent cherry-picks from upstream fixed the most obvious leaks. This project does the broader audit.

## Constraints

- **Tech stack**: Qt6/C++20, no exceptions, no templates, no concepts
- **Threading**: Single-threaded — no thread-safety concerns for memory ownership
- **Qt patterns**: Use WA_DeleteOnClose + deleteLater() for widgets, parent-child for QObjects
- **Smart pointers**: unique_ptr/shared_ptr for non-Qt heap allocations

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Cherry-pick upstream leak fixes first | Avoid re-doing known fixes | ✓ Good |

---
*Last updated: 2026-03-25 after Phase 2 complete — Host, Discord, VarUnit, Updater unique_ptr migrations done*
