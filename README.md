# Formula Student dashboard and central DAQ

Arduino Mega/Nano firmware for the second central DAQ unit on a Formula Student car (Team Fateh). This repository is a working fork of the team dashboard dump; the folders here are the dashboard, CAN, display, and radio pieces used on the car.

## What it does

The unit sits on the dash, talks to the vehicle over **CAN**, and drives a **Nextion HMI** plus status LEDs. Incoming vehicle data shown on the display includes battery voltage, TPS, coolant and air temperature, RPM, speed, and gear.

## Hardware (as used on the car)

On the PCB: Arduino Mega, Arduino Nano, CAN transceiver, buck converter.

On the dash: Nextion touch HMI, crank button, kill switch, toggle switches, SD module, NeoPixel LED, USB-B.

Vehicle wiring into the DAQ: 12 V and chassis ground, analogue gear, CAN H/L, proximity sensor, SPI to the SD card, and UART to LoRa.

## Layout

| Folder | Role |
| --- | --- |
| `DASH_MASTER` | Main dash controller |
| `HMI Display` | Nextion UI |
| `CAN` | CAN bus |
| `Gear` | Gear position |
| `LoRa` | Radio link |
| `SPI comm` | SPI / SD |
| `Neopixel LEDs` | Dash lighting |
| `MISC` | Supporting sketches |

## Note

This is team Formula Student work, not a solo project. Treat it as a public snapshot of the dashboard/DAQ implementation I worked on.
