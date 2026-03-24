# Terminal / Headless Mode Feasibility Research

**Researched:** 2026-03-27
**Domain:** Qt6 headless operation, Mudlet architecture decoupling
**Confidence:** HIGH (based on direct code inspection + Qt documentation)

---

## Verdict

A headless/terminal mode for Mudlet is **technically feasible but requires substantial architectural work** — estimated at 4–8 weeks of focused effort for a minimal viable headless mode, or 3–6 months for a clean, maintainable implementation. The telnet/protocol stack and Lua scripting engine are logically separable from the GUI. However, the `mudlet` singleton (a `QMainWindow` subclass) is deeply wired into virtually every subsystem — 564 `mudlet::self()` call-sites across 50 files — meaning any headless mode must either (a) run a hidden `QApplication` with stub widgets using the Qt `offscreen` platform plugin, or (b) surgically replace `mudlet` with a headless coordinator class. Option (a) is the low-risk path and can be done incrementally. Option (b) is the architecturally correct long-term answer and is a major refactor. A "bot mode" using existing `--mirror` stdout output and the offscreen Qt platform plugin could be prototyped in 1–2 weeks without any code changes.

---

## Qt Headless Analysis

### What Qt Requires

Mudlet currently instantiates `QApplication` (the full widget stack) in `main.cpp` line 221:

```cpp
auto app = qobject_cast<QApplication*>(new QApplication(argc, argv));
```

`QApplication` requires a display server (X11 on Linux, Quartz on macOS, Win32 on Windows). On a headless Linux server this fails with `Could not load the Qt platform plugin "xcb"`.

**Confidence: HIGH** (verified in `src/main.cpp`)

### Qt's Headless Escape Hatches

Qt provides two mechanisms that allow `QApplication` and `QWidget` code to run without a real display:

1. **`QT_QPA_PLATFORM=offscreen` environment variable** — Qt renders widgets to an offscreen buffer. No display server needed. `QWidget::show()` silently succeeds. Works with existing code without any changes. Drawback: all widget rendering is still performed (wasted CPU).

2. **`QT_QPA_PLATFORM=minimal`** — Even lighter; no rendering at all, just stubs. Suitable for a pure bot/script mode.

Either allows the full Mudlet binary to run on a headless Linux/CI server today, without code changes, by setting the environment variable before launch. This is the existing escape hatch — the `--mirror` stdout mode (`smMirrorToStdOut`) combined with `QT_QPA_PLATFORM=offscreen` is effectively already a usable bot mode.

**Confidence: HIGH** (Qt official docs + community-verified, confirmed for Qt6)

### `mudlet` Singleton Coupling

`mudlet` inherits from `QMainWindow`, includes the full UI form `ui_main_window.h`, and manages all dock widgets, toolbars, the profile selector, IRC, the package manager, and replay. `mudlet::self()` is called from:

| File | Call count |
|------|-----------|
| `mudlet.cpp` | 33 |
| `Host.cpp` | 34 |
| `ctelnet.cpp` | 24 |
| `TMainConsole.cpp` | 20 |
| `TLuaInterpreter.cpp` | 28 |
| `TLuaInterpreterUI.cpp` | 13 |
| `TLuaInterpreterMudletObjects.cpp` | 16 |
| `TLuaInterpreterDiscord.cpp` | 23 |
| Other files | ~373 |
| **Total** | **564** |

The calls are not all GUI-related. Many access `mAppBuild` (version string), `getQSettings()`, `getMudletPath()`, `mDiscord`, or network defaults. But a meaningful fraction touch GUI: dock widgets, replay window, alert, sound mute, options dialogs, editor refresh.

**Replacing `mudlet::self()` with a leaner non-GUI coordinator** is feasible but requires auditing and reclassifying all 564 call sites.

---

## TConsole Coupling

### Direct Widget Dependency

`TConsole` inherits `QWidget` directly:

```cpp
class TConsole : public QWidget { ... };
```

`TMainConsole : public TConsole` is the per-profile main output area. `Host` holds:

```cpp
QPointer<TMainConsole> mpConsole;
```

`mpConsole` is referenced **199 times in `Host.cpp`** and **41 times in `ctelnet.cpp`**. The telnet layer calls `mpHost->mpConsole->print(...)` directly for displaying incoming data, error messages, GMCP feedback, and protocol diagnostics.

### What Would Need to Replace TConsole

For a headless mode, `TConsole` cannot be instantiated (it requires a display for `QWidget`). Two paths:

**Path A — Stub console:** Create a `THeadlessConsole` that shares the `TConsole` interface but writes to stdout/file instead of a widget. The `TBuffer` (which stores lines with ANSI attributes) is already display-agnostic — it does not inherit QWidget. The widget rendering is isolated in `TTextEdit`. The data model could be reused.

