# SuperYellow Proxy

> **Multi TCP Stream Parallel Multiplexing + Adaptive Forward Error Correction — Next-Gen Proxy Protocol**

SuperYellow is a high-performance, censorship-resistant proxy protocol designed for harsh network environments. Through multi TCP stream parallelism, adaptive FEC, uTLS fingerprint camouflage, and REALITY-level SNI disguise, it delivers high-speed, low-latency, anti-censorship proxy experience.

English version: [README_EN.md](README_EN.md)

---

## Downloads

| Platform | File | Size |
|----------|------|------|
| Windows x64 | [superyellow-windows-amd64.exe](../../releases/download/v1.2.0/superyellow-windows-amd64.exe) | 13 MB |
| Android | [SuperYellow-Proxy-v1.0.apk](../../releases/download/v1.2.0/SuperYellow-Proxy-v1.0.apk) | 20 MB |
| iStoreOS/OpenWrt x64 | [SuperYellow_Client_v1.2.1_x86_64.run](../../releases/download/v1.2.0/SuperYellow_Client_v1.2.1_x86_64.run) | 6.5 MB |

Download the client for your platform, then configure your server info in the Web panel.

---

## Protocol Features

### 1. Multi TCP Stream Parallel Multiplexing

SuperYellow establishes **6 parallel TCP streams** (NumStreams = 6) between server and client. Each stream independently carries FEC parity shards — **a single stream failure does not affect overall transmission**.

| Feature | SuperYellow | Traditional Proxy |
|---------|-------------|-------------------|
| Header overhead | None, native TCP | Yes |
| Single stream failure impact | No impact | Full reconnect |
| Throughput on weak networks | High | Drops sharply |

### 2. Adaptive Forward Error Correction

SuperYellow dynamically adjusts FEC parameters every 3 seconds based on RTT and packet loss:

| Network State | Data Shards | Parity Shards | Redundancy |
|--------------|-------------|---------------|------------|
| Low latency, low loss (<2%) | 10 | 1 | 10% |
| Normal | 4 | 1 | 25% |
| High loss (>15%) | 4 | 2 | 50% |

When FEC decoding fails, the system marks the frame as "lost" and the reassembler skips it, continuing to process subsequent data.

### 3. uTLS Fingerprint Camouflage

Uses uTLS to mimic Chrome TLS fingerprint (HelloChrome_Auto). DPI cannot distinguish SuperYellow from normal HTTPS traffic.

### 4. Random Padding (64-byte aligned)

Each frame is padded to 64-byte boundary, eliminating traffic length fingerprinting.

### 5. REALITY-Level SNI Disguise

In Camo mode, the server returns the real www.microsoft.com:443 certificate — the verification phase is completely legitimate.

### 6. VLESS Protocol Passthrough

Directly passes through VLESS protocol without additional encapsulation. Compatible with PassWall, v2rayN, Clash, and other standard VLESS clients.

### 7. SHA-256 Timestamp Authorization

token = SHA256(SHA256(username:password) + timestamp_ms), valid for 15 seconds, anti-replay.

### 8. Mux.cool Multiplexing Support

Source-level support for xray Mux (CMD=3), parsing mux.cool frame protocol. Each sub-stream independently dials the target, resolving PassWall MUX auto-detection compatibility issues.

### 9. Web Management Panel

