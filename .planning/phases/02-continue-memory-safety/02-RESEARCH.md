# Phase 2: Continue Memory Safety - Research

**Researched:** 2026-03-25
**Domain:** C++/Qt6 memory ownership — deep audit of remaining raw owning pointers, manual delete patterns, and fragile collections not addressed in Phase 1
**Confidence:** HIGH

---

## Summary

Phase 1 fixed the most clearly broken patterns: the Hunspell leak, the redundant `mpTimerReplay` delete, `TMap::mpRoomDB` migration to `unique_ptr`, and the `TTrigger` collection migrations. Phase 2 targets the remaining raw owning pointers across `Host`, `Discord`, `Updater`, `VarUnit`, `ShortcutsManager`, `TRoomDB`, and `dlgRoomExits` that were identified in Phase 1 research as "correctly managed but fragile" or "confirmed issues not yet tackled."

The highest-priority items are:
1. **`Host::mStopWatchMap`** — `QMap<int, stopWatch*>` with `qDeleteAll` in destructor and `delete pStopWatch` in `destroyStopWatch()` and `removeAllNonPersistentStopWatches()`. Should become `QMap<int, std::unique_ptr<stopWatch>>`.
2. **`Host::mMMCPServer`** — `QPointer<MMCPServer>` with explicit `delete mMMCPServer` in the destructor. `MMCPServer` is a `QTcpServer` (QObject) created with `this` as Qt parent — the delete is redundant (Qt parent-child handles it). Remove the `delete` call.
3. **`Host::profileShortcuts`** — `QMap<QString, QKeySequence*>` with `qDeleteAll` in destructor. `QKeySequence` is a plain C++ value class (not QObject). Should become `QMap<QString, std::unique_ptr<QKeySequence>>`.
4. **`Discord::mPresencePtrs`** — `QMap<QString, localDiscordPresence*>` with manual `QMutableMapIterator` + `delete` in destructor. `localDiscordPresence` is a plain C++ class. Should become `QMap<QString, std::unique_ptr<localDiscordPresence>>`.
5. **`Discord::mpHandlers`** — `DiscordEventHandlers*` allocated with `new DiscordEventHandlers` and deleted in destructor. `DiscordEventHandlers` is a plain C struct. Should become `std::unique_ptr<DiscordEventHandlers>`.
6. **`Updater::feed`** — `dblsqd::Feed*` allocated with `new dblsqd::Feed()` (no Qt parent), deleted in destructor. `dblsqd::Feed` is a QObject. Per project rules, Qt objects should NOT use `unique_ptr` — but since `feed` has no Qt parent, it can be given the `Updater` instance as parent (pass `this` at construction), making it Qt-parented and removing the manual delete.
7. **`VarUnit::base`** — `TVar* base` with `delete base` in destructor and in `clear()`. `TVar` is a plain C++ class. Should become `std::unique_ptr<TVar>`.
8. **`ShortcutsManager::defaults`** — `QMap<QString, QKeySequence*>` with manual `QMutableMapIterator` + `delete` in destructor. Should become `QMap<QString, std::unique_ptr<QKeySequence>>`.
9. **`TRoomDB::rooms` / `TRoomDB::areas`** — `QHash<int, TRoom*>` and `QMap<int, TArea*>` with manual bulk delete in `~TRoomDB()` and `clearMapDB()`, and individual `delete pR` / `delete pA` in `removeRoom()` / `removeArea()`. `TRoom` and `TArea` are plain C++ classes. Migration to `unique_ptr` values is viable but requires careful ordering (map remove BEFORE unique_ptr destruction, since destructors do re-entrant cleanup). **Lower priority** — this is the most complex migration; the current bulk-delete-with-flag pattern is correct.
10. **`dlgRoomExits::makeExitFromControls()`** — local `TExit*` pattern with `delete currentExit` after comparison. `TExit` is a plain C++ struct. Should use `unique_ptr<TExit>` or return by value.

**Primary recommendation:** Address items 1–8 in order of risk: stopWatch map (confirmed correctness issue — multiple delete paths), MMCPServer redundant delete, profileShortcuts, Discord cleanup, Updater::feed parenting, VarUnit::base, ShortcutsManager::defaults. Skip TRoomDB rooms/areas for now (complex, low risk, already uses bulk-delete with optimization flag). Address `dlgRoomExits::makeExitFromControls` as a minor cleanup.

