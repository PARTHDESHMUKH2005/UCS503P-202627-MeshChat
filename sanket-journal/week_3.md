# Week 3 — Raspberry Pi Radio Interface (spidev + libgpiod)

## Overview

In the third week, I worked on the master's radio interface. The ESP32 talks to the SX1278 through a microcontroller SPI driver; the Raspberry Pi talks to the same chip through Linux userspace interfaces. I kept the register-level behaviour identical on both sides and dealt with everything that is different about running on Linux.

## Tasks Achieved

### Wiring:

| Ra-02 | BCM GPIO | Header pin | Note |
|---|---|---|---|
| SCK | GPIO 11 | 23 | SPI0 clock |
| MOSI | GPIO 10 | 19 | SPI0 |
| MISO | GPIO 9 | 21 | SPI0 |
| NSS | GPIO 22 | 15 | Driven manually |
| RESET | GPIO 25 | 22 | Pulsed low 10 ms at start |
| DIO0 | GPIO 23 | 16 | Interrupt line |
| 3.3 V, GND | — | 1 or 17, 6 | Separate regulator recommended |

### Linux Interfaces:

- SPI through `/dev/spidev0.0`, mode 0, 1 MHz, using `ioctl` transfers.
- Chip select, reset and DIO0 through libgpiod, requested by line name.
- DIO0 requested for rising-edge events; its event file descriptor is made non-blocking and waited on with `poll()`.
- Stale edge events are drained before each transmission.

### Transmit and Receive Path:

- Transmit: standby, write FIFO, set payload length, map DIO0 to TxDone, clear flags, enter transmit, wait for the DIO0 edge for up to three seconds, then return to receive mode.
- Receive: read IRQ flags, discard CRC errors, read the FIFO from the current RX address, log RSSI and SNR.
- Transmit power set to the minimum (PA configuration `0x80`, +2 dBm).

### Robustness:

- Radio initialisation retried three times at start-up; the LoRa-mode sequence is retried up to ten times with read-back.
- If a send finds the chip no longer in LoRa mode, it logs a brown-out and re-initialises the radio before sending.
- On a missed TxDone the code logs the DIO0 level, IRQ flags and operating mode before recovering.

## Problems Faced

### Header and Code Disagree on the Pin:

`radio.h` defines DIO0 as GPIO 24, but the code requests GPIO 23, and chip select (GPIO 22) is not in the header at all. Wiring from the header would give an interrupt that never fires.

### Two Chip Selects:

Opening `/dev/spidev0.0` makes the kernel toggle the hardware CE0 pin on every transfer, while the code also toggles GPIO 22 manually. CE0 is simply left unconnected, but the redundancy is easy to misunderstand.

### Power Dips:

After long runs the chip could fall out of LoRa mode, the signature of a brown-out from the Pi's small 3.3 V rail under transmit current.

### Deployment Prerequisites Not Automated:

The Makefile links `-lgpiod`, but `install.sh` installs only `libsqlite3-dev`, and nothing enables the SPI interface. The code also uses the libgpiod 1.x API, which differs from 2.x.

## Solutions & Learning Outcomes

The working wiring is the one the code uses (GPIO 23 for DIO0, GPIO 22 for chip select). Brown-outs are handled in two ways: minimum transmit power, and detect-and-reinitialise before every send; a dedicated regulator remains the proper fix. The lesson is that a driver that works on a microcontroller does not simply port to Linux: pin naming, edge events, chip-select ownership and library versions all need their own attention.
