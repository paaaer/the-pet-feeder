# Pet Feeder — ESPHome Firmware

ESPHome firmware for a Tuya-based automatic pet feeder, replacing the original WBR3 Wi-Fi module with an ESP32-C3 Super Mini. The ESP32-C3 speaks the Tuya serial protocol directly to the feeder MCU over UART, and exposes control and status via MQTT to Home Assistant.

---

## Hardware

### Feeder Control Board

The feeder's main control board carries the marking **A90_YP_2301WIFI_V1.0_W**. Throughout this firmware and the Tuya serial protocol documentation, this board is referred to as the **MCU** — this is standard Tuya terminology for the non-Wi-Fi side of the protocol (as opposed to the "Wi-Fi module" side). In practice it is the feeder's dedicated controller handling the motor, sensors, buttons, and LED indicators.

The original WBR3 Wi-Fi module is physically removed and replaced by an ESP32-C3 Super Mini soldered to the same pads.

| WBR3 Pad | ESP32-C3 | Direction | Function |
|----------|----------|-----------|----------|
| Pin 8    | 3V3      | PWR       | VCC — power supply |
| Pin 9    | GND      | PWR       | Ground |
| Pin 15   | GPIO21   | ESP → MCU | UART TX (WBR3 RXD — data to MCU) |
| Pin 16   | GPIO20   | MCU → ESP | UART RX (WBR3 TXD — data from MCU) |
| Pin EN   | GPIO10   | ESP → MCU | Module ready signal |

All signals are 3.3V — no level shifting required. GPIO10 has no strapping pin restrictions on the ESP32-C3, making it safe for use as a general output.

### EN Pin behaviour

The EN pin (GPIO10) is held LOW at boot. The MCU will not begin sending heartbeats until it sees this pin go HIGH. The firmware drives it HIGH only after WiFi and MQTT are connected, ensuring the MCU never tries to communicate before the ESP32 is ready to respond.

---

## UART Configuration

- **Baud rate:** 9600
- **TX pin:** GPIO21 (ESP → MCU)
- **RX pin:** GPIO20 (MCU → ESP)

---

## Tuya Serial Protocol

The feeder uses the standard Tuya serial protocol between the Wi-Fi module and the feeder MCU.

### Frame format

All frames follow this structure:

```
[0x55] [0xAA] [VER] [CMD] [LEN_HI] [LEN_LO] [PAYLOAD...] [CHECKSUM]
```

| Field    | Size    | Description |
|----------|---------|-------------|
| Header   | 2 bytes | Always `55 AA` |
| Version  | 1 byte  | `0x00` = v0 (ESP → MCU), `0x03` = v3 (MCU → ESP) |
| Command  | 1 byte  | Command ID |
| Length   | 2 bytes | Payload length, big-endian |
| Payload  | N bytes | Command-specific data |
| Checksum | 1 byte  | Sum of all preceding bytes, `& 0xFF` |

### Protocol versions

- The original WBR3 always sent **v0 frames** (`55 AA 00 ...`) to the MCU.
- The MCU responds with **v3 frames** (`55 AA 03 ...`), confirmed from UART captures of the original WBR3.
- The RX parser does not validate the version byte — it syncs on the `55 AA` header and reads the command from byte 3. The version byte (`buf[2]`) is accepted but ignored.
- This firmware always sends v0 frames, which the MCU accepts.

---

## Startup Handshake

After the EN pin goes HIGH, the MCU starts sending heartbeats. On the first MQTT connection the firmware sends the full 5-frame handshake sequence that the WBR3 used, confirmed from UART captures of the original module. Each frame is sent with a 200ms delay between them.

| Step | Frame (hex) | Purpose |
|------|------------|---------|
| 1 | `55 AA 00 00 00 00 FF` | Heartbeat |
| 2 | `55 AA 00 01 00 00 00` | Product info query |
| 3 | `55 AA 00 02 00 00 01` | MCU state query |
| 4 | `55 AA 00 03 00 01 04 07` | WiFi status: cloud connected |
| 5 | `55 AA 00 08 00 00 07` | Functional test (CMD 0x08) |

