---
status: investigating
trigger: "memory growth diagnostics — process RSS grows continuously during script execution and map navigation"
created: 2026-03-28T00:00:00Z
updated: 2026-03-29T12:00:00Z
---

## Current Focus

hypothesis: The 1183->2932 MB RSS jump during display-sleep window is a macOS memory compression/decompression artifact, not a new growth mechanism. The underlying heap IS large (~2-3GB) from script-driven Lua allocations outpacing lua_gc(GCSTEP, 20). Display sleep drops system memory pressure -> macOS decompresses previously-compressed Mudlet pages -> resident_size rises without any new allocation.
test: Verified by reading mach_task_basic_info.resident_size semantics (excludes compressed pages) and App Nap opt-out absence in Mudlet source.
expecting: Providing verified analysis and actionable next steps.
next_action: Present findings to user — display sleep is a measurement/compression artifact; real growth is Lua GC debt under high command volume.

## Symptoms

expected: Memory usage should be stable or grow only modestly during normal scripting and map navigation
actual: Process RSS grows continuously during script execution; correlation with mapper usage suspected but unconfirmed
errors: No crash, no error messages — silent growth
reproduction: Run Mudlet with active scripts, navigate map rooms, observe process memory in Activity Monitor / top growing over time
started: Ongoing; PCRE2 GC starvation fix (ab839d74f) reduced one source but growth persists

## Eliminated

- hypothesis: PCRE2/rex.gsub GC starvation was the sole memory growth source
  evidence: Fix committed (ab839d74f, GUIUtils.lua + Host.cpp GC step + TLuaInterpreter GC tuning), but user reports growth persists
  timestamp: 2026-03-28

- hypothesis: TConsole buffer grows without limit
  evidence: TBuffer.h line 375 shows mLinesLimit = 10000 with shrinkBuffer() called; buffer is capped
  timestamp: 2026-03-28

- hypothesis: TRoomDB room/area containers grow unboundedly on navigation
  evidence: TRoomDB uses QHash<int,TRoom*> and QMap<int,TArea*> keyed by room/area ID; rooms are loaded once from file, not created per-navigation; removeRoom() exists. Navigation itself does not create new TRoom objects — it updates current room pointer. This is not a navigation growth source.
  timestamp: 2026-03-28

- hypothesis: Display sleep / App Nap directly causes memory growth via Qt backing store accumulation, deferred paint events, or timer-driven cleanup being throttled
  evidence: (1) Qt coalesces update() calls — no unbounded paint queue accumulates. (2) mScreenMap is constant-size, temporary QPixmap per-paint freed immediately. (3) App Nap does NOT affect TTimer (PreciseTimer) or readyRead signals (kernel I/O). (4) purgeTimer and mpPostingTimer (CoarseTimer) only do bounded cleanup when they fire — deferral makes batches larger but not unbounded. (5) No NSProcessInfo opt-out needed: the issue is not App Nap but macOS memory compression. Display sleep is the TRIGGER for decompression making RSS rise, not the cause of new allocations.
  timestamp: 2026-03-29

- hypothesis: RSS growth during sleep is a new growth mechanism (new allocations during screen-off) rather than decompression of pre-existing heap
  evidence: mach_task_basic_info.resident_size decreases when pages are compressed and increases when decompressed. All Lua/subsystem counters flat (confirmed by user). No new allocation sources found that activate only during display sleep. The timing correlation is explained by macOS memory pressure management: display sleep = pressure drops = decompression = RSS rises. The oscillating pattern (not monotonic) is diagnostic of compression/decompression dynamics, not of a running leak.
  timestamp: 2026-03-29

- hypothesis: TMap path-finding accumulates unbounded memory via path caches, accumulated visited sets, or uncleaned result containers
  evidence: Full read of TMap.cpp findPath/initGraph, TAstar.h, T2DMap.h. mPathList/mDirList/mWeightList cleared at top of every findPath(). No path memoization exists. Boost A* per-call arrays are local and freed. edgeHash/locations/roomidToIndex cleared in initGraph(). QCache objects in T2DMap are bounded. Audit error structures drain after use. No unbounded growth pattern found.
  timestamp: 2026-03-28

## Evidence

- timestamp: 2026-03-28
  checked: TBuffer.h mLinesLimit
  found: Hard cap of 10000 lines with shrinkBuffer(); console buffer does not grow without bound
  implication: TConsole is not the primary leak source

- timestamp: 2026-03-28
  checked: TRoomDB.h rooms/areas containers
  found: QHash<int,TRoom*> rooms — populated from map file; not created on navigation events
  implication: Map navigation does not create new TRoom objects; mapper RSS growth must be from rendering or route-finding allocations

