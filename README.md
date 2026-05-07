# Beer Fridge Controller — Full Build Spec

A Raspberry Pi 4B-based temperature controller for a dual-zone Samsung fridge used for beer fermentation (fridge section) and keg storage (freezer section). Electronics live in the machine compartment. Tap display screens show live keg info. Brewfather integration for fermentation profile management. Boots from M.2 SATA SSD via USB — no SD card.

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
| Freezer | Keg storage | 1–2 °C | Compressor via DIN-rail contactor |
| Fridge | Fermentation | Brewfather profile | Duct fan + PTC heater |

### How it works

The Samsung fridge has a single compressor. The freezer is held at 1–2 °C by switching the compressor on and off via a DIN-rail mechanical contactor with a 10-minute minimum cooldown to protect the motor. The fermentation section has no direct cooling — a duct fan blows cold air from the freezer into the fridge section when cooling is needed. A PTC ceramic heater handles warming when fermentation temperature drops below the Brewfather profile target. Both sections have circulation fans that run at proportional PWM speed whenever their zone is actively heating or cooling. All electronics live in the machine compartment at the bottom rear of the fridge. The system boots from an M.2 SATA SSD connected via USB 3.0 — no SD card is used.

### Fan layout

| Fan | Zone | Purpose | Control |
|---|---|---|---|
| Duct fan | Between zones | Transfers cold from freezer → fridge | PWM GPIO (proportional to temp error) |
| Circulation fan A | Freezer | Circulates air around kegs | PWM GPIO (mirrors compressor state) |
| Circulation fan B | Fridge | Circulates air around fermenter | PWM GPIO (mirrors heating/cooling state) |

### Switching layout

| Load | Switch type | Reason |
|---|---|---|
| Compressor | DIN-rail mechanical contactor | Reliable, fail-open (safe), sourced from reputable distributor |
| PTC heater | Off-board enclosed relay module | Mains kept entirely off custom PCB |
| Fans | PWM GPIO direct | Variable speed control, 12V only |
| Heater/cooler interlock | SPDT relay (hardware) | Physically prevents simultaneous heating and cooling regardless of software state |

### Screen layout

| Screen | Location | Shows |
|---|---|---|
| Screen 1 | Fridge exterior | Temps, compressor, heating, fan states, active fermentation step |
| Screen 2 | Tap 1 | Beer info, % remaining, last pour, last clean |
| Screen 3 | Tap 2 | Same for second beer |

### Door sensors

Both fridge doors have factory-fitted magnetic reed switches. These are tapped into GPIO inputs. When a door opens the control loop pauses all fan operation and ignores temperature readings for a settle period after closing. Door events are logged to SQLite and Discord alerts fire if a door is left open beyond a configurable threshold.

### Temperature sensors

Each zone has two DS18B20 waterproof probes submerged in sealed containers of water — one sensor per container. The water acts as a deliberate thermal mass buffer. With 19L corny kegs in the freezer and a 50L fermenter in the fridge section, the beer itself changes temperature very slowly. Sensors submerged in water accurately represent this thermal reality and prevent compressor short cycling from brief air temperature fluctuations. The two readings per zone are averaged for the control setpoint, with divergence alerting if they separate by more than 2°C.

---

## Shopping List — AliExpress

**Estimated total: ~$130–150 USD including shipping.**

Aim to consolidate into as few sellers as possible — large stores like WAVGAT Official Store carry most electronics components. Fans, the contactor, and the SSR may come from separate sellers. When browsing, click through to a promising seller's store page and check what else they carry before placing an order.

### Core electronics

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 1 | Raspberry Pi 4B 2GB | `Raspberry Pi 4 Model B 2GB` | 1 | $45 |
| 2 | USB 3.0 M.2 SATA enclosure adapter | `M.2 SATA USB 3.0 enclosure adapter 2280` | 1 | $5 |
| 3 | DS18B20 waterproof sensor 1m | `DS18B20 waterproof temperature sensor 1m` | 4 | $2 each |
| 4 | DIN-rail mechanical contactor 230V | `Schneider LC1D09 contactor 230V coil` | 1 | $15 |
| 5 | Off-board enclosed relay module 230V | `enclosed relay module 230V 10A DIN` | 1 | $4 |
| 6 | SPDT relay module 5V | `SPDT relay module 5V single channel` | 1 | $2 |
| 7 | 120mm 12V PWM fan (duct) | `120mm 12V PWM fan 4 pin` | 1 | $5 |
| 8 | 92mm 12V PWM fan (circulation) | `92mm 12V PWM fan 4 pin` | 2 | $4 each |
| 9 | PTC ceramic heater 100W 230V | `PTC ceramic heater panel 100W 230V` | 1 | $6 |
| 10 | 1.3" color TFT ST7789 display | `1.3 inch TFT ST7789 SPI 240x240 color display` | 3 | $2.50 each |

### Power

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 11 | 12V 5A PSU module | `12V 5A switching power supply module bare board` | 1 | $6 |
| 12 | 5V 3A PSU module | `5V 3A switching power supply module bare board` | 1 | $5 |

