# Design notes, hypotheses and open questions

This file is intentionally allowed to contain design ideas, inferred architecture and future directions. It is **not** the same confidence level as [bench-findings.md](bench-findings.md).

## Preferred architecture: reflash PAN7420

Current preferred direction:

```text
keep original PCB
keep PAN7420
keep stock RF / antenna
keep stock power latch
keep stock charger
keep stock H-bridge
keep stock servo
keep stock lights
keep stock battery divider
keep stock IMU if useful

replace only firmware
```

Why this became preferable:

- the important PAN GPIO mapping is already known;
- the power HOLD signal is known;
- the board does not need invasive rewiring;
- PAN7420 has SWD pins;
- Panchip distributes PY32F030-related development resources;
- the stock radio hardware remains physically matched to the antenna / board.

### First firmware responsibilities

A replacement firmware should bring up hardware in a safe order:

1. configure PAN19 / HOLD quickly enough that the board stays powered after the button is released;
2. keep motor inputs in a safe neutral/coast state during startup;
3. initialize servo output;
4. initialize battery ADC;
5. initialize IMU if used;
6. initialize radio;
7. enable lights / application logic.

### Motor PWM goal

One original motivation is to eliminate audible motor whine.

Target to test:

```text
motor PWM ~= 20-30 kHz
```

Exact best frequency should be validated against:

- H-bridge switching losses / heating;
- motor heating;
- useful duty-cycle resolution;
- low-speed behavior.

Forward/reverse must preserve the H-bridge's required truth table. Do not assume "PWM one pin, static direction on the other" until the actual driver behavior is captured or tested safely.

## SWD plan

Hardware target pads are expected to be:

```text
G -> GND
V -> target reference voltage
C -> SWCLK
D -> SWDIO
```

First connection should be deliberately non-destructive:

```text
probe GND -> board GND
probe SWCLK -> C
probe SWDIO -> D
car powered normally from its battery
```

Do not start with erase / unlock operations.

Desired first results:

- detect Cortex-M core;
- halt / resume;
- read core registers;
- inspect memory;
- attempt to dump `0x08000000..0x0800FFFF`.

If flash is read-protected, decide separately whether losing the original firmware is acceptable before any mass erase.

### Probe candidates

Lowest-friction candidate:

```text
XIAO RP2040 -> Raspberry Pi debugprobe firmware -> CMSIS-DAP
```

ESP32-S3 is also viable with community firmware, but is not necessary if an RP2040 is available.

## PC transmitter: conservative first version

Recommended first radio prototype:

```text
PC / ROS2
   |
USB
   |
XIAO ESP32-S3 / RP2040 / similar
   |
SPI
   |
nRF24L01+ mini module
   |
2.4 GHz
   |
stock PAN7420 car firmware
```

Why use a real nRF24 first:

- the open-source Jiabaile implementation already targets nRF24L01;
- XN297 emulation code is already available;
- this avoids debugging a new radio PHY port at the same time as testing whether the 1:36 protocol matches.

### nRF24 wiring

Required:

```text
3V3
GND
CE
CSN
SCK
MOSI
MISO
```

`IRQ` is optional for the first implementation.

