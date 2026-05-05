# Firefly Architecture

## System Overview

Firefly has **two deployment paths** that share the same wire protocol and wristband firmware:

### Path A — All-in-one (firefly-fw, recommended)

```text
┌─────────────────────────────────────────────────────────────────┐
│                        DJ BOOTH                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ CDJ-3000 │  │ CDJ-3000 │  │  DJM-A9  │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       └──────────────┼─────────────┘                            │
│              PRO DJ LINK (ethernet)                             │
│              UDP 50001/50002 broadcast                          │
└──────────────────────┼──────────────────────────────────────────┘
                       │
                       │  (Wi-Fi LAN — bridge ethernet ↔ Wi-Fi
                       │   via a travel router, all on channel 11)
                       │
              ┌────────▼─────────┐
              │   firefly-fw     │
              │  XIAO ESP32-C3   │
              │  Rust + ESP-IDF  │
              │  100 Hz packets  │
              │  + SSD1306 OLED  │
              └────────┬─────────┘
                       │ ESP-NOW broadcast (channel 11, 1 Mbps PHY)
            ┌──────────┼──────────┐
      ┌─────▼─────┐         ┌─────▼─────┐
      │ wristband │  ×N     │ wristband │
      │ ESP32-C3  │         │ ESP32-C3  │
      │ WS2813    │         │ WS2813    │
      └───────────┘         └───────────┘
```

No host laptop. No USB serial. Single 30 g device that plugs into any
USB-C charger.

### Path B — Host coordinator (legacy / Link bridge)

```text
┌─────────────────────────────────────────────────────────────────┐
│                        DJ BOOTH                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ CDJ-3000 │  │ CDJ-3000 │  │  DJM-A9  │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       └──────────────┼─────────────┘                            │
│              PRO DJ LINK (ethernet)                             │
└──────────────────────┼──────────────────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   coordinator   │     Ableton Link        ┌──────────┐
              │  (Rust, Mac)    │◄───────────────────────►│ Ableton  │
              │  prodjlink-rs   │     (Wi-Fi/LAN)         │  Live    │
              │  ableton-link-rs│                         └──────────┘
              └────────┬────────┘
                       │ USB Serial (115200 baud, 36-byte v2 packets)
              ┌────────▼────────┐
              │     dongle      │  Serial → ESP-NOW bridge
              │  (ESP32-C3)     │  CRC validation + forwarding
              └────────┬────────┘
                       │ ESP-NOW broadcast (channel 11, 1 Mbps PHY)
            ┌──────────┼──────────┐
      ┌─────▼─────┐         ┌─────▼─────┐
      │ wristband │  ×N     │ wristband │
      └───────────┘         └───────────┘
```

Required when you want **Ableton Link peer participation** — the
`ableton-link-rs` crate wraps a C++ library that does not currently
cross-compile to RISC-V, so Link runs on the Mac side.

## Signal Flow

### Path A (firefly-fw, 4 hops)

1. **CDJ → firefly-fw**: Beat / Status UDP packets arrive at the XIAO
   over Wi-Fi from the Pioneer DJ Link broadcast domain. `djlink::run()`
   listens on UDP 50001 (Beat) and 50002 (Status), parses the magic
   `Qspt1WmJOL` envelope, and feeds tempo + beat-in-bar into the beat
   state.
2. **BeatSourceState (PLL)**: ingests beats, tracks the active source
   (CDJ vs internal-clock fallback), and computes absolute timestamps
   for the next beat / next bar.
3. **firefly-fw → wristbands**: 100 Hz periodic broadcast of 36-byte v2
   packets via ESP-NOW (channel 11, 11b @ 1 Mbps).
4. **Wristband LED flash**: each wristband parses the packet, updates
   its clock-offset EMA, and schedules the LED flash at the predicted
   beat time.

### Path B (host coordinator, 5 hops)

Same as Path A but with two additional hops between (1) and (3): the
Mac coordinator computes the packet, sends it over USB serial at
115 200 baud to the dongle XIAO, which validates the CRC and forwards
the frame as an ESP-NOW broadcast.

## Timing Architecture

### Predictive, Not Reactive

Packets encode "the next beat will happen at time T" as an absolute
broadcaster-clock timestamp — not "beat happened now." Transport
latency is irrelevant to synchronization accuracy. Every wristband
independently schedules its flash for the same future moment.

### Clock Offset Tracking

Each wristband maintains a running estimate of the offset between the
broadcaster's clock and its own local clock using an exponential
moving average (EMA) filter with α = 0.1:

```text
offset_estimate = 0.1 × new_offset + 0.9 × offset_estimate
```

This smooths jitter while adapting to drift. The wristband converts
broadcaster timestamps to local time via `local = remote − offset`.

### PLL Anchor

