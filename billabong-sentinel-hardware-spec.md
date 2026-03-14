# Billabong Sentinel: Rural Water Level Monitoring System
## Hardware Specification v0.2

**Project:** Billabong Sentinel
**Author:** Russell Thomas
**Date:** February 2026
**License:** CERN-OHL-S v2 (hardware) · MIT (firmware) · CC BY 4.0 (documentation)
**Status:** Draft, active schematic capture reconciliation for OSHWLab Stars 2026

---

## 1. Overview

### 1.1 Background

Rural Australian farmers managing large properties often have no practical way to monitor stock water remotely. Troughs and tanks can run dry without warning, and a dry trough on a remote paddock may go unnoticed for days. Commercial cellular-based water level monitors cost $500–$1,500 per unit plus monthly subscription fees, and cellular coverage is absent across much of the land where these devices would be used.

Billabong Sentinel is an open-source, solar-powered, LoRa mesh water monitoring network. A farmer installs a node at each trough or dam and a single gateway at the homestead. The gateway serves a local web dashboard showing water levels across the property, with no subscription fees, no cellular dependency, and no ongoing cost.

### 1.2 System Summary

Billabong Sentinel consists of two open-source hardware designs:

- **Billabong Sentinel Node:** a weatherproof, solar-powered sensor unit deployed at each water point
- **Billabong Sentinel Gateway:** a base station that aggregates all node data and serves a local web dashboard

Rev A prioritises the node hardware first. The gateway is planned as a separate always-on board rather than a solder-bridge variant of the node. All hardware is designed in EasyEDA Pro, manufactured via JLCPCB, and fully documented for community reproduction.

### 1.3 Key Innovations

1. **Mesh networking, not star topology.** LoRa nodes relay packets for each other, extending range beyond line-of-sight and routing around obstacles. No repeater infrastructure required.
2. **Enclosure seal integrity monitoring.** An internal humidity sensor detects gasket degradation before water damage occurs, allowing seal failures to be caught before they cause damage to the electronics.
3. **Water consumption rate analytics.** The gateway calculates water consumption rate (L/hour) from successive pressure readings, enabling early detection of leaks, trough overflow, and livestock behaviour anomalies.
4. **Node-first architecture.** Rev A focuses on getting the low-power field node correct before a separate gateway board is captured, reducing architectural risk.
5. **Low-power architecture.** Deep sleep, switched sensor rails, and a constrained analog front end keep the node simple and power-aware.
6. **Sub-$125 AUD per node.** Target landed cost is around AUD $125 per node in low volume, with room to reduce cost at higher quantities.

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

1. Node wakes from ESP32-C3 deep sleep timer (15-minute default, configurable)
2. Sensors powered up via switched rails
3. Pressure transducer, SHT40, and SHT31 sampled
4. Packet assembled: `{node_id, seq, water_mm, temp_c, humidity_pct, internal_temp_c, internal_humidity_pct, battery_mv, solar_mv, rssi_last}`
5. The E22-900M22S transmits via mesh route to gateway
6. All peripherals powered down; ESP32-C3 returns to deep sleep
7. Gateway timestamps packet reception, stores it in a circular buffer (planned LittleFS storage), and updates the dashboard

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
| LoRa | External E22-900M22S (SX1262-based) via SPI |
| Deep sleep current | ~5µA |
| Supply voltage | 3.3V |

WiFi can be used for maintenance or provisioning assistance when USB is not practical. BLE is used for initial node provisioning (set node name, sample interval, thresholds) via mobile app or serial terminal.

**GPIO management during deep sleep:** All GPIO pin states are explicitly configured before entering deep sleep. Sensor power rail GPIO is driven low. SPI and I2C pins are set to input (no drive) to prevent parasitic current paths.

### 3.2 LoRa Radio

