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
## 前期准备
由于上一章节访问 FSS 存在两个路径：
- Tailnet 原生路径
- Subnet Router 路径

所以需要先观察流量

### 1.安装抓包工具
安装 `tcpdump`
```
sudo apt update
sudo apt install tcpdump

tcpdump --version
```

### 2.抓 FSS 上的 SSH 和 SMB 相关的 TCP 流量
FSS 上运行：
```
sudo tcpdump -ni enp3s0 'tcp port 445 or tcp port 22'
```
然后 Mac 依次上运行：
```
nc -vz 100.105.XXX.XXX 445
nc -vz fss.cy-server.com 445
nc -vz fss.cy-server.com 22
```
#### 根据输出可以断定：
##### 对于直接访问 Tailscale IP：
```
Mac
 ↓
Tailscale
 ↓
tailscale0
 ↓
FSS
```
FSS 可以看到这是从 `iifname = tailscale0` 进入。

##### 对于从fss.cy-server.com访问：
```
Mac / 普通用户
        ↓
Tailscale Grants
        ↓
cy-server
        ↓ SNAT
192.168.50.13
        ↓
enp3s0
        ↓
FSS
```
FSS nftables 只能看到：
```
iifname = enp3s0
ip saddr = 192.168.50.13
```
说明通过 Subnet Router 后，两者在 FSS 看起来都是同一个 IP。这个身份区分必须继续由 Tailscale Grants 做。


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