- **Server panel** (http://SERVER_IP:8080): System status, user management, traffic statistics
- **Client panel** (http://ROUTER_IP:8899 or http://localhost:9999): Configuration, node switching
- Random client generation (user + password), one-click copy VLESS link and JSON config

### 10. Multi-User Support

Multiple users with quotas/passwords, traffic limits, expiry dates, real-time traffic statistics.

---

## Stability Architecture

V8 has undergone extensive real-world testing and multiple rounds of stability fixes:

### Single-Stream Silent Healing

When a TCP stream write/read fails, **only that stream is closed** — no global reconnect. The background `monitorHealth` checks every 2 seconds and automatically rebuilds failed streams. FEC redundancy provides data protection during rebuild — transparent to the user.

```
Traditional: Stream fails → triggerReconnect() → all disconnect → network storm → rebuild
SuperYellow: Stream fails → Close(that stream) → monitorHealth → silent rebuild → seamless
```

### Zombie Session Cleanup

The server periodically cleans up sessions marked as closed=true, releasing memory and resources. New connections can immediately create new sessions, preventing "zombie session denial of service".

### TCP Backpressure Control

- Write timeout: 3 seconds, preventing write goroutine pile-up
- Concurrent write limit: `fs` channel capacity of 64
- Read buffer cap: 32768 bytes, preventing uint16 overflow

### nftables Loop Prevention

The server IP is added to the `psw_vps` nftables set, bypassing PassWall transparent proxy. Prevents SuperYellow's own traffic from being intercepted by PassWall, forming an infinite loop.

---

## Architecture

```
Client Side:
  PassWall/Browser → VLESS :11081 → SuperYellow Client Plugin
       → 6 parallel TCP streams (FEC shards)
       → uTLS (Chrome fingerprint) + TLS 1.3
  ======= TCP (disguised as HTTPS) ========
Server Side:
  Protocol Detection (SNI routing)
       → Camo fallback (disguise) / SuperYellow handler
       → FEC reassembly + frame dispatch
       → Target server (TCP)
```

---

## Quick Deployment

### One-Click Server Install (Ubuntu/Debian)

```bash
wget https://raw.githubusercontent.com/roger19981223-dotcom/superyellow-proxy/main/install.sh
chmod +x install.sh
sudo ./install.sh server
```

### Install Client — iStoreOS/OpenWrt Plugin

```bash
# Upload .run package to router
scp SuperYellow_Client_v1.2.1_x86_64.run root@192.168.100.1:/tmp/

# SSH into router and install
ssh root@192.168.100.1
sh /tmp/SuperYellow_Client_v1.2.1_x86_64.run

# Edit config
vi /etc/config/superyellow_client.json
/etc/init.d/superyellow restart
```

After installation, the PassWall node is auto-registered. Web panel: `http://ROUTER_IP:8899`

LuCI integration: Services → SuperYellow Proxy

### Install Client — Windows

Run `superyellow-windows-amd64.exe` directly. Open `http://localhost:9999` in browser to configure server.

### Install Client — Android

Install `SuperYellow-Proxy-v1.0.apk`. Configure server in the app.

---

## Configuration

### Client Config (superyellow_client.json)

```json
{
  "enable": true,
  "active_node_id": "n_main",
  "nodes": [{
    "id": "n_main",
    "name": "My Server",
    "server": "YOUR_SERVER_IP:8443",
    "username": "YOUR_USERNAME",
    "password": "YOUR_PASSWORD",
    "sni": "YOUR_DOMAIN"
  }]
}
```

> **Important:** If the server has a `domain` configured, the client's `sni` must match it, otherwise the connection will be intercepted by Camo mode.

### Server Config (aether_config.json)

```json
{
  "panel_port": 8080,
  "listen_ports": "8443",
  "domain": "",
  "camo_mode": "proxy",
  "camo_target": "www.microsoft.com:443",
  "panel_user": "admin",
  "panel_pass": "your-strong-password",
  "users": [
    {"username": "Default", "password": "your-password"}
  ]
}
```

> **Note:** When `domain` is empty, a self-signed certificate is used. When set, ACME cert is auto-requested (port 80 must be free).

---

## PassWall Integration

### Add VLESS Node

In PassWall, add an Xray node:
- Protocol: VLESS
- Address: 127.0.0.1
- Port: 11081
- Transport: TCP
- Encryption: none

### Bypass Rules

SuperYellow automatically adds the server IP to the `psw_vps` nftables set, bypassing PassWall transparent proxy. If the bypass is lost after a PassWall restart, the client plugin will auto-restore it.

---

## Technical Details

### FEC Reassembler Working Principle

SuperYellow uses Reed-Solomon erasure coding for frame shard encoding. Each group has N data shards + M parity shards. The receiver can reconstruct original data once enough shards are received.

**Fault tolerance:** When FEC decoding fails (insufficient shards or corruption), the system marks the sequence number as a nil sentinel in readyBuffer. The drainReady loop logs and skips nil entries, continuing to process subsequent decoded data. A single FEC failure will not block the entire data stream.

```
readyBuffer: [seq=100: data] [seq=101: nil(skip)] [seq=102: data] [seq=103: data]
                                                    ↑ drainReady continues advancing
```

### Frame Protocol Format

Each frame starts with magic number `0x41455448` ("AETH"), 21-byte frame header.

```
[Magic:4B][Type:1B][Length:2B][ConnID:4B][Size:4B][Timestamp:8B][Token:16B][Data:NB][Padding]
```

Frame types (client→server):
- 0x01: CONNECT (establish connection)
- 0x02: DATA (bidirectional, transfer data)
- 0x03: CLOSE (bidirectional, close connection)
- 0x05: UDP (UDP proxy)

Server→client:
- 0x01: CONNECT_ACK (connection established)
- 0x02: CONNECT_NAK (connection failed)
- 0x03: CLOSE
- 0x04: DATA

### VLESS Protocol Handling

The client reads VLESS request header protocol bytes and correctly maps SOCKS5 ATYP:
- ATYP 0x01: IPv4 (4 bytes)
- ATYP 0x02/0x03: Domain (1 byte length + domain)
- ATYP 0x04: IPv6 (16 bytes)

VLESS response format: `[Version:1B][AddonLen:2B BE]` (3 bytes). Response is sent before the write goroutine starts, avoiding corruption of the xray protocol state machine.

### ProtocolDemux Routing

The server routes inbound TLS connections based on the `domain` field:
1. SNI matches domain → SuperYellow TLS handler
2. SNI doesn't match → Camo fallback (returns nginx welcome page)

After TLS handshake, the Magic number is checked: Magic present → Aether protocol handler; Magic absent → returns camouflage page.

---

## Protocol Parameters

| Parameter | Value |
|-----------|-------|
| Parallel streams | 6 |
| TLS version | TLS 1.3 only |
| ALPN | h2, http/1.1 |
| Frame magic | 0x41455448 ("AETH") |
| FEC library | github.com/klauspost/reedsolomon |
| TLS fingerprint | uTLS Chrome_Auto |
| Auth algorithm | SHA-256(SHA-256(user:pass) + timestamp_ms) |
| Token validity | 15 seconds |
| Target dial timeout | 30 seconds |
| Client ACK wait | 35 seconds |
| Write timeout | 3 seconds |
| Max concurrent writes | 64 (fs channel capacity) |
| Read buffer cap | 32768 bytes |
| Frame dispatch send timeout | 1 second |

---

## License

MIT License
