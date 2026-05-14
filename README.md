# cybersec-writeups

Personal notes, tutorials, and lab writeups from my cyber security learning journey.

I'm a postgraduate ICT student at Western Sydney University, building toward a career in cyber security. This repository is where I document what I learn — through coursework, hands-on labs, and self-directed study — in a form that's useful to me later and hopefully useful to others coming through the same material.

## Contents

### Wireshark — Network Protocol Analysis

A multi-part tutorial covering the fundamentals of network analysis with Wireshark, written while preparing for a practical exam in COMP7013 (Network Technologies) at WSU. Each section combines protocol theory with hands-on packet analysis using real captures.

- [00 — Introduction to Wireshark](wireshark/00-introduction.md) — Interface, panes, and essential display filters
- [01 — HTTP Protocol](wireshark/01-http-protocol.md) — GET/response cycles, conditional GETs, persistent vs non-persistent connections
- [02 — TCP Protocol](wireshark/02-tcp-protocol.md) — Three-way handshake, sequence numbers, flow and congestion control, retransmissions
- [03 — UDP Protocol](wireshark/03-udp-protocol.md) — Header structure, use cases, and comparison with TCP
- [04 — IP Protocol](wireshark/04-ip-protocol.md) — IPv4 header, fragmentation, TTL, and traceroute behaviour
- [05 — Ethernet and ARP](wireshark/05-ethernet-and-arp.md) — Frame structure, MAC addressing, ARP resolution, and ARP spoofing
- [06 — DHCP Protocol](wireshark/06-dhcp.md) — The DORA exchange, lease renewal, rogue DHCP servers, and DHCP starvation

## About this repository

Each writeup combines:
- **Protocol theory** — the concepts you need to understand what you're seeing
- **Wireshark mechanics** — how to find, filter, and interpret the relevant packets
- **Real-world observations** — what actual captures from modern systems look like, and where they differ from textbook examples (e.g. TCP timestamp options reducing MSS, window scaling, randomised IP IDs in modern OS stacks)
- **Security context** — where relevant, how each protocol is exploited (e.g. ARP spoofing, MITM positioning, rogue DHCP servers)

## Currently learning

- Wireshark and packet analysis
- Wireless network security
- Information security management
- Python (developing)

## Contact

- LinkedIn: [www.linkedin.com/in/fanny-patel-22043a37b) 
