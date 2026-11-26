# Week 2 — Packet Protocol, Addressing & Serialization

## Overview

In the second week, I designed and implemented the common packet layer used by both the Raspberry Pi master and the ESP32 firmware. The goal was to make both sides interpret radio messages identically and to keep packets small enough for the LoRa FIFO.

## Tasks Achieved

### Packet Types:

- Implemented `DATA` packets for user messages.
- Implemented `ACK` packets for delivery confirmation.
- Implemented `POLL` packets for master-driven channel access.
- Implemented `PRESENCE` packets for user attach and detach events.

### Packet Fields:

- Added message IDs for identifying transmissions and acknowledgements.
- Added source and destination user IDs.
- Added node IDs for radio addressing.
- Added TTL information for message expiry.
- Added a bounded binary payload with a maximum size of 100 bytes.

### Serialization and Validation:

- Implemented binary serialization and deserialization in C.
- Used compact wire formats for ACK and empty POLL packets.
- Added validation for packet types, payload lengths, truncated buffers, and invalid user IDs.
- Used numeric E.164-style phone numbers as user identifiers.
- Ensured the maximum packet remains within the SX1278 FIFO limit.

## Problems Faced

### Different Packet Shapes:

Presence packets do not need a destination user ID, while data packets require both source and destination IDs. Treating every packet as an identical full-size structure would waste airtime and make the wire format less clear.

### Malformed and Oversized Input:

Radio data can be incomplete, corrupted, or incorrectly sized. Without strict validation, a malformed payload could cause incorrect routing or memory-safety problems.

### Protocol Consistency:

The master and node firmware must use exactly the same field order, byte lengths, and packet type values.

## Solutions & Learning Outcomes

I created a shared packet definition and matching serialization logic for the master and node implementations. I added explicit length checks and user-ID validation before accepting packets. This week demonstrated the importance of defining a stable binary protocol before building higher-level routing and delivery logic.

