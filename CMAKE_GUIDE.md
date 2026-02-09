# TheRock CMake Build System - A Beginner's Guide

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Where It All Starts: Top-Level CMakeLists.txt](#1-where-it-all-starts-top-level-cmakeliststxt)
- [How a Subproject Is Declared](#2-how-a-subproject-is-declared)
- [The 4-Phase Build Per Component](#3-the-4-phase-build-per-component)
- [How Dependencies Actually Work](#4-how-dependencies-actually-work)
- [The Compiler Toolchain System (and Bootstrapping)](#5-the-compiler-toolchain-system-and-bootstrapping)
- [CMake Presets](#6-cmake-presets)
- [The Full Picture: What Happens When You Build](#7-the-full-picture-what-happens-when-you-build)
- [Key Files Reference](#8-key-files-reference)
- [Quick Cheat Sheet for Daily Work](#9-quick-cheat-sheet-for-daily-work)

---

## The Big Picture

TheRock is **not** a normal CMake project. It's a **super-project** — a CMake
project whose job is to **orchestrate building many other CMake projects** (ROCm
components like HIP, rocBLAS, etc.) in the right order with the right settings.
Think of it like a conductor managing an orchestra.

```
TheRock (super-project)
  ├── configures & builds → amd-llvm (LLVM compiler)
  ├── configures & builds → hip-clr (HIP runtime)  [depends on amd-llvm]
  ├── configures & builds → rocBLAS (math library)  [depends on hip-clr]
  └── ... 30+ more projects
```

Each subproject is a **real, standalone CMake project** (with its own
`CMakeLists.txt`). TheRock doesn't use `add_subdirectory()` on them — instead it
runs a **separate CMake configure + build** for each one, wiring up dependencies
between them via generated toolchain files and init scripts.

---

## 1. Where It All Starts: Top-Level CMakeLists.txt

When you run:

```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1100
```

CMake reads the top-level `CMakeLists.txt`. Here's what it does in order:

### Step 1 — Load the build system infrastructure

```cmake
include(cmake/therock_globals.cmake)          # Platform detection (Windows/Linux)
include(cmake/therock_features.cmake)         # Feature flag system
include(cmake/therock_amdgpu_targets.cmake)   # GPU target definitions
include(cmake/therock_subproject.cmake)        # The core macro for subprojects
include(cmake/therock_artifacts.cmake)         # Packaging system
# ... and a few more
```

**What does "build infrastructure" actually mean?**

In CMake, `include()` loads a `.cmake` file and runs it — similar to `import` in
Python or `#include` in C. These files don't build anything themselves. They
**define custom functions and macros** that the rest of the project uses.

Think of it like loading a toolbox before you start work. Each file adds
different tools:

| File loaded | What it defines (the "tools" it adds) |
| --- | --- |
| `therock_globals.cmake` | Simple boolean variables like `THEROCK_CONDITION_IS_WINDOWS` so the rest of the code can do `if(THEROCK_CONDITION_IS_WINDOWS)` instead of checking platform details every time |
| `therock_features.cmake` | The `therock_add_feature()` macro — a way to register optional components (like "math libs") that can be turned on/off with `-D` flags, including automatic dependency resolution (enabling rocBLAS auto-enables HIP) |
| `therock_amdgpu_targets.cmake` | The `therock_add_amdgpu_target()` and `therock_validate_amdgpu_targets()` macros — registers every known AMD GPU architecture and provides family-based shortcuts to select groups of them |
| `therock_subproject.cmake` | The `therock_cmake_subproject_declare()` and `therock_cmake_subproject_activate()` macros — **the most important file**. This is the engine that knows how to take a subproject declaration and generate all the Ninja targets, toolchain files, init files, and dependency wiring for it |
| `therock_artifacts.cmake` | The `therock_provide_artifact()` macro — handles packaging built components into distributable archives |
| `therock_compiler_config.cmake` | Validates that your system compiler (the one already on your machine) is suitable. On Windows it checks for MSVC (cl.exe); on Linux it may require GCC for certain components |
| `therock_job_pools.cmake` | Controls parallelism — how many things Ninja is allowed to compile at once (important because LLVM compilation is extremely memory-hungry) |

**None of these files compile code.** They just define the vocabulary
(functions/macros) that the rest of the `CMakeLists.txt` files use to describe
what to build and how. The actual compilation happens later when Ninja runs.

> **CMake beginner note:** A CMake "macro" or "function" is just like a function
> in any programming language. `therock_cmake_subproject_declare(rocBLAS ...)`
> is calling a function named `therock_cmake_subproject_declare` with `rocBLAS`
> as its first argument. These functions were defined in the `include()`'d
> files above.

### Step 2 — Figure out what GPU to build for

Your `-DTHEROCK_AMDGPU_FAMILIES=gfx1100` gets validated and expanded. Families
are convenience groups:

| What you pass | What it expands to |
| --- | --- |
| `gfx1100` | Just gfx1100 (a single GPU) |
| `gfx110X-all` | gfx1100 + gfx1101 + gfx1102 (a whole family) |
| `dgpu-all` | Every discrete GPU target |

### Step 3 — Figure out what components to build

Feature flags control what gets built:

```
THEROCK_ENABLE_ALL=ON (default — build everything)
  ├── THEROCK_ENABLE_COMPILER   → LLVM/Clang
  ├── THEROCK_ENABLE_CORE       → HIP runtime
  ├── THEROCK_ENABLE_MATH_LIBS  → rocBLAS, rocFFT, etc.
  ├── THEROCK_ENABLE_ML_LIBS    → MIOpen
  ├── THEROCK_ENABLE_COMM_LIBS  → RCCL
  ├── THEROCK_ENABLE_PROFILER   → rocprofiler, roctracer
  ├── THEROCK_ENABLE_DEBUG_TOOLS→ rocgdb
  └── THEROCK_ENABLE_DC_TOOLS   → Data center tools
```

You can do a minimal build:

```bash
cmake -B build -GNinja \
  -DTHEROCK_ENABLE_ALL=OFF \
  -DTHEROCK_ENABLE_CORE=ON \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100
```

### Step 4 — Walk into each subdirectory

```cmake
add_subdirectory(third-party)   # zlib, boost, etc.
add_subdirectory(base)          # rocm-core, amdsmi
add_subdirectory(compiler)      # LLVM/Clang
add_subdirectory(core)          # HIP, CLR
add_subdirectory(math-libs)     # rocBLAS, rocFFT...
# ... etc
```

Each subdirectory has its own `CMakeLists.txt` that **declares** its
subprojects. These `add_subdirectory()` calls do NOT build the subprojects —
they just register them with the super-project.

---

## 2. How a Subproject Is Declared

This is the heart of the system. Inside, say, `math-libs/CMakeLists.txt`,
you'll see something like:

```cmake
therock_cmake_subproject_declare(rocBLAS
  EXTERNAL_SOURCE_DIR "path/to/rocblas/source"
  COMPILER_TOOLCHAIN amd-hip        # Use the HIP compiler we built
  BUILD_DEPS rocm-cmake             # Needed at compile time
  RUNTIME_DEPS hip-clr amd-llvm     # Needed at runtime
  CMAKE_ARGS
    -DBUILD_TESTING=OFF
    -DHIP_PLATFORM=amd
)
therock_cmake_subproject_activate(rocBLAS)
```

### What each part means

- **`EXTERNAL_SOURCE_DIR`** — Path to the subproject's source code (a git
  submodule)
- **`COMPILER_TOOLCHAIN`** — Which compiler to use (`amd-hip`, `amd-llvm`, or
  system default)
- **`BUILD_DEPS`** — Other subprojects needed at compile time (headers, cmake
  modules)
- **`RUNTIME_DEPS`** — Other subprojects needed at runtime (shared libraries);
  these get merged into the `dist/` directory
- **`CMAKE_ARGS`** — Extra flags passed to the subproject's own CMake configure

### Two-step process

- **`declare`** = "Here's what this project is, where its source lives, what it
  depends on, and what settings it needs"
- **`activate`** = "Now generate all the Ninja build targets for it"

### Generated targets

After activation, Ninja gets these targets:

| Target | What it does |
| --- | --- |
| `ninja rocBLAS` | Full build (all phases below) |
| `ninja rocBLAS+configure` | Run CMake configure on rocBLAS's source |
| `ninja rocBLAS+build` | Compile rocBLAS (skip configure if up-to-date) |
| `ninja rocBLAS+stage` | Install into `rocBLAS/stage/` |
| `ninja rocBLAS+dist` | Merge stage + runtime deps into `rocBLAS/dist/` |
| `ninja rocBLAS+expunge` | Delete everything and start fresh |

---

## 3. The 4-Phase Build Per Component

Each component goes through four phases, producing this directory layout:

```
build/rocBLAS/
├── build/    ← CMake's own build tree (object files, Makefiles, etc.)
├── stage/    ← "make install" output (just rocBLAS's own files)
├── dist/     ← stage/ PLUS all runtime dependency files merged in
├── stamp/    ← Timestamps for incremental builds
│   ├── configure.stamp
│   ├── build.stamp
│   └── stage.stamp
└── prefix/   ← CMAKE_PREFIX_PATH for find_package()
```

### The four phases

1. **Configure** — Runs `cmake` on the subproject's source with generated
   toolchain and init files
2. **Build** — Runs `cmake --build` to compile everything
3. **Stage** — Runs `cmake --install` to install into `stage/` (just this
   component's files)
4. **Dist** — Uses `fileset_tool.py` to merge `stage/` plus all runtime
   dependency `dist/` directories into `dist/`

### Why separate stage/ and dist/?

- **`stage/`** has ONLY rocBLAS's files
- **`dist/`** has rocBLAS's files **plus** everything it needs at runtime (HIP
  libs, LLVM, etc.)
- This means `dist/` is a **self-contained** directory you can point to and use
  directly

### The final output

`build/dist/rocm/` merges ALL components' `dist/` directories into one unified
ROCm installation. This is what you'd use as your ROCm install.

---

## 4. How Dependencies Actually Work

When rocBLAS declares `RUNTIME_DEPS hip-clr`, here's what happens behind the
scenes:

### At configure time

1. The super-project generates a **"project init" file**
   (`rocBLAS_init.cmake`) that adds hip-clr's `include/` and `lib/` directories
   to rocBLAS's search paths

2. The super-project generates a **"dependency provider"** that intercepts
   `find_package(hip)` calls inside rocBLAS and redirects them to hip-clr's
   `stage/` directory — instead of looking system-wide

3. This init file is injected via `CMAKE_PROJECT_TOP_LEVEL_INCLUDES`, which
   is a CMake mechanism that runs a script at the start of every
   subproject's configure

### At dist time

hip-clr's `dist/` directory gets merged into rocBLAS's `dist/` directory, so
rocBLAS's dist has everything it needs to run.

### Dependency types

| Type | Purpose | Example |
| --- | --- | --- |
| `BUILD_DEPS` | Needed to compile | `rocm-cmake` (provides cmake modules) |
| `RUNTIME_DEPS` | Needed to run; merged into dist/ | `hip-clr` (provides shared libraries) |

### How find_package() is redirected

The dependency provider (`therock_subproject_dep_provider.cmake`) works like
this:

```
rocBLAS's CMakeLists.txt calls:  find_package(hip)
                                      │
                                      ▼
Dependency provider intercepts:  "Is 'hip' a known TheRock package?"
                                      │
                              ┌───────┴───────┐
                              │ YES           │ NO
                              ▼               ▼
                  Look in hip-clr's      Fall back to
                  stage/lib/cmake/       system search
                  for hip config
```

This keeps subprojects **isolated** — no component reaches out to the system for
ROCm packages. Everything comes from sister components built in the same
super-project.

---

## 5. The Compiler Toolchain System (and Bootstrapping)

### The chicken-and-egg problem

TheRock needs a compiler to build things. But one of the things it builds **is**
a compiler (LLVM/Clang). So how does that work?

The answer is **bootstrapping** — a technique where you use a simple tool to
build a better tool:

```
Your machine already has a compiler installed:
  - Windows: MSVC (cl.exe), installed with Visual Studio
  - Linux: GCC (g++) or system Clang

TheRock uses THAT existing compiler to build its own, better compiler (LLVM/Clang).
Then it uses the newly-built compiler to build everything else.
```

### The three-stage bootstrap

```
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 1: System compiler builds LLVM                            │
│                                                                 │
│ Your system's MSVC or GCC ──compiles──► amd-llvm subproject     │
│                                                                 │
│ Input:  LLVM/Clang source code (the compiler/ submodule)        │
│ Output: build/amd-llvm/dist/lib/llvm/bin/clang                  │
│         build/amd-llvm/dist/lib/llvm/bin/clang++                │
│         build/amd-llvm/dist/lib/llvm/bin/lld  (linker)          │
│                                                                 │
│ This is just a normal C++ project being compiled. LLVM happens  │
│ to be a compiler, but it's still "just C++ code" that your      │
│ system compiler can build like any other program.               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 2: Built LLVM compiles tools and runtimes                 │
│                                                                 │
│ amd-llvm's clang/clang++ ──compiles──► ROCR-Runtime, hipcc,     │
│                                        amd-comgr, hipify, etc.  │
│                                                                 │
│ These are C/C++ projects that don't need GPU compilation yet.   │
│ They use COMPILER_TOOLCHAIN=amd-llvm, which tells the build     │
│ system: "use the clang we just built, not the system compiler." │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 3: LLVM + HIP runtime compiles GPU libraries              │
│                                                                 │
│ amd-llvm's clang + HIP headers/tools ──compiles──► rocBLAS,     │
│                                                     rocFFT,     │
│                                                     MIOpen, ... │
│                                                                 │
│ These projects contain GPU kernel code (.hip files) that needs  │
│ the HIP compiler. They use COMPILER_TOOLCHAIN=amd-hip, which    │
│ is clang + extra flags like --hip-path and --hip-device-lib-path│
│ so it knows how to compile code for your AMD GPU.               │
└─────────────────────────────────────────────────────────────────┘
```

### How does the build system know which compiler to use?

Every subproject declares a `COMPILER_TOOLCHAIN` parameter:

```cmake
# In compiler/CMakeLists.txt — amd-llvm has NO toolchain specified
therock_cmake_subproject_declare(amd-llvm
  EXTERNAL_SOURCE_DIR "amd-llvm"
  # No COMPILER_TOOLCHAIN line! → uses your system compiler (MSVC/GCC)
  BUILD_DEPS rocm-cmake
  ...
)

# In core/CMakeLists.txt — ROCR uses the built LLVM
therock_cmake_subproject_declare(ROCR-Runtime
  COMPILER_TOOLCHAIN amd-llvm       # ← uses the clang we just built
  RUNTIME_DEPS amd-llvm
  ...
)

# In math-libs/CMakeLists.txt — rocRAND uses LLVM + HIP
therock_cmake_subproject_declare(rocRAND
  COMPILER_TOOLCHAIN amd-hip        # ← uses clang + HIP extensions
  RUNTIME_DEPS hip-clr
  ...
)
```

### The three toolchain modes in detail

| Toolchain | Who uses it | Compiler binary used | When to use |
| --- | --- | --- | --- |
| *(none/system)* | amd-llvm, third-party libs (zlib, boost) | Your system's MSVC (`cl.exe`) or GCC (`g++`) | For building the compiler itself, and simple C/C++ dependencies that don't need anything special |
| `amd-llvm` | ROCR-Runtime, rocminfo, hipify | `build/amd-llvm/dist/lib/llvm/bin/clang++` | For C/C++ projects that need the custom LLVM but don't have GPU kernel code |
| `amd-hip` | rocBLAS, rocFFT, rocRAND, MIOpen, etc. | Same clang++, but with HIP flags added | For projects that contain `.hip` GPU kernel code and need `--hip-path` etc. |

### What a generated toolchain file looks like

When you specify `COMPILER_TOOLCHAIN amd-llvm`, the build system generates a
file like `build/ROCR-Runtime_toolchain.cmake` containing:

```cmake
# "Dear CMake, when you configure ROCR-Runtime, use THESE compilers
#  instead of whatever is on the system PATH."
set(CMAKE_C_COMPILER
  "/path/to/build/amd-llvm/dist/lib/llvm/bin/clang"
  CACHE STRING "Set by TheRock super-project" FORCE)
set(CMAKE_CXX_COMPILER
  "/path/to/build/amd-llvm/dist/lib/llvm/bin/clang++"
  CACHE STRING "Set by TheRock super-project" FORCE)
set(CMAKE_LINKER
  "/path/to/build/amd-llvm/dist/lib/llvm/bin/lld"
  CACHE STRING "Set by TheRock super-project" FORCE)
```

For `COMPILER_TOOLCHAIN amd-hip`, it adds extra GPU-specific settings:

```cmake
# Everything from amd-llvm above, PLUS:
set(GPU_TARGETS "gfx1100" CACHE STRING "From super-project" FORCE)
set(AMDGPU_TARGETS "gfx1100" CACHE STRING "From super-project" FORCE)
set(CMAKE_HIP_ARCHITECTURES "gfx1100" CACHE STRING "From super-project" FORCE)

# Tell clang where HIP headers and device libraries live
set(CMAKE_CXX_FLAGS_INIT "... --hip-path=/path/to/hip-clr/dist ...")
```

### How does Ninja know to build the compiler first?

When a subproject says `COMPILER_TOOLCHAIN amd-llvm`, the build system
automatically adds a dependency on `amd-llvm`'s stage stamp file. In Ninja
terms:

```
rocBLAS+configure  depends on  amd-llvm/stamp/stage.stamp
```

That stamp file only exists after amd-llvm is fully compiled and installed.
So Ninja will **never** try to configure rocBLAS until the compiler is ready.
This is all automatic — you don't need to think about build ordering.

### Windows note

On Windows, the bootstrapping is slightly different. The core runtime components
use the **system MSVC compiler** instead of the built LLVM, because some Windows
components need MSVC-specific features:

```cmake
# In core/CMakeLists.txt
if(WIN32)
  set(_system_toolchain "")         # Stay with MSVC on Windows
else()
  set(_system_toolchain "amd-llvm") # Use built LLVM on Linux
endif()
```

### How GPU targets flow through the system

```
You type:  -DTHEROCK_AMDGPU_FAMILIES=gfx1100
                    │
                    ▼
Top-level:  therock_validate_amdgpu_targets()
            Expands family → concrete targets [gfx1100]
            Stores as global property THEROCK_AMDGPU_TARGETS
                    │
                    ▼
Per subproject:  therock_cmake_subproject_activate()
                 Reads global THEROCK_AMDGPU_TARGETS
                 Writes into generated toolchain file:
                   set(GPU_TARGETS "gfx1100")
                   set(AMDGPU_TARGETS "gfx1100")
                   set(CMAKE_HIP_ARCHITECTURES "gfx1100")
                    │
                    ▼
Subproject:  rocBLAS/CMakeLists.txt reads GPU_TARGETS
             from its cache (set by the toolchain file)
             and compiles kernels for gfx1100
```

---

## 6. CMake Presets

`CMakePresets.json` gives you named configurations so you don't have to remember
flags:

```bash
# Instead of typing lots of flags:
cmake --preset linux-release-package -DTHEROCK_AMDGPU_FAMILIES=gfx1100

# Which is equivalent to something like:
cmake -B build -GNinja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DTHEROCK_SPLIT_DEBUG_INFO=ON \
  -DCMAKE_C_FLAGS_RELWITHDEBINFO="-O2 -g1 -DNDEBUG" \
  ...
```

### Available presets

| Preset | Purpose |
| --- | --- |
| `linux-release-package` | Release with minimal debug info (production builds) |
| `linux-release-asan` | Address sanitizer enabled (finding memory bugs) |
| `linux-release-host-asan` | Host-only ASAN (no device instrumentation) |
| `windows-base` | Base config for Windows (MSVC, x64) |
| `windows-release` | Windows Release build |

---

## 7. The Full Picture: What Happens When You Build

```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1100
ninja -C build
```

### Phase 1: CMake Configure (the `cmake` command)

```
Parse top-level CMakeLists.txt
  │
  ├─ Load cmake/*.cmake infrastructure files
  │
  ├─ Validate GPU targets (gfx1100 ✓)
  │
  ├─ Resolve feature flags (what to build)
  │
  ├─ Walk subdirectories: base/, compiler/, core/, math-libs/, ...
  │   └─ Each declares + activates its subprojects
  │       └─ Generates init files, toolchain files, Ninja targets
  │
  └─ Output: build/build.ninja with ALL targets for ALL subprojects
```

### Phase 2: Ninja Build (the `ninja` command)

```
Ninja reads build.ninja and builds in dependency order:

  ├─ Build amd-llvm (LLVM compiler) ─────────────────────────┐
  │   configure → build → stage → dist                       │
  │                                                          │
  ├─ Build hip-clr (HIP runtime) ◄── depends on amd-llvm ───┘
  │   configure → build → stage → dist         │
  │                                             │
  ├─ Build rocBLAS ◄── depends on hip-clr ──────┘
  │   configure → build → stage → dist
  │
  ├─ ... all other enabled components ...
  │
  └─ Merge everything → build/dist/rocm/  (final output)
```

### What "configure" does for each subproject

When Ninja hits `rocBLAS+configure`, it runs something like:

```bash
cmake \
  -B build/rocBLAS/build \
  -S external/rocblas \
  -GNinja \
  -DCMAKE_TOOLCHAIN_FILE=build/rocBLAS_toolchain.cmake \
  -DCMAKE_PROJECT_TOP_LEVEL_INCLUDES=build/rocBLAS_init.cmake \
  -DBUILD_TESTING=OFF \
  -DHIP_PLATFORM=amd \
  ...
```

The toolchain file sets the compiler and GPU targets. The init file sets up
dependency paths and the `find_package()` provider.

---

## 8. Key Files Reference

### Build system infrastructure (in `cmake/`)

| File | Purpose |
| --- | --- |
| `therock_subproject.cmake` | **Core macro** — how subprojects are declared, configured, and built |
| `therock_features.cmake` | Feature flag system (`THEROCK_ENABLE_X` options) |
| `therock_amdgpu_targets.cmake` | GPU architecture definitions and family expansion |
| `therock_artifacts.cmake` | Packaging/distribution system |
| `therock_compiler_config.cmake` | Toolchain generation (`amd-llvm`, `amd-hip`) |
| `therock_subproject_dep_provider.cmake` | Intercepts `find_package()` in subprojects |
| `therock_globals.cmake` | Platform detection (Windows vs Linux) |
| `therock_job_pools.cmake` | Controls build parallelism |
| `therock_testing.cmake` | Test infrastructure |
| `therock_sanitizers.cmake` | ASAN/sanitizer configuration |

### Top-level files

| File | Purpose |
| --- | --- |
| `CMakeLists.txt` | Entry point — includes everything, walks subdirs |
| `CMakePresets.json` | Named build configurations (presets) |
| `BUILD_TOPOLOGY.toml` | Defines artifacts, features, and their dependencies |
| `version.json` | Version metadata |

### Build tools (in `build_tools/`)

| File | Purpose |
| --- | --- |
| `fileset_tool.py` | Merges `dist/` directories |
| `topology_to_cmake.py` | Generates CMake code from `BUILD_TOPOLOGY.toml` |
| `teatime.py` | Output formatting/logging wrapper |
| `fetch_sources.py` | Initializes/resets git submodules |

### Project directories

| Directory | What it contains |
| --- | --- |
| `base/` | Foundation: rocm-core, amdsmi |
| `compiler/` | LLVM/Clang/LLD, device libraries |
| `core/` | HIP runtime, CLR, ROCR |
| `math-libs/` | rocBLAS, rocFFT, rocSPARSE, rocSOLVER, hipBLAS |
| `ml-libs/` | MIOpen, composable_kernel |
| `comm-libs/` | RCCL (collective communications) |
| `profiler/` | rocprofiler, roctracer |
| `debug-tools/` | rocgdb |
| `third-party/` | Bundled dependencies (boost, zlib, etc.) |

---

## 9. Quick Cheat Sheet for Daily Work

### Building

```bash
# Build everything
ninja -C build

# Build a specific component (all phases)
ninja -C build rocBLAS

# Rebuild after editing source (skip configure)
ninja -C build rocBLAS+build

# Update dist without full rebuild
ninja -C build rocBLAS+dist

# Clean rebuild of one component
ninja -C build rocBLAS+expunge && ninja -C build rocBLAS

# Clean rebuild of everything
ninja -C build expunge && ninja -C build
```

### Configuring

```bash
# Full build for one GPU
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1100

# Minimal build (just HIP, no math libs)
cmake -B build -GNinja \
  -DTHEROCK_ENABLE_ALL=OFF \
  -DTHEROCK_ENABLE_CORE=ON \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100

# Use a preset
cmake --preset linux-release-package -DTHEROCK_AMDGPU_FAMILIES=gfx1100

# Faster rebuilds with ccache
cmake -B build -GNinja \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100

# Debug build of one component
cmake -B build -GNinja \
  -DCMAKE_BUILD_TYPE=Release \
  -Drocblas_BUILD_TYPE=RelWithDebInfo \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100
```

### Inspecting

```bash
# See all available Ninja targets
ninja -C build -t targets | grep "component_name"

# Generate compile_commands.json for IDE support
cmake --build build --target therock_merged_compile_commands

# Run tests
ctest --test-dir build
```

---

## Key CMake Concepts for Newcomers

If you're new to CMake, here are the concepts this project uses heavily:

- **`cmake -B build`** — Configure into a `build/` directory (out-of-source
  build)
- **`-GNinja`** — Use the Ninja build system (faster than Make)
- **`-D<VAR>=<VALUE>`** — Set a CMake variable
- **`CMAKE_TOOLCHAIN_FILE`** — A file that tells CMake which compiler to use
- **`CMAKE_PREFIX_PATH`** — Where to look for `find_package()` results
- **`CMAKE_PROJECT_TOP_LEVEL_INCLUDES`** — A script injected at the start of
  every project's configure (used here for dependency injection)
- **`find_package()`** — CMake's way of finding installed libraries; TheRock
  intercepts this to redirect to its own built components
- **Properties** — CMake targets can have arbitrary key-value properties
  attached; TheRock uses this extensively to track metadata about subprojects
