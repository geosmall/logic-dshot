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
