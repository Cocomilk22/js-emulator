# Documentation

**Preliminary plans.** Everything here is a draft written at the start of
the project (2026-10-11) and open for discussion: architecture, numbers and
dates will change as synthesis, place-and-route and verification results
come in.

- [architecture.md](architecture.md): draft architecture: protocol primitives, engines, ISA sketch, memories, host link, pin map, configurations, area estimate, verification, toolchain, open questions.
- [roadmap.md](roadmap.md): draft milestones, exit criteria and protocol landing plan to the 2027-01-18 deadline.
- [protocol-comparison.md](protocol-comparison.md): comparison tables, requirement matrix and feasibility per protocol at assumed clocks.
- [protocols/README.md](protocols/README.md): index of per-protocol notes (UART, SPI, I2C, JTAG, SWD, PS/2, CAN, USB low speed, 10BASE-T).
- [ihp-sram-notes.md](ihp-sram-notes.md): notes on integrating IHP SRAM macros in a Tiny Tapeout CMOS5L tile.

Competition: [Jane Street protocol emulator ASIC competition](https://blog.janestreet.com/protocol-emulator-asic-competition/).
Platform: [Tiny Tapeout](https://tinytapeout.com/) on IHP 130 nm CMOS5L.
