# TCP vs UDP

## Overview

**TCP (Transmission Control Protocol)** and **UDP (User Datagram Protocol)** are the two main transport layer protocols in the Internet Protocol Suite. They determine how data is sent between applications over a network.

**Quick Comparison:**

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented | Connectionless |
| **Reliability** | Guaranteed delivery | Best effort (may lose packets) |
| **Order** | Ordered delivery | No ordering guarantee |
| **Speed** | Slower | Faster |
| **Overhead** | Higher (20-60 bytes header) | Lower (8 bytes header) |
| **Use Cases** | Web, email, file transfer | Streaming, gaming, DNS |

---

## TCP (Transmission Control Protocol)

### What is TCP?

TCP is a **reliable, connection-oriented protocol** that guarantees delivery and order of data packets.

**Analogy:** Like a phone call - establish connection, exchange data, close connection.

### Key Features

#### 1. Connection-Oriented (Three-Way Handshake)

**Establishing Connection:**
```
Client                    Server
   |                         |
   |--- SYN -------------->  |  (Step 1: Client wants to connect)
   |<-- SYN-ACK ----------|  (Step 2: Server acknowledges)
   |--- ACK -------------->  |  (Step 3: Client confirms)
   |                         |
   [Connection established]
```

**Closing Connection (Four-Way Handshake):**
```
Client                    Server
   |                         |
   |--- FIN -------------->  |  (Client done sending)
   |<-- ACK ----------------|  (Server acknowledges)
   |<-- FIN ----------------|  (Server done sending)
   |--- ACK -------------->  |  (Client acknowledges)
   |                         |
   [Connection closed]
```

#### 2. Reliable Delivery

**How TCP Ensures Reliability:**

**Sequence Numbers:**
```
Client sends:
- Packet 1 (seq: 1000)
- Packet 2 (seq: 2000)
- Packet 3 (seq: 3000)

Server receives:
- Packet 1 ✅
- Packet 3 ✅ (out of order!)
- Packet 2 ✅ (late arrival)

Server reorders: Packet 1, 2, 3
```

**Acknowledgments (ACK):**
```
Client                    Server
   |--- Packet 1 (seq 1000) -->|
   |<-- ACK 1001 --------------|  (Server received 1000)
   |--- Packet 2 (seq 1001) -->|
   |<-- ACK 1501 --------------|  (Server received 1001-1500)
```

**Retransmission:**
```
Client                    Server
   |--- Packet 1 ------------>|  ✅ Received
   |--- Packet 2 ------------>|  ❌ Lost
   |--- Packet 3 ------------>|  ✅ Received
   |<-- ACK for Packet 1 -----|
   |<-- ACK for Packet 1 -----|  (Still waiting for Packet 2)
   |   (timeout)               |
   |--- Packet 2 (retry) ----->|  ✅ Received
   |<-- ACK for Packet 3 -----|
```

#### 3. Flow Control

Prevents sender from overwhelming receiver.

**Sliding Window:**
```
Sender window size: 4 packets
Receiver buffer: 4 packets

Sender can send 4 packets before waiting for ACK

Receiver advertises window size:
"I can handle 8 more packets" → Sender increases window
"I can only handle 2 more" → Sender decreases window
```

**Python Example:**
```python
import socket

# TCP Server
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.bind(('0.0.0.0', 8080))
server_socket.listen(5)  # Backlog of 5 connections

print("TCP Server listening on port 8080")

while True:
    # Accept connection (3-way handshake)
    client_socket, address = server_socket.accept()
    print(f"Connection from {address}")

    # Receive data
    data = client_socket.recv(1024)  # Buffer size
    print(f"Received: {data.decode()}")

    # Send response
    client_socket.send(b"Message received")

    # Close connection (4-way handshake)
    client_socket.close()
```

```python
# TCP Client
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect (3-way handshake)
client_socket.connect(('server.example.com', 8080))

# Send data
client_socket.send(b"Hello, Server!")

# Receive response
response = client_socket.recv(1024)
print(f"Server response: {response.decode()}")

# Close connection
client_socket.close()
```

#### 4. Congestion Control

Prevents network congestion.

**Slow Start:**
```
Start: Window size = 1
After ACK: Window size = 2
After ACK: Window size = 4
After ACK: Window size = 8
... (exponential growth until threshold)
```

**Congestion Avoidance:**
```
Threshold reached → Linear growth
Window: 64, 65, 66, 67...

Packet loss detected → Reduce window by half
```

### TCP Header

```
 0                   15                              31
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port        |    Destination Port   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Sequence Number                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Acknowledgment Number                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset| Flags |           Window Size              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum          |    Urgent Pointer     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Header size: 20-60 bytes (20 bytes minimum)
```

### TCP Pros and Cons

