# Firefly Wire Protocol v2

## Overview

36-byte binary packet, little-endian, transmitted over USB serial (coordinator→dongle) and ESP-NOW (dongle→wristbands). Designed for minimal parsing overhead on ESP32 microcontrollers.

## Packet Format

| Offset | Size | Field | Type | Description |
|--------|------|-------|------|-------------|
| 0–1 | 2 | sync | u8[2] | Magic bytes `0xBE 0xA7` |
| 2 | 1 | version | u8 | Protocol version `0x02` |
| 3 | 1 | total_len | u8 | Total packet length `36` |
| 4–11 | 8 | send_time_us | i64 | Coordinator clock at packet creation (microseconds) |
| 12–19 | 8 | next_downbeat_us | i64 | Coordinator time of next downbeat (microseconds) |
| 20–21 | 2 | tempo_bpm_x100 | u16 | BPM × 100 (e.g. 12630 = 126.30 BPM) |
| 22 | 1 | beat_in_bar | u8 | Current beat phase (0–3 for 4/4 time) |
| 23 | 1 | flags | u8 | Bit flags (see below) |
| 24–31 | 8 | next_beat_us | i64 | Coordinator time of next beat (microseconds) |
| 32 | 1 | on_air_mask | u8 | Bitmask: bit N = mixer channel (N+1) is on-air |
| 33 | 1 | master_device | u8 | DJ Link device number of tempo master |
| 34 | 1 | reserved | u8 | Reserved for future use (must be 0) |
| 35 | 1 | crc8 | u8 | CRC-8 over bytes [2..35) |

## Flag Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | FLAG_PLAYING | Music is playing |
| 1 | FLAG_CDJ_ACTIVE | CDJ is the timing authority (vs Link-only) |
| 2-7 | — | Reserved |

## Sentinel Values

- `next_beat_us = 0` → next beat time unknown / not available
- `next_downbeat_us = 0` → next downbeat time unknown
- `master_device = 0` → no DJ Link master detected (Link-only mode)

## CRC-8

- **Algorithm**: Non-reflected CRC-8, polynomial 0x31, init 0x00
- **NOT CRC-8/MAXIM-DOW** (which uses reflected input/output)
- **Check value**: CRC of ASCII "123456789" = `0xA2`
- **Coverage**: bytes [2..35) (version through reserved, inclusive)
- **Implementation** (C):

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

## Framing

On serial, the dongle uses a state machine to detect packets:

1. Scan for sync byte 0 (0xBE)
2. Expect sync byte 1 (0xA7) — if not, check if current byte is 0xBE (re-enter step 2), otherwise reset to step 1
3. Read remaining 34 bytes to fill 36-byte buffer
4. Validate version == 0x02
5. Validate CRC
6. If valid, forward over ESP-NOW; otherwise increment error counter

Partial frame timeout: 50ms without new bytes resets the state machine.

## Clock Synchronization

The wristband tracks the offset between coordinator and local clocks using an exponential moving average:

- First packet: `offset = send_time_us - local_time_us`
- Subsequent: `offset = 0.1 × measured + 0.9 × previous_offset`
- To convert coordinator time to local: `local = coordinator_time - offset`

## v1 → v2 Changes

- Packet size: 25 → 36 bytes
- `payload_len` field renamed to `total_len` (now includes header)
- Version bumped from 0x01 to 0x02
- Added fields: next_beat_us, on_air_mask, master_device, reserved
- Added FLAG_CDJ_ACTIVE flag bit
- CRC coverage adjusted for new size
- **Breaking change** — no backward compatibility

## Broadcast Rate

Default 20Hz (50ms interval). Configurable via coordinator `--rate` flag. Higher rates improve responsiveness to tempo changes but increase serial/ESP-NOW bandwidth.
