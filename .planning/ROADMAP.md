# Roadmap: Mudlet Memory Safety

**Milestone:** v1 — Systematic memory safety audit and fixes
**Created:** 2026-03-24

---

## Phase 1: Memory Audit & Fixes

**Goal:** Systematically audit the Mudlet codebase for memory leaks and unsafe memory usage, then fix all identified issues using Qt's parent-child ownership system or C++ smart pointers.

**Plans:** 2/2 plans executed ✓

Plans:
- [x] 01-01-PLAN.md — Fix Hunspell leak, remove redundant QTimer delete, migrate TMap::mpRoomDB to unique_ptr
- [x] 01-02-PLAN.md — Migrate TTrigger::mColorPatternList and mConditionMap to unique_ptr

**Scope:**
- Grep and analyze source files for raw owning pointer patterns
- Identify heap allocations not covered by Qt parent-child or smart pointers
- Find objects whose lifetimes are managed inconsistently
- Apply fixes: migrate to parent-child ownership, unique_ptr, shared_ptr, or deleteLater() as appropriate

**Excludes:** Changes already applied via upstream cherry-picks (already in HEAD).

**Success criteria:**
- All new/raw-new allocations in src/ are either Qt-parented or smart-pointer-managed
- No `delete` calls in destructors for members that could use smart pointers
- No close() calls on widgets that lack WA_DeleteOnClose in cleanup paths
- Build passes cleanly

**Canonical refs:**
- `.planning/codebase/CONCERNS.md` — existing technical debt and memory concerns
- `.planning/codebase/CONVENTIONS.md` — established coding patterns
- `.planning/codebase/ARCHITECTURE.md` — ownership model and key classes

### Phase 2: Continue memory safety

**Goal:** Migrate remaining raw owning pointers in Host, Discord, VarUnit, and Updater to RAII ownership (unique_ptr or Qt parent-child), eliminating manual delete patterns.
**Requirements:** MEM-HOST-MMCP, MEM-HOST-STOPWATCH, MEM-HOST-PROFILESHORTCUTS, MEM-DISCORD-HANDLERS, MEM-DISCORD-PRESENCE, MEM-VARUNIT-BASE, MEM-UPDATER-FEED
**Depends on:** Phase 1
**Plans:** 2/3 plans executed

Plans:
- [x] 02-01-PLAN.md — Remove redundant mMMCPServer delete, migrate Host::mStopWatchMap and Host::profileShortcuts to unique_ptr
- [x] 02-02-PLAN.md — Migrate Discord::mpHandlers and mPresencePtrs to unique_ptr
- [x] 02-03-PLAN.md — Migrate VarUnit::base to unique_ptr, fix Updater::feed Qt parenting

---

*Roadmap created: 2026-03-24*
*Plans added: 2026-03-25*
