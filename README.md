# Formula Student dashboard and central DAQ

Arduino firmware for the **second central DAQ unit** on a Formula Student car, built with **Team Fateh** (UNSW). The unit sits on the driver dash, reads vehicle data over **CAN**, drives a **Nextion HMI**, controls **engine crank/kill**, and provides **RPM shift lights** on a NeoPixel strip.

This repo is a public snapshot of the dashboard firmware I worked on. It is team engineering work, not a solo project — expect iterative version folders, trial sketches, and some features left commented out mid-development.

---

## System overview

```
  ECU / vehicle CAN bus (500 kbps)
           │
           ▼
  ┌─────────────────────────────┐
  │  Arduino Mega (dash master) │
  │  · MCP2515 CAN transceiver  │
  │  · Gear selector (pulse in) │
  │  · Crank / kill outputs     │
  │  · Wheel-speed interrupt    │
  │  · Brake pressure (analog)  │
  └──────┬──────────┬───────────┘
         │          │
    Serial3     NeoPixel strip
    (9600)      (19 × WS2812)
         │
         ▼
  Nextion HMI (HMI2.2.0)

  Serial2 (9600) ──► LoRa module (remote crank/kill / telemetry trials)
  SPI bus ──► SD card (logging trials; shares bus with CAN CS)
```

Each loop cycle the dash master:

1. **Listens on CAN** for ECU frames and decodes RPM, coolant temperature, and battery voltage.
2. **Reads gear position** from a selector wired to a digital input — gear is inferred from pulse width (`pulseIn`).
3. **Updates the Nextion display** over UART using Nextion text-component commands (`t4`, `t5`, `t10`, etc.).
4. **Drives the shift-light strip** — blue below ~3.5k RPM, green through mid-range, red near redline, flashing above ~9.5k.
5. **Handles LoRa commands** (in later versions) — `'c'` cranks the engine, `'k'` kills it, `'d'` sends a test telemetry string.
6. **Optional subsystems** (present in some versions): wheel-speed from a proximity sensor interrupt, brake-pressure bar graph on the HMI, SD card CSV logging, over-temp HMI background warning above 95 °C.

---

## CAN messages decoded

The firmware listens on a **500 kbps** CAN bus (8 MHz SPI clock, CS pin 10, INT pin 2).

| CAN ID (decimal) | Hex ID   | Data extracted                          | HMI fields      |
| ---------------- | -------- | --------------------------------------- | --------------- |
| `218099784`      | `0CFFF048` | RPM (bytes 0–1, `(MSB×256)+LSB`)       | `t4`            |
| `218101064`      | `0CFFF548` | Battery voltage (bytes 0–1, ×0.01 V)   | `t10`           |
|                  |          | Coolant temp (bytes 4–5, ×0.1 °C)      | `t5`            |

When CAN packets arrive, status indicators on the HMI turn **green**; loss of signal turns them **red**.

> Standalone CAN bring-up sketches live in `CAN/` — useful for verifying transceiver wiring and decoding frames on Serial before integrating into the full dash.

---

## Gear detection

Gear is read from **pin 21** using `pulseIn(21, HIGH)`. Each gear position produces a distinct pulse width:

| Pulse width (µs) | Gear shown |
| ---------------- | ---------- |
| 700 – 900        | 1          |
| 920 – 1200       | N          |
| 1220 – 1500      | 2          |
| 1520 – 2000      | 3          |
| 2020 – 2500      | 4          |
| 2520 – 3000      | 5          |
| 3020 – 3500      | 6          |

Crank is only allowed when the selector reads **Neutral** (920–1200 µs).

---

## Hardware

### On the DAQ PCB

| Component            | Role                                      |
| -------------------- | ----------------------------------------- |
| Arduino Mega         | Main dash controller                      |
| Arduino Nano         | Secondary / auxiliary (team layout)       |
| MCP2515 CAN module   | Vehicle bus interface                     |
| Buck converter       | 12 V → logic supply                       |

### On the dash assembly

