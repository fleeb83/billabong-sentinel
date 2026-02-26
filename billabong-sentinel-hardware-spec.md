# Billabong Sentinel — Rural Water Level Monitoring System
## Hardware Specification v0.2

**Project:** Billabong Sentinel
**Author:** Russell Thomas
**Date:** February 2026
**License:** CERN-OHL-W v2 (hardware) · MIT (firmware) · CC BY 4.0 (documentation)
**Status:** Draft — OSHWLab Stars 2026 Entry

---

## 1. Overview

### 1.1 Problem Statement

Rural Australian farmers managing large properties face a persistent, costly problem: stock water troughs and tanks run dry without warning. A single dry trough on a remote paddock can mean dead or stressed livestock discovered days later. Commercial cellular-based water level monitors cost $500–$1,500 per unit plus monthly subscription fees, and they don't work where there is no cellular coverage — which is most of where they are needed.

Billabong Sentinel solves this with an open-source, solar-powered, LoRa mesh water monitoring network. A farmer installs a node at each trough or dam, a single gateway at the homestead, and gets a live web dashboard showing water levels across the entire property — with no subscription, no cellular dependency, and no ongoing cost.

### 1.2 System Summary

Billabong Sentinel consists of two open-source hardware designs:

- **Billabong Sentinel Node** — a weatherproof, solar-powered sensor unit deployed at each water point
- **Billabong Sentinel Gateway** — a base station that aggregates all node data and serves a local web dashboard

Both designs share the same PCB, differentiated by a solder bridge. All hardware is designed in EasyEDA Pro, manufactured via JLCPCB, and fully documented for community reproduction.

### 1.3 Key Innovations

1. **Mesh networking, not star topology.** LoRa nodes relay packets for each other, extending range beyond line-of-sight and routing around obstacles. No repeater infrastructure required.
2. **Enclosure seal integrity monitoring.** An internal humidity sensor detects gasket degradation before water damage occurs — a genuinely novel diagnostic for field-deployed electronics.
3. **Water consumption rate analytics.** The gateway calculates water consumption rate (L/hour) from successive pressure readings, enabling early detection of leaks, trough overflow, and livestock behaviour anomalies.
4. **Shared node/gateway PCB.** A single PCB design serves both roles via a configuration solder bridge, halving design effort and enabling community members to deploy either role from the same order.
5. **Hardware watchdog for unattended autonomy.** TPL5110 hard-resets the entire system if firmware locks up — no human intervention required in remote paddock deployments.
6. **Sub-$100 AUD per node.** Target landed cost is under AUD $100 per node including PCB, components, enclosure, and sensor. Commercial equivalents charge this per month in subscription fees alone.

---

## 2. System Architecture

```
[Node 1]──LoRa mesh──[Node 2]──LoRa mesh──[Node 3]
              │                      │
         LoRa mesh               LoRa mesh
              │                      │
         [Node 4]──────────────[Gateway]──WiFi──[Web Dashboard]
                                    │
                              (Optional)
                              MQTT Broker
                                    │
                           [Home Assistant]
```

### 2.1 Mesh Protocol

- LoRa at 915MHz (AU915), ACMA Class Licence
- RadioLib library for radio abstraction; custom lightweight mesh layer
- Each node maintains a routing table updated by beacon packets
- Source routing with fallback to flooding for unrouted packets
- ACK-based delivery confirmation; gateway retransmits missed readings at next wake cycle
- Duty cycle enforced in firmware per ACMA class licence requirements
- Unique 32-bit node ID burned during provisioning; human-readable names stored at gateway

### 2.2 Data Flow

1. Node wakes from deep sleep on RTC alarm (15-minute default, configurable)
2. Sensors powered up via switched 3.3V rail
3. Pressure transducer, SHT40, SHT31 sampled; DS3231 timestamp read
4. Packet assembled: `{node_id, timestamp, water_mm, temp_C, humidity_pct, internal_temp_C, internal_humidity_pct, battery_mv, solar_mv, rssi_last}`
5. SX1276 transmits via mesh route to gateway; watchdog kicked
6. All peripherals powered down; ESP32-C3 enters deep sleep
7. Gateway receives packet, stores in circular buffer (SPIFFS), updates dashboard

