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

只读用户
├── Tailscale：共享用户
├── 网络权限：仅 SMB
├── Samba 权限：登录共享目录
└── 文件权限：读取

读写用户
├── Tailscale：共享用户
├── 网络权限：仅 SMB
├── Samba 权限：登录共享目录
└── 文件权限：读取和写入
```

| 角色   | SSH | SMB | 家庭 LAN | 读取文件 | 写入文件 | 修改权限 |
| ---- | --: | --: | -----: | ---: | ---: | ---: |
| 管理员  |   是 |   是 |      是 |    是 |    是 |    是 |
| 只读用户 |   否 |   是 |      否 |    是 |    否 |    否 |
| 只写用户 |   否 |   是 |      否 |    是 |    是 |    否 |

## 5.网络接口设计
```
lo
└── 本机服务

tailscale0
├── Samba TCP 445
└── 管理员 SSH TCP 22

wlan0
├── 允许服务器主动访问互联网
├── 不监听 Samba
├── 不监听 SSH
└── 不向 Guest Wi-Fi 客户端暴露服务
```

## 6.数据目录设计（暂定）
```
/srv/
├── shares/
│   └── shared/          # 第一阶段唯一共享目录
│
├── logs/
│   ├── samba/
│   └── firewall/
│
├── backups/
│
└── immich/              # 第二阶段预留，目前不启用
```

## 7. 日志流
```
Samba 文件操作
       │
       ▼
Samba full_audit
       │
       ▼
journald / Samba 日志
       │
       ▼
人工查询与归档


网络连接
       │
       ▼
nftables counters / log
       │
       ▼
journald

------------------------
网络流量
       │
       ▼
vnStat
       │
       ▼
每日 / 每月流量统计
```

## 8.已知限制
无法阻止已授权用户复制文件后再次传播；
- 无法保证朋友的终端设备没有被入侵；
- Guest Wi-Fi 隔离能力依赖路由器实现；
- 目标1没有实时告警；
- 没有自动封禁或自动隔离；
- 单台文件服务器存在单点故障；
- 冷备份的实时性低于在线备份；
- 日志只能帮助审计，不能自动阻止所有恶意行为。

## 9.Phase 1验收标准

### 9.1网络
- 公网无法访问 TCP 22、139、445；
- Guest Wi-Fi 和 LAN 客户端无法访问服务器 SMB；
- 用户只能访问文件服务器 TCP 445；
- 用户无法访问路由器或家庭 LAN；
- 管理员可以通过 Tailscale 使用 SSH；

### 9.2
