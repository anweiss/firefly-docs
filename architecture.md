# Firefly Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        DJ BOOTH                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ CDJ-3000 │  │ CDJ-3000 │  │  DJM-A9  │                     │
│  │   (P1)   │  │   (P2)   │  │  (P33)   │                     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                     │
│       └──────────────┼─────────────┘                            │
│              PRO DJ LINK (ethernet)                              │
│              UDP 50000/50001/50002                               │
└──────────────────────┼──────────────────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   coordinator   │     Ableton Link      ┌──────────┐
              │  (Rust, Mac)    │◄──────────────────────►│ Ableton  │
              │  prodjlink-rs   │     (WiFi/LAN)         │  Live    │
              │  ableton-link-rs│                         └──────────┘
              └────────┬────────┘
                       │ USB Serial (115200 baud, 36-byte v2 packets)
              ┌────────▼────────┐
              │     dongle      │  Serial → ESP-NOW bridge
              │  (ESP32-C3)     │  CRC validation + forwarding
              └────────┬────────┘
                       │ ESP-NOW broadcast (~1-2ms, 2.4GHz)
            ┌──────────┼──────────┐
      ┌─────▼─────┐         ┌─────▼─────┐
      │ wristband  │  ×N     │ wristband  │
      │ (ESP32-C3) │         │ (ESP32-C3) │
      │ WS2813 LEDs│         │ WS2813 LEDs│
      └────────────┘         └────────────┘
```

## Signal Flow

The system has five hops from DJ deck to wristband LED:

1. **CDJ → Coordinator**: CDJ beat packets arrive over Pro DJ Link (UDP 50001). The coordinator uses `prodjlink-rs` to join the Pioneer network and receive beat timing from the tempo master.
2. **Coordinator processing**: `BeatSourceState` ingests beats, tracks the active source (CDJ or Link fallback), and computes absolute timestamps for the next beat and next bar.
3. **Coordinator → Dongle**: The coordinator builds a 36-byte v2 packet and sends it over USB serial at 115200 baud.
4. **Dongle → Wristbands**: The dongle validates the CRC and forwards the packet as an ESP-NOW broadcast frame over 2.4GHz WiFi PHY.
5. **Wristband LED flash**: Each wristband parses the packet, updates its clock offset via an EMA filter, and schedules the LED flash at the predicted beat time.

## Timing Architecture

### Predictive, Not Reactive

Packets encode "beat will happen at time T" as an absolute coordinator-clock timestamp — not "beat happened now." This means transport latency is irrelevant to synchronization accuracy. Every wristband independently schedules its flash for the same future moment.

### Clock Offset Tracking

Each wristband maintains a running estimate of the offset between the coordinator's clock and its own local clock using an exponential moving average (EMA) filter with α=0.1:

```
offset_estimate = α × new_offset + (1 - α) × offset_estimate
```

This smooths out jitter while adapting to clock drift. With the offset estimate, wristbands convert coordinator-clock timestamps to local-clock times for precise scheduling.

### Synchronization Precision

All wristbands flash within ~tens of microseconds of each other, regardless of transport jitter or individual packet arrival times. The predictive model decouples synchronization accuracy from transport latency.

### Latency Budget

| Hop | Latency | Notes |
|---|---|---|
| DJ Link → Coordinator | ~0ms | Same Ethernet segment |
| USB Serial | ~1ms | 115200 baud, 36 bytes |
| ESP-NOW | ~1-2ms | WiFi PHY broadcast |
| **Total transport** | **~2-3ms** | **Fully compensated by predictive timing** |

### Why Not a Microphone?

A microphone-based approach is reactive: sound travels at ~1ms/ft, FFT processing adds 10-50ms of latency, crowd noise degrades signal quality, and there's no way to distinguish downbeats from regular beats. Firefly's approach is deterministic and phase-locked to the DJ's master clock — it knows beat positions before they happen.

## Beat Source State Machine

The coordinator maintains a `BeatSourceState` that selects between two timing sources:

### CDJ_ACTIVE

Active when beats from the DJ Link tempo master are less than 2 seconds old. The CDJ owns all timing — tempo is bridged from CDJ to the Ableton Link session so other Link peers follow the DJ.

CDJ beat timing uses the `Beat.next_beat` and `Beat.next_bar` fields directly (milliseconds-from-now), which the coordinator converts to absolute coordinator-clock timestamps.

### LINK_ONLY

Fallback when no CDJ tempo master is detected. After a 2-second timeout (`CDJ_TIMEOUT`), the state machine transitions to `LINK_ONLY` and the Ableton Link session provides tempo and phase.

### Transitions

```
CDJ beat received ──► CDJ_ACTIVE
                         │
                    2s timeout (no CDJ beats)
                         │
                         ▼
                      LINK_ONLY
                         │
                    CDJ beat received
                         │
                         ▼
                      CDJ_ACTIVE
