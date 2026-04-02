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

The feeder uses the standard Tuya serial protocol between the Wi-Fi module and the feeder MCU. All protocol details below are confirmed from logic analyser captures of the original WBR3 module.

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
- The MCU responds with **v3 frames** (`55 AA 03 ...`), confirmed from logic analyser captures.
- The RX parser does not validate the version byte — it syncs on the `55 AA` header and reads the command from byte 3. The version byte (`buf[2]`) is accepted but ignored.
- This firmware always sends v0 frames, which the MCU accepts.

### Heartbeat status byte

The MCU's CMD 0x00 heartbeat includes a status byte:
- `0x00` — first heartbeat after power-on
- `0x01` — subsequent heartbeats during normal operation

The firmware replies with the same `55 AA 00 00 00 00 FF` regardless of status.

---

## Startup Handshake

The boot sequence is fully event-driven — the firmware waits for actual evidence the MCU is alive before proceeding at each step, rather than using fixed timing.

### Boot sequence

The firmware handles two distinct boot scenarios:

**Full power cycle (both ESP32 and feeder board powered together):**
```
Power on → on_boot: EN LOW (held)
         → WiFi + MQTT connect
         → on_connect: EN HIGH → rising edge
         → MCU wakes, sends heartbeat (status=0x00)
         → 1s delay → handshake fires
         → MCU replies CMD01 → mcu_connected = true ✅
         → MCU sends DP state dump (DP14, CMD34, DP25, DP4=0, DP10)
         → DP4=0x00 received → mcu_ready = true ✅
```

**ESP32-only reboot (OTA flash, crash, etc.):**
```
Reboot → on_boot: EN LOW (MCU already running, ignores it)
       → WiFi + MQTT connect
       → on_connect: EN HIGH (MCU ignores it, already running)
       → 1s delay → handshake fires directly
       → MCU replies CMD01 → mcu_connected = true ✅
       → MCU replies to CMD02 with DP state dump → DP4=0x00 → mcu_ready = true ✅
       → Feed commands accepted immediately after mcu_ready ✅
       → 5s retry interval keeps retrying if CMD01 never arrives ✅
```

No special detection of "ESP-only reboot" vs "full power cycle" is needed — the handshake always fires from `on_connect` and the MCU always responds to CMD02 (state query) with a full DP dump regardless of whether it just powered on or was already running. The result is identical from the firmware's perspective.

The EN pin on this board is only sampled at MCU power-on. Driving EN LOW briefly during an ESP reboot has no effect on an already-running MCU.

### MCU readiness

`mcu_connected` (CMD01 received) and `mcu_ready` (DP4=0x00 received in startup dump) are tracked separately. CMD01 arrives fast but the motor controller takes longer to initialise. The feed script waits up to 15 seconds for `mcu_ready` before sending a feed command, preventing a silent ignore on feeds triggered immediately after a power cycle.

### Disconnect / reconnect

On MQTT disconnect the EN pin is driven **LOW** and `mcu_connected`, `mcu_heartbeat_seen` and `mcu_ready` are all cleared. On the next MQTT reconnect the full sequence repeats.

### Handshake frames

Confirmed from WBR3 logic analyser capture. Sent in order with 200ms delays between each:

| Step | Frame (hex) | Purpose |
|------|------------|---------|
| 1 | `55 AA 00 00 00 00 FF` | Heartbeat |
| 2 | `55 AA 00 01 00 00 00` | Product info query |
| 3 | `55 AA 00 02 00 00 01` | MCU state query |
| 4 | `55 AA 00 03 00 01 04 07` | WiFi status: cloud connected |
| 5 | `55 AA 00 08 00 00 07` | Functional test (CMD 0x08) |
| 6 | `55 AA 00 34 00 02 0B 00 40` | CMD34 DP11 ack (sent twice) |

Steps 1–5 match the WBR3 startup sequence. Step 6 (CMD34 ack, sent twice) clears any pending feed acknowledgement state in the MCU — confirmed from WBR3 capture where this was sent at every startup.

### Connection confirmation

`mcu_connected` is set to `true` when the MCU responds with a **CMD 0x01 product info frame**. CMD 0x01 can arrive before the handshake script finishes sending all frames (the MCU responds to frame 2 almost immediately), so the handler always sets connected unconditionally.

If CMD 0x01 never arrives, `mcu_connected` stays `false` and the **5-second retry interval** re-fires the handshake automatically.

