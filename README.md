<p align="center">
  <img src="docs/images/thumbnail.png" alt="Billabong Sentinel" width="100%"/>
</p>

<h1 align="center">Billabong Sentinel</h1>

<p align="center">
  <strong>Open-source solar-powered LoRa mesh water level monitoring for rural Australia</strong><br/>
  No cellular. No subscription. No compromise.
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

## The Problem

A dry trough on a remote paddock is one of the most costly — and most preventable — problems in Australian livestock farming. By the time it's discovered on the next routine check, animals may have been without water for days.

Commercial water level monitors exist. They cost **$500–$1,500 per unit** plus **monthly cellular subscription fees**. And they don't work where there's no phone signal — which is most of where they're actually needed.

## The Solution

Billabong Sentinel is a self-contained, solar-powered water monitoring network that runs entirely on your property with no internet, no cloud, and no recurring costs.

Deploy a **sensor node** at each trough or dam. Install one **gateway** at the homestead. Get a live web dashboard showing every water point on the property — with alerts when levels drop, leaks are detected, or a node goes silent.

**Total cost per node: ~$125 AUD.** Built for the bush. Designed to last.

---

## How It Works

```
  [Node 1] ──── LoRa mesh ──── [Node 2] ──── LoRa mesh ──── [Node 3]
      │                             │
  LoRa mesh                    LoRa mesh
      │                             │
  [Node 4] ─────────────────── [Gateway] ──── WiFi ──── [Dashboard]
                                    │
                               MQTT (optional)
                                    │
                            [Home Assistant]
```

Nodes wake every 15 minutes, sample water level and environmental conditions, transmit via a **self-healing LoRa mesh**, then return to deep sleep. The gateway aggregates all data and serves a **local web dashboard** — readable on any phone or browser on the farm network.

No node needs direct line-of-sight to the gateway. Packets hop between nodes automatically, routing around terrain and obstacles.

---

## Features

| | Feature | Detail |
|-|---------|--------|
| 🌊 | **Pressure transducer sensing** | 0–5m range, ~1mm resolution. Far more reliable than float switches. |
| 📡 | **LoRa mesh networking** | Multi-hop 915MHz mesh. Nodes relay for each other. Kilometres of range. |
| ☀️ | **Solar powered** | 1W panel + 6,000mAh 18650 backup. >100 days reserve without sun. |
| 💧 | **Consumption rate analytics** | Gateway calculates L/hour per node. Detects leaks, overflow, and dry troughs automatically. |
| 🔒 | **Seal integrity monitoring** | Internal humidity sensor alerts before water damage occurs — a novel diagnostic for field electronics. |
| 🔔 | **Configurable alerts** | Low water, overflow, node offline, seal breach, low battery, leak detection. |
| 🔁 | **Hardware watchdog** | TPL5110 hard-resets the node if firmware locks up. Full autonomy in remote deployment. |
| 🔧 | **Single shared PCB** | Node and gateway built from the same board. One solder bridge selects the role. |
| 🏠 | **Home Assistant** | Native MQTT auto-discovery. Integrates with your existing home automation setup. |
| 🌐 | **No cloud dependency** | Dashboard runs entirely on the gateway. Works with zero internet connectivity. |

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
| Sensitivity | −148dBm |
| Mesh stack | RadioLib + custom lightweight mesh protocol |

</details>

<details>
<summary><strong>Sensors</strong></summary>

| Sensor | IC | Location | Purpose |
|--------|----|----------|---------|
| Water level | Pressure transducer (0–5m) | Submerged | Depth in mm |
| Ambient temp/humidity | SHT40 | External Stevenson screen | Environmental logging |
| Enclosure diagnostic | SHT31 | PCB (internal) | Seal integrity monitoring |
| Real-time clock | DS3231 (±2ppm TCXO) | PCB | Accurate timestamps, wake alarm |

</details>

<details>
<summary><strong>Power System</strong></summary>

| Stage | Component | Notes |
|-------|-----------|-------|
| Battery | 2× 18650 in parallel | 6,000mAh, user-replaceable without tools |
| Solar charger | CN3791 | MPPT-like input tracking |
| Regulator | TPS63021 buck-boost | 3.3V across full Li-ion range (3.0–4.2V in) |
| Protection | DW01A + FS8205 | Over/under-voltage, overcurrent, reverse polarity |
| Watchdog | TPL5110 | Hard power-cycle on firmware lockup |

**Power budget:** ~1.07mAh/day at 15-minute intervals.
A 1W solar panel in rural Australia harvests roughly **1,000× more energy than the node consumes.**

