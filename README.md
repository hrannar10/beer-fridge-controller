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
| Freezer | Keg storage (19L corny kegs) | 1–2 °C | Compressor via DIN-rail contactor |
| Fridge | Fermentation (50L fermenter) | Brewfather profile | Duct fan + PTC heater |

### How it works

The Samsung fridge has a single compressor. The freezer is held at 1–2 °C by switching the compressor on and off via a DIN-rail mechanical contactor with a 10-minute minimum cooldown to protect the motor. The fermentation section has no direct cooling — a duct fan blows cold air from the freezer into the fridge section when cooling is needed. A PTC ceramic heater handles warming when fermentation temperature drops below the Brewfather profile target. Both sections have circulation fans running at proportional PWM speed whenever their zone is actively heating or cooling. All electronics live in the machine compartment at the bottom rear of the fridge. The system boots from an M.2 SATA SSD connected via USB 3.0 — no SD card is used.

### Fan layout

| Fan | Zone | Purpose | Control |
|---|---|---|---|
| Duct fan | Between zones (bottom of divider) | Transfers cold from freezer → fridge | PWM GPIO (proportional to temp error) |
| Circulation fan A | Freezer | Circulates air around kegs | PWM GPIO (mirrors compressor state) |
| Circulation fan B | Fridge | Circulates air around fermenter | PWM GPIO (mirrors heating/cooling state) |

### Switching layout

| Load | Switch type | Reason |
|---|---|---|
| Compressor | DIN-rail mechanical contactor | Reliable, fail-open (safe), no counterfeit risk |
| PTC heater | Off-board enclosed relay module | Mains strictly off custom PCB |
| Fans | PWM GPIO direct | Variable speed control, 12V only |
| Heater/cooler interlock | SPDT relay (hardware) | Physically prevents simultaneous heating and cooling regardless of software state |

### Screen layout

| Screen | Location | Shows |
|---|---|---|
| Screen 1 | Fridge exterior | Temps, compressor, heating, fan states, active fermentation step |
| Screen 2 | Tap 1 | Beer info, % remaining, last pour, last clean |
| Screen 3 | Tap 2 | Same for second beer |

### Door sensors

Both fridge doors have factory-fitted magnetic reed switches tapped into GPIO inputs. When a door opens the control loop pauses all fan operation and ignores temperature readings for a settle period after closing. Door events are logged to SQLite and Discord alerts fire if a door is left open beyond a configurable threshold.

### Temperature sensors

Each zone has two DS18B20 waterproof probes submerged in sealed water containers. The water thermal mass is intentional — with 19L corny kegs in the freezer and a 50L fermenter in the fridge, the beer itself changes temperature very slowly. Sensors in water accurately represent this thermal reality and prevent compressor short cycling from brief air fluctuations such as door openings. The two readings per zone are averaged for the control setpoint with divergence alerting if they separate by more than 2°C.

---

## Shopping List — AliExpress

**Estimated total: ~$130–150 USD including shipping.**

Aim to consolidate into as few sellers as possible. When browsing, click through to a promising seller's store page and check what else they carry before placing an order. The contactor is best sourced by model number from an electrical supplier rather than a generic AliExpress search.

### Core electronics

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 1 | Raspberry Pi 4B 2GB | `Raspberry Pi 4 Model B 2GB` | 1 | $45 |
| 2 | USB 3.0 M.2 SATA enclosure adapter | `M.2 SATA USB 3.0 enclosure adapter 2280` | 1 | $5 |
| 3 | DS18B20 waterproof sensor 1m | `DS18B20 waterproof temperature sensor 1m` | 4 | $2 each |
| 4 | DIN-rail mechanical contactor 230V | `Schneider LC1D09 contactor 230V coil` | 1 | $15 |
| 5 | Off-board enclosed relay module 230V | `enclosed relay module 230V 10A` | 1 | $4 |
| 6 | SPDT relay module 5V | `SPDT relay module 5V single channel` | 1 | $2 |
| 7 | 120mm 12V PWM fan | `120mm 12V PWM fan 4 pin quiet` | 2 | $5 each |
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
| 15 | 4.7kΩ resistor pack | `4.7k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 16 | 10kΩ resistor pack | `10k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 17 | 1kΩ resistor pack | `1k ohm resistor 1/4W 100pcs` | 1 | $1 |
| 18 | 330Ω resistor pack | `330 ohm resistor 1/4W 100pcs` | 1 | $1 |
| 19 | 5mm LED assorted pack | `5mm LED red green yellow assorted 100pcs` | 1 | $1 |
| 20 | 100nF decoupling capacitors | `100nF ceramic capacitor 0.1uF 100pcs` | 1 | $1 |

