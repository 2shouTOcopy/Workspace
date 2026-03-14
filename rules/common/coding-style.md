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
- Clean up resources before returning on error (e.g., `dlclose(handle)` on symbol lookup failure)
- Use `nullptr` (not `NULL`) for C++ pointer checks
- Guard against double-load / double-unload by checking container state first


