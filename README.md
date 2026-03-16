# Pet Feeder — ESPHome Firmware

ESPHome firmware for a Tuya-based automatic pet feeder (Miaosical SC-A80W-DW), replacing the original WBR3 Wi-Fi module with an ESP32-C3 Super Mini.

<img width="449" height="356" alt="image" src="https://github.com/user-attachments/assets/e5fef84b-e27f-462e-b66e-f866c2c98e26" />

This is an [ESPHome](https://esphome.io/) project — ESPHome is a system for programming ESP32/ESP8266 microcontrollers using YAML configuration files, compiling to native firmware with no custom C++ required for most use cases. The ESP32-C3 speaks the Tuya serial protocol directly to the feeder control board over UART, publishes all state and control topics to an MQTT broker, and automatically appears as a device in [Home Assistant](https://www.home-assistant.io/) via MQTT discovery.

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

The ESP32-C3 Super Mini is used here, but any ESP32 board will work — just update the GPIO pin numbers in the YAML to match your board's pinout.

<img width="475" height="380" alt="image" src="https://github.com/user-attachments/assets/bdd40574-5195-4e42-89a1-cfb5a307780d" />

<img width="390" height="395" alt="image" src="https://github.com/user-attachments/assets/9c042213-1d4c-42ea-8518-d168a61965e1" />

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

The boot sequence is fully event-driven — the firmware waits for actual evidence the MCU is alive before proceeding at each step, rather than using fixed timing.

### Boot sequence

```
MQTT connects
     │
     ▼
500ms delay (let MCU settle)
     │
     ▼
EN pin → HIGH  (GPIO10)
MCU starts sending heartbeats
     │
     ▼
First heartbeat received
→ handshake script fires
     │
     ▼
5-frame handshake sent (200ms between each frame):
  55 AA 00 00 00 00 FF        heartbeat
  55 AA 00 01 00 00 00        product info query
  55 AA 00 02 00 00 01        MCU state query
  55 AA 00 03 00 01 04 07     WiFi status: cloud connected
  55 AA 00 08 00 00 07        functional test
     │
     ▼
MCU replies with CMD 0x01 (product info string)
→ mcu_connected = true
→ MCU sensor set to "Connected"
     │
     ▼
MCU begins sending regular DP status reports
```

### Disconnect / reconnect

On MQTT disconnect the EN pin is driven **LOW**, which resets the MCU's communication state cleanly. Both `mcu_connected` and `mcu_heartbeat_seen` are cleared. On the next MQTT connect the full sequence above repeats from the beginning.

### Handshake frames

| Step | Frame (hex) | Purpose |
|------|------------|---------|
| 1 | `55 AA 00 00 00 00 FF` | Heartbeat |
| 2 | `55 AA 00 01 00 00 00` | Product info query |
| 3 | `55 AA 00 02 00 00 01` | MCU state query |
| 4 | `55 AA 00 03 00 01 04 07` | WiFi status: cloud connected |
| 5 | `55 AA 00 08 00 00 07` | Functional test (CMD 0x08) |

### Connection confirmation

`mcu_connected` is only set to `true` when the MCU responds with a **CMD 0x01 product info frame** — this is the real confirmation that the MCU received the handshake frames. Simply sending the frames and waiting is not enough; the MCU must prove it responded.

If the MCU never sends CMD 0x01, `mcu_connected` stays `false` and the 15-second retry interval will re-fire the handshake automatically.

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

### CMD 0x03 — WiFi status query

The MCU sends this after a reconnect to ask the Wi-Fi module to resend its WiFi status. We reply with the same cloud-connected frame used in the handshake.

```
MCU  → 55 AA 03 03 00 00 ...
ESP  → 55 AA 00 03 00 01 04 07   (cloud connected)
```

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

## Commands — Implemented (MCU → ESP, requires response)

### CMD 0x14 — Time sync request

The MCU sends this command periodically to request the current Unix timestamp so it can timestamp feed events in its internal log.

This is now implemented. The ESP32 runs an SNTP client (`pool.ntp.org`) and responds with a CMD 0x1C frame containing the current UTC Unix timestamp. If SNTP has not yet synced at the time of the request, a zeroed timestamp is sent and the MCU will retry.

**Response frame (CMD 0x1C):**
```
55 AA 00 1C 00 08 [4-byte unix time, big-endian] [hour offset] [minute offset] [checksum]
```

The firmware sends UTC time with hour/minute offset both set to zero. This means the MCU's internal feed log timestamps will be in UTC rather than local time, which is acceptable for our purposes.

**SNTP configuration:** `pool.ntp.org` and `time.google.com` are used as servers. The timezone is set to `Europe/Stockholm` in the ESPHome config — adjust this to your local timezone if needed.

---

## Data Points (DPs)

| DP | Name | Direction | Type | Notes |
|----|------|-----------|------|-------|
| DP3  `0x03` | Feed trigger | ESP → MCU | integer | Number of portions to dispense |
| DP4  `0x04` | Motor state  | MCU → ESP | enum | 0=idle, 1=scheduled, 2=feeding |
| DP10 `0x0A` | Unknown (counter?) | MCU → ESP | 4-byte int | Fires once per feed cycle and as a 60s keepalive; always `0` so far — purpose unknown |
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

## Debugging

### MQTT debug messages

Set `mqtt_debug: "true"` in the substitutions at the top of the YAML. This publishes diagnostic messages for every significant UART frame to `wifi2mqtt/pet-feeder/debug`. Events covered: feed frame sent, CMD07 DP values, CMD34 extended report, CMD14 time sync, and any unknown commands received. Set back to `"false"` for normal operation.

### Serial log verbosity

The `logger` component controls what appears in the ESPHome serial console (and the ESPHome dashboard log viewer). Change the level in the YAML:

```yaml
logger:
  level: DEBUG    # normal — shows feeding events, DP changes, handshake
  level: VERBOSE  # maximum — shows every heartbeat, CMD08 ack, CMD1C ack
```

Available levels from quietest to loudest: `ERROR`, `WARN`, `INFO`, `DEBUG`, `VERBOSE`.

### Common error states

**Feeder stuck on "Feeding" / feed button does nothing after one use:**
The `feed_script` runs in `mode: single` — it cannot be triggered again while already running. If the script timed out or errored, check that `feeding_active` was cleared. A reboot will always reset the state.

**"Unknown CMD 0x14" warnings in log:**
The MCU is requesting a time sync. This is now handled — after flashing the updated firmware with SNTP, these warnings will be replaced by `CMD14 time sync` debug messages.

**MCU not connecting / handshake not completing:**
The firmware waits for the first heartbeat from the MCU before sending the handshake. If the MCU never sends a heartbeat, the EN pin (GPIO10) may not be wired correctly or the MCU may not be powered. The 15-second retry interval will re-attempt the handshake automatically. Check the log for "First heartbeat — triggering handshake" to confirm the MCU is responding.

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
