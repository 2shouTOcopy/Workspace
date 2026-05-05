# AGENTS.md

## Scope and Precedence

- This file applies to `/product/app_src/app` and all directories below it.
- Global agent rules still apply. When global rules conflict with this file, follow this project-local file.
- Keep project-local guidance focused on this embedded app tree. Do not duplicate generic Codex behavior unless it changes how work must be done here.
- Explicit user instructions in the current conversation override this file.

## Project Overview

- Project type: embedded Linux application/common library tree.
- Main languages: C and C++.
- Runtime target: ARM/AArch64 Linux product platforms selected by `PLAT` and `PROD`.
- Build style: Makefile/Rules.mk plus product packaging scripts and cross-compilation variables such as `TOOL_PREFIX`.
- Output model: built headers/libraries are produced under `./output` and are later copied into product packaging or test projects.
- The project is organized by layered modules. Preserve these boundaries when adding or changing code.

## Repository Structure

- `source/`: main app source tree and the top-level app `Makefile`.
- `business_layer/`: business modules such as adapter, scheduler, parameter, upgrade, manager, cfg_capacity, cfg_bus, and reg_bus.
- `service_layer/`: service modules such as flash, aes, log/log_record, network, infra, media, and socket.
- `present_layer/`: presentation/transport layer modules such as `net_trans`.
- `modules/`: reusable local modules and platform-adjacent components.
- `open_library_layer/`: third-party/open-source libraries vendored into the tree.
- `configs/`: common and platform-specific make configuration, including `config.mk` and `config_<PLAT>.mk`.
- `mk_package/`, `package/`, `PackageCfg`, `prepare_pack.sh`, `build.sh`, `zoo`, `zoo.rule`: packaging and product assembly assets.
- `tests/`, `source/fwk/manager/tests/`, and module-local `unit_test` or `test` directories: unit and component test code.
- `output/`, `libs/`, generated package files, and test/build output directories are build artifacts or binary dependency locations. Do not edit them unless the task explicitly requires it.

## Build and Packaging

- Prefer the existing Makefile/Rules.mk flow. Do not introduce a new build system unless explicitly requested.
- Inspect the relevant `Makefile`, `Rules.mk`, `configs/config.mk`, and `configs/config_<PLAT>.mk` before changing build behavior.
- `build.sh` expects platform/product arguments and sources `mk_package/$2/PackageDefs`.
- `build.sh` has side effects: it runs `./zoo clean`, copies platform libraries/resources from `../../plat_libs` and `../../plat_resources`, updates files under `service_layer/media`, writes into `package/`, and moves generated `.ini`/`.zip` files to `../../pack/`.
- Do not run `build.sh` unless the requested platform/product/device are known and the user expects packaging side effects.
- For small code changes, prefer targeted module builds or tests when available instead of full packaging.
- Keep cross-compilation variables (`TOOL_PREFIX`, `PLAT`, `PROD`, `BUILD_TYPE`, `OFFLINE`, `adapter`, `log`, `AloneCompile`) compatible with existing make files.
- Do not hardcode local absolute paths into Makefiles or source files.

## Code Organization Rules

- Keep changes within the owning layer/module whenever possible.
- Do not create new cross-layer dependencies casually. Use existing public headers and module APIs.
- Public interfaces belong in module `include/` directories. Implementation details belong in `src/` or equivalent implementation directories.
- Avoid changing `open_library_layer/` and vendored third-party code unless the task is explicitly about that dependency.
- Avoid changing binary libraries, generated outputs, package artifacts, or platform resource copies.
- When adding a new module, follow the nearby module layout: `include/`, `src/`, `Makefile`, and `Rules.mk` if that layer uses it.
- Keep file and function size reasonable, but do not perform broad splitting/refactoring unless it is required for the requested change.

## Coding Style and Naming

- Follow existing local style first, then `.clang-format`.
- `.clang-format` defines C++11, 4-space indentation, no tabs, Allman braces, 120-column limit, preserved include blocks, and no automatic include sorting.
- Only format the files or newly added code segments involved in the task. Do not run whole-tree formatting.
- Preserve C compatibility in `.c` and C-facing headers.
- Match naming in the module being edited. Common patterns in this tree include PascalCase class/interface names, `I*` interface names, `Support*` helpers, and module-specific prefixes such as `Capa*`, `Param*`, `DspV2*`, `Upg*`.
- For new C++ code, prefer clear ownership and explicit initialization. Use `nullptr` in C++ code and `NULL` only where existing C code style requires it.
- Avoid exceptions across module boundaries unless the existing module already uses them. Prefer return codes and explicit error handling.
- Avoid broad use of templates, RTTI, or heavy abstractions in embedded paths unless already established nearby.

## Include and Dependency Rules

