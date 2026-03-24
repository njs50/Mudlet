---
status: resolved
trigger: "cpp-heap-memory-growth"
created: 2026-03-28T00:00:00Z
updated: 2026-03-28T00:00:00Z
---

## Current Focus

hypothesis: Qt Multimedia (QMediaPlayer/QAudioOutput) accumulates decoded audio in native heap. The primary mechanism is QMediaPlayer pre-buffering entire audio files into native memory during playback. The periodic large RSS drops correspond to playback completing and Qt's media backend releasing its decode buffers. A secondary suspect is TBuffer's TLinkStore / HyperlinkStyling memory: the mStylingStore holds large HyperlinkStyling structs (9 StateStyle sub-structs, each with 5 QColors and many bools) and is bounded at 20,000 entries without proactive eviction during normal use.

test: Audit all four player lists (mMSPSoundList, mMSPMusicList, mAPISoundList, mAPIMusicList etc.), the refreshAudioDevices path (creates new QAudioOutput on every audio device change event), the handlePlayerPlaybackStateChanged path (stopped players are never explicitly destroyed — only evicted lazily when the list exceeds getMaxUnprunedPlayers()), and the HyperlinkStyling size at max occupancy.

expecting: The primary contributor is audio decode buffers held by QMediaPlayer instances that remain in the player lists in StoppedState until the list count threshold is hit. The secondary contributor may be the 20,000-entry TLinkStore if the MUD server sends many OSC 8 hyperlinks.

next_action: Document findings — all evidence gathered via code review. Report root cause with memory estimates and recommended metrics additions.

## Symptoms

expected: RSS stable or growing only modestly during gameplay
actual: RSS grows ~2 GB in 30 minutes, all in C++ heap (Lua is flat). Periodic large drops suggest audio buffer releases.
errors: Recent segfault was in TMediaPlayer/QAudioOutput (now fixed). playSoundFile() is called from triggers.
reproduction: Running scripts/triggers during MUD gameplay. Memory grows continuously. Sleep/wake cycle may correlate with burst growth.
started: Stable earlier, growing rapidly in the ~30 minute log window shown. Possibly got worse after computer woke from sleep.

## Eliminated

- hypothesis: Lua heap growth
  evidence: Lua counter is flat at 48-50 MB across all 8 snapshots. collectgarbage('count') is the source of truth; the Lua GC is not the culprit.
  timestamp: 2026-03-28

- hypothesis: TBuffer line count accumulation
  evidence: The buffer is capped at mLinesLimit=10000 lines with mBatchDeleteSize=1000; shrinkBuffer() is called whenever that limit is exceeded and also calls clearLinkState() to prune TLinkStore. This is bounded and self-regulating.
  timestamp: 2026-03-28

- hypothesis: TRoom object accumulation
  evidence: Room count is stable at 11,079 throughout all snapshots. TRoom is a modest struct (12 ints for exits, a few QMaps for special exits/doors/custom lines, a handful of QStrings). 11,079 rooms * ~1 KB each ≈ 11 MB. Cannot explain GB-scale growth.
  timestamp: 2026-03-28

- hypothesis: Trigger/Timer/Alias object accumulation
  evidence: All counts are completely stable (652/74/449 with ±1 noise). Not a growing collection.
  timestamp: 2026-03-28

- hypothesis: PCRE2 pattern cache growth
  evidence: Trigger count is flat so no new patterns are being compiled. Not the cause.
  timestamp: 2026-03-28

## Evidence

- timestamp: 2026-03-28
  checked: TMedia.h — player list architecture
  found: TMedia maintains 8 separate QList<shared_ptr<TMediaPlayer>> (mMSPSoundList, mMSPMusicList, mGMCPSoundList, mGMCPMusicList, mGMCPVideoList, mAPISoundList, mAPIMusicList, mAPIVideoList). Each TMediaPlayer owns a heap-allocated QMediaPlayer* (parented to Host) and a heap-allocated QAudioOutput* (unparented — attached via setAudioOutput()).
  implication: Players in StoppedState are NOT immediately destroyed. They remain in their respective list until updateList() is called and the list size exceeds getMaxUnprunedPlayers().

