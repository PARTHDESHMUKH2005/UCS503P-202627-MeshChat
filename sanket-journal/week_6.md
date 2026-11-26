# Week 6 — Deployment, Operations & Field-Validation Plan

## Overview

In the sixth week, I looked at how the system is installed, run and checked on real hardware: the master service, the firmware toolchain, node provisioning, and what must be measured in the field before MeshChat can be called complete. This complements the software review by listing the physical and operational work that is still open.

## Tasks Achieved

### Master Deployment:

- `radcom-master.service` runs the daemon as a systemd service with automatic restart after five seconds; output goes to the journal through syslog.
- `make install` and `install.sh` build the program, install the binary and unit, and enable the service.
- Node IDs are given on the command line; the unit file starts the master with nodes 1 and 2.

### Firmware Toolchain:

- ESP-IDF 5.3, target `esp32c3`, `idf.py build` and `idf.py flash monitor`.
- USB serial console, so no separate UART adapter is needed.

### Re-Verification of the Software:

- Re-ran both host test suites on 26 November: node suite 43 of 43 passing; master suite 149 checks passing, 0 failed.

### Field-Validation Plan:

| Stage | What to check | Expected from calculation |
|---|---|---|
| 1. Bring-up | Version register reads `0x12` on both radios | 0x12 |
| 2. Point-to-point | One frame each way, bench distance | Received, CRC good |
| 3. Slot timing | Measure POLL to ACK time | About 62 ms idle |
| 4. Range | RSSI and SNR at increasing distance, after fixing the RSSI offset | Margin about 34 dB up, 22 dB down at 1 km |
| 5. Soak | Hours of polling with supply voltage logged | No brown-outs |
| 6. Multi-node | Two nodes, roaming phones, offline delivery | Messages delivered once |

## Problems Faced

### Deployment Steps Are Not Self-Contained:

`install.sh` does not install libgpiod or enable SPI, so a fresh Pi would fail to build or find the radio.

### Provisioning Described but Not Built:

The firmware README describes setting a node ID with `POST /config`, but no such handler exists. The node ID is a compile-time constant, so each node needs its own build. The README's WiFi name and password also differ from the source.

### Credentials in the Repository:

The WiFi passphrase is in source and in documentation. It must be changed and moved out of the repository before deployment.

### Configuration Drift:

The RF section of `hardware_config.h` (20 dBm) is not what the radio registers set; task-priority comments (6) differ from the code (15); the Pi header's DIO0 pin is wrong.

### No Measured Results Yet:

All range, power and timing figures are calculations. No field measurement exists in the repository.

## Solutions & Learning Outcomes

I prepared the staged validation plan above and a prioritised list of fixes before deployment: (1) honour `ack_id` on every reply, (2) escape the chat-list fields, (3) serve the embedded client, (4) add provisioning, (5) complete `install.sh` and correct the header, (6) fix the RSSI offset, (7) change and externalise credentials, (8) settle the frequency and radio hardware. The main lesson is that "it builds and the tests pass" is the start of deployment work, not the end: installation, configuration, measurement and security all need their own checklist.