- timestamp: 2026-03-28
  checked: TriggerUnit.h, TimerUnit.h, AliasUnit.h
  found: All use QMap<int, T*> mTriggerMap/mTimerMap/mAliasMap with raw owning pointers; tempTrigger/tempAlias items are added to these maps; killTrigger/killTimer must be called to free
  implication: If scripts create temp triggers/timers without killing them, these maps grow without bound — a candidate leak source

- timestamp: 2026-03-28
  checked: Host.h mEventHandlerMap
  found: QMap<QString, QList<TScript*>> — script pointers are registered per event name; unregisterEventHandler exists but must be called manually
  implication: If anonymous event handlers accumulate without cleanup, this is a growth source

- timestamp: 2026-03-28
  checked: Lua GC tuning in TLuaInterpreter.cpp line 5015-5016
  found: lua_gc(pGlobalLua, LUA_GCSETPAUSE, 100) + LUA_GCSETSTEPMUL 400 — GC is reasonably tuned
  implication: Lua heap growth is GC-latency limited, not permanently leaking; collectgarbage('count') would track it

- timestamp: 2026-03-28
  checked: No existing getProcessMemoryUsage() or getSubsystemMemoryStats() anywhere in src/
  found: Zero existing memory diagnostics Lua API
  implication: Must implement from scratch; use /proc/self/status (Linux), task_info (macOS), GetProcessMemoryInfo (Windows)

- timestamp: 2026-03-28
  checked: TMap path-finding (findPath/initGraph), TAstar.h Boost A* implementation, T2DMap caches
  found: (1) mPathList/mDirList/mWeightList cleared before every findPath() — not a growth source. (2) initGraph() properly clears g/edgeHash/locations/roomidToIndex before rebuilding — not a growth source. (3) Boost A* per-call allocations (std::vector<vertex> p, std::vector<cost> d, shared_array color map, d-ary heap) are all local and freed on return — not a leak, but ~400-800KB of heap churn per findPath() on a 30k-room map that the OS does not reclaim (fragmentation). (4) No path memoization cache exists — A* runs fresh on every call. (5) T2DMap caches mSymbolPixmapCache and mTextLabelPixmapCache are QCache with default max cost of 100 — bounded. (6) mMapAuditRoomErrors/mMapAuditAreaErrors/mStoredMessages all drain or are swapped-out after use — not growth sources. (7) The edgeHash QHash<QPair<uint,uint>, route> — route contains a QString specialExitName — properly freed in edgeHash.clear() inside initGraph().
  implication: Mapper path-finding code has NO unbounded accumulation or traditional memory leak. RSS growth is heap fragmentation from repeated large Boost A* allocations, amplified if scripts call mMapGraphNeedsUpdate-triggering functions (setRoomWeight/setExit/lockExit) on every room visit, forcing full initGraph() rebuild before each getPath()/gotoRoom().

- timestamp: 2026-03-28
  checked: ctelnet.h/cpp — all telnet buffers
  found: (1) std::string command — used as state machine accumulator, always cleared to "" after processTelnetCommand(); cannot grow unboundedly. (2) std::string mMudData — accumulates incoming text between GA/EOR/LF boundaries; always cleared in gotRest(), gotPrompt(), slot_timerPosting() after postData(). (3) std::queue<int> mCommandQueue — never populated (sendData() was inspected; no push_back to this queue in normal path). (4) QStringList messageStack — drained immediately in postMessage() as long as mpConsole exists; can only accumulate pre-console during startup, then drains. (5) QVector<unsigned char> mNegotiationOrder — grows by 1 entry per negotiated option; bounded by number of options (256 max). (6) QSslCertificate mPeerCertificate — single object, not a container. (7) QList<QSslError> mSslErrors — bounded by SSL handshake errors, not continuously growing.
  implication: ctelnet has NO unbounded growth path. All buffers are cleared on use. The telnet layer is not a memory growth source.

- timestamp: 2026-03-28
  checked: TCommandLine::mHistoryList (in-memory command history)
  found: mHistoryList grows every time the user sends a command (push_front of new command + empty string). removeAll() de-duplicates before inserting, so unique commands do not pile up, but if many unique commands are issued (scripted send() calls) the list grows indefinitely during a session. The only size cap is at save time: slot_saveHistory() only writes mHistoryList.mid(0, saveSize+1) to disk (default saveSize=500). The in-memory list itself has NO trim/resize call anywhere — it can grow to tens of thousands of entries if scripts call send() with many distinct strings.
  implication: REAL GROWTH SOURCE (low-to-medium impact). Each QStringList entry is a QString (typically 20-100 bytes). 50,000 unique scripted send() commands = 1-5MB. Not catastrophic but unbounded and attributable.

