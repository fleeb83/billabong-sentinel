# Billabong Sentinel

### Solar-powered LoRa mesh water level monitoring for rural Australia

---

Stock water is a constant headache on large rural properties. Troughs run dry, floats fail, tanks overflow, and by the time you find out on the next round you've already lost time and potentially stock. Commercial monitors exist but they're expensive, need cellular coverage, and charge monthly fees on top.

Billabong Sentinel is a practical alternative. It uses LoRa mesh radio to link sensor nodes at each trough or dam back to a gateway at the homestead. The gateway runs a local web dashboard and can push alerts to Home Assistant via MQTT. No internet required, no subscription, and the nodes run indefinitely on a 1W solar panel.

**Status:** active schematic capture. The current working design has the power tree, pressure front end, ESP32-C3, and LoRa module direction captured, but schematic review and cleanup are still in progress before PCB.

**GitHub:** [https://github.com/fleeb83/billabong-sentinel](https://github.com/fleeb83/billabong-sentinel)

---

### How it works

```
[Node 1]--LoRa mesh--[Node 2]--LoRa mesh--[Node 3]
             |                      |
        LoRa mesh               LoRa mesh
             |                      |
        [Node 4]--------------[Gateway]--WiFi--[Dashboard]
                                   |
                              MQTT (optional)
```

Each node wakes on the ESP32-C3 deep sleep timer every 15 minutes, powers its sensors briefly, samples water level (pressure transducer), ambient conditions (SHT40), and internal enclosure humidity (SHT31 as a seal-integrity check), then transmits the packet over the LoRa mesh and goes back to sleep. Active window is about 2-3 seconds. The gateway stays on, handles routing and storage, and serves the dashboard.

Packets route through neighbouring nodes if there's no direct path to the gateway, so coverage extends well beyond a single radio hop.

---

### Key features

- **LoRa mesh, not star topology.** Nodes relay for each other. No repeater hardware needed.
- **Pressure transducer sensing.** 0-5m range, ~1mm resolution. More accurate and more reliable than float switches.
- **Consumption rate tracking.** The gateway calculates litres per hour per node from the level readings. Useful for catching overnight leaks and spotting unusual usage.
- **Seal integrity monitoring.** An internal SHT31 watches enclosure humidity. If the gasket starts to fail, you get an alert before water reaches the PCB.
- **Node-first hardware.** Rev A prioritises a simple, low-power field node before a separate gateway board.
- **Deep-sleep first.** The node relies on ESP32-C3 deep sleep and switched sensor rails instead of overlapping wake schemes.
- **~$125 AUD per node** including PCB, enclosure, 18650s, solar panel, and sensor.

---

### Hardware

| Component | Part |
|-----------|------|
| MCU | ESP32-C3-MINI-1 (~5µA deep sleep) |
| LoRa | EBYTE E22-900M22S (SX1262-based) @ 915MHz (AU915) |
| Water level | Submersible pressure transducer, 0-5m, vented cable |
| Ambient | SHT40 (external, 3D-printed Stevenson screen housing) |
| Enclosure check | SHT31 (internal, PCB-mounted) |
| Solar charger | MCP73871 (USB/solar charging with power-path management and VPCC input control) |
| Regulator | TPS63021 buck-boost (3.3V stable across full Li-ion range) |
| Battery protection | DW01A + FS8205 |
| Sensor excitation | Switched 5V rail for the pressure transducer |
| Battery | 2x 18650 in parallel (6,000mAh) |

Target rev A node consumption is roughly 4-6mAh/day at 15-minute intervals. A 1W panel in rural Australia still provides comfortable energy margin.

The enclosure is ASA 3D-printed via JLCPCB, IP68 rated with a nitrile O-ring lid seal, IP68 cable gland, waterproof USB-C port for field firmware updates, and a Gore-Tex vent plug.

---

### Firmware

Planned around ESP-IDF v5.x (native, no Arduino). RadioLib for the E22-900M22S / SX1262 radio path with a custom lightweight mesh layer on top. FreeRTOS tasks on the gateway will handle concurrent LoRa receive, web serving, and MQTT. Remote node OTA over LoRa remains a later-stage goal.

---

### Dashboard

Planned to run locally on the gateway. Current direction is `esp_http_server` with Chart.js for trend graphs. The dashboard is intended to show live levels, 7-day history, consumption rate, alert log, and node signal strength, with MQTT output including Home Assistant auto-discovery topics.

---

### Open source

All hardware, firmware, enclosure files, and documentation are intended to be published under open licences:

| Layer | Licence |
|-------|---------|
| Hardware (schematics, PCB, enclosure) | CERN-OHL-S v2 |
| Firmware | MIT |
| Documentation | CC BY 4.0 |

At the current stage, the repository contains project definition documents. EasyEDA Pro design files, manufacturing outputs, firmware, and build documentation are planned to be added as the hardware work progresses.

**GitHub:** [https://github.com/fleeb83/billabong-sentinel](https://github.com/fleeb83/billabong-sentinel)

---

**Author:** Russell Thomas
Designed in EasyEDA Pro · Manufactured via JLCPCB · CERN-OHL-S v2 / MIT / CC BY 4.0
