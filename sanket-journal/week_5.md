# Week 5 — Phone Interface, Captive Portal & Embedded Web Client

## Overview

In the fifth week, I worked on everything the phone sees: how a phone finds the node, how it talks to it, and the browser application it runs. The embedded web client is only mentioned in one line elsewhere, so this week documents it in full along with the build step that packs it into the firmware.

## Tasks Achieved

### Captive Portal:

- DNS: a small server answers every query with the node address (192.168.4.1, 60 s lifetime).
- HTTP: any request for another host or a non-root path gets a 302 redirect to the root page.
- WebSocket endpoint `/ws`, plus a limit on open sockets and least-recently-used purging so stale connections cannot starve new phones.

### WebSocket Protocol (JSON):

| Direction | Frames |
|---|---|
| Phone to node | `attach` (phone number), `msg` (destination, text) |
| Node to phone | `attached`, `msg`, `queued`, `sent`, `status` (master online or not), `error` |

### Offline-First Web Client (1,271 lines, no frameworks):

- Identity stored locally; on first run an onboarding dialog asks for a phone number and optional name.
- IndexedDB database with three stores (contacts, chats, messages) and indexes for chat, time, status and a compound chat-and-time index added in schema version 2.
- Messages are saved before sending and marked pending; pending messages are flushed after reconnect.
- Reconnect uses exponential back-off and pauses while the tab is hidden.
- Three connection states are shown: disconnected, node only and mesh live.
- Chat list with unread counters, newest-first ordering, paging of 50 messages with scroll-up loading, and a quota that keeps at most 1,000 messages per chat.
- Material-style theme with the two Roboto fonts embedded, so no internet is needed.

### Build Pipeline:

- `build_ui.py` inlines the stylesheets and scripts into one HTML file, converts the fonts to base64 and gzips the result to 55,642 bytes.
- CMake runs the script before compiling and embeds the file in the firmware image.

### Access Point Tuning:

- One shared SSID on every node so phones roam; WPA2, channel 1, at most 10 stations.
- WiFi transmit power limited to 8.5 dBm because the board's regulator browned out at full power.
- Receive buffers, HTTP socket count, watchdog (10 s) and USB console set in `sdkconfig.defaults`.
- Sockets that connect but never attach are closed after 30 s.

## Problems Faced

### The Full Client Is Built but Not Served:

The firmware embeds the gzip file, but the HTTP handler still returns the small inline page.

### Protocol Mismatch:

The client sends a message ID and waits for `ack` frames; the firmware ignores the ID and answers `queued` and `sent`. As a result outgoing messages never reach the "delivered" double tick.

### Unescaped HTML in the Chat List:

Message bubbles are escaped, but the chat list inserts the contact name and the last-message preview with `innerHTML`. A received message containing markup would be interpreted by the recipient's browser, a stored cross-site-scripting risk. All nodes share the address 192.168.4.1, so they also share one browser origin.

### Station Limit:

The ESP32-C3 access point accepts at most 10 phones, far below the original goal of hundreds per node.

## Solutions & Learning Outcomes

The inline page and the firmware both build JSON safely (cJSON, `textContent`), and the embedded client keeps working offline. The open items are clear: serve the embedded file, align the acknowledgement frames, and escape the chat-list fields (use `textContent`). I learned that the user interface is part of the system's attack surface and its protocol, not decoration, and that a build pipeline step (CMake ordering, embedded files) can take as much care as the code.
