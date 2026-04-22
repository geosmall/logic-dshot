# Port logic-dshot from Saleae Logic 1 to Logic 2

## Context

This project is a Dshot protocol analyzer plugin currently built for Saleae Logic 1 using an older SDK (v1.1.32) and the Qbs build system. Saleae Logic 2 uses the **same AnalyzerSDK C++ API** — so the core decode logic in `WorkerThread()` doesn't change. The port is primarily a build-system migration (Qbs → CMake) plus a few modernization tweaks to match the current SampleAnalyzer conventions.

## What changes and what doesn't

| Area | Logic 1 (current) | Logic 2 (target) |
|---|---|---|
| Build system | Qbs (`logic-dshot.qbs`) | CMake (`CMakeLists.txt` + `cmake/ExternalAnalyzerSDK.cmake`) |
| SDK inclusion | Git submodule pinned at v1.1.32 | Git submodule (retained); CMake finds it locally via `ExternalAnalyzerSDK.cmake` |
| Base class | `Analyzer` | `Analyzer2` (adds `SetupResults()` virtual) |
| Results init | Created inside `WorkerThread()` | Moved to `SetupResults()` override |
| Smart pointers | `std::auto_ptr` (deprecated C++11, removed C++17) | `std::unique_ptr` |
| Source dir | `source/` | `src/` (SampleAnalyzer convention) |
| Preprocessor | (none) | `-DLOGIC2` |
| Decode logic | Unchanged | Unchanged |

## Steps

### 1. Add CMake build system

Create `CMakeLists.txt` at project root, modeled on SampleAnalyzer:

```cmake
cmake_minimum_required(VERSION 3.13)
project(DshotAnalyzer)
add_definitions(-DLOGIC2)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.14" CACHE STRING "" FORCE)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
include(ExternalAnalyzerSDK)

set(SOURCES
    src/DshotAnalyzer.cpp
    src/DshotAnalyzer.h
    src/DshotAnalyzerResults.cpp
    src/DshotAnalyzerResults.h
    src/DshotAnalyzerSettings.cpp
    src/DshotAnalyzerSettings.h
    src/DshotSimulationDataGenerator.cpp
    src/DshotSimulationDataGenerator.h
)
add_analyzer_plugin(${PROJECT_NAME} SOURCES ${SOURCES})
```

### 2. Add `cmake/ExternalAnalyzerSDK.cmake`