### 2.3 Alert System

Configurable alert thresholds stored at the gateway, triggered conditions:

| Alert | Trigger | Delivery |
|-------|---------|----------|
| Low water | Level < threshold (e.g. 200mm) | MQTT publish, dashboard banner |
| Trough overflow | Level > max threshold sustained > 10 min | MQTT publish |
| No contact | Node silent > 2× sample interval | Dashboard warning |
| Seal breach | Internal RH > 80% for 30 min | MQTT publish, dashboard warning |
| Low battery | Battery voltage < 3.3V | Dashboard warning |
| Leak detected | Consumption rate > normal range at night | MQTT publish |

---

## 3. Billabong Sentinel Node

### 3.1 Microcontroller

| Parameter | Value |
|-----------|-------|
| IC | ESP32-C3-MINI-1 (module) |
| Core | Single-core RISC-V @ 160MHz |
| Flash | 4MB (module-integrated) |
| Wireless | WiFi 2.4GHz + BLE 5.0 |
| LoRa | External SX1276 via SPI |
| Deep sleep current | ~5µA |
| Supply voltage | 3.3V |

WiFi used for OTA firmware updates during maintenance. BLE used for initial node provisioning (set node name, sample interval, thresholds) via mobile app or serial terminal.

**GPIO management during deep sleep:** All GPIO pin states are explicitly configured before entering deep sleep. Sensor power rail GPIO is driven low. SPI and I2C pins are set to input (no drive) to prevent parasitic current paths.

### 3.2 LoRa Radio

| Parameter | Value |
|-----------|-------|
| IC | SX1276 |
| Frequency | 915MHz (AU915) |
| Interface | SPI + CS (GPIO8), RST (GPIO9), DIO0 (GPIO4) |
| Max TX power | +17dBm (firmware-limited for ACMA compliance) |
| Sensitivity | -148dBm |
| Sleep current | ~0.2µA |
| Antenna | U.FL connector, external via SMA bulkhead |
| RF ESD | PRTR5V0U2X TVS on antenna trace |
| Impedance | 50Ω coplanar waveguide on antenna trace |

**PCB note:** RF trace from SX1276 to U.FL routed as 50Ω CPWG on the top copper layer with ground plane beneath. No vias on RF trace. Keepout area under antenna connection. Ground stitching vias around RF section perimeter.

### 3.3 Water Level Sensor

| Parameter | Value |
|-----------|-------|
| Type | Submersible pressure transducer, analog 4–20mA or 0.5–4.5V ratiometric |
| Range | 0–5m water column |
| Output | 0.5–4.5V ratiometric (preferred) or 4–20mA with shunt resistor |
| Cable | Vented atmospheric reference tube, 5–10m |
| Resolution | ~1.2mm (12-bit ADC, 5m range) |
| Calibration | Two-point calibration (dry = 0mm, known depth) stored in node flash |

**Input protection chain (in order):**
1. 1kΩ series resistor (limits fault current)
2. PRTR5V0U2X TVS diode to GND (clamps transients from long cable)
3. 10nF + 100Ω RC low-pass filter (rejects RF and switching noise)
4. 2:1 resistor divider (100kΩ / 100kΩ) — maps 0–4.5V sensor output to 0–2.25V, within ESP32-C3 ADC input range
5. ADC input on ESP32-C3 (12-bit, 0–2.5V attenuation mode)

**Sensor power:** 3.3V ratiometric sensors require a stable reference. Sensor excitation supplied from the switched sensor rail (not always-on), eliminating quiescent draw during deep sleep.

**Calibration procedure:** Documented in the field installation guide. Sensor suspended at known depth, calibration command issued via USB-C serial or BLE. Offset and span stored in flash.

### 3.4 Environmental Sensors

**External (ambient):**

| Parameter | Value |
|-----------|-------|
| IC | SHT40 |
| Interface | I2C (address 0x44) |
| Location | External to enclosure, inside 3D-printed Stevenson screen mini-housing |
| Housing | 25mm diameter, louvred ASA print; UV-stable; mounted on sensor cable 300mm below enclosure |

