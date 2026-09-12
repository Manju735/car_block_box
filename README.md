<div align="center">

# 📦 Car Black Box — Event Data Recorder

**A single-ECU "flight recorder" for your car — logs every gear shift, speed, and collision with a timestamp.**

![Language](https://img.shields.io/badge/Language-C-blue?style=for-the-badge&logo=c)
![Platform](https://img.shields.io/badge/MCU-PIC18-orange?style=for-the-badge&logo=microchip)
![Protocol](https://img.shields.io/badge/Protocol-I2C-yellow?style=for-the-badge)
![RTC](https://img.shields.io/badge/RTC-DS1307-purple?style=for-the-badge)
![IDE](https://img.shields.io/badge/IDE-MPLAB%20X-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)

<br/>

> A PIC18-based automotive "black box" that mirrors a real Event Data Recorder (EDR): it watches speed and gear-shift events in real time, timestamps every one with an onboard RTC, and permanently logs them to external EEPROM — so nothing is lost even after a crash or power loss. Logs can be reviewed on an onboard LCD menu or downloaded over UART.

</div>

---

## 📌 Table of Contents

- [About the Project](#-about-the-project)
- [System Architecture](#-system-architecture)
- [How It Works](#-how-it-works)
- [Application Menu Flow](#-application-menu-flow)
- [EEPROM Log Format](#-eeprom-log-format)
- [Hardware](#-hardware)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
  - [Prerequisites](#prerequisites)
  - [Build & Flash](#build--flash)
- [Controls](#-controls)
- [Concepts Used](#-concepts-used)
- [Roadmap](#-roadmap)
- [Author](#-author)

---

## 🧠 About the Project

Just like an aircraft's black box, this project silently records what a car is doing right before and during key events — but on a single PIC18 microcontroller instead of an aircraft-grade recorder.

- Reads **live speed** from a potentiometer via ADC and shows it on a 16x2 LCD dashboard
- Tracks **gear position** (Neutral → G1–G5 → Reverse) and a dedicated **collision** trigger, driven by a matrix keypad
- Timestamps every gear-change / collision event using a **DS1307 RTC** over I2C
- Persists every event to an **external I2C EEPROM**, so the log survives resets and power cycles
- Ships a full **on-device menu system**: View Log, Download Log (over UART), Clear Log, and Set Time
- Stores up to **10 logged events** in a rolling buffer

---

## 🏗️ System Architecture

```
        ┌─────────────────────────────────────────────┐
        │              PIC18 "Black Box" ECU            │
        │                                               │
  🎚️ POT ──► ADC ────────────────┐                       │
                                  │                       │
  ⌨️ Matrix Keypad ──► Gear/Menu ─┼──► App State Machine ─┼──► 📟 16x2 LCD
        (4x3, 8 keys used)        │      (dashboard /     │
                                  │       menu / log /     │
  🕐 DS1307 RTC ──► I2C ──────────┤       set-time)        │
                                  │         │              │
  💾 External EEPROM ◄── I2C ─────┘         ▼              │
     (event log, up to 10 entries)   📡 UART ──► PC / Terminal
        │                                                  │
        └──────────────────────────────────────────────────┘
```

---

## ⚙️ How It Works

```
Every loop iteration:
        │
        ▼
  Read keypad (state-change detection)
        │
        ▼
  Is SW4 pressed?  ──yes──►  Jump to Main Menu
        │no
        ▼
  Dispatch on current app state
        │
        ├── DASHBOARD ──► Read RTC time, read speed (ADC),
        │                  detect gear-shift/collision keys,
        │                  log event to EEPROM if changed
        │
        ├── MAIN MENU ──► Navigate with SW5/SW6, select with SW7
        │
        ├── VIEW LOG ────► Scroll stored events on the LCD
        │
        ├── DOWNLOAD LOG ► Dump all events over UART
        │
        ├── CLEAR LOG ───► Reset the EEPROM log counter
        │
        └── SET TIME ────► Edit HH:MM:SS, write back to DS1307
```

Every gear change or collision trigger calls `event_store()`, which writes an 13-byte record (timestamp + event code + speed) to the next free EEPROM slot — a rolling log of the last 10 events.

---

## 🖥️ Application Menu Flow

```
┌─────────────┐   SW4    ┌──────────────┐
│  Dashboard   │ ───────► │  Main Menu    │
│ (default view)│ ◄─────── │  (SW8 = back)│
└─────────────┘  SW8     └──────┬────────┘
                                  │  SW5/SW6 = navigate, SW7 = select
              ┌───────────────────┼───────────────────┬───────────────┐
              ▼                   ▼                   ▼               ▼
        ┌───────────┐     ┌──────────────┐    ┌────────────┐   ┌────────────┐
        │  View Log  │     │ Download Log │    │  Clear Log  │   │  Set Time   │
        │ (SW5/SW6   │     │  (dumps over  │    │ (resets log │   │ (HH:MM:SS   │
        │  scroll)   │     │    UART)     │    │   counter)  │   │  editor)    │
        └───────────┘     └──────────────┘    └────────────┘   └────────────┘
```

---

## 💾 EEPROM Log Format

Each stored event occupies **13 bytes** in the external EEPROM:

| Bytes | Field | Description |
|:-----:|-------|--------------|
| 0–7 | Timestamp | `"HH:MM:SS"` string from the DS1307 RTC |
| 8–9 | Event Code | 2-character code — `ON`, `GN`, `G1`–`G5`, `GR`, or ` C` (collision) |
| 10–12 | Speed | 3-digit speed value at the moment of the event |

Address `130` in EEPROM stores the running **log count** (max 10), which drives View Log / Download Log / Clear Log.

| Event Code | Meaning |
|:----------:|---------|
| `ON` | Power-on / initial state |
| `GN` | Neutral |
| `G1`–`G5` | Gear 1 through Gear 5 |
| `GR` | Reverse |
| ` C` | Collision detected |

---

## 🔧 Hardware

| Component | Interface / Pin | Purpose |
|-----------|------------------|---------|
| **MCU** | PIC18 (XC8, 20 MHz) | Core controller |
| **16x2 Character LCD** | `PORTD` (data), `RC0`–`RC2` (RS/RW/EN) | Dashboard, menus, log viewer |
| **DS1307 RTC** | I2C (`0xD0`/`0xD1`) | Real-time timestamping |
| **External EEPROM (24Cxx)** | I2C (`0xA0`/`0xA1`) | Persistent event log storage |
| **4x3 Matrix Keypad** | `PORTB` (rows `RB5`–`RB7`, cols `RB1`–`RB4`) | Navigation & gear/collision input |
| **Potentiometer** | ADC channel 4 | Simulated speed input |
| **UART** | — | Serial log download / debug |

---

## 📁 Project Structure

```
Car_Black_Box_Reference.X/
│
├── main.c                 # Entry point — init peripherals, app state machine
├── black_box.h             # Shared state enum, globals, function prototypes
│
├── dashboard.c             # Live dashboard: time, speed, gear/collision, event logging
├── main_menu.c             # Main menu navigation (View/Download/Clear Log, Set Time)
├── menu_options.c          # View Log, Download Log, Clear Log, Set Time implementations
│
├── clcd.c / clcd.h          # 16x2 character LCD driver
├── matrix_keypad.c / .h     # 4x3 matrix keypad scanning & switch mapping
├── adc.c / adc.h             # ADC driver (speed potentiometer)
├── i2c.c / i2c.h             # I2C bus driver (shared by RTC + EEPROM)
├── ds1307.c / ds1307.h       # DS1307 RTC driver — get/set/display time
├── external_eeprom.c / .h    # External I2C EEPROM read/write driver
├── uart.c / uart.h           # UART driver — log download over serial
│
├── nbproject/                # MPLAB X project metadata
├── build/ , dist/             # Compiler output (.hex, .elf, .map, etc.)
└── Makefile
```

---

## 🚀 Getting Started

### Prerequisites

- [MPLAB X IDE](https://www.microchip.com/mplab/mplab-x-ide)
- [XC8 Compiler](https://www.microchip.com/mplab/compilers)
- PIC18 development board + PICkit (or compatible) programmer
- 16x2 character LCD, DS1307 RTC module, I2C EEPROM (24Cxx), 4x3 matrix keypad, potentiometer

### Build & Flash

```bash
# 1. Open Car_Black_Box_Reference.X directly in MPLAB X IDE
#    (it's already a complete MPLAB X project — nbproject/ + Makefile included).

# 2. Select your target PIC18 device and connected programmer.

# 3. Clean & Build the project — this regenerates:
dist/default/production/Car_Black_Box_Ref.X.production.hex

# 4. Flash the .hex file to your board and power it up.
#    The dashboard should appear on the LCD immediately.
```

---

## 🎮 Controls

| Switch | Dashboard | Main Menu | Log / Set Time Screens |
|:------:|-----------|-----------|--------------------------|
| **SW1** | Gear up | — | — |
| **SW2** | Gear down | — | — |
| **SW3** | Trigger collision event | — | — |
| **SW4** | Open Main Menu | — | — |
| **SW5** | — | Move down | Scroll / Increment field |
| **SW6** | — | Move up | Change field (Set Time) |
| **SW7** | — | Select option | Confirm / Save |
| **SW8** | — | Back to Dashboard | Back to Main Menu |

---

## 💡 Concepts Used

| Concept | Applied In |
|---------|------------|
| **State Machine Design** | App-wide `State_t` driving `main()`'s dispatch loop |
| **I2C Protocol** | Shared bus driver used by both the DS1307 RTC and external EEPROM |
| **Non-Volatile Storage** | Persisting event logs to EEPROM across power cycles |
| **RTC Interfacing** | Reading/writing BCD time registers on the DS1307 |
| **Matrix Keypad Scanning** | Row/column scan with state-change detection |
| **Character LCD Interfacing** | Multi-screen UI (dashboard, menu, log viewer, time editor) |
| **UART Communication** | Streaming the event log to a PC terminal |
| **Modular Embedded C** | One driver per peripheral, tied together via `black_box.h` |

---

## 🗺️ Roadmap

- [ ] Expand log capacity beyond 10 entries (currently limited by the 130-byte layout)
- [ ] Add checksum/CRC to stored records for data integrity
- [ ] Add a real speed sensor / accelerometer input instead of a potentiometer
- [ ] Timestamp with full date (day/month/year), not just HH:MM:SS
- [ ] Add a low-power / power-loss detection hook to force a final log write

---

## 👤 Author

**Manjunatha H**

---

<div align="center">

⭐ If you found this project helpful, give it a **star** on GitHub!

</div>