| Parameter | Value |
|-----------|-------|
| Module | EBYTE E22-900M22S |
| Frequency | 915MHz (AU915) |
| Interface | SPI + NSS / NRST / BUSY / DIO1 plus TXEN and RXEN default-state controls |
| Max TX power | ~22dBm module capability, firmware-limited as needed for compliance and power budget |
| Sensitivity | ~-147dBm |
| Sleep current | ~0.2µA |
| Antenna | U.FL connector, external via SMA bulkhead |
| RF ESD | to be confirmed during RF/connector review |
| Impedance | 50Ω coplanar waveguide on antenna trace |

**PCB note:** RF trace from the E22-900M22S antenna pin to U.FL should be routed as 50Ω CPWG on the top copper layer with ground plane beneath. No vias on the RF trace. Keepout area under the antenna connection. Ground stitching vias around the RF section perimeter.

### 3.3 Water Level Sensor

| Parameter | Value |
|-----------|-------|
| Type | Submersible pressure transducer, analog 0.5–4.5V |
| Range | 0–5m water column |
| Output | 0.5–4.5V analog |
| Cable | Vented atmospheric reference tube, 5–10m |
| Resolution | ~1.2mm (12-bit ADC, 5m range) |
| Calibration | Two-point calibration (dry = 0mm, known depth) stored in node flash |

**Input protection chain (in order):**
1. 1kΩ series resistor (limits fault current)
2. TVS diode to GND sized for the long outdoor cable run (the current working netlist uses `SMF5.0A`; final suitability still needs signoff)
3. 10nF + 100Ω RC low-pass filter (rejects RF and switching noise)
4. 2:1 resistor divider (100kΩ / 100kΩ), maps 0–4.5V sensor output to 0–2.25V, within ESP32-C3 ADC input range
5. ADC input on ESP32-C3 (12-bit, 0–2.5V attenuation mode)

**Sensor power:** The pressure transducer is supplied from a dedicated switched 5V excitation rail, active only during sampling. The ESP32 ADC sees the transducer output through the divider and filter chain above. The 4–20mA option is removed from rev A to keep the analog front end simple.

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
| Interface | I2C (address 0x45, shared bus) |
| Location | PCB-mounted inside enclosure |
| Alert threshold | >80% RH sustained for 30 minutes → seal breach alert |

**I2C bus:** Single I2C bus. SHT40 external and SHT31 internal use addresses 0x44 and 0x45 respectively. Pull-ups are intended to sit on a switched sensor-side rail so they disappear during deep sleep and do not create a leakage path. The exact switched 3.3V implementation still needs to be closed in the schematic.

### 3.5 Wake Strategy

| Parameter | Value |
|-----------|-------|
| Primary wake source | ESP32-C3 internal deep sleep timer |
| Default interval | 15 minutes |
| Backup timekeeping | Not populated in rev A |
| Timestamp authority | Gateway receive timestamp |

Rev A removes the external RTC from the node to reduce quiescent current, simplify the schematic, and avoid redundant wake circuitry. If field testing later shows a hard requirement for independent wall-clock timestamps at the node, a low-current RTC can be revisited in a later revision.

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
| Reverse polarity | P-channel MOSFET (Si2301) | Only valid for a keyed external pack connection; it does not protect against an incorrectly inserted loose cell in a parallel holder |

Over-discharge threshold: 2.88V (DW01A default). Over-charge: 4.28V.

#### 3.6.3 Input Power and Battery Charging

| Parameter | Value |
|-----------|-------|
| Charge IC | MCP73871-2CAI/ML |
| Algorithm | Linear USB / external-input charging with power-path management and VPCC input control |
| Charge current | Up to 1A programmable; rev A target 500mA max charge current |
| Inputs | USB-C 5V and 5.5V nominal 1W solar panel |
| Input steering | USB and solar diode-ORed into charger input; USB presence drives MCP73871 into USB mode |
| Charge indicator | CHRG/DONE LEDs (test point accessible, not populated in production) |

