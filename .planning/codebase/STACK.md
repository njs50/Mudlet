# Technology Stack

**Analysis Date:** 2026-03-24

## Languages

**Primary:**
- C++ 20 - Core application codebase, emphasis on modern C++20 without exceptions/templates for performance
- Lua 5.1 - Scripting engine for user scripts and game automation

**Secondary:**
- Objective-C/Objective-C++ - macOS-specific integrations (Sparkle updater)
- Shell/CMake - Build configuration and platform-specific builds

## Runtime

**Environment:**
- Cross-platform desktop application: Windows, macOS (12.0+), Linux, FreeBSD
- Single-threaded: All profiles, triggers, and Lua engine run on main thread (networking via Qt background)
- Qt runtime integrated for UI and platform abstraction

**Package Manager:**
- CMake 3.25.1+ (primary build system)
- Git submodules for third-party dependencies
- No traditional package manager; all deps managed via CMake `find_package()`

## Frameworks

**Core UI:**
- Qt 6.8.2+ (minimum 6.4.0 for some components)
  - Qt6::Core - Core functionality
  - Qt6::Widgets - UI components
  - Qt6::Gui - Graphics and rendering
  - Qt6::Network - Networking (telnet, HTTP requests)
  - Qt6::Multimedia - Audio/video playback
  - Qt6::MultimediaWidgets - Multimedia UI widgets
  - Qt6::OpenGL - 3D mapper support
  - Qt6::UiTools - Dynamic UI loading
  - Qt6::Concurrent - Threading support
  - Qt6::TextToSpeech (optional) - Text-to-speech synthesis

**Scripting:**
- Lua 5.1 (exact version) - Game automation and custom triggers
- Located at: `src/TLuaInterpreter.h/cpp`, `src/TLuaInterpreterNetworking.cpp`, etc.
- Lua modules support dynamic loading (symbol exports enabled)

**Editor & UI Widgets:**
- edbee-lib (3rdparty/edbee-lib) - Source code editor widget
- qt-tags-widget (3rdparty/qt-tags-widget) - Tag input widget
- QTagEdit - Tag/autocomplete editing

**Networking (IRC):**
- Communi (3rdparty/communi) - IRC protocol client library
- Used by: `dlgIRC.h/cpp` for in-game chat relay

## Key Dependencies

**Critical:**
- Lua51 5.1.5 - Scripting engine, loaded at runtime
- PCRE2 - Regular expression engine for triggers
- ZLIB - Data compression (telnet MUD protocol)
- ZIP - Archive handling for package import/export
- PugiXML - XML parsing for profile storage
- Hunspell - Spell checking for in-game text

**Infrastructure:**
- Boost 1.44+ - Utility library (headers only)
- Qt6Keychain - Secure credential storage (macOS: built from source at 3rdparty/qtkeychain)
- assimp (optional) - 3D model loading for 3D mapper
- OpenGL/GLU (optional) - 3D rendering for 3D mapper

**Optional Features:**
- DBLSQD (3rdparty/dblsqd) - Updater system (only if WITH_UPDATER enabled)
- Sparkle 2.7.1 (macOS only) - Native macOS app updater via Sparkle framework
- Sentry SDK (3rdparty/sentry-native) - Crash reporting (only if WITH_SENTRY enabled)

**Development Tools:**
- Discord RPC (3rdparty/discord) - Discord Rich Presence integration
- LCF (3rdparty/lcf) - Lua Code Formatter for user scripts

## Configuration

**Build-time Options:**
Located in `/Users/njs50/play/Mudlet/CMakeLists.txt`

- `WITH_SENTRY` (OFF by default) - Enable crash reporting to Sentry
- `SENTRY_SEND_DEBUG` (OFF) - Send debug symbols to Sentry after build
- `WITH_UPDATER` (environment-driven) - Enable auto-updater (Linux, Windows, macOS)
- `WITH_FONTS` (environment-driven) - Include bundled fonts
- `WITH_3DMAPPER` (environment-driven) - Enable 3D room mapper
- `WITH_SHADER_HOT_RELOAD` (environment-driven, OFF default) - Shader hot-reload in 3D mapper
- `WITH_VARIABLE_SPLASH_SCREEN` (environment-driven) - Build-type dependent splash screens
- `USE_ALTERNATE_LINKER` (empty, can be 'gold', 'lld', 'bfd', 'mold') - Alternate linker selection

**Conditional Compilation (Debugging):**
Located in `src/CMakeLists.txt`, can be uncommented for debugging:

- `DEBUG_TELNET` - Telnet protocol negotiation messages
- `DEBUG_UTF8_PROCESSING` - UTF-8 decoding traces
- `DEBUG_GB_PROCESSING` - GB2312/GBK/GB18030 decoding
- `DEBUG_BIG5_PROCESSING` - BIG5 encoding traces
- `DEBUG_EUC_KR_PROCESSING` - EUC-KR decoding traces
- `DEBUG_SGR_PROCESSING` - ANSI Select Graphic Rendition sequences
- `DEBUG_OSC_PROCESSING` - ANSI Operating System Command sequences
- `DEBUG_MXP_PROCESSING` - MXP protocol tag handling
- `DEBUG_WINDOW_HANDLING` - UI window operations
- `DEBUG_MAPAUTOSAVE` - Map file autosave events
- `DEBUG_EASTER_EGGS` - Always show Easter eggs
- `DEBUG_UNDO_REDO` - Undo/redo stack operations

**Build Type & Version:**
- `APP_VERSION` - Set to 4.20.1 (major.minor.patch)
- `APP_BUILD` - Auto-generated from git SHA1 or environment `MUDLET_VERSION_BUILD`
  - Format: `{BUILD_VALUE}-{GIT_SHA1}` or `-dev-{GIT_SHA1}` for development builds
  - Written to `src/app-build.txt` during CMake configuration

**Platform-Specific:**
- macOS: `CMAKE_OSX_DEPLOYMENT_TARGET` = 12.0
- Windows: `UNICODE` and `_UNICODE` definitions for path handling
- FreeBSD: Links against system `libsysinfo` and `libkvm` for sysinfo

**Compiler Settings:**
- Standard: C++20 (CMAKE_CXX_STANDARD = 20)
- Link-time optimization (LTO) enabled in Release builds
- Compiler: Supports GCC, Clang, MSVC (checked at build time)
- Warning suppression: `-Wno-deprecated` flag applied globally

## Platform Requirements

**Development:**
- CMake 3.25.1+
- C++20 capable compiler (GCC 10+, Clang 12+, MSVC 2019+)
- Qt 6.8.2+ SDK (or 6.4.0+ for core components)
- Lua 5.1 dev libraries
- Optional: ccache for faster rebuilds

**Production:**
- **Windows:** Windows 7+ (via Qt6 minimum), x86/x64
- **macOS:** macOS 12.0+ (Monterey), Intel/Apple Silicon
- **Linux:** Most modern distributions, x86/x64
- **FreeBSD:** Tested platforms supported

**Build Time:**
- Full clean build: up to 10 minutes (check CMakeLists.txt comments)
- Incremental with ccache: significantly faster
- Link-time optimization adds overhead to final link step

---

*Stack analysis: 2026-03-24*
