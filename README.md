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

## Interaktivní BOM

* [Interaktivní BOM](./PDF/ibom.html)

## Functional references

* [ESP32-C3 pin table](./References/esp32c3_pin_table.html)
* [Shift register reference](./References/shift_register_reference.html)
* [Shift register simulator](./References/shift_register_simulator.html)

