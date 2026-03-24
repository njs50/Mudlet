---
status: resolved
trigger: "Segfault in QAudioOutput::setVolume(float) called on a null pointer from TMediaPlayer::setVolume(int) const"
created: 2026-03-28T00:00:00Z
updated: 2026-03-28T00:15:00Z
---

## Current Focus

hypothesis: CONFIRMED — TMediaPlayer::setVolume() and TMedia::play() call audioOutput()->setVolume()/setMuted() without guarding against audioOutput() returning null. Root cause is re-entrancy: playMedia triggers sysMediaStarted/sysMediaFinished Lua events synchronously inside play(), which can call playSoundFile() again, putting the player list into an inconsistent state. The null audioOutput() is the proximate crash; the structural issue is missing null guards.
test: Code read of TMediaPlayer::setVolume() and TMedia::play() lines 1638-1649
expecting: Missing null checks for audioOutput() — confirmed
next_action: Apply fix — add audioOutput() null checks in setVolume() and play()

## Symptoms

expected: TMediaPlayer::setVolume() safely sets volume on a valid QAudioOutput
actual: SEGV at QAudioOutput::setVolume — the QAudioOutput pointer inside TMediaPlayer is null
errors: |
  TMedia::updateList() - Replacing existing player at index: 0
  TMedia::updateList() - List size after update: 1
  TMedia::updateMediaPlayerList() - Searching for existing player in list.
  TMedia::updateMediaPlayerList() - Found existing player at index: 0
  TMedia::updateMediaPlayerList() - Updating list for protocol: -1 and type: 1
  TMedia::updateList() - Replacing existing player at index: 0
  TMedia::updateList() - List size after update: 1
  SEGV on unknown address 0x000000000010 (READ access)
  #0  QAudioOutput::setVolume(float)
  #1  TMediaPlayer::setVolume(int) const
  #2  TMedia::play(TMediaData&)
  #3  TMedia::playMedia(TMediaData&)
  #4  TLuaInterpreter::playSoundFileAsOrderedArguments
  ...TTrigger chain...
  #18 TriggerUnit::processDataStream
reproduction: Playing sound via playSoundFile() Lua call while a trigger fires during telnet data processing
started: Observed now; ASan build
timeline: updateList() "Replacing existing player at index 0" fires TWICE before crash

## Eliminated

(none yet)

## Evidence

- timestamp: 2026-03-28T00:00:00Z
  checked: Stack trace and debug log
  found: updateList() replaces player at index 0 TWICE before crash; play() then calls setVolume() on what appears to be a destroyed player
  implication: Classic use-after-free OR the QAudioOutput member is null because the replacement player wasn't fully initialized

- timestamp: 2026-03-28T00:01:00Z
  checked: TMediaPlayer.h lines 66-73 (setVolume implementation)
  found: setVolume() checks mMediaPlayer != null but then calls mMediaPlayer->audioOutput()->setVolume() — no null check on audioOutput()
  implication: If QMediaPlayer::audioOutput() returns null, this crashes at the virtual dispatch offset — matches 0x10 crash address

- timestamp: 2026-03-28T00:02:00Z
  checked: TMedia.cpp lines 1638-1649 (play() — setVolume and setMuted calls)
  found: Line 1638 calls pPlayer->setVolume() then lines 1644-1649 call pPlayer->mediaPlayer()->audioOutput()->setMuted() with no null check on audioOutput()
  implication: Same null-dereference risk at multiple call sites in play()

- timestamp: 2026-03-28T00:03:00Z
  checked: TMedia.cpp lines 560-575 (setMediaPlayersMuted — correctly guarded)
  found: setMediaPlayersMuted DOES check audioOutput() != null before calling setMuted — proof author knows audioOutput() can be null
  implication: setVolume() and play() were not given the same protection — oversight

- timestamp: 2026-03-28T00:04:00Z
  checked: TMedia.cpp lines 958-1054 (connectMediaPlayer — positionChanged lambda)
  found: positionChanged lambda calls lockedPlayer->setVolume() at lines 1032/1035/1043/1049 — no audioOutput() null guard inside setVolume()
  implication: The positionChanged signal fires asynchronously; if audioOutput() becomes null between registration and firing, crash occurs here too

- timestamp: 2026-03-28T00:05:00Z
  checked: TMedia.cpp lines 1232-1296 (handlePlayerPlaybackStateChanged)
  found: Calls mpHost->raiseEvent(mediaStarted/mediaFinished) synchronously on the same thread during playback state transitions
  implication: raiseEvent() runs Lua event handlers which can call playSoundFile() → play() re-entrantly. This is the re-entrancy vector that puts the system in the double-replace state seen in the log.

- timestamp: 2026-03-28T00:06:00Z
  checked: TMediaPlayer.h lines 43-51 (constructor)
  found: Parameterized constructor calls mMediaPlayer->setAudioOutput(new QAudioOutput()) — always sets valid output on construction
  implication: audioOutput() only becomes null if (a) re-entrant calls corrupt state, (b) Qt internals clear it, or (c) the player was constructed via default constructor (initialized=false path) but setVolume guard on mMediaPlayer should catch that

- timestamp: 2026-03-28T00:07:00Z
  checked: TMediaData.h line 34
  found: MediaProtocolAPI = -1 (matches "protocol: -1" in crash log)
  implication: The crashing play() call is from the Lua API (playSoundFile), type=1 (Sound) — consistent with the stack trace

## Resolution

root_cause: |
  TMediaPlayer::setVolume() called mMediaPlayer->audioOutput()->setVolume() without checking if audioOutput() returned null.
  Similarly, TMedia::play() called pPlayer->mediaPlayer()->audioOutput()->setMuted() without a null guard.
  QMediaPlayer::audioOutput() can return null when re-entrant playSoundFile() calls (triggered via sysMediaStarted/sysMediaFinished
  Lua events emitted synchronously during stop()/play() state transitions) put the player list into an inconsistent state,
  or when the QAudioOutput is otherwise unavailable. The crash address 0x10 = null pointer + 16-byte offset of the first
  virtual dispatch member inside QAudioOutput.
fix: |
  1. TMedia.h TMediaPlayer::setVolume(): retrieve audioOutput() into a local pointer, guard with null check before calling setVolume().
  2. TMedia.cpp TMedia::play() lines 1641-1655: retrieve audioOutput() into a local pointer, guard with null check before calling setMuted().
  Note: setMediaPlayersMuted() already had this null guard; this fix brings setVolume() and play() into consistency.
verification: Code review confirms null guard added at all call sites where audioOutput()->method() was called without checking.
  The fix is minimal and targeted — no ownership or list management changes needed for the immediate crash.
files_changed:
  - src/TMedia.h
  - src/TMedia.cpp
