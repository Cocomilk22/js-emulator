# UART (asynchronous serial)

Part of the [protocol notes](README.md). Written 2026-10-11. Facts below are
from the cited sources; items marked **(hypothesis)** depend on our own clock
and pad assumptions and are not yet validated by synthesis or silicon.

## How it works

**Signals.** One wire per direction: TXD (output) and RXD (input), plus ground.
Optional hardware flow control RTS/CTS (one wire each way). Full duplex, point
to point. Idle state is **mark** (logic 1, line high in TTL/CMOS UART).

**Electrical level.** "UART" is a framing, not an electrical standard:

| Variant | Levels | Needs external part? |
|---|---|---|
| TTL/CMOS UART (what the chip pins speak) | 0 V / VDDIO (3.3 V on the TT demo board) | No |
| RS-232 (TIA-232-F) | Inverted: mark = −3 to −15 V, space = +3 to +15 V | Yes: charge-pump transceiver (e.g. MAX3232) |
| RS-485 / RS-422 (TIA-485-A / -422-B) | Differential, ≥ ±1.5 V driver into 54 Ω (485) | Yes: RS-485 transceiver; half duplex needs a driver-enable pin |

**Line coding and framing.** NRZ, LSB first. A frame is: start bit (0), 5–9
data bits, optional parity (even/odd/mark/space), 1, 1.5 or 2 stop bits (1).
The common format is **8N1**: 10 bit times per byte, 80% efficiency.
Break = line held at 0 for longer than one frame. There is no clock wire: both
ends agree on the baud rate in advance and the receiver re-synchronizes on
**every start bit** (falling edge after idle).

**Timing parameters.**

| Parameter | Value | Note |
|---|---|---|
| Common baud rates | 9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600, 1 M, 2 M, 3 M baud | 115200 is the de-facto console rate |
| Bit time at 115200 | 8.68 µs | |
| Receiver oversampling (conventional) | 16× (some parts 8× or 4×) | 16550-style UARTs sample at the 7th/8th/9th of 16 sub-bit ticks and majority-vote |
| Theoretical limit, 8N1, ideal edges, 16× | ≈ 4.6 % total mismatch between the two ends | Derivation below |
| AN2141 budget, "normal" link | ±3.3 % total TX/RX mismatch | Maxim/ADI AN2141 (16× receiver) |
| AN2141 budget, "nasty" long capacitive RS-232 link | ±2 % total | AN2141; the budget is shared by both ends, so roughly ±1–1.65 % each |

Tolerance derivation (8N1, 16× oversampling): the receiver detects the start
edge with up to 1/16 bit uncertainty and samples the stop bit 9.5 bit times
later. For the stop-bit sample to stay inside the stop bit, the accumulated
error must be below 0.5 − 1/16 = 0.4375 bit, so |Δf/f| < 0.4375 / 9.5 ≈ 4.6 %
total. Edge distortion and slow edges on real lines eat into this, which is
why AN2141 budgets ±3.3 % (normal) or ±2 % (nasty link) for the sum of both
ends' errors. Any baud-synthesis error of our divider counts against it.

```text
byte 0x4D = 0b0100_1101, sent LSB first, 8N1
bit    idle  S   D0  D1  D2  D3  D4  D5  D6  D7  Stop idle
level   1    0   1   0   1   1   0   0   1   0   1        1
           ^ falling edge from idle: RX hard-syncs here, then samples at
             1.5, 2.5, ... 9.5 bit times after this edge
```

## Applications
Debug consoles, bootloaders (ROM bootloaders on most MCUs), GPS/GNSS modules
(NMEA at 9600–115200), Bluetooth/Wi-Fi/cellular modems (AT commands), MIDI
(31250 baud, current loop), DMX512 (250 kbaud over RS-485), Modbus RTU
(RS-485), LIN automotive bus (UART framing, 19.2 kbaud, single-wire
open-drain-like transceiver), smart-card ISO 7816 (half-duplex, UART-like
with guard time and parity-NACK).

## Emulator requirements
- **Timing source:** a baud generator. Integer dividers give up to ±0.5 cycle
  error per bit, which grows with low cycles-per-bit; a **fractional
  (phase-accumulator) divider** keeps long-term rate error below 0.01 % at any
  baud with ≤ 1 cycle jitter.
- **TX:** shift 10 bits out on bit ticks. No input needed. Trivial.
- **RX:** asynchronous clock recovery. Needs: 2-flop input synchronizer, falling
  edge detect from idle, wait half a bit, verify start still 0 (glitch reject),
  then sample every bit time. Optional 3-sample majority vote. Framing error =
  stop bit is 0. Hardware wants: `wait for edge`, `reset bit-phase on edge`
  ("hard sync"), `sample at phase`, shift-in.
- **Pins:** TX on any output (uo_out), RX on any input (ui_in). RTS/CTS: one
  output, one input. RS-485 half duplex: one extra output for driver enable.
- **Helpers:** parity is one XOR-reduce; nice to have, firmware can compute it.
- **Host side:** RX FIFO so bytes are not lost while the host polls.

| Core clock | 115200 baud cycles/bit | 1 Mbaud | 3 Mbaud |
|---|---|---|---|
| 25 MHz | 217 | 25 | 8.3 |
| 40 MHz | 347 | 40 | 13.3 |
| 50 MHz | 434 | 50 | 16.7 |

For RX, 16× oversampling needs ≥ 16 cycles per bit, so the RX ceiling with
classic oversampling is f/16 ≈ 3.1 Mbaud at 50 MHz; with a phase accumulator
and mid-bit sampling 8 cycles/bit is workable (≈ 6 Mbaud) **(hypothesis)**.

**External hardware:** none for TTL UART at 3.3 V. The TT demo board MCU (RP2040 on older boards, RP2350 on the ETR board)
bridges a UART to USB on `ui_in[3]` (RX into chip) and `uo_out[4]` (TX out of
chip), or `ui_in[1]`/`uo_out[0]` (TT pinout guide). RS-232/RS-485 need
transceivers. TT input pins are **not 5 V tolerant** (TT GPIO spec, sky130
pads; IHP pad tolerance not verified).

**Feasibility:** trivial (core protocol, first RTL probe target).

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

Demo settings: 9600–115200 baud standard rates (reach to 1 Mbaud), 8N1 and
parity variants, both TX and RX, several UARTs concurrently. RS-232 / RS-485
transceivers would go on a Pmod (TT UART Pmod pin order: `uio[0]` CTS,
`uio[1]` TXD, `uio[2]` RXD, `uio[3]` RTS).

## References
- Analog Devices (Maxim) AN2141, *Determining Clock Accuracy Requirements for
  UART Communications*:
  https://www.analog.com/en/resources/technical-articles/determining-clock-accuracy-requirements-for-uart-communications.html
  (PDF: https://pdfserv.maximintegrated.com/en/an/AN2141.pdf)
- TI PC16550D UART datasheet (16× sampling, FIFO model), TI literature number SNLS378
- TIA-232-F / TIA-485-A (electrical levels; standards documents, not freely available)
- Tiny Tapeout pinout recommendations (UART to the board MCU): https://tinytapeout.com/specs/pinouts/