</details>

<details>
<summary><strong>Enclosure</strong></summary>

- **Material:** ASA (UV-stable to 95°C)
- **Rating:** IP68 target
- **Manufacture:** JLCPCB 3D printing
- **Seal:** Nitrile O-ring + stainless M3 screws
- **Cable entry:** IP68 PG9 cable gland with mandatory drip loop
- **Comms:** IP68 waterproof USB-C panel mount (field firmware updates)
- **Vent:** Gore-Tex plug (pressure equalisation, no water ingress)
- **Mounting:** Stainless U-bolt lugs — fits star picket or trough rail

</details>

### Gateway

The gateway runs on the same PCB as the node. JP1 solder bridge open = gateway mode. JP1 bridged = node mode. The ESP32-S3 variant is used for additional RAM when serving the web dashboard.

---

## Firmware

Built on **ESP-IDF v5.x** — native, no Arduino layer.

```
firmware/
├── node/               # ESP-IDF project
│   ├── main/
│   └── components/
│       ├── lora_mesh/      # Custom mesh protocol over RadioLib
│       ├── sensor_drivers/ # SHT40, SHT31, DS3231 (native I2C)
│       └── provisioning/   # BLE + USB-C serial setup
└── gateway/            # ESP-IDF project
    ├── main/
    └── components/
        ├── lora_mesh/      # Mesh receive + relay + routing table
        ├── web_server/     # esp_http_server + LittleFS dashboard
        └── mqtt_client/    # esp_mqtt + Home Assistant discovery
```

**Node state machine:**
```
COLD_BOOT → PROVISION_CHECK → SAMPLE → MESH_TX → SLEEP
                  │
            (if unconfigured)
                  ↓
            PROVISIONING (BLE or USB-C serial)
```

**OTA strategy:**

| Target | Method |
|--------|--------|
| Gateway | `esp_https_ota` over WiFi |
| Nearby node | `esptool.py` via USB-C |
| Remote node | LoRa OTA relay — gateway chunks firmware, delivers over mesh |

---

## Dashboard

Served locally from the gateway. No internet required.

- Built on `esp_http_server` + Chart.js (stored in LittleFS)
- Mobile-responsive, high-contrast (readable in direct sunlight)
- Live water levels, 7-day trend charts, consumption rate per node
- Alert log, node signal strength, gateway uptime
- WebSocket push for real-time alerts
- MQTT output with Home Assistant auto-discovery

---

## Getting Started

> Full guides are in [`/docs`](docs/)

### What You Need

- Billabong Sentinel PCB (order via JLCPCB using the provided Gerbers + BOM)
- 2× 18650 cells, 5V 1–2W solar panel, submersible pressure transducer (sourced separately — see BOM)
- IP68 enclosure (order via JLCPCB 3D printing using provided STL files)
- ESP-IDF v5.x installed on your development machine

### Flash the Firmware

```bash
# Clone the repo
git clone https://github.com/fleeb83/billabong-sentinel.git
cd billabong-sentinel/firmware/node

# Build and flash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### Provision a Node

Connect via USB-C and run the provisioning script, or use the BLE setup interface:

```bash
python3 tools/provision.py --port /dev/ttyUSB0 --name "Home Trough" --gateway-id AA:BB:CC:DD
```

---

## Repository Structure

```
billabong-sentinel/
├── firmware/
│   ├── node/               # Node ESP-IDF project
│   └── gateway/            # Gateway ESP-IDF project
├── hardware/
│   ├── easyeda/            # EasyEDA Pro schematic + PCB (JSON)
│   └── enclosure/          # STL and STEP files for 3D printing
├── docs/
│   ├── assembly-guide.md
│   ├── field-install.md
│   ├── calibration.md
│   ├── troubleshooting.md
│   └── bom.md
├── tools/
│   └── provision.py        # Node provisioning script
└── README.md
```

---

## Licence

| Layer | Licence |
|-------|---------|
| Hardware (schematics, PCB, enclosure) | [CERN-OHL-W v2](https://ohwr.org/cern_ohl_w_v2.txt) |
| Firmware | [MIT](LICENSE-MIT) |
| Documentation | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) |

CERN-OHL-W v2 means: if you modify the hardware design, share your changes. If you just build and use the hardware, no obligations.

---

## About

**Author:** Russell Thomas

Designed in EasyEDA Pro · Manufactured via JLCPCB · Entered in OSHWLab Stars 2026

*Built for the bush.*
