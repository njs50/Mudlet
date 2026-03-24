---
phase: 1
slug: memory-audit-fixes
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-24
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Qt Test (Qt6::Test) |
| **Config file** | `test/CMakeLists.txt` |
| **Quick run command** | `ctest -R TLuaInterfaceTest --output-on-failure` (from build dir) |
| **Full suite command** | `ctest --output-on-failure` (from build dir) |
| **Estimated runtime** | ~60 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cmake --build /Users/njs50/play/Mudlet-build` (build must pass)
- **After every plan wave:** Run `ctest --output-on-failure` from build dir
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~120 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | Status |
|---------|------|------|-------------|-----------|-------------------|--------|
| Hunspell leak | 01 | 1 | mem-safety | build | `cmake --build /Users/njs50/play/Mudlet-build` | ⬜ pending |
| TMap::mpRoomDB | 01 | 1 | mem-safety | build | `cmake --build /Users/njs50/play/Mudlet-build` | ⬜ pending |
| TTrigger color/condition | 01 | 1 | mem-safety | build+test | `cmake --build && ctest -R dlgTriggerEditorUndoRedoTest` | ⬜ pending |
| Redundant Qt-parent delete | 01 | 1 | mem-cleanup | build | `cmake --build /Users/njs50/play/Mudlet-build` | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

None — existing test infrastructure is sufficient. The phase gate is: build clean + ctest green.

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

| Behavior | Why Manual | Test Instructions |
|----------|------------|-------------------|
| TMainConsole::mpHunspell_shared ownership | Requires call-site trace to confirm owning vs non-owning | Trace `prepareSharedDictionary()` return value ownership in mudlet.cpp before patching |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify (build passes)
- [ ] Sampling continuity: build checked after every task
- [ ] No Wave 0 gaps
- [ ] Full ctest suite green before verify-work
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
