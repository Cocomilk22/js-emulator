# SWD (Arm Serial Wire Debug)

Part of the [protocol notes](README.md). Written 2026-10-11. Primary
source: Arm *Debug Interface Architecture Specification ADIv5.0 to ADIv5.2*
(IHI 0031) and ADIv6 (IHI 0074).

## How it works

**Signals.** SWCLK (clock, from the debug probe) and SWDIO (single
**bidirectional** data line). Optional SWO (trace output, UART or Manchester
encoded) and nRESET. Synchronous, half duplex with explicit turnaround,
point to point (multi-drop SWD exists in SWDv2 with target selection).

**Electrical level.** Push-pull at target VREF, but both sides drive SWDIO at
different times, so the line must be released at turnaround. A pull-up on
SWDIO (and often a pull-down on SWCLK) is recommended so idle levels are
defined.

**Packet** (all fields LSB first):

| Phase | Bits | Driven by | Content |
|---|---|---|---|
| Request | 8 | Host | Start (1), APnDP, RnW, A[2:3], Parity (even over APnDP, RnW, A2, A3), Stop (0), Park (1) |
| Turnaround | 1 (default; 1–4 configurable) | Nobody | Line released |
| ACK | 3 | Target | OK = `001`, WAIT = `010`, FAULT = `100` (as bits ACK[2:0]; bit 0 sent first) |
| Turnaround | 1 (writes only) | Nobody | |
| Data | 32 + 1 | Host (write) or target (read) | Data LSB first + even parity |
| Turnaround | 1 (reads only) | | |
| Idle | ≥ 0 cycles (often 8) | Host | SWDIO low with clock running, or clock stopped |

**Line reset:** ≥ 50 SWCLK cycles with SWDIO high, then ≥ 2 idle (low)
cycles. **JTAG-to-SWD switch** on SWJ-DP: line reset, the 16-bit sequence
0xE79E (LSB first), line reset again. ADIv5.2 adds a **dormant state** with a
128-bit selection alert sequence. After reset the host must read DPIDR.

**Edges.** The target samples SWDIO on the **rising** edge of SWCLK and
changes its output after the rising edge; the host therefore samples target
data before the next rising edge (CMSIS-DAP samples while SWCLK is low).
SWCLK can stop between packets; there is no minimum frequency. Typical
maximum: 10–50 MHz depending on the target.

```text
SWCLK _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
SWDIO  S  AP RW A2 A3 Pa St Pk |Trn| A0 A1 A2 |Trn| D0 ... D31 Par
       <------ host drives ------> Z <-target-> Z <-- host (write) -->
```

## Applications
Programming and debugging every Arm Cortex-M MCU (and many Cortex-A/R via
SWJ-DP), CMSIS-DAP / J-Link / ST-Link probes, production flashing, security
research (readout protection testing, glitching setups), and the Raspberry Pi
RP2040's own debug port. An emulator acting as an SWD **probe** makes the
chip a tiny debugger.

## Emulator requirements
- **Bidirectional pin with fast OE switching:** SWDIO on a `uio` pin, OE
  toggled at each turnaround cycle under program control, in the same cycle as
  the clock edge (side-set of OE is ideal).
- **Mixed-width shifts:** 8-bit request, 3-bit ACK, 32-bit data, 1-bit parity:
  shift units with per-instruction bit counts.
- **Branch on ACK:** after 3 bits, branch to WAIT-retry or FAULT handling
  (needs `jmp` on shifted-in value, or push ACK to host and let it decide).
- **Parity:** even parity over 32 bits (XOR-reduce helper or the CRC/LFSR unit
  configured as a 1-bit parity accumulator).
- **Timing:** same as JTAG probe: 2–4 instructions per bit; SWCLK ≈ core/4
  to core/8.
- **Target mode** (emulate an MCU's debug port, e.g. for fuzzing debuggers):
  sample external SWCLK, respond with ACK in the next cycles; feasible at slow
  SWCLK only (external clock ≤ core/5, hypothesis).

| Core clock | SWCLK at 4 cycles/bit | Line reset (50 cycles) |
|---|---|---|
| 25 MHz | 6.25 MHz | 8 µs |
| 50 MHz | 12.5 MHz | 4 µs |

**Pins:** SWCLK on any output, SWDIO on `uio`, nRESET on an output
(open-drain on the target side: drive low or release, so `uio` preferred).
**External hardware:** level shifting only if VREF ≠ 3.3 V.

**Feasibility:** easy-moderate (turnaround handling is the only subtlety).

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

No parts for 3.3 V targets; a level-shifter Pmod for other VREF.

## References
- Arm IHI 0031, ADIv5.0–5.2: https://developer.arm.com/documentation/ihi0031/latest/
- Arm IHI 0074, ADIv6.0: https://developer.arm.com/documentation/ihi0074/latest/
- CMSIS-DAP reference implementation (SW_DP bit timing): https://github.com/ARM-software/CMSIS-DAP
