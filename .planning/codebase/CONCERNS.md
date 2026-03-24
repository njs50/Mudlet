# Codebase Concerns

**Analysis Date:** 2026-03-24

## Tech Debt

**Unused/Deprecated TArea Members:**
- Issue: Two member variables (`pos` and `span` of type `QVector3D`) in `TArea` marked with TODO comment indicating they "have not been used for some time"
- Files: `src/TArea.h:80-83`
- Impact: Dead code adds cognitive overhead; unclear if safe to remove due to serialization concerns
- Fix approach: Audit serialization code to determine if these fields are persisted in save files; if not, safely remove; if yes, deprecate gracefully across version boundary

**IRE Driver Bugfix Setting (Obsolete):**
- Issue: Legacy `USE_IRE_DRIVER_BUGFIX` setting for handling IRE game client compatibility appears historical
- Files: `src/Host.h:314-318`, `src/Host.h:652`, `src/dlgProfilePreferences.cpp:454,579,842,1564,2991`
- Impact: Settings dialog includes checkbox that may no longer be relevant; adds maintenance burden
- Fix approach: Research whether IRE games still require this workaround; if not, deprecate setting and remove from UI; migrate existing profiles gracefully

**Incomplete Implementation in dlgTriggerEditor:**
- Issue: Multiple FIXME comments indicate incomplete features and design inconsistencies
- Files: `src/dlgTriggerEditor.h:671`, `src/dlgTriggerEditor.cpp:5080,11471,12152`
- Impact: Pattern-matching and view handling logic has known limitations that make future changes difficult
- Fix approach: Prioritize completing pattern-matching options and refactor view handling switch statements to support all editor view types consistently

## Known Bugs / Issues

**Trigger Pattern Compilation Locking:**
- Issue: Comment in code states "FIXME: lock if code *OR* regex doesn't compile" - currently only locks on compile failure, not on regex validation failure
- Files: `src/TTrigger.cpp:113-114`
- Impact: Malformed regex patterns may be accepted, causing runtime failures when trigger fires
- Fix approach: Add validation to lock trigger if regex compilation fails, not just Lua code compilation

**Trigger Setup Race Condition:**
- Issue: Code comment "FIXME: ronny managed to trigger this somehow" indicates possible race where `patternKinds.size() != patterns.size()`
- Files: `src/TTrigger.cpp:140-142`
- Impact: Potential crash or undefined behavior when pattern/kind count mismatch occurs
- Fix approach: Add defensive checks to prevent this mismatch and better logging to understand trigger path

**TAction Script Execution Initialization:**
- Issue: Comment indicates incomplete logic: "FIXME: only run checkbox button scripts, but run them even if unchecked"
- Files: `src/TAction.cpp:281-285`
- Impact: Push-down button script execution behavior during profile load may not match user intent
- Fix approach: Clarify and document intended behavior for pushdown buttons during profile loading sequence

**Telnet Output Processing Redraw:**
- Issue: "FIXME: rewrite: if lines above the current line get deleted -> redraw clean slice" indicates incomplete miniconsole output handling
- Files: `src/TMainConsole.cpp:1413`
- Impact: Miniconsole display may show artifacts when rapid line deletion occurs
- Fix approach: Implement proper dirty region tracking for miniconsole redraw optimization

## Code Organization Issues

**Monolithic dlgTriggerEditor:**
- Issue: Dialog editor is 14,992 lines in single `.cpp` file (5th largest file in codebase)
- Files: `src/dlgTriggerEditor.cpp`, `src/dlgTriggerEditor.h:804`
- Impact: Extremely difficult to navigate, modify, or test; violates separation of concerns with triggers, aliases, timers, scripts, actions, keys, and variables all in one dialog
- Fix approach: Extract each item type editor into separate classes; create factory pattern to manage item-specific UI areas; establish clear interfaces between editor sections

**Large monolithic Header class (1,065 lines):**
- Issue: Central Host class has grown to contain telnet protocol, Lua interpreter, multiple UI components, profile state, networking, and scripting systems
- Files: `src/Host.h:1065`
- Impact: Circular dependencies, difficult to test individual features, high cognitive load, slow compilation
- Fix approach: Extract subsystems into separate manager classes (NetworkManager, ScriptingManager, etc.); use dependency injection to reduce coupling