The MCP73871 is intentionally a simpler rev A choice than a true MPPT charger. This board uses the part for two reasons: it supports both USB and field solar charging, and its VPCC control can back off battery charge current before collapsing a weak source. For the 5.5V, ~170mA Seeed 1W panel, that is the right tradeoff for a first board.

**Energy harvest estimate:** a 1W class panel with roughly 5 peak sun hours/day still provides far more daily energy than the revised rev A node budget of roughly 13–22mWh/day, even after linear-charger and regulator losses. The design goal is source stability and field robustness, not theoretical maximum harvest.

#### 3.6.4 3.3V Regulator

| Parameter | Value |
|-----------|-------|
| IC | TPS63021DSJT (buck-boost) |
| Input range | 1.8V–5.5V |
| Output | 3.3V, up to 1A |
| Efficiency | Up to 95% |
| Quiescent current | ~25µA typical |

A buck-boost is required because the Li-ion battery voltage (3.0–4.2V) spans the target output (3.3V). A simple LDO fails when battery voltage drops below 3.3V + dropout. The TPS63021 maintains regulated 3.3V across the full battery range, covering the gap where an LDO would drop out and ensuring stable operation during transmit current peaks.

**Sensor power rails:** A switched 3.3V rail powers the digital sensors and I2C pull-ups. A separate switched 5V excitation rail powers the pressure transducer only during sampling. Both rails are off during deep sleep.

#### 3.6.5 Power Budget

| State | Current | Duration | Energy/cycle |
|-------|---------|----------|--------------|
| Deep sleep (ESP32-C3 + regulator leakage) | ~35–45µA | ~14m 57s | ~9–11µAh |
| Wake + sensor sampling | ~35–50mA | ~1s | ~10–14µAh |
| LoRa TX (+17dBm) | ~120mA | ~0.5–1.0s | ~17–33µAh |
| LoRa RX / ACK wait | ~12mA | ~0.5s | ~2µAh |
| Total per 15-min cycle | — | — | ~38–60µAh |
| **Daily total** | — | — | **~3.6–5.8mAh** |

6000mAh reserve ÷ 5.8mAh/day still implies well over 100 days of reserve without solar in simple theoretical terms. Actual field runtime must be validated on rev A hardware, but the power story remains comfortably compatible with a 1–2W panel in rural Australia.

### 3.7 Reliability Strategy

| Parameter | Value |
|-----------|-------|
| Rev A approach | ESP-IDF task watchdogs + brownout/reset supervision |
| External watchdog | Not populated in rev A |
| Future option | Add an external watchdog in a later revision if field data justifies it |

Rev A removes the external watchdog from the node to keep the power architecture simple and avoid overlapping power-control schemes. Reliability will initially rely on conservative firmware design, ESP-IDF watchdog facilities, and field test validation.

### 3.8 USB / Debug Interface

| Parameter | Value |
|-----------|-------|
| Connector | USB-C receptacle on the node PCB, with waterproof enclosure strategy to be closed mechanically |
| USB data path | Native ESP32-C3 USB |
| Functions | Firmware flash, serial debug, 5V input power, provisioning / recovery access |
| Protection | USB data-line ESD device on the connector side of the series resistors |

When USB is connected, the ESP32-C3 native USB path is the intended programming and debug interface. The final schematic still needs the USB-C receptacle, CC pull-downs, ESD device, and any boot/recovery support to be captured and reviewed against Espressif guidance.

### 3.9 Node Identification and Provisioning

- Unique 32-bit hardware ID derived from ESP32-C3 MAC address, stored in flash during first boot
- Human-readable name (e.g. "Home Trough", "Back Dam") configured via BLE advertisement or USB-C serial during installation
- Configuration stored in NVS (Non-Volatile Storage)
- Provisioning tool: Python script (cross-platform) or simple BLE app

### 3.10 PCB Design Guidelines

