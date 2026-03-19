---
name: lovense-bluetooth-control
description: >
  Generate Web Bluetooth API code to discover, connect, and control Lovense devices via BLE.
  Use when asked to: (1) write code to control Lovense toys via Bluetooth, (2) build a web page
  or app that connects to Lovense devices, (3) send vibration/rotation/air-pump commands over BLE,
  (4) create a Lovense remote controller UI, (5) work with Lovense Bluetooth protocol or characteristics.
  Triggers on keywords: lovense, bluetooth toy, BLE vibrator, web bluetooth lovense.
---

# Lovense Bluetooth Control

Generate browser-based code that uses the Web Bluetooth API to scan, connect, and control Lovense BLE devices.

## Quick Start

Minimal example — connect and vibrate:

```javascript
const device = await navigator.bluetooth.requestDevice({
  filters: [{ namePrefix: 'LVS-' }],
  optionalServices: ['0000fff0-0000-1000-8000-00805f9b34fb']
});
const server = await device.gatt.connect();
const service = await server.getPrimaryService('0000fff0-0000-1000-8000-00805f9b34fb');
const tx = await service.getCharacteristic('0000fff2-0000-1000-8000-00805f9b34fb');
await tx.writeValue(new TextEncoder().encode('Vibrate:10;'));
```

## BLE Service & Characteristic UUIDs

Lovense devices expose two known BLE service variants. Try the legacy service first; fall back to the new one if the device does not advertise it.

| Variant | Service UUID | TX (Write) | RX (Notify) |
|---------|-------------|------------|-------------|
| Legacy  | `0000fff0-0000-1000-8000-00805f9b34fb` | `0000fff2-…` | `0000fff1-…` |
| New     | `50300001-0023-4bd4-bbd5-a6920e4c5653` | `50300002-…` | `50300003-…` |

When calling `requestDevice`, include **both** UUIDs in `optionalServices` so either variant resolves:

```javascript
optionalServices: [
  '0000fff0-0000-1000-8000-00805f9b34fb',
  '50300001-0023-4bd4-bbd5-a6920e4c5653'
]
```

## Command Reference

All commands are UTF-8 text strings terminated with `;`. Write them to the **TX** characteristic.

### Motor Control

| Command | Description | Range |
|---------|-------------|-------|
| `Vibrate:{level};` | Set vibration intensity | 0–20 (0 = stop) |

### Device Info & Utility

| Command | Response Format | Description |
|---------|----------------|-------------|
| `Battery;` | `{level}` (0–100) | Query battery percentage |
| `PowerOff;` | — | Power off the device |

### Response Handling

Subscribe to **RX** characteristic notifications to receive responses:

```javascript
const rx = await service.getCharacteristic('0000fff1-0000-1000-8000-00805f9b34fb');
rx.addEventListener('characteristicvaluechanged', e => {
  const msg = new TextDecoder().decode(e.target.value);
  console.log('Device response:', msg);
});
await rx.startNotifications();
```

## Implementation Patterns

### 1. Connection Manager Class

For any non-trivial project, wrap BLE logic in a reusable class. See [references/lovense-ble-api.md](references/lovense-ble-api.md) for the full `LovenseDevice` class implementation with auto-reconnect and error handling.

### 2. Building a Controller UI

When generating a controller web page:

1. Copy the template from [assets/lovense-controller.html](assets/lovense-controller.html)
2. Customize styling, add/remove controls as needed
3. The template includes: scan button, connect/disconnect, vibration slider, battery display, and status indicator

### 3. Pattern Playback

To play vibration patterns over time (**`intervalMs` must be ≥ 100 ms** to avoid lost commands):

```javascript
async function playPattern(device, pattern, intervalMs = 150) {
  if (intervalMs < 100) throw new RangeError('intervalMs must be >= 100 to avoid lost commands');
  for (const level of pattern) {
    await device.vibrate(level);
    await new Promise(r => setTimeout(r, intervalMs));
  }
  await device.vibrate(0);
}
// Example: ramp up then down (150 ms per step)
playPattern(device, [0,4,8,12,16,20,16,12,8,4,0], 150);
```

