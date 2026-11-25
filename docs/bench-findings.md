# Bench findings — actual 1:36 PCB

Everything in this file comes from direct visual inspection, continuity testing or voltage measurements on the actual car PCB unless explicitly marked otherwise.

## Main board components seen on the PCB

| Marking / part | Package / location | What is directly known |
|---|---|---|
| `PAN7420S7FA` | SSOP-24 | Main MCU / radio SoC. |
| `HRZP` | SOT-23 | Receives switched battery voltage and produces the ~3.0 V rail used by PAN. |
| `A1LA` | SOT-23 | Behaves as the main high-side power switch. |
| `Y1` ×4 | SOT-23 | Three are low-side switches for light outputs; the fourth is part of the power-hold circuit. |
| `6?18A 2548/33` with Microne logo | SOT-223 | Produces a 3.3 V rail used by the light connectors. Exact variant/pin naming not fully resolved. |
| `CL4056D` | IC | Present near USB-C / charge section. |
| `7122P280` | small IC | Strong IMU candidate; exact device not identified yet. |
| unmarked SOP-8 | SOP-8 | Motor H-bridge / motor driver. |
| `S4` ×2 | 2-pin SMD | Function not identified. |
| resonator / crystal | — | Present for PAN/radio clocking. |
| PCB antenna | — | Stock 2.4 GHz antenna. |
| USB-C | — | Charging input. |
| 3 small light connectors | — | Each has a common 3.3 V positive supply and independently switched negative side. |
| servo connector | — | Servo power is battery voltage; signal is controlled by PAN. |
| battery connector | ZH-like 1.5 mm | 1S battery input. |

## Power measurements

### PAN supply

Measured relative to board GND:

| State | PAN pin 8 | PAN pin 11 |
|---|---:|---:|
| ON | ~3.0 V | ~3.0 V |
| OFF | ~50 mV, high-impedance / drifting | same behavior |

More precise ON measurement later:

- PAN pin 8: **2.99 V**
- PAN pin 19: **2.77 V**

The OFF rail behaves like a genuinely disconnected / high-impedance rail rather than a sleeping MCU rail.

### A1LA / HRZP measurements

Orientation used during measurement:

- SOT-23 "single" = the single leg on one side;
- "paired 1 / paired 2" = the two legs on the opposite side, left-to-right in the measurement orientation.

Measured relative to GND:

| State | BAT+ | A1LA single | A1LA paired 1 | A1LA paired 2 | HRZP paired 1 | HRZP paired 2 |
|---|---:|---:|---:|---:|---:|---:|
| ON | 3.7 V | 3.7 V | 0 V | 3.7 V | 0 V | 3.0 V |
| OFF | 3.7 V | 0 V | 3.7 V | 3.7 V | 0 V | 0 V |

The A1LA single leg is directly connected to the HRZP single leg.

This gives the observed power path:

```text
BAT+
  |
A1LA power switch
  |
switched battery voltage
  |
HRZP
  |
~3.0 V
  |
PAN7420
```

The control node of A1LA is at ~BAT+ while OFF and is pulled to GND while ON.

## Hardware power latch

This part is now mapped well enough to be useful.

### What was verified

- The button can pull the A1LA control node to GND and start the board.
- A `Y1` transistor next to A1LA can also pull that same node to GND.
- The control input of this power `Y1` is connected through a resistor to **PAN pin 19**.
- PAN pin 19 measures **2.77 V while ON**.
- Therefore PAN pin 19 is the MCU's **power HOLD** output; active state is HIGH.

Observed behavior:

```text
OFF:
A1LA control node ~= BAT+
A1LA blocks power

button pressed:
A1LA control node -> GND
A1LA enables power
PAN starts

PAN pin 19 HIGH:
opens power Y1
Y1 keeps A1LA control node at GND
button can be released

shutdown:
PAN pin 19 stops driving Y1
A1LA control node returns high
A1LA cuts power
```

A pull-up from the A1LA control node toward BAT+ is strongly implied by the measured OFF state, but the exact resistor has not been individually documented yet.

## PAN pin map recovered on the board

PAN pin numbers follow the standard SSOP numbering from the pin-1 mark, counter-clockwise when viewed from the top of the package.

