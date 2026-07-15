# JBD BLE → Victron Bridge 🚲⚡

[![CI](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-ESP32-FF6600)](https://platformio.org/)
[![VenusOS](https://img.shields.io/badge/VenusOS-Cerbo%20GX-0047AB)](https://www.victronenergy.com/)
[![JBD](https://img.shields.io/badge/BMS-JBD%20%7C%20Jiabaida-008000)](#)

**[Deutsch (Schweizer Hochdeutsch)](README.de.md)**

Display your **JBD/Jiabaida/Xiaoxiang BMS** battery data in **Victron VenusOS** via Bluetooth Low Energy — no expensive SmartShunt required.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌──────────────┐   BLE    ┌──────────────────────┐   MQTT   ┌────────────┐│
│  │  JBD BMS     │◄────────►│  ESP32                │─────────►│ Cerbo GX   ││
│  │  (battery)   │          │  BLE → MQTT Bridge    │          │ (VenusOS)  ││
│  │  ~JBD-xxxx   │          │                      │          │            ││
│  └──────────────┘          └──────────────────────┘          │ ┌────────┐ ││
│                                                              │ │ DBus   │─►│─► D-Bus
│                                                              │ │ Driver │  ││
│                                                              │ └────────┘ ││
│                                                              │ ┌────────┐ ││
│                                                              │ │Node-RED│─►│─► Dashboard
│                                                              │ │(Debug)  │ ││
│                                                              │ └────────┘ ││
│                                                              └────────────┘│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Data flow:**
1. **ESP32** reads the JBD BMS via BLE (service 0xFF00, protocol reverse-engineered from [aiobmsble](https://github.com/patman15/aiobmsble))
2. **ESP32** publishes the values as JSON via **MQTT**
3. **Cerbo GX** runs a Python daemon that subscribes to MQTT and publishes to Victron's **D-Bus**
4. **Victron VenusOS** shows the battery as a native device — voltage, SoC, current, temperature included
5. **Node-RED** (optional) for dashboards, debugging, InfluxDB logging

## Features

- ✅ **JBD BLE protocol** — voltage, current, SoC, temperature, cell voltages
- ✅ **ESP32 with NimBLE** — stable and memory-efficient, better than stock BLE
- ✅ **VenusOS D-Bus** — battery appears like any Victron battery
- ✅ **MQTT** — flexible integration, no proprietary protocols
- ✅ **Node-RED flow** — debug and dashboard included
- ✅ **Watchdog** — detects BLE connection loss, sets "offline" state
- ✅ **CI** — GitHub Actions checks syntax and structure

## Quick Start

### 1. Flash the ESP32

```bash
cd esp32_jbd_ble_mqtt/

# Configure WiFi + MQTT
sed -i '' 's/WIFI_SSID.*/WIFI_SSID = "YourWiFi"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/WIFI_PASS.*/WIFI_PASS = "YourPassword"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/MQTT_HOST.*/MQTT_HOST = "192.168.1.100"/' esp32_jbd_ble_mqtt.ino

# Build + Upload (PlatformIO)
pio run -t upload
```

> **No PlatformIO?** Open in Arduino IDE → open `.ino` → install libraries (NimBLE, PubSubClient, ArduinoJson) → Upload.

### 2. Install the VenusOS Daemon

```bash
ssh root@<cerbo-ip>
cd /tmp
curl -sL https://github.com/silly82/jbd-ble-victron-bridge/archive/main.tar.gz | tar xz
cd jbd-ble-victron-bridge-main/venusos
./install.sh
```

**Or manually:**
```bash
scp -r venusos/* root@<cerbo-ip>:/data/dbus-mqtt-battery/
ssh root@<cerbo-ip>
/data/dbus-mqtt-battery/install.sh
```

### 3. Import Node-RED Flow (optional)

1. On Cerbo GX: VenusOS App Store → install **Node-RED**
2. Import `flows.json` → Menu → Import
3. Adjust MQTT broker config
4. Deploy

## Configuration

### ESP32 (`esp32_jbd_ble_mqtt.ino`)

```cpp
const char* WIFI_SSID     = "FRITZ!Box 7530";
const char* WIFI_PASS     = "secret";
const char* MQTT_HOST     = "192.168.1.100";     // Cerbo GX or broker
const int   MQTT_PORT     = 1883;
const char* JBD_DEVICE_NAME = "";                 // empty = auto-scan
```

> Leave `JBD_DEVICE_NAME` empty for auto-scan. The ESP will find devices named `JBD-*`, `DWF*`, `SX1*`, `SBL*`, etc.

### VenusOS Daemon (`venusos/dbus_mqtt_battery.service`)

```ini
Environment=MQTT_HOST=192.168.1.100
Environment=MQTT_TOPIC=bms/jbd/data
Environment=DBUS_INSTANCE=256
Environment=POLL_TIMEOUT=60
```

## MQTT Data Format

Topic: `bms/jbd/data`

```json
{
  "voltage": 13.25,
  "current": 5.02,
  "power": 66.5,
  "soc": 78,
  "charge": 98.5,
  "capacity": 120,
  "cycles": 42,
  "temp_count": 2,
  "temperatures": [22.5, 23.1],
  "cells": 4,
  "cell_voltages": [3.312, 3.315, 3.308, 3.310],
  "charging": true,
  "chrg_mosfet": true,
  "dischrg_mosfet": true
}
```

## VenusOS / D-Bus Paths

| D-Bus Path | Description | Unit |
|-----------|-------------|------|
| `/Dc/0/Voltage` | Battery voltage | V |
| `/Dc/0/Current` | Current (positive = charging) | A |
| `/Dc/0/Power` | Power | W |
| `/Dc/0/Temperature` | Temperature | °C |
| `/Soc` | State of charge | % |
| `/Capacity` | Rated capacity | Ah |
| `/ConsumedAmphours` | Consumed amphours | Ah |
| `/History/DischargeCycles` | Charge cycles | # |
| `/Info/MaxChargeCurrent` | Max charge current | A |
| `/Info/MaxDischargeCurrent` | Max discharge current | A |
| `/Connected` | Connection state | 0/1 |

## JBD BLE Protocol (Reverse Engineered)

Based on [aiobmsble](https://github.com/patman15/aiobmsble) by patman15.

| Parameter | Value |
|-----------|-------|
| Service UUID | `0000ff00-0000-1000-8000-00805f9b34fb` |
| RX Char (Notify) | `ff01` |
| TX Char (Write) | `ff02` |
| Init packet | `FF AA 15 01 <checksum>` |
| Command BasicInfo | `DD A5 03 00 <CRC16> 77` |
| Command Cells | `DD A5 04 00 <CRC16> 77` |
| CRC | `0x10000 − sum(payload)` |
| Frame tail | `0x77` |

## Directory Structure

```
jbd-ble-victron-bridge/
├── esp32_jbd_ble_mqtt/       ← ESP32 firmware (PlatformIO / Arduino)
│   ├── esp32_jbd_ble_mqtt.ino    ← Main sketch
│   └── platformio.ini             ← PlatformIO project
├── venusos/                   ← Cerbo GX / VenusOS
│   ├── dbus_mqtt_battery.py      ← D-Bus daemon (Python)
│   ├── dbus_mqtt_battery.service ← systemd service unit
│   └── install.sh                ← Installation script
├── flows.json                 ← Node-RED flow (importable)
├── .github/workflows/         ← GitHub Actions CI
├── LICENSE                    ← MIT
├── README.md                  ← This file (English)
└── README.de.md               ← Deutsch (Schweizer Hochdeutsch)
```

## Troubleshooting

**ESP32 can't find the JBD BMS:**
- Close the BMS app on your phone (only one connection at a time)
- Leave `JBD_DEVICE_NAME` empty (auto-scan)
- Check RSSI: place ESP32 close to the battery
- Some clones use different service UUIDs — check if `ff00` is right

**No data on Cerbo:**
```bash
# Check MQTT
mosquitto_sub -h localhost -t bms/jbd/data

# Service status
systemctl status dbus-mqtt-battery

# Log
journalctl -u dbus-mqtt-battery -n 50 --no-pager
```

**Battery not showing in VenusOS:**
- Restart service: `systemctl restart dbus-mqtt-battery`
- Change instance ID if taken: `DBUS_INSTANCE=257`
- VenusOS Remote Console → Devices → Scan for new devices

## Why Not a Pure Node-RED Solution?

[`@victronenergy/node-red-contrib-victron`](https://flows.nodered.org/node/@victronenergy/node-red-contrib-victron) provides official Victron nodes for Node-RED — but they can only **read from or write to existing D-Bus services**. There is no node that can **register a new virtual battery device** on D-Bus.

| Task | Node-RED alone |
|------|---------------|
| Subscribe to MQTT from ESP32 | ✅ Yes |
| Dashboard / visualisation | ✅ Yes |
| InfluxDB / Grafana logging | ✅ Yes |
| Alerts and automations | ✅ Yes |
| Write to *existing* VenusOS services | ✅ Yes (via `victron-output` nodes) |
| **Create a new virtual battery in VenusOS** | ❌ No — requires Python + D-Bus |

The Python daemon (`dbus_mqtt_battery.py`) is the only way to register a new `com.victronenergy.battery.*` service so VenusOS recognises the JBD BMS as a native battery. Node-RED remains an optional addition for dashboards and logging, but it cannot replace the daemon.

## Related Projects

- [aiobmsble](https://github.com/patman15/aiobmsble) — Python BLE BMS library (our foundation)
- [BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA) — Home Assistant integration
- [dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — VenusOS serial battery driver
- [velib_python](https://github.com/victronenergy/velib_python) — Victron D-Bus Python bindings

## License

MIT — do what you want, no warranty. See [LICENSE](LICENSE).