### 4. Multi-device Control

Web Bluetooth allows multiple concurrent connections. Call `requestDevice()` for each device, store references in a `Map<string, LovenseDevice>`, and issue commands independently.

## Command Timing & Reliability

### Minimum 100 ms Interval

Lovense devices require a **minimum 100 ms gap** between consecutive BLE write operations. Sending commands faster than this causes the device's BLE stack to drop commands silently — no error is returned but the command is never executed.

```
❌ Bad:  cmd1 ──(20ms)── cmd2 ──(50ms)── cmd3   → cmd2 & cmd3 may be lost
✅ Good: cmd1 ──(100ms)── cmd2 ──(100ms)── cmd3  → all delivered reliably
```

### Avoiding Lost Commands

Use a **serial command queue** that enforces the minimum interval and guarantees ordering:

```javascript
class CommandQueue {
  constructor(txChar, minIntervalMs = 100) {
    this.txChar = txChar;
    this.minInterval = minIntervalMs;
    this.queue = [];
    this.busy = false;
    this.lastSendTime = 0;
  }

  enqueue(cmd) {
    return new Promise((resolve, reject) => {
      this.queue.push({ cmd, resolve, reject });
      this._flush();
    });
  }

  async _flush() {
    if (this.busy || this.queue.length === 0) return;
    this.busy = true;

    while (this.queue.length > 0) {
      const elapsed = Date.now() - this.lastSendTime;
      if (elapsed < this.minInterval) {
        await new Promise(r => setTimeout(r, this.minInterval - elapsed));
      }
      const { cmd, resolve, reject } = this.queue.shift();
      try {
        await this.txChar.writeValue(new TextEncoder().encode(cmd));
        this.lastSendTime = Date.now();
        resolve();
      } catch (e) {
        reject(e);
      }
    }
    this.busy = false;
  }
}
```

Key techniques:

| Technique | Description |
|-----------|-------------|
| **Serial queue** | Process one command at a time; never send in parallel |
| **Timestamps** | Track `lastSendTime` and wait until 100 ms have passed |
| **Slider throttling** | Throttle UI slider `input` events so only the latest value is sent per 100 ms window |
| **Deduplication** | For high-frequency control (sliders), replace queued same-type commands with the latest value instead of appending |
| **Await write** | Always `await txChar.writeValue(...)` before sending the next command |

### Slider Throttle Example

For real-time sliders, throttle to respect the 100 ms floor:

```javascript
function throttle(fn, ms = 100) {
  let last = 0, timer = null;
  return (...args) => {
    const now = Date.now();
    clearTimeout(timer);
    if (now - last >= ms) {
      last = now;
      fn(...args);
    } else {
      timer = setTimeout(() => { last = Date.now(); fn(...args); }, ms - (now - last));
    }
  };
}

const sendVibrate = throttle(level => cmdQueue.enqueue(`Vibrate:${level};`), 100);
slider.addEventListener('input', () => sendVibrate(slider.value));
```

## Important Notes

- **HTTPS required** — Web Bluetooth only works on secure origins (`https://` or `localhost`).
- **User gesture required** — `requestDevice()` must be called from a user interaction handler (click, touch, etc.).
- **Browser support** — Chrome (desktop/Android), Edge, Opera. Not supported in Firefox or Safari.
- **One device per `requestDevice` call** — to connect multiple devices, prompt the user multiple times.
- **BLE range** — typical range is ~10 m; walls and interference reduce this.
- **Always stop motors on disconnect** — send `Vibrate:0;` before `device.gatt.disconnect()`.
- **Command interval ≥ 100 ms** — never send commands faster than 100 ms apart; use a command queue to enforce this.

## Resources

- [references/lovense-ble-api.md](references/lovense-ble-api.md) — Full `LovenseDevice` class with reconnect logic and all commands
- [assets/lovense-controller.html](assets/lovense-controller.html) — Ready-to-use single-file HTML controller page
