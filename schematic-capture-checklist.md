# Billabong Sentinel Schematic Capture Checklist

## Purpose

This document turns the rev A hardware spec into a practical schematic-capture plan for EasyEDA Pro.

Primary goal for rev A:

- build a simple, power-efficient, field-worthy node first
- defer avoidable complexity
- freeze each block against primary datasheets before moving to PCB

## Rev A Architecture Summary

- Dedicated low-power node PCB
- Separate gateway PCB later
- ESP32-C3-MINI-1 node
- EBYTE `E22-900M22S` LoRa module (`SX1262`-based)
- MCP73871 USB / solar charger
- TPS63021 3.3V buck-boost regulator
- 2x 18650 in parallel
- 0.5-4.5V analog pressure transducer
- Switched 5V excitation rail for pressure sensor
- ESP32-C3 deep sleep timer as wake source
- No external RTC in rev A
- No external watchdog in rev A

## Schematic Sheets

### Sheet 1: Power Input, Charging, and Battery

Include:

- USB power input net
- solar input connector
- 2x 18650 battery holder
- input source steering / reverse-current protection
- MCP73871 charger circuit
- battery protection stage
- dual 18650 parallel connection
- VIN_CHG bulk capacitor
- USB / solar / battery / system-power test points

Checklist:

- confirm MCP73871 charge-current resistor against the intended cells
- confirm MCP73871 VPCC divider against the Seeed panel operating range
- confirm source-selection behavior for USB-present vs solar-only cases
- confirm battery protection placement and current path
- confirm selected 2x18650 holder `C6937126` footprint, polarity marking, retention, and assembly orientation
- confirm selected VIN_CHG bulk capacitor `C3039902` capacitance, voltage rating, ESR, and package fit
- confirm reverse-polarity behavior
- confirm all charger-required passives from datasheet
- confirm connector polarity and silkscreen warnings

### Sheet 2: 3.3V System Rail and Switched Rails

Include:

- TPS63021 main 3.3V regulator
- enable pin behavior
- bulk and local decoupling
- switched 5V pressure-sensor excitation rail
- rail control nets from MCU

Checklist:

- confirm TPS63021 inductor and passives against datasheet
- confirm startup behavior from battery-only operation
- confirm sleep-state leakage paths
- confirm switched-rail default state at boot/reset
- confirm pressure-sensor rail can supply the intended transducer current
- confirm all critical rail test points

### Sheet 3: ESP32-C3 Core, Boot, USB, and Programming

Include:

- ESP32-C3-MINI-1 module
- EN / BOOT / reset support circuit
- native USB data path on ESP32-C3
- USB-C data/power/ESD
- VBUS detect path if retained
- local decoupling and module support parts

Checklist:

- confirm all mandatory ESP32-C3 support circuitry
- confirm boot-strapping pins are not abused
- confirm native USB wiring, reset behavior, and any boot/recovery access are boot-safe
- confirm USB ESD and connector pinout
- confirm exposed programming and debug access
- confirm any unused module pins are handled correctly

### Sheet 4: LoRa Radio and RF Interface

Include:

- E22-900M22S module interface
- SPI bus
- reset, BUSY, and DIO interrupt lines
- antenna connector path
- RF ESD where justified
- module-specific support and default-state controls

Checklist:

- confirm exact E22-900M22S support circuit from the module datasheet
- confirm required BUSY / DIO1 / reset handling in firmware
- confirm reset/default-state behavior at boot
- confirm RF connector choice and grounding approach
- confirm that schematic symbols/footprints match the chosen RF implementation

### Sheet 5: Pressure Sensor and Analog Front End

Include:

- pressure transducer connector
- input protection chain
- RC filter
- resistor divider
- ADC net to MCU

Checklist:

- confirm transducer supply voltage and output range from the actual chosen sensor
- confirm divider ratio leaves ADC headroom
- confirm ADC source impedance is acceptable for ESP32-C3 ADC sampling
- confirm TVS choice makes sense for a long outdoor cable
- confirm whether any extra filtering or transient protection is needed on external sensor lines

### Sheet 6: Connectors, Mechanical Interfaces, and Service Points

Include:

- solar connector
- pressure-sensor entry/connector strategy
- antenna connector chain
- USB-C panel-mount interconnect
- battery holder interconnect
- test points and field-service labels

Checklist:

- confirm connector families match enclosure and assembly plan
- confirm cable-entry assumptions match IP goals
- confirm mating direction and serviceability
- confirm test points are reachable after assembly
- confirm field polarity mistakes are hard to make

## Cross-Sheet Nets To Name Clearly

- `VBAT`
- `VSOLAR`
- `VCHG_OUT` or equivalent charger/battery node
- `+3V3`
- `+5V_SENSOR`
- `USB_VBUS`
- `LORA_NSS`
- `LORA_NRST`
- `LORA_DIO1`
- `LORA_BUSY`
- `PRESSURE_ADC`
- `SENSOR_5V_EN`

## Datasheet Review Gates

### Gate 1: Before Schematic Signoff

Verify against primary datasheets:

- ESP32-C3-MINI-1
- E22-900M22S
- MCP73871
- TPS63021
- chosen pressure transducer
- protection devices used on USB, RF, and sensor lines

Questions to close:

- are all mandatory support parts present
- are all absolute maximum ratings respected
- are boot and default states safe
- are rail voltages and sequencing coherent
- are connector-facing inputs protected enough for outdoor use

### Gate 2: Before PCB Signoff

Verify:

- decoupling placement
- RF routing constraints
- current loops and return paths
- battery/charger trace widths
- ADC routing cleanliness
- switched-rail routing and gate placement
- connector orientation and service access

## Decisions Already Locked

- node-first rev A
- separate gateway board later
- no external RTC in rev A
- no external watchdog in rev A
- single 0.5-4.5V analog transducer interface
- switched 5V excitation for pressure sensor

## Remaining Decisions During Capture

- exact 5V excitation regulator
- exact USB-C/native-USB capture details
- final GPIO allocation after boot-pin review

## Recommended Capture Order

1. Sheet 1 power input and battery
2. Sheet 2 regulation and switched rails
3. Sheet 3 ESP32-C3 core and USB/programming
4. Sheet 5 sensors and analog front end
5. Sheet 4 LoRa and RF
6. Sheet 6 connectors, labels, and test points
7. Whole-schematic datasheet review pass

## Do Not Do In Rev A

- do not force node and gateway onto one PCB
- do not support both 4-20mA and 0.5-4.5V in the same first-pass analog front end
- do not allocate strap-sensitive GPIOs casually
- do not claim a final power number before measurement
- do not apply for sponsored materials before schematic and BOM review are complete
