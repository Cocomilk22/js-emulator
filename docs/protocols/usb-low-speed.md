# USB 1.x/2.0 low-speed (1.5 Mb/s)

Part of the [protocol notes](README.md). Written 2026-10-11. Primary source:
*Universal Serial Bus Specification Revision 2.0* (USB-IF, 2000), chapter 7
(electrical) and chapter 8 (protocol). Stretch protocol.

## How it works

**Signals.** Two data wires D+ and D− (plus VBUS 5 V and GND), half duplex,
host-scheduled (the host polls; a device only speaks in response). Tiered star
topology through hubs. A low-speed (LS) device is identified by a **1.5 kΩ
pull-up on D−** (full-speed devices pull up D+); the host has 15 kΩ
pull-downs on both lines.

**Electrical level (LS/FS signalling, USB 2.0 §7.1 and Table 7-7).**

| Parameter | Value |
|---|---|
| Driver VOH | 2.8–3.6 V (driving into 15 kΩ to GND) |
| Driver VOL | 0–0.3 V |
| Differential receiver sensitivity | 200 mV, common mode 0.8–2.5 V |
| Single-ended receiver thresholds | VIL 0.8 V, VIH 2.0 V |
| LS rise/fall time | 75–300 ns (10–90 %) |
| Pull-up | 1.5 kΩ on D− to 3.0–3.6 V (LS device) |

Line states: **J** and **K** (differential 1 and 0, polarity depends on speed:
for LS, J = D− high, D+ low), **SE0** (both low, used for EOP and reset),
**SE1** (both high, illegal). Idle = J (from the pull-up).

**Line coding.**
- **NRZI:** a 0 is sent as a transition (J↔K), a 1 as no transition.
- **Bit stuffing:** after **six consecutive 1s** the transmitter inserts a 0 (a
  forced transition), so the receiver's clock recovery sees an edge at least
  every 7 bit times. The receiver strips it; seven 1s in a row is a stuff error.
- **SYNC:** 8 bits `00000001` (NRZI KJKJKJKK), used to lock the receiver DPLL.
- **EOP:** SE0 for 2 bit times, then J for 1 bit.
- LSB first.

**Packets** (USB 2.0 ch. 8): SYNC, PID (4 bits + 4-bit complement), payload,
CRC, EOP.

| Packet | Fields | Check |
|---|---|---|
| Token (IN, OUT, SETUP) | ADDR 7, ENDP 4 | **CRC5**, poly x^5 + x^2 + 1, init all 1s, sent inverted |
| Data (DATA0/DATA1) | 0–8 bytes payload for LS | **CRC16**, poly x^16 + x^15 + x^2 + 1, init all 1s, sent inverted |
| Handshake (ACK, NAK, STALL) | PID only | PID complement |

**Timing parameters.**

| Parameter | Value | Source |
|---|---|---|
| LS bit rate | 1.5 Mb/s ± 1.5 % (15 000 ppm) | USB 2.0 ch. 7 (signalling rates) |
| Bit time | 666.7 ns | |
| Device turnaround (EOP of host packet → SYNC of response) | 2–6.5 bit times at the device (7.5 with captive cable) | USB 2.0 §7.1.18.1 |
| Host response timeout | 16–18 bit times (10.7–12 µs at LS) | USB 2.0 ch. 7 (bus turnaround timer) |
| Keep-alive (LS) | Host sends an LS EOP once per 1 ms frame | USB 2.0 ch. 11 (hub) |
| Suspend | Device suspends after 3 ms of idle | USB 2.0 ch. 7 |
| Reset | Host drives SE0 ≥ 10 ms; device must detect reset after SE0 > 2.5 µs | USB 2.0 ch. 7 |
| LS max payload | 8 bytes per transaction (control/interrupt only, no bulk/iso) | ch. 5 |

Compared to full speed (12 Mb/s ± 0.25 %), LS's ±1.5 % tolerance is what makes
it bit-bangable without a crystal-accurate clock.

```text
LS packet: SYNC | PID | ... | CRC | EOP           (J = D- high for LS)
NRZI data:   0 0 0 0 0 0 0 1 | ...
line state:  K J K J K J K K | ...  SE0 SE0 J
D+ ____/‾\_/‾\_/‾\_/‾‾‾‾ ... ____________
D- ‾‾‾‾\_/‾\_/‾\_/‾\____ ... ________/‾‾‾  (both low = SE0 = EOP)
```

