# Porting MiSTer Arcade Cores to Game Bub

This document describes how a developer would port a [MiSTer FPGA](https://github.com/MiSTer-devel)
arcade core to the Game Bub platform. It assumes you've read [architecture.md](architecture.md)
and [simulator.md](simulator.md).

> **Status:** Game Bub ships only the from-scratch Game Boy / GBA cores today. There is no
> arcade core in the tree. This is a design guide — but every Game Bub interface it
> references is real and current.

## 1. Language is not the obstacle — connectivity is

It is worth saying up front, because it shapes everything below: **the fact that Game Bub's
platform is written in Chisel is not a porting problem.**

- Vivado synthesizes mixed-language designs. The MiSTer core's SystemVerilog and the
  Chisel-generated SystemVerilog compile side by side.
- `top.sv` — the actual top-level — is already SystemVerilog. It instantiates `HandheldTop`
  (Chisel, emitted as `HandheldTop.sv`) and the `hdmi` module (SystemVerilog) together.
- The Chisel footprint you have to touch is a **thin shim**: a `Module` that BlackBox-
  instantiates your core and forwards a port bundle. The substantive adapter logic — the
  glue that actually does the connecting — can be written entirely in SystemVerilog if you
  prefer (see §6).

So don't think of this as "rewrite a MiSTer core in Chisel." Think of it as: the core stays
SystemVerilog, and you wire it into a fixed set of platform interfaces. The whole job is
**connectivity** — what bus connects to what, in what protocol, at what clock.

## 2. The integration point: the inner-module slot

Do **not** fork `top.sv` and re-instantiate peripherals yourself. `HandheldTop` already
implements, once, all the connectivity every core needs:

- the QSPI link to the MCU (deserializer, FIFOs, clock-domain crossing)
- the firmware-visible memory map (register space, SRAM, SDRAM, framebuffer, overlay, and
  the per-core "module" window)
- the triple-buffered framebuffer and the DPI/HDMI video output paths
- the audio clock-domain crossing, the I2S transmitter, and HDMI audio
- the SDRAM and async-SRAM controllers, plus arbiters that share them with the MCU
- the UI overlay compositor, interrupts, power sequencing, button synchronization

`HandheldTop` instantiates exactly **one inner module per bitstream** and wires it to all of
the above. That inner module is the slot you drop your core into. It is defined by the
`HandheldModule` trait and the `HandheldIo` bundle
(`fpga/src/main/scala/platform/handheld/HandheldTop.scala`). The existing cores —
`HandheldGameboy`, `HandheldGba`, `HandheldBoot` — are your reference fillings for that slot.

Forking `top.sv` instead would mean re-deriving every item in that list. Don't.

## 3. The connectivity map

A MiSTer core is written against the `sys/` framework: its top module (`emu`) talks to
`hps_io`, `pll`, a video mixer/scaler, an audio path, and optional `sdram`/`ddram` modules.
Porting is a matter of detaching each of those interfaces and reattaching it to the
equivalent Game Bub platform interface.

| MiSTer / `sys` interface | Game Bub platform interface | Notes |
|---|---|---|
| `CLK_50M`, the per-core `pll` and its several output clocks | One `clk_sys` into the inner module (board clock wizard) | You regenerate the rest — clock enables, or an MMCM in the shim. §7 |
| `RESET`, `pll_locked` | `io.reset` (active-high); platform owns board reset & power sequencing | §7 |
| `joystick_0/1`, keyboard/PS2 | `io.buttons` — 12-button bundle, pre-synchronized | §10 |
| `ioctl_download / _wr / _addr / _dout / _index` (ROM load) | A download window in `io.mcuInterface`, fed by the MCU bitstream driver | §11 |
| `status` bits (OSD options, DIP switches) | Config registers in your `RegisterMap`, written by the MCU | §11 |
| `RTC`, `TIMESTAMP` | Config registers fed by the MCU's RTC driver (firmware already does this for GB/GBA) | §11 |
| `ioctl_upload` / NVRAM save-load | `io.sram` + the firmware's `persist_save` path | §11, §12 |
| `VGA_R/G/B`, `VGA_HS/VS/DE`, `VGA_HBLANK/VBLANK`, `CE_PIXEL` | `framebufferX/Y/Data/WriteEnable` + `vblank` (pixel-addressed writes) | §9 |
| `VGA_SL`, `VGA_F1`, `VGA_SCALER`, `FB_*` | No equivalent — platform owns scaling/scanlines. Drop them. | §9, §13 |
| `CLK_AUDIO`, `AUDIO_L/R`, `AUDIO_S`, `AUDIO_MIX` | `io.audioLeft/Right` (`SInt(16)`); platform does CDC + I2S + HDMI audio | §8 |
| The core's bundled `sdram` controller module + `SDRAM_*` pins | `io.sdram` — a `MemoryInterface` to the platform's shared SDRAM controller | §12 |
| `DDRAM_*` | No DDR on this board — retarget to `io.sdram` or block RAM | §12 |
| `SD_*` (SD card) | Owned by the MCU; never exposed to the core | — |
| `LED_*`, `BUTTONS`, `OSD_STATUS` | No equivalent. Drop, or repurpose `io.vibrate` / `io.pmod`. | — |
| `UART_*`, `USER_IN/OUT` | `io.link` / `io.pmod` if you need them; otherwise stub | — |

The rest of this document walks each bridge in that table.

## 4. The platform side: `HandheldModule` / `HandheldIo`

This is the fixed contract your shim must satisfy. Definitive source:
`fpga/src/main/scala/platform/handheld/HandheldTop.scala`.

### 4.1 The trait

```scala
trait HandheldModule {
  def io: HandheldIo
  def framebufferW: Int     // core's native framebuffer width  (pixels)
  def framebufferH: Int     // core's native framebuffer height (pixels)
  def clockSystemHz: Int    // the rate clk_sys runs at for this bitstream
  def clockSdramHz: Int     // SDRAM clock; must be an integer multiple of clockSystemHz
}
```

`clockSdramHz` being an integer multiple of `clockSystemHz` is load-bearing — it keeps the
SDRAM clock-domain crossing cheap. `HandheldGameboy` uses `clockSystemHz * 4`.

### 4.2 `HandheldIo`

The inner module is clocked by Chisel's implicit clock, which `top.sv` drives with
`clk_sys`. Everything below is in that clock domain unless noted.

- **`enable: Input(Bool)`** — high when the core should run; gated by the MCU via
  `REG_CONTROL`. Usually AND'd into the core's clock-enable.
- **`reset: Input(Bool)`** — active-high core reset, also from `REG_CONTROL`.
- **`buttons: Input(HandheldButtons)`** — 12 active-high buttons (`a b x y up down left
  right l r start select`), already synchronized and de-inverted.
- **Video** — `framebufferX/Y: UInt(8.W)`, `framebufferData: ColorARGB.rgb555()`,
  `framebufferWriteEnable: Bool`, `vblank: Bool`. One pixel write per cycle; a rising edge
  of `vblank` latches the completed frame into the triple buffer.
- **Audio** — `audioLeft/audioRight: Output(SInt(16.W))`, held values (see §8).
- **`vibrate: Output(Bool)`** — rumble motor.
- **Cartridge** (`cartridgeEnabled`, `cartridge: HandheldCartridge`) — physical GB/GBA
  cartridge pins. An arcade core disables these (`cartridgeEnabled := false.B`, all
  `*Dir := false.B`).
- **`link: HandheldLink`, `pmod: HandheldPmod`** — link port and PMOD header; stub unless
  used.
- **`mcuInterface: MemoryInterface(addressWidth = 30, dataWidth = 32)`** — the MCU's window
  into your core. This is how firmware reads/writes config registers and streams ROM data.
  Split it with `MemoryMap` / `RegisterMap` (§11).
- **`sram: Flipped(MemoryInterface(18, 16))`** — 512 KiB async SRAM, 16-bit words, arbitrated
  with the MCU.
- **`sdram: Flipped(MemoryInterface(25, 32))`** — ~32 MiB SDRAM, 32-bit logical words,
  arbitrated with the MCU. Runs in the `clk_sys` domain; `HandheldTop` does the CDC to the
  real SDRAM clock.

### 4.3 The `MemoryInterface` handshake

`sram`, `sdram`, and `mcuInterface` all use the same protocol
(`fpga/src/main/scala/lib/mem/MemoryInterface.scala`):

```
address, write, dataWrite, writeStrobe  -- inputs
dataRead, done                          -- outputs
enable                                  -- raise to start; hold inputs until `done`
```

It is multi-cycle and may stall — the MCU arbiter can interleave its own accesses.
`HandheldGba`'s `DirectReadCache` usage and `HandheldGameboy`'s `emuCart` access state
machine show how to feed a core that expects single-cycle ROM reads from this.

## 5. The integration point in the build

The FPGA build is driven by FuseSoC via `fpga/gameboy.core`. Bitstreams are selected by a
`module_*` flag (`module_boot`, `module_gameboy`, `module_gba`); each flag pulls in
filesets, a Chisel `generate` target, and an inner-module class. You'll add `module_arcade`
the same way — see §14.

## 6. The Chisel shim (and why it's thin)

`HandheldTop` instantiates the inner module as `Module(genT)` with the type bound
`T <: Module with HandheldModule`. A raw Chisel `BlackBox` is **not** a `Module`, so the slot
can't be a bare BlackBox — but the wrapping `Module` is trivial:

```scala
class HandheldArcade extends Module with HandheldModule {
  val io = IO(new HandheldIo)
  def framebufferW = 240
  def framebufferH = 160
  def clockSystemHz = 48 * 1000 * 1000
  def clockSdramHz  = clockSystemHz

  // The MiSTer core + all adapter logic, as one SystemVerilog module.
  class ArcadeInner extends BlackBox {
    val io = IO(new HandheldIo)   // ports flatten to io_enable, io_buttons_a, io_sdram_address, ...
  }
  val inner = Module(new ArcadeInner)
  io <> inner.io
}
```

Two ways to split the work:

- **Adapter in SystemVerilog (recommended if you don't want to touch Scala).** Write one
  `arcade_inner.sv` that contains the MiSTer `emu` core *and* every bridge below (clocking,
  video, audio, input, ROM replay, config registers), exposing exactly the flattened
  `HandheldIo` port names (`io_enable`, `io_framebufferX`, `io_sdram_address`, …). The Chisel
  side is then the ~10-line shim above and nothing else.
- **Adapter in Chisel.** BlackBox just the MiSTer `emu` core, and write the bridges in
  `HandheldArcade.scala`. This is what `HandheldGameboy` / `HandheldGba` effectively do for
  the native cores, so those files are close templates.

Either way, the BlackBox is a mechanical port declaration. Pick whichever language you'd
rather debug the glue in. The rest of this document describes the glue itself.

## 7. Clocking bridge

The inner module gets exactly one clock (`clk_sys` at `clockSystemHz`). A MiSTer core
expects its `pll` to hand it several. Options, in order of preference:

1. **Clock enables only.** If the core can run off one fast clock with internal
   clock-enable strobes — the way `HandheldGameboy` drives the GB core with
   `clockConfig.enable` — this is cleanest. Pick `clockSystemHz` at/above the core's fastest
   clock and synthesize slower domains as enables. Many arcade cores already structure their
   internals this way (`ce_pix`, `ce_cpu`, …).
2. **MMCM in the shim.** Instantiate a Xilinx `MMCME2`/`PLLE2` BlackBox inside the inner
   module and derive the extra clocks there. Anything genuinely in another domain then needs
   real CDC — see the `xilinx.XpmCdc*` helpers, or do it in SV.
3. **Extend the platform.** Add a clock-wizard output in `top.sv`, thread a new
   `Input(Clock())` through `HandheldIo` and `HandheldTop`. Most invasive; last resort.

Whichever you choose: gate the core's run-enable with `io.enable`, and feed `io.reset` into
the core's reset. Board-level reset, `pll_locked`, and FPGA power sequencing are already
handled by the platform — the core does not see them.

## 8. Audio bridge

`io.audioLeft` / `io.audioRight` are `SInt(16.W)` **held values**. `HandheldTop` crosses
them into the AV clock domain with a continuous handshake and feeds both the I2S DAC
(~48 kHz) and the HDMI audio path. Connect the core's signed 16-bit output directly:

```scala
io.audioLeft  := core.io.AUDIO_L.asSInt
io.audioRight := core.io.AUDIO_R.asSInt
```

`CLK_AUDIO`, `AUDIO_S`, `AUDIO_MIX` have no platform equivalent — drop them. Note the AV
side **point-samples** the held value; it is not a real resampler. For most arcade audio
that's fine; if the core emits a high-rate stream that aliases audibly, add a simple
low-pass/decimation in the bridge. Match amplitude to roughly fill 16 bits (the GB/GBA
modules shift their APU output left by 6).

## 9. Video bridge

A MiSTer core emits a free-running raster: `VGA_R/G/B`, `VGA_HS/VS`, `VGA_HBLANK/VBLANK` (or
`VGA_DE`), and a `CE_PIXEL` strobe. Game Bub wants pixel writes addressed by
`(framebufferX, framebufferY)`. Write a small state machine — `HandheldGameboy` and
`HandheldGba` each contain one to adapt almost verbatim:

- On `CE_PIXEL`, outside blanking: set `framebufferWriteEnable`, drive `framebufferData`
  (convert the core's RGB to RGB555 — drop low bits), increment `framebufferX`.
- On the start of a new line (`HBLANK` edge): `framebufferX := 0`, increment `framebufferY`.
- On `VBLANK`: reset both to 0 and pulse `io.vblank`. The **rising edge** of `io.vblank` is
  what tells `TripleBufferControl` a frame is complete — emit it exactly once per frame,
  only after a full frame has been written.

`VGA_SL` (scanlines), `VGA_F1`, `VGA_SCALER`, and the `FB_*` framebuffer-mode signals have
no equivalent: the platform owns scaling and output timing. Drop them. Resolution and
orientation constraints are covered in §13 — check those before writing this bridge, because
they decide whether you crop, downscale, or modify `HandheldTop`.

## 10. Input bridge

There is no OSD and no keyboard. Map the 12-button `HandheldButtons` bundle onto the core's
`joystick_0` (and `joystick_1`, coin, start) bitfields in the bridge. Pick a fixed,
sensible mapping — Coin is commonly Select, Start is Start, the action buttons map to
`a/b/x/y`. The bundle is already synchronized and de-inverted by `HandheldTop`.

## 11. ROM load, config, and RTC — the HPS bridge

This is the largest bridge. A MiSTer core gets its ROM and its settings from `hps_io`:
`ioctl_download` for the ROM stream, `status` bits for OSD options/DIPs, plus `RTC` /
`TIMESTAMP`. Game Bub has no HPS — the MCU plays that role, over the QSPI link, through
`io.mcuInterface`.

### 11.1 Structure `mcuInterface`

Split the MCU window with `MemoryMap` (prefix-matched on the top address bits), exactly like
`HandheldGameboy` does:

```scala
io.mcuInterface <> MemoryMap(addressWidth = 24, dataWidth = 32, entries = Seq(
  "b0000".U(4.W) -> registerInterface,   // config / DIP / RTC registers (RegisterMap)
  "b0001".U(4.W) -> downloadInterface,   // ioctl replay window
))
```

### 11.2 ROM: assemble offline, replay as `ioctl`

MiSTer assembles ROMs at runtime from an **MRA** file (which describes how to
interleave/byte-swap MAME ROM parts) and streams the result over `ioctl_download`. Game Bub
has no MRA engine.

- **Assemble offline.** Use MAME plus an `mra` tool on a PC to produce a single flat `.rom`
  blob — exactly the byte stream the core would have received over `ioctl`. Ship that blob
  on the SD card.
- **Replay it.** The `downloadInterface` is a state machine that turns each 32-bit MCU write
  into the core's `ioctl_wr` / `ioctl_addr` / `ioctl_dout` (with a fixed or register-selected
  `ioctl_index`), holding `ioctl_download` high for the duration. The MCU writes the blob
  sequentially while the core is held in reset.

For ROMs that exceed block RAM, the alternative is to stream the blob into **SDRAM** and
service the core's ROM reads from `io.sdram` — but only if the core reads ROM from an
SDRAM-like interface rather than expecting everything pre-loaded. Many arcade cores expect
the latter, which is why the `ioctl` replay window is usually the right call.

### 11.3 Config / DIP switches → registers

Every MiSTer `status` bit — DIP switches, region, freeplay, anything that lived in the OSD —
becomes a field in a config register, written by the MCU. `RegisterMap` gives combinational
reads and single-cycle writes (`fpga/src/main/scala/lib/mem/RegisterMap.scala`):

```scala
val configRegDips = RegInit(0.U.asTypeOf(new Bundle { /* ... */ }))
registerInterface <> RegisterMap(addressWidth = 16, dataWidth = 32, entries = Seq(
  0x0000 -> RegisterMap.Entry.rw(configRegDips),
))
```

Wire those register fields to the core's `status`/DIP inputs.

### 11.4 RTC

If the core has an RTC (`RTC`/`TIMESTAMP`), expose it as config registers and let the MCU's
RTC driver populate them — the firmware already does exactly this for the GB MBC3 and the
GBA cores (`makeRtcAccess` in `HandheldGameboy.scala` / `HandheldGba.scala`).

## 12. Memory bridge

- **The core's `sdram` module.** MiSTer arcade cores usually bundle their own SDRAM
  controller and expose `SDRAM_*` pins. Remove that controller and retarget the core's
  internal memory client to `io.sdram` — a `MemoryInterface` (§4.3) to the platform's
  shared, arbitrated `SdramController`. ~32 MiB, 32-bit logical words, in the `clk_sys`
  domain.
- **`DDRAM_*`.** There is no DDR on this board. Retarget to `io.sdram`, or to block RAM if
  it's small enough.
- **`io.sram`.** 512 KiB async SRAM. Natural home for the core's NVRAM / battery-backed
  high-score RAM — route it here and reuse the firmware's `persist_save` path so saves land
  in a `.sav` file. `HandheldGba`'s EWRAM-vs-emucart SRAM arbiter shows how to share the
  512 KiB between two clients.
- **Bandwidth.** `io.sdram` is a single port shared (arbitrated) between the core, ROM
  access, and the MCU. Budget accordingly.

## 13. What determines how hard the connectivity will be

Triage the core against these before committing — they decide whether each bridge is
trivial or a project:

- **Resolution & orientation (video bridge).** The panel is physically rotated 90°.
  `HandheldTop` integer-scales the framebuffer by a **fixed 2×** for the internal display
  (480×320 active) and **3×** for HDMI, then centers it. At 2×, a core's
  `framebufferW × framebufferH` must fit within **240×160** to display unmodified — exactly
  the GBA's size. Vertical arcade games (e.g. 224×256) and anything larger need either a
  `HandheldTop` change to support a per-module scale factor, or in-core downscaling. This is
  the most common blocker.
- **Block RAM (memory bridge).** The `XC7A100T` has ~4.8 Mbit. `HandheldTop` already spends
  three framebuffers plus an overlay buffer. Cores that keep large tile/sprite ROMs in block
  RAM may force you to relocate those to SDRAM.
- **Clock count (clocking bridge).** Cores that run off clock enables port easily; cores
  that genuinely need multiple async domains mean MMCM + real CDC.
- **ROM size (ROM bridge).** Total assembled blob vs ~32 MiB SDRAM; and which `ioctl_index`
  values the core expects.
- **Licensing.** `fpga/` is `GPL-3.0-only`. Most MiSTer cores are compatible, but confirm
  per-core, preserve attribution and headers, and reject anything with non-commercial or
  no-derivatives clauses.

## 14. Build integration

In `fpga/gameboy.core`:

- **Sources.** Drop the core + adapter SV under `fpga/verilog/arcade/<corename>/` and add an
  `arcade` fileset (`{ file_type: systemVerilogSource }`; mark `.vh` includes
  `{ is_include_file: true }`). If the core needs its own clock wizard, add a
  `handheld_clk_arcade` fileset with a `clk_wiz_system_arcade.v` modeled on
  `clk_wiz_system_gameboy.v`.
- **Flag & targets.** Add `arcade` (and `handheld_clk_arcade`) to the `_handheld` target's
  filesets guarded by `module_arcade ? (...)`, and add
  `module_arcade ? (platform.handheld.HandheldArcade)` to the `handheld` generator's
  `extraargs` — mirroring `module_gameboy` / `module_gba`.

Build:

```sh
fusesoc --cores-root . --work-root=build/arcade --target=handheld_rev2 --flag=module_arcade elipsitz:gameboy:gameboy
gzip -c build/arcade/.../top_handheld.bit > arcade.bit.gz   # → SD card: system/arcade.bit.gz
```

## 15. Firmware side

The MCU side mirrors the FPGA side. Each core type has a *bitstream driver* under
`firmware/handheld/src/bitstream/` — see `gameboy.rs` and `gba/` for full examples.

### 15.1 The driver

Create `firmware/handheld/src/bitstream/arcade.rs` implementing the `Bitstream` trait
(`firmware/handheld/src/bitstream/mod.rs`):

```rust
pub trait Bitstream {
    fn get_bitstream_path(&self) -> &'static str;   // e.g. "arcade.bit.gz"
    fn on_after_program(&mut self) -> Result<(), String>;
    fn set_paused(&mut self, paused: bool) -> Result<(), fpga::Error>;
    fn reset(&mut self) -> Result<(), fpga::Error>;
    fn on_vblank_irq(&mut self);
}
```

`on_after_program` should call `fpga.set_system_clock_rate(...)` with this core's
`clockSystemHz` (firmware uses it to pick a safe QSPI clock). Add inherent methods for the
ROM/save lifecycle mirroring `Gameboy::set_emulated_cartridge` / `persist_ram` /
`needs_save_persist`. In `set_emulated_cartridge`:

1. Hold the core in reset: `write_u32(REG_CONTROL, 0b0000)`.
2. Stream the assembled `.rom` blob into the download window of the module space
   (`0xC0_xx_xx_xx`, the `mcuInterface` region) — or into SDRAM via `fpga.sdram_write` —
   in chunks, sending `ui::Message::RomLoadingProgress`.
3. Write config / DIP registers.
4. Optionally load NVRAM into SRAM via `fpga.sram_write`.
5. Release: `write_u32(REG_CONTROL, 0b1011)` (reset released, running).

The MCU↔FPGA address regions (register space `0x0…`, SRAM `0x1…`, SDRAM `0x2…`, overlay
`0x38…`, module/core space `0xC…`) and the QSPI command format are in
`firmware/handheld/src/device/drivers/fpga.rs`.

### 15.2 Wire it in

The trait is minimal; the worker and game-state code match on the `CurrentBitstream` enum
directly, so adding a core touches several `match` arms:

- **`src/bitstream/mod.rs`** — add an `Arcade(arcade::Arcade)` variant to `CurrentBitstream`,
  update `get()`, add an `ensure_arcade()` constructor.
- **`src/worker/mod.rs`** — add the file extension(s) to the `rom_select_get_files` filter
  list **and** the `RunRomFile` dispatch `match`; add arms in `RunCartridge` / `SaveGame`.
- **`src/ui/state/game.rs`** — add arms in `on_game_set_paused` and `on_game_reset`.
- The Slint UI is generic — no `.slint` changes unless you add core-specific settings. If
  you do, add KVS keys in `src/kvs/keys.rs` and entries under `src/ui/state/settings.rs` +
  `res/ui/screens/settings/`.

> Worthwhile refactor while you're here: lift `set_emulated_cartridge` / `persist_save` /
> `needs_save_persist` into the `Bitstream` trait so these `match` blocks collapse to trait
> calls. They're currently duplicated per variant.

### 15.3 SD card layout

Firmware resolves system files from `/sdcard/system_rev{1,2}/` then `/sdcard/system/`:

```
system/
  arcade.bit.gz        # gzip'd module_arcade bitstream
roms/
  <game>.rom           # offline-assembled ROM blob (the ioctl byte stream)
```

`partitions.csv`, `flash_nvs.py`, and `sdkconfig.defaults` need no changes — bitstreams and
ROMs live on the SD card, not in ESP32 flash.

## 16. Simulator support (optional but recommended)

The Verilator simulator is the fastest debug loop. Add a `platform.sim.SimArcade` top-level
mirroring `SimGameboy` / `SimGba`, `sim_arcade` filesets + generator in `gameboy.core` wired
into the `sim` target, and C++ glue under `fpga/sim/arcade/` following `fpga/sim/gb/` and
`fpga/sim/gba/`. For Verilator the BlackBox needs the `.sv` available — rely on the `arcade`
fileset being in the `sim` target, or use `HasBlackBoxResource` / `addResource`.

```sh
fusesoc --cores-root . run --target=sim --flag=module_arcade elipsitz:gameboy:gameboy
./build/.../sim-verilator/VSimArcade --rom <assembled.rom>
```

## 17. Gotchas

- **Fixed 2× display scale.** Until `HandheldTop` learns a per-module scale, the effective
  max framebuffer for the internal display is 240×160. The single most common reason a core
  won't "just work."
- **One core clock.** Multi-domain cores need real CDC or a clock-enable rewrite — don't
  paper over async crossings.
- **Block RAM is shared.** Three framebuffers + overlay are already spent.
- **`io.sdram` is one arbitrated port** shared by core, ROM access, and MCU. Budget the
  bandwidth.
- **Audio is point-sampled**, not resampled. Filter in the bridge if it aliases.
- **No OSD / `conf_str`.** Every option becomes a config register + firmware (+ optional
  Slint settings UI).
- **No MRA at runtime.** ROMs must be assembled offline into the flat `ioctl` byte stream.
- **Licensing.** Per-core check; `fpga/` is `GPL-3.0-only`.

## 18. Checklist

- [ ] Triaged: resolution/orientation, block RAM, clock count, ROM size, license (§13)
- [ ] Core + adapter SV added as an `arcade` fileset; `module_arcade` flag + targets +
      generator wired up (§5, §14)
- [ ] `HandheldArcade` Chisel shim BlackBox-instantiating the SV inner module (§6)
- [ ] Clocking bridge: `clk_sys` → core's clock domains (§7)
- [ ] Audio bridge: `AUDIO_L/R` → `io.audioLeft/Right` (§8)
- [ ] Video bridge: raster → framebuffer writes + `vblank` (§9)
- [ ] Input bridge: `HandheldButtons` → core joystick/coin/start (§10)
- [ ] ROM bridge: `ioctl` replay window in `mcuInterface` (or SDRAM path) (§11)
- [ ] Config/DIP/RTC registers via `RegisterMap` (§11)
- [ ] Memory bridge: core memory clients → `io.sdram`; NVRAM → `io.sram` (§12)
- [ ] Offline ROM-assembly pipeline producing the `.rom` blob (§11)
- [ ] `arcade.rs` bitstream driver + `CurrentBitstream` variant + worker/game `match` arms
      + file extension registered (§15)
- [ ] `arcade.bit.gz` + ROMs on the SD card (§15.3)
- [ ] (Optional) `SimArcade` + sim filesets + C++ glue (§16)
- [ ] Verified in simulator, then on hardware
