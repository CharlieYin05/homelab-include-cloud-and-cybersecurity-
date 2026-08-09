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

### 查看对外网络服务
这台服务器现在对外提供了哪些服务，是谁在提供，以及监听在哪些端口？

#### 输入：
```
sudo ss -tulpn
```
参数含义：
| 参数   | 含义                           |
| ---- | ---------------------------- |
| `ss` | 查看 Socket（Socket Statistics） |
| `-t` | TCP Socket                   |
| `-u` | UDP Socket                   |
| `-l` | 只显示 Listening（监听）状态          |
| `-p` | 显示对应进程                       |
| `-n` | 数字显示，不解析服务名（22 而不是 ssh）      |


#### 输出（例子）：
```
tcp LISTEN 0 50 0.0.0.0:22 users:(("sshd"))
```
参数含义：
| 字段                     | 示例值                | 含义                                                                       |
| ---------------------- | ------------------ | ------------------------------------------------------------------------ |
| **Netid**              | `tcp`              | 使用的网络协议，这里是 **TCP**。如果是 UDP，则显示 `udp`。                                   |
| **State**              | `LISTEN`           | Socket 当前状态。`LISTEN` 表示正在监听，等待客户端连接。                                     |
| **Recv-Q**             | `0`                | 接收队列长度。对于监听 Socket，表示**等待应用程序 `accept()` 的已完成连接数**。通常为 0。                |
| **Send-Q**             | `50`               | 监听队列（backlog）上限，即最多允许约 **50 个等待处理的新连接**。这个值通常由程序调用 `listen(fd, 50)` 时设置。 |
| **Local Address:Port** | `0.0.0.0:22`       | 本地监听地址和端口。`0.0.0.0` 表示监听所有网卡（所有 IPv4 地址），`22` 是 SSH 默认端口。                |
| **Process**            | `users:(("sshd"))` | 拥有这个 Socket 的进程，这里是 `sshd`（SSH 服务）。                                      |


---

## 常用命令

查看网络接口是否up：
```
ip -br addr
```




