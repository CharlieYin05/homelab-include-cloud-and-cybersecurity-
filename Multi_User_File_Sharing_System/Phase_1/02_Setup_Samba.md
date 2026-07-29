# 搭建Samba和创建Samba用户

**2026-07-29**

---

## 为何使用Samba
SMB 协议是目前兼容性最好、支持设备最多、使用最广泛的**局域网文件共享协议**。由微软开发，因 Windows 高市占率而普及。由于 SMB 协议是公开的，因此支持大部分常用系统。

| 协议      | Windows | Linux | macOS  | iOS  | Android | NAS | 主要场景       |
| ------- | ------- | ----- | ------ | ---- | ------- | --- | ---------- |
| **SMB** | ✅ 原生    | ✅     | ✅ 原生   | ✅ 原生 | ✅       | ✅   | 通用文件共享     |
| NFS     | ⚠️ 可选   | ✅ 原生  | ✅      | ❌    | 少       | ✅   | Linux/Unix |
| AFP     | ❌       | ❌     | ⚠️ 已淘汰 | ⚠️   | ❌       | 少   | 老版本 Apple  |
| FTP     | ✅       | ✅     | ✅      | ✅    | ✅       | ✅   | 文件传输，不是共享  |
| WebDAV  | ✅       | ✅     | ✅      | ✅    | ✅       | ✅   | 云盘、远程协作    |

Windows 访问共享文件夹就是 SMB 协议。Linux 本来并不会 SMB，所以需要 Samba 来"假装自己是一台 Windows 文件服务器"。Samba = Linux 上的 SMB 服务器。


## 流程
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



