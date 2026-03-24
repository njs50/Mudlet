---
id: SEED-001
status: dormant
planted: 2026-03-28
planted_during: v1.0 / memory-safety milestone
trigger_when: milestone focused on scripting power-users, bot/automation, or CI/CD integration
scope: medium
---

# SEED-001: Headless/terminal mode — run Mudlet without GUI editors

## Why This Matters

Power users and automation workflows (bots, CI testing, server-side scripting) need Mudlet's trigger/script engine without the GUI overhead. A headless mode would enable running Mudlet as a scriptable MUD client daemon — useful for automation, headless servers, and CI integration testing of Lua scripts.

Critically: the zero-code path already works today (`QT_QPA_PLATFORM=offscreen` + `--mirror`). A proper `--headless` flag is only 1–2 weeks of work and would formalise what already exists.

## When to Surface

**Trigger:** Starting a milestone focused on scripting power-users, bot/automation use cases, or CI/CD integration.

This seed should be presented during `/gsd:new-milestone` when the milestone scope matches any of these conditions:
- Milestone goal mentions "headless", "bot", "automation", "daemon", "CLI", or "server-side"
- Milestone targets power users or scripters specifically
- Milestone includes CI/CD or testing infrastructure work

## Scope Estimate

**Medium** — a phase or two.

- **Option A (zero code):** `QT_QPA_PLATFORM=offscreen` + `--mirror` already works today — just needs documentation
- **Option B (1–2 weeks):** Add `--headless` CLI flag that skips GUI init, cleans up the stdout output path, and suppresses editor/map windows
- **Option C (3–6 months):** Full `QCoreApplication` refactor — remove Qt GUI dependency entirely. Blocked by 564 `mudlet::self()` call-sites across 50 files

Recommended starting point: Option B.

## Breadcrumbs

Related code and decisions found in the codebase:

- `.planning/research/terminal-mode-feasibility.md` — full feasibility research with code analysis
- `src/mudlet.h` / `src/mudlet.cpp` — main singleton with 564 `mudlet::self()` call-sites; central blocker for full refactor
- `src/TConsole.h` / `src/TConsole.cpp` — QWidget-derived; `smMirrorToStdOut` flag already routes output to stdout
- `src/TLuaInterpreter.h` / `src/TLuaInterpreter.cpp` — nearly GUI-free at runtime; `updateEditor()` calls are null-guarded
- `src/ctelnet.cpp` — 41 `mpConsole->print()` calls; work fine under offscreen Qt platform
- `src/main.cpp` — entry point; `--mirror` flag already implemented

## Notes

Investigated 2026-03-28 as a standalone feasibility study (not tied to any active phase). The key insight from research: TLuaInterpreter's scripting pipeline (triggers, aliases, timers, GMCP) has zero GUI dependencies at runtime. The Qt offscreen platform plugin handles all widget construction silently, making Option A viable with no code changes.

Open questions to address when this seeds activates:
- Should headless output preserve ANSI color codes or strip them?
- Should a headless profile be loaded non-interactively from a path, or reuse existing profile XML?
- Should TTS/media playback be explicitly disabled in headless mode?
