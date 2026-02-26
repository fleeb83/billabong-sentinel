<p align="center">
  <img src="docs/images/thumbnail.png" alt="Billabong Sentinel" width="100%"/>
</p>

<h1 align="center">Billabong Sentinel</h1>

<p align="center">
  Solar-powered LoRa mesh water level monitoring for rural Australia
</p>

<p align="center">
  <img alt="Hardware License" src="https://img.shields.io/badge/Hardware-CERN--OHL--W%20v2-blue?style=flat-square"/>
  <img alt="Firmware License" src="https://img.shields.io/badge/Firmware-MIT-green?style=flat-square"/>
  <img alt="Docs License" src="https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey?style=flat-square"/>
  <img alt="Platform" src="https://img.shields.io/badge/Platform-ESP--IDF%20v5-red?style=flat-square"/>
  <img alt="PCB" src="https://img.shields.io/badge/PCB-EasyEDA%20Pro-orange?style=flat-square"/>
  <img alt="Status" src="https://img.shields.io/badge/Status-Active%20Development-brightgreen?style=flat-square"/>
  <img alt="OSHWLab Stars" src="https://img.shields.io/badge/OSHWLab%20Stars-2026%20Entry-gold?style=flat-square"/>
</p>

---

Billabong Sentinel is an open-source system for monitoring stock water troughs and dams on large rural properties. A weatherproof sensor node at each water point reports back to a gateway at the homestead over a self-healing LoRa mesh. No cellular connection needed, no cloud, no subscription.

The web dashboard runs locally on the gateway. Alerts for low water, leaks, and offline nodes go out via MQTT or show up on the dashboard. Works fine with Home Assistant.

Commercial cellular water monitors run $500-$1,500 per unit plus monthly fees, and most don't work where there's no phone signal anyway. A Billabong Sentinel node comes in at around $125 AUD all up.

---

## How It Works

```
  [Node 1] ---- LoRa mesh ---- [Node 2] ---- LoRa mesh ---- [Node 3]
      |                             |
  LoRa mesh                    LoRa mesh
      |                             |
  [Node 4] --------------------- [Gateway] ---- WiFi ---- [Dashboard]
                                    |
                               MQTT (optional)
                                    |
                            [Home Assistant]
```

Nodes wake on a DS3231 RTC alarm every 15 minutes, read all sensors, transmit via LoRa mesh, kick the hardware watchdog, then go back to sleep. The active window is about 3 seconds. The gateway stays on and handles routing, storage, and serving the dashboard.

If a node doesn't have direct line-of-sight to the gateway, packets hop through other nodes automatically. The routing table updates passively from beacon traffic so there's no manual configuration.

---

## Features

| | Feature | Detail |
|-|---------|--------|
| 🌊 | **Pressure transducer** | 0-5m range, ~1mm resolution. Submersible, vented cable for atmospheric compensation. |
| 📡 | **LoRa mesh** | Multi-hop 915MHz (AU915). Nodes relay for each other. No repeater infrastructure. |
| ☀️ | **Solar powered** | 1W panel + 6,000mAh 18650 backup. Daily draw ~1mAh at 15-min intervals. |
| 💧 | **Consumption analytics** | Gateway tracks L/hour per node. Picks up leaks, overflow, and unusual usage overnight. |
| 🔒 | **Enclosure diagnostics** | Internal humidity sensor watches for gasket failure before water gets to the PCB. |
| 🔔 | **Configurable alerts** | Low water, overflow, node offline, seal breach, low battery, suspected leak. |
| 🔁 | **Hardware watchdog** | TPL5110 hard-resets the node if firmware hangs. Important for unattended remote deployment. |
| 🔧 | **Single PCB, two roles** | Node and gateway use the same board. A solder bridge selects the mode. |
| 🏠 | **Home Assistant** | MQTT auto-discovery. Drops straight into an existing setup. |
| 🌐 | **Local only** | Dashboard runs on the gateway. Works with no internet at all. |

---

## Hardware

### Node

<details>
<summary><strong>Microcontroller & Radio</strong></summary>

| Parameter | Value |
|-----------|-------|
| MCU | ESP32-C3-MINI-1 (RISC-V, 160MHz) |
| Deep sleep current | ~5µA |
| LoRa IC | SX1276 via SPI |
| Frequency | 915MHz (AU915, ACMA Class Licence) |
| TX power | +17dBm (firmware-limited) |
| Sensitivity | -148dBm |
| Mesh stack | RadioLib + custom mesh protocol |

</details>

<details>
<summary><strong>Sensors</strong></summary>