**Large Lua Interpreter (912 line header, 8,278 line implementation):**
- Issue: TLuaInterpreter is split into main file plus separate interpreter modules (UI, mapper, networking, etc.) totaling significant code complexity
- Files: `src/TLuaInterpreter.h:912`, `src/TLuaInterpreter.cpp:8278`, plus `TLuaInterpreter*.cpp` variants
- Impact: Difficult to add new API functions, understand scope of changes, or maintain API consistency
- Fix approach: Consider code generation from API specification to reduce manual binding overhead

## Memory Management Concerns

**Manual Memory Deletion in Collections:**
- Issue: Multiple manual `delete` calls for items removed from collections, particularly `TColorTable` pointers and condition map entries
- Files: `src/TTrigger.cpp:76,82,133`, `src/TTrigger.h:1` (destructor comments reference manual cleanup)
- Impact: Potential for use-after-free or double-delete if cleanup order is wrong; Qt parent-child system not fully leveraged
- Fix approach: Migrate to Qt parent-child ownership model or C++ smart pointers for non-Qt objects

**Excessive new/delete Usage (1,370 occurrences):**
- Issue: Frequent use of `new` (1,370 occurrences) vs `delete` (214 occurrences) - asymmetric count suggests reliance on Qt's parent-child cleanup
- Files: Across src/ directory - most notably `src/dlgTriggerEditor.cpp:284+`, `src/TAction.cpp:183+`, `src/mudlet.cpp:134+ usages`
- Impact: If Qt parent relationships are not set correctly, memory leaks can occur; difficult to guarantee cleanup order
- Fix approach: Audit new/delete pairs; prefer Qt smart pointers (QPointer, QScopedPointer); add assertions to verify parent relationships at construction time

## Scaling Limits

**Map Processing Performance:**
- Issue: T2DMap contains optimization comments about avoiding "very slow loop" as map gets large
- Files: `src/T2DMap.cpp:898,906`
- Impact: Large maps (thousands of rooms) may experience significant slowdown in coordinate calculation and collision detection
- Fix approach: Profile large map operations; consider spatial indexing structures (quadtree/octree) for coordinate lookups; cache computed span values

**Lua Condition Evaluation in Triggers:**
- Issue: Triggers with Lua conditions evaluate full condition map for each line of input
- Files: `src/TTrigger.h:1` (mConditionMap), `src/TTrigger.cpp:81-83` (cleanup)
- Impact: Profiles with hundreds of complex triggers may have response latency
- Fix approach: Consider lazy evaluation or just-in-time compilation of Lua conditions; add performance profiling to identify hotspots

**Undo/Redo Stack Memory:**
- Issue: Editor undo stack stores full copies of items for all modifications across all editor sections
- Files: `src/EditorUndoStack.cpp`, `src/EditorUndoStack.h`
- Impact: Long editing sessions with many changes can consume significant memory
- Fix approach: Implement command-pattern with delta storage rather than full snapshots; add memory limit with oldest commands pruned

## Single-Threaded Constraints

**No Explicit Threading Model:**
- Issue: Single-threaded architecture by design - all profiles, triggers, Lua engine run on main thread
- Files: Architecture-wide; documented in project instructions
- Impact: Cannot benefit from multi-core systems; network blocking can freeze UI; heavy scripting blocks input processing
- Mitigation: This is architectural by design - not easily changeable; document clearly in relevant classes

**Network I/O in Main Thread:**
- Issue: Networking operations use Qt's async model but are coordinated on main thread
- Files: `src/ctelnet.cpp:5304 lines`, `src/ctelnet.h:497`
- Impact: Large data transfers (large maps, logs) can cause UI responsiveness issues
- Fix approach: Evaluate moving network buffer processing to worker thread with thread-safe queue to main thread

## Security Considerations

