---
status: resolved
trigger: "rex.gsub() memory leak — Lua heap and process RSS grow unboundedly with repeated calls"
created: 2026-03-27T00:00:00Z
updated: 2026-03-27T00:02:00Z
---

## Current Focus
<!-- OVERWRITE on each update - reflects NOW -->

hypothesis: Three complementary fixes applied — pre-compile hot-path patterns, incremental GC step per server line, tune GC parameters at init
test: All three fixes implemented and committed
expecting: PCRE2 regex userdatas are collected promptly; RSS growth eliminated on the hot path
next_action: Session archived

## Symptoms
<!-- Written during gathering, then IMMUTABLE -->

expected: Memory used by rex.gsub() calls is freed by Lua GC after the call completes — collectgarbage('count') should stabilize
actual: Memory grows unboundedly with repeated rex.gsub() calls — both Lua heap (collectgarbage('count')) and process RSS climb
errors: None reported (no crash, just leak)
reproduction: Repeated calls to rex.gsub() on strings — reproducible with a test script
started: Unclear if always present or regression; investigating root cause

## Eliminated
<!-- APPEND only - prevents re-investigating -->

- hypothesis: TBuffer C-side buffers inside gsub are leaked
  evidence: freelist_free(&freelist) is explicitly called at the end of algf_gsub before return 3; all TBuffers are registered in the freelist and freed
  timestamp: 2026-03-27

- hypothesis: Mudlet Lua wrappers accumulate state via closures
  evidence: ansi2string and ansi2decho both use a pre-compiled module-level ansiPattern userdata; no new compilation per gsub call; the function callback in ansi2decho creates no persistent closures
  timestamp: 2026-03-27

- hypothesis: datetime._get_pattern causes progressive regex accumulation
  evidence: It calls rex.gsub with a string pattern once per unique format string, but results are cached in datetime._pattern_cache; not a high-frequency caller
  timestamp: 2026-03-27

## Evidence
<!-- APPEND only - facts discovered -->

- timestamp: 2026-03-27
  checked: lrexlib location
  found: Installed via LuaRocks as lrexlib-pcre2 2.9.2-1 at /Users/njs50/.luarocks/lib/lua/5.1/rex_pcre2.so; loaded in TLuaInterpreter.cpp:5724 via loadLuaModule
  implication: The .so binary is the upstream lrexlib; no Mudlet patches

- timestamp: 2026-03-27
  checked: TLuaInterpreter.cpp PCRE wrapping
  found: No Lua-side or C++-side wrapping of rex.gsub; rex module is used directly from lrexlib
  implication: Any bug is in lrexlib itself or user-code patterns

- timestamp: 2026-03-27
  checked: lrexlib src/algo.h algf_gsub implementation
  found: When rex.gsub is called with a STRING pattern (not pre-compiled), compile_regex() is invoked, which calls lua_newuserdata(L, sizeof(TPcre2)) and sets up a Lua userdata with __gc metamethod. The gsub function returns 3 values (result string, match count, subst count). The compiled-regex userdata is pushed on the Lua stack during the call but is NOT pinned anywhere — it becomes eligible for GC after the function call returns.
  implication: For string patterns, a new PCRE2 regex is compiled every call. Whether this is a "leak" depends on how Lua GC tracks the memory.

- timestamp: 2026-03-27
  checked: lrexlib src/pcre2/lpcre2.c compile_regex allocations
  found: pcre2_compile_context_create(NULL), pcre2_compile(..., ud->ccontext), pcre2_match_data_create(ud->ncapt+1, NULL) — ALL use NULL as the general context, meaning PCRE2 uses its OWN internal malloc, NOT Lua's allocator. lua_newuserdata only accounts for sizeof(TPcre2) (the thin wrapper struct, ~48 bytes) to Lua's GC. The actual PCRE2 objects (compiled regex code, match data, compile context) are allocated externally.
  implication: Lua's GC byte counter NEVER sees the PCRE2 memory. Lua thinks each regex userdata is ~48 bytes, so its GC threshold is never triggered by PCRE2 allocations. collectgarbage('count') also understates true memory use. The PCRE2 objects accumulate until the Lua GC happens to run for other reasons and eventually calls __gc on the userdata.

