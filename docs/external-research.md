# External research

This file contains facts and useful existing work found online. These are deliberately separated from measurements on the actual 1:36 PCB.

## PAN7420 / PAN742 family

### Datasheet / pinout

PAN7420 family datasheet used during the reverse engineering:

- https://b2b.big-bit.com/file/upload/202408/02/15584691322307.pdf

Relevant PAN7420S7FA SSOP-24 pin functions used in this project include:

| Package pin | PAN function |
|---:|---|
| 1 | PA14 / SWCLK |
| 2 | PB03 / SCK |
| 3 | PB04 / MISO |
| 4 | PB05 / MOSI |
| 5 | PB06 / TX, SCL |
| 6 | PB07 / RX, SDA |
| 7 | PB08 / SCL, TX |
| 8 | VDD1 / MCU |
| 9 | XC1 |
| 10 | XC2 |
| 11 | VDD2 / RF |
| 12 | GND |
| 13 | ANT |
| 14 | PF02 / NRST |
| 15 | PA03 / RX, SCL |
| 16 | PA04 / ADC |
| 17 | PA05 / ADC |
| 18 | PA06 / ADC |
| 19 | PA07 / ADC, TX |
| 20 | PB00 / ADC |
| 21 | PB01 / ADC |
| 22 | PB02 / RX |
| 23 | PA08 / PWM |
| 24 | PA13 / SWDIO |

This makes the board pads marked `C` and `D` plausible candidates for SWCLK/SWDIO, but that board-level mapping is still to be explicitly verified.

### Official PAN742 SDK

Panchip publishes a PAN742 development package:

- PAN742_DK_Public-V0.0.9:
  https://bbs.panchip.com/forum.php?mod=viewthread&tid=7973

Important items listed by Panchip in that package:

- PAN742 reference schematic;
- PAN742 product documentation;
- PY32F030 datasheet and reference manual;
- PY32F0xx DFP documentation / MDK support;
- BootLoader;
- DongleX;
- WirelessProgrammer;
- examples named:
  - `pan1026_normal_mode_compatible_with_xn297l`
  - `pan1026_enhance_mode_compatible_with_xn297l`

This is strong evidence that the PAN742 MCU side is based on the PY32F0xx/PY32F030 ecosystem and that its radio stack has explicit XN297L-compatible modes.

## PY32F030 / programming path

OpenPuya/Puya documentation:

- PY32F030 overview:
  https://py32.org/en/mcu/PY32F030xx
- PY32F030 datasheet:
  https://download.py32.org/Datasheet/en/PY32F030_Datasheet_V2.5.pdf
- PY32F0xx DFP user manual:
  https://download.py32.org/PACK_MDK/PY32F0xx_DFP%20User%20Manual%20Rev.1.0.pdf

Published PY32F030 characteristics relevant here:

- ARM Cortex-M0+;
- up to 48 MHz;
- up to 64 KiB flash;
- up to 8 KiB SRAM.

The DFP manual includes a `P32F030xx 64kB Flash` programming algorithm with:

```text
Flash: 0x08000000 .. 0x0800FFFF
RAM for flash algorithm: 0x20000000 ...
Debugger: CMSIS-DAP over SWD
```

This is why a CMSIS-DAP probe is considered a realistic path for PAN7420.

### Open-source PY32 examples / packs

Useful repositories:

- OpenPuya firmware:
  https://github.com/OpenPuya/PY32F0xx_Firmware
- Example PY32 project / DFP usage:
  https://github.com/jaydcarlson/py32-template

The Puya project files contain flash-loader references for `PY32F030x8` / 64 KiB devices.

### What is not yet proven

It is **not yet proven on the actual PAN7420 in this car** that:

- SWD access is unlocked;
- original flash can be read;
- the stock chip accepts the PY32F030 flash algorithm unchanged.

These require connecting to the real target.

## Existing Jiabaile radio reverse engineering

The most important external discovery is the DIY-Multiprotocol-TX-Module project:

- repository:
  https://github.com/pascallanger/DIY-Multiprotocol-TX-Module
- protocol table:
  https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/Protocols_Details.md
- Jiabaile implementation:
  https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/Multiprotocol/JIABAILE_nrf24l01.ino
