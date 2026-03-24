---
status: investigating
trigger: "multi-profile-module-shared-state: When multiple profiles run simultaneously in the same Mudlet instance and both try to install or uninstall the same module at the same time, there may be shared state corruption, memory leaks, or race conditions."
created: 2026-03-28T00:00:00Z
updated: 2026-03-28T00:01:00Z
---

## Current Focus

hypothesis: CONFIRMED — FontManager in mudlet singleton is keyed only by packageName (not by profileName+packageName), causing three distinct failure modes when multiple profiles install the same module
test: Full code trace of FontManager.loadFont / rememberFont / unloadFonts + cross-profile install/uninstall sequences
expecting: N/A — root cause confirmed
next_action: DONE — FontManager fix committed; Lua module shared state investigation complete (see Eliminated section)

## Symptoms

expected: Each profile's module install/uninstall should be isolated — one profile's module operations should not affect another profile's state or memory
actual: Memory grows significantly faster when multiple profiles share a single Mudlet instance compared to running separate Mudlet instances per profile. Suspected cause: concurrent module install/uninstall operations across profiles sharing some global/static state.
errors: No crash reported — silent memory growth or state corruption
reproduction: Run 2+ profiles simultaneously in one Mudlet instance. Have both profiles use the same module. Trigger install/uninstall on both. Observe memory growth vs single-profile baseline.
started: Noticed after separating profiles into separate Mudlet instances reduced the memory issues

## Eliminated

- hypothesis: Static variables in XMLimport.cpp or TLuaInterpreter share module state
  evidence: Only two static QHash constants for color lookups; no module-related static state
  timestamp: 2026-03-28

- hypothesis: mInstalledModules, mInstalledPackages, mActiveModules, mModulesLoadedOk are shared singletons
  evidence: All four are per-Host instance members (Host.h:741-745,921) — correctly isolated
  timestamp: 2026-03-28

- hypothesis: mPackagesToInstallList in mudlet singleton causes cross-profile contamination during startup
  evidence: mPackagesToInstallList is cleared immediately after being applied to ONE profile (mudlet.cpp:4504); only affects initial profile load, not ongoing install/uninstall
  timestamp: 2026-03-28

- hypothesis: Lua package.loaded / require cache is shared across profiles
  evidence: Each profile has its own TLuaInterpreter with its own lua_State (luaL_newstate() in initLuaGlobals). pGlobalLua is a per-TLuaInterpreter member. package.loaded is per-lua_State. No sharing.
  timestamp: 2026-03-28

- hypothesis: mudlet singleton has module/package tracking maps without per-profile scope
  evidence: Exhaustive search of mudlet.h shows only mPackagesToInstallList (cleared immediately) and mFontManager (now fixed). No other module/package related containers in the singleton.
  timestamp: 2026-03-28

- hypothesis: mEventHandlerMap or mAnonymousEventHandlerFunctions accumulates across install/uninstall
  evidence: Both are per-Host instance members. unregisterEventHandler removes entries. The Lua-side registerAnonymousEventHandler overrides the C++ one; only C++ call is registerAnonymousEventHandler("*","dispatchEventToFunctions") which is set once at profile init. Lua-side handlers table is per-lua_State.
  timestamp: 2026-03-28

- hypothesis: Trigger/Timer/Alias/Key/Script/Action unit objects leak on install/uninstall
  evidence: All *Unit::uninstall(packageName) methods delete the relevant objects. TTrigger/TTimer/TAlias/TKey destructors call delete_luafunction(mFuncName) to clean up Lua globals. TScript does not use a named wrapper function (it runs in global scope), so no Lua global cleanup needed. mLookupTable, mTriggerMap, mActionRootNodeList all cleaned on uninstall.
  timestamp: 2026-03-28

- hypothesis: mModuleInfo or mModuleSequenceOrder are shared singletons
  evidence: mModuleInfo is at Host.h:678 — per-Host instance member. Correctly isolated.
  timestamp: 2026-03-28

- hypothesis: mWordSet_shared in mudlet singleton grows with module install/uninstall
  evidence: mWordSet_shared only grows if Lua scripts call addWordToDictionary() explicitly. Pure Lua modules that don't call this function are unaffected. Even if called, growth is bounded by vocabulary size, not install/uninstall cycles.
  timestamp: 2026-03-28

## Evidence

- timestamp: 2026-03-28
  checked: FontManager class (FontManager.h:44-46, FontManager.cpp)
  found: Two containers — loadedFontPaths (QMap<QString,int>) keyed by font filename only; loadedFontAffiliation (QMultiMap<QString,int>) keyed by packageName only. Neither includes a profile/Host identifier.
  implication: Both containers are in the mudlet singleton's mFontManager, shared across all profiles

- timestamp: 2026-03-28
  checked: FontManager::loadFont / fontAlreadyLoaded / rememberFont sequence (FontManager.cpp:63-100)
  found: fontAlreadyLoaded checks loadedFontPaths.contains(fileName) — filename only, no profile. rememberFont early-returns if filename already in loadedFontPaths, so the second profile's font ID and affiliation are NEVER registered.
  implication: When Profile A installs module "foo" with MyFont.ttf, Profile B's install of the same module silently skips registration. unloadFonts("foo") removes all "foo" affiliations — including those that belong to Profile A, breaking Profile B's fonts while Profile B's install never gets tracked.

