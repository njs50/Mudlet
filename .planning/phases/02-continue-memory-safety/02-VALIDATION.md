---
phase: 02
slug: continue-memory-safety
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-25
---

# Phase 02 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Qt Test |
| **Config file** | test/CMakeLists.txt |
| **Quick run command** | `cd /Users/njs50/play/Mudlet-build && ctest --output-on-failure -R TLuaInterfaceTest` |
| **Full suite command** | `cd /Users/njs50/play/Mudlet-build && ctest --output-on-failure` |
| **Estimated runtime** | ~60 seconds (quick), ~120 seconds (full) |

---

## Sampling Rate

- **After every task commit:** Run `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 | tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 | tail -10`
- **After every plan wave:** Run `cd /Users/njs50/play/Mudlet-build && ctest --output-on-failure`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-T1 | 01 | 1 | MEM-HOST-MMCP, MEM-HOST-STOPWATCH | compile + unit | `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 \| tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 \| tail -10` | ✅ | ⬜ pending |
| 02-01-T2 | 01 | 1 | MEM-HOST-PROFILESHORTCUTS | compile + unit | `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 \| tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 \| tail -10` | ✅ | ⬜ pending |
| 02-02-T1 | 02 | 1 | MEM-DISCORD-HANDLERS, MEM-DISCORD-PRESENCE | compile + unit | `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 \| tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 \| tail -10` | ✅ | ⬜ pending |
| 02-03-T1 | 03 | 1 | MEM-VARUNIT-BASE | compile + unit | `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 \| tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 \| tail -10` | ✅ | ⬜ pending |
| 02-03-T2 | 03 | 1 | MEM-UPDATER-FEED | compile + unit | `cmake --build /Users/njs50/play/Mudlet-build --target mudlet_test -j$(sysctl -n hw.ncpu) 2>&1 \| tail -20 && ctest --output-on-failure -R TLuaInterfaceTest 2>&1 \| tail -10` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

---

## Manual-Only Verifications

All phase behaviors have automated verification.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 60s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
