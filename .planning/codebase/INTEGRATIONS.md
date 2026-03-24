# External Integrations

**Analysis Date:** 2026-03-24

## APIs & External Services

**Discord Rich Presence:**
- Discord RPC - Shows in-game status in Discord user profile
  - SDK/Client: Discord RPC library (3rdparty/discord)
  - Application ID: `450571881909583884` (Mudlet's official ID, hardcoded in `src/discord.cpp`)
  - Custom Game Support: Users can set custom Discord App IDs for game-specific integration
  - Auth: No authentication required; uses local IPC to Discord client
  - Implementation: `src/discord.h`, `src/discord.cpp`
  - Features: Rich presence with game state, details, timestamps, images, party info, secrets

**Telnet Protocol (MUD Connectivity):**
- Telnet with negotiation (WILL/WONT/DO/DONT commands)
  - Implementation: `src/ctelnet.h`, `src/ctelnet.cpp`
  - Features: Option negotiation, color support (ANSI, VT100)
  - MTTS (MUD Text-To-Speech) support flags
  - TLS/SSL support via `QSslSocket`
  - Compression support (zlib) for MUD server responses
  - Encoding support: UTF-8, GB2312/GBK/GB18030, BIG5, EUC-KR, Latin-1

**GMCP (Generic MUD Communication Protocol):**
- GMCP authentication and messaging
  - Implementation: `src/GMCPAuthenticator.h/cpp`
  - Usage: `src/Host.h` integrates GMCP for game-specific data exchange
  - Features: Supports standard GMCP packages and custom authentication types

**MXP (MUD eXtension Protocol):**
- Full MXP protocol support for enhanced rendering
  - Implementation: `src/TMxpProcessor.h/cpp`, `src/TMxpClient.h`, `src/TMxpNodeBuilder.h`
  - Tag handlers: `src/TMxpColorTagHandler.h`, `src/TMxpImageTagHandler.h`, `src/TMxpLinkTagHandler.h`, `src/TMxpSoundTagHandler.h`, etc.
  - Features: Custom elements, links, colors, fonts, images, sounds, frames, entity definitions

**IRC Protocol (In-Game Chat Relay):**
- IRC chat integration via Communi library
  - SDK/Client: Communi (3rdparty/communi)
  - Implementation: `src/dlgIRC.h/cpp`
  - Features: Connect to IRC servers, relay game chat to IRC channels

**MMCP (MUD Client Control Protocol):**
- MMCP server for inter-mudlet communication
  - Implementation: `src/MMCPServer.h/cpp`, `src/MMCPClient.h/cpp`, `src/MMCP.h`
  - Configuration: `src/Host.h` contains MMCP settings
  - Features: Chat relay, peek requests, emote prefixing, auto-call handling

**Map Download:**
- Download MUD maps from external sources
  - Implementation: `src/TMap.h/cpp` with `QNetworkAccessManager`
  - Network error handling: `slot_downloadError()`, `slot_replyFinished()`

## Data Storage

**Local File Storage:**
- Profile data: XML format (profiles stored as user-editable XML files)
  - Import/Export: `src/XMLimport.h/cpp`, `src/XMLexport.h/cpp`
  - Triggers, aliases, timers, keys, scripts stored as XML
  - Undo/Redo support via XML snapshots in `src/EditorItemXMLHelpers.h`

**Maps:**
- Binary map format with optional autosave
  - Implementation: `src/TMap.h/cpp`
  - Features: Room database, areas, exits, user data persistence

**Configuration:**
- Mudlet runtime settings stored locally
  - Window geometry, preferences, profile list
  - Settings persistence via Qt's settings system

**No Remote Database:**
- Mudlet does NOT use cloud storage or remote databases
- All data stays local on the user's machine

## Authentication & Identity

**Discord User Validation:**
- Optional Discord user validation for Rich Presence
  - Implementation: `src/discord.h` - `discordUserIdMatch()`, `getDiscordUserDetails()`
  - Purpose: Optional character-name revelation tied to Discord user verification
  - No backend auth required; local verification only

**Credential Storage:**
- Secure local storage via QtKeychain
  - SDK: Qt6Keychain (system library or built from 3rdparty/qtkeychain on macOS)
  - Implementation: `src/CredentialManager.h/cpp`
  - Use case: Storing game connection credentials securely per profile
  - Storage backends: System keychain (Windows Credential Manager, macOS Keychain, Linux Secret Service)

**MUD Game Login:**
- Game credentials stored locally via `CredentialManager`
  - Support for credential validation via GMCP authentication
  - Implementation: `src/GMCPAuthenticator.h/cpp` for GMCP-based auth flows

**No OAuth/External Auth:**
- Mudlet does not integrate with OAuth providers
- Local credentials only

## Monitoring & Observability

**Error Tracking (Optional):**
- Sentry crash reporting (disabled by default)
  - SDK: sentry-native (3rdparty/sentry-native)
  - Build flag: `WITH_SENTRY=ON` to enable
  - Implementation: `src/SentryWrapper.h/cpp`, `crash_reporter/` subdirectory
  - Features: Automatic crash report capture, symbol uploads, debug file transmission
  - Control: `SENTRY_SEND_DEBUG=ON` for automatic debug symbol upload

**Application Logging:**
- Qt's qDebug() system for console/file logging
  - Conditional debug logging via environment variable compilation
  - No centralized log aggregation

**Text-to-Speech (Optional):**
- Text-to-speech via Qt6::TextToSpeech module
  - Implementation: `src/TLuaInterpreterTextToSpeech.cpp`, `src/TLuaInterpreter.h` - `ttsStateChanged()`
  - Features: Automatic text-to-speech for output lines (configurable per profile)
  - Platform support: Windows (SAPI), macOS (AVSpeechSynthesizer), Linux (espeak/festival)

## CI/CD & Deployment

**Hosting/Deployment:**
- GitHub releases (manual upload or CI artifacts)
- Self-hosted auto-update server (DBLSQD) for Linux/Windows (optional feature)
- macOS: Sparkle framework for native app updates (3rdparty/sparkle-glue)

**CI/CD Pipeline:**
- GitHub Actions workflows (`.github/workflows/`)
  - `build-mudlet.yml` - Linux/macOS builds
  - `build-mudlet-win.yml` - Windows builds
  - `clangtidy-diff-analysis.yml` - Static analysis
  - `codeql-analysis.yml` - CodeQL security scanning
  - `performance-analysis.yml` - Performance benchmarking
  - `generate-changelog.yml` - Automated changelog generation
  - `update-translations.yml` - Translation updates
  - `update-3rdparty.yml` - Dependency version updates
  - `link-ptbs-to-dblsqd.yml` - Release linking to update servers

## Environment Configuration

**Required env vars (Optional features):**
- `WITH_UPDATER` - Enable auto-updater compilation (Linux, Windows, macOS)
- `WITH_FONTS` - Include bundled font resources
- `WITH_3DMAPPER` - Compile 3D mapper support
- `WITH_SHADER_HOT_RELOAD` - Enable shader hot-reload in 3D mapper
- `WITH_VARIABLE_SPLASH_SCREEN` - Include build-type splash screens
- `WITH_SENTRY` - Enable crash reporting compilation
- `SENTRY_SEND_DEBUG` - Auto-upload debug symbols to Sentry
- `MUDLET_VERSION_BUILD` - Custom build version string (else uses git SHA)
- `BUILD_COMMIT` - Override git SHA detection for build ID
- `USE_ALTERNATE_LINKER` - Select alternate linker (gold, lld, bfd, mold)

**Secrets location:**
- No secrets in codebase
- Credentials stored securely via system keychain (QtKeychain)
- Discord API calls use local IPC (no credentials sent)

## Webhooks & Callbacks

**Incoming Webhooks:**
- No HTTP webhook server implemented
- MMCP server accepts TCP connections from other Mudlet instances (port configurable)
  - Implementation: `src/MMCPServer.h/cpp`
  - Protocol: MMCP over TCP/IP

**Outgoing Webhooks:**
- HTTP requests via Lua API
  - Implementation: `src/TLuaInterpreterNetworking.cpp`
  - Methods: `sendHttpRequest()`, `getUrl()`, `postUrl()`, `headUrl()`, `deleteUrl()`, `putUrl()`, etc.
  - Cookie support: `QNetworkCookieJar` for session persistence
  - Proxy support: `QNetworkProxy` configuration per connection

**Discord Event Callbacks:**
- Discord RPC event handlers (internal only)
  - Handlers: `handleDiscordReady()`, `handleDiscordDisconnected()`, `handleDiscordError()`, `handleDiscordJoinGame()`, `handleDiscordSpectateGame()`, `handleDiscordJoinRequest()`
  - Implementation: `src/discord.cpp`
  - User-facing: Via Lua event system if needed

**MUD Server Callbacks:**
- MXP/MCP/GMCP packet handling
  - Parsed via protocol handlers, triggers user-defined scripts
  - No external webhooks; all internal event propagation

## Map Download Integration

**Public Map Sources:**
- Download maps from external URLs
  - Implementation: `src/TMap.h` - `slot_replyFinished()`, `slot_downloadError()`
  - Network handling: `QNetworkAccessManager` with SSL support
  - Error handling: Network timeouts, DNS failures, HTTP errors
  - No built-in map server; users must provide URLs

## Scripting & Extensibility

**Lua API HTTP Endpoints:**
- Full HTTP support via Lua API
  - `http.get()`, `http.post()`, `http.put()`, `http.delete()`, `http.head()`
  - Cookie jar persistence per request
  - Custom headers, timeouts, redirects
  - Response headers and status code exposed to scripts

**Lua Modules:**
- Dynamic module loading support (symbol exports enabled)
  - Modules can call Lua C API functions (`lua_gettop`, etc.)
  - Located at: `src/mudlet-lua/`
  - Busted test framework for unit testing Lua API

---

*Integration audit: 2026-03-24*
