# TCP Protocol

TCP (Transmission Control Protocol) is a transport-layer protocol that provides reliable, ordered delivery of data. Unlike UDP, which sends datagrams without confirmation, TCP guarantees that every byte arrives, in order, and without corruption.

## Three Core Mechanisms

### 1. The Three-Way Handshake (Connection Setup)

Before any data is exchanged, TCP establishes a connection through three packets:

1. **SYN** — The client sends a packet with the SYN flag set, announcing intent to connect. It includes a randomly chosen initial sequence number.
2. **SYN-ACK** — The server responds with both SYN and ACK flags set. It includes its own random initial sequence number and acknowledges the client's by incrementing it by one.
3. **ACK** — The client responds with the ACK flag set, acknowledging the server's sequence number by incrementing it by one.

Only after these three packets does data transfer begin.

### 2. Sequence and Acknowledgment Numbers

These two fields are how TCP tracks data and guarantees reliable delivery.

- **Sequence number** — "I am sending you data starting at byte X." Identifies the position of this segment's data within the overall byte stream.
- **Acknowledgment number** — "I have received everything up to byte X; send me byte X next." Tells the sender what the receiver expects next.

**Worked example:**

- Client sends 1000 bytes with Seq = 1
- Server responds with Ack = 1001, meaning "I received bytes 1–1000, send me byte 1001 next"
- Client sends the next 500 bytes with Seq = 1001
- Server responds with Ack = 1501

### 3. Connection Teardown

When the transfer completes, TCP closes the connection through a four-step exchange of FIN packets:

1. Side A sends FIN — "I have no more data to send"
2. Side B sends ACK — "Received"
3. Side B sends FIN — "I also have no more data"
4. Side A sends ACK — "Received, connection closed"

This four-way close is sometimes referred to as a graceful shutdown.

## The TCP Header

Every TCP segment carries a header containing these key fields:

| Field | Size | Purpose |
| --- | --- | --- |
| Source Port | 16 bits | Identifies the sending process |
| Destination Port | 16 bits | Identifies the receiving process |
| Sequence Number | 32 bits | Byte position of this segment's data |
| Acknowledgment Number | 32 bits | Next byte expected from the other side |
| Header Length | 4 bits | Size of the TCP header (typically 20 bytes) |
| Flags | 6 bits | SYN, ACK, FIN, RST, PSH, URG |
| Window Size | 16 bits | Available receiver buffer space (flow control) |
| Checksum | 16 bits | Error detection |
| Urgent Pointer | 16 bits | Points to urgent data (rarely used) |

## TCP Flags

The flag bits identify what kind of segment you are looking at:

| Flag | Meaning |
| --- | --- |
| SYN = 1 | Connection request |
| SYN = 1, ACK = 1 | Connection accepted |
| ACK = 1 | Acknowledgment of received data |
| FIN = 1 | Connection closing |
| RST = 1 | Connection reset (something went wrong) |
| PSH = 1 | Deliver data to the application immediately |

In Wireshark's Info column you will see notations like `[SYN]`, `[SYN, ACK]`, `[ACK]`, and `[FIN, ACK]` corresponding to these flag combinations.

## Round Trip Time (RTT)

RTT is the time between sending a segment and receiving its acknowledgment, calculated as:

```
RTT = time_of_ACK_received − time_of_segment_sent
```

In Wireshark, locate the data segment, note its timestamp, then locate its matching ACK and note that timestamp. Subtract.

## Maximum Segment Size (MSS)

MSS is the largest amount of data TCP will place in a single segment. On standard Ethernet networks it is typically 1460 bytes.

**Derivation:**

- Ethernet MTU = 1500 bytes (maximum frame payload)
- IP header = 20 bytes
- TCP header = 20 bytes
- MSS = 1500 − 20 − 20 = 1460 bytes

MSS is negotiated during the three-way handshake. Both sides include their MSS in the SYN and SYN-ACK packets via the TCP Options field.

> **Real-world observation:** If a TCP connection negotiates timestamp options (common on modern systems), the TCP header grows from 20 bytes to 32 bytes, reducing the effective MSS from 1460 to 1448. Captures from modern systems often show this. If you observe a non-standard MSS, check the TCP Options field — the answer is almost always there.

## Window Size and Flow Control

The Window Size field announces how much buffer space the receiver currently has available. If the window shrinks, the sender slows down. If it reaches zero, the sender stops transmitting until the receiver advertises new space. This prevents the sender from overwhelming the receiver.

> **Real-world observation:** Modern TCP stacks use *window scaling*, negotiated during the handshake. When window scaling is in effect, the raw value in the Window Size field is multiplied by a scaling factor (commonly 128) to get the true window size. Wireshark normally displays the calculated value, but it is worth confirming whether the value shown is raw or scaled — particularly when interpreting unusually small window numbers.

## Congestion Control

TCP also adjusts how much data it sends based on observed network conditions, through two phases:

### Slow Start

- TCP begins by sending a small amount of data (typically one or two segments)
- For every ACK received, it doubles the amount sent
- This produces exponential growth: 1, 2, 4, 8, 16 segments
- Continues until reaching a threshold (`ssthresh`)

### Congestion Avoidance

- After crossing the threshold, TCP switches to linear growth
- One additional segment is added per round trip, rather than doubling
- Much more cautious

If packet loss is detected, TCP interprets this as congestion, cuts its sending rate dramatically, and restarts the process.

In Wireshark, the slow-start to congestion-avoidance transition can be visualised through **Statistics → TCP Stream Graph → Time-Sequence-Graph**. A steep upward curve indicates slow start; a gradual linear slope indicates congestion avoidance.