---

## Commands — ESP → MCU (we send)

### CMD 0x00 — Heartbeat reply

Sent in response to every heartbeat received from the MCU.
```
55 AA 00 00 00 00 FF
```

### CMD 0x01 — Product info query

Sent during handshake. MCU responds with its product ID string.
```
55 AA 00 01 00 00 00
```

### CMD 0x02 — MCU state query

Sent during handshake. MCU responds with a dump of all current DP values.
```
55 AA 00 02 00 00 01
```

### CMD 0x03 — WiFi status

Sent during handshake and in response to MCU WiFi status queries. Payload `0x04` = cloud connected.
```
55 AA 00 03 00 01 04 07
```

### CMD 0x06 — Send DP (feed command)

Triggers the feeding motor. The payload encodes DP3 (feed trigger) as a 4-byte integer equal to the number of portions (1–12).
```
55 AA 00 06 00 08 03 02 00 04 00 00 00 [N] [CHECKSUM]
```

| Byte | Value | Description |
|------|-------|-------------|
| `03` | DP ID | DP3 — feed trigger |
| `02` | Type  | 4-byte integer |
| `00 04` | Length | 4 bytes |
| `00 00 00 N` | Value | Number of portions |
| Checksum | `(0x16 + N) & 0xFF` | N=1→0x17, N=2→0x18, N=3→0x19, N=4→0x1A, N=6→0x1C |

**Important:** Immediately after CMD 0x06, the firmware sends a CMD 0x34 DP11 ack. This is required — confirmed from WBR3 captures where the ack was always sent proactively right after the feed command, not after receiving CMD 0x34 from the MCU. Without this ack the MCU may silently ignore the feed command after extended uptime.

### CMD 0x08 — Functional test

Sent during handshake. The MCU acknowledges with its own CMD 0x08 frame.
```
55 AA 00 08 00 00 07
```

### CMD 0x1C — Time sync reply

Sent in response to CMD 0x1C from the MCU. The payload is always **8 bytes** — confirmed from WBR3 logic analyser captures. Sending fewer bytes while declaring `len=8` in the header causes the MCU's frame parser to read past the frame boundary on every response, continuously corrupting the UART stream.

```
55 AA 00 1C 00 08 [4-byte UTC unix time, big-endian] [local hour] [local minute] [UTC offset hours] [UTC offset minutes=0x00] [checksum]
```

Total frame length: **15 bytes** (6 header + 8 payload + 1 checksum).

| Byte(s) | Field | Description |
|---------|-------|-------------|
| `00 08` | Length | Payload is always 8 bytes |
| `XX XX XX XX` | UTC unix time | Current UTC timestamp, big-endian 32-bit |
| `XX` | Local hour | Current local hour (0–23), timezone-adjusted |
| `XX` | Local minute | Current local minute (0–59) |
| `XX` | UTC offset hours | Hours east of UTC (e.g. `0x01`=CET, `0x02`=CEST) |
| `0x00` | UTC offset minutes | Always `0x00` for Stockholm |

The UTC offset is derived at runtime from the difference between the SNTP-synced local time and UTC, so CET (+1) and CEST (+2) are handled automatically.

If SNTP has not yet synced, a zeroed 8-byte payload is sent (`55 AA 00 1C 00 08 00 00 00 00 00 00 00 00 23`) and the MCU retries on the next request.

### CMD 0x34 — Feed acknowledgement

Sent in two situations, confirmed from WBR3 captures:

1. **Proactively after every CMD 0x06** — immediately after the feed command, before the MCU responds. Required for the MCU to accept the feed.
2. **Twice during handshake** — clears any pending ack state from before the ESP32 connected.
```
55 AA 00 34 00 02 0B 00 40
```

---

## Commands — MCU → ESP (we receive)

### CMD 0x00 — Heartbeat

Sent approximately every 250ms. Status byte `0x00` on first heartbeat after power-on, `0x01` thereafter. We reply immediately with the heartbeat reply frame.
```
55 AA 03 00 00 01 [status] [checksum]
```

### CMD 0x01 — Product info

The MCU sends its product ID string (JSON payload) in response to our product info query. Sets `mcu_connected = true`. Logged at DEBUG level.

### CMD 0x02 — MCU state report

Sent in response to our state query during handshake. On this board the payload is always empty (0 bytes) — the MCU sends its actual state via separate CMD 0x07 DP frames immediately after.