- **Layers:** 4-layer (Top signal / GND / 3.3V power / Bottom signal)
- **RF section:** 50Ω CPWG trace from E22-900M22S to U.FL; no vias; copper pour keepout 2mm either side
- **Analog section:** Keep the pressure sensor divider and filter compact and away from the RF section, but do not split the ground plane unnecessarily on this class of board
- **Decoupling:** 100nF + 10µF ceramic at every IC power pin; bulk 47µF on 3.3V rail
- **Test points:** All critical nets (3.3V, VBAT, VSOLAR, switched 5V, SDA, SCL, SPI lines, ADC in)
- **Multi-colour silkscreen:** Top copper silkscreen includes Billabong Sentinel logo artwork and a stylised water-level graphic to identify nodes in the field. JLCPCB multi-colour silkscreen used for blue/white aesthetic matching the project's water theme.
- **Fiducials:** Minimum 3 fiducials for JLCPCB SMT assembly
- **DFM:** No via-in-pad on fine-pitch components; minimum clearances per JLCPCB DFM rules; LCSC part numbers on all components

### 3.11 GPIO Planning Constraints

- Avoid ESP32-C3 strapping pins for LoRa reset, rail enable, or other outputs that could disturb boot.
- Reserve one ADC-capable GPIO exclusively for the pressure transducer input.
- Reserve GPIOs for E22-900M22S `BUSY`, `DIO1`, `NRST`, `NSS`, `TXEN`, and `RXEN` without disturbing boot.
- Reserve one GPIO for the switched 5V excitation rail enable, and only add a separate switched 3.3V rail enable if the final sleep-leakage review justifies it.
- Reserve a GPIO or comparator input for USB-VBUS detection if that feature survives schematic review.
- Final GPIO allocation will be frozen during schematic capture against the ESP32-C3-MINI-1 datasheet and boot-strapping requirements.

---

## 4. Billabong Sentinel Gateway

### 4.1 Functional Requirements

- Receive LoRa mesh transmissions from all nodes
- Store up to 90 days of readings per node (circular buffer in LittleFS)
- Serve a responsive local web dashboard (no internet required)
- Calculate and display water consumption rate per node
- Evaluate and dispatch alerts via MQTT
- Support OTA firmware updates for gateway (WiFi) and nodes (LoRa packet relay)
- Optional: e-ink display for at-a-glance status without opening a browser

### 4.2 Hardware Differences from Node

| Item | Node | Gateway |
|------|------|---------|
| MCU | ESP32-C3-MINI-1 | ESP32-S3-WROOM-1 (or equivalent ESP32-S3 module) |
| Power | Solar + 18650 | USB-C 5V or 12V DC input |
| Battery | 2× 18650 | None (mains powered) |
| Solar / USB charger | MCP73871 | Not required |
| External RTC | Not fitted | Not required |
| External watchdog | Not fitted | Not required in the first hardware pass |
| Display | None | Optional 2.9" e-ink (SPI) |
| Board strategy | Dedicated low-power node PCB | Separate always-on gateway PCB |

The gateway is intentionally split from the node in rev A. The always-on gateway and the ultra-low-power field node have different priorities, so forcing them onto one PCB is more likely to add compromise than value at this stage.

### 4.3 Web Dashboard

- Framework: `esp_http_server` + LittleFS on ESP32-S3
- Frontend: single-page HTML/CSS/JS (no framework dependencies; served from flash)
- Data API: JSON endpoint `/api/nodes` returning latest and historical readings
- Charts: lightweight Chart.js for trend graphs
- Mobile-responsive layout; readable on phone screen in bright sunlight (high contrast)
- Auto-refreshes every 60 seconds; WebSocket push for real-time alerts

