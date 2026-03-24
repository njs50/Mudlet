# GSD Debug Knowledge Base

Resolved debug sessions. Used by `gsd-debugger` to surface known-pattern hypotheses at the start of new investigations.

---

## rex-gsub-memory-leak — PCRE2 regex GC starvation from lrexlib invisible allocations
- **Date:** 2026-03-27
- **Error patterns:** rex.gsub, rex.split, memory leak, RSS, collectgarbage, PCRE2, lrexlib, lua heap, unbounded growth
- **Root cause:** lrexlib allocates PCRE2 objects (pcre2_code, match_data, compile context) via PCRE2's own allocator (NULL general context), making them invisible to Lua's GC byte counter. Lua only sees ~48 bytes per regex userdata, so its GC threshold is never triggered by PCRE2 allocations. When rex.gsub/rex.split is called with a string pattern (not pre-compiled), a new PCRE2 compilation occurs each call. Combined with Lua 5.1's two-phase __gc for userdata, orphaned regex objects accumulate in process RSS without collectgarbage('count') reflecting the true memory usage.
- **Fix:** Three complementary changes: (1) In GUIUtils.lua, convert _Echos.Patterns[style][1] from raw string literals to rex.new() at module load time so the four hot-path patterns (Hex, Decimal, Color, Ansi) are never recompiled per-call. (2) In Host::incomingStreamProcessor(), add lua_gc(LUA_GCSTEP, 20) after doCleanup() calls to run an incremental GC step after every server line. (3) In TLuaInterpreter::initLuaGlobals(), set LUA_GCSETPAUSE=100 and LUA_GCSETSTEPMUL=400 after newstate() to make the GC trigger sooner and work harder per step.
- **Files changed:** src/mudlet-lua/lua/GUIUtils.lua, src/Host.cpp, src/TLuaInterpreter.cpp
---

## tmediaplayer-null-audiooutput-segfault — null QAudioOutput dereference in TMediaPlayer::setVolume and TMedia::play
- **Date:** 2026-03-28
- **Error patterns:** segfault, SEGV, QAudioOutput, setVolume, setMuted, null pointer, audioOutput, TMediaPlayer, TMedia, playSoundFile, 0x10, re-entrancy, sysMediaStarted, sysMediaFinished
- **Root cause:** TMediaPlayer::setVolume() called mMediaPlayer->audioOutput()->setVolume() without checking if audioOutput() returned null. TMedia::play() called pPlayer->mediaPlayer()->audioOutput()->setMuted() with the same missing guard. QMediaPlayer::audioOutput() can return null when re-entrant playSoundFile() calls (triggered via sysMediaStarted/sysMediaFinished Lua events emitted synchronously during stop()/play() state transitions) put the player list into an inconsistent state. Crash address 0x10 = null pointer + 16-byte virtual dispatch offset inside QAudioOutput.
- **Fix:** In TMedia.h TMediaPlayer::setVolume(): retrieve audioOutput() into a local pointer and guard with null check before calling setVolume(). In TMedia.cpp TMedia::play(): retrieve audioOutput() into a local pointer and guard with null check before calling setMuted(). Matches the existing null-guard pattern already present in setMediaPlayersMuted().
- **Files changed:** src/TMedia.h, src/TMedia.cpp
---

## cpp-heap-memory-growth — Qt Multimedia audio decode buffers accumulate in native heap
- **Date:** 2026-03-28
- **Error patterns:** RSS, heap, memory growth, QMediaPlayer, QAudioOutput, AVFoundation, audio, playSoundFile, StoppedState, decode buffer, sleep wake, burst, TMedia, TMediaPlayer
- **Root cause:** Three mechanisms: (1) QMediaPlayer instances in StoppedState were not purged proactively — handlePlayerPlaybackStateChanged() fired sysMediaFinished but never released the native decode buffer, leaving it held until list eviction (triggered only on new playback). (2) QMediaPlayer was parented to Host rather than owned by TMediaPlayer, so evicted TMediaPlayer shells left QMediaPlayer+QAudioOutput alive until Host teardown. (3) On macOS sleep/wake, QMediaDevices::audioOutputsChanged fired refreshAudioOutput() creating a new QAudioOutput for every live player simultaneously.
- **Fix:** (1) Call player->mediaPlayer()->setSource(QUrl()) immediately on StoppedState in handlePlayerPlaybackStateChanged() to release the decode buffer. (2) Own QMediaPlayer via std::unique_ptr<QMediaPlayer> with nullptr parent; add explicit ~TMediaPlayer() that stops+clears source; parent QAudioOutput to mMediaPlayer. (3) Add TMedia::getMediaPlayerCounts() and report media_sound_players/media_music_players/media_stopped_players in getSubsystemMemoryStats().
- **Files changed:** src/TMedia.h, src/TMedia.cpp, src/TLuaInterpreterMudletObjects.cpp
---

## search-result-wrong-line — deferred restoreEditorState overwrites search result cursor position
- **Date:** 2026-03-27
- **Error patterns:** search result, wrong line, editor, cursor, moveCaretTo, QTimer, singleShot, restoreEditorState, slot_itemSelectedInSearchResults
- **Root cause:** In slot_itemSelectedInSearchResults, the synchronous moveCaretTo(search result line) fires before the deferred QTimer::singleShot(0) callback from restoreEditorState. The timer then overwrites the correct search position with the previously-saved editor state (cursor/scroll from the last time the user visited that item).
- **Fix:** Wrapped all 6 SearchResultIsScript cases and the 1 SearchResultIsValue case in slot_itemSelectedInSearchResults inside QTimer::singleShot(0) lambdas, capturing line/column values before the defer. This ensures the search result cursor position fires after restoreEditorState's deferred callback, overriding it correctly.
- **Files changed:** src/dlgTriggerEditor.cpp
---