After the handshake the MCU begins sending regular heartbeats and DP status reports.

If the MCU connection is lost, the 15-second interval will retry the handshake automatically.

---

## Commands — ESP → MCU (we send)

### CMD 0x00 — Heartbeat reply

Sent in response to every heartbeat received from the MCU.

```
55 AA 00 00 00 00 FF
```

### CMD 0x01 — Product info query

Sent during handshake. MCU responds with its product ID string (CMD 0x01 response).

```
55 AA 00 01 00 00 00
```

### CMD 0x02 — MCU state query

Sent during handshake. MCU responds with a dump of all current DP values.

```
55 AA 00 02 00 00 01
```

### CMD 0x03 — WiFi status

Sent during handshake to tell the MCU the WiFi module has connected to the cloud. Payload `0x04` = cloud connected, which causes the MCU to show a solid LED indicator.

```
55 AA 00 03 00 01 04 07
```

### CMD 0x06 — Send DP (feed command)

Triggers the feeding motor. The payload encodes DP3 (feed trigger) as a 4-byte integer value equal to the number of portions (1–12).

```
55 AA 00 06 00 08 03 02 00 04 00 00 00 [N] [CHECKSUM]
```

| Byte | Value | Description |
|------|-------|-------------|
| `03` | DP ID | DP3 — feed trigger |
| `02` | Type  | 4-byte integer |
| `00 04` | Length | 4 bytes |
| `00 00 00 N` | Value | Number of portions |
| Checksum | `(0x16 + N) & 0xFF` | Verified from capture: N=1→0x17, N=2→0x18, N=4→0x1A |

### CMD 0x08 — Functional test

Sent during handshake. The MCU acknowledges with its own CMD 0x08 frame.

```
55 AA 00 08 00 00 07
```

---

## Commands — MCU → ESP (we receive)

### CMD 0x00 — Heartbeat

The MCU sends this periodically to keep the connection alive. We reply immediately with the heartbeat reply frame above.

```
MCU → 55 AA 03 00 00 01 [status] [checksum]
```

### CMD 0x01 — Product info

The MCU sends its product ID string after the product info query. No reply required. Logged at DEBUG level.

### CMD 0x02 — MCU state report

Sent in response to our state query during handshake. Contains a dump of all current DP values. No reply required. Logged at DEBUG level.

### CMD 0x07 — DP status report

The main status update frame. Sent whenever a DP value changes, and also periodically as a keepalive. Payload structure:

```
[DP_ID] [DP_TYPE] [VAL_LEN_HI] [VAL_LEN_LO] [VALUE...]
```

DPs we act on:

| DP ID | Name | Values | Action |
|-------|------|--------|--------|
| DP4 `0x04` | Feeding motor state | `0x00` = idle/done, `0x01` = scheduled, `0x02` = feeding | `0x00` while `feeding_active` → set feeder sensor to Idle |
| DP25 `0x19` | Physical lock button | `0x00` = unlocked, `0x01` = locked | Update lock sensor state |

Other DPs are logged but not acted on. Note: DP4 fires periodically as a keepalive even when not feeding — the `feeding_active` guard prevents false Idle transitions.

### CMD 0x08 — Functional test ack

The MCU acknowledges our CMD 0x08 sent during handshake. No action required. Logged at VERBOSE level.

### CMD 0x1C — Time sync ack

The MCU acknowledges a time sync. Logged at VERBOSE level (no action).

### CMD 0x34 — Extended feeding log

Sent by the MCU after a feed cycle completes. Contains a DP11 record with a timestamp of the feeding event for the MCU's internal log.

Used as a **fallback feeding-done signal**: if CMD 0x07 DP4=0x00 was missed for any reason, CMD 0x34 DP11 (`0x0B`) will also clear `feeding_active` and set the feeder sensor to Idle.

