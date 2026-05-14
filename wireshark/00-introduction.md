# Introduction to Wireshark

Wireshark is a network protocol analyser. It captures traffic flowing across a network interface and lets you inspect each packet in detail — from the raw bytes on the wire all the way up to the application-layer content. This guide assumes you have it installed and a capture file open.

## The Three Panes

The Wireshark interface is divided into three panes, each showing a different level of detail.

### Top pane — Packet List

A table of every captured packet, displayed in chronological order. Think of this as the table of contents for your capture. Each row shows a single packet with columns for time, source and destination addresses, protocol, length, and a summary in the Info column.

### Middle pane — Packet Details

When you select a packet in the top pane, the middle pane breaks it down by protocol layer. For a typical packet you will see:

- **Frame** — physical layer metadata (capture time, interface, frame number)
- **Ethernet II** — link layer (source and destination MAC addresses, EtherType)
- **Internet Protocol** — network layer (source and destination IP addresses, TTL, protocol number)
- **TCP or UDP** — transport layer (source and destination ports, flags, sequence numbers for TCP)
- **HTTP, DNS, or other application protocol** — the actual content being carried

Each section is expandable. This is the pane you spend most of your time in.

### Bottom pane — Packet Bytes

The raw hexadecimal and ASCII representation of the packet. Useful for low-level inspection or when working with custom protocols, but rarely needed for standard analysis.

## Display Filters

Display filters are the single most important skill in Wireshark. They let you narrow a capture of tens of thousands of packets down to exactly what you need to see.

The most commonly used filters:

| Filter | What is shown |
| --- | --- |
| `http` | Only HTTP traffic |
| `dns` | Only DNS traffic |
| `tcp` | Only TCP traffic |
| `udp` | Only UDP traffic |
| `arp` | Only ARP traffic |
| `dhcp` | Only DHCP traffic |
| `ip.addr == 128.119.245.12` | Traffic to or from a specific IP address |
| `tcp.flags.syn == 1` | Only SYN packets (connection starts) |
| `tcp.flags.fin == 1` | Only FIN packets (connection ends) |

Filters can be combined with `and`, `or`, and `not` for more precise queries. For example, `tcp and ip.addr == 192.168.1.1` shows only TCP traffic involving that address.

## Worked Example — Intro Lab

A simple lab exercise: capture traffic while loading [http://gaia.cs.umass.edu/wireshark-labs/INTRO-wireshark-file1.html](http://gaia.cs.umass.edu/wireshark-labs/INTRO-wireshark-file1.html) in a browser, then answer questions about what you observed.

> **Real-world tip:** On modern browsers, this page may be served over HTTPS rather than plain HTTP. If your capture shows TLS traffic instead of HTTP, try copying the link into a different browser (Safari worked when Chrome did not). HTTPS encrypts the payload, which makes inspection of plaintext HTTP headers impossible without decryption.

### Question 1 — Identify three protocols in the capture

The Protocol column in the top pane shows the highest-level protocol identified for each packet. Common entries include HTTP, TCP, UDP, DNS, ARP, DHCP, TLSv1.2, and ICMP. Any three valid entries are correct.

### Question 2 — Time between the HTTP GET and the 200 OK response

This is the round-trip time (RTT) for the request-response pair.

1. Find the GET packet — note its time value
2. Find the matching 200 OK response — note its time value
3. Subtract: `response_time − request_time = RTT`

In one capture, the SYN was sent at 6.7313 seconds and the SYN-ACK arrived at 6.9692 seconds, giving an RTT of approximately 238 milliseconds. The same method applies to GET/response pairs.

### Question 3 — IP addresses of the client and server

- Source IP in the GET packet = your computer's IP address
- Destination IP in the GET packet = the server's IP address

The server's IP for `gaia.cs.umass.edu` is `128.119.245.12`. Your computer's IP will be private (in the `192.168.x.x`, `10.x.x.x`, or `172.16-31.x.x` ranges) if you are behind a home router.

### Question 4 — Number of header fields in the GET request and response

A typical HTTP GET request contains the request line plus 6–8 header fields, including:

- Request line (`GET /path HTTP/1.1`)
- Host
- User-Agent
- Accept
- Accept-Language
- Accept-Encoding
- Connection

A typical 200 OK response contains the status line plus 8–10 header fields, including:

- Status line (`HTTP/1.1 200 OK`)
- Date
- Server
- Last-Modified
- ETag
- Accept-Ranges
- Content-Length
- Keep-Alive
- Connection
- Content-Type

Exact counts vary by browser and server configuration.

## Key Takeaways

- Protocols are identified in the Protocol column of the top pane
- RTT is calculated by subtracting request and response timestamps
- Source and destination IPs identify which endpoints are communicating
- Encapsulation is visible in the middle pane: HTTP sits inside TCP, which sits inside IP, which sits inside an Ethernet frame
