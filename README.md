# 🤖 Dual-Master Arduino Robot — FreeRTOS, Master/Slave I2C, Multi-Sensor

A multi-sensor, RTOS-driven autonomous robot built around **two Arduino Mega 2560 boards** that talk to each other over **I2C**. The top board runs a real **FreeRTOS** scheduler with four cooperating tasks driving motion control, sensor fusion, audio, and graphical display, while the bottom board acts as a dedicated I2C slave handling the headlight subsystem on independent silicon. A clean separation of concerns, a real RTOS, a real bus protocol — embedded engineering done properly.

## Highlights

- **Two Arduino Mega 2560 boards** in a master/slave architecture
- **FreeRTOS** preemptive scheduler with **4 prioritized tasks** running concurrently on the master
- **I2C bus protocol** for inter-board command exchange
- **6+ peripherals** integrated: motor driver, MP3 audio, TFT display, IR line followers, light sensor, joystick, 7-segment digit
- Modular firmware split across two independently flashable boards

## Architecture

```
                ┌─────────────────────────────────────┐
                │    TOP BOARD — Arduino Mega 2560    │
                │            (I2C MASTER)             │
                │                                     │
                │  ┌───────────────────────────────┐  │
                │  │       FreeRTOS Kernel         │  │
                │  ├───────────────────────────────┤  │
                │  │  T1  Motion / Line Following  │  │
                │  │  T2  Sensor Polling (LDR/JS)  │  │
                │  │  T3  TFT Display Updates      │  │
                │  │  T4  Audio Cues (DFPlayer)    │  │
                │  └───────────────────────────────┘  │
                │                                     │
                │  L298N · TCRT5000 ×N · LDR ·        │
                │  Joystick · TFT · MP3 · 7-seg       │
                └──────────────────┬──────────────────┘
                                   │
                                   │  I2C (SDA / SCL)
                                   │  single-byte command
                                   ▼
                ┌─────────────────────────────────────┐
                │   BOTTOM BOARD — Arduino Mega 2560  │
                │             (I2C SLAVE)             │
                │                                     │
                │   Receives 1 byte → sets headlight  │
                │   intensity (0 = off, 1 = dim,      │
                │   2 = bright)                       │
                └─────────────────────────────────────┘
```

The split is intentional: the master concentrates on real-time control loops under FreeRTOS, while the slave board owns the headlight PWM domain entirely. The I2C protocol between them is intentionally minimal — one byte, three states — which keeps the bus deterministic and the slave firmware trivially auditable.

## Hardware

| Component                | Role                                                        | Qty |
|--------------------------|-------------------------------------------------------------|-----|
| Arduino Mega 2560        | Master + slave compute                                      | 2   |
| L298N H-Bridge           | Dual-motor driver for the drive train                       | 1   |
| DFPlayer Mini            | MP3 audio playback for status / event cues                  | 1   |
| MCUFRIEND TFT Shield     | 2.4" colour graphical display (status, telemetry, UI)       | 1   |
| TCRT5000                 | Reflective IR sensors for line following                    | N   |
| LDR                      | Ambient light sensing (drives auto-headlight decision)      | 1   |
| Analog Joystick          | Manual override / mode selection input                      | 1   |
| 7-Segment Display        | Numeric status indicator                                    | 1   |

## FreeRTOS Tasks

The master board runs the FreeRTOS kernel with four tasks scheduled preemptively by priority. Each task has a fixed period and a clear single responsibility:

| #  | Task                | Responsibility                                                                                  |
|----|---------------------|-------------------------------------------------------------------------------------------------|
| T1 | **Motion Control**  | Reads the line-follower array, computes drive commands, drives the L298N motor channels         |
| T2 | **Sensor Polling**  | Samples the LDR + joystick, updates shared state, decides headlight intensity & dispatches I2C  |
| T3 | **Display**         | Refreshes the MCUFRIEND TFT and 7-segment digit with current robot state and telemetry          |
| T4 | **Audio**           | Triggers DFPlayer Mini cues on state transitions (mode change, line lost, light threshold, ...) |

Task periods and priorities are tuned so that the latency-critical motion task always preempts the slower display refresh and audio cue tasks — the classic priority-by-deadline pattern.

## I2C Protocol

The master ↔ slave handshake is one byte wide:

| Byte | Headlight intensity |
|------|---------------------|
| `0`  | OFF                 |
| `1`  | DIM                 |
| `2`  | BRIGHT              |

The master computes intensity from the LDR reading (and any manual override from the joystick) and pushes the byte to the slave's I2C address. The slave's only job is to map that byte to a PWM duty cycle on the headlight pin. This keeps the bus traffic O(1) per decision and isolates the headlight power stage from the rest of the system.

## Tech Stack

- **C++** (Arduino dialect)
- **Arduino IDE** for compile + flash
- **FreeRTOS** for preemptive task scheduling on the master
- **Wire** library for I2C transport
- **MCUFRIEND_kbv** + **Adafruit_GFX** for the TFT display
- **DFPlayerMini_Fast** for MP3 audio control

## Building & Flashing

1. Install the **Arduino IDE** (1.8.x or 2.x).
2. Install the required libraries via Library Manager:
   - FreeRTOS for Arduino
   - MCUFRIEND_kbv
   - Adafruit_GFX
   - DFPlayerMini_Fast
3. Open `Top_i2c/` in the IDE → select **Tools → Board → Arduino Mega 2560** → select the correct serial port → **Upload** to the top board.
4. Open `Bottom_i2c/` in a second IDE window → select **Arduino Mega 2560** + the second board's port → **Upload** to the bottom board.
5. Wire SDA/SCL between the two Megas (pins 20/21 on each), share GND, and power up.

The two boards then come up independently, the master initializes the FreeRTOS scheduler, and the slave begins listening on its I2C address — no further configuration is required.

## Course Context

Built for the **Embedded Systems** course in the B.Sc. Computer Science & Engineering programme at the **German University in Cairo (GUC)**, 2022.

## Authors

Anas ElNemr  ·  Ahmed Eltawel
