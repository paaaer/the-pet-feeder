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
---> I noticed that the U2 component is an LDO (Low-Dropout Regulator). It could be that it not enough power wise for the ESP32 C3. So I connected the board directlly to the v5 volatge rail. This works better.

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

**Scenario 1 — Full power cycle (both boards cold start):**
```
on_boot
  EN pin → LOW
  mcu_ready = false, mcu_connected = false, feeding_active = false
       ↓
WiFi connects (~2–5s)
       ↓
MQTT connects → on_connect
  feeder = "Idle", mcu = "Disconnected", lock = "Unlocked"
  EN pin → HIGH   ← MCU wakes, starts sending heartbeats
  delay 1s → do_handshake (7 frames)
       ↓
MCU responds CMD01
  mcu_connected = true
  mcu_ready is false → mcu = "Initialising"
  mcu_ready_fallback starts (8s timer)
       ↓
MCU sends startup DP dump → DP4=0x00 arrives
  mcu_ready = true, mcu = "Connected"   ← fallback cancelled by condition check
       ↓
Ready for feeds ✓
```

**Scenario 2 — ESP-only reboot (OTA flash, crash, scheduled reboot):**

The MCU stays powered and running throughout. `on_boot` clears `mcu_ready`, but the idle MCU has no motor state change to report, so it never sends DP4=0x00 after the handshake. The `mcu_ready_fallback` carries this case:
```
on_boot: mcu_ready = false
       ↓
MQTT connects → do_handshake
       ↓
CMD01 arrives
  mcu_ready is false → mcu = "Initialising"
  mcu_ready_fallback starts (8s timer)
  ... MCU idle, no DP4=0x00 sent ...
  8s later: mcu_ready = true, mcu = "Connected"
       ↓
Ready for feeds ✓
```

**Scenario 3 — MQTT network blip (brief disconnect/reconnect):**

`mcu_ready` is intentionally **not** cleared on MQTT disconnect — the motor controller state has nothing to do with whether the broker is reachable. Clearing it here was the root cause of intermittent morning feed failures after overnight broker hiccups.
```
MQTT drops → on_disconnect
  EN pin → LOW
  mcu_connected = false
  mcu_ready stays true  ← not cleared
       ↓
MQTT reconnects → on_connect
  EN pin → HIGH → do_handshake
       ↓
CMD01 arrives
  mcu_ready already true → mcu = "Connected" immediately
  no fallback needed
       ↓
Ready for feeds instantly ✓
```

**Scenario 4 — Feed triggered during initialisation window:**

If a scheduled feed fires during the 8s fallback window after an ESP reboot:
```
feed_script starts
  wait_until mcu_ready — timeout 15s
    → fallback fires at ~8s → mcu_ready = true
  proceeds immediately ✓
```

If `mcu_ready` is still false after the full 15s wait, the feed script does one automatic recovery attempt before aborting:
```
  → WARN: "not ready after 15s — handshake reset, retrying once"
  → do_handshake runs → new CMD01 → mcu_ready_fallback restarts
  → wait_until mcu_ready — timeout 15s
    → fallback fires at ~8s → mcu_ready = true
  proceeds ✓

  If still nothing after second 15s wait:
  → Error published to MQTT last_error with "after retry" in message
  → feed aborted
```

### MCU status during startup

The `mcu` sensor reflects the ready state in real time:

| `wifi2mqtt/pet-feeder/mcu` | Meaning |
|---|---|
| `Disconnected` | MQTT just connected, handshake not yet sent |
| `Initialising` | CMD01 received — waiting for DP4=0x00 or 8s fallback |
| `Connected` | Motor controller ready — feeds accepted |

### MCU readiness

`mcu_connected` (CMD01 received) and `mcu_ready` (DP4=0x00 or 8s fallback) are tracked separately. CMD01 arrives fast but the motor controller takes longer to initialise on a cold start. The feed script waits up to 15 seconds for `mcu_ready` before sending any feed command, with one automatic retry if the first wait expires.

### Disconnect / reconnect

