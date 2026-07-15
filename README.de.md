# JBD BLE → Victron Bridge 🚲⚡

[![CI](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/Licence-MIT-yellow.svg)](LICENSE)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-ESP32-FF6600)](https://platformio.org/)
[![VenusOS](https://img.shields.io/badge/VenusOS-Cerbo%20GX-0047AB)](https://www.victronenergy.com/)
[![JBD](https://img.shields.io/badge/BMS-JBD%20%7C%20Jiabaida-008000)](#)

**[English version](README.md)**

Zeig dini **JBD/Jiabaida/Xiaoxiang BMS**-Akku-Date im **Victron VenusOS** a — ohni tüüre SmartShunt. Dä ESP32 liist dä Akku über Bluetooth uus und schickt d'Wärt per MQTT a Cerbo GX.

## Architektur

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌──────────────┐   BLE    ┌──────────────────────┐   MQTT   ┌────────────┐│
│  │  JBD BMS     │◄────────►│  ESP32                │─────────►│ Cerbo GX   ││
│  │  (Akku)      │          │  BLE → MQTT Bridge    │          │ (VenusOS)  ││
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

**Dä Datewäg:**
1. **ESP32** list dä JBD BMS über BLE uus (Service 0xFF00, Protokoll us [aiobmsble](https://github.com/patman15/aiobmsble) rückentwicklet)
2. **ESP32** schickt d'Wärt als JSON per **MQTT**
3. **Cerbo GX** lauft en Python-Daemon, wo MQTT abonniert und a Victrons **D-Bus** witergit
4. **Victron VenusOS** zeigt dä Akku wie en normale Batterii a — Spannig, SoC, Strom, Temperatur
5. **Node-RED** (optional) für Dashboard, Debugging, InfluxDB

## Features

- ✅ **JBD BLE-Protokoll** — Spannig, Strom, SoC, Temperatur, Zälle-Spannige
- ✅ **ESP32 mit NimBLE** — stabil und sparsam, besser als Standard-BLE
- ✅ **VenusOS D-Bus** — Akku wird wie jede Victron-Batterii azeigt
- ✅ **MQTT** — flexibel, kei proprietäri Protokoll
- ✅ **Node-RED Flow** — Debug und Dashboard integriert
- ✅ **Watchdog** — merks wenn d'BLE-Verbindig abbricht, setzt "offline"
- ✅ **CI** — GitHub Actions prüeft Syntax und Struktur

## Schnellstart

### 1. ESP32 flashe

```bash
cd esp32_jbd_ble_mqtt/

# WLAN + MQTT konfiguriere
sed -i '' 's/WIFI_SSID.*/WIFI_SSID = "DysWLAN"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/WIFI_PASS.*/WIFI_PASS = "DysPasswort"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/MQTT_HOST.*/MQTT_HOST = "192.168.1.100"/' esp32_jbd_ble_mqtt.ino

# Build + Upload (PlatformIO)
pio run -t upload
```

> **Kei PlatformIO?** Arduino IDE ufmache → `.ino` öffne → Libraries installiere (NimBLE, PubSubClient, ArduinoJson) → Upload.

### 2. VenusOS Daemon installiere

```bash
ssh root@<cerbo-ip>
cd /tmp
curl -sL https://github.com/silly82/jbd-ble-victron-bridge/archive/main.tar.gz | tar xz
cd jbd-ble-victron-bridge-main/venusos
./install.sh
```

**Oder vo Hand:**
```bash
scp -r venusos/* root@<cerbo-ip>:/data/dbus-mqtt-battery/
ssh root@<cerbo-ip>
/data/dbus-mqtt-battery/install.sh
```

### 3. Node-RED Flow importiere (optional)

1. Uf em Cerbo GX: VenusOS App Store → **Node-RED** installiere
2. `flows.json` importiere → Menü → Import
3. MQTT-Broker-Konfiguration aapasse
4. Deploy

## Konfiguration

### ESP32 (`esp32_jbd_ble_mqtt.ino`)

```cpp
const char* WIFI_SSID     = "FRITZ!Box 7530";
const char* WIFI_PASS     = "geheim";
const char* MQTT_HOST     = "192.168.1.100";     // Cerbo GX oder Broker
const int   MQTT_PORT     = 1883;
const char* JBD_DEVICE_NAME = "";                 // leer = Auto-Scan
```

> `JBD_DEVICE_NAME` leer la = ESP scannt automatisch nach Gäret mit Name wie `JBD-*`, `DWF*`, `SX1*`, `SBL*`.

### VenusOS Daemon (`venusos/dbus_mqtt_battery.service`)

```ini
Environment=MQTT_HOST=192.168.1.100
Environment=MQTT_TOPIC=bms/jbd/data
Environment=DBUS_INSTANCE=256
Environment=POLL_TIMEOUT=60
```

## MQTT-Dateformat

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

## VenusOS / D-Bus Pfad

| D-Bus Pfad | Beschribig | Einheit |
|-----------|-------------|--------|
| `/Dc/0/Voltage` | Batteriispannig | V |
| `/Dc/0/Current` | Strom (positiv = lade) | A |
| `/Dc/0/Power` | Leistig | W |
| `/Dc/0/Temperature` | Temperatur | °C |
| `/Soc` | Ladezuestand | % |
| `/Capacity` | Nennkapazität | Ah |
| `/ConsumedAmphours` | Verbruchti Amperesunde | Ah |
| `/History/DischargeCycles` | Ladezykle | # |
| `/Info/MaxChargeCurrent` | Max. Ladestrom | A |
| `/Info/MaxDischargeCurrent` | Max. Entladestrom | A |
| `/Connected` | Verbindigsstatus | 0/1 |

## JBD BLE-Protokoll (Rückentwicklet)

Basieret uf [aiobmsble](https://github.com/patman15/aiobmsble) vom patman15.

| Parameter | Wärt |
|-----------|------|
| Service UUID | `0000ff00-0000-1000-8000-00805f9b34fb` |
| RX Char (Notify) | `ff01` |
| TX Char (Write) | `ff02` |
| Init-Packet | `FF AA 15 01 <checksum>` |
| Command BasicInfo | `DD A5 03 00 <CRC16> 77` |
| Command Cells | `DD A5 04 00 <CRC16> 77` |
| CRC | `0x10000 − sum(payload)` |
| Frame Tail | `0x77` |

## Verzeichnisstruktur

```
jbd-ble-victron-bridge/
├── esp32_jbd_ble_mqtt/       ← ESP32 Firmware (PlatformIO / Arduino)
│   ├── esp32_jbd_ble_mqtt.ino    ← Hauptsketch
│   └── platformio.ini             ← PlatformIO Projekt
├── venusos/                   ← Cerbo GX / VenusOS
│   ├── dbus_mqtt_battery.py      ← D-Bus Daemon (Python)
│   ├── dbus_mqtt_battery.service ← systemd Service Unit
│   └── install.sh                ← Installationsscript
├── flows.json                 ← Node-RED Flow (importierbar)
├── .github/workflows/         ← GitHub Actions CI
├── LICENSE                    ← MIT
├── README.md                  ← Englisch
└── README.de.md               ← Deutsch (Schwiizer Hochdütsch)
## Fehlersuechi

**ESP32 findet kei JBD BMS:**
- BMS-App uf em Handy zumache (nume ei Verbindig gliichzitig)
- `JBD_DEVICE_NAME` leer la (Auto-Scan)
- RSSI prüefe — ESP32 nöch am Akku platziere
- Mangi Klone händ abwiichendi Service-UUIDs — prüef ob `ff00` stimmt

**Kei Date uf em Cerbo:**
```bash
# MQTT prüefe
mosquitto_sub -h localhost -t bms/jbd/data

# Service-Status
systemctl status dbus-mqtt-battery

# Log
journalctl -u dbus-mqtt-battery -n 50 --no-pager
```

**Akku tuucht nöd i VenusOS uf:**
- Service neu starte: `systemctl restart dbus-mqtt-battery`
- Instance-ID ändere falls belegt: `DBUS_INSTANCE=257`
- VenusOS Remote Console → Gerät → Nach neue Gerät sueche

## Verwändti Projekt

- [aiobmsble](https://github.com/patman15/aiobmsble) — Python BLE BMS Library (d'Basis)
- [BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA) — Home Assistant Integration
- [dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — VenusOS Serial Battery Driver
- [velib_python](https://github.com/victronenergy/velib_python) — Victron D-Bus Python Bindings

## Lizenz

MIT — mach demit was du wotsch, aber kei Garantie. Lueg [LICENSE](LICENSE).