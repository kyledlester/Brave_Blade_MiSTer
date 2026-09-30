# Brave Blade (with music) for MiSTer

A fork of **[XelaNotPu's ZN-1 core for MiSTer](https://github.com/XelaNotPu/ZN1_MiSTer)**
made for one purpose: to play **Brave Blade** (Eighting / Raizing, 2000) with its
**original music**.

The upstream ZN-1 core runs Brave Blade, but only with sound effects. On the
real arcade board the music comes from a second sound system that the ZN-1
core did not implement. This fork adds that sound board to the FPGA core, so
Brave Blade plays its music and its sound effects together.

Everything else about the core is XelaNotPu's work, which in turn builds on
Robert Peip's PSX_MiSTer core. See [Credits](#credits). Next steps are to test all ZN-1 games and send a pull request to XelaNotPu for the entire core, but I am providing this individually for now.

## What's in this repository

```
Brave_Blade_MiSTer/
├── Releases/
│   ├── BraveBlade_20261001.rbf        ← the core (latest)
│   └── BraveBlade_20260930.rbf        ← previous core
├── MRA/
│   ├── Brave Blade (World) (with music).mra
│   ├── Brave Blade (USA) (with music).mra
│   ├── Brave Blade (Japan) (with music).mra
│   └── Brave Blade (Asia) (with music).mra
└── source/                            ← full core source and Quartus project
```

This repository only supports Brave Blade. For the other ZN-1 games, use the
upstream [ZN1_MiSTer](https://github.com/XelaNotPu/ZN1_MiSTer) core.

## Installation

1. Copy `Releases/BraveBlade_20261001.rbf` to `_Arcade/cores/` on your MiSTer
   SD card.
2. Copy the `.mra` files from `MRA/` to `_Arcade/` (or any folder under it).
3. Put your ROM zips in the MiSTer arcade ROM folder (`games/mame/` or
   `_Arcade/mame/`).
4. Pick **Brave Blade (World) (with music)** (or a regional version) from the arcade menu.

The MRAs load the core named `BraveBlade`; MiSTer uses the newest dated
`BraveBlade_*.rbf` in `_Arcade/cores/`.

A standard MiSTer with a **32 MB SDRAM module** is enough.

## ROMs required

No ROMs are included. You need, as MAME romsets:

| Zip | Used for |
|---|---|
| `coh1002m.zip` | Tecmo TPS BIOS (`m534002c-61.ic353`) and motherboard security key (`mg01.ic652`) |
| `brvblade.zip` | Game ROMs, security key, and the sound board ROMs: `spu0u049.bin` + `spu1u412.bin` (68000 sound program) and `ra-bbl_rom2.336` (YMF271 samples) |
| `brvbladeu.zip` / `brvbladej.zip` / `brvbladea.zip` | Only for the USA / Japan / Asia MRAs: the region EEPROM (`at28c16_usa` / `_japan` / `_asia`) |

Despite their names, the `spu*` files are the sound CPU's program, not
PlayStation SPU data.

The four Brave Blade sets share every game and sound ROM. In MAME they differ
only in a small region EEPROM, which the regional MRAs load.

## Status

| MRA | Status |
|---|---|
| Brave Blade (World) (with music) | Tested on MiSTer hardware with `BraveBlade_20260930.rbf`: music and sound effects. `BraveBlade_20261001.rbf` (loading screen on CRT, timing improvements) not yet hardware-tested |
| Brave Blade (USA / Japan / Asia) (with music) | Not yet tested on hardware (same core and ROMs as World, plus the region EEPROM) |

## How the music was added

### The missing hardware

Brave Blade runs on Raizing's ZN-1 board (PS9805). Next to the PlayStation
hardware it carries its own sound board:

- a **Motorola 68000** at 12 MHz running its own sound program,
- a **Yamaha YMF271-F "OPX"** sound chip at 16.9344 MHz, with a 4 MB sample ROM,
- an 8-bit **command latch** that the game's main (PSX) CPU writes to, plus an
  interrupt line into the 68000.

The game's main CPU never plays music itself. It sends one-byte commands to the
sound board, and the sound board's 68000 drives the YMF271. MAME emulates this
board in `src/mame/sony/zn.cpp` (`raizing_zn_state`) and
`src/devices/sound/ymf271.cpp`. Those were the reference for this work.

### 1. Measuring the real behaviour in MAME

Before writing any hardware, the board was traced in MAME 0.289 with Lua
scripts (`source/sim/mame/`), recording 150 seconds of boot, attract mode,
coin-up, start and gameplay:

- **Command interface.** The game writes a command as a halfword to
  `0x1FB00000`, then writes `0x1FB00004` to interrupt the 68000 (IRQ level 2,
  held until acknowledged). The 68000 reads the command from `0x180008` about
  17 µs later. 23 commands were seen in the capture; for example, `08` starts
  the attract music.
- **Sound program.** 512 KB of program ROM and only 16 KB of RAM are actually
  used. The program writes YMF271 registers through a busy-poll routine and
  keeps musical tempo by polling the YMF271's two timers.
- **YMF271 features actually used.** Nine 4-operator FM voices (sine wave,
  algorithms 3/6/12 with feedback), twelve 8-bit PCM voices, both timers, and
  the "sample ended" status flags. Every register write was logged, giving an
  exact stimulus to test against.
- **ROM layout.** The MRA's byte stream for the 68000 program (two ROMs
  interleaved) and the sample ROM were checked byte-for-byte against MAME's own
  copies (matching SHA-1).

### 2. A reference model first

- MAME's YMF271 was ported to a standalone C++ program
  (`source/sim/model/ymf_mame.h`) that replays the captured register writes. Its
  output matches MAME's own recording (correlation 0.998).
- A second, fixed-point model (`ymf_fx.h`) was written with the same structure
  the hardware would have: integer arithmetic only, lookup tables generated
  from MAME's formulas, and one sample computed per 44.1 kHz tick. It matches
  the MAME port on 99.98 % of samples.

### 3. The hardware (`source/rtl/raizing_snd/`)

- **68000:** Jorge Cwik's cycle-exact [fx68k](https://github.com/ijor/fx68k),
  imported unchanged, clocked at an average 12 MHz from the core clock.
- **Sound board glue:** the command latch and interrupt, 16 KB of work RAM,
  and a 4 KB cache in front of the 68000 program. The program and the 4 MB
  sample ROM live in SDRAM, in the part of the banked-ROM area Brave Blade never
  uses, so a 32 MB module is still enough.
- **YMF271:** a new sequential implementation. The core clock is exactly twice
  the chip's clock, so each 44.1 kHz sample has a fixed budget of 768 clocks.
  In that time the engine walks all 12 channel groups, with a single shared
  multiplier. The PlayStation core already uses nearly all of the FPGA's
  multiplier blocks. PCM voices prefetch their sample data from SDRAM ahead of
  time so playback never waits on memory.
- **Integration:** the PSX bus decode for the two command addresses, a 16-byte
  read mode and a fairness rule on the SDRAM port the sound board shares, and
  an output mixer. It keeps MAME's balance between the PlayStation SPU (sound
  effects) and the YMF271 (music), 0.35 : 1.0, with +6 dB overall headroom.
  The sound board only switches on when the MRA supplies its ROMs.