| PAN pin | Board function | Evidence |
|---:|---|---|
| 2 | Light channel 2 control | traced through resistor and Y1 |
| 3 | Motor H-bridge input A | direct continuity to driver pin 2 |
| 4 | Motor H-bridge input B | direct continuity to driver pin 3 |
| 5 | IMU/test-pad signal | direct continuity |
| 6 | IMU/test-pad signal | direct continuity |
| 7 | Light channel 3 control | traced through resistor and Y1 |
| 8 | PAN VDD | measured ~2.99 V ON |
| 11 | PAN VDD / RF supply | measured ~3.0 V ON |
| 12 | GND | continuity to BAT- / board GND |
| 15 | Light channel 1 control | traced through resistor and Y1 |
| 18 | Servo signal | traced through a resistor to servo signal |
| 19 | Power HOLD | traced through resistor to power Y1; 2.77 V ON |
| 20 | RC-connected to the 3.0 V / HRZP-output area | traced; exact function unknown; **not the HOLD pin** |
| 21 | Battery voltage ADC input | midpoint of two-resistor divider; measured approximately VBAT / 2 |

### Important correction preserved here

PAN pin 7 controls **LED3**.

PAN pin 18 controls the **servo signal**.

## Motor driver

The unmarked SOP-8 device is used as the motor H-bridge.

Known by continuity:

- PAN pin 3 -> driver input pin 2;
- PAN pin 4 -> driver input pin 3;
- one driver pin has direct continuity to BAT+;
- two driver pins have direct continuity to BAT- / GND;
- no driver pin was found directly tied to the 3.3 V lighting rail.

This strongly indicates that the motor power stage is supplied directly from the battery, not from the 3.3 V lighting rail.

The exact motor-driver part number is still unknown.

## Servo

Verified:

- servo power is approximately **battery voltage**, not 3.3 V;
- servo signal -> resistor -> **PAN pin 18**.

The exact stock pulse timing has not yet been captured.

## Lighting

There are three independently controlled light connectors.

Verified topology for each channel:

```text
3.3 V rail -> light connector "+"
light connector "-" -> Y1 switched node
Y1 -> GND when enabled
Y1 control <- resistor <- PAN GPIO
```

PAN control pins:

- PAN 15 -> LED1
- PAN 2 -> LED2
- PAN 7 -> LED3

The 3.3 V positive rail for these connectors comes from the Microne `6118`-marked regulator area.

The negative pins of the light connectors are therefore **not permanent GND**.

## Y1 behavior

All tested Y1 devices use the same board-level pattern:

- one paired leg -> BAT- / GND;
- the other paired leg -> control signal;
- the single leg -> switched load node;
- when the control is active, the switched node is pulled toward BAT-.

The exact transistor type / BJT-vs-MOSFET identity has not been proven from marking alone, so the documentation intentionally describes the observed behavior instead of assigning transistor terminal names.

## Battery measurement

PAN pin 21 is connected to the midpoint of a two-resistor divider from battery voltage.

Observed:

```text
BAT+ -- R --+-- PAN21
            |
            R
            |
           GND
```

The resistors appear to form an approximately 1:2 divider, and the live measurement matched approximately:

```text
PAN21 ~= VBAT / 2
```

This allows the MCU to measure a 1S LiPo without exposing the ADC input to full battery voltage.

## IMU / four large test pads

Four large pads near the probable IMU were traced.

With the car oriented forward, pad order is:

```text
GND | 3.0 V from HRZP | PAN6 | PAN5
```

This is consistent with a 4-wire digital IMU interface such as I2C, but that protocol has **not yet been captured electrically**.

The device marking seen near this area is:

```text
7122P280
```

Exact identification is still open.

## 3.3 V regulator / light rail

The SOT-223 Microne-marked part reads approximately:

```text
6?18A
2548/33
```

Its exact suffix / pin naming is not considered resolved.

What is directly verified and useful:

- a **3.3 V** rail exists while the car is ON;
- this rail feeds the positive pins of the three lighting connectors;
- this rail is not the motor-driver supply;
- PAN itself is supplied from the separate **3.0 V HRZP rail**.

## Charging section

Observed on the PCB:

- USB-C input;
- `CL4056D`;
- red and green charging/status LEDs;
- 1S battery connector.

The detailed charger wiring has not been reverse engineered because it is not currently necessary for control or PAN reflashing.

## SWD pads

The PCB has four programming/test pads marked:

```text
G V C D
```

Their exact electrical identity has not yet been proven by the same continuity-test standard used above.

The working hypothesis from silkscreen plus PAN documentation is:

```text
G = GND
V = target VDD / reference
C = SWCLK
D = SWDIO
```

This belongs to the external-research / next-test category until continuity to the PAN SWD pins is explicitly recorded.
