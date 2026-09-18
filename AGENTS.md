# AGENTS.md

Chat instructions override this file.
Smallest correct change. Plausibility is not correctness.

This repo is **M5 Zeroputer**: Cardputer Adv (ESP32-S3) + RP2040 Zero.
Firmware first. Project need is dual-MCU embedded control, not a host SDK.

---

## Never

- Invent pin maps, UART/SPI framing, baud rates, or HAL return values. Read the schematic, `platformio.ini`, or existing headers.
- Claim a flash, build, or test passed unless you ran it.
- Commit secrets, Wi-Fi passwords, API keys, `.env`, or dumped flash.
- Add Python, Rust, Node, or CMake because a template mentioned them.
- Rewrite one MCU's firmware in the other MCU's tree "to unify."
- Block in an ISR. malloc/printf in an ISR. Ignore `esp_err_t` or Pico SDK status.
- Touch `logo.png`, applied board defs, or generated files.

---

## Stack (this repo)

| Need | Code lives | Tooling |
|---|---|---|
| UI, keyboard, display, Wi-Fi/BLE, audio, SD, shell | Cardputer Adv / ESP32-S3 | PlatformIO `espressif32`, Arduino or ESP-IDF as `platformio.ini` says |
| Hard real-time GPIO, PIO, extra pins | RP2040 Zero | PlatformIO `raspberrypi` / Pico SDK |
| Inter-board link | Shared protocol headers only | Framed UART control + SPI bulk. Do not invent a third bus |
| Host utilities (if the task names them) | A `host/` or `tools/` dir, only after asking | Do not create this to "help" |

If `platformio.ini` does not exist yet, create **two** environments in one file (`cardputer` and `rp2040`), not two build systems.
If source layout is missing, use:

```text
firmware/cardputer/   # ESP32-S3
firmware/rp2040/      # RP2040 Zero
firmware/shared/      # protocol frames, opcodes, CRC — no HAL calls
docs/                 # pin map, protocol spec
```

Do not put both MCUs' main loops in one translation unit.

---

## Commands

```text
# Install PlatformIO CLI if missing (once)
pipx install platformio

# Build
pio run -e cardputer
pio run -e rp2040

# Flash (board must be connected)
pio run -e cardputer -t upload
pio run -e rp2040 -t upload

# Serial
pio device monitor -e cardputer
pio device monitor -e rp2040

# Static check when configured
pio check -e cardputer
pio check -e rp2040
```

If env names in `platformio.ini` differ, use those names. Do not invent `cmake --build`.

Host sanitizers do not apply to firmware on-device. Done means: the env you edited **builds**, and you state whether it was flashed.

---

## Error-proofing

- Check every ESP-IDF / Pico / Arduino call that returns a status.
- UART frames: length, opcode, CRC/checksum, max size. Drop truncated frames. Never treat short reads as zeros.
- SPI bulk: timeout, CS discipline, explicit length. No unbounded `memcpy` from a wire length.
- Timeouts on every bus wait. Watchdog-friendly loops on both MCUs.
- Bounded buffers only. `snprintf`, never `sprintf`. No `gets`.
- Init order documented: clocks → buses → protocol → UI/app.
- Shared `firmware/shared` types are packed and sized the same on both cores. Static assert sizes.
- Shell commands (`help`, `status`, `gpio`, `pio`, `craft`, …) parse arguments; reject unknown tokens; do not execute raw strings as firmware.

---

## Code style

- C11 / C++17. Match whatever the first source file uses.
- Braces on every `if`/`for`/`while`.
- Pin names from a single header per board. No raw GPIO numbers in app code.
- Comments only for why, invariants, and wire layout.
- No new dependency, library, or PlatformIO lib without asking.

---

## Workflow

1. Name which MCU the change belongs to. If both, change `firmware/shared` plus both sides in one task only when the wire contract changes.
2. Read the current pin map and frame struct before editing.
3. Patch the smallest path. Add a parse/reject test or a host-side frame fixture if one exists.
4. `pio run -e <env>` for every env you touched.
5. Report: files, env built, flashed or not, residual risk (no hardware, no monitor log, etc.).

## Stop

- Pinout or connector is unspecified and would change hardware.
- Protocol change would break the other MCU and you only have one tree open.
- Board not present and the task requires a flash or bus capture.
- Task needs credentials or radio behavior you cannot legally or physically exercise.

## Done

- Behavior exists on the MCU the need required.
- Invalid frames and HAL failures are explicit.
- Touched env built because you ran `pio run`.
- Diff does not add a second language or build system.
