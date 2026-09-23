# SlimeVR\_Dock

Continuation of the tracker project: https://github.com/Skaut0071/SlimeVR_Tracker

This repository contains a smart (overengineered) charging dock station for my previously designed SlimeVR FBT trackers.

The dock is built around an ESP32-C3, which controls the charging system and provides protection logic for connected trackers if a fault condition occurs.

## Features

* 8 USB-C charging ports (J2-J9), arranged as 4 dual-port channels
* Powered from a single USB-C PD input (20V / 3A, negotiated by a CH224K), stepped down to 5V (up to 8.5A) for the ports
* Per-port voltage/current monitoring with an INA226 (8x) over I2C
* Overcurrent protection and per-channel enable/fault handling with an AP22652 load switch (4x)
* Temperature/humidity monitoring (BME280 module on the I2C header)
* 4-pin PWM fan header with tachometer input, driven by the ESP32-C3
* Audible error alerts through an on-board buzzer
* Wireless connectivity over Wi-Fi

## Hardware overview

| Block | Main parts | Notes |
| --- | --- | --- |
| MCU | ESP32-C3-WROOM-02 (U1) | Controls charging, reads monitors, handles faults |
| Power input | USB-C receptacle (J1\_MainPW1), CH224K (U2), NDT452AP (Q1) | Requests 20V from a PD source |
| 5V step-down | TPS548A20 (U4), 2.2uH inductor (L1) | 20V to 5V, 8.5A |
| 3V3 regulator | XC6220B331MR (U3) | Powers the ESP32-C3 and logic |
| Charging channels (x4) | 2x USB-C, 2x INA226, AP22652 load switch, BC857 | Each channel serves two ports; see schematic sheet below |
| I/O expansion | 2x 74HC165 (U17, U18), 74HC595 (U19) | Shift registers for reading channel status and sending control signals |
| Headers | Control, Monitor, Prog (3-pin), Temp (4-pin), Fan (4-pin) | Dev/debug access, sensor and fan connectors |
| Buzzer | BZ1 | Error alerts |

The board is a 2-layer, 1.6mm PCB, roughly 100 x 78 mm (JLCPCB gerbers included).

## Schematic structure

The design is hierarchical (KiCad):

| Sheet | File | Description |
| --- | --- | --- |
| Root - SlimeVR Dock (rev 3.1) | `Board/SlimeVR_Dock.kicad_sch` | MCU, PD input, 3V3 regulator, shift registers, connectors, buzzer |
| 20V to 5V 8.5A Stepdown (rev 2.1) | `Board/StepDown.kicad_sch` | TPS548A20 buck converter |
| Dual USB-C charger monitoring (rev 1.8) | `Board/DUSBSS.kicad_sch` | Instantiated 4x; load switch, protection and detection for a pair of ports |
| Overcurrent protection (rev 1.2) | `Board/USB-C.kicad_sch` | USB-C connector with INA226 current measurement, instantiated 2x per channel |

The PCB is currently at rev 5 (`Board/SlimeVR_Dock.kicad_pcb`).

## Firmware flashing note

To flash new firmware versions, you must use the internal programming pins (`Prog` header).
Or in future, firmware flashing over web interface of the device.

USB-C cannot be used for flashing in this design, because the same USB connection is used to power the board.

## Repository layout

* `Board/` - KiCad project (schematics, PCB, custom footprints in `Library.pretty`)
* `Gerb/` - Gerber/fabrication files for JLCPCB
* `PDF/` - PDF schematic, BOM (CSV) and interactive BOM
* `3D/` - 3D model exports

## Interactive BOM

* [Interactive BOM](https://skaut0071.github.io/SlimeVR_Dock/PDF/ibom.html)

## PDF Schematic

* [PDF Schematic](https://skaut0071.github.io/SlimeVR_Dock/PDF/SlimeVR_Dock.pdf)
