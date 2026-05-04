# Embedded Project — Two-Arduino Lane-Following Car with Headlights, MP3 Player & Touchscreen UI

A FreeRTOS-scheduled, dual-microcontroller robot car built for the Embedded Systems course at the German University in Cairo (GUC), December 2021. Two Arduino Mega boards split the workload — one drives the chassis (line-following motors, ambient-light-triggered headlights, blinker warnings), the other runs the cabin (TFT touchscreen UI, DFPlayer Mini MP3 module, joystick-controlled gear display on a 7-segment) — and they coordinate over I2C.

---

## What It Does

- **Drives itself along a black line** using two TCRT5000 IR reflectance sensors (front-mounted, one per side) feeding a 4-state controller: forward / turn-left / turn-right / stop.
- **Turns headlights on automatically when it gets dark.** The Top Arduino reads an ambient-light sensor, classifies brightness into three buckets (bright / dim / dark), and sends a single byte over I2C to the Bottom Arduino, which drives two PWM-controlled headlight LEDs at 0 / 50 / 255 duty cycle accordingly.
- **Flashes turn-signal blinkers** on the side opposite the line-follower trigger (left blinker when veering left to chase the line, both blinkers when forced to stop).
- **Plays MP3 files from an SD card** via a DFRobot DFPlayer Mini, controlled from a custom touchscreen UI drawn on a 2.4" MCUFRIEND TFT shield: Play / Pause / Next / Previous, Volume +/-, and a 6-mode equalizer (Normal / Pop / Rock / Jazz / Classic / Bass).
- **Shows the current "gear"** as a digit on a 7-segment display, driven by an analog joystick (read on A8 / A9). Four joystick zones map to four 7-segment patterns.

---

## Hardware

| Component | Qty | Notes |
|---|---|---|
| Arduino Mega 2560 (or compatible) | 2 | One per "level": Top = cabin, Bottom = chassis |
| 2-wheel car chassis kit | 1 | DC gear motors + caster |
| L298N dual H-bridge motor driver | 1 | Driven by Bottom Arduino |
| TCRT5000 line-follower modules | 2 | Analog out, threshold ~350 |
| LDR / ambient light sensor | 1 | Read on Top Arduino (A13) |
| DFRobot DFPlayer Mini | 1 | UART, software-serial on pins 52 / 53 |
| MicroSD card | 1 | MP3 files in DFPlayer's expected naming convention |
| 8-Ohm speaker | 1 | Wired to DFPlayer's SPK_1 / SPK_2 |
| 2.4" MCUFRIEND TFT LCD shield (320x240) | 1 | With 4-wire resistive touch |
| Common-cathode 7-segment display | 1 | Driven on pins 25, 27, 29, 31, 33, 35, 37 |
| Analog 2-axis joystick | 1 | X -> A8, Y -> A9 |
| 5 mm LEDs | 4 | 2 headlights (PWM) + 2 blinker indicators |
| Jumper wires, breadboard, battery pack | - | I2C bus needs both grounds tied together |

**Communication buses**

- **I2C** — Top Arduino is master, Bottom is slave at address `5`. One byte sent per loop iteration carrying the headlight intensity command (`0` = off, `1` = dim, `2` = bright).
- **SoftwareSerial @ 9600 baud** — Top Arduino <-> DFPlayer Mini on pins 52 (RX) / 53 (TX).
- **8-bit parallel** — TFT shield (MCUFRIEND) on the standard Uno/Mega shield footprint.
- **Hardware Serial @ 9600 baud** — both boards open `Serial` for debug prints.

---

## Pin Map

### Top Arduino (cabin: TFT + MP3 + joystick + 7-seg + light sensor)

