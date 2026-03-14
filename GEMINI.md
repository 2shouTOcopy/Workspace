# Coding Style

## Immutability (CRITICAL)

ALWAYS create new objects, NEVER mutate existing ones:

```
// Pseudocode
WRONG:  modify(original, field, value) → changes original in-place
CORRECT: update(original, field, value) → returns new copy with change
```

Rationale: Immutable data prevents hidden side effects, makes debugging easier, and enables safe concurrency.

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, 800 max
- Extract utilities from large modules
- Organize by feature/domain, not by type

## Error Handling

ALWAYS handle errors comprehensively:
- Handle errors explicitly at every level
- Provide user-friendly error messages in UI-facing code
- Log detailed error context on the server side
- Never silently swallow errors

## Input Validation

ALWAYS validate at system boundaries:
- Validate all user input before processing
- Use schema-based validation where available
- Fail fast with clear error messages
- Never trust external data (API responses, user input, file content)

## Code Quality Checklist

Before marking work complete:
- [ ] Code is readable and well-named
- [ ] Functions are small (<50 lines)
- [ ] Files are focused (<800 lines)
- [ ] No deep nesting (>4 levels)
- [ ] Proper error handling
- [ ] No hardcoded values (use constants or config)
- [ ] No mutation (immutable patterns used)

---

## C++ Coding Conventions

### Naming Conventions

| Element                  | Convention                  | Example                                     |
| ------------------------ | --------------------------- | ------------------------------------------- |
| Class                    | PascalCase                  | `ModuleLoader`, `LibInfo`                   |
| Abstract class           | `C` prefix + PascalCase     | `CAbstractUserModule`                       |
| Public method            | PascalCase                  | `Load()`, `SetDir()`, `GetUserModuleImpl()` |
| Private method           | PascalCase (same as public) | `LoadPluginFile()`                          |
| Member variable          | `m_` prefix + camelCase     | `m_mutex`, `m_pluginDir`, `m_openLibs`      |
| Local variable           | camelCase                   | `fullPath`, `soName`, `pluginName`          |
| Function pointer typedef | `pfn` prefix + PascalCase   | `pfnCreateModule`, `pfnDestroyModule`       |
| Type alias (`using`)     | PascalCase                  | `using OptLibMap = std::map<...>`           |
| Constants / macros       | UPPER_SNAKE_CASE            | `DEFAULT_LOAD_SO_PATH`                      |
| Struct member (POD)      | camelCase (no prefix)       | `handle`, `fnCreateOpt`, `pUserModuleImpl`  |
| Pointer member in struct | `p` prefix                  | `pUserModuleImpl`                           |
| Iterator variable        | `it` / `iter`               | `auto it = m_openLibs.find(...)`            |

### Formatting

- **Indentation**: Tabs (not spaces)
- **Brace style**: Allman style — opening brace on a new line
  ```cpp
  if (condition)
  {
      // body
  }
  ```
- **Namespace**: No wrapping namespace (flat, single-class-per-file style)
- **Line width**: No strict limit, but keep lines readable
- **Parameter alignment**: Align continuation lines with the opening parenthesis
  ```cpp
  int LoadPluginFile(const std::string &fullPath, bool global,
                     const std::string &pluginName);
  ```

### Header Files

- Use `#pragma once` as include guard (not `#ifndef`/`#define`)
- Include order:
  1. Standard library headers (`<string>`, `<vector>`, `<mutex>`)
  2. Project headers (`"CommonDef.h"`, `"ModuleBase.h"`)
- Use **forward declarations** instead of `#include` when only pointers/references are needed:
  ```cpp
  class CAbstractUserModule;  // forward declaration in header
  #include "ModuleBase.h"     // full include only in .cpp
  ```

### Class Design

- **Singleton**: Use Meyers' Singleton pattern (function-local static)
  ```cpp
  static ModuleLoader& Instance()
  {
      static ModuleLoader s_instance;
      return s_instance;
  }
  ```
- **Non-copyable**: Explicitly delete copy constructor and assignment operator
  ```cpp
  ModuleLoader(const ModuleLoader&) = delete;
  ModuleLoader& operator=(const ModuleLoader&) = delete;
  ```
