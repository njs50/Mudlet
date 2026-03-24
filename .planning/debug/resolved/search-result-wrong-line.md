---
status: resolved
trigger: "Clicking a search result in the trigger/script/alias search panel navigates to the wrong line"
created: 2026-03-25T00:00:00Z
updated: 2026-03-25T00:01:00Z
---

## Current Focus

hypothesis: CONFIRMED — restoreEditorState deferred callback (QTimer::singleShot(0)) overrides the moveCaretTo call made synchronously in slot_itemSelectedInSearchResults
test: trace the sequence of operations when clicking a search result
expecting: timer fires after moveCaretTo, overwriting the search result cursor position
next_action: wrap moveCaretTo calls in slot_itemSelectedInSearchResults in QTimer::singleShot(0) so they fire after the restoreEditorState callback

## Symptoms

expected: Clicking a search result opens that item in the editor and jumps to the correct matching line
actual: The editor opens the item but jumps to the wrong line — the line appears to match a previous search query's result, not the current one
errors: None reported
reproduction: Run a search in the trigger/script/alias search panel. Click a result. Note the line. Run a different search. Click a result. The editor jumps to the line from the first search, not the second.
started: Exists on development branch (commit 8ab246d74 introduced the regression when adding restoreEditorState with QTimer::singleShot)

## Eliminated

- hypothesis: bug introduced by Phase 1/2 unique_ptr migration
  evidence: git diff shows dlgTriggerEditor.cpp has ZERO changes between development and fix-memory-leaks HEAD
  timestamp: 2026-03-25

- hypothesis: stale line numbers stored in QTreeWidget item data
  evidence: setAllSearchData correctly stores index (line number) in PatternOrLineRole; logic is sound
  timestamp: 2026-03-25

## Evidence

- timestamp: 2026-03-25
  checked: dlgTriggerEditor.cpp diff between development and HEAD
  found: dlgTriggerEditor.cpp is IDENTICAL — zero changes from our Phase 1/2 work
  implication: bug exists on development branch, not introduced by this branch's changes

- timestamp: 2026-03-25
  checked: commit 8ab246d74 ("Fix: Remember scroll position and cursor position in script editor")
  found: introduced restoreEditorState() which uses QTimer::singleShot(0) to defer cursor/scroll restore
  implication: the deferred callback fires AFTER the synchronous moveCaretTo() in slot_itemSelectedInSearchResults

- timestamp: 2026-03-25
  checked: slot_itemSelectedInSearchResults (line 1938) and restoreEditorState (line 12683)
  found: sequence when clicking search result for SearchResultIsScript:
    1. slot_triggerSelected() called -> loads script -> calls restoreEditorState() which schedules QTimer::singleShot(0)
    2. highlightSearchMatches() called (synchronous)
    3. controller->moveCaretTo(PatternOrLineRole, PositionRole) called (synchronous) -- CORRECT position
    4. All synchronous code returns
    5. QTimer fires -> restoreEditorState lambda runs -> controller->moveCaretTo(savedLine, savedColumn) -- OVERRIDES correct position
  implication: the saved editor state (from previous visit) overwrites the search result cursor position

## Resolution

root_cause: In slot_itemSelectedInSearchResults, the synchronous moveCaretTo(search result line) is called before the deferred QTimer::singleShot(0) callback from restoreEditorState fires. The timer then overwrites the correct search position with the previously-saved editor state (cursor/scroll from last time the user was in this item).
fix: Wrapped all 6 SearchResultIsScript cases and the 1 SearchResultIsValue case in slot_itemSelectedInSearchResults inside QTimer::singleShot(0) lambdas, capturing line/column values before the defer. This ensures the search result cursor position fires after restoreEditorState's deferred callback, overriding it correctly.
verification: confirmed fixed by user — clicking search results now navigates to the correct line
files_changed: [src/dlgTriggerEditor.cpp]
