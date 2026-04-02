# Flipper Omega 🐬

> A full SMD PCB implementation of the [diy_flipper_zero](https://github.com/lamtranBKHN/diy_flipper_zero) firmware by lamtranBKHN, built around the **WeAct Studio STM32WB55CGU6** devkit.

---

## Overview

**Flipper Omega** is a compact, two-PCB sandwich design that replicates the core functionality of the Flipper Zero using off-the-shelf modules and fully SMD components. It targets the `ssd1306_mcp23017` branch of the diy_flipper_zero firmware, which uses an I2C OLED display and an MCP23017 I/O expander to overcome the reduced GPIO count of the WeAct devkit compared to the original Flipper Zero's STM32WB55RGV6.

---

## Hardware Summary

### Main MCU
| Component | Details |
|---|---|
| **MCU** | WeAct Studio STM32WB55CGU6 devkit |
| **Core** | ARM Cortex-M4 @ 64 MHz + Cortex-M0+ (BLE stack) |
| **RAM** | 256 KB |
| **Flash** | 1 MB |
| **Wireless** | BLE 5.4, 802.15.4 (Zigbee/Thread/Matter) |
| **Dimensions** | 56.06 × 21.08 mm |

### Two-PCB Sandwich Design

The design splits into two PCBs connected via a **6-pin FFC cable (0.5mm pitch)**:

**PCB Inferior (Bottom):**
- WeAct STM32WB55CGU6 devkit
- TP4056 LiPo charger + DW01/FS8205A battery protection
- INA219BID battery monitor (I2C)
- CC1101 Sub-GHz radio (SPI)
- microSD card slot (SPI)
- IR transmitter (3× AM2520F3C03-P22 LEDs) + TSOP38238 receiver
- MLT-8530 passive piezo buzzer
- LCM0827A3038F vibration motor
- USB Type-C (charging + programming)
- Power switch (DSHP01TSGER)

**PCB Superior (Top):**
- MCP23017T-E/ML I/O expander (QFN-28, I2C)
- 5-way tactile navigation switch
- Back button (SMD tactile)
- RGB status LED (YLED5050RGB)
- SSD1306 OLED display connector (I2C)

### FPC Inter-Board Connector (7 pins)

| Pin | Signal | Description |
|---|---|---|
| 1 | GND | Ground |
| 2 | VCC | 3.3V power |
| 3 | I2C_SDA | Shared I2C data |
| 4 | I2C_SCL | Shared I2C clock |
| 5 | MCP_INT | MCP23017 interrupt → PB0 |
| 6 | GPB0 | Vibration motor control |
| 7 | GND | Ground (stability) |

---

## Pin Mapping

### I2C Bus (I2C1)
| Signal | STM32 Pin | Connected to |
|---|---|---|
| SDA | PB9 | OLED, MCP23017, INA219 |
| SCL | PA9 | OLED, MCP23017, INA219 |

### SPI Bus (SPI1, shared)
| Signal | STM32 Pin |
|---|---|
| MOSI | PB5 |
| MISO | PA6 |
| SCK | PB3 |
| CC1101 CS | PA15 |
| SD CS | PA10 |

### Other Peripherals
| Signal | STM32 Pin | Function |
|---|---|---|
| IR TX | PA8 | IR LED driver (TIM) |
| IR RX | PA0 | TSOP38238 output |
| BUZZ | PB8 | Piezo buzzer (TIM16 PWM) |
| MCP_INT | PB0 | MCP23017 interrupt |

### MCP23017 Button Mapping
| Button | MCP Pin | Port |
|---|---|---|
| Up | GPA0 | Port A |
| Right | GPA1 | Port A |
| OK | GPA2 | Port A |
| Back | GPA3 | Port A |
| Down | GPA4 | Port A |
| Left | GPA5 | Port A |

### MCP23017 Output Mapping
| Function | MCP Pin | Port |
|---|---|---|
| Vibration motor | GPB0 | Port B |
| RGB Red | GPB1 | Port B |
| RGB Green | GPB2 | Port B |
| RGB Blue | GPB3 | Port B |

> **Note:** All button inputs use MCP23017 internal pull-ups (GPPUA/GPPUB = 0xFF). Wire buttons active-low: one side to MCP GPIO, other side to GND.

---

## I2C Device Addresses

| Device | Address |
|---|---|
| SSD1306 OLED | 0x3C |
| MCP23017 | 0x20 (A0/A1/A2 → GND) |
| INA219 | 0x40 (A0/A1 → GND) |

---

## Power Architecture

```
USB Type-C (VBUS)
    │
    ├── TP4056 (LiPo charger, PROG = 1kΩ → ~1A)
    │       │
    │   DW01 + FS8205A (battery protection)
    │       │
    │     LiPo Battery (3.7V, ≥1000mAh recommended)
    │       │
    │    INA219 shunt (0.1Ω) ← current monitoring
    │       │
    │    Power switch (DSHP01TSGER)
    │       │
    └── WeAct onboard 3.3V LDO → VCC (all peripherals)
```

> **Battery note:** The PROG resistor (R16 = 1kΩ) sets charge current to ~1000mA. Use a LiPo battery of at least 1000mAh. Formula: `I(mA) = 1000 / R(kΩ)`.

---

## Schematic Notes

- **I2C pull-ups:** The SSD1306 OLED module includes onboard pull-ups. R21/R22 (3.3kΩ) on the PCB are **DNP (Do Not Populate)** by default — only solder if your OLED module does **not** include pull-up resistors.
- **Vibration motor:** Driven via MMBT2222A NPN transistor (Q1) controlled by MCP23017 GPB0. Includes 1N4148W flyback diode.
- **IR transmitter:** 3× IR LEDs in parallel, each with 20Ω series resistor. MMBT2222A driver with 330Ω base resistor. Controlled by PA8 (IR_TX).
- **Piezo buzzer:** MMBT2222A driver with 1kΩ base resistor, controlled by PB8 (BUZZ/TIM16 PWM).
- **MCP23017 interrupt:** IOCON.MIRROR = 1 — both INTA and INTB are OR'd and routed to a single MCU pin (PB0).
- **MCP23017 address:** A0/A1/A2 all tied directly to GND → address 0x20. No resistors needed.
- **SD card pull-ups:** 10kΩ on CS, CMD, DAT0, DAT1, DAT2 lines. 100nF + 10µF decoupling on VDD.

---

## Firmware

This board targets the **`ssd1306_mcp23017`** branch of [lamtranBKHN/diy_flipper_zero](https://github.com/lamtranBKHN/diy_flipper_zero).
BTW, thanks to Nucleus Dark, the creator of the whole diy flipper zero project. 
[GthiN89/FuckingCheapFlipperZero-DIY-Flipper-zero-The-real-on](https://github.com/GthiN89/FuckingCheapFlipperZero-DIY-Flipper-zero-The-real-on/tree/dev) 
(https://www.youtube.com/@NucleusDark-xe1ne)
### Flashing
On this Repository i Will NOT get into the flashing part because its a process i havent documented as well i as i wanted to. I will link an amazing youtube video on how to flash the board OTP.
https://www.youtube.com/watch?v=i19l9XjMNNI

---
## Display
It must be a SSD1306 Display, doesn't matter if its 0.96" or 1.3", as long as the communication protocol is I2C, It *Will* work, if it isn't SSD1306 it *WILL NOT WORK!!!*
## What Works / Limitations
---
| Feature | Status |
|---|---|
| OLED display (SSD1306 I2C) | ✅ |
| Button input (via MCP23017) | ✅ |
| RGB LED (via MCP23017) | ✅ |
| Vibration motor (via MCP23017) | ✅ |
| Sub-GHz radio (CC1101) | ✅ |
| microSD card | ✅ |
| IR TX/RX | ✅ |
| Piezo buzzer (TIM16) | ✅ |
| Battery monitoring (INA219) | ✅ |
| USB (programming + charging) | ✅ |
| NFC | ❌ Not supported |
| iButton | ⚠️ Pin available, not wired on this PCB |

---

## Bill of Materials (Key Components)

| Component | Part | Package | LCSC |
|---|---|---|---|
| MCU devkit | WeAct STM32WB55CGU6 | Module | — (AliExpress) |
| I/O Expander | MCP23017T-E/ML | QFN-28 | C2678 |
| Battery monitor | INA219BID | SOIC-8 | C2859699 |
| Sub-GHz radio | CC1101 | Module | — (AliExpress) |
| LiPo charger | TP4056 | ESOP-8 | C725790 |
| Battery protection IC | DW01A | SOT-23-6 | C351410 |
| Battery protection FET | FS8205A | SOT-23-6 | C2830320 |
| OLED display | SSD1306 128×64 I2C | Module (THT) | — (AliExpress) |
| IR LED | AM2520F3C03-P22 | SMD 2520 | C6859641 |
| IR receiver | TSOP38238 | THT | C141632 |
| Buzzer | MLT-8530 | SMD 8.5×8.5mm | C94599 |
| Vibration motor | LCM0827A3038F | SMD disc | C2759981 |
| RGB LED | YLED5050RGB | SMD 5050 | — (search LCSC) |
| NPN transistor | MMBT2222A | SOT-23 | C83939 |
| Flyback diode | 1N4148W | SOD-123 | C81598 |
| FPC connector | FPC 0.5mm 6P bottom | SMD | — (search LCSC) |
| Power SIP switch | DSHP01TSGER | SMD | — (search LCSC) |

---

## Safety Notes

- Do **not** connect motors, speakers or high-current loads directly to MCP23017 GPIO pins. Always use a transistor or MOSFET driver stage.
- Do **not** drive the vibration motor directly from any I/O pin without the MMBT2222A driver circuit.
- The charge current is set to ~1000mA. Use a LiPo battery rated for at least 1C charge (≥1000mAh). Adjust R16 if using a smaller battery: `R(kΩ) = 1000 / I(mA)`.
- VBAT (VB pin of WeAct) is left unconnected — the devkit handles this internally.

---

## Credits

- Firmware: [lamtranBKHN/diy_flipper_zero](https://github.com/lamtranBKHN/diy_flipper_zero)
- Original Flipper Zero: [flipperdevices/flipperzero-firmware](https://github.com/flipperdevices/flipperzero-firmware)
- WeAct Studio STM32WB55CGU6: [WeActStudio/WeActStudio.STM32WB55CoreBoard](https://github.com/WeActStudio/WeActStudio.STM32WB55CoreBoard)

---

## License

GPL-3.0 — see [LICENSE](LICENSE) for details.