### PCB components

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 13 | 40-pin GPIO female header 2.54mm | `2x20 pin female header 2.54mm raspberry pi` | 1 | $1 |
| 14 | Screw terminal blocks 3.5mm pitch | `PCB screw terminal block 3.5mm 2pin 3pin assorted` | 1 | $3 |
| 15 | BC547 NPN transistor | `BC547 NPN transistor 50pcs` | 1 | $1 |
| 16 | 1N4007 flyback diode | `1N4007 diode 50pcs` | 1 | $1 |
| 17 | 4.7kΩ resistor pack | `4.7k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 18 | 10kΩ resistor pack | `10k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 19 | 1kΩ resistor pack | `1k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 20 | 330Ω resistor pack | `330 ohm resistor 1/4W 100pcs` | 1 | $1 |
| 21 | 5mm LED assorted pack | `5mm LED red green yellow assorted 100pcs` | 1 | $1 |
| 22 | 100nF decoupling capacitors | `100nF ceramic capacitor 0.1uF 100pcs` | 1 | $1 |

### Misc and installation

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 23 | Dupont jumper wire kit | `Dupont jumper wire 120pcs male female 20cm` | 1 | $2 |
| 24 | M12 cable glands nylon | `M12 cable gland nylon waterproof 10pcs` | 1 | $2 |
| 25 | M3 hex standoffs + screws kit | `M3 brass standoff spacer hex screw kit assorted` | 1 | $4 |
| 26 | M3 threaded heat inserts | `M3 heat set insert brass knurled 50pcs` | 1 | $3 |
| 27 | PCB conformal coating spray | `MG Chemicals 419C conformal coating spray` | 1 | $8 |
| 28 | Multimeter | `digital multimeter voltage current AC DC` | 1 | $10 |
| 29 | Pi 4B heatsink kit | `Raspberry Pi 4 heatsink kit aluminum` | 1 | $3 |
| 30 | Foam weatherstripping tape 10mm | `foam weatherstrip tape self adhesive 10mm` | 1 | $2 |
| 31 | Prototype HAT for Pi 4B | `Raspberry Pi 4 prototype HAT GPIO expansion` | 1 | $4 |

### Important notes

- **Pi 4B (#1):** Has dramatically more headroom than the Zero 2W — needed for three displays, PWM fan control, Flask, SQLite, Brewfather API, and future load cells simultaneously. Confirm 2GB version from a seller with 1000+ orders.
- **M.2 SATA adapter (#2):** Confirm listing specifies **SATA** not NVMe — the two are electrically incompatible despite the same physical connector. The user already has an M.2 SATA drive.
- **DS18B20 sensors (#3):** Buy 4 — two per zone, submerged in water containers. Water thermal mass matches the reality of large liquid volumes (19L corny kegs, 50L fermenter) and prevents compressor short cycling.
- **DIN-rail contactor (#4):** A mechanical contactor is preferred over an SSR for this application. Cheap AliExpress SSRs are widely documented as counterfeits that fail below rated capacity. Critically, SSRs **fail closed (ON)** — meaning the compressor would run forever if the SSR fails. A mechanical contactor fails open (OFF) which is the safe failure mode. The Schneider LC1D09 is a well-known reliable part; search for it specifically by model number.
- **Off-board relay module (#5):** All mains voltage (230V) switching happens through this off-board enclosed module, never on the custom PCB. This is a hard safety rule.
- **SPDT relay (#6):** Hardware interlock ensuring the heater and cooler circuits physically cannot be powered simultaneously regardless of software state.
- **Fans (#7, #8):** Must be **4-pin PWM**. Confirm listing shows four wires: red (12V), black (GND), blue (PWM), yellow (tach).
- **ST7789 display (#10):** Confirm listing states **ST7789 driver** and **240×240 resolution**.
- **PSU modules (#11, #12):** Bare board switching PSUs wired directly to mains inside the enclosure. The 12V module powers all fans. The 5V module powers the Pi via GPIO 5V pins for clean, stable power.
- **Conformal coating (#27):** Apply to all PCBs before installation. The garage environment means temperature swings and humidity — conformal coating protects solder joints and traces.

---

## Component Details

### Raspberry Pi 4B

The brain of the system. Quad-core 1.8GHz ARM Cortex-A72, 2GB RAM, built-in WiFi and Bluetooth, full 40-pin GPIO header. Boots from M.2 SATA SSD connected to USB 3.0 — no SD card. Powered via the 5V GPIO pins directly from the onboard 5V PSU module. Fit the heatsink kit — the Pi 4B runs warm and the machine compartment limits airflow.

### M.2 SATA SSD (user supplied) + USB adapter

The Pi 4B boots and runs entirely from an M.2 SATA SSD connected via a USB 3.0 adapter. This eliminates SD card wear entirely — SQLite logs can be written directly at any frequency without concern. The bootloader is configured once to prefer USB boot over SD card. The SD card slot is left empty permanently.

**One-time setup:**
1. Flash Raspberry Pi OS Lite 64-bit to the SSD using a desktop PC and the USB adapter
2. Boot the Pi once from a temporary SD card
3. Run `raspi-config` → Advanced → Bootloader → USB Boot
4. Reboot, remove SD card — Pi boots from SSD permanently

### DS18B20 temperature sensors

Waterproof probes using the 1-Wire protocol. All four sensors chain on a single GPIO pin (GPIO 4) with one shared 4.7kΩ pull-up resistor on the PCB. Each sensor has a unique 64-bit ROM address.

Two sensors per zone, each submerged in a separate sealed container of water (~100ml). The water thermal mass is intentional and beneficial — it accurately represents the slow thermal response of large liquid volumes in each zone (19L corny kegs in the freezer, 50L fermenter in the fridge). This prevents the control loop from reacting to brief air temperature fluctuations and eliminates compressor short cycling. The door sensor settle delay handles door-opening events in software. Fill containers with distilled or boiled water to prevent algae growth.

### DIN-rail mechanical contactor (compressor)

A DIN-rail mounted mechanical contactor switches mains power to the compressor. Mechanical contactors are preferred over solid state relays for this application because:
- Cheap SSRs from AliExpress are widely documented as counterfeits rated far below their claimed capacity
- SSRs **fail closed (ON)** — a failed SSR means the compressor runs forever, potentially freezing kegs solid
- Mechanical contactors **fail open (OFF)** — the safe failure mode
- Contactors are rated and tested for motor inductive loads specifically

The Schneider Electric LC1D09 with a 230V AC coil is the target part — search by model number for reliable sourcing. Mount on DIN rail inside the machine compartment enclosure.

### Off-board enclosed relay module (heater)

All 230V mains switching for the PTC heater runs through an off-board enclosed relay module — never through the custom PCB. This is a hard safety rule. A minor mistake in trace clearance or creepage on a beginner PCB can cause 230V to arc into the 3.3V logic. The custom PCB is strictly low-voltage.

### SPDT relay (hardware interlock)

A single-pole double-throw relay provides a physical interlock between the heater and cooler circuits. Regardless of software state, the SPDT relay ensures only one of the two loads can receive power at any time. This is a true hardware interlock — it cannot be defeated by a software bug, race condition, or unexpected restart state.

### PTC ceramic heater 100W

Self-regulating ceramic heating element at 230V mains. PTC resistance increases with temperature — it physically cannot overheat even if the relay fails to switch off. Switched by the off-board relay module.

### Fans

Three 4-pin PWM 12V brushless fans — one 120mm duct fan and two 92mm circulation fans. Powered from the 12V PSU module. Speed controlled directly by the Pi 4B via hardware PWM GPIO at 25kHz. The tach wire is left unconnected.

PWM fan wiring per fan:
- Red → 12V PSU output
- Black → common GND
- Blue → Pi GPIO PWM pin via 1kΩ resistor on PCB
- Yellow → not connected

### Displays

Three 1.3" color TFT screens using the ST7789 SPI driver at 240×240 resolution. All three share the SPI bus with individual chip select pins. Driven by the `st7789` Python library.

### Door sensors

Factory-fitted magnetic reed switches on both fridge doors. Each wires to a GPIO input with a 10kΩ pull-up on the PCB. Normally closed (GPIO LOW). Opens (GPIO HIGH) when the door opens. Splice into original wires — do not cut them — so the fridge's own door indicator light continues to function.

---

## Custom PCB

> **Phase 2 of the build.** Design after the prototype is fully proven. Every GPIO assignment and circuit is confirmed on breadboard before committing to FR4.

> **Critical rule:** The custom PCB carries **low-voltage signals only** (3.3V, 5V, 12V). No mains voltage (230V) passes through the PCB under any circumstances. All mains switching is handled by off-board enclosed modules.

A HAT-style PCB mounting directly on the Pi 4B's 40-pin GPIO header. All external connections terminate in screw terminal blocks.

### What the PCB provides

- 4× DS18B20 sensor screw terminals (3-pin: VCC, GND, DATA) with shared 4.7kΩ pull-up
- 3× fan PWM output screw terminals (2-pin) with 1kΩ signal resistors
- Contactor control output screw terminal (2-pin: low-voltage coil signal) with 1kΩ resistor
- SPDT relay signal output screw terminal (2-pin: low-voltage only)
- Off-board heater relay signal output screw terminal (2-pin: low-voltage only)
- 2× door sensor screw terminals (2-pin each) with 10kΩ pull-up resistors
- 3× display chip select breakouts to screw terminals
- 5V rail breakout terminal (for PSU module connection)
- Status LED with 330Ω resistor
- 100nF decoupling capacitors on 3.3V and 5V rails
- M3 mounting holes at all four corners

### PCB component list

| Component | Part | Qty |
|---|---|---|
| GPIO connector | 2×20 female header 2.54mm | 1 |
| Sensor terminals | 3-pin screw terminal 3.5mm | 4 |
| Fan PWM terminals | 2-pin screw terminal 3.5mm | 3 |
| Contactor signal terminal | 2-pin screw terminal 3.5mm | 1 |
| SPDT relay signal terminal | 2-pin screw terminal 3.5mm | 1 |
| Heater relay signal terminal | 2-pin screw terminal 3.5mm | 1 |
| Door sensor terminals | 2-pin screw terminal 3.5mm | 2 |
| Display CS terminals | 2-pin screw terminal 3.5mm | 3 |
| 5V rail terminal | 2-pin screw terminal 3.5mm | 1 |
| Signal resistors | 1kΩ 1/4W | 5 |
| Door pull-up resistors | 10kΩ 1/4W | 2 |
| 1-Wire pull-up | 4.7kΩ 1/4W | 1 |
| LED resistor | 330Ω 1/4W | 1 |
| Status LED | 5mm green LED | 1 |
| Decoupling caps | 100nF ceramic | 4 |
| Mounting hardware | M3 standoffs + screws | 4 |

### PCB design and ordering workflow

1. **Design in KiCad** — free and professional. This board is ~20 through-hole components.
2. **DRC** — run design rule check before export.
3. **Export Gerbers** — standard Gerber + drill files from KiCad's plot dialog.
4. **Order from JLCPCB** — 5 boards for ~$15 USD including shipping, 1–2 week turnaround.
5. **Solder** — all through-hole components, solderable by hand. No SMD required.
6. **Test outside enclosure** — verify all signals with multimeter before fitting to Pi.

---

## Machine Compartment Enclosure

The machine compartment sits at the bottom rear of the Samsung fridge behind the kick plate — ambient temperature, ventilated, already containing the compressor and its mains wiring.

### What goes inside

- Raspberry Pi 4B with PCB HAT and M.2 SATA adapter plugged into USB 3.0
- DIN-rail mechanical contactor (compressor)
- Off-board enclosed relay module (heater)
- SPDT relay module (heater/cooler interlock)
- 12V PSU module (fans)
- 5V PSU module (Pi)
- Mains terminal block for live/neutral/earth distribution

### Printed enclosure design

PETG enclosure sized to the machine compartment. Print at 40%+ infill, 4+ perimeters. M3 heat-set threaded inserts for all fastener points.

**Features:**
- DIN rail mounted inside for contactor
- Standoff mounts for Pi+PCB stack
- Off-board relay module mount
- **SSR/contactor heatsink cutout:** The contactor mounts against the inner wall with its body exposed through a cutout to the machine compartment air for cooling. The PETG wall never acts as a heatsink — metal to air only.
- PSU module mounts with strain relief
- Mains cable entry on one face with proper grommet
- Low-voltage cable exits on opposite face
- Ventilation slots on sides and top
- Lid fixed with M3 screws into heat-set inserts
- All external connection points labelled with embossed text

**Inside cable separation:**
- Mains wiring (brown/blue/green-yellow) on one side
- All low-voltage wiring on the other side
- Never bundled together

**Conformal coating:** Spray all PCBs before installation and allow to fully cure.

---

## Wiring

### GPIO pin assignment

```
# 1-Wire temperature sensors (all 4 on one bus)
GPIO 4   →  DS18B20 data bus (4.7kΩ pull-up to 3.3V on PCB)

