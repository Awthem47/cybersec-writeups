# DHCP Protocol

DHCP (Dynamic Host Configuration Protocol) is the application-layer protocol that lets devices automatically receive network configuration from a server. When a device joins a network, it doesn't yet have an IP address, a default gateway, or knowledge of where to find DNS — DHCP delivers all of this. Without it, every device would need to be configured manually.

DHCP runs on top of UDP, using two well-known ports:

- **Port 67** — server side
- **Port 68** — client side

It is one of the few protocols where the client begins life with no IP address at all, which forces the protocol to use broadcasts for the early exchanges.

## The Four-Message Exchange — DORA

The DHCP exchange follows a four-step sequence, commonly remembered by the acronym **DORA**: Discover, Offer, Request, Acknowledge.

### Step 1 — DHCP Discover (client → broadcast)

A new device joins the network and has no IP address yet. It broadcasts a Discover message asking any available DHCP server to respond.

- **Source IP:** `0.0.0.0` (the device has no address yet)
- **Destination IP:** `255.255.255.255` (broadcast — everyone on the local network receives this)
- **Source MAC:** the client's MAC address
- **Destination MAC:** `FF:FF:FF:FF:FF:FF` (broadcast)
- **Source port:** 68 (client)
- **Destination port:** 67 (server)

The Discover message includes a randomly generated **transaction ID**, which is used to match all four messages of the same exchange together. It may also include a list of options the client would like configured (subnet mask, default gateway, DNS, lease time).

### Step 2 — DHCP Offer (server → broadcast)

One or more DHCP servers on the network respond with an Offer. The Offer contains a proposed IP address for the client to use, along with the other configuration parameters.

- **Source IP:** the DHCP server's IP address
- **Destination IP:** `255.255.255.255` (still broadcast — the client doesn't have an IP yet, so the server can't address it directly via IP)
- **Source port:** 67
- **Destination port:** 68
- Contains the **offered IP address** the client may use
- Includes subnet mask, default gateway, DNS server, and lease duration
- Carries the same transaction ID as the Discover

If multiple DHCP servers exist on the network, the client may receive multiple Offers and must choose one.

### Step 3 — DHCP Request (client → broadcast)

The client formally requests the offered IP address. This step is also broadcast, which serves two purposes: it confirms the client's selection to the chosen server, and it informs any other DHCP servers (whose offers were not accepted) that they can release the addresses they had reserved.

