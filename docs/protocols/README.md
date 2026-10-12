# Protocol notes

Preliminary engineering notes, written 2026-10-11, on the protocols this
emulator targets: what a bit-banging protocol engine on Tiny Tapeout IHP
(CMOS5L, 6x4 tiles) must provide for each one.

Each file covers how the protocol works (wires, electrical level, line
coding, framing, timing numbers), a timing sketch, applications, the
**emulator requirements** (sampling, clock recovery, pin direction,
timeouts, helpers, cycles per bit at 25/40/50 MHz) and the external parts
that go on Pmods.

Labels used throughout: numbers with a cited source are taken from that
source; **(hypothesis)** marks estimates that depend on the core clock, pad
behaviour or area, none of which has been measured yet (first synthesis and
place-and-route runs pending).

| Protocol | File | Tier | Summary |
|---|---|---|---|
| UART | [uart.md](uart.md) | Core | Trivial; first RTL target |
| SPI | [spi.md](spi.md) | Core | Easy; controller limited by IO round trip, not by the core |
| I2C / SMBus | [i2c.md](i2c.md) | Core | Easy; needs open-drain emulation with readback |
| JTAG | [jtag.md](jtag.md) | Firmware-only | Easy; shows a protocol added purely in firmware |
| SWD | [swd.md](swd.md) | Firmware-only | Easy-moderate; needs fast OE turnaround |
| PS/2 | [ps2.md](ps2.md) | Firmware-only | Trivial timing; needs 5 V level shifting |
| CAN (classic) | [can.md](can.md) | Stretch | Moderate; needs bit-time DPLL, stuffing, CRC-15, in-bit compare, transceiver |
| USB low speed | [usb-low-speed.md](usb-low-speed.md) | Stretch | Feasible with NRZI/stuff/CRC helpers; 6.5-bit turnaround is the hard part |
| 10BASE-T Ethernet | [ethernet-10base-t.md](ethernet-10base-t.md) | Stretch (conditional) | TX-only at exactly 40 MHz with hardware Manchester + CRC-32; RX not planned |

Related documents:

- [../protocol-comparison.md](../protocol-comparison.md): comparison tables,
  requirement matrix, feasibility per protocol and clock.
- [../architecture.md](../architecture.md): primitives, engines, ISA sketch,
  host link and configurations.
- [../roadmap.md](../roadmap.md): milestones to the 2027-01-18 deadline.

## Platform facts used by all files

Checked 2026-10-11 unless noted.

- Tiny Tapeout user IO: `clk`, `rst_n`, `ena`, 8 inputs `ui_in`, 8 outputs
  `uo_out`, 8 bidirectional `uio_in`/`uio_out`/`uio_oe` (active-high OE)
  ([TT GPIO spec](https://tinytapeout.com/specs/gpio/),
  [IHP template project.v](https://github.com/TinyTapeout/ttihp-verilog-template)).
  All IO is 3.3 V digital; there are no analog pads.
- The IHP template's default `CLOCK_PERIOD` is 20 ns (50 MHz) in
  `src/config.json`. TT's published pad limits (33 MHz output, 66 MHz input,
  4 mA, not 5 V tolerant, ~20 ns mux round trip) are for **sky130** pads;
  equivalent numbers for the IHP/CMOS5L chip are not published and are
  treated as unknown.
- The demo board clock comes from the on-board MCU, configurable 1 Hz to
  66.5 MHz ([TT clock spec](https://tinytapeout.com/specs/clock/)). The
  current ETR board uses an **RP2350**
  ([TT PCB-ETR page](https://tinytapeout.com/specs/pcb-etr/); the
  [pinout page](https://tinytapeout.com/specs/pinouts/) still says RP2040).
  Its MicroPython SDK
  ([tt-micropython-firmware](https://github.com/TinyTapeout/tt-micropython-firmware))
  generates 40, 48 and 50 MHz exactly (system clock / 2); crystal tolerance
  not checked.
- Demo settings we aim for: common UART baud rates, SPI modes 0–3, I2C
  100 and 400 kHz, controller and target roles, several protocols running
  concurrently.