**Internal (enclosure diagnostic):**

| Parameter | Value |
|-----------|-------|
| IC | SHT31 |
| Interface | I2C (address 0x44, shared bus — SHT40 uses 0x44 on separate I2C port or address-select resistor) |
| Location | PCB-mounted inside enclosure |
| Alert threshold | >80% RH sustained for 30 minutes → seal breach alert |

**I2C bus:** Single I2C bus (SDA GPIO5, SCL GPIO6). SHT40 external and SHT31 internal use different I2C addresses (0x44 and 0x45 via ADDR pin). DS3231 at 0x68. Pull-ups: 4.7kΩ to switched 3.3V sensor rail — pull-ups are powered down with the sensor rail during deep sleep, eliminating the pull-up current path through floating GPIO pins.

### 3.5 Real-Time Clock

| Parameter | Value |
|-----------|-------|
| IC | DS3231SN |
| Accuracy | ±2ppm TCXO |
| Interface | I2C |
| Backup | CR2032 coin cell (holds time through main battery failure) |
| Alarm | DS3231 INT/SQW pin connected to ESP32-C3 deep sleep wakeup GPIO |
| Current (main) | ~200µA active, ~110µA standby |
| Current (coin cell) | ~3µA (backup mode only) |

The DS3231 remains powered from the always-on battery rail at all times. It is the primary wake source — its alarm output triggers ESP32-C3 wakeup rather than relying on the ESP32 internal timer, ensuring accurate timing independent of sleep drift.

### 3.6 Power System

#### 3.6.1 Battery

- 2× 18650 Li-ion cells in parallel (3.7V nominal, 6000mAh combined at 2× 3000mAh cells)
- Cells in 18650 PCB holders with spring contacts (user-replaceable without tools after lid removal)
- Parallel configuration chosen: capacity increase with no voltage conversion, simple protection circuit

#### 3.6.2 Battery Protection

| Function | IC | Notes |
|----------|----|-------|
| Overcharge / over-discharge / overcurrent | DW01A | Monitors cell voltage and current |
| Protection FETs | FS8205A (dual NMOS) | Controlled by DW01A |
| Reverse polarity | P-channel MOSFET (Si2301) | Gate to source via 100kΩ; conducts only with correct polarity |

Over-discharge threshold: 2.88V (DW01A default). Over-charge: 4.28V.

#### 3.6.3 Solar Charging

| Parameter | Value |
|-----------|-------|
| Charge IC | CN3791 |
| Algorithm | MPPT-like input voltage tracking (maximises power harvest from panel) |
| Charge current | Up to 1A (set by RPROG resistor; 500mA recommended for 18650 cycle life) |
| Panel input | 5V, 1–2W, JST-PH 2-pin (IP68 rated) |
| Backfeed protection | SS34 Schottky between panel and charge IC input |
| Charge indicator | CHRG/DONE LEDs (test point accessible, not populated in production) |

**Energy harvest estimate:** 1W solar panel × 5 peak sun hours/day (rural Australia) = 5Wh/day harvested. Node consumption ≈ 1.5mAh/day × 3.7V ≈ 5.6mWh/day. Surplus energy is approximately 1000×. Even a partially shaded panel provides months of runtime surplus.

#### 3.6.4 3.3V Regulator

| Parameter | Value |
|-----------|-------|
| IC | TPS63021DSJT (buck-boost) |
| Input range | 1.8V–5.5V |
| Output | 3.3V, up to 1A |
| Efficiency | Up to 95% |
| Quiescent current | ~50µA |

A buck-boost is required because the Li-ion battery voltage (3.0–4.2V) spans the target output (3.3V). A simple LDO fails when battery voltage drops below 3.3V + dropout. The TPS63021 maintains regulated 3.3V across the full battery range, maximising usable capacity and preventing brownouts during transmission peaks.

**Sensor power rail:** A secondary switched rail powers all sensors (SHT40, SHT31, pressure transducer, SX1276 RF switch if used) via a P-channel MOSFET (Si2301) controlled by GPIO3. This rail is cut during deep sleep, eliminating sensor quiescent currents and I2C pull-up paths.

