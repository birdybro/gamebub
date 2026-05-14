# Worked Example: Porting the Joust 2 MiSTer Core

This is a concrete, end-to-end walkthrough of porting one real core —
[Arcade-Joust2_MiSTer](https://github.com/MiSTer-devel/Arcade-Joust2_MiSTer) — onto Game Bub.
Read [porting-mister-cores.md](porting-mister-cores.md) first; this document is the applied
version of that guide, with every integration file shown inline.

> **Honesty up front.** The code below was written against the published Joust 2 sources and
> the Game Bub interfaces as they exist in this repo, but it has **not** been synthesized or
> hardware-tested in preparing this document. Treat it as a concrete, accurate-in-shape
> starting point, not a known-good drop-in. The Joust 2 `rtl/` sources are GPL and are *not*
> vendored into this repo — you copy them in yourself (§3). Places that need verification
> against the real RTL or against synthesis results are called out explicitly.

---

## 1. Survey the core

Joust 2 runs on Williams 2 hardware. The MiSTer repo's top file is `Arcade-Joust2.sv` (the
`emu` module) — that's the part we *replace*. The actual hardware is `rtl/williams2.vhd`,
and its VHDL entity is the real contract we port against:

```vhdl
entity williams2 is port(
  clock_12             : in  std_logic;
  reset                : in  std_logic;
  pause_cpu            : in  std_logic;
  rom_addr             : out std_logic_vector(16 downto 0);   -- unused in this port (see below)
  rom_do               : in  std_logic_vector( 7 downto 0);
  rom_rd               : out std_logic;
  dn_addr              : in  std_logic_vector(18 downto 0);   -- ROM download bus
  dn_data              : in  std_logic_vector( 7 downto 0);
  dn_wr                : in  std_logic;
  dn_index             : in  std_logic_vector( 7 downto 0);
  video_r              : out std_logic_vector( 3 downto 0);
  video_g              : out std_logic_vector( 3 downto 0);
  video_b              : out std_logic_vector( 3 downto 0);
  video_i              : out std_logic_vector( 3 downto 0);   -- intensity
  video_hblank         : out std_logic;
  video_vblank         : out std_logic;
  video_hs             : out std_logic;
  video_vs             : out std_logic;
  audio_l              : out std_logic_vector(13 downto 0);   -- 14-bit unsigned
  audio_r              : out std_logic_vector(13 downto 0);
  btn_auto_up          : in  std_logic;
  btn_advance          : in  std_logic;
  btn_high_score_reset : in  std_logic;
  btn_coin             : in  std_logic;
  btn_start_1          : in  std_logic;
  btn_start_2          : in  std_logic;
  btn_left_1           : in  std_logic;
  btn_right_1          : in  std_logic;
  btn_trigger1_1       : in  std_logic;   -- "flap"
  btn_left_2           : in  std_logic;
  btn_right_2          : in  std_logic;
  btn_trigger1_2       : in  std_logic;
  sw_cocktail_table    : in  std_logic;
  seven_seg            : out std_logic_vector( 7 downto 0);
  nvram_addr           : in  std_logic_vector( 9 downto 0);   -- hiscore CMOS read-back
  nvram_data_out       : out std_logic_vector( 3 downto 0);
  dbg_out              : out std_logic_vector(31 downto 0)
);
```

