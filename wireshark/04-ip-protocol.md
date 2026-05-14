# IP Protocol

IP (Internet Protocol) operates at the network layer. Its responsibility is delivering datagrams from source to destination across networks, possibly involving many intermediate routers. Every device on the internet has an IP address, and IP handles both addressing and routing.

## The IPv4 Header

| Field | Size | Purpose |
| --- | --- | --- |
| Version | 4 bits | IPv4 = 4, IPv6 = 6 |
| Header Length | 4 bits | Size of the header (typically 20 bytes) |
| Total Length | 16 bits | Entire datagram size (header + data) |
| Identification | 16 bits | Unique ID — fragments of the same datagram share this |
| Flags | 3 bits | Controls fragmentation behaviour |
| Fragment Offset | 13 bits | Position of this fragment within the original datagram |
| TTL | 8 bits | Decremented at each router; 0 causes the packet to be dropped |
| Protocol | 8 bits | What is encapsulated: TCP = 6, UDP = 17, ICMP = 1 |
| Header Checksum | 16 bits | Error detection for the header only |
| Source IP | 32 bits | Sender's address |
| Destination IP | 32 bits | Receiver's address |

## Worked Example — Traceroute Analysis

The following observations come from a real traceroute capture. Traceroute discovers the path to a destination by sending packets with progressively increasing TTL values, causing each router along the path to send back an ICMP "time exceeded" message before dropping the packet.

### Identifying the source

The source IP of the first probe packet identifies the originating computer. In this capture, the source was `192.168.1.118` — a private address indicating the device is behind a home router.

### Identifying the encapsulated protocol

The Protocol field in the IP header indicates what is inside. In this capture the value was `ICMP (0x01)` — protocol number 1, used by the traceroute responses.

Three protocol numbers are worth memorising:

- ICMP = 1
- TCP = 6
- UDP = 17

### Calculating the IP payload size

- Header Length: 20 bytes (the standard minimum, indicating no IP options)
- Total Length: 40 bytes
- Payload size = Total Length − Header Length = 40 − 20 = 20 bytes

This subtraction is how IP payload size is always derived.

### Determining whether a datagram has been fragmented

Three IP header values determine fragmentation status:

- **"Don't Fragment" flag** — if set, fragmentation is forbidden
- **"More Fragments" flag** — if set, more fragments follow this one
- **Fragment Offset** — if zero, this is either an unfragmented datagram or the first fragment of one

In this capture:

- **UDP traceroute probes** (40 bytes total): not fragmented. More Fragments = 0, Offset = 0, and total length is well under the 1500-byte MTU.
- **2000-byte ICMP pings**: fragmented. Each was split into two pieces.

### Fields that change between successive datagrams

These fields vary from one datagram to the next:

- **Identification** — each datagram gets a unique ID
- **TTL** — traceroute deliberately increments this each round
- **Header Checksum** — must be recalculated because other fields changed
- **Source and Destination Port** (in the UDP traceroute probes) — incremented per probe to allow matching responses back to specific probes

### Fields that remain constant

Across all probes to the same destination:

- **Version** (always 4)
- **Header Length** (always 20 bytes when no options are used)
- **Source IP** (always the originating computer)
- **Destination IP** (always the target)
- **Protocol** (UDP for traceroute probes, ICMP for the responses)

Source IP, Destination IP, Version, and Protocol *must* stay constant — they define what the packet is and where it is going.

### Pattern in the Identification field

The Identification field increments with each new datagram. On older systems this was a strict +1 counter. On modern operating systems (including macOS and many Linux variants), Identification values are randomised or chosen non-sequentially to prevent traffic analysis attacks. Captures from modern systems often show seemingly random Identification values rather than the textbook +1 progression.

### Identification and TTL in router responses

Looking at three consecutive ICMP "TTL exceeded" replies from the first-hop router (`192.168.1.1`):

| Packet | Identification | TTL | Total Length |
| --- | --- | --- | --- |
| 73 | 16887 | 64 | 68 |
| 75 | 16893 | 64 | 68 |
| 77 | 16900 | 64 | 68 |

- Identification changes between replies (new datagram = new ID), but the increments are not strictly +1 — likely because the router is also generating IDs for unrelated traffic
- TTL stays constant at 64 because the router is always one hop away
- The starting TTL of 64 suggests Linux-based router firmware

## IP Fragmentation

### When fragmentation happens

