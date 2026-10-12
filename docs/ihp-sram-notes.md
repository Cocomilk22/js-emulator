# IHP SRAM macros on CMOS5L: integration notes

Technical summary of Ken Pettit's public walkthrough *Dropping a real SRAM
macro into your CMOS5L tile* (written 2026-09-29, read 2026-10-11):
https://kdp1965.github.io/ihp-um-janestreet-prism/ihp_sram_macro.html, with
its worked example repository https://github.com/kdp1965/ihp-um-janestreet-prism.
Cross-checked against the IHP-Open-PDK LEF files at the PDK commit the
walkthrough uses (`2bbec755dc67ca3db0261c3d6163e15735d66710`,
https://github.com/IHP-GmbH/IHP-Open-PDK).

Labels: "Walkthrough:" = statements from the walkthrough; "PDK files:" = read
from the PDK; "Our estimate:" = our own analysis, a hypothesis until we harden.

## 1. Macros and sizes

- **Walkthrough:** the macros live in `$PDK_ROOT/ihp-sg13cmos5l/libs.ref/sg13cmos5l_sram/`
  (cdl, doc, gds, lef, lib, verilog). The name encodes the shape, e.g.
  `RM_IHPSG13_1P_512x32_c2_bm_bist` = single-port, 512 words × 32 bits,
  column mux 2, bit mask, BIST port. Single-port sizes run from 64×16 to
  8192×32; dual-port up to 1024×32. The worked example uses **two 512×32 macros** (2 KB
  each, 416.64 × 191.34 µm, 0.08 mm²) as TX/RX packet FIFOs.
- **Walkthrough:** one macro is 6.5 % of an 8x4 tile (1.22 mm²); the example's **8x4 die is
  1724.16 × 710.64 µm**. A flop in the CMOS5L library is "about 52 µm²"
  before muxing; the macro stores a bit in under 5 µm² (~11× denser).
- **PDK files:** `sg13cmos5l_sram` is a symlink to the SG13G2 SRAM library, so the
  macro set is the same. Sizes we care about (LEF `SIZE`):

| Macro | W × H (µm) | Area (µm²) | Share of 6x4 (≈ 0.92 mm², estimate) | Share of 8x4 (1.225 mm²) |
|---|---|---|---|---|
| `RM_IHPSG13_1P_64x16_c2` (no mask/BIST) | 236.80 × 64.36 | 15 240 | 1.7 % | 1.2 % |
| `RM_IHPSG13_1P_256x16_c2_bm_bist` | 236.80 × 118.78 | 28 127 | 3.1 % | 2.3 % |
| `RM_IHPSG13_1P_512x16_c2_bm_bist` | 236.80 × 191.34 | 45 309 | 4.9 % | 3.7 % |
| `RM_IHPSG13_1P_256x32_c2_bm_bist` | 416.64 × 118.78 | 49 489 | 5.4 % | 4.0 % |
| `RM_IHPSG13_1P_512x32_c2_bm_bist` | 416.64 × 191.34 | 79 720 | 8.7 % | 6.5 % |
| `RM_IHPSG13_2P_256x16_c2_bm_bist` | 419.95 × 136.97 | 57 520 | 6.3 % | 4.7 % |

- **PDK files:** CMOS5L standard cells have the same footprints as SG13G2 ones
  (e.g. `sg13cmos5l_dfrbpq_1` 12.96 × 3.78 µm = 49.0 µm²,
  `sg13cmos5l_dfrbp_1` 13.92 × 3.78 = 52.6 µm², `mux2_1` 18.1 µm²), so SG13G2
  area figures carry over.
- **Our estimate:** the 6x4 die area is not published. Jane Street's "8x4 is ~30 %
  more" and the example's 1.225 mm² 8x4 die put 6x4 at about **0.92 mm²**
  (1.225 / 1.33), above the brief's nominal 0.7 mm² (24 × 200 × 150 µm). This
  is an estimate until we harden an empty 6x4 design.

## 2. RTL wrapper

- **Walkthrough:** read `doc/<macro>.txt` first. **`A_DLY` must be tied to 1.**
  Nothing happens unless `A_MEN` is high; `A_REN` reads, `A_WEN` writes,
  both high = write-through; every input is sampled on the rising `A_CLK`.
  **Read data is registered**: address on one edge, `A_DOUT` valid after it.
  Tie all `A_BIST_*` inputs to 0. `A_BM` is a per-bit write mask.
- **Walkthrough:** three files: the instance (port names from the datasheet), a
  `(* blackbox *)` module header for Yosys wrapped in `` `ifndef SIM ``, and
  the PDK behavioural model (`RM_IHPSG13_1P_core_behavioral_bm_bist.v` plus
  the per-macro file) compiled with `-DSIM -DFUNCTIONAL`. The author vendors the
  models into `test/` (Apache-2.0) because the GitHub test job has no PDK
  checkout.
- **Walkthrough (gotcha):** the gate-level netlist keeps the SRAM as a black box, so
  **gate-level simulation also needs the behavioural model**.
- **Walkthrough:** keep hierarchical instance paths stable (the author uses separate `if`
  generate blocks), because the flow config refers to instances by path.
- **Our estimate:** our "next PC drives the SRAM address" fetch fits the registered-read
  macro: the instruction for the next cycle comes out after the edge that
  samples the address. The critical path is `A_DOUT` → decode → condition →
  next-PC → `A_ADDR` setup; the `.lib` files give clock-to-out and setup.
  `A_MEN`/`A_REN` would be high almost every cycle (the engine fetches every
  cycle), which costs power, not correctness. Host loading uses the same port
  while the engine is halted (mux on `A_ADDR`/`A_DIN`/`A_WEN`).

## 3. LibreLane config

- **Walkthrough:** a `MACROS` block in `src/config.json`: `gds`, `lef`, `spice` (the
  CDL, needed by LVS), and three `lib` corners mapped onto LibreLane's names
  (`nom_typ_1p20V_25C`, `nom_slow_1p08V_125C`, and the −55 °C fast lib mapped
  onto `nom_fast_1p32V_m40C`), plus one `instances` entry per instance with
  `location` (lower-left, µm) and `orientation` (N/FS).
- **Walkthrough:** all signal pins are on one long edge, on Metal2. The author stacks the two
  macros with their pin faces toward each other (FS and N) and puts the FIFO
  logic in the ~100 µm gap between them.
- **Walkthrough:** `FP_MACRO_HORIZONTAL_HALO` = 2 µm, and explicit
  `ROUTING_OBSTRUCTIONS` on Metal2–Metal4 over each macro, because **a TT
  CMOS5L tile routes on Metal2–Metal4 only** (TopMetal belongs to the
  chip-level harness), so nothing can route over a macro.

## 4. Power on Metal4

- **Walkthrough:** the macro has three supplies as vertical Metal4 columns: `VDD!`
  (periphery, lower part), `VDDARRAY!` (bit cells, upper part), `VSS!`
  (full height). For the 512×32: 36 columns, VDD at x = 4.26 + 11.24·k,
  VSS at 9.88 + 11.24·k (in macro coordinates).
- **Walkthrough:** the CMOS5L tile PDN is **vertical Metal4 VPWR/VGND stripes
  only**. A macro is fed only where a tile stripe lands on one of its
  columns. **A macro with no power passes DRC and LVS and is a brick on
  silicon.** The example's grid: `FP_PDN_MULTILAYER` 0, `FP_PDN_VWIDTH` 2.1,
  `FP_PDN_VSPACING` 3.52, `FP_PDN_VPITCH` 44.96 (4 × 11.24),
  `FP_PDN_VOFFSET` 6.15, with the macro placed at x = 3.36 so the first
  stripe pair hits the first column pair. The macro's columns are not
  perfectly regular (middle band, right array 0.67 µm off), so the author has a
  custom LibreLane step that nudges stripes onto columns, and options for
  extra stripes (grid only: 10 pairs per SRAM; every-other: 17; all
  columns: 36). All-columns cost little routing in the example's design but caused
  shorts with Metal4 pins of a latch macro above the SRAM, so the author uses
  every-other. Without a custom step, setting the pitch to 11.24 µm with
  plain pdngen feeds every column pair of the left array, at the cost of 4×
  the stripes across the whole tile.
- **Walkthrough:** `PDN_MACRO_CONNECTIONS` needs two lines per instance (`VDD!` and
  `VDDARRAY!` to VPWR, `VSS!` to VGND); the instance regex needs escaped
  brackets. Check by looking at Metal4 over the macro in the OpenROAD GUI and
  in the IR-drop report (the example's worst case is well under 1 mV).
- **PDK files:** our candidate macros (256×16, 64×16) use the **same column
  rhythm**: VDD at 4.26 + 11.24·k, VSS at 9.88 + 11.24·k (20 VDD columns for
  the 236.8 µm wide macros). **Our estimate:** so the walkthrough's grid numbers should apply if
  every macro's x position is 3.36 + 44.96·n; the irregular columns of our
  macros still need checking.

## 5. Sign-off gotchas

| Check | Finding in the walkthrough | Fix used there |
|---|---|---|
| Antenna | ~190 inputs on Metal2 pins; the checker could not always see their drivers | Custom step adding one `sg13cmos5l_antennanp` diode per SRAM input |
| Magic DRC | Reads the macro LEF abstract; a Metal4 obstruction bar at the VDD/VDDARRAY break makes a full-height stripe an "illegal overlap" (22 in the example's run) | `RUN_MAGIC_DRC` 0, `ERROR_ON_ILLEGAL_OVERLAPS` 0, **`RUN_KLAYOUT_DRC` 1 with the CMOS5L deck** (the deck the precheck runs) |
| LVS | Needs the CDL; unconnected macro power pins mean a stripe missed a column | Point `spice` at the CDL; fix the PDN |
| TT precheck | SRAM GDS has marker layers 25/0 (SRAM.drawing) and 16/0 (DigiBnd.drawing); older prechecks rejected them | Whitelisted by TT maintainers since September 2026; update local precheck copies |
| Tests | — | Write-and-read-back, back-to-back reads, partial `A_BM` write; run on RTL, post-synthesis and final gate-level netlists |

## 6. How this design uses it (preliminary)

1. **Instruction memory:** one 256×16 macro per engine (four at 6x4, six at
   8x4 plus an optional shared 512×32 data macro); see
   [architecture.md](architecture.md). Each macro must sit on the 44.96 µm
   power rhythm, so floorplan and power are the main integration risk.
2. **Fallbacks:** if the 256×16 macros do not route, the 64×16 macro
   (15.2 k µm²) gives 64 words per engine at about half the area; the last
   resort is a 64×16 latch array per engine.
3. **Floorplan idea:** the 236.8 µm wide macros stacked in one column at the
   left edge, x = 3.36 µm, in face-to-face pairs, with engine logic next to
   each macro's pin face.
4. **Flow settings to reuse:** `MACROS` block, halo, Metal2–4 obstructions,
   Metal4-only PDN on the macro's column rhythm, `PDN_MACRO_CONNECTIONS` for
   `VDD!`/`VDDARRAY!`, KLayout DRC instead of Magic, CDL for LVS, the PDK
   commit the precheck uses.
5. **Verification:** vendor the behavioural models so RTL, post-synthesis and
   gate-level runs in CI all use them; run the write/read-back, back-to-back
   read and partial-mask tests on every netlist.
