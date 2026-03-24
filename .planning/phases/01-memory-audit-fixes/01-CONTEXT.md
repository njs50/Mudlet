# Phase 1: Memory Audit & Fixes - Context

**Gathered:** 2026-03-24
**Status:** Ready for planning

<domain>
## Phase Boundary

Systematically audit `src/` for heap allocations that are not covered by Qt's parent-child ownership system or C++ smart pointers, then fix all identified issues. Excludes fixes already applied via the Phase 0 upstream cherry-picks (already in HEAD on the `fix-memory-leaks` branch).

</domain>

<decisions>
## Implementation Decisions

### Audit Scope
- **D-01:** Flag **owning raw pointers only** — allocations where (a) no Qt parent is set AND (b) the result is not wrapped in a smart pointer. Raw `new Foo(parentWidget)` with a correct Qt parent is safe and should NOT be flagged.
- **D-02:** Focus areas: member variables declared as raw pointers in `.h` files that are not deleted in the destructor and have no Qt parent; heap allocations in cleanup/reset functions that use `close()` without `WA_DeleteOnClose`; and non-Qt objects in collections (maps/lists) that are never deleted.
- **D-03:** Scope is `src/` only — exclude `3rdparty/`, `test/`, and generated files.

### Smart Pointer Strategy
- **D-04:** Use `unique_ptr`/`shared_ptr` **only for non-Qt types** (plain C++ objects, Lua state wrappers, raw data structures). Qt-derived classes (`QObject` subclasses, `QWidget` subclasses) stay on Qt's parent-child ownership — do not wrap them in smart pointers.
- **D-05:** When migrating a non-Qt raw pointer to `unique_ptr`, use the standard `std::unique_ptr<T>` — no custom deleters unless the type requires it (e.g., `lua_State` needs `lua_close`).

### Fix Approach
- **D-06:** **Audit first, fix in one pass.** Produce a complete catalogue of all issues with `file:line` references before writing any fixes. Then apply all fixes in a single pass.
- **D-07:** Each fix gets its own atomic commit with a descriptive message (matches the upstream cherry-pick style already on this branch).

### Risk Tolerance
- **D-08:** Fix all issues found, including high-traffic classes (Host, TMainConsole, TLuaInterpreter). Verify with a clean build and the Qt test suite (`test/`). The fixes are mechanical ownership changes — no behavioral changes.
- **D-09:** No special comment-flagging for high-traffic changes beyond the normal commit message. Keep diffs clean.

### Claude's Discretion
- Whether to use `deleteLater()` vs `delete` for QWidget subclasses in non-event-loop cleanup contexts (use `deleteLater()` when in doubt — it's always safe)
- Exact smart pointer type (unique_ptr vs shared_ptr) when ownership semantics are clear from context
- Order of fixes within the single-pass implementation

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Codebase analysis
- `.planning/codebase/CONCERNS.md` §Memory Management Concerns — catalogues the 1,370 `new` vs 214 `delete` asymmetry and TTrigger.cpp manual deletion issues
- `.planning/codebase/CONVENTIONS.md` — coding patterns to follow when writing fixes
- `.planning/codebase/ARCHITECTURE.md` — ownership model, key classes and their relationships

### Prior fixes (already applied — do not re-fix)
- `src/TLuaInterpreter.cpp` — Lua state leak fixed in commit `21228ad3a`
- `src/AliasUnit.cpp`, `src/KeyUnit.cpp`, `src/ScriptUnit.cpp`, `src/TimerUnit.cpp`, `src/TriggerUnit.cpp` — uninstall() leak fixed in `6f462e003`
- `src/TLuaInterpreterMudletObjects.cpp` — tempComplexRegexTrigger fixed in `45b44ae33`
- `src/TMainConsole.cpp` — resetMainConsole widget lifecycle fixed in `7deb9901e`
- `src/Host.cpp` — stopWatch + mMMCPServer fixed in `5c1778710` + `b9ce65e62`
- `src/discord.cpp`, `src/mudlet.cpp` — Discord map cleanup fixed in `b9ce65e62`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Qt parent-child: `new Foo(this)` pattern used throughout — where `this` is a QObject, child is auto-deleted
- `deleteLater()`: already used correctly in resetMainConsole and elsewhere for deferred widget deletion
- `WA_DeleteOnClose`: used on `TConsole`, `TDetachedWindow`, `dlgComposer`, `mudlet` — the pattern to follow for QWidget subclasses that own themselves

### Established Patterns
- **QWidget cleanup**: set `WA_DeleteOnClose` OR use `deleteLater()` — `close()` alone only hides non-WA_DeleteOnClose widgets
- **QObject trees**: passing `this` (or a parent QWidget) as constructor argument gives Qt ownership — no manual delete needed
- **Non-Qt heap objects**: currently managed with raw pointers and manual delete in destructors; candidates for `unique_ptr`
- **Collections of pointers**: `QMap<QString, T*>` and `QList<T*>` throughout — items must be explicitly deleted before/during container clear

### Integration Points
- `src/Host.h` / `src/Host.cpp` — central class; destructor and `closeChildren()` are the main cleanup paths
- `src/TMainConsole.cpp` — `resetMainConsole()` is the profile-reset cleanup path (already fixed)
- `src/TTrigger.cpp` / `src/TTrigger.h` — manual TColorTable and condition map deletion flagged in CONCERNS.md
- `src/TLuaInterpreter.h` — `pGlobalLua` (raw `lua_State*`) is a candidate for custom smart pointer with `lua_close` deleter

</code_context>

<specifics>
## Specific Ideas

- No specific UI or interaction requirements — this is internal code quality work
- Fix style should match the upstream cherry-pick commits already in HEAD (short, factual commit messages; minimal diffs)

</specifics>

<deferred>
## Deferred Ideas

- Refactoring monolithic classes (Host, TLuaInterpreter, dlgTriggerEditor) — separate concern, not memory safety
- Adding tests for memory correctness (Valgrind/ASan integration) — good follow-up but out of scope for this phase
- Fixing FIXME bugs in TTrigger.cpp pattern compilation locking — separate from memory management

</deferred>

---

*Phase: 01-memory-audit-fixes*
*Context gathered: 2026-03-24*