## Applications
Keyboards, mice, game controllers, UPS and HID sensors, barcode scanners,
cheap dongles, MCU bootloaders built on bit-banged USB (e.g. Micronucleus on
ATtiny with V-USB). Emulator use: a USB HID device for a test rig, a USB
**sniffer** for LS traffic, fault injection (malformed packets, timing
violations) for testing hosts.

## Emulator requirements
- **Clock:** 1.5 MHz bit clock with ≤ 1.5 % error. 50 MHz / 1.5 MHz = 33.33
  cycles per bit: a fractional divider gives 0.004 % rate error with 1-cycle
  (20 ns, 3 % of a bit) jitter, well inside budget **(hypothesis until the
  clock plan is fixed)**.
- **RX clock recovery:** asynchronous; DPLL that re-phases on every
  transition (NRZI + stuffing guarantee one at least every 7 bits). Sample at
  mid-bit. This is the same "resync on edge" primitive as UART/CAN, applied
  on every edge instead of only at a start bit.
- **Real-time line decoding** (the expensive part in software): NRZI decode,
  bit de-stuffing, SE0/EOP detection, CRC5/CRC16 check. In pure firmware this
  is ~8–15 instructions per bit; V-USB manages it on an AVR at 12 MHz
  (8 cycles/bit) only with hand-cycle-counted assembly and CRC computed after
  the packet. A **hardware NRZI + stuff/destuff + CRC pipeline** between the
  pin and the shift register reduces firmware to byte-level work.
- **Turnaround:** a device must answer within 6.5 bit times ≈ 4.3 µs ≈ 216
  cycles at 50 MHz. Decoding the token, deciding, and starting SYNC must fit;
  practical approach (as V-USB does) is to **pre-build** responses (ACK/NAK,
  next DATA packet) in a TX FIFO before the token arrives.
- **Pin pair control:** D+ and D− driven as a pair with states J, K, SE0, and
  released (Hi-Z) for receive. Both on `uio` pins, switching OE at
  turnaround. Receive with two single-ended inputs (SE0 detection) and treat
  D+ (or D−) as the data line, as V-USB does.
- **Higher layers** (enumeration, descriptors, control transfers) belong in
  host software or a larger firmware image; they are not timing-critical
  apart from the turnaround above.

| Core clock | Cycles per LS bit | Cycles in 6.5-bit turnaround |
|---|---|---|
| 25 MHz | 16.7 | 108 |
| 40 MHz | 26.7 | 173 |
| 50 MHz | 33.3 | 217 |

**External hardware.** TT IO at 3.3 V meets VOH 2.8–3.6 V if pads drive the
light USB load. Add: series resistors on D+/D− (V-USB uses 68 Ω), the 1.5 kΩ
pull-up on D− (to 3.3 V, switchable through a spare output for soft
connect), and **5 V protection** if the host side could ever float to 5 V
(TT inputs not 5 V tolerant). No PHY required for LS. Pad slew vs the 75 ns
minimum rise time is **not verified** for IHP pads (a too-fast edge may need
an RC; unlikely to matter on a short cable).

**Feasibility:** feasible with line-coding helpers; risky without them.
Device-side LS is the recommended stretch target; host-side LS (keep-alive
generation, SOF-less frame timing) is a further extension. Full speed
(12 Mb/s ± 0.25 %) is **not** recommended: 4.17 cycles/bit at 50 MHz needs an
exact 48 MHz clock (4 cycles/bit) and 1-cycle-accurate edge handling.

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

The series resistors, the 1.5 kΩ pull-up on D− and ESD protection go on a
Pmod. Line coding, CRC and clock recovery are done on chip by the NRZI/stuffing/CRC
helper stage and the DPLL re-phase. Without pad characterization, meeting
the USB LS output levels and rise/fall times is a **hypothesis** to check on
silicon.

## References
- USB 2.0 specification (USB-IF): https://www.usb.org/document-library/usb-20-specification
- USB 2.0 Electrical Compliance Test Specification v1.07: https://www.usb.org/sites/default/files/USB%202%200%20Electrical%20Compliance%20Specification%28v1.07%29.pdf
- V-USB (software-only LS USB on AVR): https://www.obdev.at/products/vusb/index.html
- Pico-PIO-USB (FS USB on RP2040 PIO): https://github.com/sekigon-gonnoc/Pico-PIO-USB
- Tektronix, *Troubleshooting USB 2.0 Problems with Your Oscilloscope* (SYNC/EOP/PID summary): https://download.tek.com/document/Troubleshooting-USB2%5FApp-Note%5F55W-61092-1.pdf
