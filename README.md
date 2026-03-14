<h1 align="center">Billabong Sentinel</h1>

<p align="center">
  Solar-powered LoRa mesh water level monitoring for rural Australia
</p>

<p align="center">
  <img alt="Hardware License" src="https://img.shields.io/badge/Hardware-CERN--OHL--S%20v2-blue?style=flat-square"/>
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

**Status:** schematic capture has completed a disciplined review pass against the current rev-A direction, and the project is now moving into PCB floorplanning and layout preparation. The power tree, pressure front end, ESP32-C3, and LoRa module direction are captured in the working schematic, but the design is not yet at PCB signoff. The earlier SHT40/SHT31 environmental sensors have been deferred from rev A because the practical GPIO budget is now consumed by the pressure path, USB, LoRa control, and bring-up circuitry.

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

Nodes wake on the ESP32-C3 deep sleep timer every 15 minutes, power the sensors briefly, transmit via LoRa mesh, then go back to sleep. The active window is about 2-3 seconds. The gateway stays on and handles routing, storage, and serving the dashboard.

If a node doesn't have direct line-of-sight to the gateway, packets hop through other nodes automatically. The routing table updates passively from beacon traffic so there's no manual configuration.

---

## Features

| | Feature | Detail |
|-|---------|--------|
| 🌊 | **Pressure transducer** | 0-5m range, ~1mm resolution. Submersible, vented cable for atmospheric compensation. |
| 📡 | **LoRa mesh** | Multi-hop 915MHz (AU915). Nodes relay for each other. No repeater infrastructure. |
| ☀️ | **Solar powered** | 1W panel + 6,000mAh 18650 backup. Target rev A draw is roughly 4-6mAh/day at 15-min intervals. |
| 💧 | **Consumption analytics** | Gateway tracks L/hour per node. Picks up leaks, overflow, and unusual usage overnight. |
| 🔔 | **Configurable alerts** | Low water, overflow, node offline, and suspected leak. |
| 🔁 | **Node-first hardware** | Rev A prioritises a simple, low-power field node before a separate gateway board. |
| 🔧 | **Deep-sleep first** | The node relies on ESP32-C3 deep sleep and switched sensor rails instead of overlapping wake schemes. |
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
| LoRa module | EBYTE E22-900M22S (SX1262-based) via SPI |
| Frequency | 915MHz (AU915, ACMA Class Licence) |
| TX power | ~22dBm module capability, firmware-limited as needed for compliance and power budget |
| Sensitivity | ~-147dBm |
| Mesh stack | RadioLib + custom mesh protocol |

</details>

<details>
<summary><strong>Sensors</strong></summary>

| Sensor | IC | Location | Purpose |
|--------|----|----------|---------|
| Water level | Pressure transducer (0-5m) | Submerged | Depth in mm |
</details>

<details>
<summary><strong>Power System</strong></summary>

| Stage | Component | Notes |
|-------|-----------|-------|
| Battery | 2x 18650 in parallel | 6,000mAh, user-replaceable without tools |
| Solar charger | MCP73871 | USB/solar charging with power-path management and VPCC input control |
| Regulator | TPS63021 buck-boost | 3.3V stable across full Li-ion range (3.0-4.2V in) |
| Protection | DW01A + FS8205 | Over/under-voltage, overcurrent, reverse polarity |
| Sensor excitation | Switched 5V rail | Pressure transducer powered only during sampling |

Target rev A node consumption is roughly 4-6mAh/day at 15-minute intervals. A 1W panel in rural Australia still provides comfortable energy margin.

</details>

<details>
<summary><strong>Enclosure</strong></summary>

ASA 3D-printed via JLCPCB, IP68 rated. Nitrile O-ring lid seal, IP68 PG9 cable gland, internal-only USB-C service access after lid removal, Gore-Tex vent plug, and stainless mounting lugs for star picket or trough rail. The enclosure should need opening only rarely, ideally never in normal service.

</details>

### Gateway

Planned as a separate always-on board using an ESP32-S3-class device for the extra RAM needed to serve the dashboard.

---

## Firmware

Planned around ESP-IDF v5.x, no Arduino layer.

```
firmware/
├── node/               # ESP-IDF project
│   ├── main/
│   └── components/
│       ├── lora_mesh/      # Mesh protocol over RadioLib
│       └── provisioning/   # BLE + native USB setup
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
            PROVISIONING (BLE or native USB)
```

OTA:

| Target | Method |
|--------|--------|
| Gateway | `esp_https_ota` over WiFi |
| Nearby node | `esptool.py` via USB-C |
| Remote node | LoRa OTA relay (gateway chunks firmware, delivers over mesh) |

---

## Dashboard

Planned to run locally on the gateway. Current direction is `esp_http_server` with Chart.js assets in LittleFS, mobile-friendly layouts, WebSocket push for live alerts, and MQTT output with Home Assistant auto-discovery.

Views: per-node water level and trend, consumption rate, alert log, signal strength, gateway status.

---

## Current Workflow

1. Capture the schematic in EasyEDA Pro.
2. Review the schematic against primary datasheets before layout.
3. Create the PCB in EasyEDA Pro.
4. Review the PCB against datasheets and implementation constraints before release.

## Repository Status

This repository currently contains planning and specification documents only. The `firmware/`, `hardware/`, `docs/`, and manufacturing outputs described above are planned deliverables that will be added as the design moves past schematic capture.

---

## Licence

| Layer | Licence |
|-------|---------|
| Hardware (schematics, PCB, enclosure) | CERN-OHL-S v2 |
| Firmware | MIT |
| Documentation | CC BY 4.0 |

CERN-OHL-S v2 is the chosen hardware licence for the design work in this project.

---

**Author:** Russell Thomas

Designed in EasyEDA Pro · Manufactured via JLCPCB · OSHWLab Stars 2026
