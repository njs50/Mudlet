# Testing Patterns

**Analysis Date:** 2026-03-24

## Test Framework

**C++ Unit Tests:**
- Framework: Qt Test (`#include <QtTest/QtTest>`)
- Runner: CMake-based via `ctest`
- Config file: `/Users/njs50/play/Mudlet/test/CMakeLists.txt`

**Lua Tests:**
- Framework: Busted (Lua testing library)
- Config file: `/Users/njs50/play/Mudlet/src/mudlet-lua/tests/.busted`
- Setup: `luarocks --lua-version 5.1 install busted`

**Run Commands:**
```bash
# C++ unit tests (from build directory)
ctest

# Specific C++ test
./test/SecureStringUtilsTest

# Lua tests (from Mudlet application)
runTests /full/path/to/src/mudlet-lua/tests

# Specific Lua test
runTests /full/path/to/src/mudlet-lua/tests/Alias_spec.lua
```

## Test File Organization

**Location:**
- C++ unit tests: `/Users/njs50/play/Mudlet/test/` (co-located with source)
- C++ functional tests: `/Users/njs50/play/Mudlet/test/functional_tests/`
- Lua tests: `/Users/njs50/play/Mudlet/src/mudlet-lua/tests/` (mirroring Lua structure)

**Naming:**
- C++ tests: `*Test.cpp` (e.g., `TLuaInterfaceTest.cpp`, `SecureStringUtilsTest.cpp`)
- Lua tests: `*_spec.lua` (e.g., `Alias_spec.lua`, `StringUtils_spec.lua`)

**Lua Test Structure (from README):**
- Each test file mirrors its companion in `lua/` directory
- File organization: `tests/DB_spec.lua` tests `lua/DB.lua`

## Test Structure

**C++ Unit Test Pattern:**

```cpp
#include <QtTest/QtTest>
#include "TLuaInterpreter.h"

class TVarTest : public QObject {
Q_OBJECT

private:
    lua_State* L = luaL_newstate();
    LuaInterface* interface = new LuaInterface(L);

private slots: // NOLINT(readability-redundant-access-specifiers)

    void init()
    {
        L = luaL_newstate();
        interface = new LuaInterface(L);
    }

    void cleanup() {
        lua_close(L);
    }

    void execLua(const QString& string) {
        luaL_loadstring(L, string.toUtf8().constData());
        lua_pcall(L, 0, 0, 0);
    }

    void testRetrieveStrings()
    {
        execLua("test = '1'");
        interface->getVars(false);
        VarUnit* vu = interface->getVarUnit();
        TVar* base = vu->getBase();
        QList<TVar*> children = base->getChildren();
        TVar* testVar = children.first();
        QCOMPARE(testVar->getName(), "test");
        QCOMPARE(testVar->getValue(), "1");
        QCOMPARE(testVar->getValueType(), LUA_TSTRING);
    }
};

#include "TLuaInterfaceTest.moc"
QTEST_MAIN(TVarTest)
```

**Key Patterns:**
- `private slots:` for test methods (not `public:`)
- `init()` - setup before each test
- `cleanup()` - teardown after each test
- `initTestCase()` - setup before all tests
- `cleanupTestCase()` - teardown after all tests
- `QTEST_MAIN(ClassName)` at end to generate entry point
- `#include "ClassName.moc"` before QTEST_MAIN

**Lua Test Pattern (from `Alias_spec.lua`):**

```lua
describe("Alias processing", function()

    -- Test for nested alias processing with self-deletion (GitHub issue #8817)
    describe("nested processing", function()

        it("should not crash when inner alias kills itself during nested expandAlias", function()
            local inner_id
            local outer_executed = false
            local inner_executed = false

            local outer_id = tempAlias("^test_outer$", function()
                outer_executed = true
                expandAlias("test_inner")
            end)

            inner_id = tempAlias("^test_inner$", function()
                inner_executed = true
                killAlias(inner_id)
            end)

            -- Verify both aliases exist before the test
            assert.are.equal(1, exists(outer_id, "alias"), "Outer alias should exist before test")
            assert.are.equal(1, exists(inner_id, "alias"), "Inner alias should exist before test")

            expandAlias("test_outer")

            assert.is_true(outer_executed, "Outer alias should have executed")
            assert.is_true(inner_executed, "Inner alias should have executed")
            assert.are.equal(0, exists(inner_id, "alias"), "Inner alias should have been cleaned up")
            assert.are.equal(1, exists(outer_id, "alias"), "Outer alias should still exist")

            killAlias(outer_id)
        end)
    end)

end)
```