What this tells us, mapped onto the [connectivity map](porting-mister-cores.md#3-the-connectivity-map):

| Joust 2 / `williams2` | Game Bub | Notes |
|---|---|---|
| `clock_12` — single 12 MHz clock, the whole core | `clk_sys` at 12 MHz | Best case: no extra clocks, no CDC. §5, §6 |
| `reset`, `pause_cpu` | `io.reset`, `!io.enable` | Pause via `pause_cpu`, not by gating the clock. §6 |
| `dn_addr/dn_data/dn_wr/dn_index` | download window in `io.mcuInterface` | `dn_index` 0 = ROM, 4 = NVRAM restore. §6, §7 |
| `rom_addr/rom_do/rom_rd` | **unused** | This MiSTer port keeps all ROMs internal to `williams2`; the wrapper leaves these dangling. Tie `rom_do := 0`. |
| `video_r/g/b/i` (4-bit each), `video_hblank/vblank/hs/vs` | framebuffer write port | 384×260 raster, ~292×240 visible. Needs platform changes — §4. |
| `audio_l/audio_r` (14-bit unsigned) | `io.audioLeft/Right` (`SInt(16)`) | Center + scale. §6 |
| `btn_*` | `io.buttons` | One-player mapping; P2 tied off. §6 |
| `btn_auto_up/advance/high_score_reset`, `sw_cocktail_table` | config register | These were MiSTer `status[]` / DIP bits. §6 |
| `nvram_addr/nvram_data_out` | optional save path | 1024×4-bit hiscore CMOS. §7 |
| `seven_seg`, `dbg_out` | unused | Leave open. |
| SDRAM / DDRAM | **unused** | Joust 2 needs neither — nothing to bridge. |

Two facts make Joust 2 a comparatively friendly port: **one clock**, and **no external
memory**. The one hard part is video resolution.

### The resolution problem

`williams2` produces a **384×260** total raster (comment in the VHDL: "64 hcnt × 6 = 384
pixels … 1 line is 64 µs / 15.625 kHz"), of which roughly **292×240** is visible. Two
platform limits collide with this:

1. `HandheldTop` integer-scales the framebuffer by a **fixed 2×** for the internal panel,
   capping the usable framebuffer at 240×160.
2. `HandheldIo.framebufferX/Y` are `UInt(8.W)` — they can't even *address* a 292-wide image.

Per the decision for this example, we **extend the platform** (§4): widen the framebuffer
coordinates and make the display scale per-core, so Joust 2 can render its native ~292×240
at 1× centered on the 480×320 panel.

Also note: `williams2` exposes no pixel-clock-enable. Its pixels advance at ~6 MHz (every
2nd `clock_12`), but there's no `video_ce` output. The cleanest fix is a **one-line edit to
the vendored `williams2.vhd`**: bring its internal pixel strobe out as a `video_ce` port.
Making small surgical edits to a vendored core is normal and expected. §6 assumes you've
done this; it also describes the zero-edit fallback.

---

## 2. Files we touch

Nothing in this example is committed — it's a walkthrough. But a real port would create /
edit exactly these:

```
fpga/verilog/arcade/joust2/        (NEW)  vendored Joust 2 rtl/ sources + clock wizard
fpga/gameboy.core                  (EDIT) new filesets, module_arcade flag, generator
fpga/src/main/scala/platform/handheld/HandheldTop.scala   (EDIT) widen coords, per-core scale
fpga/src/main/scala/platform/handheld/HandheldArcade.scala (NEW) the inner module / adapter
firmware/handheld/src/bitstream/arcade.rs                 (NEW) the bitstream driver
firmware/handheld/src/bitstream/mod.rs                    (EDIT) CurrentBitstream variant
firmware/handheld/src/worker/mod.rs                       (EDIT) file extension + dispatch
firmware/handheld/src/ui/state/game.rs                    (EDIT) pause/reset match arms
```

---

## 3. Vendoring the Joust 2 sources

Copy the core's hardware sources (not `sys/`, not the Quartus project files) into
`fpga/verilog/arcade/joust2/`. From the MiSTer repo's `rtl/`:

- **VHDL** — `williams2.vhd`, `cpu09l_128.vhd`, `cpu68_2.vhd`, `pia6821.vhd`,
  `joust2_sound_board.vhd`, `williams_cvsd_board.vhd`, `hc55564.vhd`, `joust2_cmos_ram.vhd`,
  `decodeur_7_seg.vhd`, `dpram.vhd`, `gen_ram.vhd`
- **Verilog** — everything under `jt51/` and `mc6809/`
- **Drop** — `pll.v` / `pll.qip` (we make our own clock wizard, §5), `nvram.v` and `pause.v`
  (MiSTer hiscore/OSD framework glue — we handle pause and NVRAM ourselves), and the whole
  `sys/` folder.
- **Preserve** the GPL license headers, and add the upstream repo + commit hash to a
  `fpga/verilog/arcade/joust2/README` for attribution.

> **Color LUT:** `Arcade-Joust2.sv` contains a 256-entry color lookup table indexed by
> `{video_color[3:0], video_i[3:0]}` → 8-bit. Copy that table's contents — we reuse it in
> the adapter (§6) so colors match the original.

Vivado synthesizes mixed VHDL/Verilog without issue. FuseSoC just needs each file tagged
with the right `file_type` (`vhdlSource` vs `verilogSource`) — §4.

---

## 4. Platform changes: extend the framebuffer

Three small edits let any core declare a native resolution larger than 240×160 and its own
integer scale. Game Boy / GBA are unaffected because the new knobs default to today's
behavior.

### 4a. `HandheldModule` trait — add per-core scale

In `fpga/src/main/scala/platform/handheld/HandheldTop.scala`:

```scala
trait HandheldModule {
  def io: HandheldIo
  def framebufferW: Int
  def framebufferH: Int
  def clockSystemHz: Int
  def clockSdramHz: Int

  /** Integer scale factor for the internal 480x320 panel. */
  def displayScaleInternal: Int = 2   // default: today's behavior
  /** Integer scale factor for 720x480 HDMI output. */
  def displayScaleHdmi: Int = 3       // default: today's behavior
}
```

### 4b. `HandheldIo` — widen the framebuffer coordinates

```scala
class HandheldIo extends Bundle {
  // ...
  // Video output  (was UInt(8.W) — too narrow for >256-wide cores)
  val framebufferX = Output(UInt(10.W))
  val framebufferY = Output(UInt(10.W))
  val framebufferData = Output(ColorARGB.rgb555())
  val framebufferWriteEnable = Output(Bool())
  val vblank = Output(Bool())
  // ...
}
```

### 4c. `HandheldTop` — use the per-core scale, widen the write address

In `HandheldTop`, the scale factors are currently hardcoded `val videoScale = 2` (internal)
and `= 3` (HDMI). Replace both with the module's values:

```scala
// internal-display branch:
val videoScale = module.displayScaleInternal
// HDMI branch:
val videoScale = module.displayScaleHdmi
```

The centering math (`videoOffsetX = (screenWidth - (videoWidth * videoScale)) / 2`, etc.)
already derives from `videoScale` and `module.framebufferW/H`, so it follows automatically.
For Joust 2 at 1×: internal offset `(480-292)/2 = 94` x, `(320-240)/2 = 40` y — centered with
a border, which is fine.

Finally, the framebuffer write-address multiply assumes 8-bit coords:

```scala
// was: (module.io.framebufferY * videoWidth.U(8.W)) + module.io.framebufferX
val address = (module.io.framebufferY * videoWidth.U(10.W)) + module.io.framebufferX
```

`framebufferInterface` is already `addressWidth = 18` and the framebuffer `SRAM`s are sized
`videoWidth * videoHeight`, so no other changes are needed there.

### 4d. Block RAM budget — check after synthesis

This is the real risk. The `XC7A100T` has ~4.86 Mbit of block RAM. `HandheldTop` spends
**three** framebuffers plus the overlay buffer:

```
3 × (292 × 240 × 15 bits)  ≈ 3.15 Mbit
overlay 240 × 160 × 16 bits ≈ 0.61 Mbit
                              --------
                              ≈ 3.76 Mbit
```

…leaving ~1.1 Mbit for *all* of `williams2`'s internal ROMs and RAMs (program ROM, graphics
ROMs, sound ROMs, work RAM). Williams 2 ROM sets are on the order of a few hundred KB — this
**may not fit**. If synthesis overflows BRAM, options in order of preference:

- Move `williams2`'s large graphics ROMs out to SDRAM via `io.sdram` (Joust 2 doesn't
  otherwise use it) — more work, requires editing the VHDL ROM blocks.