### 4. Verification

- **YMF271 hardware vs. the fixed-point model:** bit-identical over the full
  150-second capture (6.6 million stereo samples), including with deliberately
  slow memory. The worst case uses 584 of the 768 clocks per sample.
- **Whole sound board, running the real Brave Blade sound program** in a
  Verilator simulation, with the game's commands injected at the times MAME
  recorded, over 120 seconds:
  - it made 177,684 YMF271 register writes against MAME's 177,696;
  - after each of the 23 commands the write counts match MAME;
  - output loudness matches within 1–4 %.
- **FPGA build:** fits the Cyclone V at 97 % logic. All core clocks meet timing
  in every temperature/voltage corner. The small HDMI/video timing margins
  already present in the upstream core are equal or better.
- **Hardware:** Brave Blade (World) tested on MiSTer with the released core: music and sound effects both working.

Details, numbers and reproduction steps are in
[`source/rtl/raizing_snd/README.md`](source/rtl/raizing_snd/README.md) and
[`source/sim/README.md`](source/sim/README.md).

## Known limitations

- **Save states do not include the sound board.** After loading a save state,
  the music may be wrong until the game issues its next music command.
- **Song start timing** can differ from MAME by one tempo tick (8 ms). The
  sound program services its timers by polling, and when it is busy two ticks
  can merge. The tempo itself is unaffected.
