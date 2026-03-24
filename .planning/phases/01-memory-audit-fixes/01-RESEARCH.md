# Phase 1: Memory Audit & Fixes - Research

**Researched:** 2026-03-25
**Domain:** C++/Qt6 memory ownership — raw pointer audit, smart pointer migration, Qt parent-child system
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Flag **owning raw pointers only** — allocations where (a) no Qt parent is set AND (b) the result is not wrapped in a smart pointer. Raw `new Foo(parentWidget)` with a correct Qt parent is safe and should NOT be flagged.
- **D-02:** Focus areas: member variables declared as raw pointers in `.h` files that are not deleted in the destructor and have no Qt parent; heap allocations in cleanup/reset functions that use `close()` without `WA_DeleteOnClose`; and non-Qt objects in collections (maps/lists) that are never deleted.
- **D-03:** Scope is `src/` only — exclude `3rdparty/`, `test/`, and generated files.
- **D-04:** Use `unique_ptr`/`shared_ptr` **only for non-Qt types** (plain C++ objects, Lua state wrappers, raw data structures). Qt-derived classes (`QObject` subclasses, `QWidget` subclasses) stay on Qt's parent-child ownership — do not wrap them in smart pointers.
- **D-05:** When migrating a non-Qt raw pointer to `unique_ptr`, use the standard `std::unique_ptr<T>` — no custom deleters unless the type requires it (e.g., `lua_State` needs `lua_close`).
- **D-06:** **Audit first, fix in one pass.** Produce a complete catalogue of all issues with `file:line` references before writing any fixes. Then apply all fixes in a single pass.
- **D-07:** Each fix gets its own atomic commit with a descriptive message (matches the upstream cherry-pick style already on this branch).
- **D-08:** Fix all issues found, including high-traffic classes (Host, TMainConsole, TLuaInterpreter). Verify with a clean build and the Qt test suite (`test/`). The fixes are mechanical ownership changes — no behavioral changes.
- **D-09:** No special comment-flagging for high-traffic changes beyond the normal commit message. Keep diffs clean.