- **Access control sections**: Order as `public → private`, group by purpose with separate `private:` for data members
- **Default constructors/destructors**: Use `= default` when no special logic needed
- **Inner structs**: Declare POD-like structs as `private` nested types within the owning class
- **Type aliases**: Use `using` (not `typedef`) for container types inside class scope

### String & Parameter Passing

- Pass strings by `const std::string&` (not by value, not by `const char*`)
  ```cpp
  void SetDir(const std::string &dir);
  int Load(const std::string &pluginName, bool global);
  ```
- Place `&` / `*` next to the type (not the variable name): `const std::string &dir`
- Return `std::vector<std::string>` by value (RVO / move semantics)

### Thread Safety

- Use `std::lock_guard<std::mutex>` for all access to shared data
- Scope lock guards in blocks `{}` to minimize lock duration:
  ```cpp
  {
      std::lock_guard<std::mutex> lock(m_mutex);
      if (m_openLibs.find(name) != m_openLibs.end())
          return 0;
  }
  // unlocked section — do expensive work here
  ```
- Declare `std::mutex` as a `private` member variable (`m_mutex`)

### Error Handling (C++ Specific)

- Return `int` error codes: `0` = success, negative values = failure (`-1`, `-2`, `-3`)
- Check return values of C APIs immediately (`dlopen`, `dlsym`, `stat`, `dlclose`)
- Clean up resources before returning on error (e.g., `dlclose(handle)` on symbol lookup failure)
- Use `nullptr` (not `NULL`) for C++ pointer checks
- Guard against double-load / double-unload by checking container state first

### Logging

- Use `std::cerr` for diagnostic/error output
- Prefix log messages with `[ClassName]` tag:
  ```cpp
  std::cerr << "[ModuleLoader] " << fullPath << " not found\n";
  ```
- Include context in error messages (file path, plugin name, system error via `dlerror()`)
- Use `\n` (not `std::endl`) for log lines unless explicit flush is needed

### Resource Management

- Use `dlopen` / `dlclose` for dynamic library loading
- Store handles in structured containers (`std::map<std::string, LibInfo>`)
- Call destroy/clean-up functions before releasing handles
- Erase from all tracking containers on unload (both `m_openLibs` and `m_loadPluginNames`)

---

# Git Workflow

## Commit Message Format

```
<type>: <description>

<optional body>
```

Types: feat, fix, refactor, docs, test, chore, perf, ci

## Pull Request Workflow

When creating PRs:
1. Analyze full commit history (not just latest commit)
2. Use `git diff [base-branch]...HEAD` to see all changes
3. Draft comprehensive PR summary
4. Include test plan with TODOs
5. Push with `-u` flag if new branch

## Feature Implementation Workflow

1. **Plan First**
   - Create implementation plan before coding
   - Identify dependencies and risks
   - Break down into phases

2. **TDD Approach**
   - Write tests first (RED)
   - Implement to pass tests (GREEN)
   - Refactor (IMPROVE)
   - Verify 80%+ coverage

3. **Code Review**
   - Review code immediately after writing
   - Address CRITICAL and HIGH issues
   - Fix MEDIUM issues when possible

4. **Commit & Push**
   - Detailed commit messages
   - Follow conventional commits format

---

# Progress Tracking & Context Management

## Progress Documentation (CRITICAL)

1. **每完成一小阶段任务后**，必须立即更新进展文档（`doc/progress.md` 或同类文件），总结完成内容、变更清单、下一步计划。
2. **进展文档统一放在项目的 `doc/` 目录下**，不要放在其他位置。
3. 进展文档应包含：已完成任务、新增/修改文件清单、关键设计决策、下一步待办。

## Context Limit Warning (CRITICAL)

当会话上下文即将超出限制时，**必须主动提醒用户**，并同时：
1. 在 `doc/` 目录下创建或更新一份**会话总结文档**（如 `doc/session_summary.md`），包含：
   - 当前整体进度（哪些 Phase 已完成、哪些正在进行）
   - 未完成的具体任务列表
   - 关键设计决策和上下文信息
   - 下一个会话应该从哪里开始
2. 确保新会话可以仅通过阅读 `doc/` 下的文档快速恢复上下文并继续工作。

