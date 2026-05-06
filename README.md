# Beer Fridge Controller — Full Build Spec

A Raspberry Pi 4B-based temperature controller for a dual-zone Samsung fridge used for beer fermentation (fridge section) and keg storage (freezer section). Electronics live in the machine compartment. Tap display screens show live keg info. Brewfather integration for fermentation profile management.

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
| Freezer | Keg storage | 1–2 °C | Compressor via SSR |
| Fridge | Fermentation | Brewfather profile | Duct fan + PTC heater |

### How it works

The Samsung fridge has a single compressor. The freezer is held at 1–2 °C by switching the compressor on and off via a solid state relay with a 10-minute minimum cooldown to protect the motor. The fermentation section has no direct cooling — a duct fan blows cold air from the freezer into the fridge section when cooling is needed. A PTC ceramic heater handles warming when fermentation temperature drops below the Brewfather profile target, important during cold Icelandic winters. Both sections have circulation fans that run at proportional PWM speed whenever their zone is actively heating or cooling. All electronics live in the machine compartment at the bottom rear of the fridge — ambient temperature, no condensation risk, direct access to compressor wiring.

### Fan layout

| Fan | Zone | Purpose | Control |
|---|---|---|---|
| Duct fan | Between zones | Transfers cold from freezer → fridge | PWM GPIO (proportional to temp error) |
| Circulation fan A | Freezer | Circulates air around kegs | PWM GPIO (mirrors compressor state) |
| Circulation fan B | Fridge | Circulates air around fermenter | PWM GPIO (mirrors heating/cooling state) |

### Switching layout

| Load | Switch type | Reason |
|---|---|---|
| Compressor | Fotek SSR-25DA (solid state relay) | Silent, handles inductive motor load, no mechanical wear |
| PTC heater | Signal relay on PCB | Low switching frequency, mains isolation |
| Fans | PWM GPIO direct | Variable speed control, 12V only |

### Screen layout

| Screen | Location | Shows |
|---|---|---|
| Screen 1 | Fridge exterior | Temps, compressor, heating, fan states, active fermentation step |
| Screen 2 | Tap 1 | Beer info, % remaining, last pour, last clean |
| Screen 3 | Tap 2 | Same for second beer |

### Door sensors

Both fridge doors have factory-fitted magnetic reed switches. These are tapped into GPIO inputs. When a door opens the control loop pauses all fan operation and ignores temperature readings for a settle period after closing. Door open events are logged to SQLite and Discord alerts fire if a door is left open beyond a configurable threshold.

---

## Shopping List — AliExpress

**Estimated total: ~$130–150 USD including shipping.**

Buy from stores with 4.7★+ ratings and 500+ orders. Aim to consolidate into as few sellers as possible — large stores like WAVGAT Official Store or CJMCU carry most electronics components. Fans and SSR may come from separate sellers.

### Core electronics

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 1 | Raspberry Pi 4B 2GB | `Raspberry Pi 4 Model B 2GB` | 1 | $45 |
| 2 | Samsung PRO Endurance 32GB microSD | `Samsung PRO Endurance 32GB microSD` | 1 | $8 |
| 3 | DS18B20 waterproof sensor 1m | `DS18B20 waterproof temperature sensor 1m` | 4 | $2 each |
| 4 | Fotek SSR-25DA solid state relay | `Fotek SSR-25DA solid state relay 25A` | 1 | $8 |
| 5 | 5V signal relay SRD-5VDC-SL-C | `SRD-5VDC-SL-C 5V relay` | 1 | $2 |
| 6 | 120mm 12V PWM fan (duct) | `120mm 12V PWM fan 4 pin` | 1 | $5 |
| 7 | 92mm 12V PWM fan (circulation) | `92mm 12V PWM fan 4 pin` | 2 | $4 each |
| 8 | PTC ceramic heater 100W 230V | `PTC ceramic heater panel 100W 230V` | 1 | $6 |
| 9 | 1.3" color TFT ST7789 display | `1.3 inch TFT ST7789 SPI 240x240 color display` | 3 | $2.50 each |

### Power

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 10 | 12V 5A PSU module | `12V 5A switching power supply module bare board` | 1 | $6 |
| 11 | 5V 3A PSU module | `5V 3A switching power supply module bare board` | 1 | $5 |

### PCB components

