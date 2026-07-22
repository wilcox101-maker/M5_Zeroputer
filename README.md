# M5 Zeroputer

**Cardputer Adv + RP2040 Zero Hybrid Controller**

<image-card alt="M5 Zeroputer Logo" src="logo.png" ></image-card>

A powerful portable/vehicle dual-processor platform with a rich UI, OS-style shell with Red Hat hacking tools, full audio support, and versatile real-time control.

## Overview

**M5 Zeroputer** fuses the **M5Stack Cardputer Adv** (ESP32-S3) and **RP2040 Zero** into a true hybrid system:

- **Cardputer Adv**: User interface, 1.14" display, 56-key keyboard, audio (playback + recording), SD storage, Wi-Fi/BLE, battery.
- **RP2040 Zero**: Hard real-time PIO engine, extra GPIO, deterministic control.

Communication uses a robust **UART control plane** + **SPI high-bandwidth data plane**.

## Key Features

- **OS-Style Shell** with Red Hat hacking extensions (packet crafting, fuzzing, bus sniffing, scripting, memory tools, etc.)
- **Full Audio Support**: MP3/WAV playback & microphone recording
- **PIO-powered Real-time Control** (LEDs, sensors, actuators, precise timing)
- **Hybrid Protocol**: Durable framed UART + SPI bulk transfers with comprehensive diagnostics
- **Graphical UI** + keyboard-driven workflows
- **Broad Use Cases**: Automotive lighting, sensor logging, protocol hacking, retro gaming, portable dev tool, audio workstation, and more

## Hardware

- Main Board: M5Stack Cardputer Adv
- Co-processor: RP2040 Zero
- Connection: 14-pin EXT header (UART + SPI) + custom interposer PCB recommended

## Quick Start

1. Clone the repo
2. Open in PlatformIO
3. Flash both boards
4. Explore the shell (hotkey accessible)

(See `docs/` for detailed setup, pin maps, and protocol spec.)

## Shell Commands (Red Hat Hacking Style)

- Core: `help`, `status`, `counters`
- GPIO/PIO: `gpio`, `pio`
- Hacking: `craft`, `fuzz`, `sniff`, `mem`
- Audio: `play`, `record`, `reactive`
- Network: `wifi`, `ble`, `http`

## License

MIT License

---

**Made for embedded hacking and building.**
