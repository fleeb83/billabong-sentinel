# Billabong Sentinel

## Open-Source Solar-Powered LoRa Mesh Water Level Monitor for Rural Australia

---

### The Problem

A dry water trough on a remote paddock is one of the most common and costly problems in Australian livestock farming. By the time a farmer discovers it on the next routine check, animals may have been without water for days. Commercial water level monitors exist, but they cost $500–$1,500 per unit, require monthly cellular subscriptions, and don't work where there's no phone signal — which is most of where they're needed.

---

### The Solution

**Billabong Sentinel** is a fully open-source, solar-powered water level monitoring network built for properties without cellular coverage. A weatherproof sensor node sits at each trough or dam. A single gateway at the homestead collects data from all nodes over a self-healing LoRa mesh and serves a local web dashboard — no internet required, no subscription, no ongoing cost.

The complete system — hardware, firmware, enclosure, and documentation — is open source and designed to be reproduced by anyone with access to EasyEDA Pro and JLCPCB.

**GitHub:** [https://github.com/fleeb83/billabong-sentinel](https://github.com/fleeb83/billabong-sentinel)

---

### System Overview

```
[Node 1]──LoRa mesh──[Node 2]──LoRa mesh──[Node 3]
              │                      │
         LoRa mesh               LoRa mesh
              │                      │
         [Node 4]──────────────[Gateway]──WiFi──[Web Dashboard]
                                    │
                               MQTT (optional)
                                    │
                           [Home Assistant]
```

Each **Billabong Sentinel Node** wakes from deep sleep every 15 minutes, reads the water level and ambient conditions, transmits via LoRa mesh to the gateway, then returns to deep sleep. On a 6,000mAh battery with a 1W solar panel, nodes operate indefinitely — with over 100 days of battery reserve even without any solar input.

The **Billabong Sentinel Gateway** receives all node data, stores 90 days of history, and serves a responsive local web dashboard viewable on any phone or browser on the farm network.

---

### Key Features

- **LoRa mesh networking** — nodes relay packets for each other, extending range and routing around terrain. No repeater infrastructure required.
- **Submersible pressure transducer** — measures water depth from 0–5m with ~1mm resolution. Far more reliable than float switches.
- **Water consumption rate analytics** — the gateway calculates consumption rate (L/hour) from successive readings, enabling early detection of leaks, overflow, and livestock behaviour anomalies.
- **Seal integrity monitoring** — an internal humidity sensor detects enclosure gasket failure *before* water damage occurs. Rising internal humidity triggers an alert.
- **Configurable alerts** — low water, trough overflow, node offline, seal breach, low battery, and leak detection. Delivered via the dashboard and MQTT (Home Assistant compatible).
- **Hardware watchdog** — TPL5110 hard-resets the entire node if firmware locks up, ensuring autonomous recovery in unattended field deployment.
- **Shared node/gateway PCB** — a single PCB design serves both roles via a solder bridge, halving design and manufacturing complexity.
- **Sub-$125 AUD per node** — including PCB, components, enclosure, and sensor. Commercial equivalents charge this per month in subscription fees.

---

### Hardware

#### Node

| Component | Part |
|-----------|------|
| MCU | ESP32-C3-MINI-1 (~5µA deep sleep) |
| LoRa radio | SX1276 @ 915MHz (AU915) |
| Water level | Submersible pressure transducer, 0–5m |
| Ambient sensor | SHT40 (external, in 3D-printed Stevenson screen) |
| Enclosure diagnostic | SHT31 (internal humidity, seal integrity) |
| RTC | DS3231 (±2ppm TCXO, CR2032 backup) |
| Solar charger | CN3791 (MPPT-like input tracking) |
| Regulator | TPS63021 buck-boost (3.3V across full Li-ion range) |
| Battery protection | DW01A + FS8205 |
| Watchdog | TPL5110 (hard reset on firmware lockup) |
| Battery | 2× 18650 in parallel (6,000mAh) |

**Power consumption:** ~1.07mAh/day at 15-minute sample intervals. A 1W solar panel in rural Australia provides roughly 1,000× more energy than the node consumes.

#### Enclosure

IP68 target. ASA 3D-printed via JLCPCB. Nitrile O-ring lid seal, IP68 cable gland for sensor, IP68 waterproof USB-C port for field firmware updates, Gore-Tex vent plug for pressure equalisation, stainless mounting lugs for post or rail installation.

---

### Firmware

Built on **ESP-IDF v5.x** (native, no Arduino layer). RadioLib for SX1276 control with a custom lightweight mesh protocol. FreeRTOS tasks for concurrent LoRa receive, web serving, and MQTT on the gateway. `esp_https_ota` for WiFi OTA firmware updates; LoRa-relayed OTA for remote node updates.

---

### Web Dashboard

Served directly from the gateway (no cloud, no subscription). Single-page app using `esp_http_server` and Chart.js. Shows live water levels, 7-day trends, consumption rates, alert history, and node signal strength. Mobile-responsive. Home Assistant integration via MQTT auto-discovery.

---

### Open Source

All design files are released under open licenses chosen to maximise community collaboration:

| Layer | License |
|-------|---------|
| Hardware (schematics, PCB, enclosure) | CERN-OHL-W v2 |
| Firmware | MIT |
| Documentation | CC BY 4.0 |

The full repository — firmware, EasyEDA Pro project files, 3D enclosure STL/STEP, assembly guide, field installation guide, calibration procedure, and complete BOM — is published on GitHub and OSHWLab.

**GitHub:** [https://github.com/fleeb83/billabong-sentinel](https://github.com/fleeb83/billabong-sentinel)

---

### Who Is This For?

Any farmer, grazier, or rural landowner who needs to monitor water points across a large property — especially where there's no phone signal. The system is designed to be built and deployed by someone with basic electronics skills, and the documentation covers everything from PCB assembly to field installation to dashboard setup.

---

### About

**Author:** Russell Thomas
**Designed in:** EasyEDA Pro
**Manufactured via:** JLCPCB
**License:** CERN-OHL-W v2 · MIT · CC BY 4.0

*Built for the bush.*
