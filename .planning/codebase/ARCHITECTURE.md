# Architecture

## Pattern
Single-threaded Qt6/C++20 desktop application with embedded Lua 5.1 scripting engine. Follows a layered architecture where a central `Host` object owns all subsystems for a game profile.

**Threading model:** Single-threaded — all profiles, triggers, Lua, and UI run on the main thread. Only networking runs in background threads (handled automatically by Qt's network classes).

## Layers

```
┌─────────────────────────────────────────────────────┐
│  Application Layer (mudlet.h/cpp)                   │
│  - Main window, menu, toolbar, profile management   │
├─────────────────────────────────────────────────────┤
│  Profile/Host Layer (Host.h/cpp, HostManager.h/cpp) │
│  - One Host per game connection/profile             │
│  - Owns all subsystems below                        │
├──────────────────┬──────────────────────────────────┤
│  Scripting       │  Trigger/Alias/Timer Engine       │
│  (TLuaInterp.)   │  TTrigger, TAlias, TTimer         │
│                  │  TScript, TKey, TAction           │
├──────────────────┴──────────────────────────────────┤
│  Networking Layer (ctelnet.h/cpp)                   │
│  - Telnet protocol, MXP, GMCP, MCCP                 │
├─────────────────────────────────────────────────────┤
│  Display Layer (TConsole.h/cpp)                     │
│  - Text rendering, scrollback, input bar            │
├─────────────────────────────────────────────────────┤
│  Map Layer (TMap.h/cpp, TMapView.h/cpp)             │
│  - 2D/3D map, rooms, areas, exits                   │
└─────────────────────────────────────────────────────┘
```

## Key Abstractions

- **`mudlet`** (`src/mudlet.h/cpp`) — QMainWindow subclass; singleton application object; manages profile list, UI chrome, global settings
- **`Host`** (`src/Host.h/cpp`) — per-profile container; owns telnet, Lua interpreter, trigger/alias/timer engines, console, map
- **`HostManager`** (`src/HostManager.h/cpp`) — manages lifecycle of multiple Host instances (multi-profile support)
- **`TLuaInterpreter`** (`src/TLuaInterpreter.h/cpp`) — embeds Lua 5.1 engine; exposes Mudlet API; split across multiple `.cpp` files by domain:
  - `TLuaInterpreterUI.cpp` — UI manipulation functions
  - `TLuaInterpreterMapper.cpp` — map API
  - `TLuaInterpreterNetworking.cpp` — networking API
  - `TLuaInterpreterDiscord.cpp` — Discord rich presence API
  - `TLuaInterpreterMedia.cpp` — audio/media API
  - `TLuaInterpreterTextToSpeech.cpp` — TTS API
  - `TLuaInterpreterMMCP.cpp` — MMCP chat API
  - `TLuaInterpreterMudletObjects.cpp` — script object management
- **`ctelnet`** (`src/ctelnet.h/cpp`) — telnet protocol handler; MXP, GMCP, MCCP2/3 support; connects to game server
- **`TConsole`** (`src/TConsole.h/cpp`) — main/mini console widget; text display with ANSI color, scrollback buffer, user input
- **`TMap`** (`src/TMap.h/cpp`) — map data model; rooms, areas, exits, labels; handles import/export (MMP, Mudlet native format)
- **`TMapView`** (`src/TMapView.h/cpp`) — 2D map renderer widget; 3D view via `CameraController`

## Scripting Engine Units (trigger/alias/timer tree)

Each type is a tree of objects owned by Host:
- **`TTrigger`** / `TriggerUnit` — regex/string pattern matching on incoming text
- **`TAlias`** / `AliasUnit` — pattern matching on user input
- **`TTimer`** / `TimerUnit` — time-based event firing
- **`TScript`** / `ScriptUnit` — arbitrary Lua script objects
- **`TKey`** / `KeyUnit` — keyboard shortcut bindings
- **`TAction`** / `ActionUnit` — toolbar button actions

## Data Flows

**Server input:**
```
Socket → ctelnet (decode telnet/MCCP/ANSI) → TriggerUnit (pattern match) → TLuaInterpreter (callbacks) → TConsole (display)
```

**User input:**
```
TConsole input bar → AliasUnit (pattern match) → ctelnet (send to server) / TLuaInterpreter (command)
```

**Script execution:**
```
TTimer / TTrigger / TScript → TLuaInterpreter → Lua 5.1 VM → Mudlet API functions (C++) → Host subsystems
```

**Profile save/load:**
```
Host → XML serialization (QXmlStreamWriter/Reader) → profile directory (~/.config/mudlet/profiles/)
```

## Entry Points

- **Application start:** `main.cpp` → `mudlet` constructor → loads saved profiles
- **Profile activation:** `mudlet::slot_connection_dlgFinished()` → creates `Host` → connects `ctelnet`
- **Data receive:** `ctelnet::processSocketData()` — main ingress for all server data
- **Lua API:** `TLuaInterpreter::*` functions registered via `lua_register()` in `TLuaInterpreter.cpp`
