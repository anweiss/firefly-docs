# Firefly

Firefly is a system of battery-powered ESP32 wristbands that flash LEDs perfectly synced to DJ beats — like PixMob, but fully DIY. A coordinator taps into the Pioneer Pro DJ Link network, extracts precise beat timing, and broadcasts predictive timestamps over ESP-NOW so every wristband in the crowd flashes in unison within tens of microseconds of each other.

## Repositories

| Repository | Description |
|---|---|
| [`firefly`](https://github.com/anweiss/firefly) | Coordinator, dongle firmware, wristband firmware, shared protocol |
| [`prodjlink-rs`](https://github.com/anweiss/prodjlink-rs) | Rust implementation of the Pioneer Pro DJ Link protocol |
| [`beatbridge`](https://github.com/anweiss/beatbridge) | Bridge between Pro DJ Link and Ableton Link |
| [`ableton-link-rs`](https://github.com/anweiss/ableton-link-rs) | Rust bindings for the Ableton Link SDK |

## Documentation

- [Architecture](architecture.md) — full system design, signal flow, timing model, and component details
- [Protocol Spec](protocol.md) — v2 packet format, CRC algorithm, and framing
- [Project History](history.md) — design decisions, iterations, and lessons learned

## Current Status

**Stage 1 prototype complete.** The coordinator has 40 passing tests covering packet building, CRC validation, beat source state management, firmware simulators, and end-to-end integration. Protocol v2 is finalized with a 36-byte packet format transmitted over USB serial at 115200 baud and broadcast via ESP-NOW.