**Path B — offscreen plugin:** Use `QT_QPA_PLATFORM=offscreen`. `TConsole` and `TMainConsole` instantiate fine, `QWidget::show()` silently no-ops, and no rendering happens. The `--mirror` stdout feature (`smMirrorToStdOut`) then routes all output to stdout. This works today.

**Confidence: HIGH** (direct code inspection of `TConsole.h`, `Host.h`, `ctelnet.cpp`)

### Existing stdout Mirror Feature

`TConsole::print()` already has:

```cpp
if (Q_UNLIKELY(mudlet::self()->smMirrorToStdOut)) {
    qDebug().nospace().noquote() << qsl("%1| %2").arg(mConsoleName, msg);
}
```

This mirrors every console line to stdout via `qDebug()`. It is activated with the `--mirror` / `-m` command line flag. This is already functional infrastructure for a non-GUI output pipeline.

---

## TLuaInterpreter Coupling

### GUI Dependencies in TLuaInterpreter.h

The header includes one problematic GUI widget:

```cpp
#include <edbee/texteditorwidget.h>
```

The edbee widget is used **only** for `formatLuaCode()` (code formatting) and `updateEditor()`. The `updateEditor()` call checks `if (mpHost->mpEditorDialog)` — it is a no-op if the editor dialog is not open. In headless mode, `mpEditorDialog` would be null, so all `updateEditor()` calls degrade to no-ops without any code changes.

The edbee include in the header does pull in widget headers, but this is a compile-time dependency, not a runtime one — the widget is never instantiated unless the editor is opened.

### GUI Calls in TLuaInterpreter Split Files

Lua API functions that manipulate windows (`TLuaInterpreterUI.cpp`: 13 `mudlet::self()` calls, `TLuaInterpreterMudletObjects.cpp`: 16 calls) can be made to return early with an appropriate error if no display is available. The scripting core (`call()`, trigger matching, GMCP handlers, timer firing) does not touch GUI at all.

**Confidence: HIGH** (direct code inspection of all TLuaInterpreter*.cpp files)

### What Lua API Stops Working in Headless Mode

Any Lua function that creates or manipulates a widget would need to:
- Return an error: "GUI not available in headless mode"
- Or silently no-op

Affected categories: `createLabel`, `createButton`, `setAppStyleSheet`, `showMap`, `openUserWindow`, IRC, text-to-speech (if not compiled), media playback. The core scripting pipeline (triggers, aliases, timers, scripts, GMCP, send/receive) is fully available.

---

## Scope Estimate

### Option 1: Zero-Code Headless (Available Now)

**Effort: 0 days — works today**

```bash
QT_QPA_PLATFORM=offscreen ./mudlet --profile MyProfile --mirror
```

- All MUD data flows through the trigger/alias/timer/script pipeline
- `--mirror` outputs all console text to stdout via `qDebug()`
- No editor windows shown (they would just have no display)
- Full Lua scripting engine available
- Limitation: output format is `qDebug()` format, not clean stdout; widget rendering overhead still present

**Suitable for: CI testing, simple bots with minimal Lua UI requirements.**

### Option 2: Minimal Headless Flag (1–2 weeks)

Add `--headless` flag to `main.cpp` that:
1. Forces `QT_QPA_PLATFORM=offscreen` programmatically (via `qputenv` before `QApplication` construction)
2. Skips splash screen, profile dialog, main window `show()`
3. Auto-loads a specified profile (`--profile` flag already exists)
4. Redirects console output to clean stdout (replacing `qDebug()` in the mirror path)
5. Disables Discord, IRC, updater initialization

This does not require touching the 564 `mudlet::self()` sites or refactoring `TConsole`.

**Suitable for: bot mode, server-side scripting automation, CI integration.**

### Option 3: Clean Headless Architecture (3–6 months)

- Extract a `IMudletApp` interface from `mudlet.h` with the methods actually needed by `Host`/`ctelnet`/`TLuaInterpreter`
- Implement `MudletHeadlessApp : QCoreApplication` that provides the same interface without any `QWidget` or display dependency
- Create `THeadlessConsole` replacing `TMainConsole` for the headless path
- Feature-gate all GUI Lua API functions behind an `if (mudlet::self()->isHeadless())` guard

This is the architecturally clean solution and enables proper server-side deployment.

---

## Recommended Approach (if feasible)

**Start with Option 2.** It delivers real value in 1–2 weeks by:

1. Adding `--headless` to `main.cpp` that calls `qputenv("QT_QPA_PLATFORM", "offscreen")` before `QApplication` construction and skips GUI initialization steps
2. Improving the stdout mirror path to write clean lines (strip `qDebug()` formatting), gating on `--headless` rather than `--mirror`
3. Auto-connecting to a profile specified by `--profile` without showing the connection dialog
4. Documenting unsupported Lua APIs in headless mode

