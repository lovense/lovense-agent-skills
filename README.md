# Lovense Agent Skills

Agent skills for controlling Lovense devices via Web Bluetooth API.

## Installation

```bash
npx skills add lovense/lovense-agent-skills
```

## Available Skills

### lovense-bluetooth-control

Generate browser-based code that uses the Web Bluetooth API to scan, connect, and control Lovense BLE devices.

**What it enables your AI agent to do:**

- Write code to connect to Lovense toys via Bluetooth
- Build web-based controller UIs for Lovense devices
- Send vibration commands over BLE
- Handle device discovery, connection management, and auto-reconnect
- Implement pattern playback and multi-device control

**Includes:**

- `SKILL.md` — Core instructions, BLE UUIDs, command reference, and implementation patterns
- `references/lovense-ble-api.md` — Full `LovenseDevice` class with command queue and reconnect logic
- `assets/lovense-controller.html` — Ready-to-use single-file HTML controller page

## Requirements

- **HTTPS or localhost** — Web Bluetooth only works on secure origins
- **Supported browsers** — Chrome (desktop/Android), Edge, Opera

## License

MIT