| Component            | Role                                      |
| -------------------- | ----------------------------------------- |
| Nextion touch HMI    | Driver display (`HMI Display/HMI2.2.0.HMI`) |
| Crank button         | Engine start (via LoRa or direct output)  |
| Kill switch          | Engine shutdown                           |
| Toggle switches      | Mode / auxiliary inputs                   |
| MicroSD module       | On-car data logging (SPI)                 |
| NeoPixel LED strip   | 19-LED RPM shift lights                   |
| LoRa E32 module      | Wireless crank/kill and telemetry trials  |

### Key Mega pin assignments (dash master)

| Pin    | Function                          |
| ------ | --------------------------------- |
| 2      | CAN interrupt                     |
| 3      | Wheel-speed proximity (interrupt) |
| 7      | NeoPixel data                     |
| 10     | CAN chip select                   |
| 21     | Gear selector pulse input         |
| 26     | Kill output                       |
| 28     | Crank output                      |
| 53     | SD chip select (when logging)     |
| A8     | Brake pressure analog input       |
| Serial2| LoRa UART (9600 baud)             |
| Serial3| Nextion HMI UART (9600 baud)      |

Vehicle wiring into the unit: **12 V**, chassis ground, **CAN H/L**, analogue gear selector, proximity sensor, SPI to SD card, UART to LoRa.

---

## Repository layout

| Folder | Contents |
| ------ | -------- |
| **`DASH_MASTER/`** | Main integrated firmware. Each subfolder is a version snapshot (see below). Sketches are split across `.ino` tabs: `CAN.ino`, `Gear.ino`, `RPM_LED.ino`, `Engine_control.ino`, `LoRa.ino`, `Speed.ino`, `Brake_pressure.ino`, `Sd_card.ino`, etc. |
| **`HMI Display/`** | Nextion Editor project (`HMI2.2.0.HMI`) — compile and flash to the display separately. |
| **`CAN/`** | Standalone CAN receive trials and versioned CAN-only sketches (`can_2.1.1`, `can_2.2.2`, …). |
| **`Gear/`** | Isolated gear-selector → HMI test (`Gear1.0`). |
| **`LoRa/`** | E32 LoRa module trials, transmitter sketches, and breakout reference files. |
| **`SPI comm/`** | SPI bus sharing between CAN and SD card (`SPI_SD_CAN`). |
| **`Neopixel LEDs/`** | Shift-light colour and timing experiments. |
| **`MISC/`** | Early dash edits and one-off tests. |

---

## Version guide

Versions follow `dash_master_X.Y.Z` naming. Later numbers generally add features on top of earlier work:

| Folder | Highlights |
| ------ | ---------- |
| `dash_master_2.2.5` | Brake pressure on HMI, coolant over-temp warning |
| `dash_master_2.2.6_LoRa_SD` | LoRa + SD logging integration (partially commented) |
| `dash_master_2.3.3_crank_kill_working` | **Reliable LoRa crank/kill** — good reference for engine control |
| `dash_master_2.3.5` | Latest snapshot — crank/kill, wheel speed, LoRa telemetry command |

Start with **`dash_master_2.3.5`** for the most complete feature set, or **`dash_master_2.3.3_crank_kill_working`** if you only need CAN + gear + HMI + crank/kill + shift lights.

---

## Libraries

Install via the Arduino Library Manager or PlatformIO:

- [`CAN`](https://github.com/sandeepmistry/arduino-CAN) — MCP2515 CAN bus
- [`FastLED`](https://github.com/FastLED/FastLED) — NeoPixel shift lights
- `SD` / `SPI` — SD card logging (built into Arduino core)

Board target: **Arduino Mega 2560**.

---

## Building and flashing

1. Open the desired version folder in the Arduino IDE (e.g. `DASH_MASTER/dash_master_2.3.5/`).
2. Select **Board: Arduino Mega 2560** and the correct serial port.
3. Install the libraries above.
4. Upload the sketch — all `.ino` tabs in the folder are compiled together.
5. Flash the Nextion separately using the `.HMI` project in `HMI Display/` with Nextion Editor.

---

## Note

This is **Team Fateh Formula Student** firmware — a working fork of the team's dashboard codebase. Some subsystems (SD logging, full LoRa telemetry, speed estimation) were developed incrementally and may be commented out or split across version folders. Treat this as a reference implementation of the dash DAQ architecture rather than a single polished release.
