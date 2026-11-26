# Week 4 — ESP32 Node Firmware, WiFi Access Point & Web Interface

## Overview

In the fourth week, I implemented the ESP32 node firmware. The node acts as the bridge between nearby phones and the LoRa master. It provides local WiFi access, serves the messaging interface, tracks connected users, and forwards messages through the radio.

## Tasks Achieved

### ESP-IDF Application Structure:

- Created an ESP-IDF project for the ESP32-C3.
- Used FreeRTOS tasks and queues to separate radio, WiFi, and routing responsibilities.
- Added non-volatile storage for the node ID.
- Added startup initialization for NVS, networking, radio, and application tasks.

### LoRa Firmware:

- Implemented SPI communication with the SX1278/Ra-02 module.
- Added register configuration, reset handling, transmit, receive, and DIO0 interrupt handling.
- Added transmit and receive queues between the radio task and routing task.
- Added watchdog and timeout handling.

### WiFi and WebSocket Layer:

- Configured the ESP32 as a WiFi access point.
- Added an HTTP server and WebSocket endpoint.
- Added a captive-portal-style DNS redirect for easier phone access.
- Embedded the browser client into the firmware image.
- Added connect, disconnect, attach, and message events.

### Node Router:

- Added a session table for connected phones.
- Implemented user attach and detach handling.
- Implemented local delivery when sender and receiver are on the same node.
- Implemented LoRa forwarding for remote users.
- Implemented delivery of master packets to the correct WebSocket client.

## Problems Faced

### Concurrency:

WiFi callbacks, WebSocket events, and radio events can happen at the same time. Directly modifying shared session data from multiple tasks could cause inconsistent state.

### Limited Node Capacity:

The ESP32 access point has a small practical station limit, which conflicts with the original long-term goal of hundreds of users per node.

### User Identity and Reconnection:

The node needs an explicit attach message because a local IP address alone is not a reliable long-term user identity.

## Solutions & Learning Outcomes

I separated responsibilities into WiFi, router, and radio tasks connected through FreeRTOS queues. Session state is centrally managed by the router, and attach/detach events update the master. I learned how embedded networking requires careful ownership of shared state and how the hardware access-point limit directly affects software architecture.