# Mains switching signals (low-voltage control signals only)
GPIO 17  →  Contactor coil signal (compressor) via 1kΩ resistor
GPIO 27  →  Off-board heater relay signal via 1kΩ resistor

# SPDT hardware interlock signal
GPIO 22  →  SPDT relay signal via 1kΩ resistor

# PWM fan speed control (hardware PWM pins)
GPIO 12  →  Duct fan PWM         (120mm, between zones)
GPIO 13  →  Freezer circ fan PWM (92mm, freezer zone)
GPIO 19  →  Fridge circ fan PWM  (92mm, fridge zone)

# Door sensors (HIGH when open, 10kΩ pull-up on PCB)
GPIO 5   →  Freezer door reed switch
GPIO 6   →  Fridge door reed switch

# SPI displays (shared bus)
GPIO 11  →  SCLK
GPIO 10  →  MOSI
GPIO 24  →  DC   (shared across all 3 displays)
GPIO 25  →  RST  (shared across all 3 displays)

# Display chip selects
GPIO 8   →  Screen 1 CS  (hardware CE0)
GPIO 7   →  Screen 2 CS  (hardware CE1)
GPIO 26  →  Screen 3 CS  (software CS)

# Status LED
GPIO 23  →  Status LED via 330Ω resistor

# Reserved for future use
GPIO 16  →  RESERVED — bubble sensor interrupt (v1.4)
```

> **Note:** GPIO 12, 13, and 19 are Pi 4B hardware PWM pins — always use hardware PWM for fans at 25kHz. Verify all assignments against the Pi 4B pinout before soldering the PCB.

### Contactor wiring (compressor)

```
Pi GPIO 17 → 1kΩ resistor → contactor coil A1
Contactor coil A2 → neutral
Contactor main contact L1 → mains live in
Contactor main contact T1 → mains live to compressor
```

The contactor coil runs on 230V AC — a2 connects to neutral. The Pi GPIO signal drives a small intermediate relay or opto-isolator to switch the 230V coil, not the coil directly from GPIO.

### SPDT hardware interlock wiring

```
SPDT relay COM     → mains live supply
SPDT relay NO      → heater relay module live in
SPDT relay NC      → duct fan 12V positive (or contactor signal)
Pi GPIO 22         → SPDT relay coil signal
```

When GPIO 22 is HIGH: cooling circuit is active, heater is physically disconnected.
When GPIO 22 is LOW: heating circuit is active, cooling is physically disconnected.

### Heater relay wiring

```
Pi GPIO 27  →  1kΩ resistor → off-board relay module signal IN
Relay module NO  → mains live to PTC heater (via SPDT interlock)
Relay module COM → mains live from SPDT relay
```

### Sensor wiring

```
DS18B20 red    →  3.3V
DS18B20 black  →  GND
DS18B20 yellow →  GPIO 4 (4.7kΩ pull-up to 3.3V on PCB)
```

All four sensors wire in parallel on the same three-wire bus.

### Fan wiring

```
Fan red   (12V)  →  12V PSU output
Fan black (GND)  →  common GND
Fan blue  (PWM)  →  Pi GPIO PWM pin via 1kΩ resistor on PCB
Fan yellow (tach) →  not connected
```

### Door sensor wiring

```
Reed switch wire 1  →  GPIO 5 or 6
Reed switch wire 2  →  GND
10kΩ pull-up from GPIO pin to 3.3V on PCB
```

### Display wiring

```
VCC  →  3.3V
GND  →  GND
SCL  →  GPIO 11
SDA  →  GPIO 10
DC   →  GPIO 24
RST  →  GPIO 25
CS   →  GPIO 8 / 7 / 26 (one per screen)
```

---

## 3D Printed Parts

Print all parts in **PETG**. PLA becomes brittle in the temperature-cycling, humid garage environment within months.

### Machine compartment enclosure

Main electronics housing. See Machine Compartment Enclosure section for full design requirements. Print at 40%+ infill, 4+ perimeters. M3 heat-set inserts for all fastener points.

### Duct shroud and frame

Fits in the ~125mm hole between compartments. Accepts the 120mm fan on the freezer face with clip retention for tool-free removal. Flanged lips seal against the divider wall with foam weatherstripping. Includes a return air slot on the opposite side. Print at 40%+ infill, 3+ perimeters.

### Circulation fan brackets

Clip-on mounts for 92mm fans onto existing shelf rails or side wall features. No extra drilling required.

### Sensor water housings

Two sealed cylindrical containers (~100ml each), one per zone. Press-fit or screw cap with a sealed cable port. Print at 60%+ infill, 4+ perimeters for water tightness. Seal with food-safe silicone. Fill with distilled or boiled water.

### PTC heater bracket

Wall-mount bracket for the PTC heater panel in the fridge section. Holds heater away from wall for airflow clearance.

### Display bezels

One bezel per screen. Screen 1 mounts to fridge exterior with VHB tape. Screens 2 and 3 mount at each tap position. Include cable channel in design.

### Cable routing clips

Small saddle clips with 3M VHB tape backing for routing cables along fridge interior walls.

### Cable pass-through plates

Cover plates over holes where cables exit each zone into the machine compartment. Accepts M12 cable glands.

### Future parts

**Keg load cell platform (v1.1):** Flat sled per corny keg with four recesses for 50kg load cells and alignment lips. Cornies have a ~230mm diameter base — size alignment lips accordingly. Print at 60%+ infill.

**Keg insulation jacket (v1.5):** PETG outer shell per corny keg lined with 25mm XPS foam. Houses a PTC heater strip and a DS18B20 surface sensor. Protects kegs during cold crash mode when freezer drops well below zero.

---

## Control Logic

### Python pseudocode

```python
import time
import RPi.GPIO as GPIO
from w1thermsensor import W1ThermSensor
from datetime import datetime, timezone

