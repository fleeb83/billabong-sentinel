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
- SX1276 LoRa radio
- MCP73871 USB / solar charger
- TPS63021 3.3V buck-boost regulator
- 2x 18650 in parallel
- SHT40 external ambient sensor
- SHT31 internal enclosure sensor
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
- battery measurement divider if used
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
- switched 3.3V sensor rail
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
- EN / BOOT / auto-programming circuit
- CH340C USB-UART
- USB-C data/power/ESD
- VBUS detect path if retained
- local decoupling and module support parts

Checklist:

- confirm all mandatory ESP32-C3 support circuitry
- confirm boot-strapping pins are not abused
- confirm CH340C auto-reset topology is boot-safe
- confirm USB ESD and connector pinout
- confirm exposed programming and debug access
- confirm any unused module pins are handled correctly

### Sheet 4: LoRa Radio and RF Interface

Include:

- SX1276 device or module interface
- SPI bus
- reset and DIO interrupt lines
- antenna connector path
- RF ESD where justified
- optional matching or reference network required by chosen implementation

Checklist:

- confirm whether rev A uses a bare SX1276 implementation or a module footprint
- confirm exact reference circuit from the chosen radio implementation datasheet
- confirm DIO lines actually required in firmware
- confirm reset/default-state behavior at boot
- confirm RF connector choice and grounding approach
- confirm that schematic symbols/footprints match the chosen RF implementation

### Sheet 5: Sensors and Analog Front End

Include:

- pressure transducer connector
- input protection chain
- RC filter
- resistor divider
- ADC net to MCU
- SHT40 interface
- SHT31 interface
- switched-I2C pull-ups

Checklist:

- confirm transducer supply voltage and output range from the actual chosen sensor
- confirm divider ratio leaves ADC headroom
- confirm ADC source impedance is acceptable for ESP32-C3 ADC sampling
- confirm TVS choice makes sense for a long outdoor cable
- confirm SHT31 address pinning to 0x45
- confirm external SHT40 connector/wiring assumptions
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
- `+3V3_SENSOR`
- `USB_VBUS`
- `LORA_CS`
- `LORA_RST`
- `LORA_DIO0`
- `I2C_SDA`
- `I2C_SCL`
- `PRESSURE_ADC`
- `SENSOR_3V3_EN`
- `SENSOR_5V_EN`

## Datasheet Review Gates

### Gate 1: Before Schematic Signoff

Verify against primary datasheets:

- ESP32-C3-MINI-1
- chosen SX1276 implementation
- MCP73871
- TPS63021
- CH340C
- SHT40
- SHT31
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

- exact pressure transducer part
- exact 5V excitation regulator
- pressure transducer board connector
- exact SX1276 implementation choice for rev A footprint
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
