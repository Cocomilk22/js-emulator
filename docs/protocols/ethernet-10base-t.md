# 10 Mb/s Ethernet (10BASE-T)

Part of the [protocol notes](README.md). Written 2026-10-11. Primary
source: IEEE Std 802.3, Clause 14 (10BASE-T MAU) and Clauses 3/4 (MAC frame);
Clause 28 for auto-negotiation. Stretch protocol, hardest in this set.

## How it works

**Signals.** Two twisted pairs over RJ45 (Cat3 or better, up to 100 m): TD±
(transmit pair, pins 1/2 on a DTE) and RD± (receive pair, pins 3/6). Full or
half duplex. Point to point to a switch/hub or another NIC (crossover or
auto-MDIX).

**Electrical level.**

| Parameter | Value | Source |
|---|---|---|
| TX peak differential output | 2.2–2.8 V into 100 Ω | 802.3 §14.3.1.2.1 (UNH-IOL MAU test suite) |
| TX waveform | Must fit the Clause 14 template (pre-distortion/filtering of harmonics) | §14.3.1.2 |
| RX minimum signal after 100 m | Peaks down to ≈ ±585 mV | Clause 14 receiver tests |
| RX squelch | Rejects low-amplitude/short pulses (PHY example: < 400 mV rejected) | §14.3.1.3.1; Microchip KSZ8081 datasheet |
| Isolation | Transformer (magnetics) required, 1:1, 1500 V isolation | Clause 14 (isolation requirement) |

**Line coding: Manchester.** Each 100 ns bit has a transition in the middle:
IEEE 802.3 convention, **1 = low→high, 0 = high→low** at mid-bit, so the
signalling rate is 20 Mbaud (50 ns half-bits). The clock is embedded; the
receiver recovers it with a PLL locked on the mid-bit transitions during the
preamble.

**Idle and link integrity.** Between frames the line is at 0 V differential.
Each frame ends with a positive start-of-idle (TP_IDL) pulse. During idle a
**normal link pulse (NLP)** is sent every **16 ms ± 8 ms**, a single positive
pulse ~100 ns wide (≥ 60 ns per the template). A receiver declares link fail
if it sees no frame or NLP for 50–150 ms. Auto-negotiation (Clause 28)
replaces each NLP with a **fast link pulse (FLP) burst** (17–33 pulses
62.5 µs apart) carrying a 16-bit capability word; a 10BASE-T-only node that
only sends NLPs is accepted by auto-negotiating partners via parallel
detection (forced 10 Mb/s half duplex).

**Frame (MAC).**

| Field | Bytes | Note |
|---|---|---|
| Preamble | 7 | 0x55 ×7 (alternating 1/0, lets the RX PLL lock) |
| SFD | 1 | 0xD5 |
| Destination, source MAC | 6 + 6 | |
| EtherType / length | 2 | |
| Payload | 46–1500 | pad to 46 |
| FCS | 4 | **CRC-32**, poly 0x04C11DB7, LSB-first (reflected), init 0xFFFFFFFF, final complement |

Minimum frame 64 bytes (dest through FCS), max 1518. Interpacket gap ≥ 96 bit
times (9.6 µs). Half duplex uses CSMA/CD (collision = receiving while
transmitting; jam 32 bits; truncated binary exponential backoff in 51.2 µs
slots).

**Timing parameters.**

| Parameter | Value |
|---|---|
| Bit rate | 10 Mb/s ± 0.01 % (± 100 ppm) |
| Half-bit (baud) period | 50 ns |
| NLP period / width | 16 ± 8 ms / ~100 ns |
| IPG | 9.6 µs |
| Link-fail timer | 50–150 ms |

```text
bit:         1       0       1       1       0      (100 ns each)
half-bits:   L H     H L     L H     L H     H L    (50 ns each; L/H = TD+ minus TD- negative/positive)
note: equal neighbouring bits (1,1) need an extra edge at the bit boundary (H->L between them);
      differing neighbours (1,0) do not. Idle after TP_IDL is 0 V; one ~100 ns
      positive NLP every 16 +/- 8 ms keeps the link up.
```

## Applications
Legacy industrial and building automation, embedded web servers (ENC28J60
class), 10BASE-T1S/T1L automotive and industrial single-pair (newer,
different PHYs), test equipment. For an emulator: sending UDP telemetry or a
"hello" frame from a pin pair is a striking demo; Pico-10BASE-T showed
transmit-only UDP from RP2040 PIO with a few resistors and a pulse transformer.

