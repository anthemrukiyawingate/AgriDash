# AgriDash

Firmware for a small environmental sensor node on a Raspberry Pi Pico, in C++17. It reads temperature, pressure and humidity once a second and writes one JSON record per line over USB. It was built test-first, to a written specification, with a defined behaviour for every failure the specification lists.

**See it run:** open this repository's GitHub Pages site (the link is in the About panel of the repository). The front page runs the firmware in your browser. It is not a recording.

## What the page does

| On the page | What it is |
|---|---|
| A live node | The firmware's core, compiled unmodified to a WebAssembly engine of about 8 KB, sampling a simulated sensor. Unplug the sensor, short the data line, hang the main loop, close the serial port, and watch the records it writes. |
| The test suite | The release's own tests, compiled to WebAssembly. Press one button and they run on your device. |
| A field of 1,024 nodes | 1,024 independent instances of the same sampler under simulated storms. An oracle that knows the simulation's ground truth checks every record every node writes against the specification, and counts. You can paint faults onto the field, and you can plant a lie to see the oracle catch it. |
| The node's memory | The bytes of the fixed 256-byte line buffer at the moment a record is written. |
| The binary that ships | `agri_node.uf2`, the file that is copied onto a Pico, booted on an emulated RP2040 processor (rp2040js) and compared, measurement by measurement, with the WebAssembly engine. |

The page states what is real (the node's logic) and what is simulated (the sensor chip, the wire, the clock, the watchdog timer, the serial port).

## What "fails safely" means here

A sensor node fails safely when it never reports a number it did not measure, never stops silently, and says why it restarted.

| # | Failure | What the node does | What the host sees |
|---|---|---|---|
| F1 | Sensor not fitted | Runs without it; looks again every 30 s | `"st":"absent"`, no value |
| F2 | Sensor stops answering | Marks the sample; initialises the sensor again before the next | `"st":"fault"`, no value |
| F3 | I²C bus held low | Bounded wait (50 ms), then bus recovery, at most three attempts per period | `"st":"fault"`, no value |
| F4 | Implausible reading | Withholds it | `"st":"range"`, no value |
| F5 | Main loop hangs | Hardware watchdog resets the node after 4 s | Next boot record says `"reset":"watchdog"` |
| F6 | No host listening | Keeps sampling; never waits for a host | A gap in `seq` when the host returns |
| F7 | A record would exceed 256 bytes | Refuses to truncate; sends an error record | `"kind":"error","code":"line_too_long"` |
| F8 | Power loss or brown-out | Clean start | Boot record says `"reset":"power_on"` |

Every row has at least one test, written before the code. The full specification, including the wire protocol, is in `docs/SPEC.md` in the release.

## The records

```json
{"v":1,"kind":"boot","fw":"1.0.0","reset":"power_on","period_ms":1000,"sensors":{"bme280":"ok"}}
{"v":1,"kind":"sample","seq":42,"up_ms":42137,"bme280":{"st":"ok","t_c":21.37,"p_pa":100512.4,"rh":48.2}}
{"v":1,"kind":"sample","seq":43,"up_ms":43137,"bme280":{"st":"fault"}}
```

A value is present only when its status is `ok`. Field names carry their unit. `seq` rises by exactly one per period, so a host can count what it missed.

## How it is built

- **Core** (`core/`): the sampler, the BME280 driver and a bounded JSON encoder. Portable C++17 with no platform headers, no heap, no exceptions and no RTTI. Hardware is reached through four small interfaces: I²C bus, clock, watchdog, line sink.
- **Platform layers**: `platform/pico/` (Pico SDK) for the board, and `site/bridge.cpp` for the browser. Neither contains node logic.
- **Tests** (`tests/`): host tests with a scripted I²C bus that can inject every failure above. Driver tests use the worked example from the sensor's datasheet.

## Measured

These figures are produced by the commands beside them. The page's own figures are written by its build script from the same runs.

| Measure | Value | Command |
|---|---|---|
| Host tests | 57 test cases, 1072 assertions, all passing | `cmake -S . -B build -G Ninja && cmake --build build && ./build/agri_tests` |
| Line coverage of the core | 99.68% (313 of 314 lines) | `tools/coverage.sh` |
| Compiler warnings | None, with `-Wall -Wextra -Wpedantic -Wconversion -Wshadow -Werror`, under GCC 13 (host), Clang 23 (WebAssembly) and GCC 14 (ARM) | the three builds |
| Front page, automated check | Every failure mode above, the in-browser test run, two simulated minutes of the field with zero oracle violations, and the emulated board, at desktop and phone widths | `python3 site/build.py && node site/check_page.mjs` |

## Status and limits

- Not yet verified on hardware. The board image builds and runs on an emulated RP2040; the on-device checklist in docs/SPEC.md section 8 has not been run.
- One sensor driver is included: the BME280. Light, UV and VOC drivers for the same sensor board are planned and are left out until each passes its tests.
- The emulated-board mode on the page does not show the stuck bus or the watchdog reset. The emulator has no pin-level model of the wire and does not reset the chip. Those two behaviours are shown on the WebAssembly engine and are checked on a real board.
- Continuous integration runs where the source lives; this repository's front page and README are published from a release.

## Get the code

The source, tests and documentation are in the release archive under **Releases**. This repository's tree holds the front page and this file.

```
tar xzf agridash-0.9.0.tar.gz && cd agridash-0.9.0
cmake -S . -B build -G Ninja && cmake --build build && ./build/agri_tests      # host tests
cmake -S platform/pico -B build-pico -G Ninja -DPICO_SDK_PATH=<pico-sdk>        # board image
cmake --build build-pico                                                        # -> build-pico/agri_node.uf2
```

## Licence

MIT. Copyright (c) 2026 Anthem Rukiya Wingate and the etherware collective. See `LICENSE`.

Third-party components keep their own licences: doctest (MIT) for the host tests; on the front page only, rp2040js by Uri Shaked (MIT) and the RP2040 boot ROM (BSD-3-Clause, Raspberry Pi Ltd).
