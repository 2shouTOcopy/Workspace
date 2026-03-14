---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring C++ code. Enforces test-driven development with 80%+ coverage including unit tests and integration tests using Google Test (gtest) and Google Mock (gmock).
---

# Test-Driven Development Workflow (C++ / GTest)

This skill ensures all C++ code development follows TDD principles with comprehensive test coverage using Google Test and Google Mock.

## When to Activate

- Writing new C++ features or functionality
- Fixing bugs or issues in C++ code
- Refactoring existing C++ code
- Adding new modules, classes, or libraries
- Creating new interfaces or abstract classes

## Core Principles

### 1. Tests BEFORE Code
ALWAYS write tests first, then implement code to make tests pass.

### 2. Coverage Requirements
- Minimum 80% coverage (unit + integration)
- All edge cases covered
- Error scenarios tested
- Boundary conditions verified

### 3. Test Types

#### Unit Tests
- Individual functions and methods
- Class behavior and state transitions
- Pure functions and utility helpers
- Template instantiations
- RAII and resource management

#### Integration Tests
- Module interactions
- File I/O operations
- Network communication
- Inter-process communication (IPC)
- Hardware abstraction layer (HAL) interactions

## TDD Workflow Steps

### Step 1: Write User Journeys
```
As a [role], I want to [action], so that [benefit]

Example:
As a developer, I want to parse camera configuration from JSON,
so that I can dynamically configure the camera sensor at startup.
```

### Step 2: Generate Test Cases
For each user journey, create comprehensive test cases:

```cpp
#include <gtest/gtest.h>
#include "camera_config.h"

class CameraConfigTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Common test setup
    }

    void TearDown() override {
        // Common test cleanup
    }
};

TEST_F(CameraConfigTest, ParsesValidConfigFile) {
    // Test implementation
}

TEST_F(CameraConfigTest, HandlesEmptyConfigGracefully) {
    // Test edge case
}

TEST_F(CameraConfigTest, FallsBackToDefaultsOnMissingFields) {
    // Test fallback behavior
}

TEST_F(CameraConfigTest, RejectsMalformedJson) {
    // Test error handling
}
```

### Step 3: Run Tests (They Should Fail)
```bash
# Build and run tests - they should fail since we haven't implemented yet
cmake --build build --target tests
ctest --test-dir build --output-on-failure
```

### Step 4: Implement Code
Write minimal code to make tests pass:

```cpp
// Implementation guided by tests
class CameraConfig {
public:
    static CameraConfig fromFile(const std::string& path);
    // Implementation here
};
```

### Step 5: Run Tests Again
```bash
cmake --build build --target tests
ctest --test-dir build --output-on-failure
# Tests should now pass
```

### Step 6: Refactor
Improve code quality while keeping tests green:
- Remove duplication
- Improve naming
- Optimize performance
- Enhance readability
- Apply RAII patterns

### Step 7: Verify Coverage
```bash
# Build with coverage flags and generate report
cmake -DCMAKE_CXX_FLAGS="--coverage -fprofile-arcs -ftest-coverage" ..
cmake --build build --target tests
ctest --test-dir build
gcovr --root . --html --html-details -o coverage/index.html
# Verify 80%+ coverage achieved
```

## Testing Patterns

### Unit Test Pattern (GTest)
```cpp
#include <gtest/gtest.h>
#include "string_utils.h"

TEST(StringUtilsTest, TrimsLeadingWhitespace) {
    EXPECT_EQ(trim("  hello"), "hello");
}

TEST(StringUtilsTest, TrimsTrailingWhitespace) {
    EXPECT_EQ(trim("hello  "), "hello");
}

TEST(StringUtilsTest, HandlesEmptyString) {
    EXPECT_EQ(trim(""), "");
}

TEST(StringUtilsTest, HandlesNullptr) {
    EXPECT_THROW(trim(nullptr), std::invalid_argument);
}
```

### Class Unit Test Pattern (GTest Fixture)
```cpp
#include <gtest/gtest.h>
#include "command_parser.h"

class CommandParserTest : public ::testing::Test {
protected:
    CommandParser parser_;

    void SetUp() override {
        parser_.registerCommand("start", CommandType::Start);
        parser_.registerCommand("stop", CommandType::Stop);
    }
};

TEST_F(CommandParserTest, ParsesKnownCommand) {
    auto result = parser_.parse("start");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->type, CommandType::Start);
}

TEST_F(CommandParserTest, ReturnsNulloptForUnknownCommand) {
    auto result = parser_.parse("unknown");
    EXPECT_FALSE(result.has_value());
}

TEST_F(CommandParserTest, IsCaseInsensitive) {
    auto result = parser_.parse("START");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->type, CommandType::Start);
}
```

