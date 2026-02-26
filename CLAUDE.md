# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Competition Context

This project is an entry in the **2026 OSHWLab Stars Award** (oshwlab.com). Key constraints and goals:

- **EasyEDA Pro required** — all schematic/PCB work must be done in EasyEDA Pro (not standard). Files must be fully open-sourced on OSHWLab.
- **Functional verification** — shortlisted projects are physically shipped for verification. The hardware must actually work. MVP functionality must be solid before submission.
- **Submission deadline:** December 31, 2026
- **License:** CERN-OHL-S v2 (already chosen)

**Judging criteria (preliminary selection):**

| Dimension | Weight | What this means for Billabong Sentinel |
|-----------|--------|-------------------------------|
| Innovation | 30% | LoRa mesh + solar + pressure transducer combo for rural AU; emphasis on novel approaches |
| Practicality | 25% | Solves a real, unmet need for farmers without cellular coverage |
| Open-Source Friendliness | 20% | Complete documentation, BOM, assembly notes, firmware — everything needed to reproduce |
| Technical Implementation | 15% | PCB layout quality, protection circuits, power design |
| Environmental Sustainability | 10% | Solar-powered, long battery life, low standby power |

**Final scoring:** Expert judges 50% + Professional reviewers 30% + Public influence 20% (OSHWLab engagement + YouTube views/likes/comments).

**Judges:** Dave Jones (EEVblog), GreatScott, Robert Feranec (FEDEVEL), Max Imagination, Philip Salmony (Phil's Lab), WillCogley (NMRobotics).

**Strategic implications:** Documentation quality and open-source completeness are weighted heavily (20%). Every design decision should be explained in project documentation. The public influence component means the OSHWLab project page and any YouTube content matters — write for an audience, not just internal notes.

---

## Project Overview

Billabong Sentinel is an open-source, solar-powered LoRa mesh water level monitoring system for rural Australia. It targets farmers monitoring stock water troughs/tanks across large properties without cellular coverage. Licensed under CERN-OHL-S v2.

The system has two hardware variants:
- **Node** — weatherproof sensor unit at each trough/tank (ESP32-C3)
- **Gateway** — base station at the farm office that aggregates data and serves a local web dashboard (ESP32-S3 or ESP32)

## System Architecture

```
[Node 1]──LoRa mesh──[Node 2]──LoRa mesh──[Node 3]
                         │
                    LoRa mesh
                         │
                    [Gateway]──WiFi/Ethernet──[Dashboard]
```

- LoRa mesh at 915MHz (AU915 band, ACMA Class Licence)
- Nodes wake periodically (~15 min intervals), sample sensors, transmit via LoRa, return to deep sleep
- Gateway aggregates all node data, serves local web interface, optionally pushes to MQTT (Home Assistant)

## Hardware Design

**PCB design:** EasyEDA Pro (required for competition — do not use EasyEDA Standard)
**Manufacturing:** JLCPCB (PCB + PCBA)
**Component sourcing:** LCSC

The gateway may share the same PCB as the node, differentiated by a configuration jumper or solder bridge.

## Node Key Components

| Role | IC |
|------|----|
| MCU | ESP32-C3-MINI-1 (RISC-V, ~5µA deep sleep) |
| LoRa radio | SX1276 (SPI) |
| Water level | Analog pressure transducer, 0–5m, 0.5–4.5V ratiometric |
| Ambient temp/humidity | SHT40 (I2C, external) |
| Enclosure diagnostic | SHT31 (I2C, internal) |
| RTC | DS3231 (I2C, CR2032 backup) |
| Solar charger | CN3791 (MPPT-like) |
| Battery protection | DW01A + FS8205 |
| Watchdog | TPL5110 (hard reset on firmware lockup) |
| USB-UART | CH340C |

**Power:** 2× 18650 parallel (6000mAh), 5V 1–2W solar panel, 3.3V regulated output. Target: >100 days battery reserve at 15-min sample intervals (~1.5mAh/day).

**Analog input protection:** series 1kΩ resistor + TVS diode + RC filter + resistor divider to keep within 0–3.3V ADC range.

## Firmware (TBD)

- Platform: ESP-IDF v5.x (native, no Arduino layer); build with `idf.py build`, flash with `idf.py flash`, monitor with `idf.py monitor`
- LoRa mesh stack: RadioLib custom protocol **or** Meshtastic-compatible (decision pending)
- Node provisioning: unique hardware ID in flash, human-readable names configured via BLE or USB-C serial
- ACMA duty cycle limits must be enforced in firmware
- OTA: WiFi-based at gateway; USB-C at nodes during maintenance; LoRa OTA delivery to remote nodes (TBD)

## Dashboard (TBD)

Software stack not yet decided — ESPHome or custom. Served locally from the gateway. Optional MQTT output.

## Open Design Decisions

- 3.3V regulator: LDO vs boost converter (depends on minimum battery voltage floor)
- LoRa mesh firmware: RadioLib vs Meshtastic
- Gateway: separate PCB vs shared PCB with jumper
- Dashboard software stack
- OTA mechanism for remote nodes via LoRa
- External SHT40 housing/mounting
- Tilt/tamper detection (optional v1.1 feature)
- Final ACMA EIRP confirmation

## Regulatory

- LoRa operates under ACMA Radiocommunications (Low Interference Potential Devices) Class Licence 2015
- PCB standard: IPC-2221
- Components: RoHS compliant via JLCPCB/LCSC