---

## Project Constraints (from CLAUDE.md)

All fixes must comply with these directives:

- **C++20, no exceptions, no templates (complex), no concepts** — custom deleter structs are simple and acceptable.
- **Qt6 minimum 6.8.2** — use Qt6 API only.
- **Qt parent-child for Qt-derived classes** — never wrap `QObject`/`QWidget` subclasses in `unique_ptr`. Instead, pass a Qt parent at construction.
- **Use `std::unique_ptr` for non-Qt owned objects.** Project already uses this in `Host.h` for `QNetworkProxy`.
- **`mp` prefix for private pointer members** — only type changes, not renames.
- **clang-format after every .cpp/.h edit** — `$(brew --prefix llvm)/bin/clang-format -i <file>` on macOS.
- **Include minimization** — add `#include <memory>` only where not already present.
- **Single-threaded architecture** — no concurrency concerns.
- **Each fix is its own atomic commit** — one commit per file or logical group.
- **Do not commit debug defines** — no `DEBUG_*` defines left uncommented.

---

## Standard Stack

### Core (Already Present — Use What's There)

| Component | Version | Purpose | Status in Codebase |
|-----------|---------|---------|-------------------|
| Qt6 parent-child | Qt 6.8.2+ | Auto-delete QObject children | Already used everywhere |
| `std::unique_ptr<T>` | C++20 | Sole-ownership non-Qt objects | Already in Host.h, TLuaInterpreter.h, TMap.h, TTrigger.h |
| `QScopedPointer<T>` | Qt 6 | Qt-idiomatic sole-ownership | Already in Host.h (TMap, TMedia, GMCPAuthenticator), LuaInterface.h (VarUnit) |
| `QPointer<T>` | Qt 6 | Weak, self-nulling pointer for Qt objects | Already in Host.h, mudlet.h |
| `deleteLater()` | Qt 6 | Deferred widget deletion via event loop | Already used in Host.cpp |

### No New Libraries Needed

All patterns needed are already established in the codebase. Phase 2 applies them to the remaining stragglers.

---

## Architecture Patterns

### Pattern 1: Qt Parent-Child (Fix redundant deletes)

`MMCPServer` is a `QTcpServer` (QObject subclass) created with `this` (Host) as Qt parent. The `delete mMMCPServer` in `Host::~Host()` is redundant — Qt will delete it when Host is destroyed. `mMMCPServer` is declared as `QPointer<MMCPServer>` which already self-nulls after deletion.

```cpp
// BEFORE (redundant): src/Host.cpp line 434
delete mMMCPServer;

// AFTER: remove that line entirely — QPointer<MMCPServer> mMMCPServer with Qt parent
// Qt's parent-child handles deletion. No explicit delete needed.
```

### Pattern 2: unique_ptr for Plain C++ Collections (Apply This)

For `QMap<int, stopWatch*> mStopWatchMap` where the map owns all `stopWatch` instances.

```cpp
// BEFORE:
QMap<int, stopWatch*> mStopWatchMap;
// dtor: qDeleteAll(mStopWatchMap);
// destroyStopWatch: auto pStopWatch = mStopWatchMap.take(id); delete pStopWatch;
// removeAllNonPersistent: delete pStopWatch; itStopWatch.remove();

// AFTER:
QMap<int, std::unique_ptr<stopWatch>> mStopWatchMap;
// dtor: mStopWatchMap.clear();  (unique_ptrs fire destructors)
// destroyStopWatch: mStopWatchMap.take(id);  (returned unique_ptr destructs on scope exit)
// removeAllNonPersistent: itStopWatch.remove();  (no manual delete)
// All .value(id) callers: use .value(id).get() for raw pointer access
// getStopWatch(id): return mStopWatchMap.value(id).get();
```

### Pattern 3: Give Parentless QObject a Qt Parent Instead of unique_ptr