# Configuration
FREEZER_TARGET     = 1.5    # °C
HYSTERESIS_FREEZE  = 0.5    # °C
COMPRESSOR_MIN_OFF = 600    # seconds — 10 min lockout
DOOR_SETTLE_SECS   = 45     # seconds to wait after door closes
DOOR_ALERT_SECS    = 300    # seconds before Discord door alert

# GPIO — mains switching signals (low-voltage only)
CONTACTOR_COMPRESSOR = 17
RELAY_HEATER         = 27
SPDT_INTERLOCK       = 22

# GPIO — PWM fans (hardware PWM pins)
PWM_DUCT_FAN    = 12
PWM_FREEZE_CIRC = 13
PWM_FRIDGE_CIRC = 19
PWM_FREQ        = 25000  # Hz

# GPIO — door sensors
DOOR_FREEZER    = 5
DOOR_FRIDGE     = 6
STATUS_LED      = 23

# Persist compressor lockout to SQLite — survives reboots
def get_last_compressor_off():
    row = db.execute(
        "SELECT last_compressor_off FROM system"
    ).fetchone()
    return row["last_compressor_off"] if row else 0

def set_compressor(on):
    if on:
        GPIO.output(SPDT_INTERLOCK, GPIO.HIGH)  # interlock: cooling active
        GPIO.output(CONTACTOR_COMPRESSOR, GPIO.HIGH)
    else:
        GPIO.output(CONTACTOR_COMPRESSOR, GPIO.LOW)
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)
        # Persist lockout timestamp immediately — survives Pi reboot
        db.execute(
            "UPDATE system SET last_compressor_off = ?", (time.time(),)
        )

