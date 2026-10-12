# PS/2 (IBM keyboard/mouse interface)

Part of the [protocol notes](README.md). Written 2026-10-11. Sources: IBM
*Personal System/2 Hardware Interface Technical Reference* (1988, not freely
online); Adam Chapweske's widely used protocol write-ups (archived).

## How it works

**Signals.** CLK and DATA, both **open-collector/open-drain** with pull-ups
(device side typically 5–10 kΩ to +5 V). Half duplex, bidirectional,
point to point. The **device (keyboard/mouse) always generates the clock**, in
both directions; the host can only inhibit the bus (hold CLK low) or request
to send.

**Electrical level.** +5 V logic, open-collector; the host may only pull low.
3.3 V hosts need level shifting (TT inputs are not 5 V tolerant).

**Frame.** 11 bits: start (0), 8 data bits LSB first, **odd parity**, stop
(1). Host-to-device frames add an ACK bit from the device (12 clocks).

**Device-to-host.** Device drives DATA and then pulses CLK; the host reads
DATA on the **falling edge** of CLK (data is stable while CLK is low).

**Host-to-device.** Host pulls CLK low for ≥ 100 µs (inhibit), pulls DATA low
(request-to-send, acts as start bit), releases CLK. The device then generates
clocks; the host changes DATA while CLK is low and the device samples on the
**rising** edge. After stop the device pulls DATA low as ACK.

| Parameter | Value |
|---|---|
| Clock frequency | 10–16.7 kHz (clock high/low 30–50 µs each) |
| Data setup/hold around clock edge | ≥ 5 µs |
| Host inhibit before RTS | ≥ 100 µs |
| Device must start clocking after RTS | ≤ 15 ms |
| Host-to-device packet complete | ≤ 2 ms |
| Keyboard scan codes | Set 2 by default (e.g. make 0x1C 'A', break 0xF0 0x1C) |
| Mouse packet | 3 bytes (buttons/overflow/sign, dX, dY), 4 with IntelliMouse wheel |

```text
device -> host (host samples DATA on falling CLK)
CLK  ‾‾‾\__/‾‾\__/‾‾\__/ ... ‾‾\__/‾‾\__/‾‾‾‾
DATA ‾‾\_____X_D0_X_D1_X ... _X_Pa_X_Stop‾‾‾‾
       start
```

## Applications
Legacy keyboards and mice, KVM switches, retro computing, industrial
keypads and barcode wedges, and still-present PS/2 ports on some motherboards
(NKRO keyboards). For an emulator: host mode reads a keyboard; device mode
**emulates** a keyboard/mouse (scripted input injection to test equipment).

## Emulator requirements
- **Open-drain on two pins** (same mechanism as I2C), `uio` pins.
- **Externally clocked receive** in host mode: `wait` for CLK falling edge,
  sample DATA. The clock is very slow (≥ 30 µs phases = ≥ 1500 cycles at 50 MHz),
  so timing is trivial; a glitch filter is useful because PS/2 cables ring.
- **Device mode:** the engine generates a 10–16.7 kHz clock on an open-drain
  pin and must detect host inhibit (CLK held low by the host while the
  engine released it): the same "driven vs sensed" mismatch primitive as I2C
  clock stretching.
- **Timeouts:** 100 µs, 2 ms, 15 ms: needs a timer reaching ≥ 750 000 cycles
  at 50 MHz (≥ 20 bits) or a prescaler.
- **Helpers:** odd parity (XOR-reduce).

**External hardware:** bidirectional 5 V ↔ 3.3 V level shifter
(BSS138-based or TXS0102-class open-drain-friendly). Some devices work at
3.3 V, but that is out of spec and not relied on.

**Feasibility:** trivial timing; the only hardware requirement is open-drain
with readback. Good firmware-only demo of a protocol "added after
fabrication".

## External parts (Pmods) and board notes

The Tiny Tapeout IHP chip has 3.3 V digital IO only (24 user IO plus `clk`
and `rst_n`; no analog pads). External parts go on Pmod boards, documented
with each example program. Pad drive, input thresholds and speed for the
CMOS5L chip are not published, so pad-dependent numbers in this document are
hypotheses. The current Tiny Tapeout demo board (ETR) uses an RP2350 MCU.

The 5 V ↔ 3.3 V bidirectional level shifter (with the bus pull-ups) goes on a
Pmod; open-drain emulation on chip is still needed.

## References
- Adam Chapweske, *The PS/2 Mouse/Keyboard Protocol* (archived):
  https://web.archive.org/web/20180302005138/http://www.computer-engineering.org/ps2protocol/
- IBM PS/2 Hardware Interface Technical Reference, Keyboard and Auxiliary
  Device Controller sections (1988, offline; not consulted directly).