#### 3.6.5 Power Budget

| State | Current | Duration | Energy/cycle |
|-------|---------|----------|--------------|
| Deep sleep (ESP32-C3 + DS3231) | ~15µA | ~14m 57s | ~223µAh |
| Wake + sensor sampling | ~30mA | ~1s | ~8µAh |
| LoRa TX (+17dBm) | ~120mA | ~1s | ~33µAh |
| LoRa RX (mesh ACK wait) | ~12mA | ~1s | ~3µAh |
| Total per 15-min cycle | — | — | ~267µAh |
| **Daily total** | — | — | **~1.07mAh** |

6000mAh reserve ÷ 1.07mAh/day = **>5,600 days theoretical without solar.** Practical reserve accounting for temperature derating and Peukert effect: **>100 days** at worst case.

### 3.7 Hardware Watchdog

| Parameter | Value |
|-----------|-------|
| IC | TPL5110DDCT |
| Function | Hard power-cycle of entire system if not kicked within timeout |
| Timeout | Set by resistor on DELAY pin; 64s during normal operation |
| Kick signal | GPIO2 → DONE pin; one pulse per wake cycle after successful transmission |
| Recovery | Powers system back up after 50ms off; firmware resumes from cold boot |

The watchdog ensures autonomous recovery from firmware lockups in unattended deployments. A node that hangs (e.g. SX1276 lock-up during TX) will be fully power-cycled within 64 seconds and resume normal operation.

### 3.8 USB / Debug Interface

| Parameter | Value |
|-----------|-------|
| Connector | IP68 waterproof USB-C panel mount |
| USB-UART | CH340C (auto-reset via RTS/DTR) |
| Functions | Firmware flash, serial debug, 5V input power, OTA trigger |
| Protection | ESD on USB data lines (PRTR5V0U2X) |

When USB is connected, the CH340C enumerates as a serial port. The auto-reset circuit (capacitor + transistor on EN and GPIO9/BOOT) allows esptool to flash without manual button pressing — important for field firmware updates.

### 3.9 Node Identification and Provisioning

- Unique 32-bit hardware ID derived from ESP32-C3 MAC address, stored in flash during first boot
- Human-readable name (e.g. "Home Trough", "Back Dam") configured via BLE advertisement or USB-C serial during installation
- Configuration stored in NVS (Non-Volatile Storage)
- Provisioning tool: Python script (cross-platform) or simple BLE app

### 3.10 PCB Design Guidelines

- **Layers:** 4-layer (Top signal / GND / 3.3V power / Bottom signal)
- **RF section:** 50Ω CPWG trace from SX1276 to U.FL; no vias; copper pour keepout 2mm either side
- **Analog section:** ADC inputs and pressure sensor circuitry on separate analog ground island; single ground connection point to digital ground at star point near decoupling caps
- **Decoupling:** 100nF + 10µF ceramic at every IC power pin; bulk 47µF on 3.3V rail
- **Test points:** All critical nets (3.3V, VBAT, VSOLAR, DONE, SDA, SCL, SPI lines, ADC in)
- **Multi-colour silkscreen:** Top copper silkscreen includes Billabong Sentinel logo artwork and a stylised water-level graphic to identify nodes in the field. JLCPCB multi-colour silkscreen used for blue/white aesthetic matching the project's water theme.
- **Fiducials:** Minimum 3 fiducials for JLCPCB SMT assembly
- **DFM:** No via-in-pad on fine-pitch components; minimum clearances per JLCPCB DFM rules; LCSC part numbers on all components

### 3.11 I/O Summary