- **Unused YMF271 features are not implemented** (they are also missing or
  incomplete in MAME): PFM, detune, alternate loop, "Acc On", external outputs,
  and reading back external memory. LFO pitch modulation uses a close linear
  approximation. Brave Blade uses none of these.
- The OSD still shows the upstream core name.

## Building

Open `source/ZN1.qpf` in **Quartus Prime 17.0** (Lite is sufficient) and
compile. The output is `source/output_files/ZN1.rbf`; rename it to
`BraveBlade_<date>.rbf` for the MRAs. The JTAG debug probes from upstream are
excluded by default and can be re-enabled with the Verilog macro
`ZN_JTAG_DEBUG`.

## Credits

- **XelaNotPu** — the [ZN-1 MiSTer core](https://github.com/XelaNotPu/ZN1_MiSTer)
  this fork is based on: ZN-1 board support, per-manufacturer boot ROMs, CAT702
  security, ROM banking, NVRAM/EEPROM/FRAM, rotation and pause overlay.
- **Robert Peip (FPGAzumSpass)** — [PSX_MiSTer](https://github.com/MiSTer-devel/PSX_MiSTer),
  which provides the CPU, GPU, GTE, SPU, DMA and memory subsystem.
- **The MiSTer project** and **Sorgelig** — the MiSTer framework and the SDRAM
  controller.
- **Jorge Cwik** — [fx68k](https://github.com/ijor/fx68k), the 68000 core used
  for the sound CPU.
- **The MAME team**, in particular the authors of `ymf271.cpp` (R. Belmont,
  Olivier Galibert, hap) and the ZN driver: the behavioural reference for the
  sound board and the YMF271.

## No copyrighted data

This repository contains **no game ROMs and no copyrighted game data**:

- The core bitstream embeds no boot ROM, no game data, and no captured NVRAM.
  Manufacturer boot ROMs and the sound-board ROMs are loaded at runtime from
  your ROM zips. EEPROM/FRAM initialise blank and self-configure, or are
  preloaded from your own ROM zips by the MRA.
- The `.mra` files reference romsets by name only — they contain no inline ROM
  data.
- The `source/` tree contains the original FPGA logic, the standard PSX_MiSTer
  base RTL, hardware algorithm tables (CAT702), the fx68k 68000 core and its
  microcode, YMF271 lookup tables computed from published formulas, and the
  original author's pause-overlay artwork — no game/BIOS/firmware images.

## License

The core and its source are Free Software, conveyed under the **GNU General
Public License v3 or later** (the tree mixes GPLv2-or-later and GPLv3-or-later
files, so the combination is GPLv3+; every file remains available under the
terms in its own header). Full texts ship in `source/COPYING.GPL2` and
`source/COPYING.GPL3`.

This core derives from **PSX_MiSTer** by Robert Peip (FPGAzumSpass) and the
**MiSTer framework**; the ZN-1 board support (per-manufacturer boot ROM,
CAT702 security, ROM banking, NVRAM/EEPROM/FRAM) is an independent
re-implementation developed with reference to the MAME project's hardware
documentation.

The 68000 core **fx68k** is Copyright (c) Jorge Cwik, GPLv3-or-later
(`source/rtl/fx68k/LICENSE`, imported unchanged; see `ORIGIN.md`). The Raizing
sound board and YMF271 RTL are an independent re-implementation written with
reference to MAME's behaviour. The standalone MAME YMF271 port used only for
verification (`source/sim/model/ymf_mame.h`) retains its BSD-3-Clause
attribution to its original authors.

## Legal

No ROMs. This repository contains no game ROMs and no copyrighted game data, and it provides no links or instructions for obtaining them. To use this core you must supply your own ROM dumps, made from original hardware or media that you legally own, where and to the extent your local law permits.

Trademarks. "Sony", "PlayStation", "ZN-1", and "ZN-2" are trademarks of Sony Interactive Entertainment Inc. The Sony ZN-1 arcade platform was licensed to and marketed by numerous manufacturers and publishers under their own arcade sub-brands; their company names, arcade-board brands, game titles, characters, and logos are trademarks or registered trademarks of their respective owners, including without limitation:

- **Capcom** (Capcom Co., Ltd.) — the ZN-1 / ZN-2 sub-brand
- **Tecmo** (Koei Tecmo Games Co., Ltd.) — the "TPS" sub-brand
- **Taito** (Taito Corporation, a Square Enix Group company) — the "FX-1" sub-brand
- **Video System** (Video System Co., Ltd.) and **Visco** (Visco Corporation)
- **Atlus** (Atlus Co., Ltd., a Sega Group company)
- **Eighting / Raizing** (Eighting Co., Ltd.)
- **Psikyo** (Psikyo Co., Ltd.)
- **Namco** (Bandai Namco Entertainment Inc.), whose System 11 / System 12 arcade boards share the same Sony ZN chassis
- **Acclaim Entertainment**, **Atari Games / Midway**, **Hudson Soft**, **Sunsoft (Sun Corporation)**, and other publishers and rights holders of ZN-1 titles

"Yamaha" and "YMF271" are trademarks of Yamaha Corporation; "Motorola" and "68000" refer to Motorola / NXP Semiconductors. Game titles referenced in this repository — including Brave Blade (Eighting/Raizing) — are trademarks of their respective owners.

This project is not affiliated with, endorsed by, or sponsored by Sony Interactive Entertainment or any of the companies or rights holders named above. All such names are used here in a purely nominative and descriptive manner, solely to identify the hardware and software being re-implemented or referenced.

Purpose. This is an independent, non-commercial hardware-preservation and interoperability project. The FPGA logic is an original re-implementation of the ZN-1 board's behavior, developed from observation and from publicly available documentation and references (including the MAME project's hardware documentation); it contains no proprietary source code from the original manufacturers.

Security-chip emulation. ZN-1 boards used CAT702 chips as a protection measure. This core re-implements that logic for interoperability and preservation, in the same manner as MAME and comparable FPGA cores. The CAT702 is a small challenge/response algorithm rather than stored key data, so no manufacturer key material is embedded in the bitstream. Laws such as the U.S. DMCA §1201 address circumvention of technological protection measures; whether and how they apply to this kind of preservation/interoperability use can depend on your jurisdiction and circumstances. Users are responsible for their own compliance.

User responsibility. Users are solely responsible for ensuring that their use of this core — including the acquisition and use of any ROM images — complies with copyright law and all other applicable laws in their jurisdiction.

No warranty. In accordance with the GPL license: THIS PROGRAM IS PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE. THE ENTIRE RISK AS TO THE QUALITY AND PERFORMANCE OF THE PROGRAM IS WITH YOU. IN NO EVENT WILL ANY COPYRIGHT HOLDER OR CONTRIBUTOR BE LIABLE TO YOU FOR DAMAGES, INCLUDING ANY GENERAL, SPECIAL, INCIDENTAL OR CONSEQUENTIAL DAMAGES ARISING OUT OF THE USE OR INABILITY TO USE THIS PROGRAM, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