## Mocking

**Framework:** Qt's built-in signal/spy mechanism and Busted's spy module

**C++ Spy Pattern:**
```cpp
// From functional tests - using QSignalSpy to wait for signals
QSignalSpy(mudlet::self()->getActiveHost()->mpConsole, &TMainConsole::signal_newDataAlert).wait(200);
```

**Lua Spy Pattern (from `Regex_spec.lua`):**
```lua
local send = spy.on(_G, "send")
-- ... test code ...
-- Verify was called
assert.spy(send).was.called()
```

**What to Mock:**
- External dependencies (file I/O, network, system calls)
- Qt signals for state verification
- Global functions via Busted spies

**What NOT to Mock:**
- Core business logic under test
- Internal helper functions
- Data structures and containers

## Fixtures and Factories

**Test Data Pattern from `SecureStringUtilsTest.cpp`:**
```cpp
void SecureStringUtilsTest::testProfileBasedEncryption()
{
    QString plaintext = "mypassword";
    QString profileName = "TestProfile";

    QString encrypted = SecureStringUtils::encryptStringForProfile(plaintext, profileName);
    QVERIFY(!encrypted.isEmpty());
    QVERIFY(encrypted != plaintext);

    QString decrypted = SecureStringUtils::decryptStringForProfile(encrypted, profileName);
    QCOMPARE(decrypted, plaintext);
}
```

**Lua Test Factories (from `Alias_spec.lua`):**
```lua
local outer_id = tempAlias("^test_outer$", function()
    outer_executed = true
    expandAlias("test_inner")
end)

local inner_id = tempAlias("^test_inner$", function()
    inner_executed = true
    killAlias(inner_id)
end)
```

**Location:**
- C++ fixtures: Inline in test files using helper methods
- Lua fixtures: Use Busted's `before_each()` blocks and helper functions

## Coverage

**Requirements:** Not enforced

**Tool Integration:**
- Sanitizer support: Enabled via CMake on non-Windows platforms
- ASAN settings: `ASAN_OPTIONS=detect_leaks=0` in test environment
- Tests run with sanitizers by default (see `/Users/njs50/play/Mudlet/test/CMakeLists.txt`)

## Test Types

**Unit Tests:**
- Scope: Single class/function in isolation
- Location: `/Users/njs50/play/Mudlet/test/*.cpp`
- Examples:
  - `SecureStringUtilsTest.cpp` - Tests encryption/decryption functions
  - `TLuaInterfaceTest.cpp` - Tests Lua variable retrieval
  - `TLinkStoreTest.cpp` - Tests hyperlink storage
  - `TMxpTagParserTest.cpp` - Tests MXP tag parsing
  - `TEntityResolverTest.cpp` - Tests HTML entity resolution

**Functional/Integration Tests:**
- Scope: Multi-component interaction (e.g., Telnet server stub + console display)
- Location: `/Users/njs50/play/Mudlet/test/functional_tests/`
- Examples:
  - `TelnetTextDisplayedTest.cpp` - Tests telnet text appears in console
  - `TOscTest.cpp` - Tests OSC command handling
  - `TelnetBenchmark.cpp` - Performance baseline tests
- Environment: `QT_QPA_PLATFORM=offscreen` for headless testing
- Timeout: 60 seconds per test

**Lua Integration Tests:**
- Scope: Mudlet Lua API testing via Busted
- Location: `/Users/njs50/play/Mudlet/src/mudlet-lua/tests/*_spec.lua`
- Examples:
  - `Alias_spec.lua` - Alias matching and execution
  - `Trigger_spec.lua` - Trigger matching and execution
  - `Mapper_spec.lua` - Mapping API
  - `DB_spec.lua` - Database operations
  - `Regex_spec.lua` - Regular expression handling
- Execution: Via `runTests` command in Mudlet self-test profile