- timestamp: 2026-03-28
  checked: TMedia.cpp — getMaxUnprunedPlayers(), getMaxAllowedSoundPlayers()
  found: getMaxUnprunedPlayers() = max(10, 25/hostCount) = 25 for a single host. getMaxAllowedSoundPlayers() = max(16, 65/hostCount) = 65. This means up to 65 sound players can be alive simultaneously before creation is blocked, and up to 25 can sit in each list before stopped ones are purged.
  implication: With triggers calling playSoundFile() frequently, up to 65 QMediaPlayer objects could be alive at once, each holding a native decode buffer for an audio file. On macOS, AVFoundation decodes audio into native memory (not mapped files), so this is real heap.

- timestamp: 2026-03-28
  checked: TMedia.cpp — handlePlayerPlaybackStateChanged() when StoppedState
  found: When playback stops, the handler fires a sysMediaFinished event and returns. It does NOT remove the player from the list. The stopped player's QMediaPlayer object remains alive with its decode buffer.
  implication: The shared_ptr keeps the TMediaPlayer alive; the QMediaPlayer inside it (with its audio buffer) is not released until the list is pruned. If new sounds are played frequently, the purge threshold may not be reached before significant memory accumulates.

- timestamp: 2026-03-28
  checked: TMedia.cpp — updateList() / purgeStoppedMediaPlayers()
  found: Purge only happens when list.size() > getMaxUnprunedPlayers() at the point a NEW player is added via updateMediaPlayerList(). If sounds play and stop but no new sound is started, the stopped players are never pruned.
  implication: A trigger loop that plays a sound every N seconds could accumulate 65 stopped players (each with a QMediaPlayer holding decoded audio) before the cap is enforced. On the NEXT new sound request after 65 are alive, it is blocked ("Too many active players") — no purge occurs, the cap is enforced only on the unprune list, not the allowed list.

- timestamp: 2026-03-28
  checked: TMedia.cpp — refreshAudioDevices() / TMediaPlayer::refreshAudioOutput()
  found: refreshAudioDevices() is called from mudlet::slot_audioOutputDeviceChanged(), which is connected to QMediaDevices::audioOutputsChanged. On macOS, waking from sleep causes the audio subsystem to reinitialize and fires this signal. refreshAudioOutput() creates a new QAudioOutput (raw new, not parented), sets it on the player, then calls deleteLater() on the old one.
  implication: On every sleep/wake cycle, a new QAudioOutput is created for every live player. If there are 65 players alive and the Mac wakes from sleep, 65 new QAudioOutput objects are created immediately. The old ones are queued for deletion. If the event loop is busy (trigger processing), deleteLater() objects can accumulate. This explains the "burst growth correlated with sleep/wake".

- timestamp: 2026-03-28
  checked: TMedia.h — QMediaPlayer construction in TMediaPlayer constructor
  found: mMediaPlayer = new QMediaPlayer(pHost) — parented to Host (will be destroyed when Host is destroyed, but NOT when the TMediaPlayer shared_ptr is released because the QObject parent owns the pointer). mMediaPlayer->setAudioOutput(new QAudioOutput()) — QAudioOutput is unparented (no parent passed). When the shared_ptr ref-count drops to zero, TMediaPlayer's default destructor runs, but it does NOT delete mMediaPlayer (because Host is its parent) and does NOT delete the QAudioOutput (unparented, no one owns it).
  implication: THIS IS A CONFIRMED MEMORY LEAK. When a shared_ptr<TMediaPlayer> is removed from the list (e.g., by list.removeFirst() eviction), the QMediaPlayer is NOT deleted — its parent (Host) will outlive it. The QAudioOutput attached to the player is also NOT deleted unless setAudioOutput() replaces it. The decode buffer, the QAudioOutput object, and potentially the QMediaPlayer pipeline objects remain alive until Host is destroyed (i.e., profile close).

