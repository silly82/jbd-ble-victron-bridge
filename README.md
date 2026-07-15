# JBD BLE → Victron Bridge 🚲⚡

[![CI](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml/badge.svg)](https://github.com/silly82/jbd-ble-victron-bridge/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-ESP32-FF6600)](https://platformio.org/)
[![VenusOS](https://img.shields.io/badge/VenusOS-Cerbo%20GX-0047AB)](https://www.victronenergy.com/de/products/cerbo-gx)
[![JBD](https://img.shields.io/badge/BMS-JBD%20%7C%20Jiabaida-008000)](#)

**JBD/Jiabaida/Xiaoxiang BMS per Bluetooth Low Energy im Victron VenusOS anzeigen** — ohne teuren SmartShunt.

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
1. **ESP32** liest JBD BMS per BLE aus (Service 0xFF00, Protokoll reverse-engineered aus [aiobmsble](https://github.com/patman15/aiobmsble))
2. **ESP32** publiziert die Werte als JSON per **MQTT**
3. **Cerbo GX** läuft ein Python-Daemon, der MQTT subscribed und an Victrons **D-Bus** published
4. **Victron VenusOS** zeigt den Akku als "normale Batterie" an — inkl. Spannung, SoC, Strom, Temperatur
5. **Node-RED** (optional) für Dashboard, Debug, InfluxDB

## Features

- ✅ **JBD BLE Protokoll** — Spannung, Strom, SoC, Temperatur, Zellspannungen
- ✅ **ESP32** mit NimBLE — stabiler und speicherschonender als Standard-BLE
- ✅ **VenusOS D-Bus** — batterie wird wie jede Victron-Batterie angezeigt
- ✅ **MQTT** — flexible Anbindung, keine proprietären Protokolle
- ✅ **Node-RED Flow** — Debug und Dashboard inklusive
- ✅ **Watchdog** — erkennt wenn BLE-Verbindung abbricht, setzt "offline"
- ✅ **CI** — GitHub Actions prüft Syntax und Struktur

## Schnellstart

### 1. ESP32 flashen

```bash
cd esp32_jbd_ble_mqtt/

# WLAN + MQTT konfigurieren
sed -i '' 's/WIFI_SSID.*/WIFI_SSID = "DeinWLAN"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/WIFI_PASS.*/WIFI_PASS = "DeinPasswort"/' esp32_jbd_ble_mqtt.ino
sed -i '' 's/MQTT_HOST.*/MQTT_HOST = "192.168.1.100"/' esp32_jbd_ble_mqtt.ino

# Build + Upload (PlatformIO)
pio run -t upload
```

> **Kein PlatformIO?** Arduino IDE öffnen → `.ino` öffnen → Libraries installieren (NimBLE, PubSubClient, ArduinoJson) → Upload.

### 2. VenusOS Daemon installieren

```bash
# Auf Cerbo GX einloggen
ssh root@<cerbo-ip>

# Temporäres Verzeichnis
cd /tmp

# Von GitHub holen (oder per scp)
curl -sL https://github.com/silly82/jbd-ble-victron-bridge/archive/main.tar.gz | tar xz
cd jbd-ble-victron-bridge-main/venusos

# 1-Klick-Installation
./install.sh
```

**Oder manuell:**

```bash
scp -r venusos/* root@<cerbo-ip>:/data/dbus-mqtt-battery/
ssh root@<cerbo-ip>
cd /data/dbus-mqtt-battery
chmod +x install.sh && ./install.sh
```

### 3. Node-RED Flow importieren (optional)

1. Auf dem Cerbo GX: VenusOS App Store → **Node-RED** installieren
2. `flows.json` aus diesem Repo importieren → Menu → Import
3. MQTT-Broker-Konfiguration anpassen
4. Deploy

## Konfiguration

### ESP32 (`esp32_jbd_ble_mqtt.ino`)

```cpp
const char* WIFI_SSID     = "FRITZ!Box 7530";     // Dein WLAN
const char* WIFI_PASS     = "geheim";               // WLAN-Passwort
const char* MQTT_HOST     = "192.168.1.100";        // Cerbo GX oder Broker
const int   MQTT_PORT     = 1883;
const char* JBD_DEVICE_NAME = "";                   // Leer = Auto-Scan
```

> `JBD_DEVICE_NAME` leer lassen = ESP scannt automatisch nach Geräten mit Namen wie `JBD-*`, `DWF*`, `SX1*`, `SBL*`.

### VenusOS Daemon (`venusos/dbus_mqtt_battery.service`)

```ini
Environment=MQTT_HOST=192.168.1.100
Environment=MQTT_TOPIC=bms/jbd/data
Environment=DBUS_INSTANCE=256
Environment=POLL_TIMEOUT=60
```

## MQTT Datenformat

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
|------------|-------------|---------|
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

## JBD BLE Protokoll (Reverse Engineering)

Basierend auf [aiobmsble](https://github.com/patman15/aiobmsble) von patman15.

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

## Directory Structure

```
jbd-ble-victron-bridge/
├── esp32_jbd_ble_mqtt/       ← ESP32 Firmware (PlatformIO / Arduino)
│   ├── esp32_jbd_ble_mqtt.ino    ← Hauptsketch
│   └── platformio.ini             ← PlatformIO Projekt
├── venusos/                   ← Cerbo GX / VenusOS
│   ├── dbus_mqtt_battery.py      ← D-Bus Daemon (Python)
│   ├── dbus_mqtt_battery.service ← systemd Service Unit
│   └── install.sh                ← Installationsscript
├── flows.json                 ← Node-RED Flow (Import)
├── .github/workflows/         ← GitHub Actions CI
├── LICENSE                    ← MIT
└── README.md                  ← Diese Datei
```

## Troubleshooting

**ESP32 findet kein JBD BMS:**
- BMS App auf dem Handy schließen (nur 1 Verbindung gleichzeitig)
- `JBD_DEVICE_NAME` leer lassen (Auto-Scan)
- Prüfen ob RSSI stark genug: ESP32 nah am Akku platzieren
- Service UUID `ff00` prüfen (manche Klone nutzen abweichende UUIDs)

**Keine Daten auf dem Cerbo:**
```bash
# MQTT prüfen
mosquitto_sub -h localhost -t bms/jbd/data

# Service-Status
systemctl status dbus-mqtt-battery

# Log
journalctl -u dbus-mqtt-battery -n 50 --no-pager
```

**Akku taucht nicht in VenusOS auf:**
- Service neustarten: `systemctl restart dbus-mqtt-battery`
- Instance-ID ändern falls belegt: `DBUS_INSTANCE=257`
- VenusOS Remote Console → Geräte → Nach neuen Geräten suchen

## Verwandte Projekte

- [aiobmsble](https://github.com/patman15/aiobmsble) — Python BLE BMS Library (die Basis)
- [BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA) — Home Assistant Integration
- [dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — VenusOS Serial Battery Driver
- [velib_python](https://github.com/victronenergy/velib_python) — Victron D-Bus Python Bindings

## Lizenz

MIT — machen damit was du willst, aber keine Garantie. Siehe [LICENSE](LICENSE).