# WalkPad Remote

A simple web-based BLE remote for walking pad treadmills. One page, no app install, no account required.

Built as a replacement for a broken physical remote. Opens in Chrome, connects via Bluetooth, and gives you Start/Stop/Speed controls with big touch-friendly buttons.

## Features

- **One-click connect** via Web Bluetooth (Chrome)
- **Preset speeds**: 1.0 / 1.5 / 2.0 / 2.5 / 3.0 km/h (auto-starts the treadmill)
- **Fine control**: +/- buttons with 0.1 km/h step
- **Live speed display** from treadmill data (BLE notify)
- **Auto-reconnect** on page reload (no re-pairing needed)
- **Dark theme**, large buttons (designed for use while walking)
- **Diagnostics panel**: GATT services, hex BLE log, connection state
- **Mobile-friendly**: works on phone screens too

## Tested On

- **Yesoul W2** walking pad (FTMS protocol with non-standard speed resolution)
- Chrome on macOS (MacBook Pro M4 Pro)

Should work with other FTMS-compatible treadmills. Speed calibration may differ per model (see [Speed Calibration](#speed-calibration) below).

## Quick Start

### Option 1: GitHub Pages (recommended)

Open **[https://pistoganza.github.io/walkpad-remote/](https://pistoganza.github.io/walkpad-remote/)** in Chrome.

### Option 2: Run locally

```bash
git clone https://github.com/pistoganza/walkpad-remote.git
cd walkpad-remote
python3 -m http.server 8092
# Open http://localhost:8092 in Chrome
```

## How to Connect

1. Turn on the treadmill (step on the belt or press the power button)
2. Wait 5 seconds for Bluetooth to initialize
3. Open this page in **Google Chrome**
4. Click **"Connect Treadmill"** and select your device from the popup
5. Done! Use preset buttons or Start/Stop to control the treadmill

> **Important:** Do NOT pair the treadmill through System Settings. Web Bluetooth handles the connection. If already paired, remove it from Bluetooth devices first.

## Browser Support

| Browser | Status |
|---------|--------|
| Chrome (macOS, Windows, Android, ChromeOS) | Supported |
| Edge, Opera | Supported |
| Safari (macOS, iOS) | Not supported |
| Firefox | Not supported |

Web Bluetooth requires HTTPS or localhost.

## Speed Calibration

The Yesoul W2 uses a non-standard speed resolution. Instead of the FTMS standard 0.01 km/h per BLE unit, it uses **0.006 km/h per unit** (a 0.6x factor).

If your treadmill shows different speeds than the app, you can adjust the `SPEED_RES` constant in `index.html`:

```javascript
// Standard FTMS: 0.01
// Yesoul W2: 0.006
const SPEED_RES = 0.006;
```

To find your treadmill's factor:
1. Connect and set speed to any value
2. Compare the app display with the treadmill display
3. Calculate: `SPEED_RES = treadmill_display / (ble_value)`
4. Open Diagnostics to see raw BLE values

## Protocol

Uses **FTMS** (Fitness Machine Service) — a Bluetooth SIG standard for fitness equipment.

| Characteristic | UUID | Purpose |
|---|---|---|
| Control Point | `0x2AD9` | Send commands (start, stop, set speed) |
| Treadmill Data | `0x2ACD` | Live speed, distance, time (notify) |
| Speed Range | `0x2AD4` | Min/max speed, step |
| Machine Status | `0x2ADA` | Running/stopped state (notify) |

If your treadmill doesn't support FTMS, the app will detect it and show raw BLE data in the Diagnostics panel for reverse engineering.

## License

MIT