When a datagram is larger than the link's MTU (Maximum Transmission Unit), the router fragments it before forwarding. Ethernet's MTU is 1500 bytes. A 4000-byte datagram cannot fit in a single Ethernet frame, so it must be split.

### The three fields that control fragmentation

| Field | Role |
| --- | --- |
| Identification | All fragments of the same datagram share the same value |
| More Fragments flag | 1 means more fragments follow; 0 means this is the last |
| Fragment Offset | Position of this fragment's data within the original datagram, measured in units of 8 bytes |

### Walking through a real fragmentation example

A 2000-byte ICMP echo request was sent. Because this exceeds the 1500-byte Ethernet MTU, it was fragmented into two pieces.

**First fragment (packet 646):**

- More Fragments flag = 1 (more pieces coming)
- Fragment Offset = 0 (this is the first piece, data starts at byte 0)
- Total Length = 1500 (maximum for Ethernet)
- Identification = 22649
- Data length = 1500 − 20 = 1480 bytes

**Second fragment (packet 647):**

- More Fragments flag = 0 (no more pieces — this is the last)
- Fragment Offset = 185 (Wireshark's displayed value; multiplied by 8 gives 1480 bytes, meaning this fragment's data begins at byte 1480 of the original datagram)
- Total Length = 548
- Identification = 22649 (matches fragment 1, proving they belong to the same datagram)
- Data length = 548 − 20 = 528 bytes

Sanity check: 1480 + 528 = 2008 bytes of data total, which combined with the original IP header (20 bytes) and the ICMP header (8 bytes inside the data section) matches the 2000-byte original.

### Fields that change between fragments

| Field | Fragment 1 | Fragment 2 | Changed? |
| --- | --- | --- | --- |
| Total Length | 1500 | 548 | Yes |
| More Fragments flag | 1 | 0 | Yes |
| Fragment Offset | 0 | 185 | Yes |
| Header Checksum | (different) | (different) | Yes (recalculated) |
| Identification | 22649 | 22649 | No |
| Source IP | 192.168.1.118 | 192.168.1.118 | No |
| Destination IP | 142.251.150.119 | 142.251.150.119 | No |
| Protocol | ICMP (1) | ICMP (1) | No |
| TTL | 64 | 64 | No |
| Version | 4 | 4 | No |

Four fields change between fragments: Total Length, More Fragments flag, Fragment Offset, and Header Checksum. Everything else remains constant.

### Fragmentation worked example

Splitting a 4000-byte datagram (20-byte IP header + 3980 bytes of data) across an Ethernet network with MTU 1500 (max data per fragment = 1500 − 20 = 1480 bytes):

| Fragment | Total Length | Data | More Fragments | Offset (bytes) | Offset (÷ 8) |
| --- | --- | --- | --- | --- | --- |
| 1 | 1500 | 1480 | 1 | 0 | 0 |
| 2 | 1500 | 1480 | 1 | 1480 | 185 |
| 3 | 1040 | 1020 | 0 | 2960 | 370 |

Total data: 1480 + 1480 + 1020 = 3980 bytes — matches the original.

## TTL (Time to Live)

TTL prevents packets from circulating forever in the event of a routing loop. Each router that forwards a packet decrements TTL by 1. When TTL reaches 0, the router discards the packet and sends an ICMP Type 11 (TTL Exceeded) message back to the source.

### Operating system fingerprinting from TTL

Different operating systems use different default TTL values, which means observed TTL values can reveal the OS of a sender:

| Initial TTL | Likely operating system |
| --- | --- |
| 128 | Windows |
| 64 | Linux or macOS |
| 255 | Routers and network equipment |

A packet observed with TTL = 52 most likely started at 64 (Linux/macOS) and traversed 12 routers (64 − 52 = 12 hops). This technique is useful for both network troubleshooting and reconnaissance.

## Key Takeaways

- The IPv4 header is 20 bytes minimum
- Protocol field values to remember: ICMP = 1, TCP = 6, UDP = 17
- Fragmentation is identified by shared Identification, the More Fragments flag, and Fragment Offset (measured in units of 8 bytes)
- Maximum data per fragment on Ethernet = 1480 bytes (1500 MTU − 20-byte IP header)
- TTL is decremented at each router; reaching 0 causes the packet to be dropped and triggers an ICMP Type 11 reply
- IP payload size = Total Length − Header Length
- Four fields change between fragments: Total Length, Flags, Fragment Offset, Header Checksum
- Modern operating systems may randomise IP Identification values rather than incrementing sequentially