- Drop arcade cores to double-buffering instead of triple. `TripleBufferControl` and the
  read path in `HandheldTop` would need a mode for this; the architecture notes trade-offs.
- Shrink the framebuffer to the true visible window (measure it — see §6).

Measure first; don't pre-optimize.

---

## 5. The clock wizard and build wiring

### Clock wizard

`williams2` wants a single 12 MHz clock. Add a Vivado clocking wizard IP that takes the
board's 50 MHz input and produces 12 MHz, as `fpga/verilog/arcade/joust2/clk_wiz_system_arcade.v`.
The simplest path is to copy `fpga/verilog/handheld/clk_wiz_system_gameboy.v`, open it in
Vivado, and retarget the output frequency — the existing GB/GBA wizards are your template
for the exact module/port shape `top.sv` expects.

`clockSystemHz` for this core is therefore `12_000_000`. SDRAM is unused, but
`HandheldTop` still instantiates the SDRAM controller, so keep `clockSdramHz` a sane integer
multiple (`clockSystemHz * 4`) as the other cores do.

### `gameboy.core`

FuseSoC selects bitstreams by a `module_*` flag. Mirror the existing `module_gameboy` /
`module_gba` wiring for `module_arcade`. Add the filesets:

```yaml
  arcade:
    files:
      # VHDL core sources (compile order usually resolves automatically; williams2 last)
      - verilog/arcade/joust2/dpram.vhd               : { file_type: vhdlSource }
      - verilog/arcade/joust2/gen_ram.vhd             : { file_type: vhdlSource }
      - verilog/arcade/joust2/pia6821.vhd             : { file_type: vhdlSource }
      - verilog/arcade/joust2/cpu09l_128.vhd          : { file_type: vhdlSource }
      - verilog/arcade/joust2/cpu68_2.vhd             : { file_type: vhdlSource }
      - verilog/arcade/joust2/hc55564.vhd             : { file_type: vhdlSource }
      - verilog/arcade/joust2/decodeur_7_seg.vhd      : { file_type: vhdlSource }
      - verilog/arcade/joust2/joust2_cmos_ram.vhd     : { file_type: vhdlSource }
      - verilog/arcade/joust2/williams_cvsd_board.vhd : { file_type: vhdlSource }
      - verilog/arcade/joust2/joust2_sound_board.vhd  : { file_type: vhdlSource }
      - verilog/arcade/joust2/williams2.vhd           : { file_type: vhdlSource }
      # Verilog submodules (jt51/, mc6809/) — list each .v
      - verilog/arcade/joust2/jt51/jt51.v             : { file_type: verilogSource }
      # ... remaining jt51/*.v and mc6809/*.v ...

  handheld_clk_arcade:
    files:
      - verilog/arcade/joust2/clk_wiz_system_arcade.v : { file_type: verilogSource }
```