**Password Field Handling:**
- Issue: Command line can contain user passwords; masking can be disabled via profile setting
- Files: `src/Host.h:510` (mDisablePasswordMasking), `src/TCommandLine.cpp:1706`
- Impact: If logging is enabled, passwords may be stored in plaintext; if screen captured, passwords visible
- Recommendations: Audit password logging to ensure passwords are never written to log files; consider using Qt's password echo mode exclusively

**String Format Conversions:**
- Issue: Heavy use of QString::arg() with user data and file paths - generally safe but requires vigilance
- Files: Throughout codebase - `src/TAction.cpp:123`, `src/TDebug.h:1`
- Impact: No detected format string vulnerabilities but code quality depends on developer discipline
- Recommendations: Continue current practice of using QString methods; add static analysis rules to prevent mixing with C-style formatting

**Lua Sandbox Gaps:**
- Issue: Lua scripting has access to Host internals; no sandboxing of scripting environment
- Files: `src/TLuaInterpreter.cpp:8278`, `src/LuaInterface.cpp`
- Impact: Malicious or buggy scripts can crash application, leak memory, or corrupt profile data
- Recommendations: Document assumptions about script trustworthiness; audit dangerous API functions; consider capability-based restrictions for untrusted packages

## Fragile Areas

**MXP Processing Pipeline:**
- Issue: MXP frame handling has multiple tag handlers and context managers with interdependencies
- Files: `src/TMxpProcessor.cpp:346`, `src/TMxpFrameManager.h/cpp`, multiple `TMxpXxxTagHandler` files
- Why fragile: Complex state machine across multiple handler classes; one incorrect state transition breaks frame parsing; context manager has implicit assumptions about handler ordering
- Safe modification: Add comprehensive unit tests for frame state transitions before refactoring; establish clear frame state documentation; use visitor pattern for tag handling
- Test coverage: MXP tests exist but focused on specific tags; integration tests for frame state lifecycle are limited

**Trigger Pattern Matching:**
- Issue: Triggers support multiple pattern types (plain, regex, Lua patterns, color patterns) with complex evaluation logic
- Files: `src/TTrigger.cpp:2462 lines`, `src/TTrigger.h:326`
- Why fragile: Pattern kind enum used as index into collections; color patterns stored separately in TColorTable list; pattern validation incomplete
- Safe modification: Add invariant checks to ensure patternKinds.size() == patterns.size() at all mutation points; migrate color patterns to unified structure; add property-based tests for pattern types
- Test coverage: No functional tests for pattern evaluation; only unit tests for regex compilation

**Map Serialization and Restore:**
- Issue: Map saving/loading with complex JSON structures involving rooms, areas, labels, and user data
- Files: `src/TArea.cpp`, `src/TMap.cpp:3596`, `src/TRoom.cpp:2462`, JSON read/write methods throughout
- Why fragile: Schema not versioned in code; old file format migration undocumented; large coordinate space with collision handling
- Safe modification: Establish versioned schema; add explicit version checks before loading; test migration from previous versions; add round-trip tests
- Test coverage: Serialization tested but not version migration or corruption recovery

**Editor Item Command Undo/Redo:**
- Issue: Complex undo/redo with special handling for ID remapping, hierarchy changes, and item property updates
- Files: `src/EditorDeleteItemCommand.cpp:842 lines`, `src/EditorAddItemCommand.cpp`, `src/EditorMoveItemCommand.cpp`, `src/EditorModifyPropertyCommand.cpp`
- Why fragile: 40+ DEBUG_UNDO_REDO conditional debug statements indicate uncertainty in implementation; ID remapping logic touches multiple classes
- Safe modification: Remove DEBUG_UNDO_REDO conditions and make logging always-on (configurable at runtime); add invariant checks for ID maps after each command; test with large hierarchies
- Test coverage: Undo/redo unit tests exist in `test/dlgTriggerEditorUndoRedoTest.cpp` but limited to trigger editing

## Testing Gaps

**Integration Testing of Profile Loading:**
- Issue: Profile startup involves multiple phases and coordination between telnet, Lua, triggers, timers - critical path with no integration tests
- Files: `src/Host.cpp` (profile loading), `src/mudlet.cpp` (profile creation), `src/TLuaInterpreter.cpp` (script compilation)
- Risk: Changes to loading sequence can break profiles silently; startup errors may not appear until user loads specific profile
- Recommendations: Create test profiles with various configurations; test loading/unloading cycles; verify script state after profile switch

