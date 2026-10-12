# JTAG (IEEE 1149.1 Test Access Port)

Part of the [protocol notes](README.md). Written 2026-10-11. Primary source:
IEEE Std 1149.1-2013, *Standard for Test Access Port and Boundary-Scan
Architecture*.

## How it works

**Signals.** TCK (test clock, from the probe), TMS (mode select, probe →
target), TDI (data in to target), TDO (data out of target, tri-state when not
shifting), optional TRST_N (async TAP reset). Synchronous, full duplex in the
shift states. Devices are **daisy-chained**: TDO of one feeds TDI of the next,
TCK/TMS bussed to all. Variants: ARM adaptive clocking (RTCK returned by the
target), IEEE 1149.7 (cJTAG, 2-pin), IEEE 1149.6 (AC-coupled boundary scan).

**Electrical level.** Push-pull at the target's VREF (1.2–5 V; 3.3 V common).
Probes usually level-shift to VREF. TDO is driven only in Shift-DR/Shift-IR.
Pull-ups on TMS/TDI are recommended by the standard so a floating TMS walks
the TAP to Test-Logic-Reset.

**State machine.** A 16-state TAP controller advances on every **rising TCK
edge** according to TMS. Key paths: five TCKs with TMS = 1 from any state →
Test-Logic-Reset; Run-Test/Idle → Select-DR → Capture-DR → Shift-DR (shift
n bits, TMS = 1 on the last) → Exit1-DR → Update-DR; the IR path is the same
with an extra Select-IR step. Instruction register length is device-specific
(IDCODE, BYPASS, EXTEST, SAMPLE/PRELOAD are mandatory/standard
instructions; BYPASS is a 1-bit register).

**Timing rules.** TMS and TDI are **sampled by the target on the rising edge**
of TCK; TDO **changes on the falling edge** of TCK, so the probe can sample
TDO on the next rising edge with half a period of margin. Bits shift LSB
first. TCK has no minimum frequency (fully static), so bit-banging at any
rate is legal. Typical maximum TCK: 10–50 MHz depending on target and cable;
many MCUs specify ≤ 1/6 of their core clock when RTCK is not used.

```text
TCK  _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
TMS  =X 1 X 0 X 0 X 0 X 1 X==     Select-DR, Capture-DR, Shift-DR, Shift, Exit1
TDI  ===========X d0X d1X d2X==   sampled on rising TCK
TDO  -----------X q0 X q1 X q2X-   changes on falling TCK (Hi-Z outside Shift)
```

## Applications
Boundary-scan board test (interconnect test, EXTEST), CPU debug (ARM
Cortex-A/R, RISC-V debug transport, MIPS EJTAG), FPGA/CPLD configuration
(Xilinx/Lattice/Intel), flash programming through the debug port, security
research (finding and reverse engineering exposed JTAG ports; tools like
JTAGulator brute-force pin assignments), production test.

## Emulator requirements
- **Probe (host) mode:** fully synchronous, the engine owns TCK. Per bit: set
  TMS and TDI, raise TCK (side-set), sample TDO, lower TCK. With side-set
  and a 2-bit-wide output shift (TMS, TDI) this is 2 instructions per bit.
- **Long shifts:** DR scans can be thousands of bits (boundary scan chains,
  FPGA bitstreams). Needs auto-refill shift from a host FIFO and a loop counter
  > 16 bits or a host-managed chunking scheme.
- **TDO sampling:** TDO is launched on the falling edge; sampling it just
  before the rising edge gives half a period. The input synchronizer adds 1–2
  cycles of latency that the sample timing must absorb.
- **Pin discovery (novel demo):** a JTAGulator-style scan (try all pin
  permutations, look for IDCODE) is a natural "reverse engineering" demo and
  only needs pins re-assignable at runtime.
- **Target mode** (emulate a TAP): the engine samples external TCK; same limits
  as SPI peripheral mode (external TCK ≤ core/5, hypothesis).
- **Helpers:** none special; TAP state tracking is firmware. A "shift n bits
  with TMS = 1 on the last bit" is a nice ISA-level convenience.

| Core clock | TCK at 4 cycles/bit | TCK at 8 cycles/bit |
|---|---|---|
| 25 MHz | 6.25 MHz | 3.1 MHz |
| 50 MHz | 12.5 MHz | 6.25 MHz |

**Pins:** probe: TCK, TMS, TDI on outputs, TDO on an input, TRST_N on an
output; all can use dedicated `uo_out`/`ui_in`, leaving `uio` for open-drain
protocols. **External hardware:** level shifter if the target VREF ≠ 3.3 V.

**Feasibility:** easy. Good candidate for showing a protocol added by
firmware only.

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

No parts for 3.3 V targets; a level-shifter Pmod for other VREF.

## References
- IEEE 1149.1-2013: https://standards.ieee.org/ieee/1149.1/4484/
- Texas Instruments SSYA002, *IEEE Std 1149.1 (JTAG) Testability Primer*: https://www.ti.com/lit/an/ssya002c/ssya002c.pdf
- JTAGulator (pin discovery): https://github.com/grandideastudio/jtagulator
