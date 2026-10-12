# CAN (Controller Area Network), classic CAN 2.0 with a CAN FD note

Part of the [protocol notes](README.md). Written 2026-10-11. Primary
sources: ISO 11898-1:2015 (data link layer, classic CAN and CAN FD),
ISO 11898-2:2016 (high-speed physical layer), Bosch *CAN Specification 2.0*
(1991).

## How it works

**Signals.** At the controller: TXD (output) and RXD (input) to a
**transceiver**. On the bus: a differential pair CAN_H/CAN_L, terminated
with 120 Ω at both ends of a linear bus. Multi-master, half duplex,
broadcast; any node may start when the bus is idle.

**Electrical level (ISO 11898-2).** Two bus states: **dominant** (logic 0,
CAN_H ≈ 3.5 V, CAN_L ≈ 1.5 V, ≈ 2 V differential) and **recessive** (logic 1,
both ≈ 2.5 V, ≈ 0 V differential, not driven). Dominant always wins: the bus
is a wired-AND, like I2C, but implemented by the transceiver. The
transceiver loops the bus state back on RXD, so a node always sees the
resolved bus value of its own bit.

**Line coding.** NRZ with **bit stuffing**: after **5 consecutive identical
bits** the transmitter inserts one bit of opposite polarity, from SOF
through the CRC sequence. Six identical bits there is a **stuff error**.
The stuffing guarantees resynchronization edges.

**Frame (classic, base format, 11-bit ID):**

| Field | Bits | Note |
|---|---|---|
| SOF | 1 | dominant; hard sync point |
| Identifier | 11 (29 in extended format, with SRR/IDE) | **arbitration field**, MSB first |
| RTR | 1 | dominant = data frame |
| IDE, r0 | 1 + 1 | |
| DLC | 4 | 0–8 data bytes |
| Data | 0–64 | |
| CRC | 15 | **CRC-15**, poly x^15+x^14+x^10+x^8+x^7+x^4+x^3+1 (0x4599), init 0 |
| CRC delimiter | 1 | recessive |
| ACK slot | 1 | transmitter sends recessive; **every receiver that got a valid frame drives dominant** |
| ACK delimiter | 1 | recessive |
| EOF | 7 | recessive |
| IFS (intermission) | 3 | recessive |

**Arbitration.** Bitwise, non-destructive: while sending the identifier each
transmitter compares the bit it sends with the bit it reads back; a node that
sends recessive and reads dominant has **lost** and becomes a receiver within
the same bit. Lowest identifier wins. This needs the read-back to happen
**inside the bit**, after the round-trip delay through transceivers and bus.

**Error handling.** Five error types (bit, stuff, CRC, form, ACK). An
error-active node signals an error with an **error frame** (6 dominant bits,
deliberately violating stuffing). Counters TEC/REC; error passive at ≥ 128,
**bus off** at TEC ≥ 256. Recovery from bus off: 128 occurrences of 11
recessive bits.

**Bit timing.** A bit is divided into **time quanta (tq)**: Sync_Seg (1 tq,
where edges are expected), Prop_Seg, Phase_Seg1, Phase_Seg2; 8–25 tq per bit.
The **sample point** sits between PS1 and PS2, typically 75–87.5 % (CANopen
and CiA recommend 87.5 %). **Hard synchronization** at SOF; **resynchronization**
on recessive-to-dominant edges shifts the sample point by at most **SJW**
(synchronization jump width, 1–4 tq in classic CAN). Prop_Seg must cover twice
the one-way delay (bus + transceiver loop delay), which is why speed falls
with bus length.

| Parameter | Value |
|---|---|
| Bit rate (classic) | up to 1 Mb/s; common 125, 250, 500 kb/s, 1 Mb/s |
| Bus length vs rate (rule of thumb) | ~40 m at 1 Mb/s, ~100 m at 500 kb/s, ~500 m at 125 kb/s |
| Oscillator tolerance | up to ~1.58 % depending on bit timing (Bosch formula); 0.5 % typical guidance |
| CAN FD | data phase up to 5 Mb/s typical (8 Mb/s possible), 64-byte payload, CRC-17/CRC-21, stuff-bit count field |