| Signal | GPIO | Type | Notes |
|--------|------|------|-------|
| SX1276 SCK | GPIO0 | SPI CLK | |
| SX1276 MISO | GPIO1 | SPI MISO | |
| SX1276 MOSI | GPIO7 | SPI MOSI | |
| SX1276 CS | GPIO8 | SPI CS | Active low |
| SX1276 RST | GPIO9 | Digital out | Active low reset |
| SX1276 DIO0 | GPIO4 | Digital in | TX done / RX done interrupt |
| SDA | GPIO5 | I2C | 4.7kΩ pull-up to switched rail |
| SCL | GPIO6 | I2C | 4.7kΩ pull-up to switched rail |
| Pressure ADC | GPIO2 (ADC1_2) | Analog in | Protected, filtered, divided |
| Sensor rail EN | GPIO3 | Digital out | High = sensors powered |
| Watchdog DONE | GPIO10 | Digital out | Pulse to kick TPL5110 |
| RTC wakeup | GPIO2 | Deep sleep wakeup | DS3231 INT output |
| LED (DNP) | GPIO18 | Digital out | Debug only |
| USB-C VBUS detect | GPIO20 | Digital in | Detects USB connection |

*Note: GPIO2 serves dual purpose (ADC + RTC wakeup). Wakeup occurs before ADC sampling; no conflict.*

---

## 4. Billabong Sentinel Gateway

### 4.1 Functional Requirements

- Receive LoRa mesh transmissions from all nodes
- Store up to 90 days of readings per node (circular buffer in SPIFFS/LittleFS)
- Serve a responsive local web dashboard (no internet required)
- Calculate and display water consumption rate per node
- Evaluate and dispatch alerts via MQTT
- Support OTA firmware updates for gateway (WiFi) and nodes (LoRa packet relay)
- Optional: e-ink display for at-a-glance status without opening a browser

### 4.2 Hardware Differences from Node

| Item | Node | Gateway |
|------|------|---------|
| MCU | ESP32-C3-MINI-1 | ESP32-S3-WROOM-1 (more RAM for web serving) |
| Power | Solar + 18650 | USB-C 5V or 12V DC input |
| Battery | 2× 18650 | None (mains powered) |
| Solar charger | CN3791 | DNP |
| Watchdog | TPL5110 (64s) | TPL5110 (longer timeout) |
| Display | None | Optional 2.9" e-ink (SPI) |
| Config | JP1 solder bridge | JP1 open (gateway mode) |

**Shared PCB:** JP1 solder bridge selects firmware behaviour:
- JP1 bridged = Node mode (solar/battery power path active, deep sleep enabled)
- JP1 open = Gateway mode (always-on, web server active, SPIFFS logging)

Both variants populated from the same JLCPCB assembly order; DNP components noted in BOM.

### 4.3 Web Dashboard

- Framework: ESPAsyncWebServer + LittleFS on ESP32-S3
- Frontend: single-page HTML/CSS/JS (no framework dependencies; served from flash)
- Data API: JSON endpoint `/api/nodes` returning latest and historical readings
- Charts: lightweight Chart.js for trend graphs
- Mobile-responsive layout; readable on phone screen in bright sunlight (high contrast)
- Auto-refreshes every 60 seconds; WebSocket push for real-time alerts

**Dashboard views:**
1. **Overview** — card per node showing: name, current level (mm + %), trend arrow, last seen, battery status
2. **Node detail** — 24h and 7-day level charts, consumption rate, alert history, signal strength
3. **System** — all node RSSI map, gateway uptime, storage usage
4. **Alerts** — configurable thresholds per node; alert log

### 4.4 MQTT Integration

Topics published (prefix configurable):
```
billabong-sentinel/{node_name}/water_mm
billabong-sentinel/{node_name}/water_pct
billabong-sentinel/{node_name}/temp_c
billabong-sentinel/{node_name}/humidity_pct
billabong-sentinel/{node_name}/battery_mv
billabong-sentinel/{node_name}/status          # "ok" | "low_water" | "no_contact" | "seal_breach"
billabong-sentinel/gateway/status
```

Compatible with Home Assistant MQTT auto-discovery.

---

## 5. Firmware Architecture

### 5.1 Node Firmware

**Platform:** ESP-IDF (native, no Arduino layer)
**Build system:** CMake + `idf.py`
**Target SDK:** ESP-IDF v5.x

**Components/libraries:**
- RadioLib (ESP-IDF HAL — SX1276 via `spi_master` driver)
- `driver/i2c_master` (ESP-IDF 5.x I2C driver for SHT40, SHT31, DS3231)
- Custom SHT40, SHT31, DS3231 thin drivers (I2C register reads, no Arduino deps)
- `nvs_flash` (NVS — config, calibration, node ID storage)
- `esp_sleep` (deep sleep, wakeup config)
- `esp_https_ota` (WiFi OTA)
- `esp_bt` / NimBLE (BLE provisioning)

