# Week 2 — Node Radio Wiring & SX1278 Bring-Up (ESP32-C3)

## Overview

In the second week, I connected the SX1278 to the ESP32-C3 and brought the radio to a working state: SPI communication, reset, mode switching, register configuration and the DIO0 interrupt. This is the hardware layer that the packet protocol sits on.

## Tasks Achieved

### Wiring:

| ESP32-C3 | Ra-02 | Signal |
|---|---|---|
| GPIO 6 | SCK | SPI clock |
| GPIO 7 | MOSI | Data to radio |
| GPIO 2 | MISO | Data from radio |
| GPIO 10 | NSS | Chip select |
| GPIO 3 | RESET | Radio reset |
| GPIO 1 | DIO0 | RxDone / TxDone interrupt |
| 3.3 V, GND | 3.3 V, GND | Power (3.3 V only, never 5 V) |

All pin numbers live in one file, `hardware_config.h`, so a different board is a one-file change.

### SPI and Reset:

- SPI2 host, mode 0, 8 MHz, DMA channel auto, polling transfers, 256-byte maximum transfer.
- Hardware reset: RESET low for 10 ms, high for 10 ms.
- Version check: read register `0x42` and refuse to start unless it is `0x12`.

### Register Configuration:

| Setting | Value |
|---|---|
| Frequency | `0x6C4000` (433 MHz) |
| PA configuration | `0x8C` (PA_BOOST, +14 dBm) |
| Modem config 1 / 2 / 3 | `0x72` / `0x74` / `0x04` (125 kHz, CR 4/5, SF7, CRC on, AGC auto) |
| Preamble | 8 symbols |
| Sync word | `0x12` |
| FIFO base TX / RX | `0x80` / `0x00` |

### Interrupt Handling:

- DIO0 is mapped to RxDone in receive mode and to TxDone while transmitting, by rewriting the DIO mapping register around each transmission.
- A rising-edge interrupt service routine does nothing except notify the radio task, which then reads and clears the IRQ flag register.
- The radio task owns the SX1278 exclusively; no other task touches SPI.

### Watchdog:

- The radio task waits for a POLL in chunks of at most two seconds and feeds the task watchdog each time; with no POLL for six seconds it marks the master absent and re-enters receive mode.

## Problems Faced

### LoRa Mode Would Not Stick:

Writing the LoRa bit directly from standby had no effect. The datasheet allows the LoRa/FSK bit to change only in sleep mode.

### Floating DIO0:

A floating interrupt pin produced edge storms with no IRQ flag set, which could starve other tasks and trip the watchdog.

### Transmit-Complete Handling:

TxDone arrives on the same pin as RxDone, so the firmware has to know which meaning is currently mapped, and must return to receive mode afterwards or it will miss the next POLL.

### Questionable RSSI Offset:

The logging code computes RSSI as `-157 + register`. That constant applies to the high-frequency port (above 525 MHz). At 433 MHz the SX1278 uses the low-frequency port, where the offset is `-164`, so logged RSSI is probably about 7 dB too optimistic. It only affects debug logging today, but it must be corrected before any field measurement is recorded.

## Solutions & Learning Outcomes

The fix for mode switching is a fixed sequence: sleep, set the LoRa bit while still asleep, then standby. The interrupt line gets a pull-down, empty IRQ events are ignored, and the task yields briefly so it can never spin. I learned that radio bring-up is mostly about state ownership and ordering: who is allowed to touch the chip, in which mode, and what the pin means at that moment.