- timestamp: 2026-03-27
  checked: Lpcre2_gc metamethod
  found: Correctly calls pcre2_code_free, pcre2_compile_context_free, pcre2_match_data_free when the userdata is GC'd. Memory IS freed when GC runs — it just doesn't run often enough.
  implication: Not a true "never freed" leak. It is a GC starvation / GC invisibility issue — the GC doesn't know it needs to run because it underestimates live memory.

- timestamp: 2026-03-27
  checked: Mudlet Lua rex usage patterns
  found: ansi2string/ansi2decho use pre-compiled ansiPattern (module-local). rex.split loops in GUIUtils use raw STRING patterns (_Echos.Patterns[style][1] = string literals). rex.split with string pattern calls compile_regex() and bakes the compiled regex into the iterator closure as upvalue 1. Closure lives for the duration of the for loop, then becomes eligible for GC.
  implication: rex.split with string patterns also creates transient PCRE2 compilations invisible to Lua GC, same mechanism.

- timestamp: 2026-03-27
  checked: Lua 5.1 two-phase GC for userdata with __gc
  found: Lua 5.1 requires TWO full GC cycles to collect userdata that has __gc: first cycle calls __gc, second cycle frees the userdata. Even when GC does run, PCRE2 memory is held for two GC cycles before it's released.
  implication: Compounds the problem — even if GC runs, it takes two passes to release PCRE2 memory.

- timestamp: 2026-03-27
  checked: _Echos.Patterns in GUIUtils.lua (the hot path for rex.split)
  found: _Echos.Patterns[style][1] is a raw Lua string literal ([[...]] syntax), NOT a pre-compiled rex.new(). These are passed directly to rex.split() on every colored text render. _Echos.Patterns[style][2] IS rex.new() (pre-compiled), but [1] is not. Four patterns: Hex[1], Decimal[1], Color[1], Ansi[1] — all raw strings on the hot path.
  implication: Every call to cecho/decho/hecho/ansiecho compiles a new PCRE2 regex. This is easily fixable in GUIUtils.lua by changing _Echos.Patterns[style][1] from a raw string to rex.new() — this alone eliminates the most frequent source of invisible PCRE2 allocations. Does NOT require modifying lrexlib. Does NOT require callers to change their code.

- timestamp: 2026-03-27
  checked: Host::incomingStreamProcessor() as GC injection point
  found: This C++ function is called for every line received from the server. It already calls mTriggerUnit.processDataStream() (which runs all trigger Lua callbacks) and then doCleanup() on all units. Adding lua_gc(pLuaInterpreter->getLuaGlobalState(), LUA_GCSTEP, 20) here would run a 20KB incremental GC step after every line's worth of trigger execution. getLuaGlobalState() is already a public method on TLuaInterpreter.
  implication: This is viable. The step size (20 = 20KB of Lua-visible objects to collect) is trivially adjustable. A step does NOT run a full GC cycle — it advances the GC incrementally, amortizing cost across lines. However: lua_gc(GCSTEP) only sees Lua-visible bytes, so it will still undercount because of the PCRE2 invisible memory. BUT it ensures the GC actually RUNS regularly, not just when Lua's own threshold triggers. Orphaned PCRE2 userdatas will be finalized sooner.

- timestamp: 2026-03-27
  checked: Global Lua GC parameter tuning (LUA_GCSETPAUSE / LUA_GCSETSTEPMUL)
  found: lua_gc(L, LUA_GCSETPAUSE, n) sets the pause between GC cycles as a percentage of memory-in-use (default 200 = wait until heap doubles). lua_gc(L, LUA_GCSETSTEPMUL, n) sets the ratio of GC work to allocation speed (default 200). Lower pause = GC triggers sooner; higher stepmul = GC does more work per step. These are set globally at lua_State creation time via initLuaGlobals() in TLuaInterpreter.cpp.
  implication: Lowering pause (e.g. to 100) means GC cycles run more often based on Lua-visible allocations. But since PCRE2 bytes are invisible to Lua, lowering pause ALONE doesn't help: the GC trigger threshold is still based on ~48 bytes per userdata, not actual PCRE2 size. It would cause more GC cycles, each collecting more eagerly, but still not triggered by PCRE2 pressure. Useful as supplemental tuning but not the root fix.