### CMD 0x03 — WiFi status query

The MCU asks us to resend our WiFi status. We reply with the cloud-connected frame.
```
MCU  → 55 AA 03 03 00 00 ...
ESP  → 55 AA 00 03 00 01 04 07
```

### CMD 0x07 — DP status report

The main status update frame. Sent whenever a DP value changes, and periodically as a keepalive. Payload structure:
```
[DP_ID] [DP_TYPE] [VAL_LEN_HI] [VAL_LEN_LO] [VALUE...]
```

DPs we act on:

| DP ID | Name | Values | Action |
|-------|------|--------|--------|
| DP3 `0x03` | Feed trigger echo | 4-byte int = portions | Logged — confirms MCU received feed command |
| DP4 `0x04` | Motor state | `0x00`=idle, `0x01`=starting, `0x02`=feeding | `0x00` while `feeding_active` → Idle. `0x00` during startup → `mcu_ready=true` |
| DP25 `0x19` | Lock state | `0x00`=unlocked, `0x01`=locked | Update lock sensor only. Does not affect UART feeds |

**Confirmed WiFi feed sequence from capture:**
```
MCU → DP3=N echo       (confirms command received)
MCU → DP4=0x01         (motor starting)
MCU → CMD34 DP11       (feed log with timestamp)
MCU → DP4=0x02         (motor feeding)
MCU → DP4=0x00         (motor done)
```

**Confirmed physical button feed sequence from capture:**
```
MCU → DP25=0x00        (button unlocked)
MCU → DP4=0x01         (motor starting)
MCU → CMD34 DP11       (feed log)
MCU → DP4=0x02         (motor feeding)
MCU → DP4=0x00         (motor done)
MCU → DP25=0x01        (auto-locked after 5s)
```

Note: physical button press does not generate a DP3 echo — DP3 is only echoed for UART-triggered feeds.

### CMD 0x08 — Functional test ack

The MCU acknowledges our CMD 0x08 sent during handshake. No action required. Logged at VERBOSE level.

### CMD 0x1C — Time sync request

The MCU sends this every second to request the current Unix timestamp. Payload is empty (`55 AA 03 1C 00 00 1E`). We respond with CMD 0x1C containing UTC time, local hour/minute, and UTC offset from SNTP. Logged at VERBOSE level only — log line includes `unix=`, `local=HH:MM`, and `tz=+N`.

### CMD 0x34 — Extended feeding log

Sent by the MCU after every feed cycle (both UART-triggered and physical button). Contains a DP11 record with a timestamp and portion count.
```
55 AA 03 34 00 11 0B 01 [year_lo] [month] [day] [hour] [min] [sec] [type] [len_hi] [len_lo] [portions] [checksum]
```

Used as the **primary feeding-done signal** — clears `feeding_active` and sets feeder sensor to Idle. CMD 0x07 DP4=0x00 serves as secondary confirmation.

After receiving CMD 0x34 from the MCU, the firmware sends the CMD 0x34 ack back. Note that the WBR3 sends this ack proactively after CMD 0x06, so the ack is sent twice per feed cycle in total.

---

## Data Points (DPs)

| DP | Name | Direction | Type | Notes |
|----|------|-----------|------|-------|
| DP3  `0x03` | Feed trigger | ESP → MCU | integer | Portions to dispense. MCU echoes this back on UART feeds |
| DP4  `0x04` | Motor state  | MCU → ESP | enum | 0=idle, 1=starting, 2=feeding. Sequence: 1→CMD34→2→0 |
| DP10 `0x0A` | Unknown | MCU → ESP | 4-byte int | Value always `100`. Fires once per feed and every ~60s as keepalive |
| DP11 `0x0B` | Feed log | MCU → ESP | — | Appears in CMD 0x34. Contains timestamp and portion count |
| DP14 `0x0E` | Unknown | MCU → ESP | type=0x05 | Always `0x00`. Sent in startup dump only |
| DP25 `0x19` | Lock state | MCU → ESP | bool | Physical lock button. 0=unlocked, 1=locked. Auto-locks 5s after press. Does not block UART feeds |

---

## MQTT Topics

All topics are under the prefix `wifi2mqtt/pet-feeder/`.

### State topics (ESP → HA)