### Parameterized Test Pattern
```cpp
#include <gtest/gtest.h>
#include "validator.h"

struct ValidationTestCase {
    std::string input;
    bool expected_valid;
    std::string description;
};

class ValidatorTest : public ::testing::TestWithParam<ValidationTestCase> {};

TEST_P(ValidatorTest, ValidatesInput) {
    auto [input, expected_valid, description] = GetParam();
    EXPECT_EQ(Validator::isValid(input), expected_valid) << description;
}

INSTANTIATE_TEST_SUITE_P(
    ValidationCases,
    ValidatorTest,
    ::testing::Values(
        ValidationTestCase{"valid_input", true, "accepts valid input"},
        ValidationTestCase{"", false, "rejects empty string"},
        ValidationTestCase{"too_long_" + std::string(1000, 'x'), false, "rejects oversized input"},
        ValidationTestCase{"special!@#chars", false, "rejects special characters"}
    )
);
```

### Integration Test Pattern
```cpp
#include <gtest/gtest.h>
#include <fstream>
#include <filesystem>
#include "config_manager.h"

class ConfigManagerIntegrationTest : public ::testing::Test {
protected:
    std::filesystem::path temp_dir_;

    void SetUp() override {
        temp_dir_ = std::filesystem::temp_directory_path() / "config_test";
        std::filesystem::create_directories(temp_dir_);
    }

    void TearDown() override {
        std::filesystem::remove_all(temp_dir_);
    }

    void writeConfigFile(const std::string& filename, const std::string& content) {
        std::ofstream ofs(temp_dir_ / filename);
        ofs << content;
    }
};

TEST_F(ConfigManagerIntegrationTest, LoadsConfigFromDisk) {
    writeConfigFile("app.json", R"({"port": 8080, "host": "localhost"})");

    ConfigManager mgr;
    auto result = mgr.load(temp_dir_ / "app.json");

    ASSERT_TRUE(result.ok());
    EXPECT_EQ(mgr.getInt("port"), 8080);
    EXPECT_EQ(mgr.getString("host"), "localhost");
}

TEST_F(ConfigManagerIntegrationTest, HandlesCorruptedFile) {
    writeConfigFile("bad.json", "not valid json{{{");

    ConfigManager mgr;
    auto result = mgr.load(temp_dir_ / "bad.json");

    EXPECT_FALSE(result.ok());
    EXPECT_NE(result.error().find("parse"), std::string::npos);
}
```

## Mocking with Google Mock (gmock)

### Interface Mock
```cpp
#include <gmock/gmock.h>
#include "database_interface.h"

class MockDatabase : public IDatabaseConnection {
public:
    MOCK_METHOD(bool, connect, (const std::string& connStr), (override));
    MOCK_METHOD(QueryResult, execute, (const std::string& query), (override));
    MOCK_METHOD(void, disconnect, (), (override));
};
```

### Using Mock in Tests
```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "user_service.h"

using ::testing::Return;
using ::testing::_;
using ::testing::HasSubstr;

class UserServiceTest : public ::testing::Test {
protected:
    MockDatabase mock_db_;
    std::unique_ptr<UserService> service_;

    void SetUp() override {
        service_ = std::make_unique<UserService>(&mock_db_);
    }
};

TEST_F(UserServiceTest, FindsUserById) {
    QueryResult mock_result;
    mock_result.rows = {{"1", "Alice", "alice@example.com"}};

    EXPECT_CALL(mock_db_, execute(HasSubstr("SELECT")))
        .WillOnce(Return(mock_result));

    auto user = service_->findById(1);
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(user->name, "Alice");
}

TEST_F(UserServiceTest, ReturnsNulloptForMissingUser) {
    QueryResult empty_result;

    EXPECT_CALL(mock_db_, execute(_))
        .WillOnce(Return(empty_result));

    auto user = service_->findById(999);
    EXPECT_FALSE(user.has_value());
}
```

### Mocking Hardware / External Dependencies
```cpp
#include <gmock/gmock.h>
#include "camera_interface.h"

class MockCamera : public ICameraDevice {
public:
    MOCK_METHOD(bool, open, (int deviceId), (override));
    MOCK_METHOD(Frame, captureFrame, (), (override));
    MOCK_METHOD(bool, setResolution, (int width, int height), (override));
    MOCK_METHOD(void, close, (), (override));
};

class MockNetworkClient : public INetworkClient {
public:
    MOCK_METHOD(Response, post, (const std::string& url, const std::string& body), (override));
    MOCK_METHOD(Response, get, (const std::string& url), (override));
    MOCK_METHOD(bool, isConnected, (), (const, override));
};
```