Add `module_arcade` to the `_handheld` target's filesets (mirroring `module_gameboy`):

```yaml
  _handheld: &handheld
    filesets:
      - "base"
      - "hdmi"
      - "handheld"
      - "module_gba ? (handheld_clk_gba)"
      - "module_gameboy ? (handheld_clk_gameboy)"
      - "module_boot ? (handheld_clk_gameboy)"
      - "module_arcade ? (handheld_clk_arcade)"
      - "module_arcade ? (arcade)"
```

And add the inner-module class to the `handheld` generator's `extraargs`:

```yaml
        extraargs: >
          module_boot ? (platform.handheld.HandheldBoot)
          module_gba ? (platform.handheld.HandheldGba)
          module_gameboy ? (platform.handheld.HandheldGameboy)
          module_arcade ? (platform.handheld.HandheldArcade)
```

> If Vivado complains about VHDL elaboration order, give the VHDL files an explicit
> `logical_name` or reorder them so dependencies compile first (`williams2.vhd` last).

---

## 6. The inner module: `HandheldArcade.scala`

This is the adapter. `williams2` is wrapped as a Chisel `BlackBox` (the VHDL is compiled by
Vivado/FuseSoC and linked by module name); everything else is the bridge logic.

`fpga/src/main/scala/platform/handheld/HandheldArcade.scala`:

```scala
package platform.handheld

import chisel3._
import chisel3.util._
import lib.mem.{MemoryInterface, MemoryMap, RegisterMap}
import lib.video.ColorARGB

/** BlackBox for the vendored Williams 2 VHDL core. Port names match williams2.vhd. */
class Williams2 extends BlackBox {
  val io = IO(new Bundle {
    val clock_12  = Input(Clock())
    val reset     = Input(Bool())
    val pause_cpu = Input(Bool())

    // rom_addr/rom_do/rom_rd are unused in this MiSTer port; tie rom_do low.
    val rom_addr  = Output(UInt(17.W))
    val rom_do    = Input(UInt(8.W))
    val rom_rd    = Output(Bool())

    // ROM/NVRAM download bus
    val dn_addr   = Input(UInt(19.W))
    val dn_data   = Input(UInt(8.W))
    val dn_wr     = Input(Bool())
    val dn_index  = Input(UInt(8.W))

    // Video (4-bit RGB + 4-bit intensity, blanking/sync)
    val video_r      = Output(UInt(4.W))
    val video_g      = Output(UInt(4.W))
    val video_b      = Output(UInt(4.W))
    val video_i      = Output(UInt(4.W))
    val video_hblank = Output(Bool())
    val video_vblank = Output(Bool())
    val video_hs     = Output(Bool())
    val video_vs     = Output(Bool())
    // video_ce: one-line addition to williams2.vhd exposing the internal pixel strobe.
    val video_ce     = Output(Bool())

    // Audio (14-bit unsigned)
    val audio_l = Output(UInt(14.W))
    val audio_r = Output(UInt(14.W))

    // Inputs
    val btn_auto_up          = Input(Bool())
    val btn_advance          = Input(Bool())
    val btn_high_score_reset = Input(Bool())
    val btn_coin             = Input(Bool())
    val btn_start_1          = Input(Bool())
    val btn_start_2          = Input(Bool())
    val btn_left_1           = Input(Bool())
    val btn_right_1          = Input(Bool())
    val btn_trigger1_1       = Input(Bool())
    val btn_left_2           = Input(Bool())
    val btn_right_2          = Input(Bool())
    val btn_trigger1_2       = Input(Bool())
    val sw_cocktail_table    = Input(Bool())

    // Unused outputs
    val seven_seg = Output(UInt(8.W))
    val dbg_out   = Output(UInt(32.W))

    // Hiscore CMOS read-back
    val nvram_addr     = Input(UInt(10.W))
    val nvram_data_out = Output(UInt(4.W))
  })
}

class HandheldArcade extends Module with HandheldModule {
  val io = IO(new HandheldIo)

  // Visible resolution. 292x240 is the nominal Williams 2 visible window; measure the real
  // active region from video_hblank/video_vblank in the simulator and adjust (§9).
  def framebufferW = 292
  def framebufferH = 240
  def clockSystemHz = 12 * 1000 * 1000
  def clockSdramHz  = clockSystemHz * 4

  // Joust 2 renders at native 1x, centered on the panel / HDMI output.
  override def displayScaleInternal = 1
  override def displayScaleHdmi     = 1

  val core = Module(new Williams2)
  core.io.clock_12 := clock          // implicit clock = clk_sys = 12 MHz
  core.io.reset    := io.reset
  core.io.pause_cpu := !io.enable    // pause by holding the CPU, not by gating the clock
  core.io.rom_do := 0.U              // rom_addr/rom_rd unused in this port
  core.io.sw_cocktail_table := false.B

  //////////////////////////////////////////////////////////////////////////////
  // MCU interface: config registers + ROM download window
  //////////////////////////////////////////////////////////////////////////////
  // DIP / service switches that were MiSTer status[] bits.
  val configReg = RegInit(0.U.asTypeOf(new Bundle {
    val autoUp         = Bool()   // status[11]
    val advance        = Bool()   // status[10]
    val highScoreReset = Bool()   // status[12]
  }))
  core.io.btn_auto_up          := configReg.autoUp
  core.io.btn_advance          := configReg.advance
  core.io.btn_high_score_reset := configReg.highScoreReset

  val registerInterface = Wire(new MemoryInterface(addressWidth = 16, dataWidth = 32))
  val downloadInterface = Wire(new MemoryInterface(addressWidth = 21, dataWidth = 32))
  io.mcuInterface <> MemoryMap(addressWidth = 24, dataWidth = 32, entries = Seq(
    "b0000".U(4.W) -> registerInterface,
    "b0001".U(4.W) -> downloadInterface,
  ))
  registerInterface <> RegisterMap(addressWidth = 16, dataWidth = 32, entries = Seq(
    0x0000 -> RegisterMap.Entry.rw(configReg),
  ))

  // Download replay: each 32-bit MCU write is one ROM byte. The MCU writes the assembled
  // .rom blob sequentially (dn_index 0), then the NVRAM blob (dn_index 4), while the core
  // is held in reset. Address auto-increments on the FPGA side.
  val dnAddr  = RegInit(0.U(19.W))
  val dnIndex = RegInit(0.U(8.W))
  core.io.dn_addr  := dnAddr
  core.io.dn_data  := downloadInterface.dataWrite(7, 0)
  core.io.dn_index := dnIndex
  core.io.dn_wr    := downloadInterface.enable && downloadInterface.write
  downloadInterface.dataRead := 0.U
  downloadInterface.done := RegNext(downloadInterface.enable)
  when (downloadInterface.enable && downloadInterface.write) {
    // address 0 in this window = "reset stream"; bit[20] selects the dn_index (0 vs 4).
    when (downloadInterface.address === 0.U) {
      dnAddr  := 0.U
      dnIndex := Mux(downloadInterface.dataWrite(0), 4.U, 0.U)
    } .otherwise {
      dnAddr := dnAddr + 1.U
    }
  }

  //////////////////////////////////////////////////////////////////////////////
  // Video: raster -> framebuffer writes
  //////////////////////////////////////////////////////////////////////////////
  // Color LUT copied from Arcade-Joust2.sv: index {color[3:0], intensity[3:0]} -> 8-bit.
  // Take the high 5 bits for rgb555. (Fill in `joust2ColorLut` with the table contents.)
  val colorLut = VecInit(joust2ColorLut.map(_.U(8.W)))   // 256 entries
  def shade(c: UInt): UInt = colorLut(Cat(c, core.io.video_i))(7, 3)

  val fbX = RegInit(0.U(10.W))
  val fbY = RegInit(0.U(10.W))
  val prevHblank = RegNext(core.io.video_hblank)
  val prevVblank = RegNext(core.io.video_vblank)

  io.framebufferWriteEnable := false.B
  io.framebufferData := ColorARGB.rgb555().makeBlack()
  io.framebufferX := fbX
  io.framebufferY := fbY
  io.vblank := core.io.video_vblank

  when (core.io.video_ce) {
    when (core.io.video_vblank) {
      // Frame done. fbX/fbY reset for the next frame; the rising edge of io.vblank
      // (driven directly from video_vblank) latches the triple buffer.
      fbX := 0.U
      fbY := 0.U
    } .elsewhen (core.io.video_hblank && !prevHblank) {
      // New line.
      fbX := 0.U
      fbY := fbY + 1.U
    } .elsewhen (!core.io.video_hblank) {
      // Active pixel.
      io.framebufferWriteEnable := true.B
      io.framebufferData.r := shade(core.io.video_r)
      io.framebufferData.g := shade(core.io.video_g)
      io.framebufferData.b := shade(core.io.video_b)
      fbX := fbX + 1.U
    }
  }

  //////////////////////////////////////////////////////////////////////////////
  // Audio: 14-bit unsigned -> 16-bit signed
  //////////////////////////////////////////////////////////////////////////////
  // center (subtract 0x2000) and shift left 2 to fill the 16-bit range.
  io.audioLeft  := ((core.io.audio_l ## 0.U(2.W)).zext - 0x8000.S).asSInt(15, 0).asSInt
  io.audioRight := ((core.io.audio_r ## 0.U(2.W)).zext - 0x8000.S).asSInt(15, 0).asSInt

  //////////////////////////////////////////////////////////////////////////////
  // Input mapping (player 1; player 2 tied off — see §9 for link-cable 2P)
  //////////////////////////////////////////////////////////////////////////////
  core.io.btn_left_1     := io.buttons.left
  core.io.btn_right_1    := io.buttons.right
  core.io.btn_trigger1_1 := io.buttons.a       // "flap"
  core.io.btn_start_1    := io.buttons.start
  core.io.btn_coin       := io.buttons.select
  core.io.btn_left_2     := false.B
  core.io.btn_right_2    := false.B
  core.io.btn_trigger1_2 := false.B
  core.io.btn_start_2    := false.B

  //////////////////////////////////////////////////////////////////////////////
  // NVRAM read-back (optional — see §7). Drive nvram_addr from a register the MCU
  // increments; expose nvram_data_out in a status register. Omitted here for brevity.
  //////////////////////////////////////////////////////////////////////////////
  core.io.nvram_addr := 0.U

  //////////////////////////////////////////////////////////////////////////////
  // Unused platform interfaces — stub them (see HandheldBoot.scala for the pattern)
  //////////////////////////////////////////////////////////////////////////////
  io.vibrate := false.B
  io.cartridgeEnabled := false.B
  io.cartridge.bank0Out := DontCare; io.cartridge.bank0Dir := false.B
  io.cartridge.bank1Out := DontCare; io.cartridge.bank1Dir := false.B
  io.cartridge.bank2Out := DontCare; io.cartridge.bank2Dir := false.B
  io.cartridge.bank3Out := DontCare; io.cartridge.bank3Dir := false.B
  io.cartridge.pin30Out := DontCare; io.cartridge.pin30Dir := false.B
  io.cartridge.pin31Out := DontCare; io.cartridge.pin31Dir := false.B
  io.link.soOut := DontCare; io.link.siOut := DontCare
  io.link.sdOut := DontCare; io.link.scOut := DontCare
  io.link.soDir := false.B; io.link.siDir := false.B
  io.link.sdDir := false.B; io.link.scDir := false.B
  io.pmod.out := DontCare; io.pmod.dir := 0.U
  io.sram.enable := false.B; io.sram.write := false.B
  io.sram.address := DontCare; io.sram.dataWrite := DontCare; io.sram.writeStrobe := DontCare
  io.sdram.enable := false.B; io.sdram.write := false.B
  io.sdram.address := DontCare; io.sdram.dataWrite := DontCare; io.sdram.writeStrobe := DontCare
}
```