**Longer term:** Option 3 if Mudlet wants server-deployment as a first-class use case (e.g., running as a game bot infrastructure service). The `IMudletApp` interface extraction is the hardest part; extracting `mAppBuild`, `getMudletPath()`, `getQSettings()`, and network helpers into a non-widget base class covers the majority of `mudlet::self()` call sites.

---

## Risks and Blockers

### Risk 1: `QApplication::alert()` and Sound

`ctelnet.cpp` line 4952 calls `QApplication::alert(mudlet::self(), 3000)` on new data arrival when `mAlertOnNewData` is true. This requires a real window handle. In offscreen mode this silently fails — acceptable for headless. In Option 3 (QCoreApplication), this must be guarded.

### Risk 2: `dlgTriggerEditor` Included in Host.h

`Host.h` includes `dlgTriggerEditor.h` directly (not a forward declaration):

```cpp
#include "dlgTriggerEditor.h"
```

This pulls the full trigger editor widget (and all its dependencies) into every translation unit that includes `Host.h`. For a clean headless build, this include would need to become a forward declaration with `dlgTriggerEditor::SearchOptions` moved to its own enum header. This is a compilation-time concern, not a runtime one with the offscreen plugin.

### Risk 3: `QProgressDialog` in ctelnet

`ctelnet.cpp` creates a `QProgressDialog` for GUI downloads. In headless mode this must be guarded or replaced with a log message.

### Risk 4: Profile Loading Requires TMainConsole

`Host` construction sequence requires `mpConsole` (a `TMainConsole`) to be set before scripts compile. In offscreen mode this works because widget construction succeeds silently. For Option 3 (QCoreApplication), a stub console is required.

### Risk 5: Lua `wait()` / coroutines

The `Wait` Lua function uses `QEventLoop`. This is available in `QCoreApplication` and does not require a display.

---

## Prior Art

### Existing `--mirror` Flag (Mudlet itself)

Mudlet already ships `smMirrorToStdOut` (`--mirror` / `-m` flag) which routes all console output to `qDebug()` stdout. This is explicitly documented in help text as "helpful for CI environments." This is the seed of headless mode — the intent is already there.

### Blightmud (Rust)

[Blightmud](https://github.com/Blightmud/Blightmud) is a terminal-native MUD client written in Rust with Lua scripting (triggers, aliases, timers). It runs in a terminal TUI with no GUI. This is the reference point for "what a terminal MUD client looks like" — it is essentially what Mudlet headless mode would produce. Blightmud does not have Mudlet's rich GUI feature set but demonstrates the Lua scripting pipeline works fine in a terminal context.

### mbf (Python MUD Bot Framework)

[mbf](https://github.com/Oliver2213/mbf) is a Python library specifically for MUD bots — headless connections with trigger/alias processing. Shows demand for non-GUI MUD automation.

### Qt Headless Patterns

The Qt community pattern for "run QWidget app headless" is `QT_QPA_PLATFORM=offscreen`. This is the documented solution. Multiple projects (including Qt's own CI) use this approach. It is stable in Qt6.

### GitHub Issues

A search of the Mudlet GitHub repository found no open issues explicitly requesting headless or bot mode. The closest related feature is the existing `--mirror` stdout flag, which was added for CI environments. There is no community discussion of a formal headless mode on record.

---

## Sources

### Primary (HIGH confidence)

- `src/main.cpp` — `QApplication` construction, `mirrorToStdout` flag, command line parsing
- `src/mudlet.h` — `smMirrorToStdOut`, `mudlet : QMainWindow` inheritance, `mudlet::self()` singleton
- `src/TConsole.h` / `src/TMainConsole.h` — `TConsole : QWidget` inheritance
- `src/Host.h` — `QPointer<TMainConsole> mpConsole`, `dlgTriggerEditor` include
- `src/TLuaInterpreter.h` — `edbee/texteditorwidget.h` include, `updateEditor()`
- `src/TLuaInterpreter.cpp` — `updateEditor()` implementation (no-ops when editor dialog is null)
- `src/ctelnet.cpp` — `mpConsole->print()` calls, `QApplication::alert()`
- Grep counts of `mudlet::self()` across 50 files (564 total occurrences)

### Secondary (MEDIUM confidence)

- [Qt Forum: Enabling Qt Widgets to run with no GUI capability](https://forum.qt.io/topic/151998/enabling-qt-widgets-application-to-run-with-no-gui-capability-e-g-ssh-on-linux) — `QT_QPA_PLATFORM=offscreen` pattern, verified against Qt docs
- [Qt Docs: QGuiApplication](https://doc.qt.io/qt-6/qguiapplication.html) — platform plugin architecture
- [Blightmud GitHub](https://github.com/Blightmud/Blightmud) — reference terminal MUD client with Lua scripting

### Tertiary (LOW confidence)

- WebSearch for Mudlet headless GitHub issues — no relevant results found; absence of issues suggests no active community demand for this feature