**Build artifacts:** `node.bin` flashed via `idf.py flash` or `esptool.py`

**State machine:**
```
COLD_BOOT → PROVISION_CHECK → SAMPLE → MESH_TX → SLEEP
                  ↓ (if unconfigured)
            PROVISIONING (BLE/serial config)
```

**Deep sleep entry sequence:**
1. Assert sensor rail GPIO low via `gpio_set_level()`
2. Hold SPI/I2C pins as input-only via `gpio_set_direction(GPIO_MODE_INPUT)`
3. Configure DS3231 alarm; set wakeup source via `esp_sleep_enable_ext0_wakeup()`
4. Kick watchdog (DONE pulse)
5. Call `esp_deep_sleep_start()`; wake on DS3231 INT or USB-VBUS edge

### 5.2 Gateway Firmware

**Platform:** ESP-IDF (native, no Arduino layer)
**Build system:** CMake + `idf.py`
**Target SDK:** ESP-IDF v5.x

**Components/libraries:**
- RadioLib (ESP-IDF HAL — mesh receive + relay)
- `esp_http_server` (ESP-IDF native HTTP server for dashboard + JSON API)
- `esp_mqtt` (ESP-IDF native MQTT client)
- `cJSON` (ESP-IDF built-in JSON serialisation)
- `wear_levelling` + `spiffs` or LittleFS component (data storage + web assets in flash partition)
- `esp_https_ota` (gateway OTA)

**FreeRTOS tasks (ESP-IDF native):**
- LoRa receive task (Core 0): listens for mesh packets, routes ACKs
- Web server task (Core 1): serves dashboard, handles API requests
- Alert evaluation task: checks thresholds every 60s, publishes MQTT
- OTA task: `esp_https_ota` for gateway; LoRa OTA relay for nodes (chunked)

### 5.3 OTA Strategy

| Target | Method |
|--------|--------|
| Gateway | `esp_https_ota` over WiFi (ESP-IDF native) |
| Node (nearby) | USB-C serial flash via esptool |
| Node (remote) | LoRa OTA: gateway splits firmware binary into 128-byte chunks, transmits to node at maintenance window; node writes to OTA partition and reboots |

LoRa OTA is slow (~1 hour for 512KB at 1% duty cycle) but enables remote updates for nodes at km range without physical access.

---

## 6. Enclosure

### 6.1 Node Enclosure

| Parameter | Value |
|-----------|-------|
| Material | ASA (UV and heat resistant, rated to 95°C) |
| Manufacture | JLCPCB 3D printing (MJF or SLA) |
| Target IP rating | IP68 |
| Dimensions | ~120 × 80 × 50mm (to be finalised in CAD) |
| Lid seal | Nitrile O-ring in CNC-machined groove; lid secured with 4× M3 stainless screws |
| Cable entry | IP68 cable gland (PG9), sized to 8mm OD sensor cable |
| USB-C port | IP68 waterproof panel mount USB-C; tethered rubber cap |
| Solar connector | IP68 rated JST-PH 2-pin |
| Antenna | SMA bulkhead, stainless, sealed with silicone o-ring |
| Mounting | Two external lugs for M8 stainless U-bolt (trough rail or star picket mount) |
| Battery access | Lid removal exposes 18650 holders — no tools required |
| Vent | Gore-Tex vent plug (M12) — equalises pressure, excludes water and dust |
| Desiccant | Moulded 30cc silica gel sachet holder in base; sachet replaced at battery service interval |
| Stevenson screen | Small louvred housing (25mm dia) for external SHT40; prints as one piece; clips onto sensor cable |

### 6.2 Gateway Enclosure

- Standard ABS IP54 DIN rail enclosure (sourced off-shelf) — gateway lives in a farm office or shed
- PCB mounts to standard DIN rail bracket
- USB-C power input and RJ45 (optional) via panel knockouts

### 6.3 IP68 Installation Requirements