Copy this module from the SampleAnalyzer repo. It:
- Sets C++11 standard
- Checks for the `Saleae::AnalyzerSDK` target (provided by the local submodule's `AnalyzerSDKConfig.cmake`)
- Falls back to `FetchContent` from GitHub only if the submodule is missing
- Defines `add_analyzer_plugin()` which creates a MODULE library linked to `Saleae::AnalyzerSDK`
- Sets output directory to `build/Analyzers/`

### 2a. Retain SDK as git submodule

Saleae recommends using a git submodule (not FetchContent) for reproducible builds and version pinning. The `sdk/` submodule is retained from the original project, pointing to `https://github.com/saleae/AnalyzerSDK.git`. The CMake module detects the local SDK and skips the network fetch. Clone with `--recursive` to get the SDK automatically.

### 3. Move source files from `source/` to `src/`

Rename the directory to match SampleAnalyzer convention. All 8 files (4 `.h` + 4 `.cpp`).

### 4. Update `DshotAnalyzer` to use `Analyzer2` base class

**`src/DshotAnalyzer.h`:**
- Change `class DshotAnalyzer : public Analyzer` → `class DshotAnalyzer : public Analyzer2`
- Add `virtual void SetupResults();` declaration
- Change `std::auto_ptr` → `std::unique_ptr` for mSettings, mResults

**`src/DshotAnalyzer.cpp`:**
- Extract the results-initialization block from the top of `WorkerThread()` into a new `SetupResults()` method:
  ```cpp
  void DshotAnalyzer::SetupResults()
  {
      mResults.reset(new DshotAnalyzerResults(this, mSettings.get()));
      SetAnalyzerResults(mResults.get());
      mResults->AddChannelBubblesWillAppearOn(mSettings->mInputChannel);
  }
  ```
- Remove those three lines from `WorkerThread()`

### 5. Update `DshotAnalyzerSettings` to use `std::unique_ptr`

**`src/DshotAnalyzerSettings.h`:**
- Change `std::auto_ptr` → `std::unique_ptr` for the two setting interface members

### 6. Remove old build artifacts

- Remove `logic-dshot.qbs`
- Remove `scripts/` (Travis CI install scripts)
- Remove `.travis.yml` and `appveyor.yml`
- Retain `sdk/` submodule and `.gitmodules` (Saleae-recommended approach for version pinning)

### 7. Build and test on Ubuntu 24

```bash
# Install prerequisites (if needed)
sudo apt install cmake g++ build-essential

# Clone (include --recursive for the SDK submodule)
git clone --recursive https://github.com/<owner>/logic-dshot.git

# Build
cmake -S . -B build
cmake --build build

# Output: build/Analyzers/libDshotAnalyzer.so
```

**Load in Logic 2:**
1. Install Saleae Logic 2 for Linux (AppImage or .deb from saleae.com)
2. Open Logic 2 → Edit → Settings
3. Scroll to "Custom Low Level Analyzers"
4. Browse to `<project>/build/Analyzers/`
5. Restart Logic 2
6. "Dshot" should appear in the analyzer list

### 8. Update CLAUDE.md

Update build commands and remove references to Qbs, Travis, AppVeyor.

## Files to create
- `CMakeLists.txt`
- `cmake/ExternalAnalyzerSDK.cmake`
- `.gitignore`

## Files to modify (move + edit)
- `source/*.h` and `source/*.cpp` → `src/` (then edit as described in steps 4-5)

## Files retained
- `sdk/` (submodule) and `.gitmodules` — kept for reproducible builds per Saleae recommendation

## Files to remove
- `logic-dshot.qbs`
- `scripts/travis-install-linux.sh`
- `scripts/travis-install-osx.sh`
- `.travis.yml`
- `appveyor.yml`

## Verification

1. `cmake --build build` completes without errors
2. `build/Analyzers/libDshotAnalyzer.so` exists
3. `nm -D build/Analyzers/libDshotAnalyzer.so | grep -E "GetAnalyzerName|CreateAnalyzer|DestroyAnalyzer"` shows all three exports
4. If Logic 2 is installed: load the plugin and verify "Dshot" appears in the analyzer list

## Future work

### 9. Add GitHub Actions CI workflow

Replace the removed Travis CI and AppVeyor configs with a single `.github/workflows/build.yml` that builds on all three platforms. The SampleAnalyzer repo has a reference workflow. Target matrix:
- **Linux**: x86_64, ARM64
- **macOS**: x86_64, ARM64
- **Windows**: x86_64, ARM64

Automatically create GitHub releases with pre-built binaries for tagged commits.

### 10. Add FrameV2 / HLA support

The current analyzer uses FrameV1 (`AddFrame`) exclusively — see `src/DshotAnalyzer.cpp` WorkerThread frame-emit block. FrameV2 is required to enable Python High Level Analyzers (HLAs) in Logic 2 (≥ 2.3.43).

**Decision: emit both FrameV1 and FrameV2 (dual-emit), do not drop FrameV1.**

Rationale:
- Saleae's FrameV2 documentation states *"The existing 'Frame' class is still exclusively used to produce the displayed bubbles"* — dropping FrameV1 would break bubble rendering in the analyzer track. Source: <https://www.saleae.com/support/extensions-api/protocol-analyzer-sdk/framev2-hla-support-analyzer-sdk>.
- `DshotAnalyzerResults.cpp` builds bubble text and CSV/text export rows by reading FrameV1 fields (`mData1`, `mFlags`) via `GetFrame()` at lines 22, 52, 74. Switching those readers to FrameV2 is possible but has no user-visible benefit and would expand the diff into files that otherwise need zero changes.
- FrameV2 is designed to be **additive**: a single analyzer emits one FrameV1 and one FrameV2 per decoded protocol frame. Confirmed by Saleae's own I2C analyzer, which calls `UseFrameV2()` in its constructor and then emits both frames together for every decoded byte:
  ```cpp
  mResults->AddFrame( frame );
  mResults->AddFrameV2( framev2, framev2Type, starting_sample,
      result ? potential_ending_sample : last_valid_sample );
  ```
  Source: <https://github.com/saleae/i2c-analyzer/blob/master/src/I2cAnalyzer.cpp> (inside `GetByte()`). Note that Saleae's `SampleAnalyzer` reference repo still uses FrameV1 only and is not a guide for FrameV2 integration — shipping analyzers like `i2c-analyzer` are the canonical reference.

**Prerequisites — SDK submodule bump:**

The `sdk/` submodule is currently pinned at commit `31af8aa` ("lib and include folders added from 1.1.32 Analyzer SDK"). **This revision predates FrameV2 entirely** — `sdk/include/AnalyzerResults.h` defines only the legacy `Frame` class, and no `FrameV2.h` or `AddFrameV2()` exists.

FrameV2 was introduced upstream in these commits on `saleae/AnalyzerSDK`:
- `992a6a4` — "Add FrameV2" (the `FrameV2` class and `AddFrameV2()` method)
- `6db2f87` — "Move type and samples to AddFrameV2" (final API shape)
- `882bca6` — "added UseFrameV2" (required constructor opt-in)
- `84dd09e` — "Add LOGIC2 ifdef around Logic2-specific functionality" (matches our existing `-DLOGIC2`)
- `7ff3939` — "Add FrameV2::AddByteArray" (not needed for Dshot but part of the mature API surface)

Saleae does not ship semver release tags for the AnalyzerSDK repo (only `alpha-1` exists); the canonical "version" is a git SHA on `master`. Recommended target: the tip of `origin/master` (as of 2026-04: `e3c4b4f`, which also includes ARM64 build support on Linux/macOS/Windows — aligns with the CI matrix from step 9).

Minimum viable SHA is `84dd09e` (last commit needed for FrameV2 + `LOGIC2` ifdef), but there is no reason to pin below current `master`.

**Code changes (all in `src/DshotAnalyzer.cpp`):**
- In the constructor, call `UseFrameV2()` once.
- In `WorkerThread()`, immediately after the existing `mResults->AddFrame(frame)`, also build and emit a FrameV2 with type string `"dshot"` and the fields:
  - `AddInteger("throttle", channel_value)` — 11-bit payload, 0–2047
  - `AddBoolean("telemetry", telemetry_bit)` — telemetry-request bit
  - `AddBoolean("crc_valid", crc_ok)` — CRC match result
  - `AddInteger("crc", crc_nibble)` — raw 4-bit CRC (0–15), useful for HLAs that debug CRC calculation
- Use the same `starting_sample` / `ending_sample` as FrameV1.

**Out of scope for this step** — each item below was considered and deliberately excluded:

- **`DshotAnalyzerResults.{h,cpp}`.** The bubble-text and CSV/text-export readers (`GenerateBubbleText`, `GenerateExportFile`, `GenerateFrameTabularText` at lines 22, 52, 74) already consume FrameV1 `mData1`/`mFlags` via `GetFrame()` and render correctly. Saleae mandates FrameV1 for bubbles, and FrameV2 does not participate in the export path — rewriting these readers has no user-visible benefit and would expand the diff into files that otherwise need zero changes.
- **`DshotAnalyzerSettings.{h,cpp}`.** No new user-facing setting is introduced. FrameV2 fields are always emitted (there is no cost to emitting them and no HLA to toggle from the analyzer side), so no setting, serialization change, or UI control is required.
- **`DshotSimulationDataGenerator.{h,cpp}`.** The simulator produces synthetic wire-level transitions; FrameV2 is built by the decoder (`WorkerThread()`) from those transitions. Simulated captures will automatically flow through the new dual-emit path without any generator change.
- **Dropping FrameV1 entirely.** Covered under the decision rationale above — would break bubbles and export. Not revisited.
- **Shipping a Python HLA.** The analyzer's job is to emit FrameV2 correctly; HLAs are authored separately (different repo, different release cadence, often user-specific). A reference HLA can be added later as a docs-only example if demand exists.
- **Adding automated tests.** There is no existing test suite for this analyzer (the project has always relied on Logic 2's simulation mode for manual verification). Introducing a harness purely to cover FrameV2 emission is out of proportion to the change; the verification steps below exercise the same path end-to-end.
- **CI matrix changes.** Step 9's workflow already builds all six target triples. The SDK bump to a FrameV2-capable revision (which also ships ARM64 libs) is consumed transparently by the existing workflow; no `.github/workflows/build.yml` edits are needed.

**Verification:**
1. `git -C sdk log --oneline | grep -E "FrameV2|UseFrameV2"` shows the FrameV2 commits are present after the submodule bump.
2. `cmake --build build` succeeds after SDK bump and code changes.
3. Bubble rendering unchanged in Logic 2 (FrameV1 regression check — confirms dual-emit did not disturb bubble path).
4. A minimal Python HLA subscribing to `"dshot"` FrameV2s receives `throttle` / `telemetry` / `crc_valid` / `crc` fields.
5. CSV export unchanged.