```

Hysteresis via the 2-second `CDJ_TIMEOUT` prevents rapid toggling during brief CDJ communication gaps.

## DJ Link Integration (prodjlink-rs)

The coordinator uses `prodjlink-rs` to join the Pioneer Pro DJ Link network as a virtual player:

- **Device number**: 5 by default (configurable via `--device-number`)
- **Discovery**: Joins the network via UDP 50000, discovers CDJs and mixers
- **Beat packets**: Receives beat timing on UDP 50001 from the tempo master
- **Status packets**: Tracks tempo master assignment via UDP 50002
- **On-air status**: Receives channel on-air information from the mixer
- **Master handoff**: Supports the Baroque dance protocol for master negotiation with auto-negotiate. Note: CDJ-3000 sends NXS-GW (0x40) packets instead of classic 0x0a status, so the Baroque dance won't activate — but beat and master tracking work correctly.

## Ableton Link Integration (ableton-link-rs)

The coordinator uses `ableton-link-rs` to participate in an Ableton Link session on the local network:

- **CDJ active**: Bridges CDJ tempo into the Link session. Other Link peers (e.g., Ableton Live) follow the DJ's tempo.
- **Link-only mode**: Reads tempo and phase from the Link session as the timing source.
- **Shared clock**: Provides a microsecond-precision clock used for computing absolute timestamps in v2 packets.

## ESP-NOW

ESP-NOW is Espressif's peer-to-peer WiFi protocol used for dongle-to-wristband communication:

- **No infrastructure**: No router, no TCP/IP stack, no connection handshake
- **Raw frames**: 250-byte maximum payload broadcast at the WiFi PHY layer
- **Low latency**: ~1-2ms typical (vs. 10-50ms for WiFi TCP/UDP)
- **Range**: ~30-50m indoors, ~200-400m outdoors with line of sight
- **Fixed channel**: Channel 1 to avoid contention with venue WiFi
- **Wristbands are ESP-NOW only**: They never connect to WiFi — only receive ESP-NOW broadcasts

## Component Details

### Coordinator

**Language**: Rust  
**Platform**: Mac (Stage 1)

The coordinator is the brain of the system. It bridges DJ Link and Ableton Link, manages beat source state, and generates v2 timing packets.

Key internals:
- `BeatSourceState`: CDJ state tracking with time-injectable methods for deterministic testing
- Builds v2 packets at a configurable rate (default 20Hz)
- Serial reconnect on disconnect

CLI options:
| Flag | Description |
|---|---|
| `--port` | Serial port for dongle |
| `--baud` | Baud rate (default: 115200) |
| `--bpm` | Manual BPM override |
| `--quantum` | Ableton Link quantum (beats per bar) |
| `--rate` | Packet send rate in Hz |
| `--interface` | Network interface for DJ Link |
| `--device-number` | Virtual player number on DJ Link network |
| `--no-djlink` | Disable DJ Link, use Link-only mode |

### Dongle

**Platform**: ESP32-C3 (Seeed XIAO)  
**Language**: Arduino C++  
**Code size**: ~80 lines

The dongle is a simple serial-to-ESP-NOW bridge with a framing state machine:

1. **Sync byte detection**: Scans for the packet sync byte
2. **Version validation**: Confirms v2 protocol
3. **CRC validation**: Verifies packet integrity
4. **ESP-NOW broadcast**: Forwards valid packets

Additional behavior:
- Frame timeout (50ms) resets partial packets to recover from corruption
- Tracks stats: `packets_forwarded`, `crc_errors`, `version_errors`

### Wristband

**Platform**: ESP32-C3 (Seeed XIAO)  
**Language**: Arduino C++

The wristband receives ESP-NOW packets and drives LED animations synced to beats:

**Receive path**:
- ESP-NOW callback validates sync byte, protocol version, and CRC
- Parses all packet fields (tempo, beat/bar timestamps, phase, beat source)
- Updates clock offset EMA (α=0.1)

**Main loop**:
- Computes predicted local time for the next beat using clock offset estimate
- Fires FastLED animation at the scheduled moment

**Flash colors**:
| Color | Meaning |
|---|---|
| Orange | Downbeat (beat 1 of bar) |
| Blue | Regular beat |
| Green | CDJ-active beat |

**Idle behavior**: Dim blue pulse when no packets received for >3 seconds.

**Debug**: Serial output every 5 seconds with timing stats.

## Test Architecture

The coordinator has **40 tests** covering four layers:

### Unit Tests
- Packet building: field encoding, byte layout, sync/version bytes
- CRC validation: correct computation for known inputs
- `BeatSourceState` methods: state transitions, timeout behavior, beat ingestion
- On-air mask: channel status parsing

### Firmware Simulators
- **DongleSim**: Rust implementation of the dongle's framing state machine — sync detection, version check, CRC validation, frame timeout
- **WristbandSim**: Rust implementation of the wristband's packet parser and clock offset EMA filter

### End-to-End Integration
Full pipeline tests: CDJ Beat → `BeatSourceState` → `build_packet` → DongleSim → WristbandSim. Verifies that a beat event from a CDJ produces the correct LED flash time at the wristband.

### CRC Cross-Validation
The coordinator uses Rust's `crc` crate with a custom non-reflected algorithm. Firmware uses manual bit-banging. Tests verify both implementations produce identical results for the same inputs.

## Hardware (Stage 1 Prototype)

| Component | Quantity | Notes |
|---|---|---|
| Seeed XIAO ESP32-C3 | 2 | Pre-soldered headers (1 dongle + 1 wristband) |
| Grove RGB LED Stick | 1 | WS2813, 10 LEDs |
| USB-C data cable | 1 | Coordinator ↔ dongle |
| 3.7V LiPo battery | 1 | JST-PH 2.0 connector, for wristband |
| Seeed XIAO Expansion Board | 1 | Optional — OLED debug display + battery socket |

## Future Stages

| Stage | Coordinator | Wristbands |
|---|---|---|
| 1 (current) | Mac + USB dongle ESP32 | 1-2 prototypes |
| 2 | Raspberry Pi Zero 2 W + UART ESP32 | 30 battery-powered |
| 3 | Custom ESP32-S3 single-chip PCB | Unchanged from Stage 2 |

Wristband firmware is identical across all stages — only the coordinator platform changes.