Notes on the tricky parts:

- **`video_ce`.** The adapter samples on `core.io.video_ce`, the pixel strobe you exposed
  from `williams2.vhd`. *Zero-edit fallback:* if you don't want to touch the VHDL, sample on
  every `clock` cycle instead — but `williams2` holds each pixel for 2 clocks, so you'd then
  write each pixel twice and `framebufferW` doubles to ~584, blowing the BRAM budget (§4d).
  Exposing the CE is the right call.
- **`io.vblank` timing.** It's driven straight from `video_vblank`; its rising edge tells
  `TripleBufferControl` the frame is complete. Confirm in simulation that `video_vblank`
  pulses exactly once per frame and that a full frame's pixels land before it rises.
- **Color LUT.** `joust2ColorLut` is the 256-entry table from `Arcade-Joust2.sv`. Reusing it
  keeps colors faithful; a cheap approximation (`Cat(video_r, video_r(3))` ignoring
  intensity) is a fine first-light shortcut.
- **Audio expression.** Shown for intent; check the width arithmetic against your Chisel
  version, or write it as an explicit `Wire(SInt(16.W))`.

---

## 7. The firmware driver: `arcade.rs`

`firmware/handheld/src/bitstream/arcade.rs` — a new bitstream driver. Compare against
`gameboy.rs` / `gba/mod.rs`.