## Test File Organization

```
project/
├── src/
│   ├── camera/
│   │   ├── camera_config.h
│   │   ├── camera_config.cpp
│   │   ├── camera_device.h
│   │   └── camera_device.cpp
│   ├── command/
│   │   ├── command_parser.h
│   │   └── command_parser.cpp
│   └── utils/
│       ├── string_utils.h
│       └── string_utils.cpp
├── tests/
│   ├── CMakeLists.txt                    # Test build configuration
│   ├── unit/
│   │   ├── camera_config_test.cpp        # Unit tests
│   │   ├── command_parser_test.cpp
│   │   └── string_utils_test.cpp
│   ├── integration/
│   │   ├── config_manager_test.cpp       # Integration tests
│   │   └── camera_pipeline_test.cpp
│   ├── mocks/
│   │   ├── mock_camera.h                 # Mock definitions
│   │   ├── mock_database.h
│   │   └── mock_network.h
│   └── test_main.cpp                     # Custom main (optional)
└── CMakeLists.txt                        # Root CMake with testing enabled
```

## CMake Test Configuration

### Root CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.14)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Enable testing
enable_testing()

# Fetch Google Test
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
FetchContent_MakeAvailable(googletest)

# Add source library
add_subdirectory(src)

# Add tests
add_subdirectory(tests)
```

### tests/CMakeLists.txt
```cmake
# Unit tests
add_executable(unit_tests
    unit/camera_config_test.cpp
    unit/command_parser_test.cpp
    unit/string_utils_test.cpp
)
target_link_libraries(unit_tests
    PRIVATE
        GTest::gtest_main
        GTest::gmock
        my_project_lib
)

# Integration tests
add_executable(integration_tests
    integration/config_manager_test.cpp
    integration/camera_pipeline_test.cpp
)
target_link_libraries(integration_tests
    PRIVATE
        GTest::gtest_main
        GTest::gmock
        my_project_lib
)

# Register tests with CTest
include(GoogleTest)
gtest_discover_tests(unit_tests)
gtest_discover_tests(integration_tests)
```

## Test Coverage Verification

### Build with Coverage
```bash
# Configure with coverage flags (GCC/Clang)
cmake -B build -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_FLAGS="--coverage -fprofile-arcs -ftest-coverage"

# Build and run tests
cmake --build build
ctest --test-dir build --output-on-failure

# Generate coverage report with gcovr
gcovr --root . --filter src/ --html --html-details -o build/coverage/index.html

# Or generate with lcov
lcov --capture --directory build --output-file build/coverage.info
lcov --remove build/coverage.info '/usr/*' '*/tests/*' --output-file build/coverage_filtered.info
genhtml build/coverage_filtered.info --output-directory build/coverage_html
```

### Coverage Thresholds
```bash
# Check coverage meets minimum threshold
gcovr --root . --filter src/ --fail-under-line 80 --fail-under-branch 80
# Exit code non-zero if coverage < 80%
```

## Common Testing Mistakes to Avoid

### ❌ WRONG: Testing Implementation Details
```cpp
// Don't test private internals
EXPECT_EQ(obj.internal_buffer_.size(), 5);  // Accessing private state
```

### ✅ CORRECT: Test Observable Behavior
```cpp
// Test through the public interface
obj.addItem("test");
EXPECT_EQ(obj.count(), 1);
EXPECT_EQ(obj.getItem(0), "test");
```

### ❌ WRONG: No Test Isolation
```cpp
// Tests depend on each other
static int shared_counter = 0;

TEST(CounterTest, Increment) {
    shared_counter++;
    EXPECT_EQ(shared_counter, 1);
}

TEST(CounterTest, IncrementAgain) {
    // Depends on previous test's state!
    shared_counter++;
    EXPECT_EQ(shared_counter, 2);
}
```

### ✅ CORRECT: Independent Tests
```cpp
// Each test sets up its own state
TEST(CounterTest, IncrementFromZero) {
    Counter counter;
    counter.increment();
    EXPECT_EQ(counter.value(), 1);
}

