# Beer Fridge Controller — Full Build Spec

A Raspberry Pi 4B-based temperature controller for a dual-zone Samsung fridge used for beer fermentation (fridge section) and keg storage (freezer section). The Samsung control board is fully bypassed — the Pi has exclusive control of the compressor, defrost cycle, and all zone management. Electronics live in the machine compartment. Tap display screens show live keg info. Brewfather integration for fermentation profile management. Boots from M.2 SATA SSD via USB — no SD card.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Shopping List — AliExpress](#shopping-list--aliexpress)
3. [Component Details](#component-details)
4. [Custom PCB](#custom-pcb)
5. [Machine Compartment Enclosure](#machine-compartment-enclosure)
6. [Wiring](#wiring)
7. [3D Printed Parts](#3d-printed-parts)
8. [Control Logic](#control-logic)
9. [Software Stack](#software-stack)
10. [Brewfather Integration](#brewfather-integration)
11. [Discord Notifications](#discord-notifications)
12. [Physical Installation](#physical-installation)
13. [Build Phases](#build-phases)
14. [Roadmap](#roadmap)

---

## System Overview

### Zones

| Zone | Purpose | Temperature | Controlled by |
|---|---|---|---|
| Freezer | Keg storage (19L corny kegs) | 1–2 °C | Compressor via DIN-rail contactor |
| Fridge | Fermentation (50L fermenter) | Brewfather profile | Duct fan + PTC heater |

### How it works

The Samsung control board is completely bypassed. The Pi has sole, exclusive control over the compressor, defrost cycle, fans, heater, and lighting. There are no firmware conflicts, no thermistor interference, and no defrost cycle surprises — the Pi manages everything directly. The freezer is held at 1–2 °C by switching the compressor on and off via a DIN-rail mechanical contactor with a 10-minute minimum cooldown. The fermentation section is cooled by a duct fan blowing cold air from the freezer through the bottom of the divider wall. A PTC ceramic heater handles warming during cold fermentation periods. All four fans are 120mm 12V PWM units. The Pi also manages a periodic defrost cycle to prevent evaporator icing since the Samsung's own defrost heater is no longer driven by the factory board. Electronics live in the machine compartment. The system boots from an M.2 SATA SSD via USB 3.0 — no SD card.

### Fan layout

| Fan | Location | Purpose | Control |
|---|---|---|---|
| Duct fan | Bottom of divider wall | Cold air from freezer → fridge | PWM (proportional to temp error) |
| Exhaust fan | Top of divider wall | Warm fridge air → freezer (v1.7 only) | PWM (cold crash mode) |
| Freezer circ fan | Freezer zone | Circulates air around kegs | PWM (mirrors compressor) |
| Fridge circ fan | Fridge zone | Circulates air around fermenter | PWM (mirrors heating/cooling) |

All four fans are 120mm 12V 4-pin PWM. Buy all four with the initial order — the exhaust fan is installed but inactive until v1.7. The two-hole divider layout exploits natural convection: cold air (denser) enters at the bottom, warm air (lighter) exits at the top.

### Switching layout

| Load | Switch type | Reason |
|---|---|---|
| Compressor | DIN-rail mechanical contactor | Reliable, fail-open (safe) |
| Contactor 230V coil | Off-board relay module #1 | Mains off PCB, GPIO drives 3.3V signal only |
| PTC heater | Off-board relay module #2 | Mains off PCB |
| Heater/cooler interlock | SPDT relay (hardware) | Physically prevents simultaneous heating and cooling |
| Freezer LED strip | IRLB8721 MOSFET on PCB | 12V switching, GPIO driven, door-sensor triggered |

### Screen layout

| Screen | Location | Shows |
|---|---|---|
| Screen 1 | Fridge exterior | Temps, compressor, heating, fan states, active fermentation step |
| Screen 2 | Tap 1 | Beer info, % remaining, last pour, last clean |
| Screen 3 | Tap 2 | Same for second beer |

### Door sensors

Both fridge doors have factory-fitted magnetic reed switches tapped into GPIO inputs. Freezer door sensor also triggers the freezer LED strip — on when open, off when closed. When a door opens the control loop pauses fans and ignores temperature readings for a settle period after closing. Door events are logged to SQLite and Discord alerts fire if a door is left open beyond the configured threshold.

### Temperature sensors

Each zone has two DS18B20 waterproof probes submerged in sealed water containers. The water thermal mass matches the slow thermal response of 19L corny kegs and a 50L fermenter, preventing compressor short cycling. Readings are validated before use — DS18B20 returns -127°C on disconnect which is caught before averaging runs.

### Defrost cycle

Since the Samsung control board is fully bypassed, the factory defrost heater is no longer driven automatically. The Pi manages defrost directly: compressor and fans stop on a configurable interval (default 8 hours), allowing residual heat to melt frost from the evaporator coil for a configurable duration (default 25 minutes), then resumes normal operation. Interval and duration are tunable in config based on real-world frost buildup in the garage environment.

---

## Shopping List — AliExpress

**Estimated total: ~$150–170 USD including shipping.**

Aim to consolidate into as few sellers as possible. The contactor and PSUs are best sourced by model number. Buy all four fans upfront.

### Core electronics

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 1 | Raspberry Pi 4B 2GB | `Raspberry Pi 4 Model B 2GB` | 1 | $45 |
| 2 | USB 3.0 M.2 SATA enclosure adapter | `M.2 SATA USB 3.0 enclosure adapter 2280` | 1 | $5 |
| 3 | DS18B20 waterproof sensor 1m | `DS18B20 waterproof temperature sensor 1m` | 4 | $2 each |
| 4 | DIN-rail mechanical contactor 230V | `Schneider LC1D09 contactor 230V coil` | 1 | $15 |
| 5 | Off-board enclosed relay module 230V (×2) | `enclosed relay module 230V 10A` | 2 | $4 each |
| 6 | SPDT relay module 5V | `SPDT relay module 5V single channel` | 1 | $2 |
| 7 | 120mm 12V PWM fan | `120mm 12V PWM fan 4 pin quiet` | 4 | $5 each |
| 8 | PTC ceramic heater 100W 230V | `PTC ceramic heater panel 100W 230V` | 1 | $6 |
| 9 | 1.3" color TFT ST7789 display | `1.3 inch TFT ST7789 SPI 240x240 color display` | 3 | $2.50 each |
| 10 | 12V warm white LED strip IP65 | `12V LED strip warm white IP65 adhesive 1m` | 1m | $3 |
| 11 | IRLB8721 N-channel MOSFET | `IRLB8721 MOSFET TO-220 5pcs` | 1 | $2 |

### Power

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 12 | Mean Well HDR-15-12 DIN rail PSU | `Mean Well HDR-15-12` | 1 | $18 |
| 13 | Mean Well HDR-15-5 DIN rail PSU | `Mean Well HDR-15-5` | 1 | $18 |
| 14 | Inline fuse holder + 5A fuse | `inline fuse holder 5A 250V panel mount` | 2 | $2 each |

### PCB components

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 15 | 40-pin GPIO female header 2.54mm | `2x20 pin female header 2.54mm raspberry pi` | 1 | $1 |
| 16 | Screw terminal blocks 3.5mm pitch | `PCB screw terminal block 3.5mm 2pin 3pin assorted` | 1 | $3 |
| 17 | 4.7kΩ resistor pack | `4.7k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 18 | 10kΩ resistor pack | `10k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 19 | 1kΩ resistor pack | `1k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 20 | 330Ω resistor pack | `330 ohm resistor 1/4W 100pcs` | 1 | $1 |
| 21 | 2N3904 NPN transistor | `2N3904 NPN transistor 50pcs` | 1 | $1 |
| 22 | 1N4007 diode pack | `1N4007 diode 50pcs` | 1 | $1 |
| 23 | 5mm LED assorted pack | `5mm LED red green yellow assorted 100pcs` | 1 | $1 |
| 24 | 100nF decoupling capacitors | `100nF ceramic capacitor 0.1uF 100pcs` | 1 | $1 |

### Misc and installation

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 25 | Dupont jumper wire kit | `Dupont jumper wire 120pcs male female 20cm` | 1 | $2 |
| 26 | M12 cable glands nylon | `M12 cable gland nylon waterproof 10pcs` | 1 | $2 |
| 27 | M3 hex standoffs + screws kit | `M3 brass standoff spacer hex screw kit assorted` | 1 | $4 |
| 28 | M3 threaded heat inserts | `M3 heat set insert brass knurled 50pcs` | 1 | $3 |
| 29 | PCB conformal coating spray | `MG Chemicals 419C conformal coating spray` | 1 | $8 |
| 30 | Multimeter | `digital multimeter voltage current AC DC` | 1 | $10 |
| 31 | Pi 4B heatsink kit | `Raspberry Pi 4 heatsink kit aluminum` | 1 | $3 |
| 32 | Foam weatherstripping tape 10mm | `foam weatherstrip tape self adhesive 10mm` | 1 | $2 |
| 33 | Prototype HAT for Pi 4B | `Raspberry Pi 4 prototype HAT GPIO expansion` | 1 | $4 |
| 34 | Shielded cable 4-core 22AWG | `shielded cable 4 core 22AWG signal` | 3m | $4 |
| 35 | Thermal fuse 10A 72°C | `thermal fuse 10A 72 degree` | 5 | $2 |

### Important notes

- **Pi 4B (#1):** Confirm 2GB version, seller with 1000+ orders.
- **M.2 SATA adapter (#2):** Must specify **SATA** not NVMe — electrically incompatible despite same connector. User already has the drive.
- **DS18B20 sensors (#3):** Buy 4 — two per zone in water housings. Water thermal lag is intentional. Use shielded cable (#34) for runs and add 100Ω series resistor near GPIO 4.
- **Contactor (#4):** Search by model number — Schneider LC1D09, 230V AC coil. Fails **open (OFF)** — safe failure mode. Do not substitute an SSR — cheap AliExpress SSRs fail **closed (ON)**.
- **Off-board relay modules (#5):** Buy 2. Module 1 drives the 230V contactor coil. Module 2 switches the PTC heater. All 230V switching off-board — mains never on the custom PCB. Check logic polarity (most cheap modules are active-LOW) and document before wiring.
- **SPDT relay (#6):** Hardware interlock — VCC power to relay modules is routed through SPDT contacts so only one module can be energised at a time regardless of GPIO state.
- **Fans (#7):** All four identical 120mm 4-pin PWM. Buy all four now. Exhaust fan installed at build time, inactive until v1.7.
- **LED strip (#10):** IP65 silicone-coated for condensation resistance. Warm white (~3000K) recommended. 1m is sufficient for the freezer section.
- **IRLB8721 MOSFET (#11):** Drives the LED strip from GPIO. N-channel, TO-220 package, rated well above the strip's current draw.
- **Mean Well PSUs (#12, #13):** Enclosed DIN-rail units replace bare-board PSU modules. Industrial-grade isolation, vibration tolerant, certified. HDR-15-12 for fans and LED strip, HDR-15-5 for Pi. Worth the cost — cheap open-frame PSUs are the single biggest real-world reliability risk in builds like this.
- **2N3904 transistors (#21):** One per fan PWM output. Open-collector stage ensures correct PWM signal for PC fans (5V open-drain as per Intel PWM spec).
- **1N4007 diodes (#22):** Flyback protection across relay coils and LED strip.
- **Shielded cable (#34):** Use for all DS18B20 sensor runs. Twisted pair with shield reduces 1-Wire bus instability near compressor and fan motors.
- **Thermal fuse (#35):** Wire in series with PTC heater as additional protection. 72°C rating trips before anything dangerous.
- **Conformal coating (#29):** Apply to all PCBs before installation.

---

## Component Details

### Raspberry Pi 4B

Quad-core 1.8GHz ARM Cortex-A72, 2GB RAM, built-in WiFi and Bluetooth, 40-pin GPIO. Boots from M.2 SATA SSD on USB 3.0. Powered via 5V GPIO pins from Mean Well HDR-15-5. Fan speeds via pigpio DMA PWM. Fit heatsink kit.

### Samsung control board — bypassed

The Samsung fridge's control board is completely disconnected. The Pi has exclusive control over all fridge functions. This eliminates all risk of firmware conflicts, thermistor interference, and defrost timing surprises. The Pi manages the defrost cycle directly (see Control Logic). The Samsung display and door indicator lights are no longer active — the fridge section retains its original lighting, the freezer section gets a Pi-controlled LED strip.

### M.2 SATA SSD (user supplied) + USB adapter

Boots and runs from M.2 SATA SSD via USB 3.0. Eliminates SD card wear. SQLite written directly at any frequency.

**One-time setup:**
1. Flash Raspberry Pi OS Lite 64-bit to SSD on a desktop PC
2. Boot Pi once from a temporary SD card
3. `raspi-config` → Advanced → Bootloader → USB Boot
4. Reboot, remove SD card permanently

### DS18B20 temperature sensors

1-Wire protocol, all four chained on GPIO 4 with one shared 4.7kΩ pull-up and a 100Ω series resistor near the GPIO pin. Each sensor has a unique 64-bit ROM address. Two per zone in ~100ml sealed water containers. Use shielded 4-core cable for all sensor runs — the compressor and fans create electrical noise that can destabilise the 1-Wire bus on long unshielded cables. Readings validated before use.

### DIN-rail mechanical contactor (compressor)

Schneider LC1D09, 230V AC coil, driven by off-board relay module #1. Pi GPIO sends 3.3V signal to relay module which switches the 230V coil. Fails **open (OFF)**. Mount on DIN rail. The Samsung control board's compressor wiring is disconnected and replaced with this contactor circuit.

### SPDT relay (hardware interlock) — revised implementation

The SPDT relay gates **VCC power to the relay modules** rather than their signal inputs. This ensures genuine hardware exclusivity regardless of GPIO state:

- SPDT COM → 5V VCC supply
- SPDT NO → VCC of relay module #2 (heater) — heater relay only receives power when SPDT is in heating position
- SPDT NC → VCC of relay module #1 (contactor coil) — compressor relay only receives power when SPDT is in cooling position

Both relay modules share GND and receive GPIO signals normally. But only the powered module can energise its relay. Even if both GPIO outputs are simultaneously HIGH (software bug), only one module has power to act on it. Genuinely hardware-enforced.

### Off-board relay modules

Two enclosed relay modules, all 230V switching off PCB:
- **Module 1:** Contactor coil driver (GPIO 17 → signal, powered via SPDT NC)
- **Module 2:** PTC heater (GPIO 27 → signal, powered via SPDT NO)

Check logic polarity before wiring — most cheap modules are **active-LOW** (LOW signal energises relay). Document which polarity your specific modules use and set GPIO initial states accordingly at boot.

### Mean Well PSUs

Enclosed DIN-rail switching PSUs. Industrial-grade isolation, certified, vibration-tolerant — appropriate for permanent installation in a machine compartment near a running compressor. Bare-board open-frame PSUs vary widely in isolation quality and are the most likely single point of failure in builds like this.

- **HDR-15-12 (12V, 1.25A):** Powers all four fans and the freezer LED strip
- **HDR-15-5 (5V, 3A):** Powers the Pi via GPIO 5V pins

### PTC ceramic heater 100W + thermal fuse

Self-regulating 230V ceramic element with a 72°C thermal fuse wired in series. The PTC element cannot overheat by design, but the thermal fuse provides independent backup protection. Switched by off-board relay module #2.

### Fans

Four 120mm 12V 4-pin PWM brushless fans. Speed controlled via pigpio DMA PWM at 25kHz through a 2N3904 open-collector transistor stage per fan. The transistor stage provides correct open-drain PWM signal as per the Intel PWM fan specification — more robust than driving fan PWM directly from Pi GPIO.

**Per-fan transistor circuit:**
```
Pi GPIO → 1kΩ → 2N3904 base
2N3904 emitter → GND
2N3904 collector → fan blue (PWM) wire
Fan blue wire also pulled up internally to 5V inside fan
Fan red → 12V PSU
Fan black → GND
Fan yellow (tach) → not connected
```

### Freezer LED strip

12V warm white IP65 silicone-coated LED strip mounted inside the freezer section. Driven by an IRLB8721 N-channel MOSFET on the PCB. Triggered by the freezer door sensor — on when the door opens, off when it closes. IP65 rating provides condensation resistance.

**MOSFET circuit:**
```
Pi GPIO 20 → 1kΩ → IRLB8721 gate
IRLB8721 source → GND
IRLB8721 drain → LED strip negative
LED strip positive → 12V PSU
1N4007 diode across strip (cathode to 12V)
```

### Displays

Three 1.3" color TFT ST7789 at 240×240. Share SPI bus with individual chip selects. Driven by `st7789` Python library.

### Door sensors

Factory reed switches on both doors. GPIO inputs with 10kΩ pull-up on PCB. Normally closed (LOW). Door open → HIGH. Splice into original wires without cutting. Freezer door sensor (GPIO 5) additionally drives the LED strip MOSFET.

---

## Custom PCB

> **Phase 2.** Design after prototype is fully proven. PCB carries **low-voltage signals only** — 3.3V, 5V, 12V. No 230V mains under any circumstances.

HAT-style board on Pi 4B 40-pin GPIO header. All external connections via screw terminals.

### GPIO state table

Document startup, safe, and boot transient states. Pi GPIOs float briefly at boot before software initialises them — pull-downs on relay signal outputs prevent brief relay energisation during boot.

| GPIO | Function | Safe state | Boot pull | Active-LOW modules? |
|---|---|---|---|---|
| 17 | Relay module #1 signal (contactor) | LOW | Pull-down | Yes — LOW energises |
| 27 | Relay module #2 signal (heater) | LOW | Pull-down | Yes — LOW energises |
| 22 | SPDT relay signal | LOW | Pull-down | — |
| 12 | Duct fan PWM | LOW (0%) | Pull-down | — |
| 13 | Freezer circ fan PWM | LOW (0%) | Pull-down | — |
| 19 | Fridge circ fan PWM | LOW (0%) | Pull-down | — |
| 21 | Exhaust fan PWM | LOW (0%) | Pull-down | — |
| 20 | Freezer LED MOSFET | LOW (off) | Pull-down | — |
| 5 | Freezer door sensor | Input | Pull-up | — |
| 6 | Fridge door sensor | Input | Pull-up | — |

> **Active-LOW relay modules:** if your relay modules energise on LOW signal, the safe state (relay off) requires the GPIO to be HIGH. Verify your specific module polarity before wiring and invert GPIO logic if needed. The GPIO state table above assumes active-LOW — adjust if modules differ.

### What the PCB provides

- 4× DS18B20 sensor terminals (3-pin: VCC, GND, DATA) with 4.7kΩ pull-up and 100Ω series resistor near GPIO 4
- 4× fan PWM output terminals (2-pin) with 2N3904 open-collector transistor stages and 1kΩ base resistors
- Relay module #1 signal terminal (3.3V, pull-down to safe state)
- Relay module #2 signal terminal (3.3V, pull-down to safe state)
- SPDT relay signal terminal (3.3V)
- IRLB8721 MOSFET circuit for LED strip with 1kΩ gate resistor and 1N4007 flyback diode
- 2× door sensor terminals (2-pin) with 10kΩ pull-ups
- 3× display chip select terminals
- 5V rail breakout terminal
- Status LED with 330Ω resistor
- 100nF decoupling caps on 3.3V and 5V rails
- M3 mounting holes at corners

### PCB component list

| Component | Part | Qty |
|---|---|---|
| GPIO connector | 2×20 female header 2.54mm | 1 |
| Sensor terminals | 3-pin screw terminal 3.5mm | 4 |
| Fan PWM terminals | 2-pin screw terminal 3.5mm | 4 |
| Relay #1 signal terminal | 2-pin screw terminal 3.5mm | 1 |
| Relay #2 signal terminal | 2-pin screw terminal 3.5mm | 1 |
| SPDT relay signal terminal | 2-pin screw terminal 3.5mm | 1 |
| LED strip output terminal | 2-pin screw terminal 3.5mm | 1 |
| Door sensor terminals | 2-pin screw terminal 3.5mm | 2 |
| Display CS terminals | 2-pin screw terminal 3.5mm | 3 |
| 5V rail terminal | 2-pin screw terminal 3.5mm | 1 |
| Fan open-collector transistors | 2N3904 NPN | 4 |
| LED MOSFET | IRLB8721 N-channel TO-220 | 1 |
| Flyback diodes | 1N4007 | 2 |
| Signal resistors (fan base + gate) | 1kΩ 1/4W | 5 |
| Door pull-up resistors | 10kΩ 1/4W | 2 |
| GPIO pull-down resistors (relay signals) | 10kΩ 1/4W | 3 |
| 1-Wire pull-up | 4.7kΩ 1/4W | 1 |
| 1-Wire series resistor | 100Ω 1/4W | 1 |
| LED resistor | 330Ω 1/4W | 1 |
| Status LED | 5mm green LED | 1 |
| Decoupling caps | 100nF ceramic | 4 |
| Mounting hardware | M3 standoffs + screws | 4 |

### Fuse map

| Fuse | Rating | Type | Protects |
|---|---|---|---|
| Fuse 1 | 5A slow-blow | Inline holder | 12V Mean Well HDR-15-12 mains feed |
| Fuse 2 | 5A slow-blow | Inline holder | 5V Mean Well HDR-15-5 mains feed |
| Thermal fuse | 10A 72°C | In-line with heater | PTC heater secondary protection |

### PCB design and ordering

1. **KiCad** — free, professional. ~28 through-hole components.
2. **DRC** — run before export. Pay attention to clearance rules.
3. **Export Gerbers** — standard Gerber + drill files.
4. **JLCPCB** — 5 boards ~$15 USD, 1–2 weeks.
5. **Test outside enclosure** before fitting to Pi. Verify all GPIO pull states at power-on before connecting relay modules.

---

## Machine Compartment Enclosure

Bottom rear of Samsung fridge behind kick plate. Ambient temperature, ventilated, direct access to compressor wiring. Samsung control board disconnected and removed.

### Contents

- Pi 4B with PCB HAT, M.2 SATA adapter on USB 3.0
- DIN rail with mechanical contactor and Mean Well HDR-15-12
- Mean Well HDR-15-5
- Off-board relay module #1 (contactor coil)
- Off-board relay module #2 (heater)
- SPDT relay module
- Inline fuse holders on both PSU mains feeds
- Mains terminal block

### Printed enclosure design

PETG, 40%+ infill, 4+ perimeters, M3 heat-set inserts.

- DIN rail mount for contactor and Mean Well PSUs
- Standoff mounts for Pi+PCB
- Off-board relay module mounts
- SPDT relay module mount
- Contactor body exposed through cutout in PETG wall — PETG never acts as heatsink
- Fuse holders accessible without full disassembly
- **E-stop maintenance switch:** mains-side physical kill switch mounted on outside of enclosure, accessible without opening lid, interrupts live feed before all other components. Label clearly.
- Mains cable entry with grommet on one face, low-voltage exits on opposite face
- Ventilation slots on sides and top
- M3 screw lid into heat-set inserts
- All external connections labelled in embossed text
- Mains and low-voltage wiring on separate sides, never bundled

Apply conformal coating to all PCBs before installation.

---

## Wiring

### GPIO pin assignment

```
# 1-Wire sensors
GPIO 4   →  DS18B20 data (4.7kΩ pull-up + 100Ω series on PCB)

# Mains switching signals (3.3V to relay modules, pull-downs on PCB)
GPIO 17  →  Relay module #1 signal (contactor coil)
GPIO 27  →  Relay module #2 signal (heater)
GPIO 22  →  SPDT relay signal

# PWM fans — pigpio DMA PWM via 2N3904 open-collector stages
GPIO 12  →  Duct fan PWM         (120mm, bottom of divider)
GPIO 13  →  Freezer circ fan PWM (120mm, freezer zone)
GPIO 19  →  Fridge circ fan PWM  (120mm, fridge zone)
GPIO 21  →  Exhaust fan PWM      (120mm, top of divider — v1.7)

# Freezer LED strip
GPIO 20  →  IRLB8721 gate via 1kΩ (freezer LED strip)

# Door sensors (HIGH when open, 10kΩ pull-up on PCB)
GPIO 5   →  Freezer door reed switch
GPIO 6   →  Fridge door reed switch

# SPI displays (shared bus)
GPIO 11  →  SCLK
GPIO 10  →  MOSI
GPIO 24  →  DC   (shared)
GPIO 25  →  RST  (shared)
GPIO 8   →  Screen 1 CS (CE0)
GPIO 7   →  Screen 2 CS (CE1)
GPIO 26  →  Screen 3 CS (software)

# Status + reserved
GPIO 23  →  Status LED via 330Ω
GPIO 16  →  RESERVED — bubble sensor interrupt (v1.4)
```

### SPDT interlock wiring — hardware VCC gating

```
5V supply → SPDT COM
SPDT NO   → VCC of relay module #2 (heater)
SPDT NC   → VCC of relay module #1 (contactor coil)
GPIO 22   → SPDT relay coil signal

GPIO 22 HIGH: SPDT energised → module #1 powered (cooling active)
              module #2 unpowered (heater cannot energise)
GPIO 22 LOW:  SPDT de-energised → module #2 powered (heating available)
              module #1 unpowered (compressor cannot energise)
```

Both modules still receive GPIO signal inputs but only the powered module can act on them. Hardware-enforced regardless of software state.

### Contactor drive wiring

```
GPIO 17 → relay module #1 signal IN (3.3V)
Relay module #1 VCC ← from SPDT NC contact
Relay module #1 NO  → contactor coil A1 (230V AC)
Relay module #1 COM → 230V live (fused)
Contactor coil A2   → neutral
Contactor L1        → 230V live in (fused, after e-stop switch)
Contactor T1        → 230V live to compressor
```

### Heater wiring

```
GPIO 27 → relay module #2 signal IN (3.3V)
Relay module #2 VCC ← from SPDT NO contact
Relay module #2 NO  → thermal fuse → PTC heater live
Relay module #2 COM → 230V live (fused, after e-stop switch)
PTC heater neutral  → neutral
```

### Sensor wiring

```
DS18B20 red    → 3.3V
DS18B20 black  → GND
DS18B20 yellow → 100Ω series → GPIO 4 (4.7kΩ pull-up to 3.3V on PCB)
Use shielded cable. Connect shield to GND at PCB end only.
All four in parallel on same bus.
```

### Fan wiring (per fan, via 2N3904 stage on PCB)

```
Pi GPIO → 1kΩ → 2N3904 base
2N3904 emitter → GND
2N3904 collector → fan blue (PWM) wire
Fan red  → 12V PSU
Fan black → GND
Fan yellow (tach) → not connected
```

### Freezer LED strip wiring

```
GPIO 20 → 1kΩ → IRLB8721 gate
IRLB8721 source → GND
IRLB8721 drain  → LED strip negative
LED strip positive → 12V PSU
1N4007 across strip (cathode to 12V, anode to drain)
```

---

## 3D Printed Parts

Print all parts in PETG. PLA becomes brittle in the garage environment.

### Machine compartment enclosure

See Machine Compartment Enclosure section. 40%+ infill, 4+ perimeters, M3 heat-set inserts. Include E-stop switch mount on exterior.

### Duct shroud and frame (×2)

Same design printed twice — duct fan (bottom, cold air in) and exhaust fan (top, warm air out). 120mm fan clip retention. Flanged lips seal with foam weatherstripping. Include drip lip and drain path at the bottom of each shroud for condensation management — especially important for v1.7 cold crash mode when the temperature differential creates significant condensation. Design for tool-free fan removal for periodic cleaning. 40%+ infill, 3+ perimeters.

### Circulation fan brackets (×2)

Clip-on mounts for 120mm fans inside each zone. No drilling. Fridge circ fan aimed at widest point of fermenter.

### Sensor water housings (×2)

Sealed cylinders, ~100ml each. Press-fit cap with sealed cable port. 60%+ infill, 4+ perimeters. Seal with food-safe silicone. Fill with distilled water.

### PTC heater bracket

Wall-mount for PTC heater in fridge section. Holds heater away from wall for airflow clearance.

### Display bezels (×3)

Screen 1 on fridge exterior with VHB tape. Screens 2 and 3 at tap positions. Include cable channel.

### Cable routing clips

Saddle clips with 3M VHB tape for routing cables along fridge walls.

### Cable pass-through plates

Cover plates over wall penetrations. Accepts M12 cable glands.

### LED strip channel

A printed channel or clip system to hold the LED strip neatly along the top or side of the freezer interior. Routes the cable to the M12 cable gland pass-through.

### Future parts

**Keg load cell platform (v1.1):** Sled per corny keg, four 50kg load cell recesses, ~230mm diameter alignment lips. 60%+ infill.

**Keg insulation jacket (v1.5):** PETG outer shell + 25mm XPS foam per corny keg. Houses PTC heater strip and DS18B20 surface sensor.

---

## Control Logic

### Dependencies

```bash
sudo apt install pigpio python3-pigpio
sudo systemctl enable pigpiod
sudo systemctl start pigpiod
```

### Python pseudocode

```python
import time
import pigpio
import RPi.GPIO as GPIO
from datetime import datetime, timezone
import threading

# All thresholds from config — no magic numbers in code
FREEZER_TARGET      = 1.5
HYSTERESIS_FREEZE   = 0.5
COMPRESSOR_MIN_OFF  = 600      # seconds
DOOR_SETTLE_SECS    = int(config["doors"]["settle_seconds"])
DOOR_ALERT_SECS     = int(config["doors"]["alert_minutes"]) * 60
DEFROST_INTERVAL    = int(config["defrost"]["interval_hours"]) * 3600
DEFROST_DURATION    = int(config["defrost"]["duration_minutes"]) * 60

# GPIO
CONTACTOR_RELAY  = 17
HEATER_RELAY     = 27
SPDT_INTERLOCK   = 22
PWM_DUCT         = 12
PWM_FREEZE_CIRC  = 13
PWM_FRIDGE_CIRC  = 19
PWM_EXHAUST      = 21
FREEZER_LED      = 20
DOOR_FREEZER     = 5
DOOR_FRIDGE      = 6
STATUS_LED       = 23
WATCHDOG_PATH    = '/dev/watchdog'
PWM_FREQ         = 25000

# pigpio DMA PWM
pi = pigpio.pi()

def set_fan_speed(pin, percent):
    pi.set_PWM_frequency(pin, PWM_FREQ)
    pi.set_PWM_dutycycle(pin, int(percent * 2.55))

def fans_off():
    for pin in [PWM_DUCT, PWM_FREEZE_CIRC, PWM_FRIDGE_CIRC, PWM_EXHAUST]:
        set_fan_speed(pin, 0)

# Hardware watchdog — kicked from background thread
watchdog_fd = open(WATCHDOG_PATH, 'w')
watchdog_alive = True

def watchdog_thread():
    while watchdog_alive:
        watchdog_fd.write('1')
        watchdog_fd.flush()
        time.sleep(10)  # kick every 10s, watchdog timeout is 15s

threading.Thread(target=watchdog_thread, daemon=True).start()

# Safe GPIO initialisation — set outputs to safe state before configuring
# Active-LOW relay modules: HIGH = relay off (safe), LOW = relay on
GPIO.setmode(GPIO.BCM)
for pin in [CONTACTOR_RELAY, HEATER_RELAY, SPDT_INTERLOCK, FREEZER_LED]:
    GPIO.setup(pin, GPIO.OUT, initial=GPIO.HIGH)  # adjust if active-HIGH modules
for pin in [PWM_DUCT, PWM_FREEZE_CIRC, PWM_FRIDGE_CIRC, PWM_EXHAUST]:
    GPIO.setup(pin, GPIO.OUT, initial=GPIO.LOW)
for pin in [DOOR_FREEZER, DOOR_FRIDGE]:
    GPIO.setup(pin, GPIO.IN, pull_up_down=GPIO.PUD_UP)

def get_last_compressor_off():
    row = db.execute("SELECT last_compressor_off FROM system").fetchone()
    return row["last_compressor_off"] if row else 0

def set_compressor(on):
    if on:
        GPIO.output(HEATER_RELAY, GPIO.HIGH)   # heater off (active-LOW)
        time.sleep(0.05)
        GPIO.output(SPDT_INTERLOCK, GPIO.HIGH) # SPDT to cooling position
        GPIO.output(CONTACTOR_RELAY, GPIO.LOW) # contactor on (active-LOW)
    else:
        GPIO.output(CONTACTOR_RELAY, GPIO.HIGH) # contactor off
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)
        db.execute("UPDATE system SET last_compressor_off = ?", (time.time(),))

def set_heater(on):
    if on:
        GPIO.output(CONTACTOR_RELAY, GPIO.HIGH) # compressor off
        time.sleep(0.05)
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)   # SPDT to heating position
        GPIO.output(HEATER_RELAY, GPIO.LOW)     # heater on (active-LOW)
    else:
        GPIO.output(HEATER_RELAY, GPIO.HIGH)    # heater off

def handle_freezer_light():
    if door_is_open(DOOR_FREEZER):
        GPIO.output(FREEZER_LED, GPIO.HIGH)
    else:
        GPIO.output(FREEZER_LED, GPIO.LOW)

def read_validated_sensor(sensor_id):
    try:
        value = read_sensor(sensor_id)
        if value is None or value < -55 or value > 125:
            log_error(f"Invalid reading from {sensor_id}: {value}")
            post_discord("🌡️ Sensor error",
                f"Invalid reading from {sensor_id}: {value}°C",
                COLOR_ALERT)
            return None
        return value
    except Exception as e:
        log_error(f"Sensor read exception {sensor_id}: {e}")
        return None

def read_zone_temp(sensor_a, sensor_b):
    a = read_validated_sensor(sensor_a)
    b = read_validated_sensor(sensor_b)
    if a is None and b is None:
        return None
    if a is None: return b
    if b is None: return a
    if abs(a - b) > 2.0:
        post_discord("🌡️ Sensor divergence",
            f"{sensor_a}/{sensor_b} differ by {abs(a-b):.1f}°C",
            COLOR_WARN)
    return (a + b) / 2

def defrost_due():
    last = db.execute("SELECT last_defrost FROM system").fetchone()["last_defrost"]
    return (time.time() - last) > DEFROST_INTERVAL

def run_defrost():
    """Stop compressor and fans, allow evaporator to thaw naturally."""
    post_discord("🧊 Defrost starting",
        "Compressor paused for evaporator defrost cycle.", COLOR_INFO)
    set_compressor(False)
    fans_off()
    db.execute("UPDATE system SET defrost_active = 1")
    time.sleep(DEFROST_DURATION)
    db.execute(
        "UPDATE system SET last_defrost = ?, defrost_active = 0",
        (time.time(),)
    )
    post_discord("✅ Defrost complete", "Resuming normal operation.", COLOR_OK)

def control_loop():
    handle_freezer_light()
    handle_door_events()
    check_for_step_change()

    # Run defrost if due — suspends control loop for defrost duration
    if defrost_due():
        run_defrost()
        return

    if any_door_open_or_settling():
        # Fans paused during door events.
        # Compressor and heater intentionally left in current state —
        # do not interrupt a running compressor cycle for a brief door open.
        fans_off()
        return

    t_freezer = read_zone_temp('freezer_1', 'freezer_2')
    t_fridge  = read_zone_temp('fridge_1',  'fridge_2')

    if t_freezer is None or t_fridge is None:
        set_compressor(False)
        set_heater(False)
        fans_off()
        return

    fridge_target = get_fridge_target()

    # --- Freezer / compressor ---
    freeze_error = t_freezer - FREEZER_TARGET
    last_off = get_last_compressor_off()
    if freeze_error > HYSTERESIS_FREEZE:
        if time.time() - last_off > COMPRESSOR_MIN_OFF:
            set_compressor(True)
            set_fan_speed(PWM_FREEZE_CIRC, fan_speed(abs(freeze_error)))
    elif freeze_error < -HYSTERESIS_FREEZE:
        set_compressor(False)
        set_fan_speed(PWM_FREEZE_CIRC, 0)

    # --- Fridge / fermentation ---
    fridge_error = t_fridge - fridge_target
    if fridge_error > 0.3:
        set_heater(False)
        set_fan_speed(PWM_DUCT,        fan_speed(fridge_error))
        set_fan_speed(PWM_FRIDGE_CIRC, fan_speed(fridge_error))
        set_fan_speed(PWM_EXHAUST, 0)
    elif fridge_error < -0.3:
        set_fan_speed(PWM_DUCT, 0)
        set_fan_speed(PWM_EXHAUST, 0)
        set_heater(True)
        set_fan_speed(PWM_FRIDGE_CIRC, 50)
    else:
        set_fan_speed(PWM_DUCT, 0)
        set_fan_speed(PWM_EXHAUST, 0)
        set_heater(False)
        set_fan_speed(PWM_FRIDGE_CIRC, 0)

    log(time.time(), t_freezer, t_fridge, fridge_target)
    time.sleep(10)  # 10s loop — watchdog kicks every 10s on background thread

def fan_speed(error_c):
    if error_c < 0.3:   return 0
    elif error_c < 1.0: return 30
    elif error_c < 2.0: return 60
    else:               return 100
```

### Safety rules

- Compressor 10-minute lockout persisted to SQLite — survives reboots
- SPDT relay gates VCC to relay modules — true hardware interlock regardless of GPIO state
- `set_compressor()` explicitly writes heater GPIO to safe state before switching SPDT
- `set_heater()` explicitly writes compressor GPIO to safe state before switching SPDT
- Watchdog kicked every 10s from background thread — Pi reboots after 15s hang
- Door-open: fans paused, compressor and heater intentionally left unchanged (explicit comment)
- Zone sensor total failure: all outputs go safe
- GPIO outputs initialised to safe state before configuring as outputs — no boot transients
- Active-LOW relay polarity documented in GPIO state table
- Defrost cycle managed by Pi — evaporator coil kept clear
- Thermal fuse in heater circuit — independent of software

---

## Software Stack

### OS and runtime

- **Raspberry Pi OS Lite 64-bit** — headless, boots from M.2 SATA SSD
- **Python 3** with `pigpio`, `RPi.GPIO`, `w1thermsensor`, `sqlite3`, `st7789`, `Flask`, `requests`
- Control loop as `systemd` service with hardware watchdog
- `pigpiod` running as system service

### Hardware watchdog setup

```bash
# /boot/config.txt
dtparam=watchdog=on

# /etc/systemd/system.conf
RuntimeWatchdogSec=15
ShutdownWatchdogSec=2min
```

Watchdog kicked every 10 seconds from a background thread independent of the control loop. Loop sleep is also 10 seconds. If either the loop or the watchdog thread hangs, the Pi reboots within 15 seconds.

### Data storage

SQLite to M.2 SATA SSD with WAL mode enabled at startup:

```python
db.execute("PRAGMA journal_mode=WAL;")
```

WAL mode reduces lock contention, improves concurrent read safety, and provides better resilience against unclean shutdown.

```sql
-- System state
CREATE TABLE system (
    last_compressor_off REAL    DEFAULT 0,
    last_defrost        REAL    DEFAULT 0,
    defrost_active      INTEGER DEFAULT 0,
    biofine_notified    INTEGER DEFAULT 0
);

-- Temperature and state log
CREATE TABLE log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    ts              INTEGER NOT NULL,
    t_freezer       REAL,
    t_fridge        REAL,
    fridge_target   REAL,
    compressor      INTEGER,
    heater          INTEGER,
    duct_fan_pct    INTEGER,
    freeze_circ_pct INTEGER,
    fridge_circ_pct INTEGER,
    defrost_active  INTEGER
);
CREATE INDEX idx_log_ts ON log(ts);

-- Door event log
CREATE TABLE door_events (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    ts      INTEGER NOT NULL,
    door    TEXT,
    event   TEXT
);

-- Keg/beer information
CREATE TABLE kegs (
    tap               INTEGER PRIMARY KEY,
    beer_name         TEXT,
    style             TEXT,
    abv               REAL,
    ibu               INTEGER,
    brewed            TEXT,
    tapped            TEXT,
    last_pour         TEXT,
    last_clean        TEXT,
    full_weight_kg    REAL,
    current_weight_kg REAL,
    fining_added      TEXT,
    notes             TEXT
);

-- Active fermentation metadata
CREATE TABLE fermentation_meta (
    batch_id     TEXT,
    batch_name   TEXT,
    recipe_name  TEXT,
    recipe_style TEXT,
    pitch_date   TEXT,
    active       INTEGER DEFAULT 0
);

-- Fermentation steps from Brewfather
CREATE TABLE fermentation_profile (
    step_index    INTEGER PRIMARY KEY,
    step_name     TEXT,
    target_temp   REAL,
    duration_days REAL
);
```

### Beer style detection

```python
HAZY_STYLES = [
    "new england", "neipa", "hazy", "hefeweizen",
    "witbier", "wheat", "berliner", "gose"
]

def is_hazy_style(style):
    return any(s in style.lower() for s in HAZY_STYLES)
```

### Web dashboard (Flask)

Local web server at `http://kegerator.local`. Pages:
- Live temperatures, compressor/heater/fan states, door status, defrost status
- Temperature history graph
- Keg cards: beer info, % remaining, last pour, last clean, fining status
- Fermentation page: Brewfather connect, batch select, profile start
- Settings: credentials, fallback temp, Discord webhook, door thresholds, defrost interval and duration

### Config file

```ini
# /etc/kegerator/config.ini

[brewfather]
user_id       = your_user_id
api_key       = your_api_key

[fermentation]
fallback_temp = 18.0

[discord]
webhook_url   = https://discord.com/api/webhooks/...

[doors]
alert_minutes  = 5
settle_seconds = 45

[defrost]
interval_hours   = 8
duration_minutes = 25
```

### Database backup

```bash
# /etc/cron.daily/kegerator-backup
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/mnt/backup"
cp /var/lib/kegerator/kegerator.db "$BACKUP_DIR/kegerator-$DATE.db"
find "$BACKUP_DIR" -name "kegerator-*.db" -mtime +30 -delete
```

---

## Brewfather Integration

Connects to Brewfather only when triggered from the web UI. Profile runs from local SQLite after import.

### On fermentation start

```python
def on_fermentation_start(batch):
    style = batch["recipe"]["style"]["name"]
    if is_hazy_style(style):
        post_discord("🌫️ Hazy style detected",
            f"**{batch['name']}** — {style}\n"
            "No fining agents needed. Cold crash optional.",
            COLOR_INFO)
    else:
        post_discord("🔬 Clear style — add Clarity Ferm at pitching",
            f"**{batch['name']}** — {style}\n"
            "Add Clarity Ferm to wort after cooling, before pitching.\n"
            "Available at brew.is (1,400 kr). Vegan.",
            COLOR_INFO)
```

### Step change detection

```python
last_step_index = None

def check_for_step_change():
    global last_step_index
    step = get_current_step()
    if step is None or step["step_index"] == last_step_index:
        return
    last_step_index = step["step_index"]
    meta = db.execute(
        "SELECT batch_name, recipe_style FROM fermentation_meta WHERE active=1"
    ).fetchone()
    is_cold_crash = step["target_temp"] <= 4.0
    if is_cold_crash:
        post_discord("❄️ Cold crash starting",
            f"**{meta['batch_name']}** dropping to {step['target_temp']}°C\n"
            "Check blowoff tube is submerged in sanitiser.",
            COLOR_INFO)
    else:
        post_discord("📈 Fermentation step change",
            f"**{meta['batch_name']}** → {step['step_name']} at "
            f"{step['target_temp']}°C for {step['duration_days']} days",
            COLOR_WARN)
```

### Target temperature calculation

```python
def get_fridge_target():
    meta = db.execute(
        "SELECT pitch_date FROM fermentation_meta WHERE active=1"
    ).fetchone()
    if not meta:
        return float(config["fermentation"]["fallback_temp"])
    pitch_date = datetime.fromisoformat(meta["pitch_date"])
    days_fermenting = (
        (datetime.now(timezone.utc) - pitch_date).total_seconds() / 86400
    )
    steps = db.execute(
        "SELECT * FROM fermentation_profile ORDER BY step_index"
    ).fetchall()
    elapsed = 0.0
    for step in steps:
        elapsed += step["duration_days"]
        if days_fermenting < elapsed:
            return step["target_temp"]
    return steps[-1]["target_temp"]
```

### Biofine Clear notification (clear styles, v1.3+)

```python
def check_crash_fining_trigger():
    meta = db.execute(
        "SELECT batch_name, recipe_style FROM fermentation_meta WHERE active=1"
    ).fetchone()
    if not meta or is_hazy_style(meta["recipe_style"]):
        return
    already_notified = db.execute(
        "SELECT biofine_notified FROM system"
    ).fetchone()["biofine_notified"]
    if already_notified:
        return
    tilt_temp = get_tilt_temperature()
    if tilt_temp is not None:
        beer_temp = tilt_temp
        temp_source = f"Tilt: {tilt_temp:.1f}°C"
    else:
        beer_temp = read_zone_temp('fridge_1', 'fridge_2')
        temp_source = f"Fridge sensor: {beer_temp:.1f}°C"
    gravity_stable = gravity_unchanged_for_hours(24)
    if beer_temp is not None and beer_temp <= 2.5 and gravity_stable:
        post_discord("🍺 Add Biofine Clear now",
            f"**{meta['batch_name']}** has reached crash temp ({temp_source})\n"
            f"Gravity stable — fermentation complete.\n\n"
            f"**Closed transfer — no oxygen contact:**\n"
            f"1. Measure 1–2 ml Biofine Clear per litre\n"
            f"2. Dilute 1:1 with CO₂-purged water in sanitised vessel\n"
            f"3. Purge vessel headspace with CO₂\n"
            f"4. Connect to fermenter gas-in port\n"
            f"5. Push gently with CO₂ — fermenter stays sealed\n"
            f"6. Confirm addition in dashboard to start 48hr countdown",
            COLOR_INFO)
        db.execute("UPDATE system SET biofine_notified = 1")
```

---

## Discord Notifications

> **v1.3 feature.** Silently skipped if webhook not configured.

### What gets posted

- **Fermentation start:** Style detection — Clarity Ferm for clear styles, skip for hazy
- **Step changes:** New target and duration. Cold crash includes blowoff reminder
- **Biofine Clear (clear styles, v1.3+):** Closed-transfer instructions at crash temp
- **Transfer ready:** 48 hours after Biofine confirmed
- **Defrost:** Cycle start and completion
- **Doors:** Left open beyond threshold
- **Sensor errors:** Invalid reading or zone failure
- **System:** Pi restarted, compressor running unusually long
- **Daily digest:** Both zone temps, fermentation step progress, keg levels (v1.1+)

### Implementation

```python
def post_discord(title, message, color=0x3498db):
    webhook_url = config["discord"].get("webhook_url")
    if not webhook_url:
        return
    try:
        requests.post(webhook_url, json={
            "embeds": [{"title": title, "description": message, "color": color}]
        }, timeout=5)
    except requests.RequestException:
        pass

COLOR_INFO  = 0x3498db
COLOR_OK    = 0x2ecc71
COLOR_WARN  = 0xe67e22
COLOR_ALERT = 0xe74c3c
```

---

## Physical Installation

### Machine compartment

Remove kick plate. Disconnect and remove Samsung control board. Mount printed enclosure. M.2 SATA adapter on Pi USB 3.0. E-stop switch on enclosure exterior. Route all cables before closing lid. Mean Well PSUs and contactor on DIN rail.

### Divider wall — two 125mm holes

- **Bottom:** Duct fan, cold air from freezer into fridge
- **Top:** Exhaust fan, warm air out (v1.7, installed at build time)

Cold air enters at bottom, warm air exits at top. Seal shroud flanges with foam weatherstripping. Passive return slot on opposite side of divider for normal operation. Drip lips and drain paths in printed shrouds for condensation management.

### Circulation fans

Freezer fan: low on side wall, horizontal airflow across keg zone bottom. Fridge fan: high on back wall, aimed at widest point of fermenter.

### Sensor placement

Freezer water housing: mid-height between kegs, away from evaporator and direct airflow. Fridge water housing: same height as fermenter body, away from duct fan outlet.

### Freezer LED strip

Mount along the top interior of the freezer section using the printed channel or clip system. Cable routes through M12 cable gland to machine compartment. Test door trigger before sealing cable gland.

### Door sensors

Locate factory reed switches near hinge area. Splice thin shielded cable without cutting original wires. Route along door frame into machine compartment.

### PTC heater

Back wall of fridge section at mid-height on printed bracket. Away from fermenter and circulation fan outlet.

### Display cables

Machine compartment → M12 cable glands → fridge zones. Screen 1 to fridge exterior. Screens 2 and 3 to tap positions.

### Cable discipline

Mains and low-voltage wiring physically separated throughout. M12 cable glands at every wall penetration. Sensor cables use shielded wire with shield grounded at PCB end only.

---

## Build Phases

### Phase 1 — Prototype

Pi 4B with prototype HAT and jumper wires. Samsung control board disconnected. Validate all GPIO, control loop, fans, displays, door sensors and LED strip, defrost cycle, Brewfather. **Do not design PCB until Phase 1 is fully stable.**

### Phase 2 — Custom PCB

KiCad design, JLCPCB order (5 boards ~$15). Verify GPIO pull states at power-on before connecting relay modules. Test SPDT VCC gating manually before connecting mains. Test outside enclosure first.

### Phase 3 — Machine compartment integration

Design PETG enclosure around confirmed PCB. Print, test fit. Apply conformal coating. Final cable runs, install, seal glands, replace kick plate.

---

## Roadmap

### v1.0 — MVP (this build)
- Samsung control board fully bypassed — Pi has exclusive control
- Temperature control: freezer 1–2°C, fridge follows Brewfather profile
- Pi-managed defrost cycle — configurable interval and duration, prevents evaporator icing
- DIN-rail contactor (compressor), two off-board relay modules, SPDT hardware interlock with VCC gating
- Mean Well enclosed DIN-rail PSUs — industrial-grade, vibration-tolerant
- 2N3904 open-collector transistor stage per fan — correct Intel PWM spec signal
- DS18B20 on shielded cable with 100Ω series resistor — 1-Wire bus stability
- Thermal fuse in series with PTC heater
- E-stop maintenance switch on enclosure exterior
- Fuse map: 5A slow-blow per PSU feed, 10A 72°C thermal fuse on heater
- GPIO state table documented — safe states and active-LOW polarity
- Compressor lockout persisted to SQLite — survives reboots
- Hardware watchdog via background thread, kicks every 10s, timeout 15s
- Inline fuse holders on both PSU mains feeds
- `set_compressor()` and `set_heater()` explicitly disable the other before switching
- pigpio DMA PWM for all four 120mm fans
- Sensor validation — -127 and out-of-range caught before use
- Dual DS18B20 per zone in water housings — averaging, divergence alerting, safe shutdown on failure
- Door sensors: fans paused on open (compressor/heater intentionally unchanged), settle window, log events
- Freezer LED strip: IRLB8721 MOSFET, door-triggered, IP65, warm white
- Brewfather: on-demand connect, batch select, manual profile start, style detection
- Step transitions use `total_seconds()` — immediate, not midnight granularity
- Style-aware notifications: Clarity Ferm for clear styles, skip for hazy
- `biofine_notified` persisted to SQLite
- `ts` uses AUTOINCREMENT — no same-second collision
- SQLite WAL mode enabled at startup
- All thresholds from config — no magic numbers
- Duct shroud drip lips and drain paths for condensation
- Daily SQLite backup via cron
- M.2 SATA SSD boot, direct SQLite writes
- Local Flask dashboard at `http://kegerator.local`

### v1.1 — Keg weight
- HX711 ADC + 50kg load cells per corny keg
- Printed load cell sleds, ~230mm corny base
- Live % remaining on displays and dashboard
- Pour volume estimation
- Low keg Discord notification

### v1.2 — Dashboard polish + compressor verification
- Temperature history graphs
- Compressor runtime tracking
- Days-since-clean alerts
- Pour history log
- Door open statistics
- SCT-013 current clamp on compressor live wire + ADS1115 ADC — detects commanded ON with no current (failed contactor, tripped overload) and commanded OFF with current present (failed-closed fault)

### v1.3 — Notifications, Tilt, remote access
- Full Discord suite: fermentation events, defrost notifications, Biofine Clear closed-transfer instructions (clear styles only), 48hr fining countdown, daily digest
- Tilt hydrometer over Bluetooth: gravity and temperature as supplementary data. Primary control runs from DS18B20. Biofine notification falls back to fridge sensor if Tilt unavailable. Handles Tilt stuck in krausen and dead battery gracefully
- Tailscale for secure remote access
- Push notifications for out-of-range temperature and overdue cleaning

### v1.4 — Bubble detection
- TCRT5000 IR sensor on blowoff tube, hardware interrupt on GPIO 16
- Bubble rate logged and graphed
- Discord: fermentation active, stuck fermentation, completion detection
- 3D printed black PETG clip-on housing — opaque to block ambient light

### v1.5 — Keg insulation jackets
- PETG + 25mm XPS foam jacket per corny keg
- PTC heater strip 12V 30W per jacket
- DS18B20 surface sensor per jacket
- Jacket heaters activate when freezer drops below configurable threshold

### v1.6 — Freeze-point calculation
- Load cell weight (v1.1) + Brewfather ABV → thermal mass per keg
- Corny tare weight 5kg: `liquid_kg = total_kg − 5.0`
- ABV-adjusted freeze point: `freeze_point_c = −0.42 × abv`
- Cooling rate calibrated from SQLite data after first crash
- Discord alert when safe window running short

### v1.7 — Cold crash mode
- Freezer drops to −18°C to −20°C during cold crash step
- Exhaust fan (top of divider) activates — forced air loop, thermodynamics aligned
- Keg jacket heaters (v1.5) protect kegs at serving temperature
- Freeze-point calculation (v1.6) monitors each keg dynamically
- Post-crash warmup: both divider fans pause until freezer returns to 1–2°C
- Discord: cold crash start, crash complete, warmup complete
- Estimated crash time 50L from 18°C to 2°C: 4–6 hours

### v1.8 — DIY glycol chilling (optional, complex)
- Dedicated secondary compressor (salvaged or 12V DC camping fridge unit)
- Evaporator coil in glycol reservoir (30–40% food-grade propylene glycol)
- 12V submersible pump circulates glycol through silicone coil around fermenter
- Estimated crash time: 45–90 minutes
- Treat as a separate dedicated project
