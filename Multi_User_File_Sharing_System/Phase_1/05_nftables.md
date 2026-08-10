# FSS 防火墙设置

**2026-08-01**

---
## 目标：
实现分层防御（Defense in Depth）中的服务器本机防护这一环。即是 VPN 规则失效未授权用户也无法访问 FSS 的未授权端口（多一层授权 Authorization）。大致想要实现效果如下：
```
FSS 主机防火墙

LAN 192.168.50.0/24
        │
        ├── SMB 445 ────────── 按需求允许
        └── 其他端口 ────────── 默认拒绝

Tailscale / tailscale0
        │
        ├── SMB 445 ────────── 允许
        └── 其他 ────────────── 受限

cy-server 192.168.50.13
        │
        ├── SMB 445 ────────── 允许
        └── SSH 22 ─────────── 允许

Established / Related ───────── 允许
Loopback ────────────────────── 允许

其他未授权入站 ─────────────────── DROP
```
---



---

## 网络知识补充

### L3 Packet结构为：
```
IP Packet
│
├── IP Header
│      ├── Source IP
│      ├── Destination IP
│      ├── TTL
│      ├── Protocol（TCP/UDP）
│      └── ...
│
└── Payload
       │
       ▼
   TCP Segment
   或
   UDP Datagram
```

### 关于 Socket

当 sshd 需要占用 22/TCP 时：
```
客户端
    │
连接22端口
    │
    ▼
systemd监听22端口
    │
发现有人连接
    │
启动 sshd.service
    │
把连接交给 sshd
    │
sshd开始认证
```
22 端口先由 systemd 的 socket 接管，有连接时再启动 sshd。socket 相当于 进程的 “端口占位器 + 连接接收器”。


### 数据流
```
                TCP Packet
                     │
                     ▼
              Network Card (NIC)
                     │
                     ▼
               Linux Kernel
                     │
             （nftables 检查）
                     │
         ┌───────────┴───────────┐
         │                       │
     ACCEPT                  DROP
         │                       │
         ▼                       ▼
   查 Socket Table           直接丢弃
         │
         ▼
   有没有程序监听？
         │
   ┌─────┴─────┐
   │           │
  有           没有
   │           │
   ▼           ▼
交给应用      TCP RST
(sshd/smbd)  (Connection Refused)
```

---

## 常用运维命令

查看某服务的网络接口是否up：
```
ip -br addr
```

查看服务器当前监听的网络接口（对外网络服务）：
```
sudo ss -tulpn
```






