Here’s a clean, ready-to-drop **`logic-analyzer-capture.md`** you can add to your repo.

---

# Logic Analyzer Capture — Tuya UART (Pet Feeder)

This document describes how to capture and analyze the UART communication between the Tuya Wi-Fi module (WBR3) and the feeder MCU using a logic analyzer.

The goal is to **observe and reverse-engineer the protocol** by passively listening (“eavesdropping”) on the original hardware.

---

## Hardware Used

* Logic analyzer: NanoDLA (24 MHz class)
<img width="191" height="183" alt="image" src="https://github.com/user-attachments/assets/084a33cc-4253-4b1f-885b-014056e5d12c" />

* Probe wires: standard Dupont leads
<img width="292" height="215" alt="image" src="https://github.com/user-attachments/assets/1af730b3-114f-4cce-b84b-eba450fd2f78" />

* Software: PulseView https://sigrok.org/wiki/PulseView
<img width="394" height="219" alt="image" src="https://github.com/user-attachments/assets/ae0d3c1e-9ea2-46e9-8066-da73e1c5f264" />

Typical inexpensive hardware works fine — UART at 9600 baud is very forgiving.

---

## Wiring Setup

You connect the logic analyzer **in parallel** with the original Wi-Fi module (do NOT remove it).

| Logic Analyzer Pin | Connect To      | Description              |
| ------------------ | --------------- | ------------------------ |
| GND                | Board GND       | Common ground (required) |
| D0                 | TX (WBR3 → MCU) | Data from Wi-Fi module   |
| D1                 | RX (MCU → WBR3) | Data from MCU            |
| D2                 | EN              | Module enable signal     |

**Important:**

* Do NOT inject voltage — this is passive listening only
* Ensure shared ground, otherwise signals will be garbage
* Signals are **3.3V TTL**

---

## What You’re Capturing

You are observing:

* Full **Tuya serial protocol**
* Boot handshake
* Feed commands
* Status updates (DP frames)
* Time sync (CMD 0x1C)

This is exactly how the protocol in `README.md` was derived.

---

## PulseView Setup

### 1. Install PulseView

* Official site: [https://sigrok.org/wiki/PulseView](https://sigrok.org/wiki/PulseView)
* Available on Linux, Windows, macOS

---

### 2. Device Configuration

In PulseView:

* Select your logic analyzer device
* Set sample rate to:

  * **≥ 1 MHz** (safe for 9600 baud)
* Enable channels:

  * D0
  * D1
  * D2 (optional but useful)

---

### 3. Add UART Decoders

Add two UART protocol decoders:

#### Channel 1 — ESP → MCU

* Input: **D0**
* Baud rate: **9600**
* Data bits: 8
* Parity: None
* Stop bits: 1

#### Channel 2 — MCU → ESP

* Input: **D1**
* Same settings

This gives you **bidirectional decode**.

---

### Helpful guide (decoder setup)

* [https://sigrok.org/wiki/Protocol_decoder:UART](https://sigrok.org/wiki/Protocol_decoder:UART)

---

## Capture Procedure

### 1. Start Capture

* Click **Run**
* Power on the feeder

---

### 2. Observe Boot Sequence

You should see:

```
55 AA 00 00 ...   (ESP → MCU heartbeat)
55 AA 03 00 ...   (MCU → ESP heartbeat)
55 AA 00 01 ...   (product query)
55 AA 03 01 ...   (product response)
```

This confirms correct wiring and decoding.

---

### 3. Trigger Events

Capture different behaviors:

#### Power-on

* Full handshake
* DP state dump

#### Feed via app

* CMD 0x06
* CMD 0x34
* DP4 transitions

#### Physical button

* No DP3 echo
* Lock state changes (DP25)

---

### 4. Stop and Analyze

* Stop capture
* Zoom into frames
* Inspect decoded UART bytes

---

## Interpreting Frames

All frames follow:

```
55 AA [VER] [CMD] [LEN_H] [LEN_L] [PAYLOAD...] [CHK]
```

Example:

```
55 AA 03 00 00 01 01 04
```

* `03` → MCU → ESP
* `00` → Heartbeat
* `01` → status

---

## Useful Tips

### 1. Label Channels

Rename in PulseView:

* D0 → `ESP_TX`
* D1 → `MCU_TX`
* D2 → `EN`

Saves your sanity later.

---

### 2. Use Hex View

Enable:

* “Hex dump” view
* Makes protocol patterns obvious

---

### 3. Capture Long Sessions

Some bugs only appear after:

* Long uptime
* Repeated feed cycles

---

### 4. Watch for Desync

Classic issue:

* Wrong payload length
* UART stream shifts

Symptom:

* Frames stop aligning on `55 AA`

(You actually hit this with CMD 0x1C earlier.)

---

### 5. EN Pin Insight

Capturing EN helps understand:

* Boot timing
* When MCU starts talking
* Differences between cold boot vs ESP reboot

---

## Common Mistakes

### No data visible

* Missing GND connection
* Wrong pins (TX/RX swapped)

### Garbage decode

* Wrong baud rate (must be 9600)
* Sample rate too low

### Only one direction visible

* Only connected one line
* Or wrong channel assignment

---

## Why This Matters

This capture method lets you:

* Reverse engineer closed protocols
* Verify firmware behavior against original hardware
* Debug subtle timing issues
* Confirm required quirks (like CMD34 ack)

Without this, you’re guessing.

With it, you’re just reading the truth off the wire.

---

## References

* PulseView documentation: [https://sigrok.org/wiki/PulseView](https://sigrok.org/wiki/PulseView)
* UART decoder docs: [https://sigrok.org/wiki/Protocol_decoder:UART](https://sigrok.org/wiki/Protocol_decoder:UART)
* Tuya protocol reverse engineering: derived from real captures (this project)

---

If you want, next step could be:

* a **PulseView .sr session template**
* or a **decoded example capture with annotations (gold for debugging)**