| Pin | Role |
|---|---|
| A0 | LCD `RD` |
| A1 | LCD `WR` |
| A2 | LCD `CD` (also touch `YP`) |
| A3 | LCD `CS` (also touch `XM`) |
| A4 | LCD `RESET` |
| A8 | Joystick X-axis |
| A9 | Joystick Y-axis |
| A13 | Ambient-light sensor |
| 8 | Touch `YM` |
| 9 | Touch `XP` |
| 25 | 7-segment middle bar |
| 27 | 7-segment top-left |
| 29 | 7-segment top |
| 31 | 7-segment top-right |
| 33 | 7-segment bottom-left |
| 35 | 7-segment bottom |
| 37 | 7-segment bottom-right |
| 52 | DFPlayer RX (SoftwareSerial) |
| 53 | DFPlayer TX (SoftwareSerial) |
| SDA / SCL (20 / 21 on Mega) | I2C master to Bottom Arduino |

### Bottom Arduino (chassis: motors + line followers + headlights + blinkers)

| Pin | Role |
|---|---|
| A0 | Left line-follower (analog) |
| A1 | Right line-follower (analog) |
| A3 | Light sensor (declared but actually controlled from Top) |
| 5 | `OnLED` digital input + alternate headlight PWM (`led_pin1`) |
| 6 | `led_pin2` — second headlight PWM output |
| 11 | Right wheel ENB (PWM speed @ 220) |
| 12 | Left wheel ENA (PWM speed @ 220) |
| 13 | Left blinker LED |
| 24 | Motor 2 IN1 |
| 26 | Motor 2 IN2 |
| 31 | Motor 1 IN1 |
| 33 | Motor 1 IN2 |
| 40 | "On" indicator LED |
| 51 | Right blinker LED |
| SDA / SCL (20 / 21 on Mega) | I2C slave (address `5`) from Top Arduino |

> The pin maps are taken straight from the `#define` and `pinMode()` calls in `Bottom_i2c/Bottom_i2c.ino` and `Top_i2c/Top_i2c.ino`. Note: the Bottom-Arduino motor IN-pins use the same numbers (31, 33) the Top Arduino uses for 7-segment segments — they are on **different boards**, no conflict.

---

## Software Stack

