# Firefly Wire Protocol v2

## Overview

36-byte binary packet, little-endian. Identical packets travel over:

- **Path A (firefly-fw, all-in-one)**: ESP-NOW broadcast at 100 Hz.
- **Path B (legacy host)**: USB serial 115200 baud (Mac → dongle), then
  ESP-NOW broadcast (dongle → wristbands).

Designed for minimal parsing overhead on ESP32 microcontrollers.

## Packet Format

| Offset | Size | Field | Type | Description |
|--------|------|-------|------|-------------|
| 0–1 | 2 | sync | u8[2] | Magic bytes `0xBE 0xA7` |
| 2 | 1 | version | u8 | Protocol version `0x02` |
| 3 | 1 | total_len | u8 | Total packet length `36` |
| 4–11 | 8 | send_time_us | i64 | Broadcaster clock at packet creation (microseconds) |
| 12–19 | 8 | next_downbeat_us | i64 | Broadcaster time of next downbeat (microseconds) |
| 20–21 | 2 | tempo_bpm_x100 | u16 | BPM × 100 (e.g. 12630 = 126.30 BPM) |
| 22 | 1 | beat_in_bar | u8 | Current beat phase (0–3 for 4/4 time) |
| 23 | 1 | flags | u8 | Bit flags (see below) |
| 24–31 | 8 | next_beat_us | i64 | Broadcaster time of next beat (microseconds) |
| 32 | 1 | on_air_mask | u8 | Bitmask: bit N = mixer channel (N+1) is on-air |
| 33 | 1 | master_device | u8 | DJ Link device number of tempo master |
| 34 | 1 | reserved | u8 | Reserved for future use (must be 0) |
| 35 | 1 | crc8 | u8 | CRC-8 over bytes [2..35) |

## Flag Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | FLAG_PLAYING | Music is playing |
| 1 | FLAG_CDJ_ACTIVE | CDJ is the timing authority (vs Link-only / internal-clock fallback) |
| 2-7 | — | Reserved |

## Sentinel Values

- `next_beat_us = 0` → next beat time unknown / not available
- `next_downbeat_us = 0` → next downbeat time unknown
- `master_device = 0` → no DJ Link master detected (Link-only / idle mode)
- `on_air_mask = 0` → unknown / not available (firefly-fw does not yet
  parse mixer status, so it always sends 0)

## CRC-8

- **Algorithm**: Non-reflected CRC-8, polynomial 0x31, init 0x00
- **NOT CRC-8/MAXIM-DOW** (which uses reflected input/output)
- **Check value**: CRC of ASCII "123456789" = `0xA2`
- **Coverage**: bytes [2..35) (version through reserved, inclusive)
- **Implementation** (C, `shared/protocol.h`):

```c
uint8_t firefly_crc8(const uint8_t *data, size_t len) {
    uint8_t crc = 0x00;
    for (size_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t bit = 0; bit < 8; bit++) {
            if (crc & 0x80)
                crc = (crc << 1) ^ 0x31;
            else
                crc <<= 1;
        }
    }
    return crc;
}
```

The same algorithm is implemented in:

- Rust host coordinator: `crc` crate with a custom `CRC8_FIREFLY`
  algorithm definition (poly 0x31, init 0x00, refin/refout false).
- Rust firefly-fw: `firefly-fw/src/protocol.rs` — hand-port mirroring
  the C bit-bang.
- Arduino dongle / wristband: inline in `shared/protocol.h`.

All four implementations must produce identical checksums. CI exercises
this via the firmware simulators in `coordinator/src/firmware_sim.rs`
plus the native C++ tests in `tests/test_firmware.cpp`.

## Hello Frames

Wristbands send 8-byte hello frames so the broadcaster can pair them
as ESP-NOW unicast peers (max 8 peers). Distinct from the data packet
to keep parsing trivial:

| Offset | Size | Field | Description |
|---|---|---|---|
| 0–1 | 2 | sync | `0xBE 0xA8` (note: A8, not A7) |
| 2–7 | 6 | (reserved / wristband info) | |

Cadence: every 1 s when idle, every 5 s when actively receiving beats.

## Framing (Path B serial)

On serial, the dongle uses a state machine to detect packets:

1. Scan for sync byte 0 (0xBE).
2. Expect sync byte 1 (0xA7) — if not, check if current byte is 0xBE
   (re-enter step 2), otherwise reset to step 1.
3. Read remaining 34 bytes to fill the 36-byte buffer.
4. Validate version == 0x02.
5. Validate CRC.
6. If valid, forward over ESP-NOW; otherwise increment error counter.

Partial frame timeout: 50 ms without new bytes resets the state
machine.

The state machine is extracted into `shared/dongle_logic.h` so it can
be tested without an ESP32.

## Clock Synchronization

The wristband tracks the offset between broadcaster and local clocks
using an exponential moving average:

- First packet: `offset = send_time_us - local_time_us`
- Subsequent: `offset = 0.1 × measured + 0.9 × previous_offset`
- To convert broadcaster time to local: `local = broadcaster_time - offset`

In addition, the wristband runs a **PLL anchor** in
`tick_predicted_beats` — an internal beat clock anchored to the most
recent CDJ beat received. This lets it interpolate across short ESP-NOW
outages (e.g. ~500 ms bursts when Wi-Fi STA preempts the radio for
beacon recovery) without visibly stalling the beat flash.

Helpers shared between firmware paths live in `shared/beat_clock.h`.

## Broadcast Rate

| Path | Rate | Why |
|---|---|---|
| **Path A** (firefly-fw) | **100 Hz** (10 ms period) | The single-radio C3 cannot sustain 200 Hz × ~3 retransmits without saturating the IDF ESP-NOW TX queue (`ESP_ERR_ESPNOW_NO_MEM`) when Wi-Fi STA contention spikes. 10 ms phase granularity is well below human flicker-fusion for beat-synced LEDs. |
| **Path B** (host coordinator) | **200 Hz** (5 ms period) | Default; configurable via `--rate`. The host has a dedicated USB serial channel with no Wi-Fi STA contention on the dongle's radio. |

PHY rate is **11b @ 1 Mbps** (`WIFI_PHY_RATE_1M_L`) on both paths. Long
Range mode caused bursty arrival that broke beat-flash timing — do not
enable it.

## v1 → v2 Changes

- Packet size: 25 → 36 bytes.
- `payload_len` field renamed to `total_len` (now includes header).
- Version bumped from 0x01 to 0x02.
- Added fields: `next_beat_us`, `on_air_mask`, `master_device`,
  `reserved`.
- Added FLAG_CDJ_ACTIVE flag bit.
- CRC coverage adjusted for new size.
- **Breaking change** — no backward compatibility with v1.
