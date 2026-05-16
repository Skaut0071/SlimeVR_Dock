# SlimeVR\_Dock

Continuation of the tracker project: https://github.com/Skaut0071/SlimeVR\_Tracker

This repository contains a smart (overengineered) charging dock station for my previously designed SlimeVR FBT trackers.

The dock is built around an ESP32-C3, which controls the charging system and provides protection logic for connected trackers if a fault condition occurs.

Main functionality includes:

* Charging status monitoring for connected trackers
* Temperature monitoring
* Active voice error alerts
* Wireless connectivity over Wi-Fi

## Firmware flashing note

To flash new firmware versions, you must use the internal programming pins.

USB-C cannot be used for flashing in this design, because the same USB connection is used to power the board.

## Interactive BOM

* [Interactive BOM](https://skaut0071.github.io/SlimeVR_Dock/PDF/ibom.html)

## PDF Schematic

* [PDF Schematic](https://skaut0071.github.io/SlimeVR_Dock/PDF/SlimeVR_Dock.pdf)

## Functional references

The reference material was merged into a single page with tabs/subsections.

* [Full ESP32-C3 and shift-register reference](https://skaut0071.github.io/SlimeVR_Dock/References/esp32c3_full_reference.html)