```rust
use crate::device::drivers::fpga;
use crate::bitstream::Bitstream;

const SYSTEM_CLOCK_RATE: u32 = 12_000_000;

// Module-space register addresses (the 0xC... region; matches HandheldArcade's MemoryMap).
const REG_CONFIG:        u32 = 0xC000_0000;          // configReg (DIP / service)
const DOWNLOAD_WINDOW:   u32 = 0xC010_0000;          // downloadInterface base
const REG_CONTROL:       u32 = 0x0000_0000;          // platform REG_CONTROL

pub struct Arcade {
    rom_loaded: bool,
}

impl Arcade {
    pub fn new() -> Self { Arcade { rom_loaded: false } }

    /// Stream an assembled Joust 2 .rom blob into the core.
    pub fn set_rom(&mut self, path: &str) -> Result<(), String> {
        // 1. Hold the core in reset.
        fpga::write_u32(REG_CONTROL, 0b0000)?;

        // 2. Reset the download stream to dn_index 0 (ROM), then stream bytes.
        fpga::write_u32(DOWNLOAD_WINDOW + 0, 0)?;             // addr 0, data bit0=0 -> index 0
        let mut file = crate::util::open_file(path)?;
        let mut addr = DOWNLOAD_WINDOW + 4;
        let mut buf = [0u8; 4096];
        loop {
            let n = file.read(&mut buf).map_err(|e| e.to_string())?;
            if n == 0 { break; }
            for &byte in &buf[..n] {
                fpga::write_u32(addr, byte as u32)?;
                addr += 4;
                // (send ui::Message::RomLoadingProgress periodically)
            }
        }

        // 3. (Optional) restore NVRAM: open <rom>.hi, write DOWNLOAD_WINDOW+0 with data
        //    bit0=1 to select dn_index 4, then stream those bytes the same way.

        // 4. Default DIP / service switches.
        fpga::write_u32(REG_CONFIG, 0)?;

        // 5. Release reset and run.
        fpga::write_u32(REG_CONTROL, 0b1011)?;
        self.rom_loaded = true;
        Ok(())
    }
}

impl Bitstream for Arcade {
    fn get_bitstream_path(&self) -> &'static str { "arcade_joust2.bit.gz" }
    fn on_after_program(&mut self) -> Result<(), String> {
        fpga::set_system_clock_rate(SYSTEM_CLOCK_RATE);
        Ok(())
    }
    fn set_paused(&mut self, paused: bool) -> Result<(), fpga::Error> {
        fpga::write_u32(REG_CONTROL, 0b1010 | (!paused as u32))
    }
    fn reset(&mut self) -> Result<(), fpga::Error> {
        fpga::write_u32(REG_CONTROL, 0b0000)?;
        fpga::write_u32(REG_CONTROL, 0b1011)
    }
    fn on_vblank_irq(&mut self) { /* nothing per-frame for Joust 2 */ }
}
```

> `open_file`, `read`, the exact `fpga` accessor names, and `ui::Message` are sketched —
> match them to the real APIs in `firmware/handheld/src/util/`, `device/drivers/fpga.rs`,
> and `ui/`. The shape (reset → stream → configure → run) is the contract.

**NVRAM save-back** is optional for a first port. To support it: have the adapter drive
`nvram_addr` from an MCU-incremented register and expose `nvram_data_out` in a status
register, then add a `persist_save()` inherent method that reads the 1024 nibbles back into
a `<rom>.hi` file. Restore is just the `dn_index 4` stream in `set_rom`.

### Wiring it into the firmware

- **`src/bitstream/mod.rs`** — add `Arcade(arcade::Arcade)` to `CurrentBitstream`, update
  `get()`, add `ensure_arcade()` (programs `arcade_joust2.bit.gz`, constructs the driver).
- **`src/worker/mod.rs`** — add the ROM extension (e.g. `"j2"` or a generic `"arc"`) to the
  `rom_select_get_files` filter **and** the `RunRomFile` dispatch `match`; the arm calls
  `ensure_arcade()` then `arcade.set_rom(path)`.
- **`src/ui/state/game.rs`** — add `CurrentBitstream::Arcade` arms to `on_game_set_paused`
  and `on_game_reset`.
