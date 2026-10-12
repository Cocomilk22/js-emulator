# Protocol comparison and requirement matrix

**Status: Draft — open for discussion.** Preliminary, written 2026-10-11.
Builds on the per-protocol notes in [protocols/](protocols/README.md); numbers
come from there and carry the same sources. Everything about the chip's
clock, pads and area is a **hypothesis** until the first synthesis and
place-and-route runs measure it; CMOS5L pads are not characterized yet.

## 1. How the protocols differ

| Protocol | Clocking | Duplex | Topology | Signal wires | Typical speed | Line coding | Error detection | Arbitration / multi-master |
|---|---|---|---|---|---|---|---|---|
| UART | Asynchronous, agreed baud, resync per start bit | Full | Point to point | TX, RX (+RTS/CTS) | 9.6 k–3 Mbaud | NRZ, start/stop framing | Optional parity, framing error | None |
| SPI | Synchronous, controller drives SCLK | Full | One controller, many peripherals (CS each) | SCLK, MOSI, MISO, CS_n | 1–50 MHz (flash > 100 MHz) | NRZ, CPOL/CPHA | None | None |
| I2C | Synchronous, controller drives SCL, targets may stretch | Half | Multi-drop bus | SCL, SDA | 100 k / 400 k / 1 MHz | NRZ + START/STOP conditions | ACK/NACK; SMBus PEC CRC-8 | Yes, bitwise on SDA (wired-AND) |
| JTAG | Synchronous, probe drives TCK | Full (in shift states) | Daisy chain | TCK, TMS, TDI, TDO (+TRST) | 1–50 MHz | NRZ, TAP state machine | None (IDCODE checks) | None |
| SWD | Synchronous, probe drives SWCLK | Half, explicit turnaround | Point to point (multi-drop in v2) | SWCLK, SWDIO | 1–50 MHz | NRZ, packets | Parity, 3-bit ACK | None |
| PS/2 | Synchronous, **device** drives CLK | Half | Point to point | CLK, DATA | 10–16.7 kHz | NRZ, 11-bit frame | Odd parity, ACK | Host inhibit only |
| CAN | Asynchronous, tq bit timing, hard + bounded resync | Half | Multi-master bus | TXD, RXD (to transceiver); CAN_H/L on bus | 125 k–1 Mb/s (FD data to 5–8 M) | NRZ + 5-bit stuffing | CRC-15, stuff, form, ACK, bit errors | Yes, bitwise on ID (wired-AND) |
| USB LS | Asynchronous, DPLL on every edge | Half, host-scheduled | Tiered star | D+, D− | 1.5 Mb/s ± 1.5 % | NRZI + 6-bit stuffing | CRC5, CRC16, PID check | None (host polls) |
| 10BASE-T | Asynchronous, PLL on mid-bit edges | Full or half (CSMA/CD) | Point to point | TD±, RD± (pairs, via magnetics) | 10 Mb/s ± 100 ppm | Manchester | CRC-32 FCS | CSMA/CD in half duplex |

## 2. Emulator requirement matrix

"Precision" is the tightest timing the engine must hit, as a fraction of a bit
and in 50 MHz cycles (20 ns). "Per-bit budget" is core cycles available per bit
at 50 MHz at the highest rate we would target.