| Topic | Retained | Values | Description |
|-------|----------|--------|-------------|
| `availability` | yes | `online` / `offline` | LWT — broker sets `offline` on disconnect, firmware sets `online` on connect |
| `feeder` | yes | `Idle` / `Feeding` / `Error` | Feeder motor state. Set by MCU responses — never by a timer |
| `mcu` | yes | `Connected` / `Disconnected` | MCU handshake state |
| `lock` | yes | `Locked` / `Unlocked` | Physical lock button on feeder body. Display only — does not affect remote feeding |
| `rssi` | yes | dBm integer | WiFi signal strength, updated every 60s |
| `ip` | no | IP address string | Device IP address, updated on change |
| `debug` | no | text | Serial log mirror — published when `mqtt_debug` is set to the topic string |

### Command topics (HA → ESP)

| Topic | Values | Description |
|-------|--------|-------------|
| `feedCmd` | `1`–`12` | Trigger a feed cycle. Payload is the number of portions to dispense |

### Home Assistant discovery

ESPHome publishes MQTT discovery messages automatically to `homeassistant/sensor/pet-feeder/...` on every connect. This creates the following entities in Home Assistant:

| Entity | Type | Topic |
|--------|------|-------|
| Feeder Status | Sensor | `wifi2mqtt/pet-feeder/feeder` |
| MCU Status | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/mcu` |
| Lock Status | Sensor | `wifi2mqtt/pet-feeder/lock` |
| RSSI | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/rssi` |
| IP | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/ip` |
| Portion Size | Number | `wifi2mqtt/pet-feeder/number/portion_size/state` |
| Dispense Meal | Button | `wifi2mqtt/pet-feeder/button/dispense_meal/command` |

The `state_topic` for each sensor is explicitly set in the firmware to match the clean topic paths above, ensuring discovery config and state publishes always use the same topic.

### Topic notes

- All state topics use `retain: true` so Home Assistant always has the last known state after a restart.
- `availability` uses the MQTT LWT mechanism — the broker automatically publishes `offline` on unexpected disconnect.
- `feeder` is only set to `Idle` by MCU responses (CMD34 DP11 or CMD07 DP4=0x00), never on a timer. `Error` is set on 30-second timeout or cooldown block.

---

## Feeder State Machine
```
MQTT connect
     │
     ▼
  [wait mcu_ready] ── 15s timeout ──► [Error]
     │
     ▼
   [Idle] ◄──────────────────────────────────────────┐
     │                                                │
     │  feedCmd received / Dispense button pressed    │
     ├── cooldown active (<30s since last) ──────────► [Error] (immediate)
     ▼                                                │
 [Feeding]                                            │
     │                                                │
     ├── CMD34 DP11 received ─────────────────────────┤  (primary done signal)
     ├── CMD07 DP4=0x00 received ────────────────────►┘  (secondary done signal)
     │
     │  wait_until feeding_active=false (max 30s)
     │
     └── 30s timeout — no done signal received
           │
           ▼
        [Error] → handshake reset
