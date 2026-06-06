# SuperYellow Proxy

> **基于多 TCP 流并行复用 + 自适应 FEC 前向纠错的新一代代理协议**

SuperYellow 是一个高性能、抗检测的隧道代理协议，专为恶劣网络环境设计。通过多 TCP 流并行复用、自适应前向纠错、uTLS 指纹伪装、REALITY 级 SNI 伪装等技术，实现高吞吐、低延迟、抗封锁的代理体验。

English version: [README_EN.md](README_EN.md)

---

## 核心协议优势

### 1. 多 TCP 流并行复用

SuperYellow 在服务端与客户端之间建立 **6 条并行 TCP 流**（NumStreams = 6），每条流独立携带 FEC 分片组，**单流丢包不影响其他流**。

| 特性 | SuperYellow | 传统单连接方案 |
|------|--------|---------------|
| 队头阻塞 | 无（独立流） | 严重 |
| 单流丢包影响 | 仅影响一个分片组 | 阻塞整个连接 |
| 丢包网络吞吐量 | 高 | 急剧下降 |

### 2. 自适应 FEC 前向纠错

SuperYellow 每 3 秒基于 RTT 和丢包率动态调整 FEC 参数：

| 网络状态 | 数据分片 | 校验分片 | 冗余率 |
|---------|---------|---------|-------|
| 低延迟低丢包 (<2%) | 10 | 1 | 10% |
| 正常状态 | 4 | 1 | 25% |
| 高丢包 (>15%) | 4 | 2 | 50% |

FEC 解码失败时，系统自动标记该序列为"跳过"，避免 reassembler 死锁阻塞后续数据。

### 3. uTLS 指纹伪装

使用 uTLS 模拟 Chrome TLS 指纹（HelloChrome_Auto），DPI 无法区分 SuperYellow 流量与正常 HTTPS 浏览。

### 4. 智能 Padding（64 字节对齐）

每帧填充到 64 字节边界，隐藏流量长度特征。

### 5. REALITY 级 SNI 伪装

Camo 模式下，服务端返回真实 www.microsoft.com:443 证书，握手阶段完全合法。

### 6. VLESS 协议透传

直接透传 VLESS 协议，无需额外封装，兼容 PassWall、v2rayN、Clash 等标准 VLESS 客户端。

### 7. SHA-256 时间戳授权

token = SHA256(SHA256(username:password) + timestamp_ms)，令牌 15 秒有效期，防重放。

### 8. Mux.cool 多路复用支持

原生支持 xray Mux（CMD=3），解析 mux.cool 帧协议，每个子流独立拨号目标，兼容 PassWall 重启后 MUX 自动启用的场景。

### 9. Web 管理面板