```text
node A sends ID 0x123 (0b001_0010_0011), node B sends ID 0x125 (0b001_0010_0101)
bit:      SOF 10  9  8  7  6  5  4  3  2  1  0
A TX:      0   0  0  1  0  0  1  0  0  0  1  1
B TX:      0   0  0  1  0  0  1  0  0  1   (B sent 1, read 0: lost, now receiver)
bus:       0   0  0  1  0  0  1  0  0  0  1  1   -> A (lower ID) wins
one bit:  |sync|  prop  | phase1 |^sample point| phase2 |
```

## Applications
Automotive (powertrain, body, OBD-II diagnostics at 500 kb/s), industrial
automation (CANopen, DeviceNet), agriculture (ISO 11783/ISOBUS), trucks
(SAE J1939 at 250 kb/s), medical devices, robotics and drones (DroneCAN,
servo buses), elevators, marine (NMEA 2000). Emulator use: bus sniffing with
timestamps, ECU simulation, fault injection (error frames, bit errors,
arbitration attacks) for testing.

## Emulator requirements
- **Bit-time engine with tq granularity:** a phase counter that runs per tq,
  places the sample point, supports hard sync on SOF and **bounded resync**
  (± SJW) on recessive→dominant edges. This is the same DPLL primitive as
  UART/USB with a bounded correction.
- **In-bit decision:** at the sample point the engine must compare TX vs RX
  (arbitration loss and bit error) and decide before the next bit is driven.
  With firmware doing it, the budget is PS2 (e.g. 2 tq). A hardware
  **driven-vs-sensed comparator flag** sampled at the sample point (shared
  with I2C) makes it a single conditional branch.
- **Real-time destuffing and stuffing** (5-bit run rule) and **CRC-15** on the
  destuffed stream, so that the receiver knows the CRC is good in time to
  drive the **ACK slot** one bit after the CRC delimiter. Without hardware
  CRC, firmware has ~1 µs per bit at 1 Mb/s, i.e. 50 cycles at 50 MHz, which
  is tight but possible for a serial CRC update; with a CRC unit it is free.
- **Error frame generation** (6 dominant bits) and counters in firmware.
- **Acceptance filtering** (match ID against masks) in firmware or host.

| Core clock | Cycles per bit at 500 kb/s | 1 Mb/s | 1 Mb/s with 10 tq |
|---|---|---|---|
| 25 MHz | 50 | 25 | 2.5 cycles/tq (not integer; use 25 tq of 1 cycle or 5 tq of 5) |
| 50 MHz | 100 | 50 | 5 cycles/tq |

**Pins:** TXD on any output, RXD on any input (no bidir pin needed: the
transceiver does the wired-AND). Optional STBY/S output to the transceiver.

**External hardware:** a **CAN transceiver is mandatory**: 3.3 V parts such
as TI SN65HVD230/231/232 or NXP TJA1051T/3 (with VIO pin). Its loop delay
(TXD→bus→RXD, order of 100–250 ns depending on part; check datasheet) must
be included in Prop_Seg.

**Feasibility:** moderate. Classic CAN up to 500 kb/s–1 Mb/s is feasible with
the shared helpers (bit-time DPLL, stuffing, CRC unit, mismatch flag). CAN FD
data phase at 2–5 Mb/s gives 10–25 cycles per bit at 50 MHz: research item,
not v0.

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

The transceiver (e.g. a 3.3 V SN65HVD230 class part) and the bus termination
go on a Pmod; the chip sees only TXD/RXD at 3.3 V.

## References
- ISO 11898-1:2015: https://www.iso.org/standard/63648.html
- Bosch CAN Specification 2.0 (1991): http://esd.cs.ucr.edu/webres/can20.pdf
- TI SLLA270, *Controller Area Network Physical Layer Requirements*: https://www.ti.com/lit/an/slla270/slla270.pdf
- TI SN65HVD230 datasheet: https://www.ti.com/lit/ds/symlink/sn65hvd230.pdf
- CiA, CAN bit timing recommendations: https://www.can-cia.org/can-knowledge/