**Pros:**
- ✅ Guaranteed delivery
- ✅ Ordered packets
- ✅ Error checking
- ✅ Flow control
- ✅ Congestion control
- ✅ Widely supported

**Cons:**
- ❌ Higher latency (handshakes, ACKs)
- ❌ Higher overhead (larger headers)
- ❌ Head-of-line blocking (waiting for lost packet)
- ❌ Not suitable for real-time

---

## UDP (User Datagram Protocol)

### What is UDP?

UDP is a **connectionless, unreliable protocol** that sends packets without guarantees.

**Analogy:** Like sending postcards - fire and forget, may arrive out of order or not at all.

### Key Features

#### 1. Connectionless

No handshake, no connection state.

```
Client                    Server
   |                         |
   |--- Packet 1 ---------->|  (Just send it!)
   |--- Packet 2 ---------->|
   |--- Packet 3 ---------->|
```

#### 2. Unreliable

No ACKs, no retransmission.

```
Client                    Server
   |--- Packet 1 ---------->|  ✅ Received
   |--- Packet 2 ---------->|  ❌ Lost (no one knows or cares)
   |--- Packet 3 ---------->|  ✅ Received
```

#### 3. No Ordering

Packets may arrive out of order.

```
Client sends: 1, 2, 3, 4, 5
Server receives: 1, 3, 2, 5, 4
```

**Python Example:**
```python
import socket

# UDP Server
server_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server_socket.bind(('0.0.0.0', 8080))

print("UDP Server listening on port 8080")

while True:
    # Receive datagram
    data, client_address = server_socket.recvfrom(1024)
    print(f"Received from {client_address}: {data.decode()}")

    # Send response
    server_socket.sendto(b"Message received", client_address)
```

```python
# UDP Client
client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Send datagram (no connection)
client_socket.sendto(b"Hello, Server!", ('server.example.com', 8080))

# Receive response (may timeout)
client_socket.settimeout(5)  # 5 second timeout
try:
    data, server_address = client_socket.recvfrom(1024)
    print(f"Server response: {data.decode()}")
except socket.timeout:
    print("No response from server")

client_socket.close()
```

### UDP Header

```
 0                   15                              31
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port        |    Destination Port   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Length          |        Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Header size: 8 bytes (fixed)
```

**Much simpler than TCP!**

### UDP Pros and Cons

**Pros:**
- ✅ Very fast (no handshakes)
- ✅ Low overhead (8-byte header)
- ✅ No connection state (stateless)
- ✅ Good for broadcasting/multicasting
- ✅ Low latency

**Cons:**
- ❌ No delivery guarantee
- ❌ No ordering
- ❌ No congestion control (can flood network)
- ❌ No flow control
- ❌ Application must handle reliability (if needed)

---

## Detailed Comparison

### Performance Comparison

| Metric | TCP | UDP |
|--------|-----|-----|
| **Latency** | Higher (handshakes, ACKs) | Lower (send immediately) |
| **Throughput** | Lower (flow/congestion control) | Higher (no controls) |
| **Overhead** | 20-60 bytes | 8 bytes |
| **Connection Setup** | 1.5 RTT (round-trip time) | 0 RTT |
| **Packet Loss Handling** | Retransmit | Ignore |

**Example Latency:**
```
TCP:
- 3-way handshake: 1.5 RTT
- Data transfer: 1 RTT
- Total: 2.5 RTT

UDP:
- Data transfer: 1 RTT
- Total: 1 RTT

For 100ms RTT:
TCP: 250ms to first byte
UDP: 100ms to first byte
```

### Use Case Comparison

| Use Case | Protocol | Why |
|----------|----------|-----|
| **Web Browsing (HTTP/HTTPS)** | TCP | Need reliable delivery |
| **Email (SMTP, IMAP)** | TCP | Can't lose messages |
| **File Transfer (FTP, SFTP)** | TCP | Must deliver all data |
| **SSH** | TCP | Reliability critical |
| **Video Streaming (YouTube)** | TCP | Buffering OK, quality matters |
| **Live Streaming (Twitch)** | UDP | Latency > quality |
| **Video Calls (Zoom, Skype)** | UDP | Real-time, drop frames OK |
| **Online Gaming** | UDP | Low latency critical |
| **DNS Queries** | UDP | Small, fast, can retry |
| **VoIP** | UDP | Real-time audio |
| **IoT Sensors** | UDP | Fire and forget |
| **NTP (Time Sync)** | UDP | Fast, periodic updates |

---

## When to Use Each

### Use TCP When:
1. ✅ **Reliability is critical** - File transfers, emails, transactions
2. ✅ **Order matters** - Chat messages, database replication
3. ✅ **Can't afford data loss** - Financial data, medical records
4. ✅ **Latency is acceptable** - Web browsing (buffering OK)
5. ✅ **Standard protocol** - Most applications default to TCP