## Common Patterns

**Assertion Pattern:**
```cpp
QCOMPARE(actual, expected);           // For equality
QVERIFY(condition);                   // For boolean
QVERIFY(!condition);                  // For negation
QVERIFY(pointer);                     // For non-null
QVERIFY(!pointer);                    // For null
```

**Async Signal Testing:**
```cpp
QSignalSpy spy(object, &ClassName::signalName);
// ... trigger action ...
QVERIFY(spy.wait(200)); // Wait up to 200ms for signal
QCOMPARE(spy.count(), expectedCount);
```

**Functional Test Pattern (from `TelnetTextDisplayedTest.cpp`):**
```cpp
void TelnetTextDisplayedTest::init()
{
    mpServer = new TelnetServerStub(qApp);
    mpServer->start(mpLocalhost, mpPort.toUShort());
    mudlet::start();
    mudlet::self()->setupConfig();
    mudlet::self()->takeOwnershipOfInstanceCoordinator(std::make_unique<MudletInstanceCoordinator>("MudletInstanceCoordinator"));
    mudlet::self()->init();
    mudlet::self()->setStorePasswordsSecurely(false);
    deleteProfileDirectory(mpHostname);
}

void TelnetTextDisplayedTest::test_TelnetTextDisplayed()
{
    QString messageFromTheMud("\x1B[1z<B>Greetings < hunters & sorcerers</B>\x1B[7z");
    QString messageToExpect("Greetings < hunters & sorcerers");

    mpServer->setWelcomeMessage(messageFromTheMud);
    startProfile(mpHostname, mpLocalhost, mpPort);
    QSignalSpy(mudlet::self()->getActiveHost()->mpConsole, &TMainConsole::signal_newDataAlert).wait(200);

    QCOMPARE(mudlet::self()->getActiveHost()->mpConsole->getCurrentLine(""), messageToExpect);
}

void TelnetTextDisplayedTest::cleanup()
{
    delete mpServer;
    mpServer = nullptr;
    deleteProfileDirectory(mpHostname);
    delete mudlet::self();
}
```

**Lua Test Assertion Patterns (from `Alias_spec.lua`):**
```lua
assert.are.equal(expected, actual, "message")      -- Equality
assert.is_true(condition, "message")               -- Boolean true
assert.is_false(condition, "message")              -- Boolean false
assert.spy(mock_function).was.called()             -- Function called
assert.spy(mock_function).was.called_with(args)    -- Called with args
```

## CMake Test Configuration

**Unit Tests Configuration (from `test/CMakeLists.txt`):**
```cmake
foreach(test_name ${UNIT_TESTS})
    add_executable(${test_name} ${test_name}.cpp)
    add_dependencies(${test_name} ${LIB_MUDLET_TARGET})
    target_link_libraries(${test_name} PRIVATE Qt6::Test ${LIB_MUDLET_TARGET})
    add_test(NAME ${test_name} COMMAND $<TARGET_FILE:${test_name}>)
    set_tests_properties(${test_name} PROPERTIES
        ENVIRONMENT "ASAN_OPTIONS=detect_leaks=0"
    )
endforeach()
```

**Functional Tests Configuration (from `test/functional_tests/CMakeLists.txt`):**
```cmake
foreach(test_file ${FUNCTIONAL_TEST_SOURCES})
    get_filename_component(test_name ${test_file} NAME_WE)
    add_executable(${test_name} ${test_file} ${FUNCTIONAL_TEST_UTILS})
    add_dependencies(${test_name} ${LIB_MUDLET_TARGET})
    target_link_libraries(${test_name} PRIVATE Qt6::Test ${LIB_MUDLET_TARGET})
    set_target_properties(${test_name} PROPERTIES ENABLE_EXPORTS ON)
    add_test(NAME ${test_name} COMMAND $<TARGET_FILE:${test_name}>)
    set_tests_properties(${test_name} PROPERTIES
        ENVIRONMENT "QT_QPA_PLATFORM=offscreen;ASAN_OPTIONS=detect_leaks=0"
        LABELS "functional"
        TIMEOUT 60
    )
endforeach()
```

---

*Testing analysis: 2026-03-24*
