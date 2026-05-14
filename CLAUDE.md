# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Game Bub is an open-source FPGA retro emulation handheld supporting Game Boy, Game Boy Color, and Game Boy Advance. The repo holds everything to build a physical device: PCB design (`pcb/`), 3D-printed shell STLs (`3d/`), FPGA HDL (`fpga/`), and microcontroller firmware (`firmware/handheld/`).

The hardware pairs a Xilinx XC7A100T FPGA (does all game emulation and most I/O) with an ESP32-S3 MCU (boots the device, configures the FPGA, renders the UI, loads ROMs from microSD). See `docs/architecture.md` for the full hardware/software architecture — read it before working on FPGA↔MCU interactions.

The two software components — `fpga/` (Chisel/SystemVerilog) and `firmware/handheld/` (Rust) — are developed independently and have separate toolchains.

## FPGA (`fpga/`)

The FPGA cores are custom, from-scratch Game Boy and GBA emulators written in **Chisel** (Scala), with a thin SystemVerilog top-level (`fpga/verilog/handheld/top.sv`) providing glue, PLLs, and the HDMI module. FuseSoC orchestrates Chisel elaboration (via `sbt`) and the Vivado/Verilator builds.

Three separate bitstreams are built from the same source, selected by a FuseSoC `--flag`: `boot` (loads the FPGA from the MCU), `gameboy`, and `gba`. The Chisel entry points are `platform.handheld.HandheldBoot` / `HandheldGameboy` / `HandheldGba`, all wrapping `HandheldTop`.

### Common commands (run from `fpga/`)

```sh
# Run the full Chisel/Scala test suite (CPU tests, bus tests, peripherals)
sbt test

# Run a single test spec
sbt "testOnly gba.cpu.ARM7TDMISpec"

# Build a bitstream (requires Vivado 2023.2 + Artix-7 support)
fusesoc --cores-root . --work-root=build/gameboy --target=handheld_rev2 --flag=gameboy elipsitz:gameboy:gameboy

# Build the Verilator simulator (see docs/simulator.md)
fusesoc --cores-root . run --target=sim --flag=module_gameboy elipsitz:gameboy:gameboy
# Run it (the fusesoc command above ends in a harmless "Failed to run" error)
./build/elipsitz_gameboy_gameboy_1.0.0/sim-verilator/VSimGba --bios-path <bios.bin> <rom.gba>
```

Most FPGA development happens in the Verilator simulator (`fpga/sim/`, C++ + SDL2) rather than rebuilding bitstreams — it's fast to compile but slow to run (~5 fps for GBA). Requires submodules checked out (`git submodule update --init --recursive`), plus `sbt`, `fusesoc`, `verilator`, and SDL2.

### Code layout

- `fpga/src/main/scala/gameboy/` — Game Boy / Color core (`cpu`, `ppu`, `apu`, `cart`, plus `Timer`, `Joypad`, DMA, etc.)
- `fpga/src/main/scala/gba/` — GBA core (`cpu` is ARM7TDMI, plus `ppu`, `apu`, `cart`, `mem`, `link`, DMA, timers). `GameBoyPlayer.scala` bridges the GB core for GBA's GBC backwards-compatibility mode.
- `fpga/src/main/scala/platform/handheld/` — board glue shared by all bitstreams: `HandheldTop`, SDRAM/SRAM controllers, `SpiReceiver` (MCU link), `I2sTransmitter` (audio), `DpiDriver` (display), `TripleBufferControl` (framebuffer).
- `fpga/src/main/scala/platform/sim/` — `SimGameboy` / `SimGba` top-levels for the Verilator sim.
- `fpga/src/main/scala/{axi,lib,xilinx}/` — reusable AXI, utility, and Xilinx-primitive wrappers.
- `fpga/verilog/` — SystemVerilog top-level + XDC constraints; `hdmi/` is a submodule.

The emulator cores are board-agnostic "inner modules"; `HandheldTop` instantiates one and handles SPI, interrupts, RAM arbitration between the core and the MCU, audio, and the rotated triple-buffered framebuffer. See `docs/architecture.md` for clock domains (`system`, `av`, `sdram`, `spi`) — clock-domain crossings are pervasive here.

## MCU Firmware (`firmware/handheld/`)

Rust on ESP-IDF (via `esp-idf-svc` / `esp-idf-hal`), targeting the ESP32-S3 Xtensa core. UI is built with **Slint** (`.slint` files in `res/ui/`, compiled at build time by `build.rs`). Requires the `esp` Rust toolchain (`rust-toolchain.toml` pins it) — install per the esp-rs book — and `espflash`.

### Common commands (run from `firmware/handheld/`)

```sh
# Build, flash, and run on a connected device.
# Device must be in flashing mode: hold "Home" while powering on.
cargo run --release --features=rev2

# After first flash, write device-specific factory data (serial, calibration)
python3 flash_nvs.py --serial <serialno> --revision 2
```

`rev1` / `rev2` cargo features select the hardware revision and are mutually required — always pass one. `flash.toml` / `espflash.toml` configure a pre-built QIO bootloader (`esp32s3-qio-bootloader-fast.bin`); see comments there for why.

### Code layout (`src/`)

- `main.rs` — entry point; spawns the UI, worker, and interrupt threads.
- `device/drivers/` — hardware drivers (`fpga`, `lcd`, `dac`, `imu`, `rtc`, `sdcard`, `fuel_gauge`, `io_expander`, `usb`). `device/interrupt.rs` runs the dedicated interrupt thread.
- `worker/` — background worker thread; communicates with the UI thread via message passing.
- `ui/` — Slint integration (`slint.rs`), button handling, and `ui/state/` (one module per screen: `main_menu`, `rom_select`, `settings`, `tools`, `game`).
- `bitstream/` — per-core drivers (`gameboy.rs`, `gba/`) that load the bootrom, configure the emulated cartridge, manage save files, and stream IMU data. `gba/game_db.rs` and `gba/save_type_detector.rs` determine cartridge save type.
- `kvs/` — key-value store backed by ESP NVS for persistent settings.

The MCU never drives the display directly — the FPGA does. UI frames are sent to the FPGA over QSPI as 15-bit RGB + alpha and composited with the emulator's video output.

## `python/`

Helper Jupyter notebooks and scripts for cartridge dumping and bring-up — not part of the device build. `requirements.txt` lists deps.

## License

Mixed by directory: `fpga/` and `firmware/` are GPL-3.0-only; `3d/` and `pcb/` are CC-BY-SA-4.0. The "Game Bub" name and logo are not covered — don't reuse them for derived products.
