# Jiabaile 1:36 / PAN7420 reverse engineering

Reverse engineering of the Jiabaile/JBL 1:36 gyro car (3611/3611A family) with a PAN7420S7FA MCU.

## Goal

- control the car from a PC / ROS 2;
- keep the original power, charger, H-bridge, servo and lighting hardware;
- preferably reflash the original PAN7420 instead of replacing the MCU;
- raise motor PWM above the audible range;
- understand enough of the radio protocol to build a small PC transmitter.

## Current status

The car PCB is already mapped well enough to control the important hardware:

- motor H-bridge inputs are known;
- servo signal is known;
- all three light-control GPIOs are known;
- the hardware power latch is understood and its PAN `HOLD` GPIO is known;
- battery voltage measurement is known;
- the probable IMU bus is exposed on four test pads and traced to PAN pins;
- PAN power rails and the switched-power path are understood.

The preferred next direction is **reflashing PAN7420 through SWD**. This has not yet been tested on the actual car.

For PC radio control, the lowest-risk first implementation is **XIAO + real nRF24L01+** using the existing open-source Jiabaile/XN297 work. Compatibility of the existing `JIABAILE_GYRO` implementation with the 1:36 car is **not yet proven**.

## Documentation

- [Bench findings](docs/bench-findings.md) — measurements, continuity tests and board observations made on the actual 1:36 PCB.
- [External research](docs/external-research.md) — datasheets, SDKs, existing Jiabaile reverse engineering and programming resources found online.
- [Design notes and open questions](docs/design-notes.md) — implementation options, hypotheses, risks and ideas that are not yet verified facts.

## Next experiments

1. Turn a XIAO RP2040 (or another suitable board) into a CMSIS-DAP probe and connect to PAN7420 over SWD **without erasing anything**.
2. Try to read CPU/memory and dump the original flash.
3. Build a minimal XIAO + nRF24L01+ transmitter and test whether the existing `JIABAILE_GYRO` protocol can bind to the stock 1:36 firmware.
4. If it cannot, compare the 1:36 RF protocol against the existing Jiabaile implementation instead of assuming compatibility.
5. Identify / capture the stock IMU bus only if needed for the new firmware.

## Important status convention

- **Verified on board** = measured or traced on the actual PCB.
- **External fact** = supported by a linked datasheet / SDK / source repository.
- **Hypothesis** = plausible interpretation or proposed design, not yet proven on this hardware.