A local decoupling capacitor near the RF module (for example 10-47 uF plus the module's existing ceramics) is a practical robustness measure.

### 1:36 protocol uncertainty

The existing Multiprotocol implementation has `JIABAILE_STD` and `JIABAILE_GYRO`, but the actual 1:36 PAN7420 car has not been tested against it.

Possible outcomes:

1. **bind works immediately** — use the existing implementation;
2. **same PHY, different bind / packet details** — adapt the existing implementation;
3. **different PHY** — capture / reverse the stock 1:36 remote.

Because PAN742 SDK exposes XN297L-compatible radio examples, outcome 2 is considered plausible, but this is still a hypothesis.

## PC / ROS 2 USB transport

For a Linux ROS 2 host, a clean design is:

```text
rclcpp node
  |
/dev/hidrawX
  |
USB HID interrupt OUT
  |
XIAO
```

Potential command report:

```text
seq
car_id
throttle
steering
gyro
flags
reserved...
```

### Latency design rule: latest command wins

Do not queue old steering/throttle commands.

Recommended model:

```text
USB packet arrives
        |
latest_command = packet
        |
RF scheduler reads latest_command at its next slot
```

If ten PC commands arrive before the next RF slot, older commands should be discarded rather than queued.

This prevents command backlog from turning temporary host jitter into persistent control latency.

### USB timing

A 1 ms HID interrupt interval is a reasonable first target.

The RF timing should be generated on the XIAO by a hardware timer / deterministic task, not by asking Linux/Windows to sleep for the exact Jiabaile packet period.

For the existing `JIABAILE_GYRO` implementation the packet period is 8205 us, so the stock protocol itself is likely a larger latency contributor than SPI or USB.

## Multiple cars

A single radio can in principle time-multiplex multiple receiver addresses/channels.

However, **multi-car operation with the stock 1:36 protocol has not been validated**.

If PAN firmware is replaced on every car, a custom multi-car protocol becomes much easier to design explicitly:

```text
car_id
sequence
throttle
steering
gyro_gain
lights
flags
checksum / CRC
```

Potential additions:

- explicit failsafe timeout;
- acknowledgements;
- battery telemetry;
- per-car packet schedule.

Do not claim a specific car count or update rate until measured.

## One-board Nordic transmitter

A XIAO nRF52840 / nRF54 device is attractive because it combines:

- USB;
- MCU;
- 2.4 GHz radio;
- antenna;

on one small PCB.

Concept:

```text
PC -> USB -> XIAO Nordic -> direct proprietary 2.4 GHz -> car
```

This is aesthetically better than XIAO + nRF24, but currently higher risk because the known XN297/Jiabaile code must be ported from nRF24 register behavior to Nordic RADIO/ESB primitives and validated over the air.

Suggested order:

1. prove the protocol using a real nRF24L01;
2. only then port to Nordic if a one-board transmitter is worth the effort.

## Fallback: remove PAN and install a XIAO

This remains a viable Plan B if PAN programming becomes impractical.

Known nets available at former PAN pads would provide:

```text
motor INA
motor INB
servo signal
LED1
LED2
LED3
power HOLD
battery ADC divider
GND
power rail
IMU bus if desired
```

This would preserve the original PCB as a carrier / power / actuator board.

### Important power caveat

Do not assume every XIAO variant accepts power in exactly the same way.

In particular, newer XIAO Nordic boards may document their `3V3` pin as an output. A replacement-MCU design should verify the exact XIAO schematic before feeding the stock ~3.0 V rail into a board power pin.

This is one reason reflashing PAN is currently cleaner than replacing it.

## IMU direction

Current board evidence exposes:

```text
GND | 3.0 V | PAN6 | PAN5
```

near the probable IMU.

The PAN datasheet gives pin 5/6 alternate functions compatible with SCL/SDA, so I2C is a strong hypothesis.

If the stock firmware is still intact, the best time to capture the bus is **before reflashing**:

- logic analyzer;
- 3.3 V-compatible inputs;
- capture startup and steady-state traffic;
- determine SCL vs SDA, address and register sequence.

This is optional if a new firmware can instead use another known IMU or if the stock chip is identified from its marking.

## Battery-voltage handling

PAN21 already has a working external divider approximately equal to VBAT/2.

A replacement PAN firmware can reuse it directly.

Firmware should implement a conservative low-voltage policy for the 1S battery. The exact cutoff strategy should be based on the battery pack/protection behavior rather than assuming that the existing battery always contains its own protection PCB.

## Measurement / lab notes

Useful equipment for this project:

- multimeter;
- 2-channel oscilloscope;
- logic analyzer, preferably >= 20 MS/s;
- CMSIS-DAP / J-Link-class SWD probe;
- microscope / fine probes;
- current-limited bench supply.



### Lab equipment already discussed

The available lab image / inventory discussed in this project included:

- UNI-T UTD2102CM oscilloscope — 2-channel DSO, suitable for motor/servo timing work;
- UNI-T UTP3305 bench supply — useful for current-limited power-up tests;
- UNI-T UT161E multimeter;
- Ersa i-CON1 soldering station;
- ARINST VNA-DL 1-8800 MHz — potentially useful later for RF/antenna work, not required for the first firmware steps.

A separate logic analyzer is still useful because the oscilloscope is not an MSO. For this project, 8 channels, 3.3 V logic support, >=20 MS/s and I2C/SPI/UART decoding are sufficient for most reverse-engineering tasks.

### H-bridge oscilloscope warning

Both motor terminals may be switching.

Do **not** casually connect a grounded oscilloscope ground clip to one motor terminal.

Safer approaches:

- CH1 from motor terminal A to board GND;
- CH2 from motor terminal B to board GND;
- math `CH1 - CH2`;

or use a proper differential probe.

## Things deliberately not treated as facts yet

- exact identity of `7122P280`;
- exact identity of the SOP-8 motor driver;
- exact identity of `Y1`;
- exact role of PAN20;
- exact 1:36 RF packet format;
- whether stock PAN flash is read-protected;
- whether the PY32F030 flash loader works unchanged on this PAN7420;
- whether XIAO Nordic can reproduce the stock RF format without an external nRF24;
- exact maximum number of simultaneously controlled cars.