### Misc and installation

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 21 | Dupont jumper wire kit | `Dupont jumper wire 120pcs male female 20cm` | 1 | $2 |
| 22 | M12 cable glands nylon | `M12 cable gland nylon waterproof 10pcs` | 1 | $2 |
| 23 | M3 hex standoffs + screws kit | `M3 brass standoff spacer hex screw kit assorted` | 1 | $4 |
| 24 | M3 threaded heat inserts | `M3 heat set insert brass knurled 50pcs` | 1 | $3 |
| 25 | PCB conformal coating spray | `MG Chemicals 419C conformal coating spray` | 1 | $8 |
| 26 | Multimeter | `digital multimeter voltage current AC DC` | 1 | $10 |
| 27 | Pi 4B heatsink kit | `Raspberry Pi 4 heatsink kit aluminum` | 1 | $3 |
| 28 | Foam weatherstripping tape 10mm | `foam weatherstrip tape self adhesive 10mm` | 1 | $2 |
| 29 | Prototype HAT for Pi 4B | `Raspberry Pi 4 prototype HAT GPIO expansion` | 1 | $4 |

### Important notes

- **Pi 4B (#1):** Confirm 2GB version from seller with 1000+ orders.
- **M.2 SATA adapter (#2):** Confirm listing specifies **SATA** not NVMe — electrically incompatible despite the same physical connector. User already has an M.2 SATA drive.
- **DS18B20 sensors (#3):** Buy 4 — two per zone submerged in water containers. Thermal lag is intentional and desirable given 19L corny kegs and 50L fermenter.
- **DIN-rail contactor (#4):** Search by model number — Schneider LC1D09 with 230V AC coil. Mechanical contactors **fail open (OFF)** which is the safe failure mode. Cheap AliExpress SSRs are widely documented as counterfeits that **fail closed (ON)** — meaning the compressor runs forever. Do not substitute an SSR here.
- **Off-board relay module (#5):** All 230V mains switching goes through this enclosed off-board module. Mains never appears on the custom PCB under any circumstances.
- **SPDT relay (#6):** Hardware interlock — heater and cooler circuits physically cannot be powered simultaneously regardless of software state.
- **120mm fans (#7):** Two units — one for the duct (bottom of divider, cold air in) and one for the exhaust (top of divider, warm air out). Same spec simplifies the order and the printed shroud design.
- **Fans (#7, #8):** Must be **4-pin PWM**. Confirm listing shows: red (12V), black (GND), blue (PWM), yellow (tach).
- **ST7789 display (#10):** Confirm listing states **ST7789 driver** and **240×240 resolution**.
- **PSU modules (#11, #12):** Bare board switching PSUs wired directly to mains inside enclosure. 12V powers all fans, 5V powers Pi via GPIO pins.
- **Conformal coating (#25):** Apply to all PCBs before installation — garage environment means temperature swings and humidity.

---

## Component Details

### Raspberry Pi 4B

Quad-core 1.8GHz ARM Cortex-A72, 2GB RAM, built-in WiFi and Bluetooth, full 40-pin GPIO header. Boots from M.2 SATA SSD connected to USB 3.0. Powered via 5V GPIO pins from the onboard PSU module. Fit heatsink kit before installing.

### M.2 SATA SSD (user supplied) + USB adapter

Boots and runs entirely from M.2 SATA SSD via USB 3.0 adapter. Eliminates SD card wear — SQLite writes directly at any frequency without concern. SD card slot permanently empty.

**One-time setup:**
1. Flash Raspberry Pi OS Lite 64-bit to SSD using a desktop PC and the USB adapter
2. Boot Pi once from a temporary SD card
3. Run `raspi-config` → Advanced → Bootloader → USB Boot
4. Reboot, remove SD card — Pi boots from SSD permanently

### DS18B20 temperature sensors

Waterproof probes using the 1-Wire protocol. All four chain on GPIO 4 with one shared 4.7kΩ pull-up on the PCB. Each has a unique 64-bit ROM address.

Two sensors per zone, each submerged in ~100ml of water in a sealed container. The water thermal mass intentionally matches the slow thermal response of large liquid volumes — 19L corny kegs and a 50L fermenter do not respond quickly to brief air temperature changes. This prevents compressor short cycling. Door-opening events are handled by the software settle delay, not the physical buffer. Fill containers with distilled or boiled water to prevent algae growth.

### DIN-rail mechanical contactor (compressor)

A Schneider LC1D09 (or equivalent quality contactor) with a 230V AC coil switches mains power to the compressor. Mechanical contactors are used here instead of SSRs for two critical reasons:

- Cheap AliExpress SSRs are widely documented as counterfeits failing well below rated capacity
- SSRs **fail closed (ON)** — a failed SSR means the compressor runs forever, potentially freezing kegs solid
- Mechanical contactors **fail open (OFF)** — the safe failure mode

The Pi GPIO drives the 230V contactor coil via a small intermediate opto-isolator — the GPIO never connects directly to the 230V coil circuit. Mount on DIN rail inside the machine compartment enclosure.

### Off-board enclosed relay module (heater)

All 230V mains switching for the PTC heater runs through an off-board enclosed relay module — never through the custom PCB. This is a hard safety rule. Routing mains on a beginner PCB risks arc-over into the 3.3V logic if trace clearance or creepage rules are violated. The custom PCB carries low-voltage signals only.

### SPDT relay (hardware interlock)

A single-pole double-throw relay physically ensures only one of the heater or cooler circuits can receive power at any time. This cannot be defeated by a software bug, race condition, or unexpected restart state.

### PTC ceramic heater 100W

Self-regulating 230V ceramic heating element. PTC resistance increases with temperature — physically cannot overheat even if the relay fails to switch off. Switched by the off-board relay module.

### Fans

Three 4-pin PWM 12V brushless fans plus one additional 120mm exhaust fan (used in v1.7 cold crash mode). All powered from the 12V PSU module. Speed controlled via hardware PWM GPIO at 25kHz. Tach wire left unconnected.

Fan layout:
- **Duct fan (120mm):** Bottom of divider wall, blows cold air from freezer into fridge. Cold air is denser and sinks — entering at the bottom creates a natural convection loop.
- **Exhaust fan (120mm):** Top of divider wall, blows warm fridge air back into freezer. Warm air rises — exiting at the top works with natural convection. Active during cold crash mode (v1.7) only.
- **Freezer circ fan (92mm):** Circulates air around kegs.
- **Fridge circ fan (92mm):** Circulates air around fermenter, aimed at widest point of vessel.

### Displays

Three 1.3" color TFT screens using the ST7789 SPI driver at 240×240 resolution. All three share the SPI bus with individual chip select pins. Driven by the `st7789` Python library.

### Door sensors

Factory-fitted magnetic reed switches on both doors. Each wires to a GPIO input with 10kΩ pull-up on the PCB. Normally closed (GPIO LOW). Opens (GPIO HIGH) when door opens. Splice into original wires — do not cut them.

---

## Custom PCB

> **Phase 2 of the build.** Design after prototype is fully proven on breadboard.

> **Critical rule:** The custom PCB carries **low-voltage signals only** (3.3V, 5V, 12V). No mains voltage (230V) passes through the PCB under any circumstances.

A HAT-style PCB mounting directly on the Pi 4B's 40-pin GPIO header. All external connections terminate in screw terminal blocks.

### What the PCB provides

- 4× DS18B20 sensor screw terminals (3-pin: VCC, GND, DATA) with shared 4.7kΩ pull-up
- 3× fan PWM output screw terminals (2-pin) with 1kΩ signal resistors
- Contactor opto-isolator signal output terminal (2-pin, low-voltage only)
- SPDT relay signal output terminal (2-pin, low-voltage only)
- Off-board heater relay signal output terminal (2-pin, low-voltage only)
- 2× door sensor screw terminals (2-pin each) with 10kΩ pull-up resistors
- 3× display chip select breakouts to screw terminals
- 5V rail breakout terminal
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

1. **Design in KiCad** — free and professional. ~20 through-hole components, a good first project.
2. **DRC** — run design rule check before export.
3. **Export Gerbers** — standard Gerber + drill files from KiCad.
4. **Order from JLCPCB** — 5 boards for ~$15 USD, 1–2 week turnaround.
5. **Solder** — all through-hole, hand solderable.
6. **Test outside enclosure** before fitting to Pi.

---

## Machine Compartment Enclosure

Sits at the bottom rear of the Samsung fridge behind the kick plate — ambient temperature, ventilated, already containing the compressor and mains wiring.

### What goes inside

- Raspberry Pi 4B with PCB HAT and M.2 SATA adapter on USB 3.0
- DIN-rail with mechanical contactor
- Off-board enclosed relay module (heater)
- SPDT relay module
- 12V and 5V PSU modules
- Mains terminal block

### Printed enclosure design

PETG, 40%+ infill, 4+ perimeters, M3 heat-set inserts throughout.

**Features:**
- DIN rail mount for contactor
- Standoff mounts for Pi+PCB stack
- Off-board relay module mount
- **Contactor heat management:** Contactor mounts with its body exposed through a cutout in the PETG wall, dissipating heat directly into machine compartment air. PETG never acts as a heatsink.
- PSU module mounts with mains strain relief
- Mains cable entry with proper grommet on one face
- Low-voltage cable exits on opposite face
- Ventilation slots on sides and top
- M3 screw lid into heat-set inserts
- External connection points labelled with embossed text

**Cable separation:** Mains wiring on one side, all low-voltage on the other. Never bundled together.

**Conformal coating:** Spray all PCBs before installation, allow to fully cure.

---

## Wiring

### GPIO pin assignment

```
# 1-Wire temperature sensors (all 4 on one bus)
GPIO 4   →  DS18B20 data bus (4.7kΩ pull-up to 3.3V on PCB)

# Mains switching signals (low-voltage control only)
GPIO 17  →  Contactor opto-isolator signal (compressor)
GPIO 27  →  Off-board heater relay signal via 1kΩ resistor
GPIO 22  →  SPDT relay signal (hardware interlock)

# PWM fan speed control (hardware PWM pins)
GPIO 12  →  Duct fan PWM         (120mm, bottom of divider)
GPIO 13  →  Freezer circ fan PWM (92mm)
GPIO 19  →  Fridge circ fan PWM  (92mm)
GPIO 21  →  Exhaust fan PWM      (120mm, top of divider — v1.7)

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

> **Note:** GPIO 17 drives the contactor coil via an opto-isolator — the GPIO never connects directly to the 230V coil circuit. GPIO 12, 13, 19, and 21 are Pi 4B hardware PWM pins — always use hardware PWM for fans at 25kHz.

### Contactor wiring (compressor)

```
Pi GPIO 17 → opto-isolator → contactor coil A1 (230V AC)
Contactor coil A2           → neutral
Contactor main contact L1   → mains live in from fridge socket
Contactor main contact T1   → mains live to compressor
```

### SPDT hardware interlock wiring

```
SPDT relay COM  → 5V signal supply
SPDT relay NO   → heater relay signal enable
SPDT relay NC   → cooling active signal
Pi GPIO 22      → SPDT relay coil

When GPIO 22 HIGH: cooling active, heater physically disconnected
When GPIO 22 LOW:  heating available, cooling signal disconnected
```

### Heater relay wiring

```
Pi GPIO 27       → 1kΩ → off-board relay module signal IN
Relay module NO  → mains live to PTC heater (via SPDT interlock)
Relay module COM → mains live from SPDT interlock
```

### Sensor wiring

```
DS18B20 red    →  3.3V
DS18B20 black  →  GND
DS18B20 yellow →  GPIO 4 (4.7kΩ pull-up to 3.3V on PCB)
```

### Fan wiring

```
Fan red   (12V)   →  12V PSU output
Fan black (GND)   →  common GND
Fan blue  (PWM)   →  Pi GPIO PWM pin via 1kΩ resistor on PCB
Fan yellow (tach) →  not connected
```

### Door sensor wiring

```
Reed switch wire 1  →  GPIO 5 or 6
Reed switch wire 2  →  GND
10kΩ pull-up from GPIO to 3.3V on PCB
```

---

## 3D Printed Parts

Print all parts in **PETG**. PLA becomes brittle in the garage environment within months.

### Machine compartment enclosure

See Machine Compartment Enclosure section. 40%+ infill, 4+ perimeters, M3 heat-set inserts.

### Duct shroud and frame (×2)

Same design printed twice — one for the duct fan (bottom of divider, blowing cold air in) and one for the exhaust fan (top of divider, blowing warm air out, active during v1.7 cold crash only). Accepts a 120mm fan with clip retention. Flanged lips seal against divider wall with foam weatherstripping. Print at 40%+ infill, 3+ perimeters.

The two-hole layout exploits natural convection: cold air enters at the bottom (cold air is denser, sinks), warm air exits at the top (warm air rises). Fans work with thermodynamics rather than against it.

### Circulation fan brackets

Clip-on mounts for 92mm fans onto existing shelf rails. No drilling required. Fridge circulation fan aimed at widest point of fermenter body.

### Sensor water housings

Two sealed cylindrical containers (~100ml each). Press-fit or screw cap with sealed cable port. 60%+ infill, 4+ perimeters for water tightness. Seal with food-safe silicone. Fill with distilled water.

### PTC heater bracket

Wall-mount for PTC heater in fridge section. Holds heater away from wall for airflow clearance.

### Display bezels

One per screen. Screen 1 on fridge exterior with VHB tape. Screens 2 and 3 at tap positions. Include cable channel in design.

### Cable routing clips

Saddle clips with 3M VHB tape for routing cables along fridge interior walls.

### Cable pass-through plates

Cover plates over wall penetrations. Accepts M12 cable glands.

### Future parts

**Keg load cell platform (v1.1):** Sled per corny keg with four 50kg load cell recesses and ~230mm diameter alignment lips matching corny keg base. 60%+ infill.

**Keg insulation jacket (v1.5):** PETG outer shell per corny keg lined with 25mm XPS foam. Houses PTC heater strip and DS18B20 surface sensor. Protects kegs during cold crash mode when freezer drops to -18°C.

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
DOOR_SETTLE_SECS   = 45     # seconds after door closes
DOOR_ALERT_SECS    = 300    # seconds before Discord door alert

# GPIO
CONTACTOR_COMPRESSOR = 17
RELAY_HEATER         = 27
SPDT_INTERLOCK       = 22

PWM_DUCT_FAN    = 12
PWM_FREEZE_CIRC = 13
PWM_FRIDGE_CIRC = 19
PWM_EXHAUST_FAN = 21   # v1.7 cold crash exhaust
PWM_FREQ        = 25000

DOOR_FREEZER = 5
DOOR_FRIDGE  = 6
STATUS_LED   = 23

# Compressor lockout — persisted to SQLite, survives reboots
def get_last_compressor_off():
    row = db.execute(
        "SELECT last_compressor_off FROM system"
    ).fetchone()
    return row["last_compressor_off"] if row else 0

def set_compressor(on):
    if on:
        GPIO.output(SPDT_INTERLOCK, GPIO.HIGH)  # interlock: cooling
        GPIO.output(CONTACTOR_COMPRESSOR, GPIO.HIGH)
    else:
        GPIO.output(CONTACTOR_COMPRESSOR, GPIO.LOW)
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)
        db.execute(
            "UPDATE system SET last_compressor_off = ?",
            (time.time(),)
        )

def set_heater(on):
    if on:
        GPIO.output(SPDT_INTERLOCK, GPIO.LOW)   # interlock: heating
        GPIO.output(RELAY_HEATER, GPIO.HIGH)
    else:
        GPIO.output(RELAY_HEATER, GPIO.LOW)

# Initialise PWM
duct_pwm    = GPIO.PWM(PWM_DUCT_FAN,    PWM_FREQ)
freeze_pwm  = GPIO.PWM(PWM_FREEZE_CIRC, PWM_FREQ)
fridge_pwm  = GPIO.PWM(PWM_FRIDGE_CIRC, PWM_FREQ)
exhaust_pwm = GPIO.PWM(PWM_EXHAUST_FAN, PWM_FREQ)
for pwm in [duct_pwm, freeze_pwm, fridge_pwm, exhaust_pwm]:
    pwm.start(0)

def fan_speed(error_c):
    if error_c < 0.3:   return 0
    elif error_c < 1.0: return 30
    elif error_c < 2.0: return 60
    else:               return 100

def control_loop():
    handle_door_events()
    check_for_step_change()

    if any_door_open_or_settling():
        for pwm in [duct_pwm, freeze_pwm, fridge_pwm, exhaust_pwm]:
            pwm.ChangeDutyCycle(0)
        return

    t_freezer = (read_sensor('freezer_1') + read_sensor('freezer_2')) / 2
    t_fridge  = (read_sensor('fridge_1')  + read_sensor('fridge_2'))  / 2
    fridge_target = get_fridge_target()

    check_sensor_divergence(t_freezer, t_fridge)

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

    # --- Fridge / fermentation ---
    # SPDT relay enforces hardware interlock — software mirrors it
    fridge_error = t_fridge - fridge_target
    if fridge_error > 0.3:
        set_heater(False)
        duct_pwm.ChangeDutyCycle(fan_speed(fridge_error))
        fridge_pwm.ChangeDutyCycle(fan_speed(fridge_error))
        exhaust_pwm.ChangeDutyCycle(0)   # exhaust only active in v1.7
    elif fridge_error < -0.3:
        duct_pwm.ChangeDutyCycle(0)
        exhaust_pwm.ChangeDutyCycle(0)
        set_heater(True)
        fridge_pwm.ChangeDutyCycle(50)
    else:
        duct_pwm.ChangeDutyCycle(0)
        exhaust_pwm.ChangeDutyCycle(0)
        set_heater(False)
        fridge_pwm.ChangeDutyCycle(0)

    log(time.time(), t_freezer, t_fridge, fridge_target)
    time.sleep(30)
```

### Safety rules

- Compressor 10-minute minimum off time persisted to SQLite — survives Pi reboots
- SPDT relay provides hardware interlock — heater and cooler physically cannot run simultaneously
- All outputs default to OFF on startup and crash
- Fan speed proportional to temperature error
- Door open events pause all fans and skip control loop
- Sensor divergence triggers Discord alert
- All state changes logged to SQLite

---

## Software Stack

### OS and runtime

- **Raspberry Pi OS Lite 64-bit** — headless, boots from M.2 SATA SSD
- **Python 3** with `RPi.GPIO`, `w1thermsensor`, `sqlite3`, `st7789`, `Flask`, `requests`
- Control loop runs as `systemd` service — auto-restarts on boot or crash

### Data storage

SQLite written directly to M.2 SATA SSD. No RAM disk needed — SSD write endurance is not a concern.

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

-- Fermentation steps from Brewfather profile
CREATE TABLE fermentation_profile (
    step_index    INTEGER PRIMARY KEY,
    step_name     TEXT,
    target_temp   REAL,
    duration_days INTEGER
);
```

### Beer style detection

The system reads beer style from Brewfather and adjusts notifications accordingly. Hazy styles skip all fining suggestions. Clear styles receive Clarity Ferm reminders at pitching and Biofine Clear reminders at cold crash.

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
- Live temperatures, compressor/heater/fan states, door status
- Temperature history graph
- Keg cards: beer info, % remaining, last pour, last clean, fining status
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

Connects to Brewfather only when triggered from the web UI. Profile runs from local SQLite after import.

### Flow

```
Web UI → "Connect to Brewfather"
     ↓
Single API call — fetches batch list
     ↓
Pick batch, confirm pitch date
     ↓
"Start fermenting" — profile + style saved to SQLite
     ↓
Control loop follows profile locally
```

### On fermentation start

```python
def on_fermentation_start(batch):
    style = batch["recipe"]["style"]["name"]

    if is_hazy_style(style):
        post_discord(
            "🌫️ Hazy style detected",
            f"**{batch['name']}** — {style}\n"
            "No fining agents needed. Cold crash optional.",
            COLOR_INFO
        )
    else:
        post_discord(
            "🔬 Clear style — add Clarity Ferm at pitching",
            f"**{batch['name']}** — {style}\n"
            "Add Clarity Ferm to wort after cooling and before "
            "pitching yeast for maximum clarity.\n"
            "Available locally at brew.is (1,400 kr).",
            COLOR_INFO
        )
```

### Step change and cold crash detection

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
    batch_name = meta["batch_name"]
    style = meta["recipe_style"]
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

### Biofine Clear notification (clear styles only)

When the Tilt hydrometer confirms the beer has genuinely reached crash temperature, notify to add Biofine Clear via closed CO₂ transfer — but only for clear beer styles.

```python
biofine_notification_sent = False

def check_crash_fining_trigger():
    global biofine_notification_sent

    meta = db.execute(
        "SELECT batch_name, recipe_style FROM fermentation_meta WHERE active=1"
    ).fetchone()
    if not meta or is_hazy_style(meta["recipe_style"]):
        return  # hazy styles — skip fining entirely

    tilt_temp    = get_tilt_temperature()
    tilt_gravity = get_tilt_gravity()
    gravity_stable = gravity_unchanged_for_hours(24)

    if (tilt_temp <= 2.5
            and gravity_stable
            and not biofine_notification_sent):
        post_discord(
            "🍺 Add Biofine Clear now",
            f"**{meta['batch_name']}** has reached {tilt_temp:.1f}°C\n"
            "Gravity stable at {tilt_gravity:.3f} — fermentation confirmed complete.\n\n"
            "**Closed transfer method:**\n"
            "1. Measure 1–2ml Biofine Clear per litre of beer\n"
            "2. Dilute 1:1 with CO₂-purged water in small sanitised vessel\n"
            "3. Purge vessel headspace with CO₂\n"
            "4. Connect to fermenter gas-in port\n"
            "5. Push in gently with CO₂ — fermenter stays sealed\n"
            "6. Confirm addition in dashboard to start 48hr countdown",
            COLOR_INFO
        )
        biofine_notification_sent = True
```

### Fining countdown

```python
@app.route("/fermentation/fining-added", methods=["POST"])
def fining_added():
    ts = datetime.now(timezone.utc).isoformat()
    db.execute(
        "UPDATE kegs SET fining_added = ? WHERE tap = ?",
        (ts, request.json["tap"])
    )
    return jsonify({"status": "ok", "ready_at": "48 hours from now"})

def check_fining_complete():
    row = db.execute(
        "SELECT beer_name, fining_added FROM kegs WHERE fining_added IS NOT NULL"
    ).fetchone()
    if not row:
        return
    fining_time = datetime.fromisoformat(row["fining_added"])
    hours_elapsed = (datetime.now(timezone.utc) - fining_time).total_seconds() / 3600
    if hours_elapsed >= 48:
        post_discord(
            "✅ Ready to transfer",
            f"**{row['beer_name']}** — Biofine Clear has been working "
            f"for {hours_elapsed:.0f} hours\n"
            "Beer should be at peak clarity. Transfer to keg when ready.",
            COLOR_OK
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

Create webhook URL in Discord (channel settings → Integrations → Webhooks) and add to `config.ini`. Silently skipped if not configured.

### What gets posted

**Fermentation start:** Style detection — hazy styles skip fining suggestions, clear styles get Clarity Ferm reminder.

**Step changes:** New target temp and duration. Cold crash start includes blowoff tube reminder.

**Cold crash fining (clear styles only):** Biofine Clear closed-transfer instructions when Tilt confirms crash temperature and gravity is stable.

**Fining complete:** Transfer-ready notification 48 hours after Biofine Clear addition confirmed in dashboard.

**Doors:** Door left open beyond threshold.

**System:** Pi restarted, sensor divergence, compressor running unusually long.

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
        pass

COLOR_INFO  = 0x3498db  # blue
COLOR_OK    = 0x2ecc71  # green
COLOR_WARN  = 0xe67e22  # amber
COLOR_ALERT = 0xe74c3c  # red
```

---

## Physical Installation

### Machine compartment

Remove kick plate at bottom rear of Samsung fridge. Mount printed enclosure on floor or side wall. M.2 SATA adapter plugs into Pi USB 3.0 inside enclosure. Route all cables before closing lid.

### Divider wall — two holes

Cut two ~125mm holes through the divider wall between compartments:
- **Bottom hole:** Duct fan, blowing cold air from freezer into fridge
- **Top hole:** Exhaust fan, blowing warm fridge air back to freezer (active in v1.7 only)

Cold air enters at the bottom (denser, sinks naturally), warm air exits at the top (lighter, rises naturally). Thermodynamics and fans work together.

Seal both shroud flanges with foam weatherstripping. Include a passive return slot on the opposite end of the divider for normal operation when the exhaust fan is inactive.

### Circulation fans

Freezer fan: low on side wall, blowing horizontally across keg zone bottom. Fridge fan: high on back wall aimed at widest point of fermenter, with duct fan inlet on opposite side.

### Sensor placement

Freezer water housing: mid-height between kegs, away from evaporator coil and direct fan airflow. Fridge water housing: same height as fermenter body, away from duct fan outlet.

### Door sensors

Locate factory reed switches near hinge area. Splice thin cable (20–24 AWG) into both wires without cutting. Route along door frame into machine compartment.

### PTC heater

Back wall of fridge section at mid-height on printed bracket. Away from fermenter and circulation fan outlet.

### Display cables

Machine compartment → M12 cable glands → fridge zones. Screen 1 to fridge exterior. Screens 2 and 3 to tap positions.

### Cable discipline

Mains wiring physically separated from low-voltage throughout. M12 cable glands at every wall penetration.

---

## Build Phases

### Phase 1 — Prototype
Get everything working with Pi 4B and prototype HAT with jumper wires. Validate all GPIO assignments, control loop, PWM fans, three displays, door sensors, Brewfather API. **Do not design PCB until Phase 1 is fully stable.**

### Phase 2 — Custom PCB
Design in KiCad once Phase 1 is proven. Order from JLCPCB (5 boards, ~$15). Test outside enclosure first. Keep prototype HAT as backup.

### Phase 3 — Machine compartment integration
Design PETG enclosure around confirmed PCB and components. Print, test fit, adjust. Apply conformal coating to all boards. Final cable runs, install, seal glands, replace kick plate.

---

## Roadmap

### v1.0 — MVP (this build)
- Temperature control: freezer 1–2°C, fridge follows Brewfather profile
- DIN-rail mechanical contactor (compressor), off-board relay (heater), SPDT hardware interlock
- Compressor lockout persisted to SQLite — survives reboots
- Dual DS18B20 per zone in water housings — averaging and divergence alerting
- Door sensors: pause control on open, settle window on close, log events
- Brewfather: on-demand connect, batch selection, manual profile start, style detection
- Style-aware notifications: Clarity Ferm reminder for clear styles at pitching, no fining for hazy styles
- Three displays: fermentation status + two tap info screens
- SQLite direct to M.2 SATA SSD
- Local Flask web dashboard
- Phase 1 → Phase 2 → Phase 3 build sequence

### v1.1 — Keg weight
- HX711 ADC + 50kg load cells per corny keg
- Printed load cell sleds for ~230mm diameter corny base
- Live % remaining on displays and dashboard
- Pour volume estimation from weight delta
- Low keg Discord notification

### v1.2 — Dashboard polish
- Temperature history graphs
- Compressor runtime tracking
- Days-since-clean alerts
- Pour history log
- Door open statistics

### v1.3 — Notifications and remote access
- Full Discord suite: fermentation events, cold crash with blowoff reminder, Biofine Clear instructions (clear styles only), fining countdown, daily digest
- Tilt hydrometer integration over Bluetooth: gravity and temperature as supplementary data, primary control loop always runs from DS18B20 and bubble rate, handles Tilt stuck in krausen and dead battery gracefully
- Tailscale for secure remote access
- Push notifications for temperature out of range, overdue cleaning

### v1.4 — Bubble detection
- TCRT5000 IR sensor on blowoff tube, hardware interrupt on GPIO 16
- Bubble rate logged and graphed on dashboard
- Discord: fermentation active after pitching, stuck fermentation alert, completion detection
- 3D printed black PETG clip-on housing for IR pair — opaque to block ambient light

### v1.5 — Keg insulation jackets
- Printed PETG shell + 25mm XPS foam per corny keg
- PTC heater strip 12V 30W inside jacket
- DS18B20 surface sensor per keg inside jacket
- Keg surface temp logged and displayed
- Jacket heaters activate when freezer drops below configurable threshold
- Groundwork for cold crash mode in v1.7

### v1.6 — Freeze-point calculation
- Load cell weight (v1.1) + Brewfather ABV → thermal mass per keg
- Corny keg tare weight: 5kg, liquid mass = total − 5kg
- ABV-adjusted freeze point: `freeze_point_c = -0.42 × abv`
- Estimates safe duration below zero from current mass, temp, and cooling rate
- Cooling rate calibrated automatically from real SQLite data after first crash
- Discord alert when keg safe window running short

### v1.7 — Cold crash mode
- Freezer drops to -18°C to -20°C (natural compressor limit) during cold crash step
- Exhaust fan (120mm, top of divider) activates — warm fridge air actively pushed back to freezer, cold air enters at bottom via duct fan, thermodynamics and fans aligned
- Keg jacket heaters (v1.5) protect kegs at serving temperature
- Freeze-point calculation (v1.6) monitors each keg dynamically
- Post-crash warmup: exhaust and duct fans pause until freezer returns to 1–2°C, jacket heaters stay active during warmup
- Discord: cold crash start, crash complete, warmup complete
- Estimated crash time 50L fermenter 18°C → 2°C: 4–6 hours

### v1.8 — DIY glycol chilling (optional, complex)
- Dedicated secondary compressor (salvaged or 12V DC camping fridge unit) with evaporator coil submerged in glycol reservoir
- 30–40% food-grade propylene glycol + water prevents freezing at operating temps
- Small 12V submersible pump circulates glycol through food-grade silicone coil wrapped around fermenter
- Pi controls secondary compressor and pump — glycol kept at -7°C continuously, pump activates on cold crash trigger
- Estimated crash time 50L fermenter: 45–90 minutes
- Noted as significant additional complexity — treat as a separate dedicated build
