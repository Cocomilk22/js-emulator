# I2C (Inter-Integrated Circuit) and SMBus

Part of the [protocol notes](README.md). Written 2026-10-11. Primary source:
NXP UM10204 *I2C-bus specification and user manual*, Rev. 7.0 (2021-10-01).

## How it works

**Signals.** Two wires shared by all devices: SCL (clock) and SDA (data). Half
duplex, multi-drop bus, multi-controller capable, 7-bit (or 10-bit)
addressing.

**Electrical level: open-drain, wired-AND.** Every device only pulls a line
**low** or releases it; a pull-up resistor (typically 1–10 kΩ) makes it high.
A line is high only if every device releases it. This is what makes clock
stretching and arbitration possible, and what a bit-banging engine must
emulate: drive 0 = output-enable on with output 0; drive 1 = output-enable
off (high impedance) and let the pull-up do it. **Never drive a 1.** Levels
are relative to VDD: VIL = 0.3 VDD, VIH = 0.7 VDD.

**Framing.**
- **START (S):** SDA falls while SCL is high. **STOP (P):** SDA rises while SCL
  is high. **Repeated START (Sr):** a START without a preceding STOP. These are
  the only times SDA may change while SCL is high.
- **Data:** SDA changes while SCL is low and is sampled while SCL is high; 8
  bits MSB first, then a 9th **ACK** bit driven by the receiver (0 = ACK,
  1 = NACK).
- **Address byte:** 7-bit address + R/W̄ bit. 10-bit addressing uses the
  `11110xx` prefix. Reserved addresses: general call 0x00, etc.
- **Clock stretching:** a target may hold SCL low after any falling edge to
  delay the controller; the controller must release SCL and **wait until SCL is
  actually high** before timing the high phase.
- **Arbitration:** two controllers that start together both drive SDA; one
  that releases SDA (1) but reads 0 has lost and must stop driving at once,
  continuing as a receiver. Controllers also synchronize their SCL by wired-AND
  (the low phase is set by the longest low).

**Timing parameters** (UM10204 §6.1, SDA/SCL bus-line characteristics table for Sm/Fm/Fm+):

| Parameter | Standard-mode | Fast-mode | Fast-mode Plus |
|---|---|---|---|
| f_SCL max | 100 kHz | 400 kHz | 1 MHz |
| tLOW min (SCL low) | 4.7 µs | 1.3 µs | 0.5 µs |
| tHIGH min (SCL high) | 4.0 µs | 0.6 µs | 0.26 µs |
| tHD;STA min (hold after START) | 4.0 µs | 0.6 µs | 0.26 µs |
| tSU;STA min (repeated START setup) | 4.7 µs | 0.6 µs | 0.26 µs |
| tSU;DAT min (data setup) | 250 ns | 100 ns | 50 ns |
| tHD;DAT min (data hold) | 0 ns | 0 ns | 0 ns |
| tVD;DAT max (data valid after SCL low) | 3.45 µs | 0.9 µs | 0.45 µs |
| tSU;STO min (STOP setup) | 4.0 µs | 0.6 µs | 0.26 µs |
| tBUF min (bus free STOP→START) | 4.7 µs | 1.3 µs | 0.5 µs |
| tr max (rise time, set by pull-up × Cb) | 1000 ns | 300 ns | 120 ns |
| tSP (spike suppression) | — | 50 ns | 50 ns |
| Cb max (bus capacitance) | 400 pF | 400 pF | 550 pF |

Also in UM10204: High-speed mode (3.4 MHz) needs a current-source pull-up on
SCL driven by the controller, and Ultra Fast-mode (5 MHz) is push-pull and
unidirectional. Neither is a target here.

**SMBus** (System Management Bus, smbus.org spec 3.x) is I2C-compatible with
extra rules: 10–100 kHz (to 1 MHz in v3), a **clock-low timeout of 25–35 ms**
(a device holding SCL low longer must reset), optional **PEC** byte
(CRC-8, polynomial x^8 + x^2 + x + 1 = 0x07), and fixed voltage thresholds.
PMBus is built on SMBus.