- **服务端面板** (http://服务器IP:8080)：系统状态、用户管理、流量统计
- **客户端面板** (http://localhost:9999)：开关、节点切换

### 10. 多用户支持

支持多用户名/密码、流量配额、到期时间、实时速率统计。

---

## 稳定性架构

V8 经过大量实战测试和多轮稳定性修复，具备以下可靠性保障：

### 单流静默自愈

当某条物理流出现写入/读取失败时，**仅关闭该条脏流**，不触发全局重连。后台 `monitorHealth` 僵尸检测器每 3 秒巡检一次，自动重建断开的流。FEC 冗余在网络抖动期间提供保护，用户无感知。

```
传统方案: 单流故障 → triggerReconnect() → 全部流断开 → 重连风暴 → 隧道死亡
SuperYellow Proxy: 单流故障 → Close(脏流) → monitorHealth 检测 → 静默重建 → 隧道自愈
```

### 僵尸会话即时清理

服务端在提取会话时，如果发现会话已标记为 closed=true（僵尸），立即从内存 map 中清除并释放资源，允许新连接立即接入，避免"僵尸会话拒绝服务"死锁。

### 带超时的帧队列发送

FEC reassembler 解码后的帧通过 outputCh 传递给帧分发循环。发送使用 1 秒超时（而非直接丢弃），仅在 outputCh 持续满载时才丢帧，避免突发流量导致的数据丢失。

### TCP 背压控制

- 写入超时：3 秒（快速释放锁，避免写入 goroutine 堆积）
- 并发写入限制：`fs` 通道容量 64（限制同时竞争写入锁的 goroutine 数量）
- 读取缓冲区上限：32768 字节（防止 uint16 溢出）

### nftables 代理循环防护

隧道服务器 IP 通过 `psw_vps` nftables 集合绕过 PassWall 透明代理，防止 SuperYellow 隧道自身的出站流量被 PassWall 拦截后通过隧道回环。

---

## 架构图

```
客户端侧:
  PassWall/浏览器 -> VLESS :11080 -> SuperYellow 客户端
       -> 6 条并行 TCP 流 (FEC 分片)
       -> uTLS (Chrome 指纹) + TLS 1.3
  ======= TCP (伪装为 HTTPS) =======
服务端侧:
  协议识别 (SNI+ALPN)
       -> Camo 返回(微软) / SuperYellow 处理
       -> FEC 解码 + 帧重组
       -> 目标服务器 (TCP)
```

---

## 快速部署

### 一、安装服务端 (Ubuntu/Debian)

```bash
wget https://raw.githubusercontent.com/roger19981223-dotcom/superyellow-proxy/main/install.sh
chmod +x install.sh
sudo ./install.sh server
```

### 二、安装客户端 (OpenWrt/iStoreOS)

```bash
wget https://raw.githubusercontent.com/roger19981223-dotcom/superyellow-proxy/main/install.sh
chmod +x install.sh
./install.sh client
```

---

## 配置说明

### 客户端配置 (aether_client.json)

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

> **重要：** 如果服务端设置了 domain 字段，客户端的 sni 必须与之匹配，否则连接会被 Camo 模式丢弃。

### 服务端配置 (aether_config.json)

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

> **注意：** domain 为空时使用自签名证书；设为域名时自动申请 ACME 证书（需 80 端口空闲）。

---

## PassWall 集成

### 添加 VLESS 节点

在 PassWall 中添加 Xray 节点：
- 协议：VLESS
- 地址：127.0.0.1
- 端口：11080
- 传输：TCP
- 加密：none

### PassWall 配置持久化脚本

PassWall 重启会丢失手动配置（tcp_node 回默认、Direct.ip_list 清空、nftables bypass 清空）。部署自动修复：

```bash
cat > /etc/aether-fix.sh << 'EOF'
#!/bin/sh
# 1. 检查 tcp_node 是否指向 SuperYellow
CURRENT_NODE=$(uci get passwall.@global[0].tcp_node 2>/dev/null)
if [ "$CURRENT_NODE" != "gCWK09oR" ]; then
    uci set passwall.@global[0].tcp_node='gCWK09oR'
    uci commit passwall
    /etc/init.d/passwall restart
    sleep 10
fi
# 2. 确保 MUX 关闭（SuperYellow 已在源码支持 MUX，但关闭可减少开销）
uci set passwall.gCWK09oR.mux='0' 2>/dev/null
uci commit passwall 2>/dev/null
# 3. 恢复 nftables 绕过（PassWall 重启会清空）
nft add element inet passwall psw_vps { YOUR_SERVER_IP } 2>/dev/null
# 4. 确保 sniffing_override_dest 开启
uci set passwall.@global[0].sniffing_override_dest='1' 2>/dev/null
uci commit passwall 2>/dev/null
# 5. 确保 Docker 容器运行
docker restart aether-client 2>/dev/null
EOF
chmod +x /etc/aether-fix.sh
echo '*/5 * * * * /etc/aether-fix.sh >/dev/null 2>&1' | crontab -
```

> **说明：** Mux.cool (CMD=3) 已在客户端源码层支持，无需依赖 PassWall 配置。此脚本仅解决 PassWall 重启后的配置丢失问题。

---

## 技术细节

### FEC Reassembler 工作原理

SuperYellow 使用 Reed-Solomon 纠删码对数据帧进行分片编码。每组包含 N 个数据分片和 M 个校验分片。接收端在收到足够分片后重组原始数据。

**死锁防护：** 当某组 FEC 解码失败（分片不足或损坏），该序列号被标记为 nil 占位符写入 readyBuffer，drainReady 循环遇到 nil 时跳过并记录日志，继续处理后续已解码的数据。这确保了一个 FEC 组的失败不会阻塞整个数据流。

```
readyBuffer: [seq=100: data] [seq=101: nil(skip)] [seq=102: data] [seq=103: data]
                                                     ↑ drainReady 从这里继续输出
```

### 帧协议格式

每帧以魔数 `0x41455448` ("AETH") 开头，包含 21 字节帧头：

```
[魔数:4B][类型:1B][保留:2B][ConnID:4B][长度:4B][时间戳:8B][token:16B][数据:NB][padding]
```

帧类型：
- 0x01: CONNECT（客户端→服务端，建立连接）
- 0x02: DATA（双向，传输数据）
- 0x03: CLOSE（双向，关闭连接）
- 0x05: UDP（客户端→服务端，UDP 代理）

服务端→客户端类型：
- 0x01: CONNECT_ACK（连接建立成功）
- 0x02: CONNECT_NAK（连接建立失败）
- 0x03: CLOSE
- 0x04: DATA

### VLESS 协议处理

客户端读取 VLESS 请求头后分析协议字节，正确映射 SOCKS5 ATYP：
- ATYP 0x01: IPv4（4 字节）
- ATYP 0x02/0x03: 域名（1 字节长度 + 域名）
- ATYP 0x04: IPv6（16 字节）

VLESS 响应格式：`[版本:1B][AddonLen:2B BE]`（3 字节），发送后才启动写入 goroutine，避免 xray 协议握手被破坏。

### ProtocolDemux 路由

当服务端配置了 `domain` 字段时，所有入站 TLS 连接按以下规则路由：
1. SNI 匹配 domain → SuperYellow TLS 处理器
2. ALPN 为 `aether/1` → SuperYellow TLS 处理器
3. 其他 → Camo 处理器（代理到 www.microsoft.com:443）

因此客户端必须配置正确的 `sni` 字段（与服务端 domain 一致），否则会被 Camo 模式丢弃。

---

## 协议参数

| 参数 | 值 |
|------|---|
| 并行流数 | 6 |
| TLS 版本 | TLS 1.3 only |
| ALPN | aether/1 |
| 帧头魔数 | 0x41455448 ("AETH") |
| FEC 库 | github.com/klauspost/reedsolomon |
| TLS 指纹 | uTLS Chrome_Auto |
| 授权算法 | SHA-256(SHA-256(user:pass) + timestamp_ms) |
| Token 有效期 | 15 秒 |
| 目标拨号超时 | 30 秒 |
| 客户端 ACK 等待 | 35 秒 |
| 写入超时 | 3 秒 |
| 并发写入限制 | 64 (fs 通道容量) |
| 读取缓冲区上限 | 32768 字节 |
| 帧队列发送超时 | 1 秒 |

---

## 许可证

MIT License