| # | Item | Search term | Qty | ~Price |
|---|---|---|---|---|
| 12 | 40-pin GPIO female header 2.54mm | `2x20 pin female header 2.54mm raspberry pi` | 1 | $1 |
| 13 | Screw terminal blocks 3.5mm pitch | `PCB screw terminal block 3.5mm 2pin 3pin assorted` | 1 | $3 |
| 14 | Mains-rated screw terminals 5.08mm | `PCB screw terminal block 5.08mm 2pin` | 1 | $2 |
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

- **Pi 4B (#1):** Has dramatically more headroom than the Zero 2W — needed for three displays, PWM fan control, Flask, SQLite, Brewfather API, and future load cells running simultaneously. Confirm 2GB version from a seller with 1000+ orders.
- **Samsung PRO Endurance (#2):** Standard microSD cards fail from continuous SQLite writes within months. The PRO Endurance is rated for dashcam-style continuous write workloads.
- **DS18B20 sensors (#3):** Buy 4 — two per zone, averaged together for the control setpoint, with divergence alerting if the two readings in a zone separate by more than 2°C.
- **Fotek SSR-25DA (#4):** Solid state relay for the compressor. Silent switching, handles inductive motor loads cleanly, rated 25A at 230V. Mount on the inner wall of the enclosure with thermal paste — it generates heat under load.
- **PSU modules (#10, #11):** Bare board switching PSUs wired directly to mains inside the enclosure. Far more reliable than wall adapters for continuous operation. The 12V module powers all three fans. The 5V module powers the Pi via its 5V GPIO pins for cleaner, more stable power than USB-C.
- **Fans (#6, #7):** Must be **4-pin PWM**. Confirm listing shows four wires: red (12V), black (GND), blue (PWM), yellow (tach).
- **ST7789 display (#9):** Confirm listing states **ST7789 driver** and **240×240 resolution**. Similar-looking 128×128 screens exist and are incompatible.
- **Heat inserts (#26):** For the enclosure lid and fan mounts — heated in with a soldering iron, give far more durable threads than printed ones.
- **Conformal coating (#27):** Spray all PCBs before installation. The garage temperature swings and humidity make this non-negotiable.
- **Prototype HAT (#31):** Used in Phase 1 only. Replaced by the custom PCB in Phase 2.

---

## Component Details

### Raspberry Pi 4B

The brain of the system. Quad-core 1.8GHz ARM Cortex-A72, 2GB RAM, built-in WiFi and Bluetooth, full 40-pin GPIO header. Powered via the 5V GPIO pins directly from the onboard PSU module — cleaner and more stable than USB-C. Fit the heatsink kit before installing — the Pi 4B runs warm and the machine compartment limits airflow.

### DS18B20 temperature sensors

Waterproof probes using the 1-Wire protocol. All four sensors chain on a single GPIO pin (GPIO 4) with one shared 4.7kΩ pull-up resistor on the PCB. Each sensor has a unique 64-bit ROM address so they are independently addressable on the same bus.

Two sensors per zone, each submerged in a separate ~100ml sealed water container. The water acts as a thermal mass buffer — it takes several minutes to respond to air temperature changes, preventing false spikes from door openings. The two readings per zone are averaged for the control setpoint. If the two readings in a zone diverge by more than 2°C, a Discord alert fires — catching a sensor falling out of its housing, a fan failure, or a door left open before it becomes a problem.

Use small sealed glass jars (spice jars work perfectly) or 3D printed PETG housings. Feed each probe through a silicone-sealed hole in the lid. Fill with distilled or boiled water to prevent algae growth inside the sealed container.

### Fotek SSR-25DA

Solid state relay rated 25A at 230V AC, controlled by 3–32V DC on the input side. The Pi GPIO 3.3V signal drives it directly through a 1kΩ current-limiting resistor on the PCB. No moving parts — completely silent switching. Handles the inductive load of the compressor motor cleanly without the voltage spikes that mechanical relays produce. Mount on the inner wall of the machine compartment enclosure with thermal paste — it dissipates heat during operation.

### Signal relay (heater)

A PCB-mounted SRD-5VDC-SL-C relay switches the mains live wire to the PTC heater. Driven by a BC547 NPN transistor with a 1N4007 flyback diode across the coil to protect the GPIO from back-EMF. The heater switches infrequently so a mechanical relay is appropriate and simpler than a second SSR.

### PTC ceramic heater 100W

Self-regulating ceramic heating element at 230V mains. PTC (Positive Temperature Coefficient) means resistance increases as the element heats — it physically cannot overheat even if the relay fails to switch off. 100W is adequate for the fridge section volume. Mounts on the back wall of the fridge section on a printed bracket.

### Fans

Three 4-pin PWM 12V brushless fans — one 120mm duct fan and two 92mm circulation fans. Powered from the 12V PSU module. Speed controlled directly by the Pi 4B via hardware PWM GPIO at 25kHz, proportional to how far the zone temperature is from its target. The yellow tach wire is left unconnected — RPM monitoring can be added in a future version.

PWM fan wiring per fan:
- Red → 12V PSU output
- Black → common GND
- Blue → Pi GPIO PWM pin via 1kΩ resistor on PCB
- Yellow → not connected

### Displays

Three 1.3" color TFT screens using the ST7789 SPI driver at 240×240 resolution. All three share the SPI bus (SCLK, MOSI, DC, RST) with individual chip select pins. Driven by the `st7789` Python library. Cables routed from the machine compartment through M12 cable glands.

### Door sensors

Factory-fitted magnetic reed switches on both fridge doors. Each switch wires to a GPIO input pin with a 10kΩ pull-up resistor on the PCB. Normally closed (GPIO reads LOW). Opens (GPIO reads HIGH) when the door opens and the magnet moves away from the switch. Do not cut the original wires — splice into them so the fridge's own door-open indicator light continues to function.

---

## Custom PCB

> **Phase 2 of the build.** Design after the prototype is fully proven. Every GPIO assignment and circuit is confirmed on the breadboard before committing to FR4.

A HAT-style PCB that mounts directly on the Pi 4B's 40-pin GPIO header. Replaces all prototype jumper wires with proper traces. All external connections terminate in screw terminal blocks.

### What the PCB provides

- 4× DS18B20 sensor screw terminals (3-pin: VCC, GND, DATA) with shared 4.7kΩ pull-up
- 3× fan PWM output screw terminals (2-pin) with 1kΩ signal resistors
- SSR control output screw terminal (2-pin) with 1kΩ current-limiting resistor
- Heater relay circuit: BC547 transistor, 1N4007 diode, SRD-5VDC-SL-C relay, mains-rated screw terminals
- 2× door sensor screw terminals (2-pin each) with 10kΩ pull-up resistors
- 3× display chip select breakouts to screw terminals
- 5V rail breakout to screw terminals (for PSU module connection)
- Status LED with 330Ω resistor
- 100nF decoupling capacitors on 3.3V and 5V rails
- M3 mounting holes at all four corners for standoffs

### PCB design and ordering workflow

1. **Design in KiCad** — free and professional. This board is ~25 through-hole components — a good first KiCad project.
2. **DRC** — run design rule check before export to catch any shorts or clearance violations.
3. **Export Gerbers** — standard Gerber + drill files from KiCad's plot dialog.
4. **Order from JLCPCB** — 5 boards for ~$15 USD including shipping, 1–2 week turnaround. Order 5, build 2, keep 3 as spares.
5. **Solder** — all through-hole components, solderable by hand with basic equipment. No SMD required.
6. **Test outside the enclosure** — verify all GPIO signals with a multimeter and the prototype code before fitting to Pi.

### PCB component list

| Component | Part | Qty |
|---|---|---|
| GPIO connector | 2×20 female header 2.54mm | 1 |
| Sensor terminals | 3-pin screw terminal 3.5mm | 4 |
| Fan PWM terminals | 2-pin screw terminal 3.5mm | 3 |
| SSR control terminal | 2-pin screw terminal 3.5mm | 1 |
| Heater relay | SRD-5VDC-SL-C | 1 |
| Heater mains terminals | 2-pin screw terminal 5.08mm (mains rated) | 2 |
| Door sensor terminals | 2-pin screw terminal 3.5mm | 2 |
| Display CS terminals | 2-pin screw terminal 3.5mm | 3 |
| 5V rail terminal | 2-pin screw terminal 3.5mm | 1 |
| NPN transistor | BC547 | 1 |
| Flyback diode | 1N4007 | 1 |
| 1-Wire pull-up | 4.7kΩ 1/4W | 1 |
| Signal resistors | 1kΩ 1/4W | 5 |
| Door pull-up resistors | 10kΩ 1/4W | 2 |
| LED resistor | 330Ω 1/4W | 1 |
| Status LED | 5mm green LED | 1 |
| Decoupling caps | 100nF ceramic | 4 |
| Mounting hardware | M3 standoffs + screws | 4 |

---

## Machine Compartment Enclosure

The machine compartment sits at the bottom rear of the Samsung fridge behind the kick plate — ambient temperature, ventilated, already containing the compressor and its mains wiring.

### What goes inside

- Raspberry Pi 4B with PCB HAT
- Fotek SSR-25DA mounted on inner wall with thermal paste
- 12V PSU module (fans)
- 5V PSU module (Pi)
- Mains terminal block for live/neutral/earth distribution
- All cable management

### Printed enclosure design

PETG enclosure sized to the machine compartment. Print at 40%+ infill, 4+ perimeters. Use M3 heat-set threaded inserts for all screw points — far more durable than printed threads.

**Features to include:**
- Standoff mounts for Pi+PCB stack at a comfortable working height
- SSR mount on inner wall with cutout so the SSR body is exposed to compartment air for cooling
- PSU module mounts with strain relief for mains wiring
- Mains cable entry on one face with proper grommet
- Low-voltage cable exits on the opposite face — one pass-through per zone for sensor, fan, and display cables
- Ventilation slots on sides and top
- Lid fixed with M3 screws into heat-set inserts
- All external connection points labelled with embossed text in the print

**Inside cable separation:**
- Mains wiring (brown/blue/green-yellow) on one side of the enclosure
- All low-voltage wiring on the other side
- No mains and signal wires bundled together at any point

**Conformal coating:** Spray all PCBs before installation and allow to fully cure.

---

## Wiring

### GPIO pin assignment

```
# 1-Wire temperature sensors (all 4 on one bus)
GPIO 4   →  DS18B20 data bus (4.7kΩ pull-up to 3.3V on PCB)

# Mains switching
GPIO 17  →  SSR control in+ (compressor) via 1kΩ resistor
GPIO 27  →  Heater relay transistor base via 1kΩ resistor

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

# Display chip selects (one per screen)
GPIO 8   →  Screen 1 CS  (hardware CE0)
GPIO 7   →  Screen 2 CS  (hardware CE1)
GPIO 26  →  Screen 3 CS  (software CS)

# Status LED
GPIO 23  →  Status LED via 330Ω resistor

# Reserved for future use
GPIO 16  →  RESERVED — bubble sensor interrupt (v1.4)
```

> **Note:** GPIO 12, 13, and 19 are Pi 4B hardware PWM pins — use these for fans at 25kHz. Software PWM is not stable enough for fan control. Verify all assignments against the Pi 4B pinout before soldering the PCB.

### SSR wiring (compressor)

```
SSR input (+)   →  GPIO 17 (via 1kΩ resistor on PCB)
SSR input (−)   →  GND
SSR output (L1) →  mains live in from fridge socket
SSR output (T1) →  mains live to compressor
Neutral         →  passes through directly (not switched)
```

The SSR load terminals carry mains voltage — use minimum 1.5mm² wire and ensure all connections are fully insulated with heatshrink.

### Heater relay circuit (on PCB)

```
GPIO 27        →  1kΩ resistor → BC547 base
BC547 emitter  →  GND
BC547 collector →  relay coil (−)
Relay coil (+)  →  5V
1N4007 cathode  →  5V  (anode to collector — suppresses back-EMF)
Relay NO        →  mains live to PTC heater
Relay COM       →  mains live in
```

### Sensor wiring

```
DS18B20 red    →  3.3V
DS18B20 black  →  GND
DS18B20 yellow →  GPIO 4 (4.7kΩ pull-up to 3.3V on PCB)
```

All four sensors wire in parallel on the same three-wire bus. Each is individually addressed by its 64-bit ROM in software.

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
10kΩ pull-up from GPIO pin to 3.3V (on PCB)
```

Normally closed — GPIO reads LOW. Door open → switch opens → GPIO reads HIGH.

### Display wiring

```
VCC  →  3.3V
GND  →  GND
SCL  →  GPIO 11 (SCLK)
SDA  →  GPIO 10 (MOSI)
DC   →  GPIO 24
RST  →  GPIO 25
CS   →  GPIO 8 / 7 / 26 (one per screen)
```

---

## 3D Printed Parts

Print all parts in **PETG**. PLA becomes brittle in the temperature-cycling, humid garage environment within months. PETG handles condensation, temperature swings, and mechanical stress without complaint.

### Machine compartment enclosure

The main electronics housing in the machine compartment. See the Machine Compartment Enclosure section for full design requirements. Print at 40%+ infill, 4+ perimeters. Use M3 heat-set inserts for all fastener points.

### Duct shroud and frame

Fits in the ~125mm hole cut between compartments. Accepts the 120mm fan on the freezer face with clip retention for tool-free removal. Flanged lips on both sides seal against the divider wall with foam weatherstripping. Includes a return air slot on the opposite side of the divider to prevent pressure against the fan. Print at 40%+ infill, 3+ perimeters.

### Circulation fan brackets

Clip-on mounts for the 92mm circulation fans onto existing shelf rails or side wall features inside each zone — no extra drilling required. Angle fans slightly to circulate air around the zone rather than blowing directly at the fermenter or kegs.

### Sensor water housings

Two sealed cylindrical containers (~100ml each), one per zone. Press-fit or screw cap with a cable port for the probe leads. Print at 60%+ infill, 4+ perimeters for water tightness. Seal cap and cable port with food-safe silicone. Fill with distilled or boiled water.

### PTC heater bracket

Wall-mount bracket for the PTC heater panel in the fridge section. Holds the heater away from the wall for airflow clearance, with cable routing channel. Print at 40%+ infill.

### Display bezels

Rectangular frame per screen with a rear lip holding the screen flush. Screen 1 mounts to the fridge exterior with VHB tape. Screens 2 and 3 mount at each tap position. Include a cable channel in the bezel for routing the display cable to the M12 cable gland pass-through.

### Cable routing clips

Small saddle clips with 3M VHB tape backing for routing sensor and display cables along fridge interior walls away from door seals and hinge areas.

### Cable pass-through plates

Cover plates over the holes where cables exit each fridge zone into the machine compartment. Each plate accepts two M12 cable glands — one for sensor cables, one for fan and display cables.

### Future parts

**Keg load cell platform (v1.1):** Flat sled per keg with four recesses for 50kg load cells at the corners and alignment lips to prevent the keg sliding. Print at 60%+ infill — takes the full weight of a full keg.

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

# GPIO — mains switching
SSR_COMPRESSOR  = 17
RELAY_HEATER    = 27

# GPIO — PWM fans (hardware PWM pins only)
PWM_DUCT_FAN    = 12
PWM_FREEZE_CIRC = 13
PWM_FRIDGE_CIRC = 19
PWM_FREQ        = 25000  # Hz

# GPIO — door sensors
DOOR_FREEZER    = 5
DOOR_FRIDGE     = 6

# GPIO — status LED
STATUS_LED      = 23

last_compressor_off = 0
door_opened_at  = {DOOR_FREEZER: None, DOOR_FRIDGE: None}
door_closed_at  = {DOOR_FREEZER: None, DOOR_FRIDGE: None}

GPIO.setmode(GPIO.BCM)
GPIO.setup(DOOR_FREEZER, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(DOOR_FRIDGE,  GPIO.IN, pull_up_down=GPIO.PUD_UP)

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

def door_is_open(pin):
    return GPIO.input(pin) == GPIO.HIGH

def recently_closed(pin):
    closed_at = door_closed_at.get(pin)
    if closed_at is None:
        return False
    return (time.time() - closed_at) < DOOR_SETTLE_SECS

def handle_door_events():
    for pin, name in [(DOOR_FREEZER, "Freezer"), (DOOR_FRIDGE, "Fridge")]:
        if door_is_open(pin):
            if door_opened_at[pin] is None:
                door_opened_at[pin] = time.time()
                log_door_event(name, "opened")
            open_duration = time.time() - door_opened_at[pin]
            if open_duration > DOOR_ALERT_SECS:
                post_discord("🚪 Door alert",
                    f"{name} door open for {int(open_duration/60)} minutes",
                    COLOR_WARN)
        else:
            if door_opened_at[pin] is not None:
                door_closed_at[pin] = time.time()
                door_opened_at[pin] = None
                log_door_event(name, "closed")

def control_loop():
    global last_compressor_off

    handle_door_events()

    # Pause control if any door is open or in settle window after closing
    if (door_is_open(DOOR_FREEZER) or door_is_open(DOOR_FRIDGE)
            or recently_closed(DOOR_FREEZER) or recently_closed(DOOR_FRIDGE)):
        for pwm in [duct_pwm, freeze_pwm, fridge_pwm]:
            pwm.ChangeDutyCycle(0)
        return

    # Average two sensors per zone
    t_freezer = (read_sensor('freezer_1') + read_sensor('freezer_2')) / 2
    t_fridge  = (read_sensor('fridge_1')  + read_sensor('fridge_2'))  / 2
    fridge_target = get_fridge_target()  # from active Brewfather profile

    # Alert if sensors in same zone diverge
    if abs(read_sensor('freezer_1') - read_sensor('freezer_2')) > 2.0:
        post_discord("🌡️ Sensor alert",
            "Freezer sensors diverging — check water housings", COLOR_WARN)
    if abs(read_sensor('fridge_1') - read_sensor('fridge_2')) > 2.0:
        post_discord("🌡️ Sensor alert",
            "Fridge sensors diverging — check water housings", COLOR_WARN)

    # --- Freezer / compressor ---
    freeze_error = t_freezer - FREEZER_TARGET
    if freeze_error > HYSTERESIS_FREEZE:
        if time.time() - last_compressor_off > COMPRESSOR_MIN_OFF:
            GPIO.output(SSR_COMPRESSOR, GPIO.HIGH)
            freeze_pwm.ChangeDutyCycle(fan_speed(abs(freeze_error)))
    elif freeze_error < -HYSTERESIS_FREEZE:
        if GPIO.input(SSR_COMPRESSOR) == GPIO.HIGH:
            last_compressor_off = time.time()
        GPIO.output(SSR_COMPRESSOR, GPIO.LOW)
        freeze_pwm.ChangeDutyCycle(0)

    # --- Fridge / fermentation (hard interlock: never heat and cool together) ---
    fridge_error = t_fridge - fridge_target
    if fridge_error > 0.3:
        GPIO.output(RELAY_HEATER, GPIO.LOW)
        duct_pwm.ChangeDutyCycle(fan_speed(fridge_error))
        fridge_pwm.ChangeDutyCycle(fan_speed(fridge_error))
    elif fridge_error < -0.3:
        duct_pwm.ChangeDutyCycle(0)
        GPIO.output(RELAY_HEATER, GPIO.HIGH)
        fridge_pwm.ChangeDutyCycle(50)  # gentle circulation while heating
    else:
        duct_pwm.ChangeDutyCycle(0)
        GPIO.output(RELAY_HEATER, GPIO.LOW)
        fridge_pwm.ChangeDutyCycle(0)

    log(time.time(), t_freezer, t_fridge, fridge_target)
    time.sleep(30)
```

### Safety rules

- Compressor has a hard 10-minute minimum off time tracked in software
- Heater and duct fan are mutually exclusive — hard interlocked in code
- All outputs default to OFF (SSR low, relay low, fans 0%) on startup and crash
- Fan speed is proportional to temperature error — no abrupt on/off switching
- Door open events pause all fans and skip the control loop entirely
- Sensor divergence in the same zone triggers a Discord alert
- All state changes logged to SQLite with timestamp

---

## Software Stack

### OS and runtime

- **Raspberry Pi OS Lite 64-bit** — headless, no desktop
- **Python 3** with `RPi.GPIO`, `w1thermsensor`, `sqlite3`, `st7789`, `Flask`, `requests`
- Control loop runs as a `systemd` service — auto-restarts on boot or crash

### Data storage

```sql
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
alert_minutes = 5
settle_seconds = 45
```

---

## Brewfather Integration

The Pi connects to Brewfather only when you press the button in the web UI — no background polling. Once imported, the profile runs entirely from local SQLite.

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
○ Pilsner           — Planning
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

### Flask endpoints

```python
@app.route("/brewfather/batches")
def fetch_batches():
    auth = (config["brewfather"]["user_id"],
            config["brewfather"]["api_key"])
    r = requests.get(
        "https://api.brewfather.app/v2/batches",
        params={"limit": 10}, auth=auth
    )
    return jsonify(r.json())

@app.route("/fermentation/start", methods=["POST"])
def start_fermentation():
    data = request.json
    steps = data["recipe"]["fermentation"]["steps"]
    pitch_date = data["pitch_date"]
    db.execute("DELETE FROM fermentation_profile")
    db.execute("DELETE FROM fermentation_meta")
    db.execute("INSERT INTO fermentation_meta VALUES (?,?,?,?,1)",
        (data["_id"], data["name"], data["recipe"]["name"], pitch_date))
    for i, step in enumerate(steps):
        db.execute("INSERT INTO fermentation_profile VALUES (?,?,?,?)",
            (i, step["name"], step["stepTemp"], step["stepTime"]))
    return jsonify({"status": "started"})
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
    return steps[-1]["target_temp"]  # hold at last step when complete

def get_current_step():
    """Return the current active fermentation step dict."""
    meta = db.execute(
        "SELECT pitch_date FROM fermentation_meta WHERE active=1"
    ).fetchone()
    if not meta:
        return None
    pitch_date = datetime.fromisoformat(meta["pitch_date"])
    days_fermenting = (datetime.now(timezone.utc) - pitch_date).days
    steps = db.execute(
        "SELECT * FROM fermentation_profile ORDER BY step_index"
    ).fetchall()
    elapsed = 0
    for step in steps:
        elapsed += step["duration_days"]
        if days_fermenting < elapsed:
            return dict(step)
    return dict(steps[-1])

# Track which step was active on the last loop iteration
last_step_index = None

def check_for_step_change():
    """Detect step transitions and fire appropriate Discord notifications."""
    global last_step_index
    step = get_current_step()
    if step is None:
        return
    if step["step_index"] == last_step_index:
        return  # still on the same step

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

Call `check_for_step_change()` at the top of the control loop on each iteration. It is lightweight — just a DB read and a comparison — and fires a Discord message only when the step actually changes.

---

## Discord Notifications

> **v1.3 feature.** No hardware changes required — purely a software addition.

Create a webhook URL in your Discord server (channel settings → Integrations → Webhooks) and add it to `config.ini`. If not set, all Discord calls are silently skipped.

### What gets posted

**Fermentation:** Profile started (with all steps listed), step changes with new target and duration, cold crash start with blowoff tube reminder, fermentation complete, temperature out of range, temperature recovered.

**Doors:** Door left open beyond threshold (configurable, default 5 minutes).

**System:** Pi restarted, sensor divergence alert, compressor running unusually long.

**Daily digest:** Both zone temps, active fermentation step and days remaining, keg levels once load cells are added.

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

### Setup steps

1. Create a `#kegerator` channel in your Discord server
2. Channel settings → Integrations → Webhooks → New Webhook
3. Copy the webhook URL
4. Paste into `/etc/kegerator/config.ini` under `[discord]`
5. Done — no bot token, no OAuth, no API registration needed

---

## Physical Installation

### Machine compartment

Remove the kick plate at the bottom rear of the Samsung fridge. Mount the printed electronics enclosure on the floor or side wall of the compartment. Route all cables before closing the enclosure lid — sensor and display cables run up through the back panel into each fridge zone. Plan cable lengths before cutting.

### Duct fan

Cut an ~125mm hole through the divider wall between compartments. The printed duct shroud clips into the hole with the 120mm fan on the freezer face. Seal the shroud flanges to the divider with foam weatherstripping tape — uncontrolled passive air mixing between zones causes the temperature control loop to fight itself. Add a return air slot on the opposite end of the divider so air can flow back from fridge to freezer without building pressure against the fan.

### Circulation fans

Mount the freezer fan low on a side wall, blowing horizontally across the bottom of the keg zone — cold air settles at the bottom and cross-circulation is more effective than top-down. Mount the fridge fan high on the back wall blowing down across the fermenter, with the duct fan inlet on the opposite side so cold air travels the full length of the zone.

### Sensor placement

Mount the water housings in each zone using VHB tape or the printed bracket. Freezer: mid-height between kegs, away from the evaporator coil and direct fan airflow. Fridge: at the same height as the fermenter body, away from the duct fan outlet. Both need to read representative air temperature, not airstream temperature.

### Door sensors

Locate the factory reed switches in the door frames — small rectangular or cylindrical components near the hinge area with two thin wires. Splice into both wires with thin cable (20–24 AWG) and route along the door frame, down the back of the fridge, and into the machine compartment. Do not cut the original wires — splicing in preserves the fridge's own door-open indicator light.

### PTC heater

Mount the PTC heater on the back wall of the fridge section at mid-height using the printed bracket. Keep it away from the fermenter and the circulation fan outlet. The signal relay in the machine compartment enclosure switches its mains supply via the cable gland pass-through.

### Display cables

Route display cables from the machine compartment up through M12 cable glands into the fridge body. Screen 1 exits to the fridge exterior panel. Screens 2 and 3 route to the tap positions — plan cable runs before finalising where the taps are mounted.

### Cable discipline

Keep mains wiring (compressor, heater, PSU inputs) physically separated from low-voltage wiring (sensors, fans, displays) throughout the machine compartment and all cable runs. Never bundle them together. Use M12 cable glands at every wall penetration to prevent condensation wicking along cables.

---

## Build Phases

### Phase 1 — Prototype

Get everything working with the Pi 4B and the prototype HAT with jumper wires. Validate all GPIO assignments, test the control loop with real sensors and real temperatures, confirm PWM fan control at all speed levels, verify all three displays, test door sensor logic including the settle window, confirm Brewfather API calls work end to end. This is the development environment — expect to revise wiring. **Do not design the PCB until Phase 1 is completely stable.**

### Phase 2 — Custom PCB

Once Phase 1 is fully proven with no further wiring changes needed, design the PCB in KiCad replicating exactly what worked. Order from JLCPCB (5 boards, ~$15 USD). Solder one board, test it fully outside the enclosure before installing. Keep the prototype HAT as a backup.

### Phase 3 — Machine compartment integration

Design the PETG enclosure around the confirmed PCB and all component positions. Print, test fit, adjust if needed. Make all final cable runs through the fridge. Apply conformal coating to all PCBs. Install everything in the machine compartment, seal cable glands, close the enclosure, replace the kick plate.

---

## Roadmap

### v1.0 — MVP (this build)
- Temperature control: freezer at 1–2 °C, fridge following Brewfather fermentation profile
- SSR for compressor, signal relay for PTC heater, PWM for all fans
- Dual DS18B20 per zone with averaging and divergence alerting
- Door sensors: pause control on open, settle window on close, log events
- Brewfather integration: on-demand connect, batch selection, manual profile start
- Three displays: fermentation status + two tap info screens
- SQLite logging including door events and fan duty cycles
- Local Flask web dashboard
- Phase 1: prototype on HAT → Phase 2: custom PCB → Phase 3: machine compartment install

### v1.1 — Keg weight
- HX711 ADC modules + 50kg load cells per keg
- Printed load cell sleds per keg
- Live % remaining on tap displays and web dashboard
- Pour volume estimation from weight delta
- Low keg Discord notification

### v1.2 — Dashboard polish
- Temperature history graphs with zone and setpoint overlays
- Compressor runtime tracking (maintenance indicator)
- Days-since-clean alerts with Discord notifications
- Pour history log
- Door open frequency and duration statistics

### v1.3 — Notifications and remote access
- Full Discord notification suite: fermentation events, daily digest, door alerts, sensor alerts
- Tailscale for secure remote dashboard access from anywhere
- Push notifications for temperature out of range and overdue cleaning

### v1.4 — Bubble detection and Tilt integration
- TCRT5000 IR photoelectric sensor clipped onto blowoff tube
- Hardware interrupt on GPIO 16 counts bubbles in real time
- Bubble rate (bubbles per minute) logged to SQLite alongside temperature
- Fermentation activity graph on web dashboard
- Discord notification when fermentation becomes active after pitching
- Stuck fermentation alert: bubble rate at zero for unexpectedly long period before expected end of primary
- Fermentation complete detection from sustained near-zero bubble rate
- 3D printed black PETG clip-on housing holding IR LED and phototransistor on opposite sides of blowoff tube — opaque to block external light interference
- **Optional Tilt hydrometer integration** — Pi 4B receives Tilt BLE broadcasts over built-in Bluetooth using `aioblescan`. Tilt gravity and temperature displayed on dashboard and logged to SQLite as a supplementary data source. No extra hardware needed. Tilt data is treated as advisory only — the primary control loop and fermentation activity detection always run from DS18B20 temperature and bubble rate regardless of Tilt status. This handles the two known failure modes gracefully: Tilt stuck in krausen (wildly inaccurate gravity readings ignored, bubble rate still valid), and dead Tilt battery (system continues normally with no degradation). When Tilt data is present and plausible it enriches the dashboard with a gravity curve and attenuation percentage. When absent or suspect it is simply not shown.