On MQTT disconnect the EN pin is driven **LOW** and `mcu_connected` and `mcu_heartbeat_seen` are cleared. `mcu_ready` is intentionally preserved — it is only reset by `on_boot` (a full ESP restart), not by transient broker disconnects.

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
| `availability` | yes | `online` / `offline` / `restarting` | LWT — broker sets `offline` on disconnect, firmware sets `online` on connect. `restarting` is published just before a scheduled reboot so it can be distinguished from a crash |
| `feeder` | yes | `Idle` / `Feeding` / `Error` | Feeder motor state. Set by MCU responses — never by a timer |
| `mcu` | yes | `Disconnected` / `Initialising` / `Connected` | MCU handshake state. `Initialising` = CMD01 received, waiting for motor controller ready |
| `lock` | yes | `Locked` / `Unlocked` | Physical lock button on feeder body. Display only — does not affect remote feeding |
| `last_error` | yes | timestamped string | Last error reason — retained, persists across reboots. See [Last error topic](#last-error-topic) |
| `rssi` | yes | dBm integer | WiFi signal strength, updated every 60s |
| `ip` | no | IP address string | Device IP address, updated on change |
| `debug` | no | text | Serial log mirror — published when `logger: mqtt_topic:` is configured |

### Command topics (HA → ESP)

| Topic | Values | Description |
|-------|--------|-------------|
| `feedCmd` | `1`–`12` | Trigger a feed cycle. Payload is the number of portions to dispense |
| `number/feed_max_retries/command` | `0`–`5` | Set the number of MCU reset+retry attempts on feed timeout |
| `switch/auto-reboot/command` | `ON` / `OFF` | Enable or disable the scheduled daily reboot |
| `number/reboot-hour/command` | `0`–`23` | Hour of day to reboot (local time). Default `3` = 03:00 |

### Home Assistant discovery

ESPHome publishes MQTT discovery messages automatically to `homeassistant/sensor/pet-feeder/...` on every connect. This creates the following entities in Home Assistant:

| Entity | Type | Topic |
|--------|------|-------|
| Feeder Status | Sensor | `wifi2mqtt/pet-feeder/feeder` |
| MCU Status | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/mcu` — values: `Disconnected` / `Initialising` / `Connected` |
| Lock Status | Sensor | `wifi2mqtt/pet-feeder/lock` |
| Last Error | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/last_error` |
| RSSI | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/rssi` |
| IP | Sensor (diagnostic) | `wifi2mqtt/pet-feeder/ip` |
| Portion Size | Number (config) | `wifi2mqtt/pet-feeder/number/portion_size/state` |
| Feed Max Retries | Number (config) | `wifi2mqtt/pet-feeder/number/feed_max_retries/state` — default 3, range 0–5 |
| Reboot Hour | Number (config) | `wifi2mqtt/pet-feeder/number/reboot-hour/state` — default 3, range 0–23 |
| Auto Reboot | Switch (config) | `wifi2mqtt/pet-feeder/switch/auto-reboot/state` — default ON |
| Dispense Meal | Button | `wifi2mqtt/pet-feeder/button/dispense_meal/command` |

The `state_topic` for each sensor is explicitly set in the firmware to match the clean topic paths above, ensuring discovery config and state publishes always use the same topic.

### Topic notes

- All state topics use `retain: true` so Home Assistant always has the last known state after a restart.
- `availability` uses the MQTT LWT mechanism — the broker automatically publishes `offline` on unexpected disconnect.
- `feeder` is only set to `Idle` by MCU responses (CMD34 DP11 or CMD07 DP4=0x00), never on a timer. `Error` is set on 30-second timeout or cooldown block. Each time `Error` is set, the reason and timestamp are also published to `last_error`.
- `last_error` is retained at the broker, so even after an ESP reboot you can query it to see what went wrong before the restart.

---

## Feeder State Machine
```
MQTT connect
     │
     ▼
  [wait mcu_ready] ── 15s timeout ──► [Error]
     │
     ▼
   [Idle] ◄──────────────────────────────────────────────────────┐
     │                                                            │
     │  feedCmd received / Dispense button pressed                │
     ├── cooldown active (<30s since last) ──────────────────► [Error] (immediate)
     ▼                                                            │
 [Feeding]                                                        │
     │                                                            │
     ├── CMD34 DP11 received ─────────────────────────────────────┤  (primary)
     ├── CMD07 DP4=0x00 received ────────────────────────────────►┘  (secondary)
     │
     │  wait_until feeding_active=false (max 30s)
     │
     └── 30s timeout — no done signal from MCU
           │
           ▼
        [Error]
           │
           ├── retries remaining? yes
           │     → EN LOW → 2s → EN HIGH → do_handshake
           │     → wait mcu_ready → retry feed ──────────────────► [Feeding]
           │     (repeats up to Feed Max Retries times)
           │
           └── retries exhausted → last_error: "all retries exhausted"
```

`Idle` is set exclusively by the MCU. `Error` is set on timeout or cooldown block. Feed timeouts trigger an automatic MCU reset and retry cycle before giving up.

---

## Feed Guards

The feed script checks two conditions before sending a feed command, in order:

1. **Cooldown guard** — a 30-second cooldown is enforced between feeds. This prevents rapid re-triggers from HA automations and gives the MCU time to fully complete a feed cycle before the next one begins. If blocked, feeder status is set to `Error` and the script stops immediately. Cooldown is skipped on automatic retries (same feed cycle).

2. **MCU ready guard** — waits up to 15 seconds for `mcu_ready` (DP4=0x00 received in startup dump). This prevents silent ignores on feeds triggered immediately after a power cycle before the motor controller has initialised. If the timeout expires without `mcu_ready`, one automatic handshake retry is attempted before aborting.

If either guard blocks, no feed frame is sent.

Note: the physical lock button state is tracked for display purposes but has no effect on remote feeding — the lock only disables the physical buttons on the feeder body.

---

## Feed Retry and MCU Reset

The MCU enters a degraded UART state after extended uptime where it receives the feed frame but does not respond. This appears as a 30-second feed timeout — the feeder shows `Error` and `last_error` shows `Feed timeout — no done signal from MCU`.

When a timeout occurs the firmware automatically resets the MCU and retries the feed:

```
30s timeout
  │
  ├── retries remaining (default 3)?
  │     │
  │     ▼
  │   last_error: "Feed timeout — retrying (N left)"
  │   EN LOW → mcu_connected=false, mcu_ready=false
  │   2s delay
  │   EN HIGH → 1s delay → do_handshake
  │   wait_until mcu_ready (up to 20s)
  │   feed_script re-executes with same portions
  │     │
  │     ├── success → Idle ✓
  │     └── timeout again → repeat until retries exhausted
  │
  └── retries exhausted
        last_error: "Feed failed — all retries exhausted"
```

The EN pin cycle (LOW → 2s → HIGH) forces the MCU to re-initialise its UART communication layer. A plain handshake without EN cycling is not sufficient to recover from this state.

Each retry attempt uses the same portion count as the original command. Retries are transparent to HA — the feeder sensor stays on `Error` during the reset/retry cycle and returns to `Idle` on success.

### Configuration

| Entity | Default | Range | Description |
|--------|---------|-------|-------------|
| `Feed Max Retries` | 3 | 0–5 | Number of MCU reset+retry attempts before giving up. Set to `0` to disable retries. |

Configure via MQTT: `wifi2mqtt/pet-feeder/number/feed_max_retries/command` → send `0`–`5`.

---

## Scheduled Daily Reboot

The firmware supports an optional daily reboot at a configurable hour. This keeps the ESP32 fresh over long uptime (WiFi stack, memory) and gives the MCU a clean reset cycle.

### Reboot sequence

```
on_time fires at configured hour (:00)
  Auto Reboot switch ON?
    │
    ▼
  availability → "restarting"   ← distinguishes from crash in MQTT logs
  EN LOW
  mcu_connected = false, mcu_ready = false
  2s delay                      ← MCU UART state fully resets
  ESP.restart()
    │
    ▼
  on_boot: EN stays LOW
  WiFi + MQTT reconnect
  on_connect: EN HIGH → do_handshake
  Normal operation resumes ✓
```

The EN LOW → 2s → restart pattern is the same used by the feed retry MCU reset (`mcu_reset_and_retry`), extracted into the shared `mcu_reboot` script. A plain `ESP.restart()` without the EN hold is not used — the 2s LOW period is required for the MCU to fully reinitialise its UART state.

After the reboot, `availability` transitions `restarting` → `offline` → `online`. A crash or brownout reboot shows only `offline` → `online` with no `restarting` prefix.

### Configuration

| Entity | Default | Description |
|--------|---------|-------------|
| `Auto Reboot` switch | ON | Enable or disable the daily reboot entirely |
| `Reboot Hour` number | `3` (03:00) | Local hour to reboot. Set well clear of scheduled feed times |

Configure via MQTT:
- `wifi2mqtt/pet-feeder/switch/auto-reboot/command` → `ON` / `OFF`
- `wifi2mqtt/pet-feeder/number/reboot-hour/command` → `0`–`23`

Both settings are persisted in flash — survive reboots without reflashing.

---

## Known MCU Behaviour

### CMD34 ack requirement

The original WBR3 sends a CMD 0x34 DP11 ack immediately after every CMD 0x06, before the MCU has responded. Without this proactive ack the MCU may silently ignore feed commands after extended uptime. The firmware replicates this behaviour exactly, confirmed from logic analyser captures. Two CMD34 acks are also sent during the handshake to clear any pending state.

### Motor controller initialisation delay

After a full power cycle, the MCU responds to the handshake (CMD01) quickly but the motor controller takes 1–2 additional seconds to initialise. Sending a feed command before the motor controller is ready results in a silent ignore. The firmware tracks `mcu_ready` separately from `mcu_connected` and waits for DP4=0x00 in the startup dump before allowing feed commands.

### Physical button lock behaviour

The physical lock button only disables the buttons on the feeder body. UART feed commands are accepted regardless of lock state — the lock does not block remote feeding.

### MCU UART state degradation after extended uptime

After extended uptime (observed at 1–3 days) the MCU enters a state where it receives UART feed commands but does not respond to them — no DP3 echo, no DP4 motor state change, no CMD34. The feed times out after 30 seconds. A plain handshake retry without toggling the EN pin does not recover the MCU from this state.

The firmware handles this automatically via the feed retry mechanism: on timeout, the EN pin is cycled LOW → 2s → HIGH, which forces the MCU to re-initialise its UART communication layer. The feed is then retried with the same portion count. See [Feed Retry and MCU Reset](#feed-retry-and-mcu-reset).

### CMD 0x1C payload length — stream corruption bug (fixed)

The MCU sends a time sync request every second. The correct response payload is **8 bytes**, as confirmed by WBR3 logic analyser captures. An earlier version of this firmware declared `len=8` in the frame header but only wrote 6 bytes of payload, producing a 13-byte frame where the MCU expected 15. On every time sync response the MCU consumed 2 bytes from the next UART frame as part of the CMD 0x1C payload, continuously desynchronising the stream.

Because CMD 0x1C fires every second this corruption was permanent and affected every frame that followed — including the CMD01 handshake response, DP status reports, and CMD34 feeding logs. Symptoms were startup failures (handshake never completing, `mcu_ready` never set) and remote feed commands silently failing.

The fix sends all 8 payload bytes: `[4B UTC unix timestamp] [local hour] [local minute] [UTC offset hours] [UTC offset minutes]`, matching the WBR3 exactly.

---

## Debugging

### Last error topic

Whenever the feeder status goes to `Error`, the reason and a local timestamp are published to:
```
wifi2mqtt/pet-feeder/last_error
```

The message is **retained** — the broker holds the last value indefinitely. You can query it any time, including after an ESP reboot, to see what went wrong most recently.

| Message | Cause |
|---------|-------|
| `[HH:MM:SS] Feed blocked — Xs cooldown remaining` | A feed was triggered within 30 seconds of the previous one |
| `[HH:MM:SS] MCU not ready — motor not initialised after 15s` | Motor controller never signalled ready (DP4=0) after startup |
| `[HH:MM:SS] MCU not ready — motor not initialised after retry` | Second wait for mcu_ready also timed out |
| `[HH:MM:SS] Feed timeout — retrying (N left)` | MCU did not respond — EN pin reset in progress, N more attempts remaining |
| `[HH:MM:SS] Feed failed — all retries exhausted` | MCU failed to respond after all retry attempts |

If SNTP hasn't synced yet when the error occurs, the timestamp prefix is omitted and the message is written without brackets.

### MQTT log mirror

ESPHome's `logger` component can mirror the serial log to an MQTT topic automatically — no custom code required. Add `mqtt_topic:` to the `logger:` block:

```yaml
logger:
  level: DEBUG
  mqtt_topic: wifi2mqtt/pet-feeder/debug
```

Whatever passes the configured `level` filter appears on both serial and MQTT. Remove `mqtt_topic:` (or set it to an empty string) to disable. See the [ESPHome logger docs](https://esphome.io/components/logger/) for details.

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

When the feeder status shows `Error`, check `wifi2mqtt/pet-feeder/last_error` first — it contains the timestamped reason for the most recent error, retained at the broker.

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
