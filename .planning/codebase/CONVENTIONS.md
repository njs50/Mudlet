# Coding Conventions

**Analysis Date:** 2026-03-24

## Naming Patterns

**Files:**
- Classes: PascalCase without prefix (e.g., `Host.h`, `TConsole.h`)
- Main Qt classes: PascalCase with `T` prefix (e.g., `TLuaInterpreter.h`, `TAlias.h`, `TConsole.h`)
- Dialog classes: `dlg` prefix in lowercase (e.g., `dlgTriggerEditor.h`, `dlgConnectionProfiles.h`)
- Test files: `*Test.cpp` for unit tests, `*_spec.lua` for Lua tests
- Implementation split: Core logic in `TLuaInterpreterXxx.cpp` files (e.g., `TLuaInterpreterUI.cpp`, `TLuaInterpreterMapper.cpp`)

**Classes:**
- PascalCase with optional `T` prefix for main domain classes: `class TConsole`, `class Host`, `class TAlias`
- For non-T classes: simple PascalCase: `class SecureStringUtils`, `class CredentialManager`

**Member Variables:**
- camelCase with `m` prefix: `mProfileName`, `mIsRunning`, `mElapsedTime`
- Private pointer members: `mp` prefix: `mpHost`, `mpConsole`, `mpMyChildrenList`
- Qt member pointers: `mp` prefix: `mpServer`, `mpRegex`

**Functions/Methods:**
- camelCase: `getProfileName()`, `setName()`, `match()`, `isActive()`
- Qt signal/slot names: camelCase: `signal_newDataAlert`, `slot_updatePassword()`
- Private slot methods: `slot_` prefix for clarity: `slot_load()`, `slot_updateDescription()`

**Variables:**
- Local variables: camelCase: `newName`, `profileName`, `haystackLength`
- Constants: `UPPERCASE_WITH_UNDERSCORES` in constexpr: `ENCRYPTION_VERSION_CURRENT`, `SALT_SIZE`
- Boolean members/params: start with `is` or `has`: `isActive()`, `mIsRunning`, `mIsInitialised`

**Type Names:**
- Using types: `using NamedMatchesRanges = QMap<QString, QPair<int, int>>;`
- Struct names: PascalCase with inline definition: `struct TFontAttributes`

## Code Style

**Formatting:**
- Tool: clang-format (`.clang-format` in `src/`)
- Configuration file: `/Users/njs50/play/Mudlet/src/.clang-format`
- Key settings:
  - Column limit: 200 characters
  - Indentation: 4 spaces (never tabs)
  - Pointer alignment: Left (`int* ptr` not `int *ptr`)
  - Constructor initializers: One per line with `BreakConstructorInitializers: BeforeComma`
  - Brace style: LLVM-based custom with braces after classes/functions/structs

**Linting:**
- Tool: clang-tidy via `.clang-tidy` configuration file
- Run before committing C++ code: `clang-format -i path/to/file.cpp path/to/file.h`
- On macOS use Homebrew LLVM: `$(brew --prefix llvm)/bin/clang-format -i file.cpp`

## Import Organization

**Order in C++ Files:**
1. Local project headers (with `#include "..."`)
2. Qt headers (with `#include <Qt*...>`)
3. Third-party headers (with `#include <boost/...>`, `#include <edbee/...>`)
4. System headers (with `#include <...>`)
5. Lua headers (wrapped in `extern "C"` block)
6. STL headers at end

**Example Pattern from `TLuaInterpreter.h`:**
```cpp
#include "TMediaData.h"
#include "TTrigger.h"
#include "utils.h"

#include <QEvent>
#include <QFileSystemWatcher>
#include <QNetworkAccessManager>

#include <edbee/texteditorwidget.h>

extern "C" {
#if defined(INCLUDE_VERSIONED_LUA_HEADERS)
#include <lua5.1/lauxlib.h>
#include <lua5.1/lua.h>
#include <lua5.1/lualib.h>
#else
#include <lauxlib.h>
#include <lua.h>
#include <lualib.h>
#endif
}

#include <list>
#include <string>
#include <memory>
#include <optional>
```

**Include Minimization Rules (Critical for build speed):**
- In headers: Use forward declarations instead of includes whenever possible
  - Example: `class Host;` instead of `#include "Host.h"` when only pointers/references needed
  - See `Host.h` lines 62-78 for forward declaration patterns
- In source files: Only include headers actually used in that file
- No "just in case" includes - verify each one is needed

## Error Handling

**Qt-Style Error Handling Pattern:**
```cpp
if (!file.open(QIODevice::ReadOnly)) {
    qWarning() << "Failed to open file:" << file.errorString();
    return false;
}
```

