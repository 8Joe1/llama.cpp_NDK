# llama.cpp Android Builder

A build system for compiling [llama.cpp](https://github.com/ggml-org/llama.cpp) for Android ARM64 with a full suite of compiler optimizations applied in a reproducible, containerized environment.

> **Target:** Android API 35 · ABI `arm64-v8a` · Tuned for Cortex-A78

---

## Optimization Pipeline

This builder applies four distinct optimization passes in sequence. Each pass builds on top of the previous one.

```
Source Code
    │
    ▼
┌─────────────────────────────────────────────┐
│  Phase 1 — Instrumented Build                       │
│  PGO Generate + ThinLTO + MLGO                      │
└─────────────────────────────────────────────┘
    │
    ▼  (run on device, collect profiles)
┌─────────────────────────────────────────────┐
│  Phase 2 — Optimized Build                          │
│  PGO Use + ThinLTO + MLGO + Security Hardening      │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  BOLT — Binary Layout Optimization                  │
│  Code reordering based on execution frequency       │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  Strip + Package                                    │
│  llvm-strip + Zstd level 19                         │
└─────────────────────────────────────────────┘
    │
    ▼
llama.cpp.tar.zst
```

### Why each optimization matters

| Optimization | What it does |
|---|---|
| **PGO** (Profile-Guided Optimization) | Recompiles the binary using real execution data collected from the target device. Enables the compiler to make better decisions about branch prediction, inlining, and code layout based on actual workloads rather than heuristics. |
| **ThinLTO** | Performs whole-program analysis and optimization at link time across translation units. Enables cross-file inlining and dead code elimination without the full compile-time cost of monolithic LTO. |
| **MLGO** | Uses trained ML models to guide two compiler decisions: function inlining (replacing heuristic cost models) and register allocation eviction (reducing spills). Available in NDK r29+ (Clang 21). |
| **BOLT** | Post-link binary optimizer that reorders basic blocks and functions according to a hot/cold split, improving instruction cache utilization and reducing branch mispredictions at runtime. |
| **Zstd -19** | Maximum-ratio compression for the final release archive using all available CPU threads. |

---

## Requirements

| Requirement | Notes |
|---|---|
| **Docker** | Must be running. The build is fully containerized on Arch Linux. |
| **adb** | Required to push binaries to the device and pull profile data back. |
| **Android device** | ARM64. Cortex-A78 or newer recommended for best results. |

No other dependencies need to be installed on the host — the Docker container handles the NDK, compiler toolchain, and all build tools.

---

## Usage

### Phase 1 — Instrumented Build

```bash
bash build.sh
```

This phase will:

1. Pull the Arch Linux base image and install all build dependencies
2. Download Android NDK r29 inside the container
3. Clone the latest `llama.cpp` at HEAD
4. Build with instrumentation flags (`-fprofile-generate`) so the binaries collect runtime profile data when executed
5. Output the instrumented binaries to `./BUILD_BEFORE/bin`

Once complete, the script prints the exact `adb` commands needed for the next step.

> **Note:** The binaries produced in Phase 1 run slower than normal. This is expected — they contain instrumentation overhead. Do not use them for benchmarking.

---

### Device Profiling (between Phase 1 and Phase 2)

Run the instrumented binaries on the target device to generate profile data. The more representative the workload, the better the final binary will be optimized.

```bash
# Push binaries to device
adb push ./BUILD_BEFORE/bin /data/local/tmp/llama
adb shell chmod +x /data/local/tmp/llama/*

# Run inference workloads (repeat with different prompts/models for better coverage)
adb shell /data/local/tmp/llama/llama-cli \
    -m /sdcard/model.gguf \
    -p "Your prompt here" \
    -n 128

# Pull the generated profile data back to the host
adb pull /sdcard/pgo_data ./pgo_profiles/raw
```

> **Tip:** Profile data from multiple devices can be merged. Simply pull from each device into the same `./pgo_profiles/raw/` directory before running Phase 2.

---

### Phase 2 — Optimized Build

```bash
bash build.sh --phase2
```

This phase will:

1. Merge all `*.profraw` files in `./pgo_profiles/raw/` into a single `merged.profdata` using `llvm-profdata`
2. Recompile llama.cpp with the merged profile (`-fprofile-use`), ThinLTO, MLGO, and security hardening flags applied
3. Run BOLT on each output binary to reorder code layout for cache efficiency
4. Strip all debug symbols with `llvm-strip`
5. Package the final binaries into `llama.cpp.tar.zst` using Zstd at compression level 19

---

## Output Structure

```
./BUILD_BEFORE/         Instrumented binaries from Phase 1.
                        Used for device profiling only. Not for production use.

./BUILD_OPT/            PGO + ThinLTO + MLGO optimized binaries from Phase 2.

./BUILD_BOLT/           Final binaries after BOLT layout optimization and symbol stripping.
                        These are the production binaries.

./pgo_profiles/
    raw/                Raw *.profraw files pulled from the device.
    merged.profdata     Merged profile used for the Phase 2 build.

./llama.cpp.tar.zst     Release archive containing BUILD_BOLT.
                        This is the final deliverable.
```

---

## Configuration

All tunable parameters are declared at the top of `build.sh` as named variables.

### Target

```bash
Architecture="armv8.2-a"                          # Base ARM architecture
Architecture_Features="dotprod+fp16+crypto+crc"   # ISA extensions
Processor="cortex-a78"                            # Microarchitecture tuning target
Processor_ABI="arm64-v8a"                         # Android ABI
Android_Platform="android-35"                     # Minimum API level
NDK_VERSION="r29"                                 # Android NDK version
```

### Compiler Flags

```bash
# Architecture
Arch_Flags="-march=armv8.2-a+... -mtune=cortex-a78"

# Optimization
Opt_Flags="-O3 -ffast-math -fno-finite-math-only -funroll-loops -fomit-frame-pointer"
#   -fno-finite-math-only is required: llama.cpp uses NaN and Infinity internally.
#   Omitting it causes a compile error.

# MLGO (NDK r29 / Clang 21+)
MLGO_Flags="-mllvm -enable-ml-inliner=release -mllvm -regalloc-enable-advisor=release"

# ThinLTO
LTO_Flags="-flto=thin -fwhole-program-vtables"

# PGO
PGO_Gen_Flags="-fprofile-generate=/sdcard/pgo_data"        # Phase 1
PGO_Use_Flags="-fprofile-use=merged.profdata -fprofile-correction"  # Phase 2

# Security (Phase 2 only)
Security_Flags="-fstack-protector-strong -D_FORTIFY_SOURCE=2"

# Linker
Linker_Flags="-Wl,--gc-sections -Wl,-O2"
```

### CMake

```bash
CMake_Extra="-DGGML_STATIC=ON -DBUILD_SHARED_LIBS=OFF"
```

---

## How PGO Works

Standard compilers optimize based on static analysis and general heuristics. PGO replaces those heuristics with real data.

```
Phase 1: Build with -fprofile-generate
    → Binary records execution counts for every branch and function at runtime

Device: Run the instrumented binary on real workloads
    → *.profraw files are written to /sdcard/pgo_data/

llvm-profdata merge: Combine all *.profraw into merged.profdata
    → Aggregates frequency data across all runs

Phase 2: Build with -fprofile-use=merged.profdata
    → Compiler now knows exactly which code paths are hot and which are cold
    → Hot paths get inlined, unrolled, and placed for cache locality
    → Cold paths are moved away to reduce binary working set
```

The quality of PGO optimization is directly proportional to the representativeness of the profiling workload. Running a variety of prompts and model sizes during the device profiling step will produce better results than a single short run.

---

## Troubleshooting

**`No *.profraw files found`**
Profile data was not pulled from the device. Run `adb pull /sdcard/pgo_data ./pgo_profiles/raw` before invoking Phase 2.

**`error: some routines require non-finite math`**
`-ffast-math` enables `-ffinite-math-only`, which breaks llama.cpp. Ensure `-fno-finite-math-only` is present in `Opt_Flags` after `-ffast-math`.

**`Unknown command line argument '-regalloc-evict-advisor'`**
Flag was renamed in Clang 21. Use `-regalloc-enable-advisor=release` instead.

**BOLT silently copies the binary without optimizing**
BOLT requires the binary to contain relocation information. If it fails, it falls back to copying the unmodified binary. The build will still succeed.