```text
S  A6 A5 A4 A3 A2 A1 A0 R/W ACK  D7 ... D0 NACK P
SCL ‾‾‾‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\ ... _/‾‾‾‾‾‾
SDA ‾‾\___X__X__X__X__X__X__X__X___/‾ ... ‾‾‾‾\__/‾‾
      ^START: SDA falls while SCL high     STOP: SDA rises while SCL high ^
      data changes only while SCL low; target pulls SDA low in the ACK slot
```

## Applications
Sensors (temperature, IMU, humidity), EEPROMs (24Cxx), RTCs, IO expanders
(PCF8574), PMICs and power monitors (SMBus/PMBus), display controllers,
DDR SPD EEPROMs, HDMI DDC/EDID, camera control (SCCB, an I2C variant),
board management (IPMI's IPMB), smart batteries.

## Emulator requirements
- **Open-drain emulation per pin:** drive-low or release, via output enable
  (OE) only. Must be impossible for firmware to drive high on an open-drain
  pin by mistake (a per-pin mode bit is safer than relying on firmware).
- **Readback of the actual line** while driving, for: clock stretching (wait
  until SCL reads high after release), arbitration (released SDA reads 0 =
  lost), and bus-busy detection. A hardware **"driven vs sensed mismatch"**
  flag that firmware can branch on is the cleanest primitive (also used by CAN).
- **START/STOP detection** in target mode: SDA edge while SCL high. Needs
  `wait` on a combined condition (edge on one pin, level on another).
- **Timing:** coarse. At 1 MHz Fm+ and 50 MHz core, one bit is 50 cycles.
  tSU;DAT 50 ns = 2.5 cycles and tHD;DAT 0 ns are easy if SDA changes ≥ 1
  cycle after the SCL falling edge as seen at the pin.
- **Timeouts:** SMBus 25–35 ms clock-low timeout and a bus-stuck recovery
  (9 clocks) need a long timer (35 ms = 1.75 M cycles at 50 MHz, so ≥ 21-bit
  counter, or a prescaled timer).
- **Spike filter:** Fm/Fm+ require 50 ns spike suppression; a 2–3 sample
  digital glitch filter on the input does this at ≥ 40 MHz.
- **Helpers:** CRC-8 for SMBus PEC (CRC unit).

| Core clock | Cycles per bit at 100 kHz | 400 kHz | 1 MHz |
|---|---|---|---|
| 25 MHz | 250 | 62 | 25 |
| 50 MHz | 500 | 125 | 50 |

**Pins:** SCL and SDA must be on **bidirectional `uio` pins** (only those have
OE control on Tiny Tapeout). TT recommends `uio[2]` = SCL, `uio[3]` = SDA
(Pmod top row). External pull-ups (2.2–4.7 kΩ to 3.3 V) are needed; IHP pad
internal pull-ups, if any, are **not verified** and would be too weak for
Fm+ anyway. Max bus speed is set by pull-up × capacitance, not by the chip.

**External hardware:** pull-up resistors; a level shifter (e.g. BSS138 or
PCA9306) for 5 V buses since TT inputs are not 5 V tolerant.

**Feasibility:** easy (core protocol); target mode is moderate because of
START/STOP detection and fast ACK response.

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

Demo settings: 100 kHz and 400 kHz (1 MHz as reach), controller and target
roles. The pull-up resistors go on a Pmod, but the pins must still be driven
**open-drain** on chip (drive low or release with `uio_oe`), so the
open-drain pin mode stays in the design. TT I2C Pmod convention: `uio[2]`
SCL, `uio[3]` SDA (`uio[0]` INT, `uio[1]` RESET; bottom row `uio[6]` SCL,
`uio[7]` SDA).

## References
- NXP UM10204 Rev 7.0: https://www.nxp.com/documents/user_manual/UM10204.pdf
  (archived copy: https://web.archive.org/web/20210426060837/https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
- SMBus specification: https://smbus.org/specs/
- Tiny Tapeout I2C pinout: https://tinytapeout.com/specs/pinouts/
