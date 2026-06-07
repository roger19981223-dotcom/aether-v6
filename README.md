# SuperYellow Proxy

[English](README_EN.md) | **中文**

## 简介

SuperYellow Proxy 是一个基于 VLESS 协议的隧道代理，支持多 TCP 流并行复用与自适应 FEC 纠错，适用于弱网环境下的稳定翻墙。

## 特性

- **多 TCP 流并行复用** — 6 条 TCP 流并行传输，提升吞吐量
- **自适应 FEC 纠错** — 根据丢包率自动调整冗余度，弱网下保持可用
- **uTLS 指纹伪装** — TLS 握手与正常浏览器完全一致
- **SNI Demux 路由** — 基于 SNI 的流量分发，非 Aether 流量返回正常网页
- **VLESS 协议透传** — 兼容 xray VLESS，支持 Mux.cool 多路复用
- **单流自愈** — 单条流故障自动隔离，其余流继续工作
- **僵尸会话清理** — 自动检测并清理死连接

## 下载

| 平台 | 文件 | 说明 |
|------|------|------|
| Windows | `superyellow-windows-amd64.exe` | 命令行客户端 |
| Android | `SuperYellow-Proxy-v1.0.apk` | Android 客户端 |
| iStoreOS/OpenWrt | `SuperYellow_Client_v1.2.2_x86_64.run` | 路由器插件 (x86_64) |

👉 [GitHub Releases](https://github.com/roger19981223-dotcom/superyellow-proxy/releases)

## 服务端部署

### 环境要求

- Linux 服务器 (推荐 Ubuntu/Debian)
- 公网 IP
- 域名（可选，用于 TLS）

### 安装

```bash
# 下载服务端
wget https://github.com/roger19981223-dotcom/superyellow-proxy/releases/latest/download/aether-server-linux-arm64

# 创建配置目录
mkdir -p ~/aether-server

# 创建配置文件
cat > ~/aether-server/aether_config.json << 'EOF'
{
  "panel_port": "8080",
  "listen_port": "8443",
  "camo_mode": "self",
  "panel_user": "admin",
  "panel_pass": "your-password-here"
}
EOF

# 启动
chmod +x aether-server-linux-arm64
./aether-server-linux-arm64
```

### systemd 服务

```bash
cat > /etc/systemd/system/aether-server.service << 'EOF'
[Unit]
Description=SuperYellow Proxy Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/ubuntu/aether-server
ExecStart=/home/ubuntu/aether-server/aether-server
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable aether-server
systemctl start aether-server
```

## 客户端使用

### Windows

```cmd
superyellow-windows-amd64.exe
```

浏览器打开 `http://127.0.0.1:8899` 配置服务器信息。

### iStoreOS/OpenWrt 路由器

1. 上传 `.run` 文件到路由器 `/tmp/`
2. SSH 登录路由器执行：
   ```bash
   sh /tmp/SuperYellow_Client_v1.2.2_x86_64.run
   ```
3. 打开 Web 面板 `http://<路由器IP>:8899`，填入服务器信息
4. 在 PassWall 中添加节点：
   - 类型：Xray
   - 协议：VLESS
   - 地址：`127.0.0.1`
   - 端口：`11081`
   - 加密：none
   - TLS：关闭
5. 在 PassWall 中切换到 SuperYellow 节点

### Android

安装 `SuperYellow-Proxy-v1.0.apk`，在应用内配置服务器信息。

## 配置说明

### 服务端配置 (aether_config.json)

| 字段 | 说明 | 示例 |
|------|------|------|
| `panel_port` | 管理面板端口 | `8080` |
| `listen_port` | 隧道监听端口 | `8443` |
| `camo_mode` | 伪装模式 | `self`（返回 nginx 默认页） |
| `panel_user` | 面板用户名 | `admin` |
| `panel_pass` | 面板密码 | `your-password` |
| `domain` | 域名（启用 ACME 自动证书） | `example.com` |

### 客户端配置 (superyellow_client.json)

| 字段 | 说明 | 示例 |
|------|------|------|
| `server` | 服务器地址:端口 | `1.2.3.4:8443` |
| `username` | 用户名 | `Default` |
| `password` | 密码 | `your-password` |
| `sni` | SNI 域名（需与服务端 domain 一致） | `example.com` |

## 技术参数

| 参数 | 值 |
|------|-----|
| TCP 流数量 | 6 |
| FEC 数据分片 | 4 |
| FEC 校验分片 | 1 |
| TLS 版本 | 1.3 |
| ALPN | h2, http/1.1 |
| VLESS 端口 | 11081 (客户端) |
| Web 面板端口 | 8899 (客户端) |
| 服务端面板端口 | 8080 |
| 服务端隧道端口 | 8443 |

## License

MIT