def set_heater(on):
    if on:
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)   # interlock: heating active
        GPIO.output(RELAY_HEATER, GPIO.HIGH)
    else:
        GPIO.output(RELAY_HEATER, GPIO.LOW)

# Initialise PWM
duct_pwm   = GPIO.PWM(PWM_DUCT_FAN,    PWM_FREQ)
freeze_pwm = GPIO.PWM(PWM_FREEZE_CIRC, PWM_FREQ)
fridge_pwm = GPIO.PWM(PWM_FRIDGE_CIRC, PWM_FREQ)
for pwm in [duct_pwm, freeze_pwm, fridge_pwm]:
    pwm.start(0)

def fan_speed(error_c):
    """Proportional fan speed 0–100% from temperature error magnitude."""
    if error_c < 0.3:   return 0
    elif error_c < 1.0: return 30
    elif error_c < 2.0: return 60
    else:               return 100

def control_loop():
    handle_door_events()
    check_for_step_change()

    if (door_is_open(DOOR_FREEZER) or door_is_open(DOOR_FRIDGE)
            or recently_closed(DOOR_FREEZER) or recently_closed(DOOR_FRIDGE)):
        for pwm in [duct_pwm, freeze_pwm, fridge_pwm]:
            pwm.ChangeDutyCycle(0)
        return

    # Average two sensors per zone
    t_freezer = (read_sensor('freezer_1') + read_sensor('freezer_2')) / 2
    t_fridge  = (read_sensor('fridge_1')  + read_sensor('fridge_2'))  / 2
    fridge_target = get_fridge_target()

    # Alert if sensors in same zone diverge
    if abs(read_sensor('freezer_1') - read_sensor('freezer_2')) > 2.0:
        post_discord("🌡️ Sensor alert",
            "Freezer sensors diverging — check water housings", COLOR_WARN)
    if abs(read_sensor('fridge_1') - read_sensor('fridge_2')) > 2.0:
        post_discord("🌡️ Sensor alert",
            "Fridge sensors diverging — check water housings", COLOR_WARN)

    # --- Freezer / compressor ---
    freeze_error = t_freezer - FREEZER_TARGET
    last_off = get_last_compressor_off()
    if freeze_error > HYSTERESIS_FREEZE:
        if time.time() - last_off > COMPRESSOR_MIN_OFF:
            set_compressor(True)
            freeze_pwm.ChangeDutyCycle(fan_speed(abs(freeze_error)))
    elif freeze_error < -HYSTERESIS_FREEZE:
        set_compressor(False)
        freeze_pwm.ChangeDutyCycle(0)

    # --- Fridge / fermentation (SPDT relay enforces hardware interlock) ---
    fridge_error = t_fridge - fridge_target
    if fridge_error > 0.3:
        set_heater(False)
        duct_pwm.ChangeDutyCycle(fan_speed(fridge_error))
        fridge_pwm.ChangeDutyCycle(fan_speed(fridge_error))
    elif fridge_error < -0.3:
        duct_pwm.ChangeDutyCycle(0)
        set_heater(True)
        fridge_pwm.ChangeDutyCycle(50)
    else:
        duct_pwm.ChangeDutyCycle(0)
        set_heater(False)
        fridge_pwm.ChangeDutyCycle(0)

    log(time.time(), t_freezer, t_fridge, fridge_target)
    time.sleep(30)
