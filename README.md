# Firefly

Firefly is a system of battery-powered ESP32 wristbands that flash LEDs perfectly synced to DJ beats — like PixMob, but fully DIY. A coordinator (host laptop or all-in-one ESP32-C3 device) taps into the Pioneer Pro DJ Link network, extracts precise beat timing, and broadcasts predictive timestamps over ESP-NOW so every wristband in the crowd flashes in unison within tens of microseconds of each other.

## Repositories

| Repository | Description |
|---|---|
| [`firefly`](https://github.com/anweiss/firefly) | All-in-one firmware (`firefly-fw/`), legacy host coordinator + dongle, wristband firmware, shared wire protocol |
| [`prodjlink-rs`](https://github.com/anweiss/prodjlink-rs) | Rust implementation of the Pioneer Pro DJ Link protocol |
| [`beatbridge`](https://github.com/anweiss/beatbridge) | Bridge between Pro DJ Link and Ableton Link |
| [`ableton-link-rs`](https://github.com/anweiss/ableton-link-rs) | Rust bindings for the Ableton Link SDK |

## Documentation

- [Architecture](architecture.md) — full system design, both deployment paths (all-in-one + host), signal flow, timing model, component details
- [Protocol Spec](protocol-v2.md) — v2 packet format, CRC algorithm, framing, broadcast rates per path
- [Project History](project-history.md) — design decisions, iterations, lessons learned
- [Original planning notes](planning.md) — initial brainstorm and stage roadmap (historical)

## Current Status

**Stage 1 prototype: complete.** Stage 3 lite (single-chip all-in-one) **landed early** as `firefly-fw/` — the host coordinator + USB-serial dongle pair has been collapsed into a single XIAO ESP32-C3 running Rust on ESP-IDF that joins the Wi-Fi network, listens for Pro DJ Link directly, runs the BeatSourceState PLL, and broadcasts ESP-NOW to wristbands at 100 Hz, with an SSD1306 OLED status display. The legacy host path remains for Ableton Link peer participation (the Link C++ library doesn't cross-compile to RISC-V yet).

Validated end-to-end with a synthetic DJ Link sender at 124 BPM for 60 s — wristband locked on `cdj=yes / playing=yes` for the full burst with ~4 % ESP-NOW drop rate, smoothed over by the wristband's PLL anchor.

Wire protocol v2 is finalized: 36-byte packets, non-reflected CRC-8 (poly 0x31), little-endian. Identical packets travel over USB serial (host path) and ESP-NOW (both paths). The coordinator has 40 passing tests including DongleSim + WristbandSim end-to-end pipelines.
