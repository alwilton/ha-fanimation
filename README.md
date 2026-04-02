# Fanimation BT 3-Speed Fan Integration for Home Assistant

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-41BDF5.svg)](https://github.com/hacs/integration)

Control your Fanimation 3-speed AC Bluetooth ceiling fan from Home Assistant.

## Features

- **Fan speed control**: Off, Low, Medium, High (preset modes + percentage slider)
- **Downlight brightness**: 0-100% dimming
- **BLE auto-discovery**: Automatically detects Fanimation fans advertising service UUID `0xE000`
- **Connect-per-command**: Connects briefly to send commands, then disconnects — won't lock out the fanSync phone app
- **Retry with verification**: Commands are verified against the fan's response and retried if needed

## Requirements

- Home Assistant 2024.1.0 or later
- A Bluetooth adapter accessible to Home Assistant (built-in, USB dongle, or ESPHome Bluetooth proxy)
- HA machine within BLE range (~10m) of the fan, or an ESP32 Bluetooth proxy near the fan (see below)

## Installation via HACS

1. Open HACS in your Home Assistant
2. Click the **three dots menu** (top right) → **Custom repositories**
3. Add repository URL: `https://github.com/alwilton/ha-fanimation`
4. Category: **Integration**
5. Click **Add**, then find "Fanimation BT 3-Speed Fan" in HACS and click **Download**
6. **Restart Home Assistant**

## Setup

After restart:

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **"Fanimation BT 3-Speed Fan"**
3. Either:
   - **Auto-discovered**: If HA detected the fan via BLE, confirm the setup
   - **Manual**: Enter the fan's BLE MAC address (e.g., `E0:E5:CF:28:81:30`)

## Entities Created

| Entity | Type | Controls |
|--------|------|----------|
| Fan | `fan` | On/Off, Speed (Low/Medium/High), Percentage slider |
| Light | `light` | On/Off, Brightness (0-100%) |

Both entities appear under a single device in HA.

## Protocol

This integration communicates with Fanimation fans using their BLE GATT protocol:

- **Service UUID**: `0000e000-0000-1000-8000-00805f9b34fb`
- **Write characteristic**: `0000e001-...` (commands to fan)
- **Notify characteristic**: `0000e002-...` (responses from fan)
- **Packet format**: 10 bytes `[0x53, CmdType, Speed, Dir, Uplight, Downlight, TimerLo, TimerHi, FanType, Checksum]`

Protocol was reverse-engineered from the fanSync Android APK.

## Supported Fans

Tested with AC Standard (3-speed) Fanimation fans with a downlight. Other Fanimation BLE fans using the same `0xE000` service should also work.

## Identifying Your Fan in HA Bluetooth

If you need to find your fan's MAC address, go to **Settings → System → Hardware → All Hardware → Bluetooth** in HA. Each discovered device shows its advertisement data as JSON. Look for these identifiers:

**Primary identifier — service UUID:**
```json
"service_uuids": ["0000e000-0000-1000-8000-00805f9b34fb"]
```
This is the definitive Fanimation BLE service UUID. If a device has this, it's your fan.

**Secondary clues:**

| Field | Expected value |
|-------|---------------|
| `name` | `"CeilingFan"` (common Fanimation device name) |
| `connectable` | `true` |
| `rssi` | −50 to −70 dBm if within 5–10m; weaker than −80 suggests out of range |

**Example of a matching entry:**
```json
{
  "name": "CeilingFan",
  "address": "XX:XX:XX:XX:XX:XX",
  "rssi": -65,
  "service_uuids": ["0000e000-0000-1000-8000-00805f9b34fb"],
  "connectable": true
}
```

> **Note:** The fan may not always include the service UUID in every advertisement broadcast. If it doesn't appear in the auto-discovery flow, use **manual setup** and enter the MAC address directly.

## Bluetooth Range and ESPHome Bluetooth Proxy

If your Home Assistant server is far from the fan (e.g., in a closet or another room), BLE connections may be unreliable or fail entirely. The recommended solution is an **ESPHome Bluetooth Proxy** running on an ESP32 device located near the fan.

### How It Works

The ESP32 acts as a BLE relay -- it connects to the fan over Bluetooth locally and forwards the traffic to Home Assistant over WiFi. This integration uses `bleak` for BLE communication, and Home Assistant routes these connections through the proxy transparently. No changes to this integration are needed.

### ESP32 Proxy Setup

If your ESP32 already runs ESPHome, add the following to its YAML configuration:

```yaml
bluetooth_proxy:
  active: true

esp32_ble_tracker:
```

`active: true` is required because this integration writes commands to the fan (not just passive scanning).

Then rebuild and flash the ESP32. Home Assistant will discover the proxy automatically and begin routing BLE connections through it.

### Coexistence with Other ESP32 Functions

The Bluetooth proxy runs alongside other ESPHome components (sensors, switches, relays, GPIO, etc.) without interference. If the ESP32 is already performing other tasks via ESPHome, adding the Bluetooth proxy will not disrupt them.

### Multiple Proxies

If you have several ESP32 devices running ESPHome Bluetooth proxies, Home Assistant will select the one with the best signal to the fan automatically.

### Dedicated Proxy

If your existing ESP32 devices do not run ESPHome, you can use a separate ESP32 board (~$5) as a dedicated Bluetooth proxy. Flash it with ESPHome using the config above plus your WiFi credentials:

```yaml
esphome:
  name: ble-proxy-office

esp32:
  board: esp32dev

wifi:
  ssid: "YourWiFi"
  password: "YourPassword"

api:
  encryption:
    key: "your-api-key"

bluetooth_proxy:
  active: true

esp32_ble_tracker:
```

## Troubleshooting

- **Fan not discovered**: Ensure the HA machine has Bluetooth and is within range. Check HA logs for BLE errors.
- **Commands fail intermittently**: BLE can be flaky. The integration retries commands automatically. Make sure no other device (phone app) is connected to the fan at the same time.
- **"Cannot connect" during setup**: Power-cycle the fan, then try again. The fan's BLE radio may need a reset.