```

`Idle` is set exclusively by the MCU. `Error` is set on timeout or cooldown block.

---

## Feed Guards

The feed script checks two conditions before sending a feed command, in order:

1. **Cooldown guard** — a 30-second cooldown is enforced between feeds. This prevents rapid re-triggers from HA automations and gives the MCU time to fully complete a feed cycle before the next one begins. If blocked, feeder status is set to `Error` and the script stops immediately.

2. **MCU ready guard** — waits up to 15 seconds for `mcu_ready` (DP4=0x00 received in startup dump). This prevents silent ignores on feeds triggered immediately after a power cycle before the motor controller has initialised. If the timeout expires without `mcu_ready`, feeder status is set to `Error` and the script stops.

If either guard blocks, no feed frame is sent.

Note: the physical lock button state is tracked for display purposes but has no effect on remote feeding — the lock only disables the physical buttons on the feeder body.

---

## Known MCU Behaviour

### CMD34 ack requirement

The original WBR3 sends a CMD 0x34 DP11 ack immediately after every CMD 0x06, before the MCU has responded. Without this proactive ack the MCU may silently ignore feed commands after extended uptime. The firmware replicates this behaviour exactly, confirmed from logic analyser captures. Two CMD34 acks are also sent during the handshake to clear any pending state.

### Motor controller initialisation delay

After a full power cycle, the MCU responds to the handshake (CMD01) quickly but the motor controller takes 1–2 additional seconds to initialise. Sending a feed command before the motor controller is ready results in a silent ignore. The firmware tracks `mcu_ready` separately from `mcu_connected` and waits for DP4=0x00 in the startup dump before allowing feed commands.

### Physical button lock behaviour

The physical lock button only disables the buttons on the feeder body. UART feed commands are accepted regardless of lock state — the lock does not block remote feeding.

### CMD 0x1C payload length — stream corruption bug (fixed)

The MCU sends a time sync request every second. The correct response payload is **8 bytes**, as confirmed by WBR3 logic analyser captures. An earlier version of this firmware declared `len=8` in the frame header but only wrote 6 bytes of payload, producing a 13-byte frame where the MCU expected 15. On every time sync response the MCU consumed 2 bytes from the next UART frame as part of the CMD 0x1C payload, continuously desynchronising the stream.

Because CMD 0x1C fires every second this corruption was permanent and affected every frame that followed — including the CMD01 handshake response, DP status reports, and CMD34 feeding logs. Symptoms were startup failures (handshake never completing, `mcu_ready` never set) and remote feed commands silently failing.

The fix sends all 8 payload bytes: `[4B UTC unix timestamp] [local hour] [local minute] [UTC offset hours] [UTC offset minutes]`, matching the WBR3 exactly.

---

## Debugging

### MQTT log mirror

Set `mqtt_debug` in the substitutions to the debug topic to mirror the serial log to MQTT:

```yaml
mqtt_debug: "wifi2mqtt/pet-feeder/debug"   # enable
mqtt_debug: ""                              # disable (default)
```

This uses ESPHome's native `logger: mqtt_topic:` feature — it forwards every log message that passes the configured `logger: level:` filter to the MQTT topic. No separate message list; whatever you see on serial you also get on MQTT. Set back to `""` for normal operation.

### Serial log verbosity

Set `level` in the `logger:` block in `pet-feeder.yaml`:

```yaml
logger:
  level: WARN     # quiet   — only warnings and errors
  level: INFO     # normal  — key lifecycle events and feeding activity
  level: DEBUG    # verbose — full protocol detail, all frames
  level: VERBOSE  # maximum — every heartbeat, ack frame, CMD1C time sync
```

Pick one. What each level shows:

| Level | What you see |
|-------|-------------|
| `WARN` | Feed blocked/timeout, MCU not connected retries, unknown commands |
| `INFO` | Boot/handshake milestones, MCU connected/ready, SNTP synced, feed sent, motor starting, feeding done |
| `DEBUG` | Everything above + every CMD07 DP report, CMD02/CMD03 detail, lock raw state, handshake frame-by-frame |
| `VERBOSE` | Everything above + every heartbeat, every CMD34/CMD06 ack frame, CMD1C time sync |

**INFO output for a normal feed cycle:**
```
[I] Boot: EN LOW
[I] EN pin HIGH — starting handshake
[I] MCU connected
[I] MCU ready — motor controller initialised
[I] SNTP time synchronised
[I] Feed sent: 2 portions
[I] MCU echoed feed: 2 portions
[I] Motor starting
[I] Feeding done — CMD34 DP11
```

### Common error states

**Feeder stuck on "Feeding" / feed button does nothing after one use:**
The `feed_script` runs in `mode: single`. If the script timed out, a reboot will reset the state.

**"MCU not ready" error immediately after power cycle:**
The motor controller hasn't finished initialising. The script waits up to 15s — if this appears consistently, check that the MCU is receiving power and the EN pin wiring is correct.

**"Feed blocked — cooldown" in logs / feeder shows Error:**
A feed was attempted within 30 seconds of the previous one. Wait and retry.

**CMD1C time sync spam at VERBOSE level:**
Normal — the MCU requests time sync every second. Silent at DEBUG level. Log line format: `CMD1C time sync: unix=NNNN local=HH:MM tz=+N cs=0xXX`.

**MCU not connecting / handshake not completing:**
Check the log for `CMD01 product info received`. If absent, check GPIO10 (EN pin) wiring. The 5-second retry interval will keep retrying automatically.

---

## Configuration

At the top of `pet-feeder.yaml`:
```yaml
substitutions:
  device_name: pet-feeder
  topic_prefix: wifi2mqtt/pet-feeder
  mqtt_debug: ""   # set to "wifi2mqtt/pet-feeder/debug" to mirror serial log to MQTT
```

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