| Protocol | Target rate (v0) | Per-bit budget @50 MHz | Precision needed | Pin needs | Clock recovery | Helpers that matter | Long timeouts | External part |
|---|---|---|---|---|---|---|---|---|
| UART | ≤ 1 Mbaud (3 M stretch) | 50 (16.7) | Rate: ±3.3 % total mismatch (AN2141, both ends); RX phase 1/16 bit | 1 out + 1 in | Hard sync on start bit, sample mid-bit | Fractional baud gen, edge-triggered phase reset | Break detect | None (RS-232/485 need transceivers) |
| SPI controller | 6–12.5 MHz SCLK | 4–8 | Sample point vs IO round trip (~1 cycle) | 3 out + 1 in | None (own clock) | Side-set SCLK, variable-width shift, sample delay | None | None |
| SPI peripheral | ≤ 10 MHz SCLK (hyp.) | ≥ 5 | React to external edge within ½ SCLK | 3 in + 1 bidir | External clock edges | Edge wait, fast shift-in/out | CS timeout optional | None |
| I2C controller | 1 MHz (Fm+) | 50 | tSU;DAT 50 ns = 2.5 cycles | 2 bidir, open-drain | None; obeys stretching | Open-drain mode, readback, mismatch flag, glitch filter | SMBus 25–35 ms | Pull-ups |
| I2C target | 400 k–1 MHz | 50–125 | ACK driven before next SCL high | 2 bidir, open-drain | External SCL | START/STOP detect (edge on SDA while SCL high) | SMBus timeout | Pull-ups |
| JTAG probe | 6–12.5 MHz TCK | 4–8 | TDO sampled ≥ 1 cycle after falling TCK | 4 out + 1 in | None | Side-set TCK, 2-bit-wide shift (TMS+TDI), long shift chains | None | Level shift if VREF ≠ 3.3 V |
| SWD probe | 6–12.5 MHz SWCLK | 4–8 | OE switch on the turnaround cycle | 1 out + 1 bidir | None | Side-set OE, mixed-width shift, branch on ACK, parity | Retry on WAIT | Level shift if needed |
| PS/2 host/device | 10–16.7 kHz | ~3000 | ~5 µs (250 cycles) | 2 bidir, open-drain | External CLK (host mode) | Open-drain, readback, odd parity | 100 µs, 2 ms, 15 ms | 5 V ↔ 3.3 V level shifter |
| CAN classic | 500 k–1 Mb/s | 50–100 | 1 tq = 5 cycles at 1 Mb/s/10 tq; decision inside PS2 | 1 out + 1 in | Hard sync SOF, resync ±SJW | Bit-time DPLL, stuff/destuff (5), CRC-15, TX-vs-RX compare | Bus-off recovery (128×11 bits) | CAN transceiver (SN65HVD23x, TJA1051) |
| USB LS device | 1.5 Mb/s | 33.3 | ±1.5 % rate; mid-bit sampling; 6.5-bit turnaround (217 cycles) | 2 bidir (pair) + 1 out (pull-up) | DPLL on every edge | NRZI, stuff/destuff (6), CRC5/CRC16, J/K/SE0 pair states, EOP detect | 3 ms suspend, reset > 2.5 µs | Series R, 1.5 kΩ pull-up |
| 10BASE-T TX | 10 Mb/s | 5 (4 at 40 MHz) | Every half-bit exactly 50 ns: core must be 40 or 60 MHz | 2 out (pair) | None for TX | Manchester encoder, CRC-32, pair drive with 0 V idle, NLP timer | NLP every 16 ± 8 ms | Magnetics, resistors/filter, ideally a line buffer |
| 10BASE-T RX | 10 Mb/s | 4–5 | 50 ns vs 100 ns intervals with ±1 sample error | 1–2 in | PLL on mid-bit edges | Manchester decoder at ≥ 4× | Link-fail 50–150 ms | Magnetics + comparator with squelch |

## 2b. Target settings and roles

Planned demo settings: standard settings, both roles where the protocol has
two, and several protocols running concurrently.

| Protocol | Settings to demonstrate | Roles | Concurrency |
|---|---|---|---|
| UART | 9600, 19200, 38400, 57600, 115200 (plus 230400–1 M as reach), 8N1 and 8E1/8O1, optional RTS/CTS | Both ends (TX and RX); full duplex on two engines or one | Several UARTs at different bauds at once |
| SPI | Modes 0, 1, 2, 3; MSB/LSB first; 8/16/32-bit words; controller SCLK up to core/4 | Controller and peripheral | Controller on one engine talking to a peripheral on another (internal loopback) |
| I2C | 100 kHz and 400 kHz (1 MHz Fm+ as reach); 7-bit addressing; clock stretching; repeated START | Controller and target | Controller + target pair on the same bus, plus multi-controller arbitration test |
| Others | As in section 2 | Probe/host roles first (JTAG, SWD), host and device (PS/2), device (USB LS) | Any mix that fits the pin budget |