These steps are mandatory for IP68 integrity and must be followed during field installation:

1. **Drip loop** — sensor cable must form a downward U-loop before entering the cable gland. Water running down the cable drips off the bottom of the loop and cannot wick into the gland.
2. **Cable gland torque** — tighten to manufacturer specification (typically 2.5Nm for PG9). Hand-tight is insufficient.
3. **O-ring inspection** — before each lid reinstallation, inspect the nitrile O-ring. Replace if flattened, cracked, or shows UV degradation. Spare O-rings included with each unit.
4. **USB-C cap** — tethered cap must be fully seated and retaining ring finger-tightened when port not in use.
5. **Vent plug** — must not be painted over, obscured by silicone sealant, or blocked.
6. **SMA seal** — confirm antenna SMA nut is tight and O-ring is seated.

---

## 7. Open-Source Documentation Plan

Comprehensive documentation is central to the OSHWLab Stars submission. All of the following will be published before the submission deadline:

### 7.1 OSHWLab Project Page

- Full project description with photographs of assembled hardware and field deployment
- EasyEDA Pro schematic (publicly viewable and forkable)
- PCB layout (public)
- 3D enclosure STL/STEP files
- BOM with all LCSC part numbers resolved
- Revision history and design decisions documented in project log

### 7.2 GitHub Repository

```
billabong-sentinel/
├── firmware/
│   ├── node/               # ESP-IDF project (CMakeLists.txt, sdkconfig.defaults, main/, components/)
│   └── gateway/            # ESP-IDF project (CMakeLists.txt, sdkconfig.defaults, main/, components/)
├── hardware/
│   ├── easyeda/            # Exported EasyEDA Pro JSON (schematic + PCB)
│   └── enclosure/          # STL and STEP files for 3D printing
├── docs/
│   ├── assembly-guide.md   # Step-by-step with photos
│   ├── field-install.md    # IP68 requirements, sensor mounting, commissioning
│   ├── calibration.md      # Pressure transducer two-point calibration procedure
│   ├── troubleshooting.md  # Common failure modes and diagnosis
│   └── bom.md              # Full BOM with sources and approximate costs
└── README.md
```

### 7.3 Assembly Guide

Step-by-step guide covering:
- PCB inspection and test point checks before assembly
- 18650 cell installation and polarity verification
- Sensor cable installation through cable gland
- External SHT40 Stevenson screen mounting
- Antenna installation and U.FL to SMA connection
- Firmware flash procedure via USB-C
- Provisioning (setting node name, gateway address, thresholds)
- IP68 lid sealing procedure

### 7.4 YouTube / Community Content

Build log videos covering design decisions, PCB assembly, field deployment, and dashboard demonstration. These also contribute to the Public Influence component of the competition score.

---

## 8. Regulatory & Compliance

| Requirement | Detail |
|-------------|--------|
| LoRa frequency | 915MHz (AU915 band) |
| Licence | ACMA Radiocommunications (Low Interference Potential Devices) Class Licence 2015 |
| Max EIRP | ≤30dBm (firmware limited to +17dBm TX + 2dBi antenna = 19dBm EIRP; well within limit) |
| Duty cycle | Firmware-enforced; 15-min sample interval provides <0.1% duty cycle |
| PCB standard | IPC-2221B |
| RoHS | All components sourced via LCSC — RoHS compliant |
| Battery safety | DW01A + FS8205 protection meets UN 38.3 transport requirements |
| Hardware licence | CERN-OHL-W v2 (weakly reciprocal — modifications must be shared, end products need not be) |
| Firmware licence | MIT |
| Documentation licence | CC BY 4.0 |

---

## 9. Environmental Sustainability

| Dimension | Approach |
|-----------|----------|
| Energy source | Solar primary; 18650 battery buffer; no mains power at node |
| Standby consumption | ~15µA — comparable to a coin cell slowly discharging |
| Component lifetime | 18650 cells rated >500 cycles; solar panel >20 year life; IP68 enclosure protects PCB from corrosion |
| Repairability | 18650 cells user-replaceable without tools; sensor cable field-replaceable via cable gland; O-rings a consumable spares item |
| Vs. cellular alternative | Cellular water monitors require always-on modems (~50mA); Billabong Sentinel nodes use ~1000× less power |
| PCB longevity | Conformal coating on assembled PCB recommended for coastal/humid deployments |
| Enclosure material | ASA chosen over ABS for UV stability — no UV degradation cracking over 10+ year outdoor life |
| End of life | PCB is RoHS compliant; 18650 cells recycled via battery recycling programs |