```

### Safety rules

- Compressor has a hard 10-minute minimum off time — persisted to SQLite, survives reboots
- SPDT relay provides **hardware interlock** — heater and cooler physically cannot run simultaneously regardless of software state
- All outputs default to OFF on startup and crash
- Fan speed is proportional to temperature error
- Door open events pause all fans and skip the control loop
- Sensor divergence in the same zone triggers Discord alert
- All state changes logged to SQLite with timestamp

---

## Software Stack

### OS and runtime

- **Raspberry Pi OS Lite 64-bit** — headless, no desktop, boots from M.2 SATA SSD
- **Python 3** with `RPi.GPIO`, `w1thermsensor`, `sqlite3`, `st7789`, `Flask`, `requests`
- Control loop runs as a `systemd` service — auto-restarts on boot or crash

### Data storage

SQLite database written directly to the M.2 SATA SSD. No RAM disk required — SSD write endurance is not a concern.

```sql
-- System state (persists across reboots)
CREATE TABLE system (
    last_compressor_off REAL DEFAULT 0
);

-- Temperature and state log
CREATE TABLE log (
    ts              INTEGER PRIMARY KEY,
    t_freezer       REAL,
    t_fridge        REAL,
    fridge_target   REAL,
    compressor      INTEGER,
    heater          INTEGER,
    duct_fan_pct    INTEGER,
    freeze_circ_pct INTEGER,
    fridge_circ_pct INTEGER
);

-- Door event log
CREATE TABLE door_events (
    ts      INTEGER PRIMARY KEY,
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
    notes             TEXT
);

-- Active fermentation metadata
CREATE TABLE fermentation_meta (
    batch_id     TEXT,
    batch_name   TEXT,
    recipe_name  TEXT,
    pitch_date   TEXT,
    active       INTEGER DEFAULT 0
);

