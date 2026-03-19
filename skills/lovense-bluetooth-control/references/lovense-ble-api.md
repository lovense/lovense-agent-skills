# Lovense BLE API Reference

Complete `LovenseDevice` class for Web Bluetooth communication with Lovense devices.

## Table of Contents

- [LovenseDevice Class](#lovensedevice-class)
- [Service Discovery](#service-discovery)
- [Command Encoding](#command-encoding)
- [Auto-Reconnect](#auto-reconnect)
- [Error Handling](#error-handling)

## LovenseDevice Class

```javascript
class LovenseDevice {
  // BLE Service/Characteristic UUIDs
  // Legacy service (older firmware)
  static LEGACY_SERVICE = {
    service: "0000fff0-0000-1000-8000-00805f9b34fb",
    tx: "0000fff2-0000-1000-8000-00805f9b34fb",
    rx: "0000fff1-0000-1000-8000-00805f9b34fb",
  };

  // Nordic UART Service (some newer devices)
  static NUS_SERVICE = {
    service: "6e400001-b5a3-f393-e0a9-e50e24dcca9e",
    tx: "6e400002-b5a3-f393-e0a9-e50e24dcca9e",
    rx: "6e400003-b5a3-f393-e0a9-e50e24dcca9e",
  };

  // Device-type services: each Lovense device type (A–Z) maps to
  //   {hex(letter)}300001-0023-4bd4-bbd5-a6920e4c5653
  // e.g. 'P' (Edge) → 0x50 → 50300001-…, 'E' (Edge 2) → 0x45 → 45300001-…
  static DEVICE_TYPE_SERVICES = Array.from({ length: 26 }, (_, i) => {
    const h = (0x41 + i).toString(16);
    const S = "-0023-4bd4-bbd5-a6920e4c5653";
    return {
      service: `${h}300001${S}`,
      tx: `${h}300002${S}`,
      rx: `${h}300003${S}`,
    };
  });

  static SERVICES = [
    LovenseDevice.LEGACY_SERVICE,
    ...LovenseDevice.DEVICE_TYPE_SERVICES,
    LovenseDevice.NUS_SERVICE,
  ];

  constructor() {
    this.device = null;
    this.server = null;
    this.txChar = null;
    this.rxChar = null;
    this.connected = false;
    this.batteryLevel = null;
    this._listeners = new Map();
    this._reconnecting = false;
    // Command queue state — enforces >= 100 ms between writes
    this._cmdQueue = [];
    this._cmdBusy = false;
    this._lastSendTime = 0;
    this._minInterval = 100; // ms
  }

  // ── Connection ──────────────────────────────────────────

  async scan() {
    this.device = await navigator.bluetooth.requestDevice({
      filters: [{ namePrefix: "LVS-" }],
      optionalServices: LovenseDevice.SERVICES.map((s) => s.service),
    });
    this.device.addEventListener("gattserverdisconnected", () =>
      this._onDisconnect(),
    );
    return this.device;
  }

  async connect() {
    if (!this.device) throw new Error("Call scan() first");
    this.server = await this.device.gatt.connect();

    // Try each service variant until one works
    for (const { service, tx, rx } of LovenseDevice.SERVICES) {
      try {
        const svc = await this.server.getPrimaryService(service);
        this.txChar = await svc.getCharacteristic(tx);
        this.rxChar = await svc.getCharacteristic(rx);
        break;
      } catch {
        /* try next variant */
      }
    }

    if (!this.txChar) throw new Error("No compatible Lovense service found");

    // Subscribe to notifications
    this.rxChar.addEventListener("characteristicvaluechanged", (e) =>
      this._onData(e),
    );
    await this.rxChar.startNotifications();

    this.connected = true;
    this._emit("connected", { name: this.device.name });

    // Query device info
    await this.getBattery();

    return this;
  }

  async disconnect() {
    if (!this.connected) return;
    try {
      await this.vibrate(0);
    } catch {}
    this.device?.gatt?.disconnect();
    this.connected = false;
    this._emit("disconnected");
  }

  // ── Motor Commands ──────────────────────────────────────

  async vibrate(level = 0) {
    this._validateLevel(level, 0, 20);
    return this._send(`Vibrate:${level};`);
  }

  async stopAll() {
    await this.vibrate(0);
  }

  // ── Device Info ─────────────────────────────────────────



  async getBattery() {
    const resp = await this._sendAndWait("Battery;");
    this.batteryLevel = parseInt(resp, 10);
    this._emit("battery", this.batteryLevel);
    return this.batteryLevel;
  }

  async powerOff() {
    return this._send("PowerOff;");
  }

  // ── Pattern Playback ───────────────────────────────────

  async playPattern(pattern, intervalMs = 150) {
    if (intervalMs < 100)
      throw new RangeError("intervalMs must be >= 100 to avoid lost commands");
    for (const level of pattern) {
      if (!this.connected) break;
      await this.vibrate(level);
      await this._delay(intervalMs);
    }
    await this.vibrate(0);
  }



  // ── Event System ────────────────────────────────────────

  on(event, callback) {
    if (!this._listeners.has(event)) this._listeners.set(event, []);
    this._listeners.get(event).push(callback);
    return this;
  }

  off(event, callback) {
    const list = this._listeners.get(event);
    if (list)
      this._listeners.set(
        event,
        list.filter((cb) => cb !== callback),
      );
    return this;
  }

  // ── Private ─────────────────────────────────────────────

  async _send(cmd) {
    if (!this.txChar) throw new Error("Not connected");
    return new Promise((resolve, reject) => {
      this._cmdQueue.push({ cmd, resolve, reject });
      this._flushQueue();
    });
  }

  async _flushQueue() {
    if (this._cmdBusy || this._cmdQueue.length === 0) return;
    this._cmdBusy = true;
    while (this._cmdQueue.length > 0) {
      const elapsed = Date.now() - this._lastSendTime;
      if (elapsed < this._minInterval) {
        await this._delay(this._minInterval - elapsed);
      }
      const { cmd, resolve, reject } = this._cmdQueue.shift();
      try {
        const data = new TextEncoder().encode(cmd);
        await this.txChar.writeValue(data);
        this._lastSendTime = Date.now();
        resolve();
      } catch (e) {
        reject(e);
      }
    }
    this._cmdBusy = false;
  }

  async _sendAndWait(cmd, timeoutMs = 3000) {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(
        () => reject(new Error(`Timeout waiting for response to: ${cmd}`)),
        timeoutMs,
      );
      const handler = (e) => {
        clearTimeout(timer);
        this.rxChar.removeEventListener("characteristicvaluechanged", handler);
        resolve(new TextDecoder().decode(e.target.value));
      };
      this.rxChar.addEventListener("characteristicvaluechanged", handler);
      this._send(cmd).catch(reject);
    });
  }

  _onData(event) {
    const value = new TextDecoder().decode(event.target.value);
    this._emit("data", value);
  }

  _onDisconnect() {
    this.connected = false;
    this._emit("disconnected");
    if (!this._reconnecting) this._tryReconnect();
  }

  async _tryReconnect(maxRetries = 3, delayMs = 2000) {
    this._reconnecting = true;
    for (let i = 0; i < maxRetries; i++) {
      try {
        this._emit("reconnecting", { attempt: i + 1, max: maxRetries });
        await this._delay(delayMs);
        await this.connect();
        this._reconnecting = false;
        return;
      } catch {
        /* retry */
      }
    }
    this._reconnecting = false;
    this._emit("reconnectFailed");
  }

  _emit(event, data) {
    (this._listeners.get(event) || []).forEach((cb) => cb(data));
  }

  _validateLevel(level, min, max) {
    if (level < min || level > max)
      throw new RangeError(`Level must be ${min}–${max}, got ${level}`);
  }

  _delay(ms) {
    return new Promise((r) => setTimeout(r, ms));
  }
}
```

## Command Timing

Lovense devices require a **minimum 100 ms gap** between consecutive BLE write operations. Commands sent faster than this are silently dropped by the device firmware — no error is returned.

The `LovenseDevice` class enforces this automatically via an internal serial command queue (`_cmdQueue`). Every call to `_send()` is enqueued and executed sequentially with at least 100 ms between writes.

**For UI sliders or other high-frequency inputs**, throttle at the call site to avoid flooding the queue:

```javascript
function throttle(fn, ms = 100) {
  let last = 0,
    timer = null;
  return (...args) => {
    const now = Date.now();
    clearTimeout(timer);
    if (now - last >= ms) {
      last = now;
      fn(...args);
    } else {
      timer = setTimeout(
        () => {
          last = Date.now();
          fn(...args);
        },
        ms - (now - last),
      );
    }
  };
}

const device = new LovenseDevice();
const setVibration = throttle((level) => device.vibrate(level), 100);
slider.addEventListener("input", () => setVibration(slider.value));
```

## Service Discovery

The scan/connect flow tries each service variant in order:

1. Request the BLE device with `namePrefix: 'LVS-'`
2. Declare **all** possible service UUIDs in `optionalServices` — this is mandatory because Web Bluetooth only allows access to services declared at scan time
3. Connect to GATT server
4. Try legacy service (`0000fff0-…`), then each device-type service (`{hex}300001-…` for A–Z), then Nordic UART (`6e400001-…`)
5. Acquire TX (write) and RX (notify) characteristics
6. Subscribe to RX notifications for responses

> **Important:** `getPrimaryServices()` can only enumerate services listed in `optionalServices` during `requestDevice()`. If a device's service UUID is missing from that list, it will be invisible — the fallback enumeration will return zero services.

## Command Encoding

All commands are plain ASCII/UTF-8 strings ending with `;`. Encode with `TextEncoder` before writing:

```javascript
const data = new TextEncoder().encode("Vibrate:10;");
await txCharacteristic.writeValue(data);
```

Response data from the RX characteristic is also UTF-8 text, decoded with `TextDecoder`.



## Error Handling

Common BLE errors and mitigations:

| Error               | Cause                                              | Fix                                                 |
| ------------------- | -------------------------------------------------- | --------------------------------------------------- |
| `NotFoundError`     | User cancelled device picker or no device in range | Ensure device is powered on and user selects it     |
| `NetworkError`      | GATT connection failed                             | Retry; move closer to device                        |
| `SecurityError`     | Not a secure context                               | Serve page over HTTPS or localhost                  |
| `NotAllowedError`   | No user gesture                                    | Call `requestDevice()` inside a click/touch handler |
| `InvalidStateError` | Write before connection ready                      | Await `connect()` before sending commands           |

Always wrap BLE operations in try/catch and provide user-friendly error messages.
