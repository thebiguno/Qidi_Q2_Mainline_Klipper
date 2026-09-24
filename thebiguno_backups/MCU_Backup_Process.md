# QIDI Q2 Stock Firmware Backup Notes

This document outlines what I did to verify / backup existing firmware before modifying anything.

Printer was running stock QIDI firmware **1.1.1**.

The goal was to establish working SWD access and make verified backups of both MCU firmware images before modifying anything for mainline Klipper/Katapult.

## Hardware identified

- Linux/AP board: **X-9-1 V1.5**
- Main motion-control board: **X-9-3 V1.3.0**
- Toolhead board: **A-10 V1.1.3**
- Programmer: **ST-LINK/V2-1 section of a NUCLEO-F411RE**
- Host: macOS with `stlink` 1.8.0 installed

The two **CN2 jumpers on the Nucleo were removed**, disconnecting the onboard STM32F411 from the ST-Link so CN4 could be used for an external target.

---

# 1. Mainboard: X-9-3 V1.3.0

The rear printer cover was removed to expose the X-9-3.

The 2x3 SWD header on the X-9-3 was identified as:

    6 GND      5 RST
    4 GND      3 SWCLK
    2 3V3      1 SWDIO

Pin 4 and pin 6 were verified with a multimeter as GND.

Pin 2 was measured at approximately 3.3 V relative to GND.

## Mainboard ST-Link wiring

NUCLEO CN4 pinout:

    CN4-1  VDD_TARGET
    CN4-2  SWCLK
    CN4-3  GND
    CN4-4  SWDIO
    CN4-5  NRST
    CN4-6  SWO

Connections:

    X-9-3 pin 1 SWDIO  -> CN4 pin 4 SWDIO
    X-9-3 pin 2 3V3    -> CN4 pin 1 VDD_TARGET
    X-9-3 pin 3 SWCLK  -> CN4 pin 2 SWCLK
    X-9-3 pin 4 GND    -> CN4 pin 3 GND
    X-9-3 pin 5 RST    -> CN4 pin 5 NRST

    CN4 pin 6 SWO      -> not connected

IMPORTANT: the mainboard remained powered by the Q2's own 24 V supply.

The Nucleo did NOT supply 3.3 V to the X-9-3. CN4 pin 1 is only the ST-Link target-voltage reference.

## Verify SWD communication

    st-info --probe

Result:

    Found 1 stlink programmers
    flash:    524288
    sram:     196608
    chipid:   0x413
    dev-type: STM32F4x5_F4x7

This confirmed working SWD communication with the Q2 main MCU.

## Mainboard firmware backup

A normal read initially failed:

    st-flash read q2-x9-3-v1.3-stock.bin 0x08000000 0x80000

with:

    Can not connect to target. Please use 'connect under reset'

Reading under reset succeeded:

    st-flash --connect-under-reset read q2-x9-3-v1.3-stock.bin 0x08000000 0x80000

Flash address:

    0x08000000

Length:

    0x80000 = 524288 bytes = 512 KiB

Verify size:

    ls -l q2-x9-3-v1.3-stock.bin

Expected:

    524288 bytes

Create SHA-256:

    shasum -a 256 q2-x9-3-v1.3-stock.bin

Result:

    473055c2ee205241f2e70da224db894a5d540f885e837c952f636e8a33eb9a86

Save checksum:

    shasum -a 256 q2-x9-3-v1.3-stock.bin > q2-x9-3-v1.3-stock.bin.sha

Make a second independent dump:

    st-flash --connect-under-reset read q2-x9-3-v1.3-stock.bin.verify 0x08000000 0x80000

Compare:

    cmp q2-x9-3-v1.3-stock.bin q2-x9-3-v1.3-stock.bin.verify

`cmp` produced no output, confirming that both dumps were byte-for-byte identical.

---

# 2. Toolhead: A-10 V1.1.3

The front and rear toolhead covers were removed and the three A-10 mounting screws were removed.

The small VIN/GND/TX/RX daughterboard connected to the drag chain was disconnected from the A-10. This electrically isolated the toolhead board from the printer.

Other local toolhead peripherals were left connected.

The A-10 has an unpopulated 2x3 SWD footprint numbered:

    5   3   1
    6   4   2

A temporary friction-fit 2x3 header was used instead of soldering.

Multimeter checks confirmed:

    pin 2 = 3.3 V
    pin 4 = GND
    pin 6 = GND

Successful SWD communication subsequently confirmed:

    pin 1 = SWDIO
    pin 3 = SWCLK

Pin 5 was not used. Its NRST function was not independently verified.

## Toolhead ST-Link wiring

Unlike the mainboard, the isolated A-10 needed to be powered from the Nucleo's 3.3 V rail.

Connections:

    A-10 pin 1 SWDIO -> CN4 pin 4 SWDIO
    A-10 pin 3 SWCLK -> CN4 pin 2 SWCLK
    A-10 pin 4 GND   -> CN4 pin 3 GND

A-10 pin 2 needs both power and target-voltage sensing:

    A-10 pin 2 3V3 -> Nucleo CN6 pin 4 (+3.3 V)
    A-10 pin 2 3V3 -> Nucleo CN4 pin 1 (VDD_TARGET)

Not connected:

    A-10 pin 5
    A-10 pin 6
    CN4 pin 5 NRST
    CN4 pin 6 SWO

The Q2 itself remained powered off/unplugged and the drag-chain interface remained disconnected.

## Verify toolhead SWD communication

    st-info --probe

Result:

    Found 1 stlink programmers
    flash:    131072
    sram:     65536
    chipid:   0x414
    dev-type: F1xx_HD

This confirmed successful SWD access to the toolhead MCU.

## Toolhead firmware backup

Read the complete 128 KiB flash:

    st-flash read q2-a10-v1.1.3-stock.bin 0x08000000 0x20000

Flash address:

    0x08000000

Length:

    0x20000 = 131072 bytes = 128 KiB

Verify size:

    ls -l q2-a10-v1.1.3-stock.bin

Expected:

    131072 bytes

Create SHA-256:

    shasum -a 256 q2-a10-v1.1.3-stock.bin

Result:

    a89f3d7746481cb71799181804e5fc49c187b90303e88a35a2cf6329264202f8

Save checksum:

    shasum -a 256 q2-a10-v1.1.3-stock.bin > q2-a10-v1.1.3-stock.bin.sha

Make a second independent dump:

    st-flash read q2-a10-v1.1.3-stock.bin.verify 0x08000000 0x20000

Compare:

    cmp q2-a10-v1.1.3-stock.bin q2-a10-v1.1.3-stock.bin.verify

`cmp` produced no output, confirming that both toolhead dumps were byte-for-byte identical.

---

# Backup status

Mainboard:

    q2-x9-3-v1.3-stock.bin
    q2-x9-3-v1.3-stock.bin.verify
    q2-x9-3-v1.3-stock.bin.sha

SHA-256:

    473055c2ee205241f2e70da224db894a5d540f885e837c952f636e8a33eb9a86

Toolhead:

    q2-a10-v1.1.3-stock.bin
    q2-a10-v1.1.3-stock.bin.verify
    q2-a10-v1.1.3-stock.bin.sha

SHA-256:

    a89f3d7746481cb71799181804e5fc49c187b90303e88a35a2cf6329264202f8

Both stock firmware images were read twice and the duplicate files compared identical before any erase or flash operation was performed.

These files back up the normal MCU flash regions. They do not necessarily contain MCU option bytes or other non-flash configuration state.

NO MCU ERASE OR WRITE HAS BEEN PERFORMED YET.