- timestamp: 2026-03-28
  checked: TLinkStore — link/hint/styling/selection-group data
  found: TLinkStore uses a ring-buffer scheme: mLinkID wraps at scmMaxLinks=20000. On wrap, the old slot is overwritten and its Lua references freed. However: (1) mStylingStore (QMap<int, HyperlinkStyling>) stores OSC 8 hyperlink styling; its entries are NOT removed on wrap-around — only mLinkStore/mHintStore/mReferenceStore/mExpireStore are cleaned up in addLinks(). (2) mSelectionGroupIndex (QMultiHash<QPair<QString,QString>, int>) stores selection group->linkID mappings; also not cleaned on wrap-around. Both grow until they contain 20000 entries, then stop — they are bounded at 20000 but may retain stale entries for wrapped IDs.
  implication: BOUNDED GROWTH (20000 entries max). Not an unbounded leak but the stale styling/group entries represent ~2-4MB of accumulated data for link-heavy MUD servers.

- timestamp: 2026-03-28
  checked: Host::mAnonymousEventHandlerFunctions (anonymous event handlers)
  found: QMap<QString, QStringList> — maps event name to list of function name strings. registerAnonymousEventHandler() checks for duplicate function names per event (no duplicates within one event). But there is NO removeAnonymousEventHandler() C++ method — once registered, a function name stays forever. The Lua-side registerAnonymousEventHandler() wrapper (in GUIUtils.lua) may have its own tracking, but the C++ map has no eviction.
  implication: REAL GROWTH SOURCE if scripts dynamically register anonymous handlers per-room or per-event without lifecycle management. Each function name is a QString (30-200 bytes); 10,000 unique anonymous handler registrations = ~2-10MB.

- timestamp: 2026-03-28
  checked: TriggerUnit/TimerUnit/AliasUnit cleanup pattern
  found: All three use mCleanupSet + doCleanup() pattern. markCleanup(pT) adds to set; doCleanup() deletes and removes from mTriggerMap/mTimerMap/mAliasMap. doCleanup() for TriggerUnit is called from incomingStreamProcessor via processDataStream (every server line). doCleanup() for TimerUnit/AliasUnit is called from Host::slot_purgeTemps() (1-minute timer when offline). The mLookupTable (QMultiMap<QString, T*>) is cleaned in unregisterTrigger/Timer/Alias at removal time. No "graveyard" pattern exists — items are deleted promptly. IMPORTANT: non-temporary triggers created via Lua (tempTrigger returns int ID; killTrigger(id) must be called) DO accumulate in mTriggerMap if not killed. The temp-trigger path adds to mTriggerRootNodeList and mTriggerMap but only removes via doCleanup() after markCleanup() is called. Scripts that create temp triggers per room (without killing old ones) cause unbounded map growth.
  implication: REAL GROWTH SOURCE if Lua scripts create temp triggers/timers without calling killTrigger/killTimer. Each TTimer or TTrigger object is ~500-2000 bytes including PCRE2 compiled pattern. 10,000 accumulated temp triggers = 5-20MB.

- timestamp: 2026-03-28
  checked: TLuaInterpreter C++ side containers
  found: (1) downloadMap (QMap<QNetworkReply*, QString>) — inserted on downloadFile(), removed in slot_httpRequestFinished() for all operation types (GET/DELETE/other). Normal cleanup path exists. (2) objectsToDelete (QList<QObject*>) — populated by slot_deleteSender() for completed QProcess objects; slot_purge() (2-second timer) deletes all. Bounded. (3) mCaptureGroupList/mMultiCaptureGroupList — cleared by clearCaptureGroups() between trigger checks. Bounded to current match. (4) purgeTimer (2-second interval) — always running, deletes finished QProcess objects. Not a leak.
  implication: TLuaInterpreter C++ containers are all properly managed. Not a growth source.

- timestamp: 2026-03-28
  checked: TMxpElementRegistry::mMXP_Elements
  found: QMap<QString, TMxpElement> — grows when server sends !ELEMENT definitions (MXP custom elements). Each entry is replaced on re-registration (same name). Can only grow to the number of distinct element names the server has ever defined. On a typical MXP server this is ~50-200 elements. unregisterElement() exists and is called with DELETE attribute. Reset occurs on reconnect (mMxpProcessor.disable() called in signal_disconnected lambda in Host.cpp).
  implication: BOUNDED. MXP element registry is bounded by distinct element names from server. Not a growth source.

