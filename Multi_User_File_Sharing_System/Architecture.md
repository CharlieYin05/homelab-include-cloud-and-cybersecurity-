# 架构

## 目标1架构图
```
                      Internet
                          │
                          │ 无端口转发
                          ▼
                 ┌──────────────────┐
                 │   家庭路由器       │
                 │ NAT / Guest Wi-Fi│
                 └────────┬─────────┘
                          │
                    Guest Wi-Fi
                          │
                          ▼
                ┌───────────────────┐
                │ Linux File Server │
                │                   │
                │  Tailscale        │
                │      │            │
                │  nftables         │
                │      │            │
                │  Samba            │
                │      │            │
                │  POSIX ACL        │
                │      │            │
                │ /srv/shares/shared│
                └───────────────────┘


管理员设备                         用户设备
    │                                  │
    └──────── Tailscale VPN ───────────┘
                     │
                     ▼
              Tailscale Grants
          管理员：SSH + SMB
          用户：仅 SMB TCP 445
```
## 1.分层架构
### 1.1网络接入层
建立到服务器的安全网络连接。
- 路由器NAT
- Guest Wi-Fi
- Tailscale

### 1.2网络授权层
判断用户/设备能否访问服务器特定端口。
- Tailscale Grants
- nftables

### 1.3服务认证层
判断到达服务器的用户是谁。
- Linux用户
- Samba用户
- Samba密码认证

### 1.4资源授权层
判断用户登录后可以操作哪些文件。
- Samba share 配置（Samba 只暴露 /srv/shares/shared）
- Linux 用户组
- POSIX ACL（ACL默认管理员rwx,用户rw）

### 1.5可观测性层
负责记录系统发生了什么。
- Samba full_audit
- journald
- nftables 日志
- vnStat

|    层级   |             记录内容            |
|----------|--------------------------------|
|文件操作   |用户，文件路径，创建，删除，重命名    |
|身份认证   |登陆成功，登陆失败                 |
|网络访问   |被允许或拒绝的端口连接              |
|流量      |Tailscale 和 WiFi 接口的上传和下载量|
|主机状态   |服务启动，停止，异常和系统错误        |

### 1.6数据保护层
降低误删、硬盘损坏和系统故障造成的数据损失。
- 独立冷备份硬盘
- 备份脚本或手动备份流程


## 2. 访问流程
### 2.1 用户访问共享目录
```text
用户设备
   │
   ▼
Tailscale 身份认证
   │
   ▼
建立 WireGuard 加密隧道
   │
   ▼
Tailscale Grants 检查
   │
   ├── TCP 445：允许
   └── 其他端口：拒绝
   │
   ▼
nftables 再次检查
   │
   ▼
Samba 用户认证
   │
   ▼
POSIX ACL 权限检查
   │
   ▼
访问共享文件
```

```text
管理员设备
   │
   ▼
个人 Tailscale 身份
   │
   ▼
Tailscale Grants
   │
   ├── TCP 22：允许
   └── TCP 445：允许
   │
   ▼
nftables
   │
   ├── SSH 管理
   └── SMB 文件访问
```

## 3.信任边界
### 边界一：公网与家庭网络
由路由器 NAT 控制。
原则：
- 不允许来自公网的主动连接
- 不配置 SMB 或 SSH 端口转发

### 边界二：家庭网络与 Guest Wi-Fi
由路由器无线隔离功能控制。
原则：
- 文件服务器不能主动信任家庭主网络
- Guest Wi-Fi 只是额外隔离层，不是唯一安全边界

### 边界三：Tailscale 网络与文件服务器
由 Grants 和 nftables 控制。
原则：
- 接入 VPN 不等于拥有全部权限；
- 每个端口必须单独授权。

### 边界四：操作系统与共享文件
由 Samba 和 POSIX ACL 控制。
原则：
- 能连接 TCP 445 不代表能访问全部目录；
- Samba 用户权限与 Linux 文件权限同时生效。

## 4.身份与权限模型
```text
管理员
├── Tailscale：个人成员
├── 网络权限：SSH + SMB
├── Samba 权限：管理共享目录
└── 文件权限：读 / 写 / 管理

朋友只读用户
├── Tailscale：共享用户
├── 网络权限：仅 SMB
├── Samba 权限：登录共享目录
└── 文件权限：读取

朋友读写用户
├── Tailscale：共享用户
├── 网络权限：仅 SMB
├── Samba 权限：登录共享目录
└── 文件权限：读取和写入
```