- **Source IP:** `0.0.0.0` (the client still doesn't formally own an address yet)
- **Destination IP:** `255.255.255.255`
- Contains the requested IP address and identifies which server the client chose
- Same transaction ID as Discover and Offer

### Step 4 — DHCP Acknowledge (server → broadcast)

The chosen DHCP server confirms the lease with an Acknowledge message. From this moment, the client is permitted to use the IP address.

- **Source IP:** the DHCP server's IP
- **Destination IP:** `255.255.255.255`
- Confirms the lease, including final values for all configuration parameters
- Same transaction ID as the previous three messages

After the ACK, the client configures its network stack with the supplied parameters and can begin normal communication.

## Identifying DHCP in Wireshark

Apply the filter `dhcp` (on older versions of Wireshark, use `bootp` — DHCP is built on top of the older BOOTP protocol and the filter name reflects that history).

The Info column will label each message clearly:

- `DHCP Discover`
- `DHCP Offer`
- `DHCP Request`
- `DHCP ACK`

To follow a single complete exchange, filter by the transaction ID:

```
dhcp.id == 0x7f87aa08
```

Replace the hex value with the transaction ID shown in any of the four messages. All four packets of the same exchange share this ID, so this filter isolates the complete DORA sequence.

### Filtering by message type

To isolate only one type of message:

```
dhcp.option.dhcp == 1   # Discover only
dhcp.option.dhcp == 2   # Offer only
dhcp.option.dhcp == 3   # Request only
dhcp.option.dhcp == 5   # ACK only
```

These numbers correspond to the DHCP Message Type option (option 53) carried in every DHCP packet.

## Reading a DHCP Packet

Expanding the DHCP section in the detail pane of any DHCP packet reveals:

- **Message type** (Discover, Offer, Request, ACK)
- **Transaction ID** (links the four messages of an exchange)
- **Client MAC address** (the hardware address requesting the lease)
- **Your (client) IP address** (filled in by the server in Offer and ACK)
- **Server IP address**
- **Options** — a list of configuration parameters, including subnet mask, router (default gateway), DNS server, lease time, and domain name

The Options field is where most of the useful information lives. DHCP options are extensible, and modern networks may include many of them (NTP servers, TFTP boot servers, vendor-specific options for VoIP phones, and so on).

## Lease Renewal

DHCP leases are time-limited. When a lease is half-expired, the client attempts to renew it by sending a unicast Request directly to the original DHCP server. If the server responds with an ACK, the lease is extended. If the server does not respond, the client tries again at 87.5 percent of the lease time, this time via broadcast to any available DHCP server.

If renewal fails entirely before the lease expires, the client must release its IP address and start the full DORA exchange again.

## Security: DHCP-Based Attacks

Like ARP, DHCP has no built-in authentication. The protocol is designed to work before any trust relationships exist, which makes it inherently vulnerable to certain attacks.

### Rogue DHCP Server

An attacker connects a malicious DHCP server to the network. When legitimate clients broadcast Discover messages, the rogue server responds with an Offer containing:

- A valid-looking IP address
- The attacker's machine as the **default gateway** — meaning all internet-bound traffic from the victim flows through the attacker
- The attacker's machine as the **DNS server** — meaning the attacker can resolve domain names to any IP they choose, redirecting victims to phishing sites

This is a powerful man-in-the-middle position established without ever needing to ARP-spoof. The victim simply asked for network configuration and received a poisoned response.

### DHCP Starvation

An attacker floods the DHCP server with Discover messages, each one spoofing a different client MAC address. The server allocates a lease for each request, eventually exhausting its pool of available IP addresses. Legitimate clients joining the network can no longer obtain a lease.

DHCP starvation is often used as a precursor to rogue DHCP attacks: starve the legitimate server, then introduce the rogue server to pick up requests it can no longer serve.

### Mitigations

- **DHCP Snooping** on managed switches — switches inspect DHCP traffic, distinguish trusted ports (where legitimate servers live) from untrusted ports (where clients live), and drop server-type messages arriving on untrusted ports
- **Port security** — limit the number of MAC addresses permitted on a single switch port, defeating starvation attacks that rely on many spoofed MACs
- **Dynamic ARP Inspection** (which complements DHCP snooping by validating ARP packets against the trusted lease database)

DHCP snooping in particular is the most effective single mitigation: with it enabled, rogue DHCP servers are silently dropped at the switch before any client ever sees them.

## Useful Filters for DHCP Analysis

| Filter | What it shows |
| --- | --- |
| `dhcp` | All DHCP traffic |
| `bootp` | Same as `dhcp` on older Wireshark versions |
| `dhcp.id == 0xABCDEFAB` | All four packets of a specific exchange |
| `dhcp.option.dhcp == 1` | Discover messages only |
| `dhcp.option.dhcp == 2` | Offer messages only |
| `dhcp.option.dhcp == 3` | Request messages only |
| `dhcp.option.dhcp == 5` | ACK messages only |

## Key Takeaways

- DHCP follows the four-step DORA sequence: Discover, Offer, Request, Acknowledge
- DHCP runs on UDP, using port 67 (server) and port 68 (client)
- All four messages in an exchange are broadcast because the client has no IP address until the process completes
- The transaction ID links all four messages of a single exchange and is the most useful way to follow a specific lease negotiation
- Lease renewal attempts begin at 50 percent of the lease duration
- DHCP has no authentication, which enables rogue server and starvation attacks
- DHCP Snooping on managed switches is the standard mitigation against rogue DHCP servers