`tick_predicted_beats` runs an internal beat clock anchored to the
most recent CDJ beat received. This lets the wristband interpolate
across short ESP-NOW outages — e.g. the occasional ~500 ms burst when
Wi-Fi STA preempts the radio for beacon recovery — without visibly
stalling the beat flash.

### Synchronization Precision

All wristbands flash within tens of microseconds of each other,
regardless of transport jitter or individual packet arrival times.
The predictive model decouples synchronization accuracy from transport
latency.

### Latency Budget

| Hop | Latency | Notes |
|---|---|---|
| DJ Link → broadcaster | ~0–5 ms | Same Ethernet/Wi-Fi LAN |
| USB Serial (Path B only) | ~1 ms | 115 200 baud, 36 bytes |
| ESP-NOW | ~1–2 ms | 1 Mbps PHY broadcast |
| **Total transport** | **~3–7 ms** | **Fully compensated by predictive timing** |

### Why Not a Microphone?

A microphone-based approach is reactive: sound travels at ~1 ms/ft,
FFT processing adds 10–50 ms of latency, crowd noise degrades signal
quality, and there's no way to distinguish downbeats from regular
beats. Firefly's approach is deterministic and phase-locked to the
DJ's master clock — it knows beat positions before they happen.

## Beat Source State Machine

The broadcaster (firefly-fw or host coordinator) maintains a
`BeatSourceState` that selects between timing sources:

### CDJ_ACTIVE

Active when beats from the DJ Link tempo master are less than 2
seconds old. The CDJ owns all timing.

- **Path B**: tempo is bridged from CDJ to the Ableton Link session so
  Link peers (e.g. Ableton Live) follow the DJ.
- **Path A**: no Link session — the firefly-fw simply uses the CDJ's
  beat timing as authoritative.

CDJ beat timing uses the `Beat.next_beat` and `Beat.next_bar` fields
directly (milliseconds-from-now), converted to absolute clock
timestamps.

### LINK_ONLY (Path B) / IDLE (Path A)

Fallback when no CDJ tempo master is detected. After a 2-second
timeout (`CDJ_TIMEOUT`):

- **Path B** transitions to `LINK_ONLY` — Ableton Link provides tempo
  and phase.
- **Path A** transitions to internal-clock idle — broadcasts continue
  with `is_playing = false` so wristbands enter their idle pulse.

### Transitions

```text
CDJ beat received ──► CDJ_ACTIVE
                         │
                    2s timeout (no CDJ beats)
                         │
                         ▼
                   LINK_ONLY (B) / IDLE (A)
                         │
                    CDJ beat received
                         │
                         ▼
                      CDJ_ACTIVE
```

Hysteresis via the 2-second `CDJ_TIMEOUT` prevents rapid toggling
during brief CDJ communication gaps.

## DJ Link Integration

