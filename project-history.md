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

- Stage 1 prototype: complete (Path B host coordinator).
- **Stage 3 lite shipped early as Path A `firefly-fw/`** — single-chip
  XIAO ESP32-C3 all-in-one broadcaster (Wi-Fi STA + DJ Link UDP +
  ESP-NOW + SSD1306 OLED), validated end-to-end at 124 BPM for 60 s.
  See "Stage 3 lite" section below.
- Protocol v2: finalized and tested.
- prodjlink-rs: 782 tests passing.
- beatbridge: 85 tests passing.
- firefly coordinator: 40 tests passing.
- Real hardware validation: ongoing on Path A (firefly-fw + 1
  wristband).

## Stage 3 lite — All-in-one firefly-fw on XIAO ESP32-C3

The original Stage 3 plan called for a custom ESP32-S3 PCB to collapse
the host coordinator + USB-serial dongle into a single device. We
shipped that goal early on the cheaper / available XIAO ESP32-C3 by
porting the Rust coordinator to `esp-idf-svc` (std-on-IDF), losing
only Ableton Link peer participation (the C++ Link library does not
cross-compile to RISC-V).

Key milestones:

1. **Bring-up**: nightly toolchain via espup, target
   `riscv32imc-esp-espidf`, `WIFI_SSID` / `WIFI_PASS` baked in via
   `env!()` at compile time.
2. **DJ Link UDP listener**: hand-rolled UDP parser for ports 50001
   (Beat) + 50002 (Status), magic envelope `Qspt1WmJOL`, tempo and
   beat-in-bar extraction.
3. **BeatSourceState port**: hand-port from `coordinator/src/main.rs`
   with PLL anchor (`tick_predicted_beats`) for smooth interpolation
   across packet drops.
4. **OLED port**: SSD1306 on I²C0 (SDA=GPIO6, SCL=GPIO7) on a
   dedicated thread, mirroring the dongle's FreeRTOS oled_task to
   prevent I²C latency from galloping the broadcast loop.
5. **Channel alignment**: wristband firmware moved from
   `ESPNOW_CHANNEL=6` to `=11` to match the AP's Wi-Fi channel.
   firefly-fw inherits its ESP-NOW channel from Wi-Fi STA, so the AP
   must be pinned to channel 11.
6. **Stutter debugging**: ESP-NOW + Wi-Fi STA on a single radio at
   200 Hz × ~3 retransmits saturated the IDF TX queue with
   `ESP_ERR_ESPNOW_NO_MEM`. Initial fix attempt
   (`esp_wifi_set_ps(WIFI_PS_NONE)`) backfired — full-duty radio made
   the saturation worse. Final fix: drop broadcast rate to 100 Hz +
   power-of-two rate-limit on the warning. ~4 % drop rate, smoothed
   over by the wristband's PLL.
7. **Antenna**: solder the XIAO's IPEX U.FL antenna. The PCB antenna
   detunes badly without a USB-cable counterpoise — works on a
   laptop, fails on charger / different host.
8. **CI**: added `firefly-fw-fmt` (nightly rustfmt) and
   `firefly-fw-clippy` (cross-compile + clippy `-D warnings`) jobs
   using `esp-rs/xtensa-toolchain@v1.5`.

End-to-end validation: synthetic Pro DJ Link beat sender → firefly-fw
→ wristband. 60 s @ 124 BPM, wristband locked on
`cdj=yes / playing=yes` for the full burst.
