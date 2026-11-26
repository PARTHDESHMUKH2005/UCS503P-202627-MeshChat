# Week 1 — Hardware Selection, Power Budget & RF Planning

## Overview

In the first week, while the architecture was being fixed, I worked on the physical side of MeshChat: which radio hardware the system actually runs on, which RF settings it uses, how much airtime each packet costs, how strong the link will be, and how the master's offline queue should be stored. The goal was to turn the design document's goals into numbers that can be checked on a bench.

## Tasks Achieved

### Hardware Selection:

- Node: ESP32-C3 mini development board with an SX1278 (Ra-02, AI-Thinker) LoRa module.
- Master: Raspberry Pi with a second Ra-02 module on SPI0.
- Antennas: 433 MHz antennas on both ends; the design document recommends investing in the master's antenna because it carries the budget for every node.
- Power: the design document recommends a separate 3.3 V regulator for the master's radio because transmit current spikes of roughly 120 mA can exceed what the Pi's GPIO 3.3 V pin is sized for.

### RF Parameter Set:

- 433 MHz, spreading factor 7, 125 kHz bandwidth, coding rate 4/5, 8-symbol preamble, CRC enabled, explicit header.
- Private sync word `0x12` instead of the LoRaWAN public word `0x34`, so the network ignores public LoRa traffic.
- Frequency register word: `Frf = f / (32 MHz / 2^19) = f(MHz) x 16384`, so 433 MHz is `0x6C4000`.

### Airtime Analysis (computed from the Semtech formula):

| Frame | Bytes | Airtime |
|---|---|---|
| ACK or empty POLL | 4 | 31.0 ms |
| PRESENCE | 23 | 61.7 ms |
| DATA, 20-character message | 58 | 112.9 ms |
| DATA, maximum payload | 138 | 225.5 ms |

- Polling slot cost: about 62 ms idle, 175 ms for an uplink message, 144 ms for a downlink delivery.
- An idle cycle over N nodes takes roughly 62 x N ms, so about 0.3 s for 5 nodes and 1.2 s for 20.

### Link Budget (1 km, about 103 dB loss including obstruction, antenna gains ignored):

| Case | TX power | Margin over SF7 sensitivity (-123 dBm) |
|---|---|---|
| Planned | 20 dBm | 40 dB |
| Uplink as configured (node) | 14 dBm | 34 dB |
| Downlink as configured (master) | 2 dBm | 22 dB |

### Offline Queue Schema:

- Wrote the first version of the SQLite queue table in `db.txt` (committed 3 September): destination and source user ID, message ID, TTL, payload blob and length, expiry time and creation time.
- The final `queue.c` adds an index on the destination user ID and WAL journaling.

## Problems Faced

### Radio Chip Cannot Reach the Designed Band:

The design document specifies an SX1276 at 865–867 MHz and states that the SX1278 cannot reach that band. The modules in use are SX1278 (about 420–525 MHz), so the prototype runs at 433 MHz. This is a hardware limit, not a setting. An extra trap: every chip in the SX1276/77/78/79 family reports version `0x12`, so the firmware's version check cannot reveal which band variant is fitted.

### Unresolved Regulatory Question:

The design document says 433 MHz belongs to a different exemption class, while the hardware configuration comments call it permitted. I did not resolve this; it needs a definite answer before any real deployment.

### Power Asymmetry:

The firmware configuration later sets different transmit power on each side (14 dBm node, 2 dBm master), which leaves the downlink with 18 dB less margin than the plan assumed.

## Solutions & Learning Outcomes

I documented the deviation and its migration path instead of hiding it: moving to an SX1276 at 865.0625 MHz means writing `0xD84400` to the frequency registers and clearing the low-frequency-mode bit. I learned to turn every design claim into a number early — airtime, margin, current — because those numbers later told us which problems were software and which were physics.