-- Fermentation steps from Brewfather profile
CREATE TABLE fermentation_profile (
    step_index    INTEGER PRIMARY KEY,
    step_name     TEXT,
    target_temp   REAL,
    duration_days INTEGER
);
```

### Web dashboard (Flask)

Local web server at `http://kegerator.local`. Pages:
- Live temperatures, compressor/heater/fan states, door status
- Temperature history graph
- Keg cards: beer info, % remaining, last pour, last clean
- Fermentation page: Brewfather connection, batch selection, profile start
- Settings: Brewfather credentials, fallback temp, Discord webhook, door alert threshold

### Display screens

**Screen 1 — Fermentation** cycles between:
- Zone temps vs targets, compressor/heater/fan states, door status
- Active batch name, current step, target temp, days remaining in step

**Screens 2 & 3 — Tap displays:**
- Beer name (large), style, ABV, IBU
- % remaining as colour-coded bar (green → amber → red)
- Last pour timestamp
- Days since last clean (turns red when overdue)

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
```

---

## Brewfather Integration

The Pi connects to Brewfather only when you press the button in the web UI — no background polling. Once imported the profile runs entirely from local SQLite.

### Flow

```
Web UI → "Connect to Brewfather" button
     ↓
Single API call — fetches your batch list
     ↓
Pick the batch, confirm or adjust pitch date
     ↓
"Start fermenting" — profile saved to SQLite
     ↓
Control loop follows profile from local DB
Brewfather not contacted again for this batch
```

### Fermentation page UI

```
[ Connect to Brewfather ]
         ↓
○ Mosaic IPA        — Fermenting
○ Stout             — Planning
         ↓
[ Load profile ]
         ↓
Mosaic IPA
  Step 1  Primary fermentation   18.5 °C   7 days  ← active
  Step 2  Diacetyl rest          21.0 °C   2 days
  Step 3  Cold crash              2.0 °C   3 days

Pitch date: [2026-05-06]  (editable)

[ Start fermenting ]
```

### Step change detection and cold crash notification

```python
last_step_index = None

def check_for_step_change():
    global last_step_index
    step = get_current_step()
    if step is None or step["step_index"] == last_step_index:
        return
    last_step_index = step["step_index"]
    batch_name = db.execute(
        "SELECT batch_name FROM fermentation_meta WHERE active=1"
    ).fetchone()["batch_name"]

    is_cold_crash = step["target_temp"] <= 4.0

    if is_cold_crash:
        post_discord(
            "❄️ Cold crash starting",
            f"**{batch_name}** dropping to {step['target_temp']}°C\n"
            "Check blowoff tube is submerged in sanitiser.",
            COLOR_INFO
        )
    else:
        post_discord(
            "📈 Fermentation step change",
            f"**{batch_name}** → {step['step_name']} at "
            f"{step['target_temp']}°C for {step['duration_days']} days",
            COLOR_WARN
        )
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
    days_fermenting = (datetime.now(timezone.utc) - pitch_date).days
    steps = db.execute(
        "SELECT * FROM fermentation_profile ORDER BY step_index"
    ).fetchall()
    elapsed = 0
    for step in steps:
        elapsed += step["duration_days"]
        if days_fermenting < elapsed:
            return step["target_temp"]
    return steps[-1]["target_temp"]
```

---

## Discord Notifications

> **v1.3 feature.** No hardware changes required.

Create a webhook URL in your Discord server (channel settings → Integrations → Webhooks) and add to `config.ini`. All Discord calls are silently skipped if not configured.

### What gets posted

**Fermentation:** Profile started, step changes with new target and duration, cold crash start with blowoff tube reminder, fermentation complete, temperature out of range, temperature recovered.

**Doors:** Door left open beyond threshold.

**System:** Pi restarted, sensor divergence alert, compressor running unusually long.

**Daily digest:** Both zone temps, active fermentation step and days remaining, keg levels once load cells added.

### Implementation

```python
def post_discord(title, message, color=0x3498db):
    webhook_url = config["discord"].get("webhook_url")
    if not webhook_url:
        return
    try:
        requests.post(webhook_url, json={
            "embeds": [{"title": title,
                        "description": message,
                        "color": color}]
        }, timeout=5)
    except requests.RequestException:
        pass  # never let Discord affect the control loop