| Sensor | IC | Location | Purpose |
|--------|----|----------|---------|
| Water level | Pressure transducer (0-5m) | Submerged | Depth in mm |
| Ambient temp/humidity | SHT40 | External Stevenson screen | Environmental logging |
| Enclosure diagnostic | SHT31 | PCB (internal) | Seal integrity |
| Real-time clock | DS3231 (±2ppm TCXO) | PCB | Timestamps, wake alarm |

</details>

<details>
<summary><strong>Power System</strong></summary>

| Stage | Component | Notes |
|-------|-----------|-------|
| Battery | 2x 18650 in parallel | 6,000mAh, user-replaceable without tools |
| Solar charger | CN3791 | MPPT-like input voltage tracking |
| Regulator | TPS63021 buck-boost | 3.3V stable across full Li-ion range (3.0-4.2V in) |
| Protection | DW01A + FS8205 | Over/under-voltage, overcurrent, reverse polarity |
| Watchdog | TPL5110 | Hard power-cycle if firmware stops responding |

~1.07mAh/day at 15-minute intervals. A 1W panel in rural Australia produces roughly 1,000x that.

</details>

<details>
<summary><strong>Enclosure</strong></summary>

ASA 3D-printed via JLCPCB, IP68 rated. Nitrile O-ring lid seal, IP68 PG9 cable gland, waterproof USB-C port for field firmware updates, Gore-Tex vent plug, and stainless mounting lugs for star picket or trough rail.

</details>

### Gateway

Same PCB as the node. JP1 solder bridge open = gateway mode, bridged = node mode. Uses an ESP32-S3 for the extra RAM needed to serve the dashboard.

---

## Firmware

Built on ESP-IDF v5.x, no Arduino layer.

```
firmware/
├── node/               # ESP-IDF project
│   ├── main/
│   └── components/
│       ├── lora_mesh/      # Mesh protocol over RadioLib
│       ├── sensor_drivers/ # SHT40, SHT31, DS3231 (native I2C)
│       └── provisioning/   # BLE + USB-C serial setup
└── gateway/            # ESP-IDF project
    ├── main/
    └── components/
        ├── lora_mesh/      # Receive, relay, routing table
        ├── web_server/     # esp_http_server + LittleFS dashboard
        └── mqtt_client/    # esp_mqtt + Home Assistant discovery
```

Node state machine:
```
COLD_BOOT -> PROVISION_CHECK -> SAMPLE -> MESH_TX -> SLEEP
                  |
            (if unconfigured)
                  v
            PROVISIONING (BLE or USB-C serial)
```

OTA:

| Target | Method |
|--------|--------|
| Gateway | `esp_https_ota` over WiFi |
| Nearby node | `esptool.py` via USB-C |
| Remote node | LoRa OTA relay (gateway chunks firmware, delivers over mesh) |

---

## Dashboard

Served from the gateway over the local network. Built on `esp_http_server` with Chart.js assets in LittleFS. Mobile-friendly, high contrast for outdoor use. WebSocket push for live alerts. MQTT output with Home Assistant auto-discovery.

Views: per-node water level and trend, consumption rate, alert log, signal strength, gateway status.

---

## Getting Started

> Full guides are in [`/docs`](docs/)

### What You Need

- Billabong Sentinel PCB (order via JLCPCB using the Gerbers and BOM in `/hardware`)
- 2x 18650 cells, 5V 1-2W solar panel, submersible pressure transducer (see BOM for sources)
- IP68 enclosure (order via JLCPCB 3D printing using the STL files in `/hardware/enclosure`)
- ESP-IDF v5.x on your dev machine

### Flash the Firmware

```bash
git clone https://github.com/fleeb83/billabong-sentinel.git
cd billabong-sentinel/firmware/node

idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### Provision a Node

```bash
python3 tools/provision.py --port /dev/ttyUSB0 --name "Home Trough" --gateway-id AA:BB:CC:DD
```

---

## Repository Structure

```
billabong-sentinel/
├── firmware/
│   ├── node/
│   └── gateway/
├── hardware/
│   ├── easyeda/            # EasyEDA Pro schematic + PCB
│   └── enclosure/          # STL and STEP files
├── docs/
│   ├── assembly-guide.md
│   ├── field-install.md
│   ├── calibration.md
│   ├── troubleshooting.md
│   └── bom.md
├── tools/
│   └── provision.py
└── README.md
```

---

## Licence

| Layer | Licence |
|-------|---------|
| Hardware (schematics, PCB, enclosure) | [CERN-OHL-W v2](https://ohwr.org/cern_ohl_w_v2.txt) |
| Firmware | [MIT](LICENSE-MIT) |
| Documentation | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) |

CERN-OHL-W v2: if you modify the hardware design, share the changes. No obligations if you just build and use it.

---

**Author:** Russell Thomas

Designed in EasyEDA Pro · Manufactured via JLCPCB · OSHWLab Stars 2026