**Dashboard views:**
1. **Overview:** card per node showing: name, current level (mm + %), trend arrow, last seen, battery status
2. **Node detail:** 24h and 7-day level charts, consumption rate, alert history, signal strength
3. **System:** all node RSSI map, gateway uptime, storage usage
4. **Alerts:** configurable thresholds per node; alert log

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
- RadioLib (ESP-IDF HAL, SX1262 / E22-900M22S via `spi_master` driver)
- `driver/i2c_master` (ESP-IDF 5.x I2C driver for SHT40 and SHT31)
- Custom SHT40 and SHT31 thin drivers (I2C register reads, no Arduino deps)
- `nvs_flash` (NVS: config, calibration, node ID storage)
- `esp_sleep` (deep sleep, wakeup config)
- `esp_https_ota` (WiFi OTA)
- `esp_bt` / NimBLE (BLE provisioning)

**Build artifacts:** `node.bin` flashed via `idf.py flash` or `esptool.py`

**State machine:**
```
COLD_BOOT → PROVISION_CHECK → SAMPLE → MESH_TX → SLEEP
                  ↓ (if unconfigured)
            PROVISIONING (BLE/native USB config)
```

**Deep sleep entry sequence:**
1. Assert sensor rail GPIO low via `gpio_set_level()`
2. Hold SPI/I2C pins as input-only via `gpio_set_direction(GPIO_MODE_INPUT)`
3. Configure timer wakeup via `esp_sleep_enable_timer_wakeup()`
4. Call `esp_deep_sleep_start()`
5. Wake on timer expiry or USB-VBUS edge

### 5.2 Gateway Firmware

**Platform:** ESP-IDF (native, no Arduino layer)
**Build system:** CMake + `idf.py`
**Target SDK:** ESP-IDF v5.x

**Components/libraries:**
- RadioLib (ESP-IDF HAL, mesh receive + relay)
- `esp_http_server` (ESP-IDF native HTTP server for dashboard + JSON API)
- `esp_mqtt` (ESP-IDF native MQTT client)
- `cJSON` (ESP-IDF built-in JSON serialisation)
- `wear_levelling` + LittleFS component (data storage + web assets in flash partition)
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
| Battery access | Lid removal exposes 18650 holders (no tools required) |
| Vent | Gore-Tex vent plug (M12); equalises pressure, excludes water and dust |
| Desiccant | Moulded 30cc silica gel sachet holder in base; sachet replaced at battery service interval |
| Stevenson screen | Small louvred housing (25mm dia) for external SHT40; prints as one piece; clips onto sensor cable |

### 6.2 Gateway Enclosure

- Standard ABS IP54 DIN rail enclosure (sourced off-shelf); intended for a farm office or shed
- PCB mounts to standard DIN rail bracket
- USB-C power input and RJ45 (optional) via panel knockouts

### 6.3 IP68 Installation Requirements

These steps are mandatory for IP68 integrity and must be followed during field installation:

1. **Drip loop:** sensor cable must form a downward U-loop before entering the cable gland. Water running down the cable drips off the bottom of the loop and cannot wick into the gland.
2. **Cable gland torque:** tighten to manufacturer specification (typically 2.5Nm for PG9). Hand-tight is insufficient.
3. **O-ring inspection:** before each lid reinstallation, inspect the nitrile O-ring. Replace if flattened, cracked, or shows UV degradation. Spare O-rings included with each unit.
4. **USB-C cap:** tethered cap must be fully seated and retaining ring finger-tightened when port not in use.
5. **Vent plug:** must not be painted over, obscured by silicone sealant, or blocked.
6. **SMA seal:** confirm antenna SMA nut is tight and O-ring is seated.

---

## 7. Open-Source Documentation Plan

The following will be published before the submission deadline:

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
| RoHS | All components sourced via LCSC; RoHS compliant |
| Battery safety | DW01A + FS8205 protection meets UN 38.3 transport requirements |
| Hardware licence | CERN-OHL-S v2 (strongly reciprocal hardware licence for shared design documentation and products) |
| Firmware licence | MIT |
| Documentation licence | CC BY 4.0 |

---

## 9. Environmental Sustainability

