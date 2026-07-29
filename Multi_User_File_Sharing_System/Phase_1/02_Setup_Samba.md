# 搭建Samba和创建Samba用户

**2026-07-29**

---

## 为何使用 Samba
SMB 协议是目前兼容性最好、支持设备最多、使用最广泛的**局域网文件共享协议**。由微软开发，因 Windows 高市占率而普及。由于 SMB 协议是公开的，因此支持大部分系统。

| 协议      | Windows | Linux | macOS  | iOS  | Android | NAS | 主要场景       |
| ------- | ------- | ----- | ------ | ---- | ------- | --- | ---------- |
| **SMB** | ✅ 原生    | ✅     | ✅ 原生   | ✅ 原生 | ✅       | ✅   | 通用文件共享     |
| NFS     | ⚠️ 可选   | ✅ 原生  | ✅      | ❌    | 少       | ✅   | Linux/Unix |
| AFP     | ❌       | ❌     | ⚠️ 已淘汰 | ⚠️   | ❌       | 少   | 老版本 Apple  |
| FTP     | ✅       | ✅     | ✅      | ✅    | ✅       | ✅   | 文件传输，不是共享  |
| WebDAV  | ✅       | ✅     | ✅      | ✅    | ✅       | ✅   | 云盘、远程协作    |

Windows 访问共享文件夹就是 SMB 协议。Linux 本来并不会 SMB，所以需要 Samba 来"假装自己是一台 Windows 文件服务器"。Samba = Linux 上的 SMB 服务器。

## 为何不像 Immich 一样 Docker 部署
- Samba 运行起来更像系统服务（system service），而不是像 Immich 那样的应用（application），因此不装在容器里能更好的调用系统内核。
- Samba 深度依赖 Linux 用户和组以及 Linux 文件权限。
- 容器内外 UID/GID 容易不一致导致权限映射困难。
- Samba 本来就必须访问服务器创建的用户，权限和磁盘资源，容器隔离价值不大。

## 文件共享流程
```
Windows Explorer
        │
        ▼
SMB Protocol
        │
        ▼
smbd（Samba 服务）
        │
        ├─ 验证 SMB 用户名/密码（Samba 密码库）
        │
        ▼
映射到 Linux 用户（UID/GID）
        │
        ├─ 检查 Samba 共享配置（如 valid users）
        │
        ▼
Linux Kernel
        │
        ├─ 检查 Unix 权限位
        ├─ 检查 ACL
        └─ 检查 SELinux/AppArmor（如果启用）
        │
        ▼
Disk (/srv/storage)
```

---

## 过程

### 安装

### 创建 Samba 用户
创建一名 Samba 用户之前一定要先创建他的 Linux 用户并分好组。因为 Samba 用户 = Linux 用户 + Samba 密码。Samba 用户最终在要 Samba 数据库找到并映射到具体 Linux 用户
- Samba 账号负责认证（Authorization）
- Linux 账号负责授权（Authentication）
```
Samba 用户
      │
必须对应
      ▼
Linux 用户
      │
最终决定权限的是
      ▼
Linux Group + ACL
```






