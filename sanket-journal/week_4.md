# Week 4 — Over-the-Air Integration of Node and Master

## Overview

In the fourth week, I worked on the seam between the two halves of the system: the ESP32 node firmware and the Raspberry Pi master, which only meet over the radio. Each side was written and tested alone; this week was about making sure they agree on bytes, timing and meaning.

## Tasks Achieved

### Byte-Level Agreement:

- Confirmed that `packet.c` and `packet.h` are byte-for-byte identical in the master and the node firmware.
- Compared the radio register setup on both sides. Frequency, modem configuration, preamble, sync word, FIFO bases and DIO mapping match exactly; the only intended differences are transmit power (node +14 dBm, master +2 dBm) and the SPI access method.

### Timing Agreement:

| Item | Value | Check |
|---|---|---|
| Master slot timeout | 500 ms | Longest reply frame is 225.5 ms on air |
| Node "master absent" timeout | 6 s | Longer than a full idle cycle of 20 nodes (about 1.2 s) |
| Master transmit wait | 3 s | Longest frame is 225.5 ms |

### Acknowledgement Flow (as implemented):

- The master sends POLL; the polled node replies with DATA, PRESENCE or ACK.
- The node copies the POLL's message ID into the reply's `ack_id`, confirming a delivery that rode on the POLL.
- The master acknowledges every DATA and PRESENCE; the node removes a packet from its transmit queue only when that acknowledgement echoes the packet's ID.
- The master deletes a queued message only when an ACK echoes the POLL ID that carried it.

### Diagnostics Built Into the Link:

- The master's transmit-complete timeout logs the DIO0 level, IRQ flags and operating mode; both sides log RSSI and SNR for every received frame at debug level.

## Problems Faced

### POLL Addressing Mismatch:

The master first stamped its own ID in a POLL's `node_id`, but a node ignores any POLL whose `node_id` is not its own. In a POLL that field means destination. A comment recording this discovery is still in `poller.c`, along with now-redundant assignments.

### Queue Row ID Does Not Fit a Packet Field:

The database row ID is a 64-bit key; the packet's fields are 8 bits. The poller keeps a separate pending record (POLL ID and row ID) and deletes only on a matching ACK.

### Stale Route Echo:

If the routing table points a destination at the sender's own node, the message loops back and can be shown to the sender or lost after a false ACK. The master treats this as a stale route, detaches it and stores the message offline.

### Acknowledgement Carried in the Wrong Packet (found by code review, not yet reproduced on hardware):

When a node has its own uplink DATA or PRESENCE waiting at the moment it receives a POLL with a delivery, it sends that packet with the POLL's ID in `ack_id`. The master's DATA and PRESENCE handlers do not look at `ack_id`, so the delivery is never confirmed, the row stays in the queue and the message is delivered again on the next poll.

### At-Least-Once Delivery:

A redelivery is sent under a new POLL ID. The node's duplicate check compares message IDs, so it cannot recognise a redelivery after a lost ACK. The system therefore guarantees delivery at least once, not exactly once.

## Solutions & Learning Outcomes

The first three problems are fixed in the code. The last two are recorded as open items with a clear fix: make the master honour `ack_id` on every reply type, and give deliveries a stable end-to-end ID for duplicate suppression. The main lesson is that interface bugs live in the meaning of a field, not its size: a header that compiles and passes unit tests on both sides can still be wrong over the air.
