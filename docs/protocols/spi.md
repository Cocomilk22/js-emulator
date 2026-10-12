# SPI (Serial Peripheral Interface)

Part of the [protocol notes](README.md). Written 2026-10-11.

## How it works

**Signals.** Four wires, single controller (master), one or more peripherals:
SCLK (controller output), MOSI/COPI (controller out), MISO/CIPO (peripheral
out), and one active-low CS_n per peripheral. Full duplex: every clock shifts
one bit each way. Variants: 3-wire (bidirectional data on one pin, half
duplex), Dual/Quad/Octal SPI (2/4/8 bidirectional data lines, used by flash
and PSRAM), daisy chain (MISO of one into MOSI of the next).

**Electrical level.** Push-pull CMOS at VDDIO; no pull-ups required (except a
pull-up on CS_n to keep peripherals deselected at reset). MISO is tri-stated
by deselected peripherals, which is what lets several share one line.

**There is no formal standard.** SPI is a Motorola de-facto convention; the
peripheral's datasheet is the specification. The parameters are:

| Parameter | Options |
|---|---|
| CPOL | Clock idle level 0 or 1 |
| CPHA | 0: data sampled on the first (leading) clock edge, changed on the trailing edge; 1: changed on leading, sampled on trailing |
| Mode | 0 = (CPOL 0, CPHA 0), 1 = (0,1), 2 = (1,0), 3 = (1,1); modes 0 and 3 dominate |
| Bit order | MSB first (common) or LSB first |
| Word length | Any; 8, 16, 24, 32 common; flash commands mix widths |
| CS behaviour | Held across a transaction; some parts need CS toggled per word |

**Timing parameters** (typical numbers; always from the peripheral datasheet):

| Parameter | Typical value |
|---|---|
| SCLK | 1–20 MHz for sensors/ADCs; 50–133 MHz for SPI NOR flash; 32 MHz achieved on TT QSPI Pmod |
| CS setup before first edge (tCSS) | 5–100 ns |
| Data setup/hold to sampling edge | 2–10 ns at the peripheral |
| Peripheral clock-to-out (tV) | 5–15 ns |

**Round-trip constraint (the real limit for a controller).** In mode 0 the
controller must sample MISO on the rising edge that ends the bit, but the
peripheral only launches MISO after the previous falling edge. Available time
= half an SCLK period, minus controller output delay, peripheral tV, and
controller input delay. The TT GPIO spec measured ~20 ns worst round-trip
through the sky130 mux and pads; on our chip a 2-flop input synchronizer adds
another 1–2 core cycles. This is why SPI controllers need a **programmable
sample delay** (sample later than the nominal edge), as the RP2040 PIO and
many MCU SPI blocks allow.

```text
mode 0, MSB first, 8 bits
CS_n ‾‾\_____________________________________/‾‾
SCLK ___/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_______
MOSI ==X 7 X 6 X 5 X 4 X 3 X 2 X 1 X 0 X=====
MISO --X 7 X 6 X 5 X 4 X 3 X 2 X 1 X 0 X-----   (tri-stated outside CS)
        ^ sampled on rising, changed on falling
```

## Applications
NOR/NAND flash (boot ROMs; QSPI XIP), SD cards in SPI mode, ADCs/DACs,
displays (SPI LCD/OLED, often 3-wire + D/C pin), sensors (IMUs, pressure),
Ethernet controllers (W5500, ENC28J60), radios (nRF24, LoRa SX127x), shift
register chains (74HC595), FPGA configuration, TPMs. On this chip SPI is also
the natural **host link** (TT demo board MCU, an RP2350 on the ETR board, or the TT QSPI Pmod).

## Emulator requirements
- **Controller mode:** synchronous; the engine generates SCLK, so timing is
  fully internal. Needs: shift register with configurable width and bit order,
  **side-set** of SCLK with the data shift in the same cycle (otherwise each bit
  costs ≥ 2 instructions per edge), CS on a second side-set or SET pin,
  programmable MISO sample delay, and 1 shift-in per bit.
- **Peripheral mode:** SCLK is external and asynchronous to the core. Needs:
  2-flop synchronizer, `wait for edge on pin`, shift-in on sampled edge, and
  fast MISO update after the launching edge. Achievable external SCLK ≈ core
  clock / 4 to / 6 **(hypothesis)**: each SCLK phase must last ≥ 2 core cycles
  to be seen reliably, and MISO must be updated within half a period.
- **Dual/Quad:** shift 2 or 4 bits per clock across a contiguous pin group,
  plus direction turnaround (OE switch) after the command phase. Needs pin
  groups with OE control on `uio`.
- **Helpers:** none beyond shift/side-set; CRC-7/CRC-16 for SD cards is a
  nice-to-have (the CRC unit covers it).

| Core clock | Controller SCLK at 2 cycles/bit (f/2) | at 4 cycles/bit (f/4, realistic with sample delay) | Peripheral SCLK max (f/5, hypothesis) |
|---|---|---|---|
| 25 MHz | 12.5 MHz | 6.25 MHz | 5 MHz |
| 40 MHz | 20 MHz | 10 MHz | 8 MHz |
| 50 MHz | 25 MHz | 12.5 MHz | 10 MHz |

The f/2 column also exceeds the sky130 pad output rating of 33 MHz only above
66 MHz core; the IHP pad rating on the TT IHP chip is **not yet verified**.

**Pins:** controller: SCLK, MOSI, CS on outputs (uo_out or uio), MISO on any
input. Peripheral: SCLK, MOSI, CS on inputs, MISO on a `uio` pin (must
tri-state when CS_n is high if the bus is shared). TT recommends `uio[0..3]` =
CS, MOSI, MISO, SCK (Pmod top row).

**External hardware:** none at 3.3 V.

**Feasibility:** easy (core protocol).

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

Demo settings: modes 0–3, controller and peripheral roles, concurrently.
TT SPI Pmod convention: `uio[0]` CS, `uio[1]` MOSI, `uio[2]` MISO, `uio[3]`
SCK (bottom row `uio[4..7]`). No external parts at 3.3 V.

## References
- Motorola/Freescale (now NXP) *SPI Block Guide* (S12SPIV3/V4), the original
  definition of CPOL/CPHA (not re-checked for these notes)
- Analog Devices, *Introduction to SPI Interface*:
  https://www.analog.com/en/resources/analog-dialogue/articles/introduction-to-spi-interface.html
- Tiny Tapeout GPIO limits (round trip ~20 ns, sky130): https://tinytapeout.com/specs/gpio/
- Tiny Tapeout SPI/QSPI pinouts: https://tinytapeout.com/specs/pinouts/
