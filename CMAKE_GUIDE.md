# TheRock CMake Build System - A Beginner's Guide

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Where It All Starts: Top-Level CMakeLists.txt](#1-where-it-all-starts-top-level-cmakeliststxt)
- [How a Subproject Is Declared](#2-how-a-subproject-is-declared)
- [The 4-Phase Build Per Component](#3-the-4-phase-build-per-component)
- [How Dependencies Actually Work](#4-how-dependencies-actually-work)
- [The Compiler Toolchain System](#5-the-compiler-toolchain-system)
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

## 5. The Compiler Toolchain System

TheRock builds its own compiler (LLVM/Clang) and then uses it to build
everything else. Three toolchain modes:

| Toolchain | Used for | What it does |
| --- | --- | --- |
| *(system)* | Third-party libs (zlib, etc.) | Uses your system's gcc/clang |
| `amd-llvm` | Non-HIP C/C++ projects | Uses the LLVM that TheRock just built |
| `amd-hip` | GPU projects (rocBLAS, etc.) | Uses LLVM + HIP compiler (hipcc) |

### What happens when you specify `COMPILER_TOOLCHAIN amd-hip`

1. The system generates a **toolchain file** (`rocBLAS_toolchain.cmake`) that:
   - Points `CMAKE_C_COMPILER` and `CMAKE_CXX_COMPILER` to the built
     clang/hipcc
   - Sets `GPU_TARGETS=gfx1100` (from your `-DTHEROCK_AMDGPU_FAMILIES`)
   - Sets `CMAKE_HIP_ARCHITECTURES=gfx1100`
   - Adds HIP-specific flags and paths (`--hip-path`, `--hip-device-lib-path`)
2. Automatically adds a **build dependency** on the compiler being built first

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
