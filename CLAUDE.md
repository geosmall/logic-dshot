# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dshot protocol analyzer plugin for Saleae Logic 2. Decodes Dshot digital protocol frames (used for ESC/drone motor communication) from captured logic analyzer data. The plugin is a dynamic library loaded by the Saleae Logic 2 application.

## Build System

Uses **CMake**. The SDK is included as a git submodule in `sdk/`.

```bash
# Clone (include --recursive for the SDK submodule)
git clone --recursive <repo-url>

# Configure and build
cmake -S . -B build
cmake --build build

# Output: build/Analyzers/libDshotAnalyzer.so
```

Prerequisites: `cmake`, `g++` (or clang), `build-essential`.

To load in Logic 2: Settings → Custom Low Level Analyzers → browse to `build/Analyzers/` → restart Logic 2.

## Architecture

The plugin implements the Saleae Analyzer SDK interface (inheriting from `Analyzer2`) via four classes, all in `src/`:

- **DshotAnalyzer** — Main analyzer. `SetupResults()` initializes results; `WorkerThread()` runs the frame decode loop: reads edges from the input channel, measures pulse widths to determine bit values (>50% of nominal bit period = 1), assembles 16-bit frames, validates CRC, and emits annotated frames/markers.
- **DshotAnalyzerSettings** — User-configurable input channel and bitrate selection (Dshot150/300/600/1200). Handles settings serialization.
- **DshotAnalyzerResults** — Formats decoded frames for display (bubble text) and file export (text/CSV). Frame data contains: 11-bit channel value, telemetry request flag, CRC status.
- **DshotSimulationDataGenerator** — Produces synthetic Dshot waveforms for testing in Logic's simulation mode without real hardware.

The plugin exports three C functions (`GetAnalyzerName`, `CreateAnalyzer`, `DestroyAnalyzer`) that the Saleae Logic application uses to discover and instantiate the analyzer.

## Saleae SDK

The SDK lives in `sdk/` as a git submodule (per Saleae recommendation for reproducible builds). `cmake/ExternalAnalyzerSDK.cmake` detects the local SDK automatically; it falls back to FetchContent from GitHub only if the submodule is missing.

## Migration Plan

`LOGIC2_MIGRATION.md` is the authoritative migration plan for porting this plugin from Saleae Logic 1 to Logic 2. When the plan changes, that document must be updated accordingly — it is the single source of truth for migration scope, steps, and status.

## Dshot Protocol

Each frame is 16 bits: 11-bit throttle value (0-2047), 1-bit telemetry request, 4-bit CRC (XOR of the three channel value nibbles). Bits are encoded by pulse width ratio within a fixed bit period determined by the selected Dshot rate.