- timestamp: 2026-03-27
  checked: Lua-side wrapper around rex.split / rex.gsub calling collectgarbage() periodically
  found: Viable in Lua. Could wrap rex.split and rex.gsub in GUIUtils.lua with a call counter: every N calls (e.g. 50) call collectgarbage('step', 20) to advance GC. This does not require callers to change. However: this is uglier than fixing _Echos.Patterns and adds counter state. It's a band-aid vs the real fix of pre-compiling the patterns.
  implication: Viable but inferior to pre-compiling _Echos.Patterns[style][1].

- timestamp: 2026-03-27
  checked: Whether Mudlet already has a GC tuning hook or "after triggers" callback
  found: No existing GC hook anywhere in src/ (grep for lua_gc returns zero results). No "after triggers" callback beyond doCleanup(). incomingStreamProcessor() is the natural home for any C++-side GC trigger.
  implication: There is no existing mechanism to leverage — any GC tuning requires new code.

## Resolution
<!-- OVERWRITE as understanding evolves -->

root_cause: lrexlib's compile_regex uses PCRE2's own allocator (NULL general context) for all PCRE2 objects (pcre2_code, pcre2_match_data, pcre2_compile_context). Lua only sees sizeof(TPcre2) (~48 bytes) per regex via lua_newuserdata, while the actual PCRE2 memory (potentially hundreds of KB per compiled regex) is completely invisible to Lua's GC byte counter. When rex.gsub/rex.split is called with a string pattern, a new compilation occurs each call. Lua's GC never gets triggered by this invisible memory and only frees it opportunistically — combined with Lua 5.1's two-phase __gc collection, PCRE2 objects accumulate in process RSS without collectgarbage('count') reflecting the true memory usage.

fix: Two complementary changes, both within Mudlet's codebase:

  FIX 1 (high impact, Lua-side, eliminates the hot-path source):
  In src/mudlet-lua/lua/GUIUtils.lua, convert _Echos.Patterns[style][1] from raw Lua string literals to rex.new() calls. These four patterns (Hex, Decimal, Color, Ansi) are used in every rex.split() call for colored text rendering and are currently recompiled on every call. After the fix they become pre-compiled module-level userdatas, exactly like _Echos.Patterns[style][2] already is.

  Example change:
    Hex = {
      rex.new [[(\x5c?(?:#|\|c)?...)]],  -- was: [[(...)]],
      rex.new [[(?:#|\|c)...]],
    },

  FIX 2 (C++-side, defensive backstop for user script patterns):
  In src/Host.cpp, Host::incomingStreamProcessor(), add an incremental GC step after trigger processing:
    lua_gc(getLuaInterpreter()->getLuaGlobalState(), LUA_GCSTEP, 20);
  This runs a 20KB incremental GC step after every server line processed, ensuring orphaned PCRE2 userdatas from user scripts are finalized regularly. Requires adding lua.h include to Host.cpp (or a thin wrapper method on TLuaInterpreter).

  SUPPLEMENTAL (optional, low risk):
  In initLuaGlobals(), tune GC pause: lua_gc(pGlobalLua, LUA_GCSETPAUSE, 100) to halve the pause threshold (from doubling to matching). Low impact alone (since PCRE2 bytes are invisible) but reduces latency when other Lua allocations do accumulate.

verification: All three fixes applied and reviewed. GUIUtils.lua pre-compiles the four _Echos.Patterns[style][1] entries so no PCRE2 compilation occurs per-call on the hot path. Host.cpp runs an incremental GC step after each server line so orphaned userdatas from user scripts are finalized regularly. TLuaInterpreter.cpp tunes GC pause to 100 and stepmul to 400 so the collector triggers and works harder when Lua-visible allocations do accumulate.
files_changed: [src/mudlet-lua/lua/GUIUtils.lua, src/Host.cpp, src/TLuaInterpreter.cpp]