---

## Commands — Not Implemented

### CMD 0x14 — Time sync request

The MCU sends this command periodically to request the current Unix timestamp from the Wi-Fi module. The original WBR3 would respond with a CMD 0x1C frame containing the current time so the MCU can timestamp feed events in its internal log accurately.

Currently this command is received, logs a warning (`Unknown CMD 0x14`), and is discarded. The MCU's feed log timestamps will therefore be incorrect or zeroed.

**Response format (not yet implemented):**
```
55 AA 00 1C 00 08 [4-byte unix timestamp] [local hour offset] [local minute offset] [checksum]
```

Implementing this would require adding an SNTP `time` component to the ESPHome config to provide a real timestamp.

---

## Data Points (DPs)

| DP | Name | Direction | Type | Notes |
|----|------|-----------|------|-------|
| DP3  `0x03` | Feed trigger | ESP → MCU | integer | Number of portions to dispense |
| DP4  `0x04` | Motor state  | MCU → ESP | enum | 0=idle, 1=scheduled, 2=feeding |
| DP10 `0x0A` | Unknown      | MCU → ESP | —    | Observed in captures, purpose unknown |
| DP11 `0x0B` | Feed log     | MCU → ESP | —    | Appears in CMD 0x34 extended report |
| DP14 `0x0E` | Unknown      | MCU → ESP | —    | Observed in captures, purpose unknown |
| DP25 `0x19` | Lock state   | MCU → ESP | bool | Physical lock button on feeder |

---

## MQTT Topics

All topics are under the prefix `wifi2mqtt/pet-feeder/`.

| Topic | Direction | Retained | Values | Description |
|-------|-----------|----------|--------|-------------|
| `availability` | ESP → HA | yes | `online` / `offline` | LWT — offline set by broker on disconnect |
| `feeder` | ESP → HA | yes | `Idle` / `Feeding` / `Error` | Feeder motor state |
| `mcu` | ESP → HA | yes | `Connected` / `Disconnected` | MCU handshake state |
| `lock` | ESP → HA | yes | `Locked` / `Unlocked` | Physical lock button state |
| `rssi` | ESP → HA | yes | dBm integer | WiFi signal strength, updated every 60s |
| `ip` | ESP → HA | no | IP address string | Device IP, updated on change |
| `feedCmd` | HA → ESP | no | `1`–`12` | Trigger a feed cycle with N portions |
| `debug` | ESP → HA | no | text | Debug messages, only published when `mqtt_debug: "true"` |

---

## Feeder State Machine

```
MQTT connect
     │
     ▼
   [Idle] ◄──────────────────────────────────────────┐
     │                                                │
     │  feedCmd received / Dispense button pressed    │
     ▼                                                │
 [Feeding]                                            │
     │                                                │
     ├── CMD07 DP4=0x00 received ────────────────────►┤
     ├── CMD34 DP11 received ─────────────────────────┤
     │                                                │
     │  wait_until feeding_active=false (max 30s)     │
     │                                                │
     ├── done in time ───────────────────────────────►┘
     │
     └── 30s timeout
           │
           ▼
        [Error]
```

---

## Configuration

At the top of `pet-feeder.yaml`:

```yaml
substitutions:
  device_name: pet-feeder
  topic_prefix: wifi2mqtt/pet-feeder
  mqtt_debug: "false"   # change to "true" for MQTT debug messages
```

Setting `mqtt_debug: "true"` publishes diagnostic messages for every significant UART frame to `wifi2mqtt/pet-feeder/debug`. Keep it `"false"` in normal operation to avoid MQTT spam.

Secrets required in `secrets.yaml`:

```yaml
wifi_ssid: "..."
wifi_password: "..."
ap_password: "..."
mqtt_broker: "..."
mqtt_port: "1883"
mqtt_user: "..."
mqtt_password: "..."
```
