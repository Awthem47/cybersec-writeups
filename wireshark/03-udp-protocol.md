# UDP Protocol

UDP (User Datagram Protocol) is the minimal transport-layer protocol. Where TCP provides handshakes, sequence numbers, acknowledgments, and flow control, UDP provides almost nothing. It takes data, attaches a tiny header, and sends it. No setup, no guarantees, no recovery.

UDP is connectionless: there is no equivalent of TCP's three-way handshake. A sender may transmit at any time, and a receiver either receives the datagram or does not.

## The UDP Header

The UDP header contains only four fields:

| Field | Size | Purpose |
| --- | --- | --- |
| Source Port | 2 bytes (16 bits) | Identifies the sending process |
| Destination Port | 2 bytes (16 bits) | Identifies the receiving process |
| Length | 2 bytes (16 bits) | Total size of header plus data |
| Checksum | 2 bytes (16 bits) | Error detection |

Total header size: 8 bytes. By comparison, the minimum TCP header is 20 bytes — UDP is a fraction of the overhead.

## Reading a UDP Packet in Wireshark

Selecting any UDP packet and expanding the UDP section in the detail pane reveals all four header fields.

### Header field count and size

Four fields, each two bytes long, totalling 8 bytes of header overhead per datagram.

### What the Length field measures

The Length field gives the total size of the entire UDP datagram — header plus payload. If the Length field reads 57, the datagram contains 8 bytes of header and 49 bytes of data.

### Maximum payload size

Because the Length field is 16 bits, the maximum value it can represent is 2¹⁶ − 1 = 65,535 bytes. Subtracting the 8-byte header gives a maximum payload of 65,527 bytes. In practice, UDP datagrams are usually much smaller to avoid IP-layer fragmentation.

### Maximum port number

The port fields are also 16 bits, so the maximum port number is 65,535.

### UDP's protocol number in the IP header

The IP header's Protocol field identifies what is encapsulated. UDP's value is:

- Decimal: 17
- Hexadecimal: 0x11

For reference, TCP is 6 and ICMP is 1.

### Port number relationship between request and reply

When examining a request and its matching reply, the source and destination ports are swapped:

| Direction | Source Port | Destination Port |
| --- | --- | --- |
| Request | 52345 (ephemeral) | 53 (well-known) |
| Reply | 53 | 52345 |

The client uses a high-numbered ephemeral port; the server uses its well-known port. The reply reverses these. This is the same pattern as TCP — port numbers swap in the return direction.

## UDP vs TCP — A Comparison

| Feature | TCP | UDP |
| --- | --- | --- |
| Header size | 20+ bytes | 8 bytes |
| Connection setup | Three-way handshake | None |
| Reliable delivery | Yes | No |
| In-order delivery | Yes | No |
| Flow control | Yes (window size) | No |
| Congestion control | Yes (slow start, etc.) | No |
| Error detection | Checksum | Checksum |
| Speed | Slower (more overhead) | Faster (minimal overhead) |

## Where UDP Is Used

UDP is the right choice when speed matters more than reliability, or when the application handles its own recovery:

- **DNS** (port 53) — small request/response pairs where retrying is cheaper than maintaining state
- **DHCP** (ports 67 and 68) — bootstrap protocol used before a host has TCP capability
- **Streaming video and audio** — a lost packet should be skipped, not replayed
- **Online gaming** — current state matters more than complete history
- **SNMP** (port 161) — network monitoring

If a streaming video lost one frame and TCP retransmitted it, the player would have to pause and wait. A brief glitch is preferable to buffering. UDP enables this trade-off.

## Key Takeaways

- The UDP header contains four fields totalling 8 bytes
- Maximum payload size: 65,527 bytes
- Maximum port number: 65,535
- UDP's protocol number in the IP header is 17 (0x11)
- Port numbers swap between request and reply
- UDP is chosen when low latency matters more than guaranteed delivery