- No Slint changes needed unless you want a DIP-switch settings screen.

### SD card layout

```
system/
  arcade_joust2.bit.gz      # gzip'd module_arcade bitstream
roms/
  joust2.j2                 # the offline-assembled ROM blob (see §8)
  joust2.hi                 # optional hiscore NVRAM
```

---

## 8. Building the ROM blob offline

MiSTer assembles ROMs at runtime from an `.mra` file; Game Bub has no MRA engine, so you
assemble once on a PC. Using the `.mra` from the Joust 2 repo's `releases/` folder plus a
matching MAME romset and the [`mra`](https://github.com/sebdel/mra-tools-c) tool:

```sh
mra -A -O ./out "Joust2 (prototype, rev 1).mra"
# produces a flat .rom blob — exactly the dn_* byte stream williams2 expects for dn_index 0
mv "./out/Joust2 ... .rom" roms/joust2.j2
```

Copy `joust2.j2` to the SD card's `roms/` folder. This blob is what `arcade.rs` streams into
the download window. (The `.mra` also defines the NVRAM defaults if you implement hiscore
save/restore.)

---

## 9. Building, and what's left

### Build

```sh
# FPGA bitstream
cd fpga
fusesoc --cores-root . --work-root=build/arcade --target=handheld_rev2 --flag=module_arcade elipsitz:gameboy:gameboy
gzip -c build/arcade/.../top_handheld.bit > arcade_joust2.bit.gz   # -> SD: system/

# MCU firmware
cd ../firmware/handheld
cargo run --release --features=rev2
```

Iterate in the Verilator simulator first — add a `platform.sim.SimArcade` and `sim_arcade`
filesets mirroring `SimGba`, so you can debug the video adapter without a 20-minute Vivado
build.

### Open items / things to verify

- **Not synthesized or hardware-tested here.** The first synthesis run will tell you whether
  the BRAM budget (§4d) holds. If it overflows, relocate `williams2`'s graphics ROMs to
  SDRAM or drop arcade cores to double-buffering.
- **Measure the real visible window.** `framebufferW/H = 292×240` is the nominal Williams 2
  figure; confirm the actual active region from `video_hblank`/`video_vblank` in simulation
  and set `framebufferW/H` to exactly that — it directly drives the BRAM math.
- **`video_ce`.** Requires the one-line `williams2.vhd` edit (§6). Verify the strobe lines
  up with stable `video_r/g/b/i`.
- **Color LUT.** Drop in the real 256-entry table from `Arcade-Joust2.sv` for faithful
  colors.
- **Player 2.** Tied off here. Williams games are 2-player; you could map P2 over the link
  port to a second unit, or expose a "swap controls" config bit like the MiSTer core's
  `status[13]`.
- **NVRAM hiscores.** The read-back path and `persist_save()` are described but not written
  out — optional for first light.
- **Pause vs. `io.enable`.** We pause via `pause_cpu`. Confirm `williams2` fully freezes
  (including video) when `pause_cpu` is held, or the triple buffer may see partial frames.
- **Licensing.** Joust 2's RTL is GPL; `fpga/` is `GPL-3.0-only` — compatible. Keep the
  headers and attribution (§3).

---

## 10. Checklist (mapped to porting-mister-cores.md)

- [x] **Triaged** (§1) — one clock, no external RAM, native 384×260 raster / ~292×240
      visible, GPL. Risk: BRAM budget.
- [ ] **Sources vendored** into `fpga/verilog/arcade/joust2/` with correct `file_type`s (§3)
- [ ] **`module_arcade` flag**, filesets, and generator wired into `gameboy.core` (§2, build)
- [ ] **Platform extended** — per-core scale + 10-bit framebuffer coords in
      `HandheldTop.scala` / `HandheldIo` (§4)
- [ ] **Clock wizard** `clk_wiz_system_arcade.v` producing 12 MHz (§5)
- [ ] **`HandheldArcade.scala`** — `Williams2` BlackBox + clocking, video, audio, input,
      download, config bridges (§6)
- [ ] **`williams2.vhd`** one-line edit exposing `video_ce` (§6)
- [ ] **Color LUT** copied from `Arcade-Joust2.sv` (§6)
- [ ] **`arcade.rs`** bitstream driver + `CurrentBitstream` variant + worker / game.rs arms
      + file extension (§7)
- [ ] **ROM blob** assembled offline from the `.mra` and placed on the SD card (§8)
- [ ] **`SimArcade`** + sim filesets for the Verilator debug loop (§9)
- [ ] **Verified** in simulator, then on hardware — BRAM budget, visible window, `video_ce`,
      pause behavior (§9)
```