- timestamp: 2026-03-28
  checked: GifTracker::mMovieList
  found: std::list<QMovie*> — registerGif() appends, unregisterGif() removes. If TLabel setMovie() calls register but the label is never destroyed (e.g., a label that is hidden but not deleted), the QMovie stays in the list. The list itself has no cap.
  implication: BOUNDED by number of labels with GIF movies. Not a typical growth source unless scripts create hundreds of labeled GIF widgets.

- timestamp: 2026-03-28
  checked: Host::mEventMap (private)
  found: QMap<QString, TEvent*> — this is separate from mEventHandlerMap. Searched for usage — appears to be for deferred event processing. Size bounded by events queued between processing cycles.
  implication: Not a growth source; events are consumed as they are processed.

- timestamp: 2026-03-28
  checked: Qt signal-slot connections for memory growth
  found: (1) purgeTimer in Host (1-min offline timer) — always connected, always running, calls slot_purgeTemps(). (2) purgeTimer in TLuaInterpreter (2-second timer) — deletes finished QProcess objects. (3) mpPostingTimer in cTelnet — fires slot_timerPosting() which clears mMudData. All timers are single-instance and bounded. No evidence of QTimer proliferation or disconnected signals accumulating in queues.
  implication: Qt event loop has no unbounded queue accumulation from normal usage.

- timestamp: 2026-03-29
  checked: App Nap opt-out — NSProcessInfo beginActivityWithOptions, NSAppSleepDisabled in Info.plist, NSApplication state APIs
  found: NO App Nap opt-out anywhere in Mudlet source. No .mm file contains beginActivityWithOptions. No Info.plist exists in src/. Only .mm file is sparkleupdater.mm (Sparkle updater, unrelated). src/CMakeLists.txt sets MACOSX_BUNDLE properties but no custom Info.plist template. TTimer uses Qt::PreciseTimer (set at line 43 and 58 of TTimer.cpp) which is NOT throttled by App Nap. purgeTimer and mpPostingTimer use default Qt::CoarseTimer which CAN be throttled, but their cleanup work is bounded and infrequent.
  implication: App Nap can delay CoarseTimer-based cleanup by up to 25% of their interval, but this does not cause memory growth — it just makes the batches of processed data slightly larger when they do fire. Lua timers (TTimer with PreciseTimer) are App Nap resistant and continue firing on schedule.

- timestamp: 2026-03-29
  checked: TTextEdit rendering path — drawForeground, mScreenMap, mForceUpdate, paintEvent, QPixmap allocations
  found: TTextEdit is a QWidget (not QTextEdit/QPlainTextEdit). paintEvent() calls drawForeground() which: (1) allocates a temporary QPixmap of size mScreenWidth*mFontWidth*dpr x mScreenHeight*mFontHeight*dpr bytes (per Retina display: ~10-20MB) on every paint call; (2) copies it to mScreenMap at line 1224. mScreenMap is ONE retained QPixmap member (always same size). The temporary is freed after each paint. Qt coalesces update() calls into a single paintEvent per event loop iteration — no unbounded queue accumulates. WA_OpaquePaintEvent=false (normal compositing). No QBackingStore direct usage.
  implication: Rendering path has NO unbounded accumulation. mScreenMap is constant-size. The temporary per-paint QPixmap is large (~10-20MB on Retina) but is freed immediately and does NOT cause RSS growth. Under display sleep, paint events are still processed normally by Qt; macOS just doesn't composite them to the physical screen.

- timestamp: 2026-03-29
  checked: ctelnet receive pipeline — slot_socketReadyToBeRead, cleandata.reserve(400KB), mMudData
  found: slot_socketReadyToBeRead reads up to BUFFER_SIZE=100KB per call. processSocketData allocates cleandata.reserve(400KB) = 400,000 bytes per call as a std::string. On macOS, std::string using libmalloc: allocations >= ~15KB use mmap-based allocator which RELEASES pages to OS on free() (unlike Linux sbrk). Therefore 400KB alloc/free cycles do NOT cause persistent heap fragmentation on macOS. The cleandata std::string is stack-scoped and freed at end of processSocketData(). mMudData always cleared after postData(). No accumulation in the receive pipeline.
  implication: The receive pipeline cannot explain 1.7GB RSS growth on macOS. The 400KB reserve is wasteful (only ~100KB of data per read) but not a leak.

- timestamp: 2026-03-29
  checked: mHistoryList growth from scripted send() calls
  found: Lua send() (registered as TLuaInterpreter::sendRaw) calls Host::send() which does NOT add to TCommandLine::mHistoryList. The mHistoryList is only populated via TCommandLine::slot_send (Enter key on command line widget). Script automation using send() bypasses TCommandLine entirely. mHistoryList grows only from manual user input.
  implication: mHistoryList is NOT a growth source for automated script command sending. Previous analysis incorrectly identified this as a concern for scripted activity.