## 3. Shared controls (what every protocol reuses)

1. **Pin groups with per-pin mode.** Every protocol is a small set of
   adjacent-ish pins. Modes needed: push-pull out, input, **open-drain**
   (I2C, PS/2, SWD reset, SMBus), **pair** (USB J/K/SE0, Ethernet +/−/0).
2. **A bit clock that firmware can re-phase.** UART (start bit), CAN (SOF hard
   sync, bounded resync), USB LS (every edge), Ethernet (mid-bit edges) all use
   the same mechanism: a fractional phase accumulator producing bit or tq
   ticks, plus "reset phase on edge" with an optional ± limit. Synchronous
   protocols (SPI, JTAG, SWD, I2C controller) use the same accumulator as a
   clock generator.
3. **Shift registers with variable width and order.** 3 bits (SWD ACK) to 32
   (SWD data, JTAG); LSB first (UART, USB, CAN data, SWD, JTAG) or MSB first
   (SPI, I2C, CAN ID); 1, 2 or 4 bits per step (JTAG TMS+TDI, QSPI).
4. **Wait-on-condition.** Pin level, pin edge, combination (SDA edge while SCL
   high), bit tick, FIFO not empty, flag from another engine, timer expiry.
5. **Readback/mismatch.** I2C arbitration and stretching, CAN arbitration and
   bit error, PS/2 inhibit detection, SPI/I2C bus contention: all are "what I
   drive ≠ what the pin reads".
6. **Line-code helpers on the serial stream.** Bit stuffing (USB: insert after
   6 ones; CAN: after 5 equal bits), NRZI (USB), Manchester (Ethernet, also SWO
   and many RF/IR protocols), parity (UART, PS/2, SWD).
7. **CRC/LFSR.** CRC5 and CRC16 (USB), CRC-15 (CAN), CRC-32 (Ethernet), CRC-8
   (SMBus PEC), CRC-7/CRC-16 (SD cards). One programmable serial LFSR covers all.
8. **Timers for timeouts.** 2 ms–150 ms windows (PS/2, SMBus, USB suspend,
   Ethernet NLP) need ≥ 21–23-bit counts at 50 MHz, or a prescaler.
9. **FIFOs to the host.** All protocols stream bytes; the host should not have
   to service every byte in real time.

## 4. Key differences (what does not unify neatly)

- **Who owns the clock.** Controller-side synchronous protocols are easy
  (the engine makes the clock). Peripheral/target-side ones (SPI peripheral,
  I2C target, PS/2 host, JTAG/SWD target) must follow an external clock
  through a synchronizer, limiting external clock to roughly core/5
  **(hypothesis)**.
- **Reaction deadlines.** Most protocols tolerate microseconds of firmware
  latency between bytes. Three have hard **in-protocol** deadlines: CAN
  (decide within the same bit; drive ACK one bit after CRC), USB LS (respond
  within 6.5 bit times), I2C target (drive ACK in the next bit). These drive
  the helper hardware.
- **Exactness of rate.** UART and USB LS tolerate percent-level error;
  Ethernet tolerates 100 ppm **and** forbids cycle-level jitter, so it
  constrains the core clock itself.
- **Electrical layer.** The chip has 3.3 V digital IO only, no analog pads.
  UART/SPI/JTAG/SWD connect directly at 3.3 V; I2C and PS/2 need pull-ups
  (and PS/2 a 5 V shifter); USB LS needs a few passives; CAN needs a
  transceiver; Ethernet needs magnetics and, for receive, an analog front end.
  A bit-banging chip cannot remove these. They go on **Pmods**,
  documented with each example; open-drain emulation on chip is still
  needed (a Pmod pull-up does not make a pin open-drain).