### Use UDP When:
1. ✅ **Speed > reliability** - Real-time gaming, voice/video calls
2. ✅ **Can tolerate packet loss** - Video streaming (drop a frame)
3. ✅ **Small, independent messages** - DNS queries, NTP
4. ✅ **Broadcasting/multicasting** - Live sports scores
5. ✅ **Low latency critical** - Online FPS games
6. ✅ **Application handles reliability** - QUIC (HTTP/3)

---

## Hybrid Approaches

### QUIC Protocol (HTTP/3)

**UDP + Reliability (custom):**
```
HTTP/3 uses QUIC protocol
    ↓
QUIC runs on UDP (for speed)
    ↓
QUIC implements TCP-like features:
- Reliable delivery
- Congestion control
- Flow control
    ↓
But faster than TCP:
- No head-of-line blocking
- 0-RTT connection (cached)
- Better on poor networks
```

**Benefits:**
- UDP speed
- TCP reliability
- Better than both!

### WebRTC

**Adaptive protocol:**
```python
# WebRTC can use both
- SRTP (Secure RTP) over UDP for media (audio/video)
- SCTP over UDP for data channels
- Automatically adapts to network conditions
```

### Custom Reliability over UDP

**Implement your own reliability:**
```python
import socket
import time

class ReliableUDP:
    def __init__(self, host, port):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.host = host
        self.port = port
        self.sequence = 0

    def send_reliable(self, data):
        """Send with retry logic"""
        self.sequence += 1
        packet = f"{self.sequence}:{data}".encode()

        max_retries = 3
        for attempt in range(max_retries):
            self.sock.sendto(packet, (self.host, self.port))

            # Wait for ACK
            self.sock.settimeout(1)
            try:
                ack, _ = self.sock.recvfrom(1024)
                if ack.decode() == f"ACK:{self.sequence}":
                    return True
            except socket.timeout:
                print(f"Retry {attempt + 1}")
                continue

        return False  # Failed after retries
```

---

## TCP Variants

### TCP Cubic (Default in Linux)
- Better for high-bandwidth networks
- Faster recovery from packet loss

### TCP BBR (Bottleneck Bandwidth and RTT)
- Google's algorithm
- Better for high-latency networks
- Used by YouTube, Google services

### TCP Vegas
- Proactive congestion avoidance
- Better for stable networks

---

## Real-World Examples

### Netflix Streaming
- **Uses:** TCP (HTTP)
- **Why:** Quality > latency, can buffer
- **Adaptive bitrate:** Adjusts quality based on bandwidth

### Zoom Video Calls
- **Uses:** UDP (primary), TCP (fallback)
- **Why:** Real-time, low latency critical
- **Fallback:** TCP if UDP blocked by firewall

### Online Gaming (Fortnite, CS:GO)
- **Uses:** UDP
- **Why:** 100ms lag = death, drop packets OK
- **Technique:** Client prediction, server reconciliation

### DNS Queries
- **Uses:** UDP (primary), TCP (fallback for large responses)
- **Why:** Fast, small packets, can retry
- **Fallback:** TCP if response > 512 bytes

### SSH
- **Uses:** TCP
- **Why:** Can't lose keystrokes, order critical
- **Reliability:** Every command must arrive correctly

---

## Debugging and Monitoring

### Check Connection Type

```bash
# View active TCP connections
netstat -an | grep tcp

# View active UDP connections
netstat -an | grep udp

# Detailed TCP stats
ss -tan

# Packet capture
tcpdump -i eth0 'tcp port 80'
tcpdump -i eth0 'udp port 53'
```

### Performance Tuning (Linux)

```bash
# TCP buffer sizes
sysctl net.ipv4.tcp_rmem  # Receive buffer
sysctl net.ipv4.tcp_wmem  # Send buffer

# TCP congestion control algorithm
sysctl net.ipv4.tcp_congestion_control

# Change to BBR
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

---

## Summary

**TCP:**
- Connection-oriented, reliable
- Ordered delivery
- Higher latency, overhead
- Use for: Web, email, file transfer, SSH

**UDP:**
- Connectionless, unreliable
- No ordering guarantee
- Lower latency, overhead
- Use for: Streaming, gaming, DNS, VoIP

**Modern Trend:**
- HTTP/3 (QUIC) - UDP with custom reliability
- Best of both worlds

**Interview Tips:**
- Explain 3-way handshake (TCP connection)
- Discuss trade-offs (reliability vs speed)
- Give specific use cases for each
- Mention QUIC as evolution

---

**Next:** Learn about [DNS](dns.md) which uses UDP (and sometimes TCP) for domain resolution.