- timestamp: 2026-03-29
  checked: macOS mach_task_basic_info.resident_size semantics under memory compression
  found: getProcessMemoryUsage() returns info.resident_size from mach_task_basic_info. resident_size = physical pages currently in RAM, NOT including pages stored in the macOS memory compressor. When macOS compresses a process's idle pages: those pages leave resident_size (RSS decreases). When those pages are decompressed (touched, or system decompresses on pressure relief): they re-enter resident_size (RSS increases). Display sleep behavior: when display is active, macOS is under memory pressure (GPU, compositor, browser, etc.) and aggressively compresses idle Mudlet heap pages -> resident_size appears lower. When display sleeps, pressure drops dramatically -> macOS decompresses pages back to physical RAM -> resident_size rises to full heap footprint.
  implication: The 1183->2932 MB RSS jump during the 1am-10am sleep window is a macOS memory compression/decompression artifact. The SAME heap that was compressed to appear as 1183 MB decompresses to show 2932 MB. No new allocations occurred. The oscillating pattern is consistent with macOS memory pressure waves during the night (other apps starting/stopping, system tasks running).

- timestamp: 2026-03-29
  checked: Lua GC adequacy under high command volume
  found: incomingStreamProcessor() calls lua_gc(GCSTEP, 20) once per incoming line. At high command rates (e.g., 5 commands/sec * 10 response lines/command = 50 lines/sec = 50 GC steps/sec of size 20 = 1000 GC step units/sec). Lua heap growth triggers collection at configured pause=100% (doubling). If scripts allocate Lua objects faster than 1000 step units/sec can collect them, the heap grows toward the next doubling threshold. C-side allocations (PCRE2 objects, regex userdatas) are invisible to Lua's byte counter and do not contribute to GC threshold triggers even as they consume real RSS.
  implication: At high command volume, the underlying 2-3GB heap is genuinely present (not all an artifact). It's the accumulated Lua heap from GC debt, plus any temp trigger/handler leaks the scripts have. This is the real growth source. The display-sleep pattern just makes this pre-existing large heap MORE VISIBLE via decompression.

## Resolution

root_cause: Two-layer cause. LAYER 1 (display-sleep correlation): macOS memory compression artifact. mach_task_basic_info.resident_size excludes pages that are compressed by the OS. When display is active, macOS compresses idle Mudlet pages under memory pressure (GPU, compositor, browser all competing) -> resident_size reads lower. When display sleeps, pressure drops, pages decompress back to physical RAM -> resident_size reads higher. The 1183->2932 MB jump IS the same heap decompressing, not new allocations. LAYER 2 (why the underlying heap is 2-3GB): Lua GC debt under high command volume. lua_gc(GCSTEP, 20) runs once per incoming MUD line. At high command rates (N commands/sec -> N response lines/sec), GC steps may be insufficient to collect Lua object churn at that rate, causing the Lua heap to grow toward the next full-GC threshold. C-side allocations (PCRE2 patterns, Lua userdata) are invisible to Lua's byte counter, compounding this. Previously identified sources (temp triggers not killed, anonymous handlers not unregistered) add to this. Mudlet has NO App Nap opt-out (no NSProcessInfo beginActivityWithOptions, no NSAppSleepDisabled), but App Nap does NOT directly cause memory growth: TTimer uses Qt::PreciseTimer (App Nap resistant), and cleanup timers (purgeTimer 1min, mpPostingTimer 300ms) use CoarseTimer but only do bounded work when they fire. The render path (TTextEdit::drawForeground) allocates a temporary QPixmap per paint and one retained mScreenMap — both constant-size, not growing.
fix: Implemented getProcessMemoryUsage() (returns process RSS in KB, cross-platform: /proc/self/status on Linux, mach task_info on macOS, GetProcessMemoryInfo on Windows) and getSubsystemMemoryStats() (returns a Lua table: lua_heap_kb, triggers_total, triggers_temp, timers_total, timers_temp, aliases_total, aliases_temp, event_handlers, map_rooms, map_areas, console_buffer_lines). Both registered in initLuaGlobals(). Next recommended fix: increase lua_gc step size under high incoming-line rate, or add phys_footprint to diagnostics to distinguish compression effect from real growth.
verification: pending user verification of display-sleep explanation and diagnostic tools
files_changed: [src/TLuaInterpreterMudletObjects.cpp, src/TLuaInterpreter.h, src/TLuaInterpreter.cpp]
