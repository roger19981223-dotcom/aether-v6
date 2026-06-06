# SuperYellow Proxy

> **Next-gen proxy protocol based on multi-stream TCP parallel multiplexing + adaptive FEC forward error correction**

SuperYellow is a high-performance, censorship-resistant tunnel proxy protocol designed for hostile network environments. Through multi-stream TCP parallel multiplexing, adaptive FEC, uTLS fingerprint camouflage, and REALITY-grade SNI disguise, it achieves high throughput, low latency, and anti-blocking capabilities.

## Core Protocol Advantages

### 1. Multi-Stream TCP Parallel Multiplexing

SuperYellow establishes **6 parallel TCP streams** (NumStreams = 6) between server and client, each independently carrying FEC shard groups. **Packet loss on one stream does not affect others.**

| Feature | SuperYellow | Traditional Single-Connection |
|---------|--------|------------------------------|
| Head-of-line blocking | None (independent streams) | Severe |
| Single stream loss impact | Only one shard group | Blocks entire connection |
| Throughput under loss | High | Drastic drop |

### 2. Adaptive FEC Forward Error Correction

SuperYellow dynamically adjusts FEC parameters every 3 seconds based on RTT and packet loss rate:

| Network State | Data Shards | Parity Shards | Redundancy |
|--------------|-------------|---------------|------------|
| Low latency/loss (<2%) | 10 | 1 | 10% |
| Normal | 4 | 1 | 25% |
| High loss (>15%) | 4 | 2 | 50% |

When FEC decode fails, the system automatically marks that sequence as "skipped", preventing reassembler deadlock from blocking subsequent data.

### 3. uTLS Fingerprint Camouflage

Uses uTLS to mimic Chrome TLS fingerprint (HelloChrome_Auto). DPI cannot distinguish SuperYellow traffic from normal HTTPS browsing.

### 4. Smart Padding (64-byte Alignment)

Each frame is padded to 64-byte boundaries, hiding traffic length characteristics.

### 5. REALITY-Grade SNI Disguise

In Camo mode, the server returns the real www.microsoft.com:443 certificate, making the handshake phase completely legitimate.

### 6. VLESS Protocol Passthrough

Directly passes through VLESS protocol without additional encapsulation. Compatible with PassWall, v2rayN, Clash, and other standard VLESS clients.

### 7. SHA-256 Timestamp Authorization

token = SHA256(SHA256(username:password) + timestamp_ms), 15-second token validity, anti-replay.

### 8. Mux.cool Multiplexing Support

Native support for xray Mux (CMD=3). Parses mux.cool frame protocol with independent dial per sub-stream. Compatible with PassWall auto-enabling MUX after restart.

### 9. Web Management Panel

