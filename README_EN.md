# SuperYellow Proxy

**English** | [中文](README.md)

## Introduction

SuperYellow Proxy is a VLESS-based tunnel proxy with multi-stream TCP parallelism and adaptive FEC error correction, designed for stable proxying in poor network conditions.

## Features

- **Multi-stream TCP parallelism** — 6 parallel TCP streams for higher throughput
- **Adaptive FEC** — Auto-adjusts redundancy based on packet loss rate
- **uTLS fingerprint** — TLS handshake indistinguishable from normal browsers
- **SNI Demux routing** — Non-Aether traffic returns normal web pages
- **VLESS protocol passthrough** — Compatible with xray VLESS, supports Mux.cool
- **Single-stream self-healing** — Failed streams are isolated, remaining streams continue
- **Zombie session cleanup** — Auto-detects and cleans dead connections

## Download

| Platform | File | Description |
|----------|------|-------------|
| Windows | `superyellow-windows-amd64.exe` | CLI client |
| Android | `SuperYellow-Proxy-v1.0.apk` | Android client |
| iStoreOS/OpenWrt | `SuperYellow_Client_v1.2.2_x86_64.run` | Router plugin (x86_64) |

👉 [GitHub Releases](https://github.com/roger19981223-dotcom/superyellow-proxy/releases)

## Server Setup

```bash
# Download server binary
wget https://github.com/roger19981223-dotcom/superyellow-proxy/releases/latest/download/aether-server-linux-arm64

# Create config
mkdir -p ~/aether-server
cat > ~/aether-server/aether_config.json << 'EOF'
{
  "panel_port": "8080",
  "listen_port": "8443",
  "camo_mode": "self",
  "panel_user": "admin",
  "panel_pass": "your-password-here"
}
EOF

# Run
chmod +x aether-server-linux-arm64
./aether-server-linux-arm64
```

## Client Usage

### Windows

```cmd
superyellow-windows-amd64.exe
```

Open `http://127.0.0.1:8899` in browser to configure.

### iStoreOS/OpenWrt Router

1. Upload `.run` file to router `/tmp/`
2. SSH into router and run:
   ```bash
   sh /tmp/SuperYellow_Client_v1.2.2_x86_64.run
   ```
3. Open web panel `http://<router-ip>:8899`, fill in server info
4. Add node in PassWall:
   - Type: Xray
   - Protocol: VLESS
   - Address: `127.0.0.1`
   - Port: `11081`
   - Encryption: none
   - TLS: off
5. Switch to SuperYellow node in PassWall

### Android

Install `SuperYellow-Proxy-v1.0.apk`, configure server info in the app.

## Configuration

### Server (aether_config.json)

| Field | Description | Example |
|-------|-------------|---------|
| `panel_port` | Admin panel port | `8080` |
| `listen_port` | Tunnel listen port | `8443` |
| `camo_mode` | Camouflage mode | `self` (returns nginx default page) |
| `panel_user` | Panel username | `admin` |
| `panel_pass` | Panel password | `your-password` |
| `domain` | Domain (enables ACME auto-cert) | `example.com` |

### Client (superyellow_client.json)

| Field | Description | Example |
|-------|-------------|---------|
| `server` | Server address:port | `1.2.3.4:8443` |
| `username` | Username | `Default` |
| `password` | Password | `your-password` |
| `sni` | SNI domain (must match server domain) | `example.com` |

## Technical Parameters

| Parameter | Value |
|-----------|-------|
| TCP streams | 6 |
| FEC data shards | 4 |
| FEC parity shards | 1 |
| TLS version | 1.3 |
| ALPN | h2, http/1.1 |
| Client VLESS port | 11081 |
| Client web panel port | 8899 |
| Server panel port | 8080 |
| Server tunnel port | 8443 |

## License

MIT