### Claude's Discretion
- Whether to use `deleteLater()` vs `delete` for QWidget subclasses in non-event-loop cleanup contexts (use `deleteLater()` when in doubt — it's always safe)
- Exact smart pointer type (unique_ptr vs shared_ptr) when ownership semantics are clear from context
- Order of fixes within the single-pass implementation

### Deferred Ideas (OUT OF SCOPE)
- Refactoring monolithic classes (Host, TLuaInterpreter, dlgTriggerEditor) — separate concern, not memory safety
- Adding tests for memory correctness (Valgrind/ASan integration) — good follow-up but out of scope for this phase
- Fixing FIXME bugs in TTrigger.cpp pattern compilation locking — separate from memory management
</user_constraints>

---

## Summary

The Mudlet codebase uses three memory ownership models simultaneously: Qt's parent-child system (the majority), C++ smart pointers (already present in Host.h as `QScopedPointer<TMap>`, `QScopedPointer<TMedia>`, `std::unique_ptr<QNetworkProxy>`), and manual raw-pointer ownership for a handful of non-Qt types. The asymmetry between 1,370 `new` and 214 `delete` calls in `src/` is largely explained by Qt parent-child cleanup — the codebase is architecturally sound but has specific gaps that need closing.

The confirmed leaks and redundant ownership patterns are narrow and well-localized. The most significant issues are: (1) `mudlet::mpHunspell_sharedDictionary` — a `Hunhandle*` zeroed in the destructor without calling `Hunspell_destroy`, leaking the underlying C struct; (2) `TMap::mpRoomDB` — a plain-C++ pointer manually deleted in the destructor, a correct but fragile pattern that should be `unique_ptr`; (3) `TTrigger::mColorPatternList` and `mConditionMap` — these ARE correctly deleted in the destructor and in `setRegexCodeList()`, but the manual iteration pattern is fragile and should migrate to `QList<std::unique_ptr<TColorTable>>` and `std::map<TMatchState*, std::unique_ptr<TMatchState>>`; (4) `mudlet::mpTimerReplay` — a `QTimer` with `this` as parent, explicitly deleted in the destructor with a redundant `delete` (harmless but unnecessary, and the `QPointer` wrapper already nulls itself on deletion).

The already-fixed commits (Phase 0 cherry-picks) cover Lua state, Unit::uninstall(), tempComplexRegexTrigger, resetMainConsole, stopWatch, and Discord maps. Those must not be re-touched.

**Primary recommendation:** Audit by category — first find all `Hunhandle*` leaks (C library resources needing custom destroyers), then plain-C++ owning raw pointers that should be `unique_ptr`, then redundant manual `delete` on Qt-parented objects. Apply all fixes in one ordered pass, with one commit per file or logical group.

---

## Standard Stack

### Core (Already Present — Use What's There)

| Component | Version | Purpose | Status in Codebase |
|-----------|---------|---------|-------------------|
| Qt6 parent-child | Qt 6.8.2+ | Auto-delete QObject children | Already used everywhere |
| `std::unique_ptr<T>` | C++20 | Sole-ownership non-Qt objects | Already used in Host.h, TLuaInterpreter.h |
| `QScopedPointer<T>` | Qt 6 | Qt-idiomatic sole-ownership | Already used in Host.h for TMap, TMedia, GMCPAuthenticator |
| `QPointer<T>` | Qt 6 | Weak, self-nulling pointer for Qt objects | Already used in Host.h, mudlet.h |
| `deleteLater()` | Qt 6 | Deferred widget deletion via event loop | Already used in TDetachedWindow, TMainConsole |
| `WA_DeleteOnClose` | Qt 6 | Self-deleting widget on close | Already used in TConsole, TDetachedWindow, dlgComposer, mudlet |

### Specialised Patterns Needed for This Phase

| Pattern | Use Case | Example |
|---------|---------|---------|
| `unique_ptr` with custom deleter | Hunspell handles (`Hunhandle*`) needing `Hunspell_destroy()` | `struct HunspellDeleter { void operator()(Hunhandle* h) { Hunspell_destroy(h); } };` |
| `unique_ptr` with custom deleter | `lua_State*` needing `lua_close()` (already fixed — do not re-fix) | Reference only |
| `QList<std::unique_ptr<TColorTable>>` | Owning list of plain-C++ structs | Replace `QList<TColorTable*> mColorPatternList` |
| `std::map<TMatchState*, std::unique_ptr<TMatchState>>` | Self-keyed ownership map | Replace `std::map<TMatchState*, TMatchState*> mConditionMap` |

---

## Architecture Patterns

### Ownership Hierarchy

```
mudlet (QMainWindow — WA_DeleteOnClose)
  └─ HostManager
       └─ Host (QObject, owned by HostManager via QMap<QString, Host*> — qDeleteAll on shutdown)
            ├─ TMap          (QScopedPointer<TMap> mpMap — already smart pointer)
            │    └─ TRoomDB  (raw TRoomDB* mpRoomDB — manually deleted, CANDIDATE for unique_ptr)
            │    └─ TMapViewManager (QObject with TMap as parent — Qt-owned, correct)
            ├─ TMedia        (QScopedPointer<TMedia> — already smart pointer)
            ├─ GMCPAuthenticator (QScopedPointer<GMCPAuthenticator> — already smart pointer)
            ├─ TMainConsole  (QPointer<TMainConsole> mpConsole — Qt widget)
            │    ├─ mpHunspell_system   (Hunhandle* — needs custom deleter, partially correct)
            │    ├─ mpHunspell_shared   (Hunhandle* — NOT destroyed in destructor, LEAK)
            │    └─ mpHunspell_profile  (Hunhandle* — correctly destroyed)
            ├─ TriggerUnit / AliasUnit / TimerUnit / ScriptUnit / KeyUnit / ActionUnit
            │    └─ Tree<T> root lists — manual delete correct, Tree destructor recurses children
            └─ stopWatch* in mStopWatchMap (plain C++, qDeleteAll in destructor — correct)

mudlet (application singleton)
  ├─ mpTimerReplay (QPointer<QTimer> with 'this' parent — redundant explicit delete, harmless)
  ├─ mpHunspell_sharedDictionary (Hunhandle* — NOT destroyed, CONFIRMED LEAK)
  └─ mpShortcut* members (QPointer<QShortcut> with 'this' parent — delete+recreate pattern is intentional for key-binding switching, correct)
```

### Pattern 1: Qt Parent-Child (Safe — Do Not Touch)

Any `new Foo(this)` or `new Foo(parentWidget)` where `Foo` is a `QObject` or `QWidget` subclass. Qt deletes the child when the parent is destroyed. This is NOT a bug.

```cpp
// Source: Qt6 documentation + codebase observation
// These are CORRECT and should not be modified:
mpBlinkTimer = new QTimer(this);           // mudlet.cpp:792
mpShortcutTriggers = new QShortcut(..., this);  // mudlet.cpp:3672
mpNetworkAccessManager = new QNetworkAccessManager(this); // TMap.cpp:97
```

### Pattern 2: unique_ptr for Plain C++ Ownership (Apply This)

Non-Qt objects where the class is the sole owner. The destructor delete becomes implicit.

```cpp
// BEFORE (manual):
TRoomDB* mpRoomDB = nullptr;  // TMap.h
// In TMap.cpp ctor: mpRoomDB(new TRoomDB(this))
// In TMap.cpp dtor: delete mpRoomDB;

// AFTER (unique_ptr):
std::unique_ptr<TRoomDB> mpRoomDB;  // TMap.h
// In TMap.cpp ctor: mpRoomDB(std::make_unique<TRoomDB>(this))
// In TMap.cpp dtor: (nothing needed)
// Access: mpRoomDB->getRoom(id)  (unchanged, no -> vs . difference)
```

### Pattern 3: unique_ptr with Custom Deleter for C Library Resources (Apply This)

For `Hunhandle*` which requires `Hunspell_destroy()`, not `delete`.

```cpp
// BEFORE (manual with C function):
Hunhandle* mpHunspell_system = nullptr;
// dtor: if (mpHunspell_system) { Hunspell_destroy(mpHunspell_system); }

// AFTER (unique_ptr with custom deleter):
struct HunspellDeleter {
    void operator()(Hunhandle* h) const { if (h) { Hunspell_destroy(h); } }
};
std::unique_ptr<Hunhandle, HunspellDeleter> mpHunspell_system;
// dtor: (nothing needed — automatically calls Hunspell_destroy)
// Reset: mpHunspell_system.reset(Hunspell_create(aff, dic));
// Check: if (mpHunspell_system) { ... }
```

### Pattern 4: Owning Collection Migration (Apply This)

For `QList<TColorTable*>` where the list owns all entries.

```cpp
// BEFORE:
QList<TColorTable*> mColorPatternList;
// Cleanup: QMutableListIterator iteration with delete + remove

// AFTER:
QList<std::unique_ptr<TColorTable>> mColorPatternList;
// Cleanup: mColorPatternList.clear()  (unique_ptr destructors fire)
// Access: mColorPatternList[i].get() or mColorPatternList[i]->field
// Add: mColorPatternList.append(std::make_unique<TColorTable>())
```

### Pattern 5: Self-Keyed Ownership Map (Apply This)

For `std::map<TMatchState*, TMatchState*> mConditionMap` where key == value (both point to the same object).

```cpp
// BEFORE:
std::map<TMatchState*, TMatchState*> mConditionMap;
// Cleanup: for (auto&& [key, value] : mConditionMap) { delete value; }
// On removal: delete mConditionMap[matchState]; mConditionMap.erase(matchState);

// AFTER: Use a set of unique_ptrs (key IS the object):
// Option A - if lookup by raw pointer is needed:
std::map<TMatchState*, std::unique_ptr<TMatchState>> mConditionMap;
// Insert: auto* p = new TMatchState(...); mConditionMap[p] = std::unique_ptr<TMatchState>(p);
//   OR:   auto up = std::make_unique<TMatchState>(...);
//         auto* raw = up.get();
//         mConditionMap[raw] = std::move(up);
// Erase:  mConditionMap.erase(matchState);  (unique_ptr destructs automatically)
// Cleanup: mConditionMap.clear()  (no manual delete loop)
```

### Anti-Patterns to Avoid

- **Wrapping QObject/QWidget in unique_ptr or QScopedPointer**: Qt's parent-child system is incompatible with smart pointer ownership. Do not wrap `QTimer`, `QShortcut`, `QWidget` subclasses.
- **Using shared_ptr when unique_ptr suffices**: `TTrigger` children are solely owned by their `Tree<T>` parent node — use `unique_ptr`. Only use `shared_ptr` when ownership is genuinely shared across multiple parties.
- **Calling delete on QPointer members when the object has a Qt parent**: `mpTimerReplay` is a `QPointer<QTimer>` and `new QTimer(this)` — Qt will destroy it with `mudlet`. The explicit `delete` in the destructor is redundant (but safe due to `QPointer` self-nulling). Remove it for clarity.
- **Using deleteLater() in a non-QObject destructor context**: `deleteLater()` requires an event loop iteration. In destructors that are called during application shutdown (after the event loop has stopped), prefer direct `delete` for Qt widgets.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| C library resource lifecycle | Manual if/destroy + reset pattern | `unique_ptr` with custom deleter struct | Destructor fires even on exceptions and early returns; cleanup in multiple paths is error-prone |
| Sole-ownership heap object | Raw pointer + manual delete in destructor | `std::unique_ptr<T>` | Exception safety (even though project avoids exceptions), missed-delete on early return, double-delete risk |
| Qt object lifecycle | Smart pointer wrapping | Qt parent-child (`new Foo(this)`) | Qt's object tree handles child lifetime; smart pointers fight it |
| Qt widget self-destruction on close | Manual close() or delete | `setAttribute(Qt::WA_DeleteOnClose)` | `close()` alone only hides non-WA_DeleteOnClose widgets — this is an existing established pattern |

**Key insight:** The project already uses `QScopedPointer`, `std::unique_ptr`, and `QPointer` correctly in several places. The task is finding the remaining stragglers and applying the same patterns already established, not introducing new idioms.

---

## Confirmed Issue Catalogue

This is the pre-populated audit result from code inspection. The plan's audit wave should verify these and find any additional ones not listed here.

### Confirmed Leaks (Fix Required)

| File | Line | Issue | Fix |
|------|------|-------|-----|
| `src/mudlet.cpp` | ~4763 | `mpHunspell_sharedDictionary` set to `nullptr` in destructor **without** `Hunspell_destroy()` call — the Hunspell C struct is leaked | Add `Hunspell_destroy(mpHunspell_sharedDictionary)` before null assignment, or migrate to `unique_ptr` with `HunspellDeleter` |
| `src/TMainConsole.cpp` | ~1263 | `mpHunspell_shared` assigned from `mudlet::self()->prepareSharedDictionary()` but never destroyed in `TMainConsole::~TMainConsole()` (destructor only destroys `mpHunspell_system` and `mpHunspell_profile`) | Add `if (mpHunspell_shared) { Hunspell_destroy(mpHunspell_shared); }` in destructor, or migrate to `unique_ptr` with `HunspellDeleter` |

### Manual Delete Patterns — Correct but Fragile (Migrate to unique_ptr)

| File | Members | Current Pattern | Migration |
|------|---------|----------------|-----------|
| `src/TMap.h` / `src/TMap.cpp` | `TRoomDB* mpRoomDB` | `new TRoomDB(this)` in ctor; `delete mpRoomDB` in dtor | `std::unique_ptr<TRoomDB> mpRoomDB;` + `std::make_unique<TRoomDB>(this)` in ctor; remove dtor delete |
| `src/TTrigger.h` / `src/TTrigger.cpp` | `QList<TColorTable*> mColorPatternList` | Manual `QMutableListIterator` + `delete` in dtor and `setRegexCodeList()` | `QList<std::unique_ptr<TColorTable>> mColorPatternList`; replace cleanup with `.clear()`; update `createColorPattern` return type |
| `src/TTrigger.h` / `src/TTrigger.cpp` | `std::map<TMatchState*, TMatchState*> mConditionMap` | `delete value` in dtor loop; `delete mConditionMap[matchState]` + `erase` at line 1068 | `std::map<TMatchState*, std::unique_ptr<TMatchState>> mConditionMap`; remove manual deletes; update insertion at line 532–533; update erase at line 1068–1072 |
| `src/Host.h` / `src/Host.cpp` | `QMap<int, stopWatch*> mStopWatchMap` | `qDeleteAll(mStopWatchMap)` in dtor; `delete pStopWatch` in `destroyStopWatch()` and `clearNonPersistentStopWatches()` | `QMap<int, std::unique_ptr<stopWatch>> mStopWatchMap`; replace `qDeleteAll` with `.clear()`; update `take()` + `delete` pattern; update `value()` callers to use `.get()` |

### Redundant Explicit Delete on Qt-Parented Objects (Cleanup — Not Leaks)

| File | Line | Member | Issue | Fix |
|------|------|--------|-------|-----|
| `src/mudlet.cpp` | 4760 | `mpTimerReplay` (`QPointer<QTimer>` with `this` parent) | `delete (mpTimerReplay)` in destructor is redundant — Qt parent-child will delete it; `QPointer` self-nulls after deletion | Remove the `delete` call; the `mpTimerReplay = nullptr` line is also unnecessary |

### Already Correct — Do Not Modify

| Pattern | Files | Why Correct |
|---------|-------|-------------|
| QShortcut delete+recreate in `assignKeySequences()` | `src/mudlet.cpp:3655–3810` | Intentional: old shortcut must be destroyed before creating a new one with an updated key sequence. This is a programmatic shortcut-rebinding pattern, not a leak. `QPointer` ensures safe delete of parentless shortcut. |
| Tree<T> manual delete of children | `src/Tree.h:144–149` | TTrigger, TAlias etc. are NOT QObjects — Tree<T> must recursively `delete` children. This is correct. The `std::list<T*>* mpMyChildrenList` itself should migrate to `std::unique_ptr<std::list<T*>>` or `std::list<T*>` (value type) as a secondary cleanup. |
| `TRoomDB` manual deletes of rooms/areas | `src/TRoomDB.cpp:65–71, 1147–1154` | TRoom and TArea are plain C++ classes (not QObject), owned by TRoomDB collections. `delete room` / `delete area` in `TRoomDB::~TRoomDB()` is correct. Candidate for `QHash<int, std::unique_ptr<TRoom>>` migration but lower priority. |
| `TimerUnit` manual deletes of TTimer roots | `src/TimerUnit.cpp:50, 80, 422` | TTimer is `Tree<TTimer>` (not QObject). Manual `delete timer` in destructor and `uninstall()` is correct — Tree destructor recurses children. |
| `EditorDeleteItemCommand` deletes | `src/EditorDeleteItemCommand.cpp:753–833` | Undo/redo command explicitly takes ownership after unregistering items from units. Correct. |
| `XMLimport` deletes of mpTrigger etc. | `src/XMLimport.cpp:177–209` | These are error-path cleanup of items that failed to import. After successful import the items are owned by the unit trees. Correct. |
| Hunspell in `TMainConsole` for system + profile | `src/TMainConsole.cpp:81–87` | `mpHunspell_system` and `mpHunspell_profile` ARE correctly destroyed with `Hunspell_destroy()` in the destructor. Only `mpHunspell_shared` is missing. |

---

## Common Pitfalls

### Pitfall 1: Wrapping Qt Objects in Smart Pointers
**What goes wrong:** `std::unique_ptr<QTimer> mpTimer` — when the unique_ptr destructs, it calls `delete` on the QTimer, but Qt's parent may have already deleted it (double-delete) or the QTimer has pending events in the queue (crash).
**Why it happens:** Applying smart-pointer patterns to Qt objects that already have Qt ownership.
**How to avoid:** Check if the type inherits from `QObject`. If yes, use `QPointer` (weak, non-owning) or pass a `QObject*` parent at construction. Never use `unique_ptr` or `QScopedPointer` for Qt-parented objects.
**Warning signs:** `QObject` destructor output, double-free ASAN reports.

### Pitfall 2: Using make_unique with Types That Have C-Library Destructors
**What goes wrong:** `std::make_unique<Hunhandle>()` — won't compile because `Hunhandle` is an opaque C struct, not a C++ class with a destructor.
**Why it happens:** Applying standard `make_unique` to C library handles.
**How to avoid:** Use a custom deleter: `std::unique_ptr<Hunhandle, HunspellDeleter>`. The deleter struct calls `Hunspell_destroy()` instead of `delete`.
**Warning signs:** Compile error about incomplete type; forgetting to call the C library's destroy function.

### Pitfall 3: Losing Raw Pointer Access After Migration
**What goes wrong:** Code that stores a raw pointer locally before migrating to `unique_ptr` now has a dangling `T*` local after the owning `unique_ptr` is reset or moved.
**Why it happens:** `mStopWatchMap.take(id)` returns `stopWatch*` — after migrating the map to `unique_ptr` values, `take()` returns a `unique_ptr`, not a raw pointer. Callers need updating.
**How to avoid:** Update ALL call sites for containers that change value type. Use `map.take(key).release()` if raw ownership transfer is needed; use `map.value(key).get()` for non-owning access.
**Warning signs:** Compile errors at call sites; implicit raw pointer conversion warnings.

### Pitfall 4: Self-Keyed Map Insertion Order
**What goes wrong:** In `mConditionMap`, the key IS a pointer to the same object as the value. After migrating to `std::map<TMatchState*, unique_ptr<TMatchState>>`, inserting incorrectly by creating the unique_ptr first then passing `.get()` as the key is correct — but doing it the other way around (key from deleted pointer) is a use-after-free.
**Why it happens:** The insertion pattern must be: create object, capture raw pointer for key, transfer ownership to unique_ptr for value — in that order.
**How to avoid:** Use the pattern `auto up = std::make_unique<TMatchState>(...); auto* raw = up.get(); mConditionMap[raw] = std::move(up);`

### Pitfall 5: QMutableListIterator Incompatibility with unique_ptr
**What goes wrong:** `QMutableListIterator<TColorTable*>` cannot be used with `QList<std::unique_ptr<TColorTable>>` because unique_ptr is move-only.
**Why it happens:** Qt's iterator APIs expect copyable elements.
**How to avoid:** After migration, replace the iterator+delete pattern with `mColorPatternList.clear()`. For the `createColorPattern` return value, change return type from `TColorTable*` to `std::unique_ptr<TColorTable>` and use `mColorPatternList.append(std::move(pCT))`.

### Pitfall 6: Forgetting Forward Declarations When Adding unique_ptr to Headers
**What goes wrong:** Adding `std::unique_ptr<TRoomDB>` to `TMap.h` requires `#include <memory>` AND either `#include "TRoomDB.h"` or a forward declaration `class TRoomDB;`. With unique_ptr, the type must be complete at the point of destructor instantiation (unlike raw pointers).
**Why it happens:** unique_ptr's destructor is instantiated in the header for inline destructors.
**How to avoid:** Add `#include <memory>` to the header. Add forward declaration `class TRoomDB;` if only TMap.h needs the type. In TMap.cpp, `#include "TRoomDB.h"` is required (already present). If TMap has an inline destructor `~TMap() = default;` in the header, it will fail to compile with a forward-declared type — move `~TMap()` definition to TMap.cpp.

---

## Code Examples

### Hunspell Custom Deleter

```cpp
// Source: C++ standard (unique_ptr with custom deleter) + Hunspell C API
// In TMainConsole.h (or a shared header):
struct HunspellDeleter {
    void operator()(Hunhandle* h) const noexcept
    {
        if (h) {
            Hunspell_destroy(h);
        }
    }
};

// Member declaration (replaces Hunhandle* mpHunspell_system = nullptr;):
std::unique_ptr<Hunhandle, HunspellDeleter> mpHunspell_system;

// Reset (replaces mpHunspell_system = Hunspell_create(...)):
mpHunspell_system.reset(Hunspell_create(aff.toUtf8().constData(), dic.toUtf8().constData()));

// Old destroy pattern in destructor becomes unnecessary (unique_ptr handles it).
// Encoding accessor (unchanged API, access via .get()):
QByteArray(Hunspell_get_dic_encoding(mpHunspell_system.get()))
```

### TRoomDB Migration in TMap

```cpp
// Source: codebase inspection + C++ standard
// In TMap.h: replace
//   TRoomDB* mpRoomDB = nullptr;
// with:
//   std::unique_ptr<TRoomDB> mpRoomDB;
// Requires #include <memory> in TMap.h (already has other STL)
// TMap.h must NOT have an inline dtor if TRoomDB is only forward-declared there.

// In TMap.cpp ctor (replace mpRoomDB(new TRoomDB(this))):
, mpRoomDB(std::make_unique<TRoomDB>(this))

// In TMap.cpp dtor (remove):
// delete mpRoomDB;  <-- DELETE THIS LINE

// All callers of mpRoomDB use -> operator — no change needed at call sites.
```

### TColorTable List Migration in TTrigger

```cpp
// Source: codebase inspection + C++ standard
// In TTrigger.h: replace
//   QList<TColorTable*> mColorPatternList;
// with:
//   QList<std::unique_ptr<TColorTable>> mColorPatternList;

// In TTrigger.cpp destructor: replace iterator+delete with:
mColorPatternList.clear();  // unique_ptrs fire their destructors

// In setRegexCodeList(): replace iterator+delete with:
mColorPatternList.clear();

// In createColorPattern() return type: change from TColorTable* to std::unique_ptr<TColorTable>
// (or keep returning raw pointer and wrap at callsite if preferred)

// In addColorPattern / addAndMergeColorPattern: replace
//   mColorPatternList.append(pCT);
// with:
//   mColorPatternList.append(std::unique_ptr<TColorTable>(pCT));
// OR change createColorPattern to return unique_ptr and use std::move.

// Callers accessing list elements (e.g. TColorTable* pCT = mColorPatternList[patternNumber]):
TColorTable* pCT = mColorPatternList[patternNumber].get();
```

---

## Environment Availability

Step 2.6: SKIPPED for environment tooling — this phase is pure C++ source edits. However, build and test are required.

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| CMake 3.25.1+ | Build system | Assumed (existing dev environment) | No build dir found; build must be created |
| Qt 6.8.2+ | Qt headers/libs | Assumed (project already builds) | |
| clang-format (Homebrew LLVM on macOS) | Code style enforcement | `$(brew --prefix llvm)/bin/clang-format` | Required after all .cpp/.h edits per CLAUDE.md |
| `ctest` | Test suite verification | Part of CMake | Run from build dir after cmake configure |

**Build command (macOS):**
```bash
mkdir -p /Users/njs50/play/Mudlet/build
cd /Users/njs50/play/Mudlet/build
cmake ../../Mudlet -DCMAKE_PREFIX_PATH=$(brew --prefix qt6)
make -j $(sysctl -n hw.ncpu)
```

**Test command:**
```bash
cd /Users/njs50/play/Mudlet/build
ctest --output-on-failure
```

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Qt Test (Qt6::Test) |
| Config file | `test/CMakeLists.txt` |
| Quick run command | `ctest -R TLuaInterfaceTest --output-on-failure` (from build dir) |
| Full suite command | `ctest --output-on-failure` (from build dir) |

### Phase Requirements → Test Map

The existing test suite covers:
- `TLuaInterfaceTest` — Lua API correctness (relevant if TLuaInterpreter changes touch lua_State)
- `TMxp*Test` files — MXP processing (not directly relevant but build smoke test)
- `dlgTriggerEditorUndoRedoTest` — Trigger editor (relevant if TTrigger member types change)

Memory fix changes are structural (ownership types), not behavioral — no new test files required. Verification is: build compiles without warnings/errors + full ctest suite passes.

### Wave 0 Gaps
None — existing test infrastructure is sufficient. The phase gate is: build clean + ctest green.

---

## State of the Art

| Old Approach | Current Approach | Notes |
|--------------|------------------|-------|
| Raw pointer + manual delete in destructor | `std::unique_ptr<T>` | Already adopted in parts of Mudlet (Host.h) |
| `QScopedPointer<T>` for Qt objects | Avoid — `QScopedPointer` is redundant when Qt parent-child handles the lifetime | For non-Qt sole-ownership, use `std::unique_ptr` (already done in Host.h for `QNetworkProxy`) |
| Manual C library resource destroy | `unique_ptr` with custom deleter struct | Standard C++11/14/17/20 pattern |

**Deprecated/outdated in this codebase:**
- `QScopedPointer<T>` for non-Qt objects: The codebase uses it in `Host.h` for `TMap`, `TMedia`, `GMCPAuthenticator`. These work correctly; migrating them to `std::unique_ptr` is out of scope (not a memory safety issue, just style).

---

## Open Questions

1. **TTrigger::mConditionMap key pointer safety after migration**
   - What we know: The map keys are raw pointers to the same objects stored as values. After migrating to `unique_ptr` values, erasing an entry invalidates the key pointer — but the calling code at line 1068 already uses `mConditionMap.find(matchState)` and `mConditionMap.erase(matchState)` with a locally held `TMatchState*` pointer.
   - What's unclear: Whether any code retains a raw `TMatchState*` pointer after the corresponding map entry is erased (would become dangling).
   - Recommendation: Inspect all code paths that call `mConditionMap.erase()` and ensure no caller retains the raw pointer beyond the erase call. The existing pattern in TTrigger.cpp:1067–1072 appears safe — verify during implementation.

2. **TMainConsole::mpHunspell_shared ownership semantics**
   - What we know: `mpHunspell_shared` is obtained by calling `mudlet::self()->prepareSharedDictionary()`, which returns the same `Hunhandle*` singleton stored in `mudlet::mpHunspell_sharedDictionary`. Multiple `TMainConsole` instances may hold this same pointer.
   - What's unclear: If `mpHunspell_shared` is truly a shared (non-owning) reference to the singleton in `mudlet`, then `TMainConsole` should NOT destroy it — mudlet owns it. In that case, the fix is only in `mudlet::~mudlet()` (add `Hunspell_destroy`), and `TMainConsole::mpHunspell_shared` becomes a non-owning raw pointer (correct as-is).
   - Recommendation: Verify by tracing `prepareSharedDictionary()` — if it returns the same pointer each time (shared singleton), `TMainConsole` must NOT destroy it. The fix is: add `Hunspell_destroy(mpHunspell_sharedDictionary)` in `mudlet::~mudlet()` only. The TMainConsole member is then a non-owning observer pointer, not a leak.

---

## Project Constraints (from CLAUDE.md)

All fixes must comply with these directives:

- **C++20, no exceptions, no templates (complex), no concepts** — do not introduce complex template machinery; the custom deleter struct is simple and acceptable.
- **Qt6 minimum 6.8.2** — use Qt6 API only.
- **No exceptions** — `unique_ptr` destructor is `noexcept`, compatible with this constraint.
- **Qt parent-child for Qt-derived classes** — never wrap `QObject`/`QWidget` subclasses in `unique_ptr`.
- **clang-format after every .cpp/.h edit** — run `$(brew --prefix llvm)/bin/clang-format -i <file>` on macOS after each edit.
- **`qsl()` macro for string literals** — not directly relevant to this phase (no new UI strings).
- **`mp` prefix for private pointer members** — member renames are not required; only the type changes (raw pointer to `unique_ptr`). Keep the same names.
- **Include minimization** — adding `#include <memory>` to headers is required for `unique_ptr`. Check it isn't already present before adding.
- **Single-threaded architecture** — no concurrency concerns in these fixes.
- **Each fix is its own atomic commit** — one commit per file or logical group (per D-07).
- **Do not commit debug defines** — no `DEBUG_*` defines should be left uncommented.

---

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection (`src/TTrigger.h`, `src/TTrigger.cpp`, `src/TMap.h`, `src/TMap.cpp`, `src/TMainConsole.h`, `src/TMainConsole.cpp`, `src/mudlet.h`, `src/mudlet.cpp`, `src/Host.h`, `src/Host.cpp`, `src/Tree.h`) — all findings are from first-hand reading of the actual files on the `fix-memory-leaks` branch.
- `.planning/codebase/CONCERNS.md` — confirms 1,370 new / 214 delete asymmetry, TTrigger manual deletion flagged.
- `.planning/codebase/ARCHITECTURE.md` — ownership model.
- `.planning/codebase/CONVENTIONS.md` — coding patterns.
- `CLAUDE.md` — project constraints.
- `01-CONTEXT.md` — locked decisions.

### Secondary (MEDIUM confidence)
- Qt6 documentation (training data): QPointer, QScopedPointer, parent-child ownership, WA_DeleteOnClose, deleteLater() semantics.
- C++ standard (training data): `unique_ptr` with custom deleter, `make_unique`.

---

## Metadata

**Confidence breakdown:**
- Issue catalogue: HIGH — based on direct code inspection of named files
- Standard stack: HIGH — based on what's already in the codebase
- Architecture patterns: HIGH — directly read from source files
- Open questions (shared Hunspell pointer): MEDIUM — requires tracing one additional call path

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (codebase is stable; these are structural facts)
