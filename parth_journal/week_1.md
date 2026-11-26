# Week 1 — Project Understanding, Scope Definition & Architecture Planning

## Overview

In the first week, I studied the MeshChat problem and converted the initial idea of an infrastructure-free messaging system into a concrete software and hardware architecture. The central requirement was to allow phones to exchange messages in an area without relying on the internet, cellular networks, or a wired backbone.

## Tasks Achieved

### Project Scope Analysis:

- Identified the three main layers of the system: the phone interface, ESP32 node firmware, and Raspberry Pi master software.
- Confirmed that phones would connect to nearby ESP32 nodes over WiFi.
- Defined the LoRa radio link between ESP32 nodes and the Raspberry Pi master.
- Clarified that the current design is a single-hop star network, not a multi-hop mesh: every node communicates directly with the master.

### Requirements and State Design:

- Defined the master routing table as `User ID → Node ID`.
- Defined the node session table as `WebSocket connection → User ID`.
- Identified the need for store-and-forward messaging when a recipient is offline.
- Identified SQLite as the planned persistent queue storage on the master.

### Initial Technology Study:

- Studied ESP-IDF, FreeRTOS, Raspberry Pi Linux services, SPI, GPIO, LoRa, WiFi AP mode, WebSockets, and SQLite.
- Evaluated the limitations of ESP32-C3 WiFi station capacity.
- Reviewed the need for a compact binary packet format because every radio byte affects airtime and capacity.

## Problems Faced

### Misleading Mesh Terminology:

The initial project idea suggested a full mesh, but multi-hop routing would introduce route discovery, duplicate suppression, loop prevention, and more difficult reliability problems.

### Balancing Scope with the Prototype Timeline:

The desired user scale was much larger than the practical capacity of a small ESP32 access point and a single LoRa channel.

### Hardware and Radio Selection:

The original design document described an SX1276/RFM95W radio in the 865–867 MHz band, while available development modules and the implementation direction used an SX1278/Ra-02 at 433 MHz.

## Solutions & Learning Outcomes

The first version was narrowed to a single-cell, master-driven star topology. This removed unnecessary multi-hop routing while preserving the core messaging objective. I learned that a successful embedded software project must define the actual hardware limits and communication topology before implementing application features.