- Include only what is needed. Use forward declarations in headers when a pointer/reference is enough.
- Do not reorder include blocks only for style. `SortIncludes: false` is intentional.
- Keep public headers minimal and stable; avoid leaking implementation-only third-party headers through public APIs.
- Prefer module public include paths over relative traversal into another module's private source directory.
- When adding a library dependency, update the owning `Makefile`/`Rules.mk` and platform config consistently.
- Be careful with platform-specific include paths such as `source/misc/inc/dsp_common/$(PLAT)` and algorithm/plugin paths under `source/algos/plugins`.

## Embedded Constraints

- Treat memory, CPU, flash writes, file descriptors, sockets, handles, and threads as constrained resources.
- Avoid unbounded dynamic allocation, unbounded queues, busy loops, and frequent high-volume logging in runtime paths.
- Prefer preallocation, bounded buffers, pooling, or reuse where nearby code already uses those patterns.
- Keep `-Werror` compatibility in mind; avoid warnings in all supported platform variants.
- Preserve platform conditionals such as `PLAT`, `PROD`, `R$(PLAT)`, `_LINUX64`, `SC1000`, `SC2000E`, `SC3000`, and related macros.

## Logging and Error Handling

- Use the existing project logging macros/APIs in the module being edited. Do not introduce a new logging framework.
- Include actionable context in error logs: module/function, key parameters, path/device/platform, return code, and system error where relevant.
- Respect the logging build modes in `source/Makefile`: `SYS_LOG`, `LOG_NDEBUG`, `DEPEND_LOG`, and `LOG_COLOR`.
- Do not add noisy INFO/debug logs in high-frequency paths unless guarded by existing debug controls.
- Return and propagate errors consistently with surrounding code. Do not silently swallow failures from file, socket, thread, memory, config, or platform API calls.
- Preserve the log module ID ranges documented in `README.md`: `service_layer` 299-200, `business_layer` 199-180, `present_layer` 179-150, `other` 149-0, with special IDs 298 and 190.

## Memory, Resource, and Concurrency

- Pair every acquire with a release on all paths: memory, files, sockets, dynamic libraries, mutexes, semaphores, timers, and platform handles.
- In C++ code, prefer RAII wrappers when compatible with the local module style.
- In C code, use single-exit cleanup patterns when they make ownership easier to verify.
- Document ownership for pointers passed across module boundaries when it is not obvious.
- Avoid detached or unmanaged threads. Thread lifetime must have a clear stop/join or module shutdown path.
- Protect shared mutable state with existing mutex/atomic/thread-ownership patterns.
- Keep lock scope small and avoid calling slow I/O, callbacks, logging-heavy code, or cross-module operations while holding locks unless the existing design requires it.
- Preserve lock ordering when touching code that takes multiple locks.

## Configuration Compatibility

- Configuration changes must remain compatible across supported platform files in `configs/config_*.mk`.
- Do not remove or rename existing macros, config keys, package file names, or product/device identifiers without checking all call sites and packaging references.
- Treat `mk_package/*/PackageDefs`, `PackageCfg`, product directories under `package/products`, and media config files as compatibility-sensitive.
- When adding config, define defaults and failure behavior explicitly.
- Keep offline/debug/release differences intact: `BUILD_TYPE=debug`, `BUILD_TYPE=offline`, `OFFLINE=offline`, and `NDEBUG`/`_PACK_DEBUG` behavior must not be changed accidentally.

## Testing and Verification

- Prefer the narrowest meaningful verification for the files changed.
- Use existing tests when available:
  - `tests/storage/Makefile` and `tests/storage/storage_unit_tests.cpp`
  - `source/fwk/manager/tests/CMakeLists.txt`
  - module-local tests such as `business_layer/parameter/src/unit_test/` and `source/algos/modules/ftptrans/test/`
- If cross-compilation dependencies are unavailable locally, still run syntax/build checks that do not require missing platform SDKs when possible.
- Report exactly what was run and what could not be run.
- Do not commit or keep generated test binaries, CMake build trees, package zips, copied platform libraries, or temporary output as part of source edits.

## Documentation Expectations

- Comments should explain non-obvious intent, hardware/platform assumptions, protocol compatibility, ownership, locking, or error recovery.
- Do not add comments that merely restate the code.
- Update nearby README/docs when behavior, build flags, config, public APIs, or packaging expectations change.
- Progress or session notes, when needed, should go under `doc/` to match the existing project layout.

## Agent Behavior

- Before editing, inspect the owning module and follow its existing patterns.
- Keep changes narrowly scoped to the requested task.
- Avoid unrelated refactors, file moves, formatting churn, and generated artifact changes.
- Do not modify third-party/open-library code, platform binaries, package artifacts, or generated output unless the user explicitly asks.
- Ask before running commands with packaging side effects or commands that copy/move files outside this `app` tree.
- If a command depends on unavailable cross-toolchains, platform SDKs, or private package resources, stop and report the missing prerequisite instead of inventing replacements.
- When a task touches build scripts, packaging, platform configs, or public headers, verify references with `rg` before editing.
- When uncertain about platform/product scope, preserve all existing variants rather than optimizing for only one.
