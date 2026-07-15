# JBD BLE → Victron Bridge 🚲⚡

[![CI](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/Licence-MIT-yellow.svg)](LICENSE)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-ESP32-FF6600)](https://platformio.org/)
[![VenusOS](https://img.shields.io/badge/VenusOS-Cerbo%20GX-0047AB)](https://www.victronenergy.com/)
[![JBD](https://img.shields.io/badge/BMS-JBD%20%7C%20Jiabaida-008000)](#)

**[English version](README.md)**

Zeigen Sie Ihren **JBD/Jiabaida/Xiaoxiang BMS**-Akku im **Victron VenusOS** an — ohne teuren SmartShunt. Der ESP32 liest den Akku via Bluetooth aus und sendet die Werte per MQTT an den Cerbo GX.

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

**Der Datenweg:**
1. **ESP32** liest den JBD BMS via BLE aus (Service 0xFF00, Protokoll aus [aiobmsble](https://github.com/patman15/aiobmsble) reverse-engineered)
2. **ESP32** publiziert die Werte als JSON via **MQTT**
3. **Cerbo GX** startet einen Python-Daemon, der MQTT abonniert und an Victrons **D-Bus** weiterleitet
4. **Victron VenusOS** zeigt den Akku als native Batterie an — Spannung, SoC, Strom, Temperatur inklusive
5. **Node-RED** (optional) für Dashboards, Debugging, InfluxDB

## Features

- ✅ **JBD BLE-Protokoll** — Spannung, Strom, SoC, Temperatur, Zellspannungen
- ✅ **ESP32 mit NimBLE** — stabil und speicherschonend, besser als Standard-BLE
- ✅ **VenusOS D-Bus** — Akku erscheint wie jede Victron-Batterie
- ✅ **MQTT** — flexible Anbindung, keine proprietären Protokolle
- ✅ **Node-RED Flow** — Debug und Dashboard inklusive
- ✅ **Watchdog** — erkennt BLE-Verbindungsabbruch, setzt "offline"
- ✅ **CI** — GitHub Actions prüft Syntax und Struktur

## Schnellstart

### 1. ESP32 flashen

```bash
cd esp32_jbd_ble_mqtt/

# WLAN + MQTT konfigurieren
sed -i '' 's/WIFI_SSID.*/WIFI_SSID = "IhrWLAN"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/WIFI_PASS.*/WIFI_PASS = "IhrPasswort"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/MQTT_HOST.*/MQTT_HOST = "192.168.1.100"/' esp32_jbd_ble_mqtt.ino

# Build + Upload (PlatformIO)
pio run -t upload
```

> **Kein PlatformIO?** Arduino IDE öffnen → `.ino` öffnen → Libraries installieren (NimBLE, PubSubClient, ArduinoJson) → Upload.

### 2. VenusOS Daemon installieren

```bash
ssh root@<cerbo-ip>
cd /tmp
curl -sL https://github.com/silly82/jbd-ble-victron-bridge/archive/main.tar.gz | tar xz
cd jbd-ble-victron-bridge-main/venusos
./install.sh
```

**Oder manuell:**
```bash
scp -r venusos/* root@<cerbo-ip>:/data/dbus-mqtt-battery/
ssh root@<cerbo-ip>
/data/dbus-mqtt-battery/install.sh
```

### 3. Node-RED Flow importieren (optional)

1. Auf dem Cerbo GX: VenusOS App Store → **Node-RED** installieren
2. `flows.json` importieren → Menü → Import
3. MQTT-Broker-Konfiguration anpassen
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

> `JBD_DEVICE_NAME` leer lassen = ESP scannt automatisch nach Geräten mit Namen wie `JBD-*`, `DWF*`, `SX1*`, `SBL*`.

### VenusOS Daemon (`venusos/dbus_mqtt_battery.service`)

```ini
Environment=MQTT_HOST=192.168.1.100
Environment=MQTT_TOPIC=bms/jbd/data
Environment=DBUS_INSTANCE=256
Environment=POLL_TIMEOUT=60
```

## MQTT-Datenformat

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

## VenusOS / D-Bus Pfade

| D-Bus Pfad | Beschreibung | Einheit |
|-----------|-------------|--------|
| `/Dc/0/Voltage` | Batteriespannung | V |
| `/Dc/0/Current` | Strom (positiv = laden) | A |
| `/Dc/0/Power` | Leistung | W |
| `/Dc/0/Temperature` | Temperatur | °C |
| `/Soc` | Ladezustand | % |
| `/Capacity` | Nennkapazität | Ah |
| `/ConsumedAmphours` | Verbrauchte Amperestunden | Ah |
| `/History/DischargeCycles` | Ladezyklen | # |
| `/Info/MaxChargeCurrent` | Max. Ladestrom | A |
| `/Info/MaxDischargeCurrent` | Max. Entladestrom | A |
| `/Connected` | Verbindungsstatus | 0/1 |

## JBD BLE-Protokoll (Reverse Engineering)

Basiert auf [aiobmsble](https://github.com/patman15/aiobmsble) von patman15.

| Parameter | Wert |
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
│   └── install.sh                ← Installationsskript
├── flows.json                 ← Node-RED Flow (importierbar)
├── .github/workflows/         ← GitHub Actions CI
├── LICENSE                    ← MIT
├── README.md                  ← English
└── README.de.md               ← Deutsch (Schweizer Standardsprache)
```

## Fehlersuche

**ESP32 findet kein JBD BMS:**
- BMS-App auf dem Handy schliessen (nur eine Verbindung gleichzeitig)
- `JBD_DEVICE_NAME` leer lassen (Auto-Scan)
- RSSI prüfen — ESP32 nahe am Akku platzieren
- Service UUID `ff00` prüfen (manchen Klone nutzen abweichende UUIDs)

**Keine Daten auf dem Cerbo:**
```bash
# MQTT prüfen
mosquitto_sub -h localhost -t bms/jbd/data

# Service-Status
systemctl status dbus-mqtt-battery

# Log
journalctl -u dbus-mqtt-battery -n 50 --no-pager
```

**Akku erscheint nicht in VenusOS:**
- Service neu starten: `systemctl restart dbus-mqtt-battery`
- Instance-ID ändern falls belegt: `DBUS_INSTANCE=257`
- VenusOS Remote Console → Geräte → Nach neuen Geräten suchen

## Verwandte Projekte

- [aiobmsble](https://github.com/patman15/aiobmsble) — Python BLE BMS Library (unsere Basis)
- [BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA) — Home Assistant Integration
- [dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — VenusOS Serial Battery Driver
- [velib_python](https://github.com/victronenergy/velib_python) — Victron D-Bus Python Bindings

## Lizenz

MIT — machen Sie damit, was Sie wollen, aber ohne Garantie. Siehe [LICENSE](LICENSE).