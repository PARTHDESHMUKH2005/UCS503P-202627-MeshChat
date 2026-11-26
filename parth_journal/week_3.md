# Week 3 — Raspberry Pi Master Software & Message Routing

## Overview

In the third week, I implemented the Raspberry Pi master software. The master is responsible for coordinating nodes, maintaining user locations, transferring messages, and storing messages for users who are temporarily offline.

## Tasks Achieved

### Radio and Service Integration:

- Added Linux SPI and GPIO access for the SX1278 radio.
- Implemented radio initialization, transmit, receive, and timeout handling.
- Added signal handling for controlled shutdown.
- Added a Makefile, installation script, and systemd service definition.
- Added syslog-based operational logging.

### Routing Table:

- Implemented an in-memory open-addressing hash table.
- Added user attach, detach, lookup, and node-eviction operations.
- Added support for users moving between nodes.
- Added removal of all users when a node becomes unreachable.

### Polling and Channel Access:

- Implemented round-robin node polling.
- Added a maximum response timeout for each node.
- Added absent-node detection after repeated timeouts.
- Added keepalive handling for nodes that temporarily disappear.
- Added piggybacking of queued data onto POLL packets.

### SQLite Message Queue:

- Implemented persistent message storage.
- Added FIFO retrieval.
- Added destination-node-aware queue lookup.
- Added TTL expiry handling.
- Added deletion only after delivery acknowledgement.

## Problems Faced

### Offline Delivery:

A message cannot be sent directly when the destination user is not currently attached to a node. The system needed to retain the message without losing it or sending it to the wrong node later.

### Node Failure and Stale Routes:

The master can retain an old user-to-node mapping after a node loses power or radio contact.

### Reliable Delivery Without Collisions:

Allowing all nodes to transmit whenever they wanted would create LoRa collisions and make delivery unpredictable.

## Solutions & Learning Outcomes

The master now uses a persistent SQLite queue and only removes a queued message after the destination node acknowledges it. Round-robin polling gives each node a controlled transmission slot, while timeout tracking evicts stale routes. I learned how embedded network coordination requires both a routing model and a failure-recovery strategy.