> **Real-world observation:** On modern high-speed connections transferring small files, the entire transfer can complete inside the slow-start phase. The classic exponential-to-linear curve may be difficult to observe because TCP never needs to switch phases. This is itself a meaningful observation about how modern networks behave differently from older textbook examples.

## Retransmissions

If a segment is lost and no ACK arrives within the expected window, TCP resends it. In Wireshark this appears as:

- Two packets with the same sequence number
- Wireshark labels the second one as `[TCP Retransmission]`

The filter `tcp.analysis.retransmission` isolates these events.

## Useful Filters for TCP Analysis

| Filter | What it shows |
| --- | --- |
| `tcp` | All TCP traffic |
| `tcp.flags.syn == 1` | SYN packets (handshake starts) |
| `tcp.flags.fin == 1` | FIN packets (connection closes) |
| `tcp.flags.reset == 1` | RST packets (connection errors) |
| `tcp.analysis.retransmission` | Retransmitted segments |
| `frame contains "POST"` | Packets containing an HTTP POST |

## Worked Example — TCP Transfer Analysis

The following observations come from a real capture of an HTTP POST uploading a text file to a remote server. The analysis demonstrates how the concepts above appear in practice.

### Identifying the client endpoint

Looking at the first SYN packet going from client to server:

- Client IP: 192.168.1.118
- Client port: 61205

The client port is an *ephemeral port* — a high-numbered port chosen randomly for the duration of the connection. It will differ on every new connection.

### Identifying the server endpoint

- Server IP: 128.119.245.12
- Server port: 80

Port 80 is the well-known port for HTTP. Servers listen on fixed well-known ports; clients connect from ephemeral ports.

### Identifying SYN, SYN-ACK, and the first data segment

The SYN-ACK has both SYN and ACK flags set, distinguishing it from the initial SYN (which has only SYN set). The acknowledgment number in the SYN-ACK is the client's initial sequence number plus one, confirming receipt of the SYN.

The first data segment in this capture was the HTTP POST itself, at sequence number 1 — because the handshake's SYN consumed sequence number 0.

### The first six data segments

| Segment | Seq Number (relative) | Length (bytes) | Time sent | ACK received | RTT |
| --- | --- | --- | --- | --- | --- |
| 1 (POST) | 1 | 713 | 0.612136 | 0.868450 | 0.256314s |
| 2 | 714 | 1428 | 0.612448 | 0.868453 | 0.256005s |
| 3 | 2142 | 1428 | 0.612451 | 0.868454 | 0.256003s |
| 4 | 3570 | 1428 | 0.612452 | 0.868455 | 0.256003s |
| 5 | 4998 | 1428 | 0.612453 | 0.868457 | 0.256004s |
| 6 | 6426 | 1428 | 0.612453 | 0.868458 | 0.256005s |

Sequence numbers increase by exactly the number of data bytes in the preceding segment. Segment 1 sent 713 bytes (Seq = 1), so segment 2 begins at Seq = 714. Each subsequent segment of 1428 bytes increments the sequence number by 1428.

Note that all six RTTs are nearly identical (~0.256s). This is because all six segments were transmitted in a single burst before any ACKs returned. On a slower link, segments would be sent at different times and would experience different RTTs.

### Why MSS was 1428 rather than 1460

The MSS in this capture is 1428 because the connection negotiated TCP timestamp options:

- Ethernet MTU: 1500
- IP header: 20
- TCP header with timestamps: 32 (20 base + 12 options)
- MSS = 1500 − 20 − 32 = 1448, rounded down to 1428 after accounting for additional padding

This is a useful diagnostic: any time the observed MSS is not 1460, inspect the TCP Options field of the SYN packets.

### Window size and throttling

The minimum raw Window Size value advertised by the receiver in this capture was 504 bytes. However, the connection had negotiated window scaling with a factor of 128, so the actual advertised buffer was 504 × 128 = 64,512 bytes. The window never reached zero, so the sender was never forced to pause.

### Retransmissions

There were no retransmissions in this capture. Verified by filtering with `tcp.analysis.retransmission` and finding no matches, and by confirming no duplicate sequence numbers across the 108 client data segments.

### Acknowledgment behaviour

Most ACKs in this capture acknowledged exactly 1428 bytes — one segment at a time. However, about 13 percent of ACKs used cumulative acknowledgment, jumping by 2856 bytes (two segments). This is normal TCP behaviour: the receiver is permitted to acknowledge multiple segments with a single ACK.

### Throughput calculation

```
Total bytes transferred: 153,034 (the entire uploaded file)
Total time: 1.690247s − 0.612136s = 1.078s
Throughput: 153,034 ÷ 1.078 ≈ 141,946 bytes/sec ≈ 138.6 KB/s
```

This is dramatically faster than older textbook examples (often around 30 KB/s) — a reflection of how much real-world consumer connection speeds have improved.

### Slow start visibility

On this connection the entire 153 KB file transferred in ~1 second, with most data sent in a single burst. The slow-start to congestion-avoidance transition was not clearly visible in the time-sequence graph because the transfer completed too quickly for the threshold to be reached. This is itself a meaningful observation about how modern high-speed connections behave differently from older textbook captures.

## Key Takeaways

- The three-way handshake uses sequence numbers 0 and 0 on each side, with ACKs of 1 and 1
- Data transfer begins at sequence number 1 (the SYN consumed sequence number 0)
- Each segment's sequence number equals the previous sequence number plus the previous segment's data length
- RTT = `time_of_ACK_received − time_of_segment_sent`
- MSS depends on negotiated TCP options — 1448 with timestamps, 1460 without
- Throughput = total data divided by total time
- Window Size advertises receiver buffer space; check whether window scaling is in effect before interpreting raw values
- Retransmissions appear as duplicate sequence numbers and are labelled `[TCP Retransmission]`