Both paths use [prodjlink-rs](https://github.com/anweiss/prodjlink-rs)
on the host side, or a hand-rolled UDP listener on firefly-fw, to:

- **Discover CDJs / mixers** on the network and (host path only) join
  as a virtual player on UDP 50000.
- **Receive beat packets** on UDP 50001 from the tempo master.
- **Track tempo master** assignment via UDP 50002.
- **(Host path)** Receive on-air status from the mixer.
- **(Host path)** Participate in the Baroque dance master-handoff
  protocol with auto-negotiate.

⚠️ CDJ-3000 sends NXS-GW (0x40) packets instead of classic 0x0a
status, so the Baroque dance won't activate against pure CDJ-3000
setups — but beat and master tracking work correctly.

firefly-fw uses a simpler "first deck heard wins" master selection.
Sufficient for single-deck tests; multi-deck setups should use the
host path.

## Ableton Link Integration (Path B only)

The host coordinator uses
[ableton-link-rs](https://github.com/anweiss/ableton-link-rs) to
participate in an Ableton Link session on the local network:

- **CDJ active**: bridges CDJ tempo into the Link session. Other Link
  peers (e.g. Ableton Live) follow the DJ.
- **Link-only mode**: reads tempo and phase from the Link session as
  the timing source.
- **Shared clock**: provides a microsecond-precision clock used for
  computing absolute timestamps in v2 packets.

Not available on firefly-fw — the Link C++ library does not currently
cross-compile to RISC-V (esp-idf-svc + xtensa-toolchain).

## ESP-NOW

ESP-NOW is Espressif's peer-to-peer Wi-Fi protocol used for the final
hop to wristbands:

- **No infrastructure**: no router, no TCP/IP stack, no connection
  handshake.
- **Raw frames**: 250-byte maximum payload broadcast at the Wi-Fi PHY
  layer.
- **Low latency**: ~1–2 ms typical (vs. 10–50 ms for Wi-Fi TCP/UDP).
- **Range**: ~30–50 m indoors with the IPEX U.FL antenna soldered (the
  XIAO PCB antenna detunes severely without a USB-cable counterpoise
  and is unreliable on battery / charger power).
- **Fixed channel: 11**. Pin your AP to channel 11 if you're using
  firefly-fw, since ESP-NOW operates on whatever channel the Wi-Fi STA
  is on.
- **PHY rate: 11b @ 1 Mbps** (`WIFI_PHY_RATE_1M_L`). Long Range mode
  caused bursty arrival that broke beat-flash timing.
- **Hello-frame peer pairing**: wristbands periodically send 8-byte
  hello frames (sync `0xBE 0xA8`); firefly-fw / dongle add the source
  MAC as a unicast peer (max 8) and unicast a copy of every broadcast
  for redundancy.
- **Wristbands are ESP-NOW only**: they never join Wi-Fi. Lower power,
  zero provisioning.

## Component Details

### firefly-fw (Path A broadcaster)

**Language**: Rust (esp-idf-svc, std-on-IDF)
**Platform**: Seeed XIAO ESP32-C3
**Toolchain**: nightly + rust-src, target `riscv32imc-esp-espidf`
(installed via [espup](https://github.com/esp-rs/espup))

Module layout (`firefly-fw/src/`):

| File | Role |
|---|---|
| `main.rs` | Boots Wi-Fi + DJ Link + broadcaster + display threads, runs the 100 Hz broadcast loop with internal-clock fallback |
| `wifi.rs` | STA scan-and-connect, default modem-sleep PS |
| `djlink.rs` | UDP listeners on 50001 + 50002, parses Pro DJ Link Beat / Status packets |
| `espnow.rs` | Broadcaster + hello-frame peer ingest, 1 Mbps PHY rate, rate-limited TX-fail warning |
| `beat_state.rs` | CDJ beat ingest + 2 s timeout + PLL beat prediction |
| `protocol.rs` | Hand-port of v2 packet builder from `coordinator/src/main.rs` |
| `display.rs` | SSD1306 OLED on I²C0 (SDA=GPIO6, SCL=GPIO7), dedicated thread |

Wi-Fi credentials are baked in via `env!()` at compile time
(`WIFI_SSID` / `WIFI_PASS`) — no provisioning yet. Pass them on the
command line: `WIFI_SSID="Andrew IoT" WIFI_PASS="..." cargo run --release`.

### Coordinator (Path B host-side)

**Language**: Rust
**Platform**: Mac (Stage 1) / Raspberry Pi (Stage 2 plan)

Bridges DJ Link and Ableton Link, manages beat source state, and
generates v2 timing packets.

Key internals:

- `BeatSourceState`: CDJ state tracking with time-injectable methods
  for deterministic testing.
- Builds v2 packets at a configurable rate (default 200 Hz, was 20 Hz
  in earlier revisions).
- Serial reconnect on disconnect.
- Auto-detects the dongle serial port (tries `/dev/cu.usbmodem*`
  first) — `--port` flag remains supported.

CLI options:

| Flag | Description |
|---|---|
| `--port` | Serial port for dongle (auto-detected if omitted) |
| `--baud` | Baud rate (default: 115200) |
| `--bpm` | Manual BPM override |
| `--quantum` | Ableton Link quantum (beats per bar) |
| `--rate` | Packet send rate in Hz |
| `--interface` | Network interface for DJ Link |
| `--device-number` | Virtual player number on DJ Link network |
| `--no-djlink` | Disable DJ Link, use Link-only mode |

### Dongle (Path B serial→ESP-NOW)

**Platform**: ESP32-C3 (Seeed XIAO)
**Language**: Arduino C++

Serial-to-ESP-NOW bridge with a framing state machine: sync detection,
version + CRC validation, ESP-NOW forwarding. Optional SSD1306 OLED
status display on the Seeed expansion board. Frame timeout (50 ms)
resets partial packets to recover from corruption.

### Wristband

**Platform**: ESP32-C3 (Seeed XIAO)
**Language**: Arduino C++

Receives ESP-NOW packets and drives LED animations synced to beats.

**Receive path**:

- ESP-NOW callback validates sync byte, protocol version, and CRC.
- Parses all packet fields (tempo, beat / bar timestamps, phase, beat
  source).
- Updates clock offset EMA (α = 0.1).
- ⚠️ Schedules-only — never calls `FastLED.show()` from the receive
  callback. WS281x bit-bang is timing-critical; concurrent show()
  calls from the Wi-Fi task and main loop corrupt the data. All LED
  writes are single-threaded in `loop()`.

**Main loop**:

- Computes predicted local time for the next beat using the clock
  offset estimate.
- Fires FastLED animation at the scheduled moment.
- Sends 8-byte hello frames every 1 s when idle / 5 s when live so
  the broadcaster can pair the wristband as a unicast peer.

**Flash colors**:

| Color | Meaning |
|---|---|
| Orange | Downbeat (beat 1 of bar) |
| Blue | Regular beat |
| Green | CDJ-active beat |

**Idle behavior**: dim blue pulse when no packets received for >3 s.

**Optional SSD1306 OLED**: same I²C wiring as the dongle / firefly-fw
(SDA=GPIO6, SCL=GPIO7, addr 0x3C) showing status, RSSI, and packet
counts.

**CPU pinned at 160 MHz** so Wi-Fi / ESP-NOW timing is stable across
power sources (default DFS can drop to 80 MHz under no-load).

## Test Architecture

The host coordinator has **40 tests** covering four layers:

### Unit Tests

- Packet building: field encoding, byte layout, sync / version bytes.
- CRC validation: correct computation for known inputs.
- `BeatSourceState` methods: state transitions, timeout behavior, beat
  ingestion.
- On-air mask: channel status parsing.

### Firmware Simulators

- **DongleSim**: Rust implementation of the dongle's framing state
  machine — sync detection, version check, CRC validation, frame
  timeout.
- **WristbandSim**: Rust implementation of the wristband's packet
  parser and clock offset EMA filter.

### End-to-End Integration

Full pipeline tests: `CDJ Beat → BeatSourceState → build_packet →
DongleSim → WristbandSim`. Verifies that a beat event from a CDJ
produces the correct LED flash time at the wristband.

### CRC Cross-Validation

The coordinator uses Rust's `crc` crate with a custom non-reflected
algorithm. Firmware uses manual bit-banging. Tests verify both
implementations produce identical results for the same inputs.

### Native Host C++ Tests

`tests/test_firmware.cpp` exercises the extracted state machines
(`shared/dongle_logic.h`, `shared/wristband_logic.h`,
`shared/beat_clock.h`) with `g++ -std=c++17 -Wall -Wextra -Werror`.

### CI

GitHub Actions runs all of the above on every push and PR plus:

- `cargo clippy --release --all-targets -- -D warnings` cross-compiled
  to `riscv32imc-esp-espidf` for `firefly-fw` (also serves as the
  build check), via `esp-rs/xtensa-toolchain`.
- `arduino-cli compile` for both `dongle-firmware/` and
  `wristband-firmware/`.
- `cargo audit` on both Rust crates, weekly schedule + on PR.

## Hardware

### Stage 1 / Path A (firefly-fw all-in-one)

| Component | Quantity | Notes |
|---|---|---|
| Seeed XIAO ESP32-C3 | 1 + N | 1 broadcaster + N wristbands. **Solder the IPEX U.FL antenna** — the PCB antenna is unreliable on battery / charger power. |
| Grove RGB LED Stick | 1 per wristband | WS2813, 10 LEDs |
| 0.96″ SSD1306 OLED | optional | I²C @ 0x3C, SDA=GPIO6, SCL=GPIO7 |
| 3.7 V LiPo + JST-PH 2.0 | 1 per wristband | |
| USB-C charger / data cable | | broadcaster runs from any USB-C source |
| Travel router (optional) | 1 | bridges CDJ Ethernet ↔ Wi-Fi for the broadcaster; pin to channel 11 |

### Stage 1 / Path B (legacy host)

| Component | Quantity | Notes |
|---|---|---|
| Seeed XIAO ESP32-C3 | 1 + N | 1 dongle + N wristbands |
| Grove RGB LED Stick | 1 per wristband | WS2813, 10 LEDs |
| USB-C data cable | 1 | Mac ↔ dongle |
| Seeed XIAO Expansion Board | 1 | optional — OLED debug display + battery socket |
| 3.7 V LiPo + JST-PH 2.0 | 1 per wristband | |

## Stage Progression

| Stage | Broadcaster | Runs Rust DJ Link / Link | Wristbands |
|---|---|---|---|
| 1 (current) | Mac + USB dongle ESP32 (Path B) **or** XIAO ESP32-C3 standalone (Path A, firefly-fw) | Mac (Path B) / on-device (Path A) | 1–2 prototypes |
| 2 | Raspberry Pi Zero 2 W + UART ESP32 | On Pi | 30 battery-powered |
| 3 | Custom ESP32-S3 single-chip PCB | On ESP32-S3 | Unchanged from Stage 2 |

Note that **Stage 3 lite has effectively shipped early as Path A
firefly-fw on the XIAO ESP32-C3** — the same single-chip-coordinator
goal, just on the C3 instead of the planned S3, and without Ableton
Link. The S3 + Link variant remains future work.

Wristband firmware is identical across all stages — only the
broadcaster changes.
