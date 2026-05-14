# Ethernet and ARP

## Ethernet at the Link Layer

Ethernet operates at the link layer. Where IP is responsible for getting data across the entire internet using IP addresses, Ethernet is responsible for getting data across a single local network using MAC addresses. A useful analogy: IP is the street address written on a parcel, and Ethernet is the act of physically handing the parcel to the person at the front desk.

### The Ethernet Frame

When data is transmitted on a local network, it is encapsulated in an Ethernet frame:

| Field | Size | Purpose |
| --- | --- | --- |
| Destination MAC | 6 bytes | Recipient of this frame |
| Source MAC | 6 bytes | Sender of this frame |
| Type | 2 bytes | What is inside: 0x0800 = IPv4, 0x0806 = ARP |
| Data (payload) | 46–1500 bytes | The IP datagram or ARP message |
| CRC | 4 bytes | Error detection |

In Wireshark, expanding the "Ethernet II" section in the detail pane reveals the source MAC, destination MAC, and EtherType.

### MAC Addresses

Every network interface has a unique MAC address assigned by the manufacturer. MAC addresses are 48 bits (6 bytes) long, expressed in hexadecimal:

```
00:1A:2B:3C:4D:5E
```

Key facts:

- MAC addresses are permanent and burned into the hardware (though they can be spoofed in software)
- IP addresses are temporary and assigned by DHCP or manually
- The first three bytes identify the manufacturer — known as the OUI (Organisationally Unique Identifier)
- `FF:FF:FF:FF:FF:FF` is the broadcast MAC address — frames sent to it are received by every device on the local network

## ARP — Address Resolution Protocol

ARP solves a single problem: **"I know the IP address. What is the MAC address?"**

When your computer needs to send data to `192.168.1.2` on the local network, it needs the destination's MAC address to build the Ethernet frame. But it only knows the IP. ARP bridges that gap.

### How ARP Works — Two Steps

**Step 1: ARP Request (broadcast)**

The querying device sends a broadcast frame to every device on the network: "Who has IP 192.168.1.2? Tell me your MAC address."

- Source MAC: the querying device's MAC
- Destination MAC: `FF:FF:FF:FF:FF:FF` (broadcast)
- Payload: the target IP address (192.168.1.2)

**Step 2: ARP Reply (unicast)**

The device that owns that IP address responds directly to the querier: "That's me. My MAC address is AA:BB:CC:DD:EE:FF."

- Source MAC: the responding device's MAC
- Destination MAC: the querying device's MAC (sent directly, not broadcast)
- Payload: the target's MAC address

After this exchange, the querier stores the IP-to-MAC mapping in its ARP cache so it does not need to ask again for some time.

### The ARP Cache

The ARP cache is a table stored on each device, mapping IP addresses to MAC addresses. Entries expire after a configurable time — usually 1–2 minutes on most systems — and must be refreshed with a new ARP request when stale.

The current ARP cache can be viewed from a terminal with:

```
arp -a
```

### Identifying ARP in Wireshark

Apply the filter `arp` to isolate ARP traffic. Two types of messages appear in the Info column:

- `Who has 192.168.1.2? Tell 192.168.1.1` — an ARP Request
- `192.168.1.2 is at AA:BB:CC:DD:EE:FF` — an ARP Reply

## How Ethernet and ARP Work Together — End-to-End

A device at `192.168.1.1` wants to send data to `192.168.1.2` on the same local network:

1. The sender checks its ARP cache for the destination's MAC address
2. If not cached, it broadcasts an ARP Request: "Who has 192.168.1.2?"
3. The owner of `192.168.1.2` responds with an ARP Reply containing its MAC
4. The sender stores the mapping in its ARP cache
5. The sender builds an Ethernet frame with `AA:BB:CC:DD:EE:FF` as the destination MAC
6. The frame is transmitted onto the local network

## Sending Data Outside the Local Network

If the destination is on a different network — for example, `8.8.8.8` (a Google DNS server) — ARP cannot resolve it directly. `8.8.8.8` is not on the local subnet, so no local device will respond to an ARP Request for it.

Instead:

1. The sender recognises that `8.8.8.8` is outside the local network
2. It addresses the frame to the **default gateway** (typically the home router) instead
3. It issues an ARP Request for the router's MAC address
4. It builds the Ethernet frame with the router's MAC as the destination
5. The router receives the frame, strips the Ethernet layer, examines the IP destination, and forwards the packet toward `8.8.8.8` through the next hop

Critically:

- The IP destination remains `8.8.8.8` at every hop along the path
- The MAC destination changes at every hop — first the router's MAC, then the next router's MAC, and so on

This is why MAC addresses are described as link-local: they are only meaningful for a single hop.

## Security: ARP Spoofing and MITM Attacks

ARP has **no authentication**. Any device on a network can broadcast an ARP Reply claiming to own any IP address. There is no mechanism to verify that the claimed mapping is correct.

This vulnerability is exploited in **ARP spoofing** (also called ARP poisoning), where an attacker sends crafted ARP Replies such as:

```
192.168.1.1 is at [attacker's MAC]
```

Every device that receives this poisoned reply updates its ARP cache. From that moment, traffic intended for the gateway (`192.168.1.1`) is sent to the attacker's MAC address instead. The attacker can:

- Forward the traffic to the real gateway after inspecting or modifying it — achieving a **man-in-the-middle (MITM)** position
- Drop the traffic entirely — causing a denial of service
- Inject malicious responses into existing sessions

ARP spoofing is a foundational technique in many local-network attacks, and is a common stepping stone toward credential theft, session hijacking, and traffic interception.

### Mitigations

Defences against ARP spoofing include:

- **Static ARP entries** for critical hosts (effective but unmanageable at scale)
- **Dynamic ARP Inspection (DAI)** on managed switches, validating ARP packets against trusted DHCP bindings
- **Encrypted protocols** (HTTPS, SSH, VPN) that limit the value of intercepted traffic, even if the attacker successfully achieves a MITM position

## Identifying Devices from MAC Addresses

The first three bytes of a MAC address identify the manufacturer. Wireshark automatically translates these:

| MAC prefix | Manufacturer |
| --- | --- |
| `Apple_xx:xx:xx` | Apple device |
| `HewlettPacka_xx:xx:xx` | HP device |
| `LinksysG_xx:xx:xx` | Linksys router |
| `Netgear_xx:xx:xx` | Netgear device |

This is useful for taking quick inventory of a network: even without knowing the names of devices, the manufacturer often reveals what kind of device it is.

## Useful Filters for Ethernet and ARP

| Filter | What it shows |
| --- | --- |
| `arp` | All ARP traffic |
| `eth.addr == AA:BB:CC:DD:EE:FF` | Traffic to or from a specific MAC |
| `arp.opcode == 1` | ARP Requests only |
| `arp.opcode == 2` | ARP Replies only |

## Key Takeaways

- MAC addresses are 48 bits long, written in hexadecimal
- `FF:FF:FF:FF:FF:FF` is the broadcast MAC — frames sent to it reach every device on the local network
- ARP Requests are broadcast; ARP Replies are unicast
- ARP has no authentication, making it vulnerable to spoofing attacks
- The ARP cache stores IP-to-MAC mappings temporarily
- EtherType values: 0x0800 = IPv4, 0x0806 = ARP
- When sending outside the local network, the MAC destination is the router, not the final destination
- IP addresses remain constant end-to-end; MAC addresses change at every hop
- The first three bytes of a MAC address (the OUI) identify the manufacturer