**MXP/Protocol Handling Edge Cases:**
- Issue: Limited testing of edge cases in telnet protocol negotiation and MXP processing
- Files: `test/functional_tests/` has some tests, but telnet negotiation undercovered
- Risk: Malformed server responses or unusual protocol sequences could crash client
- Recommendations: Add property-based tests for telnet protocol state machine; fuzz test MXP tag parsing

**Performance Regression Testing:**
- Issue: No automated performance tests or benchmarks in CI
- Files: `test/functional_tests/TelnetBenchmark.cpp` exists but not integrated into CI
- Risk: Performance regressions silently introduced; profile with thousands of triggers may slow over time
- Recommendations: Set up performance benchmarks in CI; establish baselines for trigger evaluation, map rendering, and Lua compilation

## Performance Bottlenecks

**String Allocation in Hot Paths:**
- Issue: Frequent QString allocations in trigger evaluation and text processing
- Files: `src/TBuffer.cpp:7177`, `src/TTextEdit.cpp:3696`, telnet protocol handling
- Cause: Every text chunk processed allocates new QStrings rather than reusing buffers
- Improvement path: Profile text processing with large MUD sessions (thousands of lines); consider string pooling or buffer reuse for network data

**Lua Function Lookup:**
- Issue: TLuaInterpreter uses function lookup during trigger/timer/alias execution
- Files: `src/TLuaInterpreter.cpp:8278`
- Cause: Hash table lookups for every trigger fire; Lua code compilation happens at runtime
- Improvement path: Consider compiling frequently-used triggers to bytecode; cache compiled functions; profile with 100+ active triggers

**2D Map Rendering:**
- Issue: T2DMap rendering may redraw large areas unnecessarily
- Files: `src/T2DMap.cpp:6059`, `src/glwidget.cpp:2196`, `src/modern_glwidget.cpp:2196`
- Cause: Coordinate recalculation on every render; collision detection for every visible room
- Improvement path: Implement dirty region tracking; cache room coordinate calculations; profile with large visible areas at zoom level

## Dependencies at Risk

**Edbee Text Editor (Embedded):**
- Risk: `src/edbee/` is third-party component bundled in repo; maintenance burden if upstream stops updating
- Impact: Code completion, syntax highlighting, undo/redo all depend on Edbee
- Migration plan: Evaluate Qt's QPlainTextEdit with custom extensions; fork Edbee if needed for security updates

**Qt 6 Minimum (6.8.2):**
- Risk: Project requires recent Qt 6 - incompatible with LTS version 5.x; Qt 6 still evolving
- Impact: Developers must maintain recent toolchain; deployment to older systems limited
- Mitigation: Document Qt version requirements prominently; establish CI for multiple Qt 6 versions; monitor for critical Qt 6 regressions

**Third-party Dependencies Age:**
- Risk: Several vendored dependencies in `3rdparty/` with unclear maintenance status
- Impact: Security vulnerabilities in bundled libraries; duplicated code if upstream is imported
- Recommendations: Audit all `3rdparty/` components for known CVEs; establish policy for version updates; consider package manager integration for critical libs

## Missing Critical Features

**No Package/Module Dependency Resolution:**
- Problem: Packages can have dependencies but no automatic resolution or conflict detection
- Blocks: Users cannot safely share profiles with dependencies; installation order matters
- Work needed: Dependency graph with topological sort; conflict detection; auto-installation; version pinning

**Profile Data Backup/Recovery:**
- Problem: No built-in backup system; profile corruption loses all work
- Blocks: Users lose profiles to crashes or corruption with no recovery mechanism
- Work needed: Automatic snapshots; restore points; corruption detection; backup management UI

**Performance Profiling Tools:**
- Problem: No built-in profiler for Lua or trigger execution; users cannot diagnose slow scripts
- Blocks: Users with performance issues cannot identify bottlenecks
- Work needed: Per-function call timing; trigger fire frequency; memory allocation tracking; UI for visualization

---

*Concerns audit: 2026-03-24*