- XN297 emulation:
  https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/Multiprotocol/XN297_EMU.ino

The project contains:

```text
Protocol 102: JIABAILE
Subprotocol 0: STD
Subprotocol 1: GYRO
RF: NRF24L01
Emulation: XN297
```

### JIABAILE_GYRO details from source

The current source defines:

```text
payload size:       8 bytes
packet period:      8205 us
RF mode:            XN297-compatible, scrambled, 1 Mbit/s
```

The Gyro data packet code maps approximately:

| Byte | Meaning in existing implementation |
|---:|---|
| 0 | throttle |
| 1 | steering |
| 2 | speed mode |
| 3 | gyro |
| 4 | light / flash / steering trim flags |
| 5 | unknown / zero |
| 6 | unknown / zero |
| 7 | checksum |

The implementation also contains binding and frequency-hopping logic.

### Important compatibility caveat

This existing reverse engineering is **not yet proven to match the 1:36 PAN7420 car**.

Observed firsthand:

- a 1:43 non-gyro Jiabaile car and its remote do not cross-bind with the 1:36 gyro car / remote.

That does not prove a different physical RF layer. It may be explained by different bind/address/payload logic. Therefore the existing `JIABAILE_GYRO` implementation is a valuable starting point, not a confirmed drop-in protocol for the 1:36 model.

## XN297 emulation

The Multiprotocol project's `XN297_EMU.ino` shows that much of XN297 compatibility is implemented in software around an nRF24L01 radio:

- XN297 scrambling / whitening;
- bit reversal;
- CRC-16 handling;
- address representation;
- payload transformation;
- channel hopping.

That is the basis for the conservative PC-transmitter design:

```text
PC -> XIAO -> SPI -> nRF24L01+ -> stock car
```

## nRF24L01+

Nordic product specification:

- https://docs-be.nordicsemi.com/bundle/nRF24L01P_PS_v1.0/raw/resource/enus/nRF24L01P_PS_v1.0.pdf

Relevant facts:

- 2.4 GHz GFSK transceiver;
- SPI-controlled;
- 250 kbps, 1 Mbps and 2 Mbps modes on nRF24L01+;
- 1 Mbps is sufficient for the existing `JIABAILE_GYRO` implementation;
- 3.3 V-class supply; do not treat 5 V as a normal VCC input.

A compact "mini" nRF24L01+ module with PCB antenna is attractive for the transmitter because no PA/LNA is needed for this use case.

Typical module signals:

```text
VCC
GND
CE
CSN
SCK
MOSI
MISO
IRQ
```

For a simple transmitter, IRQ can initially be left unused.

## CMSIS-DAP probe options

### RP2040

Raspberry Pi Debug Probe firmware:

- https://github.com/raspberrypi/debugprobe
- documentation:
  https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html

RP2040 is attractive because the firmware is mature and directly implements CMSIS-DAP/SWD.

A XIAO RP2040 can be used as the MCU board for this role.

### ESP32-S3

A community CMSIS-DAP implementation exists for ESP32-S3:

- https://github.com/fl4p/daplink-s3

Seeed also uses ESP32-S3 in its XIAO Debug Mate product family, so using ESP32-S3 as a USB debug probe is not an exotic architecture.

For the first PAN experiment, RP2040 remains the simpler / more established path.

## XIAO Nordic as a one-board radio transmitter

Nordic nRF52/nRF54 devices have proprietary 2.4 GHz radio hardware and are conceptually capable of replacing the external nRF24L01 in a custom implementation.

However:

- the existing Jiabaile/XN297 source is written around nRF24L01 register semantics;
- a direct nRF52/nRF54 implementation would require porting and validating the RF framing;
- no 1:36 Jiabaile test has yet proven this path.

Therefore this is treated as a later optimization, not the lowest-risk first transmitter.

## Product / spare-part context

Jiabaile/JBL product family information previously used during research:

- manufacturer/product site:
  https://www.jiabaile.net/

The 1:36 platform is sold with separate receiver/main PCB, 1S battery, small servo and brushed motor parts. Exact spare-part revisions can change, so the actual board measurements in [bench-findings.md](bench-findings.md) take precedence over catalog assumptions.