TEST(CounterTest, IncrementFromNonZero) {
    Counter counter(10);
    counter.increment();
    EXPECT_EQ(counter.value(), 11);
}
```

### ❌ WRONG: Ignoring RAII / Resource Leaks
```cpp
TEST(FileTest, ReadsContent) {
    FILE* f = fopen("test.txt", "r");
    // If assertion fails, file handle leaks!
    ASSERT_NE(f, nullptr);
    // ... test code ...
    fclose(f);
}
```

### ✅ CORRECT: Use RAII / Fixtures for Cleanup
```cpp
TEST(FileTest, ReadsContent) {
    auto file = std::make_unique<std::ifstream>("test.txt");
    ASSERT_TRUE(file->is_open());
    // unique_ptr ensures cleanup even if test fails
}

// Or use Test Fixtures with TearDown()
class FileTest : public ::testing::Test {
protected:
    std::ifstream file_;
    void SetUp() override { file_.open("test.txt"); }
    void TearDown() override { if (file_.is_open()) file_.close(); }
};
```

## Continuous Testing

### Watch Mode During Development
```bash
# Use entr or similar tool to re-run tests on file changes
find src tests -name '*.cpp' -o -name '*.h' | entr -s 'cmake --build build && ctest --test-dir build --output-on-failure'

# Or use CMake preset with ninja for fast incremental builds
cmake --build build --target unit_tests && ctest --test-dir build -R unit --output-on-failure
```

### Pre-Commit Hook
```bash
#!/bin/bash
# .git/hooks/pre-commit
cmake --build build --target unit_tests
ctest --test-dir build -R unit --output-on-failure
if [ $? -ne 0 ]; then
    echo "Unit tests failed. Commit aborted."
    exit 1
fi
```

### CI/CD Integration
```yaml
# GitHub Actions
- name: Configure CMake
  run: cmake -B build -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="--coverage"

- name: Build
  run: cmake --build build

- name: Run Tests
  run: ctest --test-dir build --output-on-failure

- name: Generate Coverage
  run: |
    gcovr --root . --filter src/ --xml -o build/coverage.xml
    gcovr --root . --filter src/ --fail-under-line 80

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    files: build/coverage.xml
```

## GTest Assertion Quick Reference

| Assertion | Description |
|---|---|
| `EXPECT_EQ(a, b)` | `a == b` (non-fatal) |
| `ASSERT_EQ(a, b)` | `a == b` (fatal, stops test) |
| `EXPECT_NE(a, b)` | `a != b` |
| `EXPECT_TRUE(cond)` | condition is true |
| `EXPECT_FALSE(cond)` | condition is false |
| `EXPECT_LT(a, b)` | `a < b` |
| `EXPECT_LE(a, b)` | `a <= b` |
| `EXPECT_GT(a, b)` | `a > b` |
| `EXPECT_GE(a, b)` | `a >= b` |
| `EXPECT_STREQ(s1, s2)` | C strings equal |
| `EXPECT_THROW(stmt, type)` | statement throws exception of type |
| `EXPECT_NO_THROW(stmt)` | statement doesn't throw |
| `EXPECT_NEAR(a, b, tol)` | `|a - b| <= tol` (floating point) |
| `EXPECT_THAT(val, matcher)` | gmock matcher (e.g. `HasSubstr`, `Contains`) |

## Best Practices

1. **Write Tests First** - Always TDD
2. **One Assert Per Test** - Focus on single behavior; use `EXPECT_*` for supplemental checks and `ASSERT_*` only when continuation is meaningless
3. **Descriptive Test Names** - Use `TEST(SuiteName, DescriptiveAction)` pattern
4. **Arrange-Act-Assert** - Clear test structure
5. **Mock External Dependencies** - Use gmock to isolate unit tests from hardware, network, file system
6. **Test Edge Cases** - nullptr, empty containers, max values, overflow, underflow
7. **Test Error Paths** - Not just happy paths; test exception safety, error codes, errno
8. **Keep Tests Fast** - Unit tests < 50ms each; use mocks to avoid I/O
9. **Clean Up After Tests** - Use RAII, `TearDown()`, and `::testing::Environment` for global setup/teardown
10. **Review Coverage Reports** - Identify gaps and dead code

## Success Metrics

- 80%+ code coverage achieved (line and branch)
- All tests passing (green)
- No skipped or disabled tests (`DISABLED_` prefix)
- Fast test execution (<30s for unit tests)
- Integration tests cover critical module interactions
- Tests catch bugs before production / deployment

---

**Remember**: Tests are not optional. They are the safety net that enables confident refactoring, rapid development, and production reliability. In C++, they are especially critical for catching memory errors, undefined behavior, and resource leaks early.