**Logging Macros:**
- `qWarning()` - Error conditions that should be reported
- `qDebug()` - Informational messages (use with `.noquote().nospace()` for formatting control)
- `qCritical()` - Severe errors
- `qInfo()` - General information
- Pattern: Include function context in messages: `"dlgConnectionProfiles::slot_updatePassword() ERROR - ..."`

**Return Conventions:**
- Boolean methods return `true` on success, `false` on failure
- Pointer methods return `nullptr` on failure (never throw exceptions)
- Methods frequently use early returns for guard conditions:
  ```cpp
  if (!mpHost) {
      return;
  }
  ```

## Logging

**Framework:** Qt's logging system via `qDebug()`, `qWarning()`, etc.

**Patterns Observed:**
- Include class and method name in message: `"dlgConnectionProfiles::writeSecurePassword() ERROR - ..."`
- Use noquote/nospace for precise formatting control:
  ```cpp
  qDebug().noquote().nospace() << "TMxpProcessor::setMode(...) INFO - ...";
  ```
- Log errors with context about what operation failed and why
- Conditional debug logging for detailed internal state

## Comments

**When to Comment:**
- Only for unintuitive situations and "why" decisions
- NO comments for obvious code - reduces cognitive load
- Comments explain non-obvious implementation choices

**Example from `TAlias.cpp`:**
```cpp
// Guard against re-entrancy: cleanup may have deleted this alias while
// match() was still on the call stack
if (!mpMyChildrenList) {
    qWarning() << "TAlias::match() called on destroyed alias - ID:" << mID << "Name:" << mName;
    return false;
}
```

**JSDoc/TSDoc:**
- Used for public API documentation in `.h` files
- Example from `SecureStringUtils.h`:
  ```cpp
  /**
   * @brief Encrypt a string using a profile-specific encryption key
   * @param plaintext The string to encrypt
   * @param profileName Name of the profile (used for key lookup)
   * @return Base64-encoded encrypted string, or empty string if input is empty
   */
  static QString encryptStringForProfile(const QString& plaintext, const QString& profileName);
  ```

## Function Design

**Size Guidelines:**
- Prefer small, focused functions (no strict line limit, but readability is key)
- Extract helper methods for complex logic

**Parameters:**
- Use const references for large objects: `const QString&`, `const QList<T>&`
- Pass by value for small types: `int`, `bool`, `qint64`
- Pointers for optional outputs: `QByteArray& output` (modified in-place)

**Return Values:**
- Use meaningful types: `bool` for success/failure, `QString` for strings, `nullptr` for "not found" pointers
- Modern C++: Use `std::optional<T>` and `std::tuple<>` when appropriate
- Example from `GifTracker.cpp`: `std::tuple<QString, int, int> assembleReport();`

## Module Design

**Exports:**
- Public API in `.h` files with `public:` section
- Private implementation details in `.cpp` files
- Member functions use `public:`, `protected:`, `private:` access levels clearly

**Barrel Files:**
- Not commonly used; instead split functionality into separate headers
- Large classes split across implementation files (e.g., `TLuaInterpreter` split into `TLuaInterpreterUI.cpp`, `TLuaInterpreterMapper.cpp`, etc.)

**Example Structure from `Host.h` and `Host.cpp`:**
- Core declarations in header
- Friend relationships defined: `friend class XMLexport; friend class XMLimport;`
- Implementation distributed across related CPP files by domain

## Modern C++20 Practices

**Used Actively:**
- Range-based for loops: `for (auto item : container) { ... }`
- Auto type deduction: `auto nodes = TMxpTagParser::parseToMxpNodeList(text);`
- Smart pointers: `QSharedPointer<T>`, `std::unique_ptr<T>`, `std::make_unique<T>()`
- Default comparisons: `bool operator==(const Type& other) const = default;` (TFontAttributes example)
- Constexpr for compile-time constants

**NOT Used (Intentional Avoidance):**
- Exceptions - avoided for performance/complexity reasons
- Complex templates - kept simple for maintainability
- Concepts - avoided per project philosophy

## String Handling

**String Literals:**
- Use `qsl()` macro (QStringLiteral) for all string literals: `QString name = qsl("timer");`
- For user-visible strings use `tr()` for translation: `tr("Connection failed: %1")`
- Always include translator comments with `//:` prefix:
  ```cpp
  //: Toast notification shown when user dismisses an editor tip banner
  QString message = tr("Banner hidden. <a href='undo'>Undo</a>");
  ```

**In .ui XML files:**
- Set `notr="true"` on strings that don't need translation:
  ```xml
  <property name="text">
    <string notr="true">-</string>
  </property>
  ```

**Memory and String Safety:**
- Use `QString` for UI/string operations
- Use `QByteArray` for binary data
- Secure clearing: `SecureStringUtils::secureStringClear(QString&)` for sensitive data
- Example from `SecureStringUtils.h`: Profile-based encryption with PBKDF2-SHA256

---

*Convention analysis: 2026-03-24*
