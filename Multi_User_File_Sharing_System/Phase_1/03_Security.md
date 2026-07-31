# 安全与防火墙设置

**2026-08-01**

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

### 关于 UDP
UDP 有 Ckecksum，但是没有 acknowledgment number（确认号）和 Sequence number（序号），因此：
- 不保证重传
- 不保证顺序
- 不保证一定送达

### 关于 TCP
三次握手不是因为有校验码，而是依靠 Header 里的：
- Sequence Number
- ACK
- Retransmission
- Flow Control
- Congestion Control
来保证可靠性

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
                Ethernet Cable
                      │
                      ▼
                Network Card (NIC)
                      │
                      ▼
             Linux Network Stack
                      │
              （进入 nftables）
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
      SSH Socket             Samba Socket
          │                       │
          ▼                       ▼
        sshd                    smbd
```
