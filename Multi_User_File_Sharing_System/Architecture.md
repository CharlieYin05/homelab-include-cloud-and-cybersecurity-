# 架构

## 目标1架构图

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

## 1.网络接入层
建立到服务器的安全网络连接。
- 路由器NAT
- Guest Wi-Fi
- Tailscale

## 2.网络授权层
判断用户/设备能否访问服务器特定端口。
- Tailscale Grants
- nftables

## 3.服务认证层
判断到达服务器的用户是谁。
- Linux用户
- Samba用户
- Samba密码认证

## 4.资源授权层
判断用户登录后可以操作哪些文件。
- Samba share 配置（Samba 只暴露 /srv/shares/shared）
- Linux 用户组
- POSIX ACL（ACL默认管理员rwx,用户rw）

## 5.可观测性层
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