| Dimension | Approach |
|-----------|----------|
| Energy source | Solar primary; 18650 battery buffer; no mains power at node |
| Standby consumption | ~35–45µA target in rev A |
| Component lifetime | 18650 cells rated >500 cycles; solar panel >20 year life; IP68 enclosure protects PCB from corrosion |
| Repairability | 18650 cells user-replaceable without tools; sensor cable field-replaceable via cable gland; O-rings a consumable spares item |
| vs cellular | Cellular water monitors require always-on modems (~50mA); Billabong Sentinel nodes still use orders of magnitude less standby power |
| PCB longevity | Conformal coating on assembled PCB recommended for coastal/humid deployments |
| Enclosure material | ASA chosen over ABS for UV stability; no UV degradation cracking over 10+ year outdoor life |
| End of life | PCB is RoHS compliant; 18650 cells recycled via battery recycling programs |

---

## 10. Bill of Materials: Node (Key Components)

| Function | Current Direction | Part | LCSC # / Status |
|----------|-------------------|------|-----------------|
| MCU | locked | ESP32-C3-MINI-1-N4 | C2838502 |
| LoRa module | locked | E22-900M22S | C411293 |
| Solar / USB charger | locked | MCP73871-2CAI/ML | C185603 |
| Battery protection IC | locked | DW01A | C351410 |
| Battery protection FET | locked | FS8205A | C2830320 |
| 3.3V regulator | locked | TPS63021DSJR | C202140 |
| 5V excitation regulator | current direction | TPS613222ADBVR | C2071163 |
| 5V excitation high-side switch | current direction | AO3401A + 2N7002 gate drive | C15127 + C8545 |
| Sensor input TVS | current netlist part | SMF5.0A | C908213 |
| Solar input diode | locked | B5819W | C8598 |
| USB ideal-diode path | locked | AO3401A | C15127 |
| VIN_CHG bulk capacitor | locked | user-selected charger-input bulk capacitor | C3039902 |
| 2x 18650 holder | locked | MYOUNG BH-18650-B1BA007 | C6937126 |
| Pressure connector | current direction | XH-3A-P | C2979572 |
| USB data ESD | intended direction, not yet fully captured in latest netlist | USBLC6-2SC6 | C2827654 |
| USB-C receptacle | intended direction, not yet fully captured in latest netlist | TYPE-C-31-M-12 | C165948 |

This table reflects the current design direction rather than a release-ready BOM. The latest working netlist already captures the power tree, pressure front end, ESP32-C3, E22-900M22S, and 5V excitation path, but the USB-C/native-USB block and some sensor-side details still need closure before schematic signoff.

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

- Exact 5V excitation regulator for the chosen pressure transducer
- Final external SHT40 mini-enclosure dimensions (pending PCB layout confirmation)
- Pressure transducer connector at PCB (terminal block vs JST-XH)
- Gateway e-ink display inclusion in v1.0 vs v1.1
- BLE provisioning app: native or web BLE?

---

## 12. Revision History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | Feb 2026 | Initial draft |
| 0.2 | Feb 2026 | OSHWLab Stars entry revision: resolved TBD components, added consumption analytics, alert system, firmware architecture, dashboard spec, documentation plan, sustainability section, PCB guidelines, BOM with LCSC numbers |
| 0.3 | Mar 2026 | Architecture simplification pass: removed shared-PCB assumption, removed node RTC and external watchdog from rev A, fixed pressure-sensor direction, corrected power-budget assumptions, and split node/gateway hardware strategy |
| 0.4 | Mar 2026 | Active schematic-capture reconciliation: updated docs to current ESP32-C3, E22-900M22S, native USB, MCP73871, TPS63021, and TPS613222 design direction; flagged remaining USB and schematic-signoff gaps |

---

*Billabong Sentinel is an open source hardware project. Designed in EasyEDA Pro. Manufactured via JLCPCB. Built for the bush.*