`Updater::feed` is a `dblsqd::Feed*` (QObject) created with `new dblsqd::Feed()` — no parent, manual `delete(feed)` in destructor. The Updater IS a QObject (`QObject* parent` in its ctor). The fix is to pass `this` as parent at construction rather than wrapping in `unique_ptr` (which would conflict with Qt's ownership model).

```cpp
// BEFORE:
feed = new dblsqd::Feed();
// ~Updater(): delete (feed);

// AFTER:
feed = new dblsqd::Feed(this);  // Qt parent-child owns it
// ~Updater(): remove the delete (feed); line
```

Note: `feed` is declared as `dblsqd::Feed* feed` in `updater.h` — no type change needed, just pass `this` at construction.

### Pattern 4: unique_ptr with value type for plain C++ struct

`DiscordEventHandlers` is a C struct allocated with `new DiscordEventHandlers` and deleted in `Discord::~Discord()`. Since it's a plain struct (no virtual destructor needed), use `std::unique_ptr<DiscordEventHandlers>`.

```cpp
// BEFORE:
DiscordEventHandlers* mpHandlers = nullptr;
// dtor: delete mpHandlers;
// ctor: mpHandlers = new DiscordEventHandlers; memset(mpHandlers, 0, sizeof(DiscordEventHandlers));

// AFTER:
std::unique_ptr<DiscordEventHandlers> mpHandlers;
// dtor: (nothing needed)
// ctor: mpHandlers = std::make_unique<DiscordEventHandlers>(); memset(mpHandlers.get(), 0, sizeof(DiscordEventHandlers));
// Usage: Discord_Initialize(..., mpHandlers.get(), ...)
```

### Pattern 5: Owning Map Migration for Plain C++ Class (Apply This)

For `QMap<QString, localDiscordPresence*> mPresencePtrs` where the map owns all entries.

```cpp
// BEFORE:
QMap<QString, localDiscordPresence*> mPresencePtrs;
// dtor: QMutableMapIterator iteration + delete each value

// AFTER:
QMap<QString, std::unique_ptr<localDiscordPresence>> mPresencePtrs;
// dtor: mPresencePtrs.clear();  (unique_ptrs fire)
// Insertion: mPresencePtrs.insert(key, std::make_unique<localDiscordPresence>());
// Access: mPresencePtrs.value(key).get() for raw pointer access
//   OR: auto* pDiscordPresence = mPresencePtrs.value(key).get();
```

### Pattern 6: unique_ptr for VarUnit::base

`VarUnit::base` is a `TVar* base` — sole owner, manually deleted in destructor and `clear()`.

```cpp
// BEFORE:
TVar* base;
// ~VarUnit(): delete base;
// clear(): delete base; base = nullptr;
// setBase(TVar*): base = pVariable;
// getBase(): return base;

// AFTER:
std::unique_ptr<TVar> base;
// ~VarUnit(): (nothing needed)
// clear(): base.reset(); (replaces delete + nullptr assign)
// setBase(TVar*): base.reset(pVariable);
// getBase(): return base.get();
```

Note: Callers of `getBase()` receive a `TVar*` (non-owning) — no caller changes needed.

### Pattern 7: unique_ptr Map for ShortcutsManager::defaults

```cpp
// BEFORE:
QMap<QString, QKeySequence*> defaults;
// ~ShortcutsManager(): QMutableMapIterator + delete each value

// AFTER:
QMap<QString, std::unique_ptr<QKeySequence>> defaults;
// ~ShortcutsManager(): defaults.clear();
// registerShortcut: defaults.insert(key, std::make_unique<QKeySequence>(*sequence));
// getDefault: return defaults.value(key).get();
```

### Pattern 8: unique_ptr Map for Host::profileShortcuts

```cpp
// BEFORE:
QMap<QString, QKeySequence*> profileShortcuts;
// ~Host(): qDeleteAll(profileShortcuts); profileShortcuts.clear();
// Creation: profileShortcuts.insert(entry, new QKeySequence(*...));

// AFTER:
QMap<QString, std::unique_ptr<QKeySequence>> profileShortcuts;
// ~Host(): profileShortcuts.clear();
// Creation: profileShortcuts.insert(entry, std::make_unique<QKeySequence>(*...));
// Access: all callers of profileShortcuts.value(key) must use .get()
```

### Anti-Patterns to Avoid

- **Wrapping QObject/QWidget in unique_ptr**: Never. Qt parent-child owns QObjects.
- **Passing `this` as parent to QObject when the parent is being destroyed**: Safe at construction time in constructors; unsafe in destructors.
- **Changing QPointer to unique_ptr**: `QPointer` is a weak, non-owning handle. Do not change `QPointer<MMCPServer> mMMCPServer` to `unique_ptr` — the pointer is already managed by Qt parent-child. Just remove the redundant `delete`.

---

## Confirmed Issue Catalogue

### Category A: Redundant Explicit Delete on Qt-Parented Objects (Cleanup — Not Leaks)

| File | Line | Member | Issue | Fix |
|------|------|--------|-------|-----|
| `src/Host.cpp` | ~434 | `mMMCPServer` (`QPointer<MMCPServer>` with `this` parent) | `delete mMMCPServer` in destructor is redundant — `MMCPServer` extends `QTcpServer` (QObject), created with `new MMCPServer(this)` | Remove the `delete mMMCPServer;` line |

### Category B: Manual delete Patterns — Correct but Fragile (Migrate to unique_ptr)

| File | Members | Current Pattern | Migration |
|------|---------|----------------|-----------|
| `src/Host.h` / `src/Host.cpp` | `QMap<int, stopWatch*> mStopWatchMap` | `qDeleteAll` in dtor; `take()` + `delete` in `destroyStopWatch()`; iterator + `delete` in `removeAllNonPersistentStopWatches()` | `QMap<int, std::unique_ptr<stopWatch>>`; clear replaces `qDeleteAll`; `take()` returns `unique_ptr` (auto-deletes); iterator `remove()` alone suffices |
| `src/Host.h` / `src/Host.cpp` | `QMap<QString, QKeySequence*> profileShortcuts` | `qDeleteAll(profileShortcuts)` in dtor | `QMap<QString, std::unique_ptr<QKeySequence>>`; `clear()` replaces `qDeleteAll`; update insertion and access sites |
| `src/discord.h` / `src/discord.cpp` | `QMap<QString, localDiscordPresence*> mPresencePtrs` | Manual `QMutableMapIterator` + `delete` loop in dtor | `QMap<QString, std::unique_ptr<localDiscordPresence>>`; `clear()` replaces loop; update `new localDiscordPresence` → `make_unique`; access via `.get()` |
| `src/discord.h` / `src/discord.cpp` | `DiscordEventHandlers* mpHandlers` | `new DiscordEventHandlers` + `memset` in ctor; `delete mpHandlers` in dtor | `std::unique_ptr<DiscordEventHandlers> mpHandlers`; `make_unique<DiscordEventHandlers>()` + `memset(.get(), ...)` in ctor; remove dtor delete; pass `.get()` to `Discord_Initialize` |
| `src/VarUnit.h` / `src/VarUnit.cpp` | `TVar* base` | `delete base` in dtor; `delete base; base = nullptr;` in `clear()` | `std::unique_ptr<TVar> base`; `base.reset()` in `clear()`; `setBase()` → `base.reset(pVariable)`; `getBase()` → `return base.get()` |
| `src/ShortcutsManager.h` / `src/ShortcutsManager.cpp` | `QMap<QString, QKeySequence*> defaults` | Manual `QMutableMapIterator` + `delete` in dtor | `QMap<QString, std::unique_ptr<QKeySequence>>`; `clear()` replaces loop; update `registerShortcut()` insertion; `getDefault()` returns `.get()` |

### Category C: Parentless QObject — Give Qt Parent (Not unique_ptr)

| File | Member | Current | Fix |
|------|--------|---------|-----|
| `src/updater.h` / `src/updater.cpp` | `dblsqd::Feed* feed` | `new dblsqd::Feed()` (no parent); `delete(feed)` in dtor | `feed = new dblsqd::Feed(this)` — Updater IS a QObject with a valid parent chain; remove `delete(feed)` |

### Category D: Local Raw Pointer Temporary (Minor Cleanup)

| File | Function | Current | Fix |
|------|----------|---------|-----|
| `src/dlgRoomExits.cpp` | `makeExitFromControls()` callers | `TExit* currentExit = makeExitFromControls(...)` followed by comparison and `delete currentExit` — repeated ~7 times in `isModified()` | Change `makeExitFromControls()` return type from `TExit*` to `std::unique_ptr<TExit>`; remove all `delete currentExit` calls |

### Category E: Already Correct — Do Not Modify

| Pattern | Files | Why Correct |
|---------|-------|-------------|
| `TRoomDB::rooms` / `areas` bulk delete | `src/TRoomDB.cpp` | Correct: uses `mBulkDeletionMode` flag to suppress re-entrant cleanup in TRoom/TArea destructors during bulk deletion. `areas.remove()` before `delete pA` in `removeArea()` is intentional. Complex interaction makes unique_ptr migration high-risk for low gain. |
| `dlgIRC::userModels` / `bufferTexts` | `src/dlgIRC.cpp` | `IrcUserModel(buffer)` and `QTextDocument(buffer)` pass `buffer` as Qt parent — Qt ownership is set at construction. The `delete userModels.take(buffer)` / `delete bufferTexts.take(buffer)` in `slot_onBufferRemoved` fires when the buffer is being removed; since `buffer` is the parent, Qt would also delete the children. The manual delete is technically redundant but harmless — low priority. |
| `ShortcutsManager::shortcuts` (non-owning) | `src/ShortcutsManager.cpp` | The `shortcuts` map holds non-owning raw pointers to `mudlet`'s member `QKeySequence` stack variables — correctly NOT deleted in destructor. |
| `Tree<T>` manual delete of children | `src/Tree.h` | TTrigger, TAlias etc. are NOT QObjects — Tree<T> must recursively delete children. Correct. |
| `EditorDeleteItemCommand` / `XMLimport` deletes | Various | Intentional ownership transfer for undo/redo and error-path cleanup. Correct. |

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Owning map of plain C++ objects | Manual `qDeleteAll` + clear | `QMap<K, std::unique_ptr<V>>`; call `.clear()` | `qDeleteAll` fires at every destruction path; unique_ptr fires unconditionally including exceptions (even though project avoids them) |
| Plain C++ struct lifecycle | Manual delete in dtor | `std::unique_ptr<T>` | RAII — destructor fires even on early return |
| Parentless QObject lifecycle | Manual delete in dtor | Pass `this` as Qt parent at construction | Qt's tree handles cleanup automatically; avoids double-delete risk with QObject's internal cleanup |
| Local raw pointer for temporary | `T* p = new T(...); ... delete p;` | `std::unique_ptr<T> p = std::make_unique<T>(...)` | Automatic cleanup, exception-safe, clearer intent |

**Key insight:** Every place the project still uses `qDeleteAll(someMap)` is a candidate for `unique_ptr` values in the map. The pattern is mechanical.

---

## Common Pitfalls

### Pitfall 1: Calling .value() on QMap<K, unique_ptr<V>>
**What goes wrong:** `QMap<QString, std::unique_ptr<QKeySequence>>::value(key)` returns a `const std::unique_ptr<QKeySequence>&` (or a copy, which won't compile for unique_ptr). Qt's `QMap::value()` returns by value — this will fail to compile because `unique_ptr` is not copyable.
**Why it happens:** Qt's `QMap::value(key)` is `T value(const Key &key, const T &defaultValue = T()) const` — which tries to copy the value type.
**How to avoid:** Use `mMap[key].get()` (direct map lookup by reference) OR use `mMap.find(key)` + dereference iterator. Do NOT use `QMap::value()` for unique_ptr maps. Replace `mStopWatchMap.value(id)` with `mStopWatchMap.contains(id) ? mStopWatchMap[id].get() : nullptr` or the helper `getStopWatch(id)` which can wrap this.
**Warning signs:** Compile error "use of deleted function 'std::unique_ptr<...>::unique_ptr(const std::unique_ptr<...>&)'".

### Pitfall 2: take() Returns unique_ptr — Callers Must Update
**What goes wrong:** `mStopWatchMap.take(id)` after migration returns `std::unique_ptr<stopWatch>` not `stopWatch*`. Code doing `auto pStopWatch = mStopWatchMap.take(id); delete pStopWatch;` will fail to compile (can't delete a unique_ptr).
**Why it happens:** `QMap::take()` returns by value. After migration, the value type is unique_ptr, so the caller gets ownership via the returned unique_ptr — which auto-destructs at end of scope.
**How to avoid:** Remove the `delete pStopWatch;` line after `take()`. The unique_ptr returned by `take()` will destruct automatically.

### Pitfall 3: QMutableMapIterator Incompatibility with unique_ptr
**What goes wrong:** `QMutableMapIterator<QString, localDiscordPresence*>` cannot be used with `QMap<QString, std::unique_ptr<localDiscordPresence>>` because unique_ptr is move-only.
**Why it happens:** Qt's mutable iterators expect copyable element types.
**How to avoid:** Replace the manual iterator + delete loops with `mMap.clear()`. The unique_ptr destructors fire automatically.

### Pitfall 4: Discord_Initialize takes raw pointer — Must Use .get()
**What goes wrong:** `Discord_Initialize(..., mpHandlers, ...)` after migrating `mpHandlers` to `unique_ptr` — the function expects `DiscordEventHandlers*`, not `std::unique_ptr<DiscordEventHandlers>`.
**How to avoid:** Use `mpHandlers.get()` at all call sites: `Discord_Initialize(..., mpHandlers.get(), ...)`.

### Pitfall 5: getBase() Return Type After VarUnit::base Migration
**What goes wrong:** `getBase()` currently returns `TVar*` which matches the raw pointer type. After migration, `base` is `unique_ptr<TVar>`. The return must be `base.get()`.
**How to avoid:** Change the `getBase()` implementation to `return base.get();` — the return type stays `TVar*` (non-owning). No callers need to change.

### Pitfall 6: setBase() Ownership Transfer
**What goes wrong:** `setBase(TVar* pVariable)` currently does `base = pVariable`. After migration, doing `base = pVariable` would fail (can't assign `TVar*` to `unique_ptr<TVar>`). Doing `base.reset(pVariable)` is correct — it transfers ownership to the unique_ptr.
**Critical:** Before calling `setBase()` with a new TVar, the caller must NOT delete the old one — `reset()` handles that automatically. Verify no caller deletes the old base after calling setBase.
**How to avoid:** Use `base.reset(pVariable)` in `setBase()`.

### Pitfall 7: dblsqd::Feed parent and signal connections
**What goes wrong:** If `feed` was previously created without a parent, signals connected to `feed` may already exist at the point of destruction. Qt handles this correctly with parent-child and disconnect-on-destroy, so no issue — but verify no code assumes `feed` outlives `Updater` (it won't, which is the correct behavior anyway).
**How to avoid:** Confirm `feed` is only accessed within `Updater` methods and that `Updater` destructor is only called when no pending Qt event holds a reference to `feed`.

---

## Code Examples

### stopWatch map migration — Host.cpp destroyStopWatch

```cpp
// BEFORE (src/Host.cpp ~line 1450):
auto pStopWatch = mStopWatchMap.take(id);
if (!pStopWatch) {
    return false;
}
// ... use pStopWatch
delete pStopWatch;

// AFTER:
auto pStopWatch = mStopWatchMap.take(id);  // returns unique_ptr<stopWatch>
if (!pStopWatch) {
    return false;
}
// ... use pStopWatch.get()
// No delete — unique_ptr destructs at end of scope
```

### stopWatch map migration — QMap lookup

```cpp
// BEFORE (src/Host.cpp):
auto pStopWatch = mStopWatchMap.value(id);  // QMap::value() returns stopWatch*

// AFTER — QMap::value() won't compile with unique_ptr, use operator[]:
auto it = mStopWatchMap.find(id);
stopWatch* pStopWatch = (it != mStopWatchMap.end()) ? it.value().get() : nullptr;

// OR in getStopWatch() inline accessor:
stopWatch* getStopWatch(const int id) const
{
    auto it = mStopWatchMap.constFind(id);
    return (it != mStopWatchMap.constEnd()) ? it.value().get() : nullptr;
}
```

### removeAllNonPersistentStopWatches migration

```cpp
// BEFORE (src/Host.cpp ~line 3242):
QMutableMapIterator<int, stopWatch*> itStopWatch(mStopWatchMap);
while (itStopWatch.hasNext()) {
    itStopWatch.next();
    auto pStopWatch = itStopWatch.value();
    if (!pStopWatch || pStopWatch->persistent()) {
        continue;
    }
    itStopWatch.remove();
    delete pStopWatch;
}

// AFTER:
QMutableMapIterator<int, std::unique_ptr<stopWatch>> itStopWatch(mStopWatchMap);
while (itStopWatch.hasNext()) {
    itStopWatch.next();
    // unique_ptr::operator bool() checks non-null
    if (!itStopWatch.value() || itStopWatch.value()->persistent()) {
        continue;
    }
    itStopWatch.remove();  // remove() from QMutableMapIterator destructs the value
}
```

### Discord mPresencePtrs migration

```cpp
// BEFORE (src/discord.cpp dtor):
QMutableMapIterator<QString, localDiscordPresence*> itPresencePtrs(mPresencePtrs);
while (itPresencePtrs.hasNext()) {
    itPresencePtrs.next();
    delete itPresencePtrs.value();
    itPresencePtrs.remove();
}

// AFTER:
mPresencePtrs.clear();  // unique_ptrs fire destructors automatically
```

```cpp
// BEFORE (src/discord.cpp ~line 353):
auto* pTempPresence = new localDiscordPresence;
mPresencePtrs.insert(QString(), pTempPresence);

// AFTER:
mPresencePtrs.insert(QString(), std::make_unique<localDiscordPresence>());
```

```cpp
// BEFORE (access):
localDiscordPresence* pDiscordPresence = mPresencePtrs.value(nullptr);

// AFTER — QMap::value() won't compile for unique_ptr, use find():
auto it = mPresencePtrs.find(applicationID);
localDiscordPresence* pDiscordPresence = (it != mPresencePtrs.end()) ? it.value().get() : nullptr;
```

### DiscordEventHandlers migration

```cpp
// BEFORE (src/discord.h):
DiscordEventHandlers* mpHandlers = nullptr;

// AFTER:
std::unique_ptr<DiscordEventHandlers> mpHandlers;
```

```cpp
// BEFORE (src/discord.cpp ctor):
mpHandlers = new DiscordEventHandlers;
memset(mpHandlers, 0, sizeof(DiscordEventHandlers));
mpHandlers->ready = handleDiscordReady;

// AFTER:
mpHandlers = std::make_unique<DiscordEventHandlers>();
memset(mpHandlers.get(), 0, sizeof(DiscordEventHandlers));
mpHandlers->ready = handleDiscordReady;  // unique_ptr::operator-> works
```

```cpp
// Passing to C function (src/discord.cpp ~line 110):
Discord_Initialize(..., mpHandlers.get(), ...);
```

### dlgRoomExits makeExitFromControls

```cpp
// BEFORE (src/dlgRoomExits.h):
TExit* makeExitFromControls(int direction);

// AFTER:
std::unique_ptr<TExit> makeExitFromControls(int direction);
```

```cpp
// BEFORE (src/dlgRoomExits.cpp ~line 2073):
TExit* currentExit = makeExitFromControls(DIR_NORTHWEST);
if (originalExit && currentExit && *originalExit != *currentExit) {
    isModified = true;
}
delete currentExit;

// AFTER:
auto currentExit = makeExitFromControls(DIR_NORTHWEST);
if (originalExit && currentExit && *originalExit != *currentExit) {
    isModified = true;
}
// No delete — unique_ptr destructs at end of scope
```

---

## State of the Art

| Old Approach | Current Approach | Notes |
|--------------|------------------|-------|
| `qDeleteAll(map)` + clear | `map.clear()` with `unique_ptr` values | Already adopted in TTrigger, TMap after Phase 1 |
| Manual `QMutableMapIterator` + delete loop | `.clear()` with `unique_ptr` values | Same pattern |
| `new Foo()` without parent in QObject subclass | `new Foo(this)` | Pass parent at construction |

---

## Open Questions

1. **QMutableMapIterator + unique_ptr values**
   - What we know: `QMutableMapIterator<K, std::unique_ptr<V>>::remove()` should call the value's destructor (unique_ptr destructs), per Qt's container behavior. This is the `removeAllNonPersistentStopWatches` case.
   - What's unclear: Whether Qt 6.8.2's `QMutableMapIterator::remove()` correctly destructs unique_ptr values or if there is a subtle copy in the iterator internals.
   - Recommendation: After migration, test the `removeAllNonPersistentStopWatches()` path specifically in the build. If the iterator does not compile with unique_ptr values, use `it = mStopWatchMap.erase(it)` with a regular `QMap::iterator` as an alternative.

2. **Discord `mPresencePtrs.value(nullptr)` with empty QString key**
   - What we know: `mPresencePtrs.value(nullptr)` uses a null `QString` as key — equivalent to `mPresencePtrs.value(QString())`. After migration to unique_ptr, `value()` won't compile, but `find(QString())` / `find(nullptr)` should work identically.
   - Recommendation: Use `mPresencePtrs.find(applicationID)` pattern throughout. The null-key lookup is valid — just needs the iterator-based access.

3. **profileShortcuts access sites outside Host.cpp**
   - What we know: `profileShortcuts` is accessed in `Host.cpp` lines 410 (insertion) and 428 (destruction). The map is declared `private` in `Host.h` (at line 797).
   - What's unclear: Whether any other file accesses `profileShortcuts` through a method. A grep shows only Host.cpp directly touches this map.
   - Recommendation: Confirm no additional access sites before migrating.

---

## Environment Availability

Step 2.6: SKIPPED — this phase is pure C++ source edits with no new external dependencies.

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| CMake 3.25.1+ | Build verification | Yes (existing dev env) | Use existing build directory |
| Qt 6.8.2+ | Qt headers/libs | Yes (project already builds) | |
| clang-format (Homebrew LLVM on macOS) | Code style enforcement | `$(brew --prefix llvm)/bin/clang-format` | Required after all .cpp/.h edits per CLAUDE.md |
| `ctest` | Test suite verification | Part of CMake | Run from build dir |

**Build command (macOS):**
```bash
cd /Users/njs50/play/Mudlet/build
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

Memory safety fixes are structural (ownership type changes), not behavioral. The existing test suite covers:
- `TLuaInterfaceTest` — exercises `VarUnit` and `TVar` tree indirectly via Lua API; relevant to `VarUnit::base` migration
- `dlgTriggerEditorUndoRedoTest` — exercises trigger/alias/timer items; not directly affected by this phase's changes

No new test files are required. Verification gate is: build compiles cleanly + full ctest passes.

### Wave 0 Gaps
None — existing test infrastructure is sufficient. The phase gate is: build clean + ctest green.

---

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection (all findings are from reading the actual files on the `fix-memory-leaks` branch):
  - `src/Host.h` — `mStopWatchMap`, `profileShortcuts`, `mMMCPServer` declarations
  - `src/Host.cpp` — all destruction and stopWatch management patterns
  - `src/discord.h` — `mPresencePtrs`, `mpHandlers` declarations
  - `src/discord.cpp` — all creation and destruction patterns
  - `src/updater.h` / `src/updater.cpp` — `feed` declaration and lifecycle
  - `src/VarUnit.h` / `src/VarUnit.cpp` — `base` declaration and usage
  - `src/ShortcutsManager.h` / `src/ShortcutsManager.cpp` — `defaults` declaration and lifecycle
  - `src/TRoomDB.h` / `src/TRoomDB.cpp` — rooms/areas collections
  - `src/dlgRoomExits.h` / `src/dlgRoomExits.cpp` — `makeExitFromControls` pattern
  - `3rdparty/dblsqd/dblsqd/feed.h` — confirms `dblsqd::Feed` is a QObject
- `.planning/phases/01-memory-audit-fixes/01-RESEARCH.md` — Phase 1 confirmed issue catalogue
- `CLAUDE.md` — project constraints

### Secondary (MEDIUM confidence)
- Qt6 documentation (training data): `QMap::value()` return-by-value behavior, `QMutableMapIterator::remove()` semantics, Qt parent-child ownership.
- C++ standard (training data): `unique_ptr` with plain structs, `make_unique`, `reset()`, `get()`.

---

## Metadata

**Confidence breakdown:**
- Issue catalogue: HIGH — based on direct code inspection of every named file
- Standard stack: HIGH — based on what's already in the codebase
- Architecture patterns: HIGH — directly read from source files
- Open questions (QMutableMapIterator + unique_ptr): MEDIUM — requires compile verification

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (codebase is stable; these are structural facts about named files)