COLOR_INFO  = 0x3498db  # blue  — normal events
COLOR_OK    = 0x2ecc71  # green — recovery, completion
COLOR_WARN  = 0xe67e22  # amber — step changes, door alerts
COLOR_ALERT = 0xe74c3c  # red   — out of range, errors
```

---

## Physical Installation

### Machine compartment

Remove kick plate at bottom rear of Samsung fridge. Mount printed enclosure on floor or side wall. Route all cables before closing lid. The M.2 SATA adapter plugs directly into one of the Pi's USB 3.0 ports inside the enclosure.

### Duct fan

Cut an ~125mm hole through the divider wall between compartments. Printed shroud clips in with the fan on the freezer face. Seal flanges with foam weatherstripping. Add return air slot on opposite end of divider.

### Circulation fans

Freezer fan: low on side wall, blowing horizontally across the bottom of the keg zone. Fridge fan: high on back wall blowing down, with duct fan inlet on opposite side.

### Sensor placement

Mount water housings in each zone using VHB tape. Freezer: mid-height between kegs, away from evaporator coil and direct fan airflow. Fridge: at the same height as the fermenter body, away from the duct fan outlet.

### Door sensors

Locate factory reed switches in door frames near hinge area. Splice into both wires with thin cable (20–24 AWG) and route along door frame into machine compartment. Do not cut original wires.

### PTC heater

Mount on back wall of fridge section at mid-height using printed bracket. Keep away from fermenter and circulation fan outlet.

### Display cables

Route from machine compartment through M12 cable glands. Screen 1 exits to fridge exterior. Screens 2 and 3 route to tap positions.

### Cable discipline

Keep mains wiring physically separated from low-voltage wiring throughout. Use M12 cable glands at every wall penetration.

---

## Build Phases

### Phase 1 — Prototype

Get everything working with Pi 4B and prototype HAT with jumper wires. Validate all GPIO assignments, test control loop with real temperatures, confirm PWM fan control, verify three displays, test door sensor logic, confirm Brewfather API. **Do not design the PCB until Phase 1 is completely stable.**

### Phase 2 — Custom PCB

Design in KiCad once Phase 1 is fully proven. Order from JLCPCB (5 boards, ~$15 USD). Test fully outside enclosure before installing. Keep prototype HAT as backup.

### Phase 3 — Machine compartment integration

Design PETG enclosure around confirmed PCB and components. Print, test fit, adjust. Make all final cable runs, apply conformal coating to all boards, install in machine compartment, seal cable glands, replace kick plate.

---

## Roadmap

### v1.0 — MVP (this build)
- Temperature control: freezer at 1–2 °C, fridge following Brewfather fermentation profile
- DIN-rail mechanical contactor for compressor, off-board relay for PTC heater, SPDT hardware interlock
- Compressor lockout persisted to SQLite — survives Pi reboots
- Dual DS18B20 per zone in water housings — averaging and divergence alerting
- Door sensors: pause control on open, settle window on close, log events
- Brewfather integration: on-demand connect, batch selection, manual profile start
- Step change detection: cold crash notification with blowoff tube reminder
- Three displays: fermentation status + two tap info screens
- SQLite logging direct to M.2 SATA SSD
- Local Flask web dashboard
- Phase 1: prototype → Phase 2: custom PCB → Phase 3: machine compartment install

### v1.1 — Keg weight
- HX711 ADC modules + 50kg load cells per corny keg
- Printed load cell sleds sized for ~230mm corny keg base
- Live % remaining on tap displays and web dashboard
- Pour volume estimation from weight delta
- Low keg Discord notification

### v1.2 — Dashboard polish
- Temperature history graphs
- Compressor runtime tracking
- Days-since-clean alerts with Discord notifications
- Pour history log
- Door open frequency and duration statistics

### v1.3 — Notifications and remote access
- Full Discord notification suite: fermentation events, daily digest, door alerts, sensor alerts
- Tailscale for secure remote dashboard access from anywhere
- Push notifications for temperature out of range and overdue cleaning

### v1.4 — Bubble detection and Tilt integration
- TCRT5000 IR photoelectric sensor on blowoff tube
- Hardware interrupt on GPIO 16 counts bubbles in real time
- Bubble rate logged to SQLite and graphed on dashboard
- Discord notification when fermentation becomes active after pitching
- Stuck fermentation alert from sustained near-zero bubble rate
- 3D printed black PETG clip-on housing for IR sensor pair — opaque to block external light
- **Optional Tilt hydrometer integration** — Pi 4B receives Tilt BLE broadcasts over built-in Bluetooth. Gravity and temperature displayed as supplementary data only. Primary control loop always runs from DS18B20 and bubble rate regardless of Tilt status. Handles known failure modes gracefully: Tilt stuck in krausen (ignored, bubble rate still valid), dead battery (system continues normally).

### v1.5 — Keg insulation jackets
- Printed PETG outer shell per corny keg lined with 25mm XPS foam
- PTC heater strip 12V 30W per keg inside jacket
- DS18B20 surface sensor per keg inside jacket
- Keg surface temperature logged and displayed alongside zone air temp
- Jacket heaters activated when freezer drops below a configurable threshold
- Groundwork for cold crash mode in v1.7

### v1.6 — Freeze-point calculation
- Uses load cell weight (v1.1) + Brewfather ABV data already in keg table
- Calculates thermal mass of each keg dynamically: `liquid_kg = total_kg - 5.0` (corny tare weight)
- ABV-adjusted freeze point: `freeze_point = -0.42 * abv` (°C)
- Estimates safe duration below zero based on current keg mass, current temp, and cooling rate
- Cooling rate calibrated automatically from real SQLite temperature data after first cold crash
- Discord alert when a keg's safe window is running short

### v1.7 — Cold crash mode
- Freezer drops to -18°C to -20°C (natural compressor limit) during cold crash step
- Keg jacket heaters (v1.5) activate to protect kegs at serving temperature
- Duct fan runs at full speed to blast -18°C air into fridge section for rapid crash
- Freeze-point calculation (v1.6) monitors each keg and adjusts heater power dynamically
- Post-crash warmup: duct fan pauses after crash completes until freezer returns to 1–2°C
- Discord notification: cold crash start, crash complete, warmup complete