---

## 10. Bill of Materials — Node (Key Components)

| Ref | Component | Part | LCSC # |
|-----|-----------|------|--------|
| U1 | MCU | ESP32-C3-MINI-1 | C2934569 |
| U2 | LoRa | Ra-02 SX1276 module (Ai-Thinker) | C2833538 |
| U3 | Solar charger | CN3791 | C116857 |
| U4 | Battery protection | DW01A | C351116 |
| U5 | Battery protection FET | FS8205A | C32254 |
| U6 | Watchdog | TPL5110DDCT | C479270 |
| U7 | RTC | DS3231SN# | C255499 |
| U8 | Humidity internal | SHT31-DIS-B | C296787 |
| U9 | Humidity external | SHT40-AD1B | C2757512 |
| U10 | 3.3V regulator | TPS63021DSJT | C116360 |
| U11 | USB-UART | CH340C | C84681 |
| U12 | Sensor rail switch | Si2301CDS (P-MOSFET) | C10487 |
| D1 | Schottky solar | SS34 | C8678 |
| D2 | TVS sensor input | PRTR5V0U2X | C12333 |
| D3 | TVS LoRa RF | PRTR5V0U2X | C12333 |
| D4 | TVS USB | PRTR5V0U2X | C12333 |
| BT1–2 | 18650 holders | Keystone 1042P | C2681692 |
| J1 | USB-C panel mount | IP68 waterproof USB-C | TBD (Aliexpress) |
| J2 | Solar input | IP68 JST-PH 2-pin | TBD |
| J3 | Pressure sensor | IP68 cable gland PG9 | TBD |
| J4 | Antenna | U.FL SMD | C388369 |
| J5 | External SMA | SMA bulkhead IP67 | TBD |
| BT3 | RTC backup | CR2032 holder (SMD) | C70376 |

*LCSC numbers marked TBD to be confirmed during schematic capture. All other LCSC numbers verified.*

### 10.1 Externally Sourced Items (Not via JLCPCB)

| Item | Source | Notes |
|------|--------|-------|
| 2× 18650 cells (3000mAh) | Local electronics or Aliexpress | SAMSUNG 30Q or equivalent |
| 5V 1–2W solar panel | Aliexpress | 115×115mm typical |
| Submersible pressure transducer | Aliexpress | 0–5m, 0.5–4.5V, vented cable |

### 10.2 Target Cost Per Node (AUD, estimated)

| Item | Cost AUD |
|------|----------|
| PCB + PCBA (JLCPCB, 5-unit run) | ~$45 |
| Enclosure (JLCPCB 3D print) | ~$20 |
| 18650 cells × 2 | ~$15 |
| Solar panel | ~$10 |
| Pressure transducer | ~$20 |
| Antenna + cable | ~$8 |
| Misc hardware (screws, cable gland, O-rings) | ~$7 |
| **Total** | **~$125 AUD** |

*Target: reduce to <$100 AUD with optimised BOM and 10+ unit volumes.*

---

## 11. Open Questions

- Final external SHT40 mini-enclosure dimensions (pending PCB layout confirmation)
- Pressure transducer connector at PCB (terminal block vs JST-XH)
- Gateway e-ink display inclusion in v1.0 vs v1.1
- BLE provisioning app — native or web BLE?

---

## 12. Revision History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | Feb 2026 | Initial draft |
| 0.2 | Feb 2026 | OSHWLab Stars entry revision: resolved TBD components, added consumption analytics, alert system, firmware architecture, dashboard spec, documentation plan, sustainability section, PCB guidelines, BOM with LCSC numbers |

---

*Billabong Sentinel is an open source hardware project. Designed in EasyEDA Pro. Manufactured via JLCPCB. Built for the bush.*