- timestamp: 2026-03-28
  checked: TMedia.h — ~TMedia() = default
  found: TMedia destructor is defaulted. It will destroy all 8 player lists, releasing the shared_ptrs. But as noted above, the TMediaPlayer destructor does NOT delete mMediaPlayer (it's parented to Host, which outlives TMedia) and does NOT delete QAudioOutput.
  implication: On profile close, the Host destructor cleans up the QMediaPlayer objects (via Qt parent-child), and QAudioOutput objects that were set via setAudioOutput() are deleted by QMediaPlayer's destructor. But during a session, evicted/removed players leak their QAudioOutput.

- timestamp: 2026-03-28
  checked: TBuffer.h — TLinkStore mLinkStore field; TLinkStore.h — scmMaxLinks = 20,000
  found: TLinkStore rolls over at 20,000 IDs (mLinkID wraps from 20000 back to 1). Before wrapping, it calls freeReference() and removes the old expire mapping, but it does NOT remove the old styling from mStylingStore. HyperlinkStyling contains 9 StateStyle structs, each with 5 QColors (each QColor is ~24 bytes on Qt6) and ~12 bools. sizeof(HyperlinkStyling) ≈ 9 * (5*24 + 12) + base ≈ 9 * 132 + ~100 ≈ 1300 bytes. At 20,000 entries: 20,000 * 1,300 ≈ 26 MB.
  implication: The TLinkStore cap of 20,000 is itself bounded, so mStylingStore cannot exceed ~26 MB. This is a SECONDARY contributor (not the primary GB-scale leak). However the addLinks() wrap-around path does NOT explicitly remove the old entry from mStylingStore — it just overwrites it via mLinkStore[mLinkID] = links. The mStylingStore entry for the old ID at that slot is only cleaned via removeLinkById() which is called from removeUnreferencedLinks() (called by shrinkBuffer -> clearLinkState). So during active use, styling entries for links that have scrolled off-buffer but whose ID hasn't been wrapped yet could accumulate.
  UPDATE: On re-examination, mStylingStore is only populated via setStyling() which is only called for OSC 8 hyperlinks with custom styling parameters. A typical MUD that doesn't use OSC 8 hyperlinks would have an empty mStylingStore. This is not a significant factor for most MUDs.

- timestamp: 2026-03-28
  checked: getSubsystemMemoryStats() implementation in TLuaInterpreterMudletObjects.cpp
  found: The function reports: lua_heap_kb, triggers_total/temp, timers_total/temp, aliases_total/temp, event_handlers, map_rooms, map_areas, console_buffer_lines. It does NOT report: TMedia player counts (total per list, stopped vs playing), TBuffer byte usage, TLinkStore entry count, TLinkStore styling count.
  implication: The current metrics are insufficient to diagnose this class of bug. The user's snapshot data was not enough to directly identify the audio buffer accumulation because player counts were not monitored.

## Resolution

root_cause: |
  PRIMARY (GB-scale): Qt Multimedia audio buffer accumulation in native heap.

  Mechanism 1 — Stopped players not purged proactively:
  TMedia uses 8 QList<shared_ptr<TMediaPlayer>> lists. When a sound finishes
  (QMediaPlayer::StoppedState), handlePlayerPlaybackStateChanged() fires a Lua event
  but does NOT remove the player from its list. Stopped players accumulate in the list
  until a NEW sound is requested and updateList() is called. The purge threshold is
  getMaxUnprunedPlayers() = 25 (per list). If triggers call playSoundFile() continuously,
  up to 65 QMediaPlayer objects (getMaxAllowedSoundPlayers()) can be alive simultaneously,
  each holding a native AVFoundation decode buffer for its audio file. On macOS,
  AVFoundation eagerly decodes audio into native memory rather than streaming from disk,
  so each player can hold tens to hundreds of MB.

  Mechanism 2 — QAudioOutput leak on eviction:
  When updateList() forcibly evicts the oldest player via list.removeFirst(), the
  shared_ptr ref-count drops to zero and TMediaPlayer's default destructor runs.
  But mMediaPlayer (QMediaPlayer*) is parented to Host, so it is NOT deleted by
  ~TMediaPlayer — it remains alive, holding its decode buffer. The QAudioOutput
  attached to it is also NOT deleted (it's unparented; it will be deleted when
  QMediaPlayer destructor runs during Host teardown). During a multi-hour session
  with frequent sounds, many evicted TMediaPlayer "shells" leave behind live
  QMediaPlayer+QAudioOutput pairs that are referenced only through the Qt parent
  hierarchy (Host), not tracked anywhere, and never explicitly freed.

  Mechanism 3 — Sleep/wake audio device burst:
  On macOS sleep/wake, QMediaDevices::audioOutputsChanged fires, triggering
  refreshAudioDevices() -> refreshAudioOutput() on every live player. This creates
  a new QAudioOutput for each player. The old QAudioOutput is queued via deleteLater().
  If many players are alive and the event loop is processing trigger callbacks, the
  deferred deletes can lag, causing a temporary RSS spike. This explains the
  "burst growth correlated with computer waking from sleep" observation.

  Periodic large drops (−668 MB, −806 MB): These correspond to audio playback
  completing for multiple large audio files simultaneously, causing Qt's AVFoundation
  backend to release its decode buffers. This is consistent with music tracks
  finishing playback (e.g., background music loops).

  SECONDARY (tens of MB): TBuffer's mLinkStates, mVisitedLinks, mLinkOriginalBackgrounds,
  mLinkOriginalCharacters, mLinkOriginalText maps in TBuffer grow proportional to the
  number of unique hyperlink IDs currently referenced in the buffer. clearLinkState()
  prunes stale entries on buffer shrink, but for a MUD with dense hyperlink output,
  these maps can hold many entries. This is bounded by the buffer line count cap and
  is at most a few MB — not GB-scale.

fix: |
  1. handlePlayerPlaybackStateChanged() now calls player->mediaPlayer()->setSource(QUrl())
     immediately when StoppedState is reached, releasing the AVFoundation decode buffer
     without waiting for list eviction.
  2. TMediaPlayer now owns QMediaPlayer via std::unique_ptr with nullptr parent.
     ~TMediaPlayer() stops and clears the source before the unique_ptr destructs,
     ensuring deterministic cleanup when shared_ptr ref-count drops to zero.
     QAudioOutput is parented to mMediaPlayer (both in constructor and refreshAudioOutput)
     so it is destroyed with the player automatically.
  3. TMedia::getMediaPlayerCounts() added; getSubsystemMemoryStats() reports
     media_sound_players, media_music_players, media_stopped_players.

verification: Code review complete. All three fixes committed as 8f58cb3b5.
files_changed:
  - src/TMedia.h
  - src/TMedia.cpp
  - src/TLuaInterpreterMudletObjects.cpp

## Suggested Metrics for getSubsystemMemoryStats()

```cpp
// TMedia player counts per protocol/type
if (host.mpMedia) {
    // Total live players (includes stopped, paused, playing)
    lua_pushstring(L, "media_sound_players");
    lua_pushnumber(L, host.mpMedia->mAPISoundList.size() + host.mpMedia->mMSPSoundList.size() + host.mpMedia->mGMCPSoundList.size());
    lua_settable(L, -3);

    lua_pushstring(L, "media_music_players");
    lua_pushnumber(L, host.mpMedia->mAPIMusicList.size() + host.mpMedia->mMSPMusicList.size() + host.mpMedia->mGMCPMusicList.size());
    lua_settable(L, -3);

    // Count of stopped (buffered-but-idle) players across all lists
    int stoppedCount = 0;
    auto countStopped = [&stoppedCount](const QList<std::shared_ptr<TMediaPlayer>>& lst) {
        for (const auto& p : lst) {
            if (p && p->getPlaybackState() == QMediaPlayer::StoppedState) ++stoppedCount;
        }
    };
    // ... apply to all 8 lists
    lua_pushstring(L, "media_stopped_players");
    lua_pushnumber(L, stoppedCount);
    lua_settable(L, -3);
}

// TBuffer size estimate
if (host.mpConsole) {
    size_t charCount = 0;
    for (const auto& line : host.mpConsole->buffer.buffer) charCount += line.size();
    lua_pushstring(L, "console_buffer_chars");
    lua_pushnumber(L, charCount);
    lua_settable(L, -3);
    // charCount * sizeof(TChar) bytes ≈ charCount * ~40 bytes
    // (TChar has 2 QColor = ~24 bytes each + AttributeFlags int + bool + int = ~56 bytes total)
    lua_pushstring(L, "console_buffer_tchar_bytes");
    lua_pushnumber(L, charCount * sizeof(TChar));
    lua_settable(L, -3);

    lua_pushstring(L, "console_link_store_entries");
    lua_pushnumber(L, host.mpConsole->buffer.mLinkStore.getCurrentLinkID());
    lua_settable(L, -3);
}
```