- **Language**: C++ (Arduino dialect, no STL).
- **Framework**: Arduino IDE 1.8.x (sketches are `.ino` files; no `platformio.ini`).
- **RTOS**: [`Arduino_FreeRTOS`](https://github.com/feilipu/Arduino_FreeRTOS_Library) — both boards `xTaskCreate()` two tasks each and call `vTaskDelayUntil()` to hold steady periods. The empty `loop()` is intentional; FreeRTOS owns scheduling. The original FreeRTOS source ships in this repo as `FreeRTOS.zip`.
- **Libraries** (all bundled in `Libraries.zip`):
  - `Wire.h` — I2C
  - `Arduino_FreeRTOS.h` + `semphr.h`
  - `AFMotor.h` (Adafruit Motor Shield v1 — pulled in but the L298N path actually uses raw `digitalWrite`)
  - `MCUFRIEND_kbv.h` + `Adafruit_GFX.h` + `Adafruit_TFTLCD.h` — TFT driver chain
  - `TouchScreen.h` — 4-wire resistive touch
  - `SPI.h`
  - `SoftwareSerial.h`
  - `DFRobotDFPlayerMini.h`

### Task layout

| Board | Task | Period | Priority | Responsibility |
|---|---|---|---|---|
| Bottom | `t1` | 20 ms | 2 (high) | Read line followers -> drive H-bridge + blinker LEDs |
| Bottom | `t2` | 100 ms | 1 (low) | Headlight LED follow-up (intensity command arrives via I2C `receiveEvent`) |
| Top | `t1` | 100 ms | 1 (low) | TFT touch poll -> DFPlayer commands + redraw player UI; also classify light-sensor reading into `txdata` (0/1/2) for I2C send |
| Top | `t2` | 20 ms | 2 (high) | Joystick read -> 7-segment digit pattern |

The Top board's `loop()` wakes every ~200 ms and pushes the latest `txdata` byte to the Bottom over I2C — a deliberately simple "fire and forget" channel (no ack, no checksum).

---

## Project Structure

```
Embedded-Project/
+- Bottom_i2c/
|  +- Bottom_i2c.ino       Chassis sketch (~5.5 KB)
+- Top_i2c/
|  +- Top_i2c.ino          Cabin sketch (~16 KB)
+- FreeRTOS.zip            Bundled FreeRTOS port
+- Libraries.zip           Snapshot of all third-party libraries used
+- Project Report.pdf      Original GUC course-submission write-up
+- README.md               You are here
```

---

## How to Build & Flash

You need the Arduino IDE (1.8.x or newer, or Arduino IDE 2.x), two Arduino Mega 2560 boards, and a USB-A -> USB-B cable per board.

1. **Install bundled libraries.** Unzip `Libraries.zip` into your `Documents/Arduino/libraries/` folder so the IDE can resolve the `#include`s. Then either unzip `FreeRTOS.zip` into the same folder, or install the **Arduino_FreeRTOS_Library** by Phillip Stevens from the Library Manager.
2. **Open and flash the Bottom sketch.**
   - File -> Open -> `Bottom_i2c/Bottom_i2c.ino`
   - Tools -> Board -> **Arduino Mega or Mega 2560**
   - Tools -> Port -> (select the Bottom Arduino's COM/tty port)
   - Click **Upload**.
3. **Open and flash the Top sketch.**
   - File -> Open -> `Top_i2c/Top_i2c.ino`
   - Same board selection, but pick the Top Arduino's port.
   - Click **Upload**.
4. **Wire up I2C.** Tie `SDA -> SDA` and `SCL -> SCL` between the two Megas (pins 20 and 21), and **share ground**. Pull-ups on SDA/SCL aren't strictly required at short distances since the AVR's internal pull-ups are enabled by `Wire.begin()`, but a pair of 4.7 kOhm to 5 V improves reliability.
5. **Power both Megas + the L298N from the same battery pack** with a common ground.
6. **Insert an SD card** with MP3 files into the DFPlayer Mini.
7. **Place the car on a black line** (electrical tape on a light surface works well) and power on. The TFT will show the DFPlayer init sequence first, then the player UI.

> No PlatformIO config ships in the repo. If you want to migrate, the equivalent `platformio.ini` would be `[env:megaatmega2560]` with `framework = arduino`, `board = megaatmega2560`, plus the libraries listed above as `lib_deps`.

---

## Coursework Context

This was a final project for the **Embedded Systems** course at the **German University in Cairo (GUC)**, **December 2021**. The brief was to design and integrate a multi-peripheral embedded system using cooperative scheduling. Splitting the workload across two Arduinos was a deliberate choice — neither Mega's pin count nor RAM was the binding constraint, but separating the "chassis" real-time loop (20 ms motor control) from the "cabin" UI loop (100 ms TFT redraw) made the FreeRTOS priority assignment easier to reason about and gave I2C a real reason to exist beyond a textbook example. The original course report is included as `Project Report.pdf`.

---

## Known Quirks

- The Bottom sketch declares `#define lightSensor A3` and reads `analogRead(OnLED)` (where `OnLED = 5`) inside `t2`, but the **actual ambient-light decision lives on the Top Arduino** and is shipped over I2C as a byte. The Bottom's `t2` is mostly a passthrough for the LED state.
- `AFMotor.h` is included on the Bottom board but the L298N is driven with raw `digitalWrite` calls — the library reference is dead code left over from an earlier iteration and can be removed safely.
- The line-follower threshold is hard-coded at `350`. TCRT5000 modules vary; you may need to retune this for your surface contrast.
- Touch coordinate ranges (`TS_LEFT`, `TS_RT`, `TS_TOP`, `TS_BOT`) are calibrated for one specific MCUFRIEND panel — re-run a touch-calibration sketch if your taps land on the wrong button.

---

## Authors

- **Anas ElNemr** — Top level (TFT + MP3 + joystick + 7-segment), I2C, FreeRTOS scheduling
- **Ahmed Eltawel** — Bottom level (motors + line followers + headlights), chassis assembly

With contributions from teammates **Ahmed Farouk** (cabin electronics) and **Abdelrahman Hafez** (chassis wiring) on the original GUC submission.

---

## Acknowledgements

This is a personal mirror with an expanded code-walkthrough README. The friendly companion repo lives at [github.com/ahmedeltawel/Embedded-Project](https://github.com/ahmedeltawel/Embedded-Project) — same code, same project, maintained by Ahmed.