- **Server panel** (http://SERVER_IP:8080): System status, user management, traffic stats
- **Client panel** (http://localhost:9999): Toggle, node switching

### 10. Multi-User Support

Multiple username/password, traffic quotas, expiration dates, real-time rate statistics.

---

## Stability Architecture

V8 has undergone extensive real-world testing and multiple rounds of stability fixes, providing the following reliability guarantees:

### Single-Stream Silent Self-Healing

When a physical stream experiences write/read failures, **only that dirty stream is closed** — no global reconnect is triggered. A background `monitorHealth` zombie detector runs every 3 seconds, automatically rebuilding disconnected streams. FEC redundancy provides protection during network glitches, invisible to the user.

```
Traditional: Single stream fail → triggerReconnect() → all streams die → reconnect storm → tunnel dies
SuperYellow Proxy:   Single stream fail → Close(dirty) → monitorHealth detects → silent rebuild → tunnel heals
```

### Zombie Session Instant Cleanup

When the server extracts a session and finds it marked as closed=true (zombie), it immediately clears it from the memory map and releases resources, allowing new connections to be accepted immediately — preventing "zombie session denial-of-service" deadlock.

### Timeout-Based Frame Queue Sending

Decoded frames from the FEC reassembler are passed to the frame dispatch loop via outputCh. Sending uses a 1-second timeout (instead of immediate drop), only discarding frames when outputCh is continuously full — preventing data loss from traffic bursts.

### TCP Backpressure Control

- Write timeout: 3 seconds (fast lock release, preventing write goroutine pile-up)
- Concurrent write limit: `fs` channel capacity of 64 (limits goroutines competing for write mutex)
- Read buffer cap: 32768 bytes (preventing uint16 overflow)

### nftables Proxy Loop Prevention

The tunnel server IP is added to the `psw_vps` nftables set, bypassing PassWall transparent proxy. This prevents SuperYellow's own outbound traffic to the server from being intercepted by PassWall and creating an infinite loop.

---

## Architecture

```
Client Side:
  PassWall/Browser -> VLESS :11080 -> SuperYellow Client
       -> 6 parallel TCP streams (FEC shards)
       -> uTLS (Chrome fingerprint) + TLS 1.3
  ======= TCP (disguised as HTTPS) =======
Server Side:
  Protocol Detection (SNI+ALPN)
       -> Camo Response (Microsoft) / SuperYellow Processing
       -> FEC Decode + Frame Reassembly
       -> Target Server (TCP)
```

## Quick Deploy

### Install Server (Ubuntu/Debian)

```bash
wget https://raw.githubusercontent.com/roger19981223-dotcom/superyellow-proxy/main/install.sh
chmod +x install.sh
sudo ./install.sh server
```

### Install Client (OpenWrt/iStoreOS)

```bash
wget https://raw.githubusercontent.com/roger19981223-dotcom/superyellow-proxy/main/install.sh
chmod +x install.sh
./install.sh client
```

## Configuration

### Client Config (aether_client.json)

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

**Important:** If the server has `domain` set, the client `sni` must match, otherwise connections are dropped by Camo mode.

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

**Note:** Empty `domain` uses self-signed certs; setting a domain triggers ACME/Let's Encrypt (requires port 80 free).

## PassWall Integration

### Adding VLESS Node

Add an Xray node in PassWall:
- Protocol: VLESS
- Address: 127.0.0.1
- Port: 11080
- Transport: TCP
- Encryption: none

### Config Persistence Script

PassWall restart clears manual config (tcp_node reverts, Direct.ip_list emptied, nftables bypass cleared). Deploy auto-fix:

```bash
cat > /etc/aether-fix.sh << 'EOF'
#!/bin/sh
# 1. Check tcp_node points to SuperYellow
CURRENT_NODE=$(uci get passwall.@global[0].tcp_node 2>/dev/null)
if [ "$CURRENT_NODE" != "gCWK09oR" ]; then
    uci set passwall.@global[0].tcp_node='gCWK09oR'
    uci commit passwall
    /etc/init.d/passwall restart
    sleep 10
fi
# 2. Ensure MUX off (SuperYellow handles MUX in source, but off = less overhead)
uci set passwall.gCWK09oR.mux='0' 2>/dev/null
uci commit passwall 2>/dev/null
# 3. Restore nftables bypass (PassWall clears on restart)
nft add element inet passwall psw_vps { YOUR_SERVER_IP } 2>/dev/null
# 4. Ensure sniffing_override_dest enabled
uci set passwall.@global[0].sniffing_override_dest='1' 2>/dev/null
uci commit passwall 2>/dev/null
# 5. Ensure Docker container running
docker restart aether-client 2>/dev/null
EOF
chmod +x /etc/aether-fix.sh
echo '*/5 * * * * /etc/aether-fix.sh >/dev/null 2>&1' | crontab -
```

> **Note:** Mux.cool (CMD=3) is supported at the client source level. This script only addresses PassWall config loss after restart.

---

## Technical Details

### FEC Reassembler Working Principle

SuperYellow uses Reed-Solomon erasure coding to encode data frames into shards. Each group contains N data shards and M parity shards. The receiver reassembles original data after collecting enough shards.

**Deadlock Prevention:** When a FEC decode fails (insufficient shards or corruption), the sequence number is marked with a nil placeholder in the readyBuffer. The drainReady loop skips nil entries and logs the event, continuing to output subsequent decoded data. This ensures one FEC group's failure doesn't block the entire data flow.

```
readyBuffer: [seq=100: data] [seq=101: nil(skip)] [seq=102: data] [seq=103: data]
                                                     ↑ drainReady continues from here
```

### Frame Protocol Format

Each frame starts with magic `0x41455448` ("AETH"), containing a 21-byte header:

```
[magic:4B][type:1B][reserved:2B][connID:4B][length:4B][timestamp:8B][token:16B][data:NB][padding]
```

Frame types:
- 0x01: CONNECT (client→server, establish connection)
- 0x02: DATA (bidirectional, transmit data)
- 0x03: CLOSE (bidirectional, close connection)
- 0x05: UDP (client→server, UDP proxy)

Server→client types:
- 0x01: CONNECT_ACK (connection established)
- 0x02: CONNECT_NAK (connection failed)
- 0x03: CLOSE
- 0x04: DATA

### VLESS Protocol Handling

The client reads VLESS request headers and correctly maps SOCKS5 ATYP:
- ATYP 0x01: IPv4 (4 bytes)
- ATYP 0x02/0x03: Domain (1 byte length + domain)
- ATYP 0x04: IPv6 (16 bytes)

VLESS response format: `[version:1B][addonLen:2B BE]` (3 bytes). The write goroutine starts only after sending the response, preventing xray protocol handshake corruption.

### ProtocolDemux Routing

When the server has `domain` configured, all inbound TLS connections are routed as follows:
1. SNI matches domain → SuperYellow TLS handler
2. ALPN is `aether/1` → SuperYellow TLS handler
3. Other → Camo handler (proxy to www.microsoft.com:443)

Therefore, the client must configure the correct `sni` field (matching server's domain), otherwise connections are dropped by Camo mode.

---

## Protocol Parameters

| Parameter | Value |
|-----------|-------|
| Parallel streams | 6 |
| TLS version | TLS 1.3 only |
| ALPN | aether/1 |
| Frame magic | 0x41455448 ("AETH") |
| FEC library | github.com/klauspost/reedsolomon |
| TLS fingerprint | uTLS Chrome_Auto |
| Auth algorithm | SHA-256(SHA-256(user:pass) + timestamp_ms) |
| Token validity | 15 seconds |
| Target dial timeout | 30 seconds |
| Client ACK wait | 35 seconds |
| Write timeout | 3 seconds |
| Concurrent write limit | 64 (fs channel capacity) |
| Read buffer cap | 32768 bytes |
| Frame queue send timeout | 1 second |

## License

MIT License
