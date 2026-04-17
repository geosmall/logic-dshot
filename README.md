# Dshot Protocol Analyzer for Saleae Logic 2

A custom low-level protocol analyzer plugin for [Saleae Logic 2](https://www.saleae.com/) that decodes [Dshot](https://brushlesswhoop.com/dshot-and-bidirectional-dshot/) digital protocol frames used for ESC/drone motor communication.

Forked from [tracernz/logic-dshot](https://gitlab.com/tracernz/logic-dshot) and ported to Logic 2 with a CMake build system.

## Features

- Decodes Dshot150, Dshot300, Dshot600, and Dshot1200
- Displays 11-bit throttle value, telemetry request flag, and CRC validation
- Bubble text annotations and markers on the waveform
- Export decoded frames as text/CSV
- Simulation data generator for testing without hardware

## Building

Prerequisites: `cmake`, `g++` (or clang), `build-essential`

```bash
git clone --recursive https://github.com/geosmall/logic-dshot.git
cd logic-dshot
cmake -S . -B build
cmake --build build
```

Output: `build/Analyzers/libDshotAnalyzer.so` (Linux), `.dylib` (macOS), or `.dll` (Windows)

## Loading in Logic 2

Point Logic 2 at the `build/Analyzers/` directory following [Saleae's setup guide](https://www.saleae.com/support/extensions-api/protocol-analyzer-sdk/setting-up-developer-directory).

## Dshot Protocol

Each frame is 16 bits:

| Bits   | Field              | Description                    |
|--------|--------------------|--------------------------------|
| 15-5   | Throttle value     | 11-bit value (0-2047)          |
| 4      | Telemetry request  | 1-bit flag                     |
| 3-0    | CRC                | XOR of the three value nibbles |

Bits are encoded by pulse width ratio within a fixed bit period determined by the selected Dshot rate.

## Acknowledgments

Original analyzer by [tracernz](https://gitlab.com/tracernz/logic-dshot).