- **Pin type.** Only the 8 `uio` pins have output enable on Tiny Tapeout, so
  open-drain and bidirectional protocols compete for them. Two-wire
  open-drain buses use 2 each: at most four of I2C/PS/2/USB pair at once.

## 5. Feasibility per protocol

Assumptions (all **hypotheses** until the first flow run; the clock is ours
to choose, the highest that closes timing and meets the protocols; the ETR
board's SDK can generate 40, 48 and 50 MHz exactly from its RP2350): engine clock = core clock
(no clock divider needed to issue one instruction per cycle); 2-flop input
synchronizer (2 cycles input latency); IO round trip similar to sky130 TT
(~20 ns); line-code and CRC helpers available where marked. Ratings: **A** =
easy with margin, **B** = feasible with the listed helpers, **C** = marginal or
needs exact clock / external parts, **X** = not feasible.

| Protocol | 25 MHz | 40 MHz | 50 MHz | Needs helpers | Tier |
|---|---|---|---|---|---|
| UART TX/RX ≤ 1 Mbaud | A | A | A | Fractional baud gen | Core |
| SPI controller ≤ f/4 | A | A | A | Side-set, sample delay | Core |
| SPI peripheral ≤ f/5 | B | B | B | Edge wait | Core (stretch: high-speed) |
| I2C controller ≤ 1 MHz | A | A | A | Open-drain, mismatch | Core |
| I2C target ≤ 400 kHz | B | A | A | START/STOP wait | Core |
| JTAG probe | A | A | A | Side-set | Firmware-only demo |
| SWD probe | A | A | A | Side-set OE | Firmware-only demo |
| PS/2 host/device | A | A | A | Open-drain | Firmware-only demo |
| CAN classic ≤ 500 kb/s | B | B | B | DPLL, stuff, CRC-15, mismatch | Stretch (go) |
| CAN classic 1 Mb/s | C | B | B | same | Stretch |
| USB LS device | C | B | B | NRZI, stuff, CRC5/16, pair | Stretch (go) |
| USB FS | X | X | C (needs exactly 48 MHz) | all | Not planned |
| 10BASE-T TX | X | B (exact 40 MHz) | X | Manchester, CRC-32, pair | Stretch (conditional) |
| 10BASE-T RX | X | C | C | Manchester decoder, analog front end | Not in v0 |
| CAN FD data phase | X | C | C | same as CAN | Not in v0 |

**Clock recommendation (hypothesis):** close timing at **50 MHz** (template
default) at the slow corner, and treat the operating clock as a runtime
choice: 50 MHz for general use, **exactly 40 MHz** when running Ethernet TX.
Tiny Tapeout's clock comes from the RP2350 on the ETR demo board; its SDK
picks a system clock with an even PWM divider and produces 40 MHz (80 / 2),
48 MHz and 50 MHz (100 / 2) exactly, so this costs nothing in silicon
(crystal tolerance not yet checked). If timing closes only at 25–33 MHz, USB LS becomes
marginal and Ethernet TX is lost; everything core stays.

## 6. Stretch plan (draft)

Priority order: core protocols in both roles and concurrently →
firmware-only protocols and new protocols the chip was not designed for →
USB LS → CAN → 10BASE-T TX.

| Protocol | Plan | Condition to confirm |
|---|---|---|
| JTAG, SWD, PS/2 | **Go**, firmware-only, as proof of post-fab programmability | None beyond core primitives |
| CAN classic | **Go** for ≤ 500 kb/s in simulation, 1 Mb/s if timing allows | CRC unit and mismatch flag fit area; transceiver on demo PCB later |
| USB LS device | **Go** (headline stretch) | First-flow Fmax ≥ 40 MHz; NRZI/stuff/CRC helpers fit |
| 10BASE-T TX | **Conditional** | Fmax ≥ 40 MHz with margin; 40 MHz clock from the RP2350 within 100 ppm (SDK generates it exactly; crystal ppm unverified); pad drive measured or external buffer accepted |
| 10BASE-T RX, USB FS, CAN FD | **Not planned for v0** | Revisit only if area and clock are left over |