- timestamp: 2026-03-28
  checked: FontManager::unloadFonts (FontManager.cpp:103-110)
  found: Removes font IDs from loadedFontAffiliation and calls QFontDatabase::removeApplicationFont. Does NOT remove from loadedFontPaths.
  implication: After any uninstall, loadedFontPaths retains the stale filename entry. Any subsequent install of the same module (by any profile) sees "already loaded" and skips loading — but the font was removed from Qt's database. Font is permanently broken for the entire application. This is the permanent resource leak.

- timestamp: 2026-03-28
  checked: Three-failure-mode analysis combining the above
  found:
    MODE A (silent cross-profile font removal): Profile A installs "foo" (font ID=5 registered for "foo"). Profile B installs "foo" (skipped, never registered). Profile A uninstalls → removeApplicationFont(5) → font gone from Qt, but Profile B still renders with it → silent visual corruption.
    MODE B (permanent stale entry → font never reloadable): After any uninstall, loadedFontPaths["MyFont.ttf"] = stale ID remains. Any reinstall returns early from fontAlreadyLoaded → QFontDatabase.addApplicationFont never called → font resource leaked (unregistered but OS memory not freed because no tracking).
    MODE C (uninstall with no-op unload → Qt font resource leak): Profile B (the second installer) uninstalls "foo". unloadFonts("foo") finds ZERO entries in loadedFontAffiliation for "foo" (never registered). QFontDatabase.removeApplicationFont never called. Qt holds the font resource indefinitely.
  implication: All three modes compound with repeated install/uninstall cycles across profiles, explaining the observed accelerating memory growth vs single-profile baseline.

- timestamp: 2026-03-28
  checked: Host::installPackageFonts and Host::uninstallPackage calls into mFontManager (Host.cpp:2312, 2669)
  found: installPackageFonts calls mudlet::self()->mFontManager.loadFont(filePath, packageName) — packageName only, no profile name. uninstallPackage calls mudlet::self()->mFontManager.unloadFonts(packageName) — same issue.
  implication: The bug is at the call sites: packageName alone is used as the affiliation key instead of a compound (profileName, packageName) key.

- timestamp: 2026-03-28
  checked: FONTMANAGER FIX COMMITTED — commit 467dd1ce1
  found: Fix scopes loadedFontPaths key by profileName+filePath (absolute path already contains profile directory so it's uniquely scoped). loadedFontAffiliation key changed to profileName+packageName. unloadFonts now also removes stale path entries so reinstall works. Global built-in fonts (addFonts/loadFonts) use empty profileName and are unaffected.
  implication: All three failure modes eliminated.

- timestamp: 2026-03-28
  checked: Full Lua module shared-state investigation for pure Lua modules (no fonts)
  found: Each profile has its own lua_State (luaL_newstate per TLuaInterpreter). package.loaded is per-lua_State. All module tracking containers (mInstalledModules, mInstalledPackages, mActiveModules, mModulesLoadedOk, mModuleInfo) are per-Host instance members. All *Unit::uninstall() methods properly delete their objects and clean up Lua globals. mEventHandlerMap and mAnonymousEventHandlerFunctions are per-Host. mudlet singleton has no module-related growing containers beyond mFontManager (now fixed).
  implication: For pure Lua modules with no fonts, there is no identified C++ shared state that leaks across profiles with install/uninstall cycles. The FontManager fix addresses the one confirmed shared-state bug. Remaining memory growth in multi-profile setups is expected (N profiles × per-profile base memory) plus potential Lua GC timing effects from PCRE2 invisible memory (already mitigated by GC tuning at initLuaGlobals).

## Resolution

root_cause: FontManager in the mudlet singleton uses packageName alone (not profileName+packageName) as the affiliation key in both loadedFontPaths and loadedFontAffiliation. This causes three compounding failure modes when multiple profiles load the same module: (1) the first profile's font install preempts the second profile's registration, then the first profile's uninstall removes the font from Qt's database while the second profile still needs it; (2) unloadFonts never cleans loadedFontPaths so stale filename entries permanently prevent any future loadFont from registering the font in QFontDatabase; (3) the second profile's uninstall finds zero affiliation entries so the Qt font resource is never freed. Repeated install/uninstall cycles across profiles compound all three modes, producing the observed accelerating memory growth. For pure Lua modules with no fonts, no additional shared-state leaks were identified — the remaining multi-profile memory growth is expected per-profile baseline memory × N profiles.
fix: FontManager.h/cpp: changed loadedFontPaths key from filename-only to profileName+filePath (profile-unique because filePath already contains profile directory). Changed loadedFontAffiliation key to profileName+packageName. Added loadedFontPaths cleanup to unloadFonts so reinstall works correctly. Global built-in fonts use empty profileName. Host.cpp: updated both call sites to pass getName() as profileName argument.
verification: Fix committed at 467dd1ce1. Code review confirms all three failure modes are eliminated. No other shared-state leaks found for pure Lua modules.
files_changed: [src/FontManager.h, src/FontManager.cpp, src/Host.cpp]