## Emulator requirements
- **TX clock must be exact:** ±100 ppm and every half-bit 50 ns. A fractional
  divider's 1-cycle jitter is 20–25 ns, i.e. ~half a half-bit, which breaks
  the waveform template. So the **core clock must be an integer multiple of
  20 MHz**: 40 MHz (2 cycles per half-bit) or 60 MHz (3). At the 50 MHz
  template default it is **not feasible**. The TT demo board generates the
  chip clock from its MCU (RP2350 on the ETR board). Its MicroPython SDK
  picks a system clock with an even PWM divider and produces **exactly
  40 MHz** (80 MHz / 2, 50 % duty) (validated from the SDK source, not on a
  board); whether the board crystal keeps it within ±100 ppm is **not
  verified**.
- **TX datapath:** shift register → CRC-32 (on the fly) → Manchester encoder →
  differential pin pair (TD+ = d, TD− = not d, both low at idle). At 40 MHz a
  new bit is needed every 4 cycles: firmware cannot do per-bit work, the
  encoder and serializer must be hardware, firmware only refills 16–32-bit
  words. NLP generation needs a ms timer and a 4-cycle pulse.
- **RX:** needs a differential receiver with squelch (comparator or an RS-485/
  LVDS receiver, through the magnetics) to make a logic signal, then
  Manchester decoding at 4 samples per bit (40 MHz). Distinguishing a short
  (50 ns = 2 samples) from a long (100 ns = 4 samples) interval with ±1 sample
  synchronizer uncertainty is marginal. **Hypothesis: RX is not reliably
  feasible below ~80 MHz sampling**; it is a research item, not a v0 goal.
- **Higher layers:** ARP/IP/UDP/ICMP header and checksum construction is
  firmware/host work; only the FCS needs line-rate hardware.

| Core clock | Cycles per bit | Cycles per half-bit | TX feasible? |
|---|---|---|---|
| 25 MHz | 2.5 | 1.25 | No |
| 40 MHz | 4 | 2 | Yes, with hardware Manchester + CRC32 **(hypothesis)** |
| 50 MHz | 5 | 2.5 | No (non-integer half-bit) |
| 60 MHz | 6 | 3 | Yes if timing closes (unlikely target) |

**External hardware.** Mandatory pulse transformer (magnetics, e.g. an RJ45
with integrated magnetics), series/termination resistors and ideally a small
low-pass network to meet the template. TX from two 3.3 V CMOS pins in
anti-phase gives ~6.6 V peak-to-peak differential open-circuit; with a 100 Ω
load and the pad's drive limit (sky130 pads 4 mA; **IHP TT pad drive not
verified**) the amplitude will be below spec unless an external buffer
(74LVC-class) drives the transformer. Hobby designs work over short cables
out of spec. RX needs a comparator front end. Alternative: an external PHY in
**MII mode** (10 Mb/s MII = 4-bit nibbles at 2.5 MHz, trivial) or RMII
(50 MHz reference clock, 2 bits per edge, data repeated ×10 at 10 Mb/s), at the
cost of no longer driving the wire directly.

**Feasibility:** TX-only stretch at 40 MHz with hardware Manchester and CRC
helpers; RX not in v0. Go/no-go after the first flow run
gives Fmax and pad behaviour.

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

The pulse transformer (RJ45 with magnetics), series resistors and filter,
and ideally a line driver go on a Pmod. Manchester
coding and CRC-32 are done on chip by the helper stage. Pad drive into the transformer is
unknown without CMOS5L pad characterization.

## References
- IEEE 802.3 (free via IEEE GET program): https://ieeexplore.ieee.org/browse/standards/get-program/page/series?id=68
- UNH-IOL 10BASE-T MAU test suite (Clause 14 amplitudes, templates): https://www.iol.unh.edu/sites/default/files/testsuites/ethernet/CL14_MAU/MAU_Test_Suite_v5.4.pdf
- UNH-IOL 10BASE-T overview (LTP 16 ± 8 ms, 2.2–2.8 V): https://www.iol.unh.edu/sites/default/files/knowledgebase/ethernet/10basetmau.pdf
- ctrl+src, *An overview of Ethernet 10BASE-T*: https://ctrlsrc.io/posts/2023/niccle-ethernet-10base-t-overview/
- Microchip KSZ8081 PHY datasheet (squelch 400 mV example): https://ww1.microchip.com/downloads/en/DeviceDoc/10BASE-T-100BASE-TX-Physical-Layer-Transceiver-DS00002264B.pdf
- Pico-10BASE-T (RP2040 PIO transmit-only): https://github.com/kingyoPiyo/Pico-10BASE-T
- rp2040-rmii (RMII PHY path): https://github.com/essele/rp2040-rmii
