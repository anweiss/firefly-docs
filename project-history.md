# Firefly Project History

## Origins

The project started with an Ableton Link Rust port (ableton-link-rs) and a brainstorming session about IoT projects for a home DJ setup with Pioneer CDJ-3000s and a DJM-A9. The wristband idea — battery-powered ESP32 devices that flash LEDs perfectly synced to beats, like PixMob but DIY — was the clear winner.

## Architecture Decisions

- **Coordinator-plus-wristbands model**: one coordinator runs Link/DJ Link, broadcasts to wristbands via ESP-NOW. Wristbands never touch WiFi — dramatically lower power and zero provisioning.
- **Predictive timing**: send "beat will happen at time T" rather than "beat now." All wristbands schedule flashes for the same absolute time, flashing within microseconds of each other.
- **ESP-NOW over WiFi**: peer-to-peer 2.4GHz, ~1-2ms latency, no router needed. Perfect for a crowd of 30+ devices.
- **Three-stage progression**: Mac+USB dongle (prototype) → Raspberry Pi (standalone) → custom ESP32-S3 PCB (final). Wristband firmware unchanged across stages.

## Pioneer DJ Link Integration

After the initial Ableton Link prototype, the system was extended to work directly with Pioneer Pro DJ Link:

1. **prodjlink-rs development**: Full Rust implementation of the Pro DJ Link protocol — device discovery (UDP 50000), beat reception (UDP 50001), CDJ/mixer status tracking (UDP 50002), virtual CDJ emulation, tempo master tracking
2. **beatbridge**: Bridge between Pro DJ Link and Ableton Link — syncs CDJ tempo/phase to Link peers
3. **Real hardware testing**: Tested against CDJ-3000s and DJM-A9 on a real network (192.168.1.x), discovered CDJ-3000 uses NXS-GW (0x40) packets instead of classic 0x0a status
4. **CDJ-3000 keep-alive fixes**: Fixed IP address handling in keep-alive packets for CDJ-3000 compatibility
5. **Beat-based master inference**: Inferred tempo master from beat packets since CDJ-3000 doesn't send classic status
6. **Gap analysis vs beat-link-trigger**: Compared against Deep Symmetry's beat-link-trigger to identify missing features
7. **On-air status**: Implemented mixer channel on-air detection from DJM status packets
8. **Master handoff (Baroque dance)**: Full implementation of the CDJ master handoff protocol — sync counter tracking, yield/accept state machine, auto-negotiate mode. 782 prodjlink-rs tests passing.

## Protocol v2

The wire protocol was upgraded from v1 (25 bytes, Link-only) to v2 (36 bytes, DJ Link-aware):

- Added next_beat_us for per-beat flashing (v1 only had downbeat timing)
- Added on_air_mask (mixer channel status), master_device, FLAG_CDJ_ACTIVE
- Hard cut — no backward compatibility with v1

## Beat Source State Machine

The coordinator implements a state machine:

- **CDJ_ACTIVE**: CDJ beats from tempo master are fresh (< 2s old). CDJ timing is authoritative, tempo is bridged to Link.
- **LINK_ONLY**: No CDJ master detected. Link session provides timing.
- Transition is automatic with 2s hysteresis.

## CRC Bug Discovery

During integration testing, a critical CRC mismatch was discovered: the coordinator used CRC-8/MAXIM-DOW (reflected) but the firmware used non-reflected poly 0x31. Packets would have been rejected by real hardware. Fixed by defining a custom non-reflected CRC algorithm in the coordinator.

## Testing Infrastructure

- Firmware simulators (DongleSim, WristbandSim) allow testing the full pipeline without real ESP32 hardware
- 40 tests: unit tests, firmware simulation, end-to-end integration
- BeatSourceState extracted from monolithic main() for testability with time injection

## Current Status

- Stage 1 prototype: complete
- Protocol v2: finalized and tested
- prodjlink-rs: 782 tests passing
- beatbridge: 85 tests passing
- firefly coordinator: 40 tests passing
- Ready for real hardware validation
