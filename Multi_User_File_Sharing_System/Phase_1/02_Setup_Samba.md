# 搭建Samba和创建Samba用户

**2026-07-29**

---

## 流程
```
Windows Explorer
        │
        ▼
SMB Protocol
        │
        ▼
smbd
        │
        ▼
Samba User Authentication
        │
        ▼
Linux User
        │
        ▼
Linux Groups
        │
        ▼
ACL
        │
        ▼
Disk (/srv/storage)
```

## 什么是Samba
Windows 访问共享文件夹就是是 SMB 协议。Linux 本来并不会 SMB，所以需要 Samba 来"假装自己是一台 Windows 文件服务器"。Samba = Linux 上的 SMB Server，负责：
- 接收 Windows 请求
- 验证用户名密码
- 判断权限
- 读写 Linux 文件

SMB协议虽然是 Windows 的，但是协议是公开的，所以支持很多系统
| 系统      | 是否支持 SMB | 说明                                     |
| ------- | -------- | ------------------------------------------- |
| Windows | 原生支持   | SMB 就是 Windows 自家的协议                   |
| Linux   | 支持     | 通过 Samba（服务端）和 `cifs`/`smbclient`（客户端） |
| macOS   | 原生支持   | Finder 可以直接连接 SMB                      |
| iOS     | 原生支持   | 文件 App 可以直接连接服务器                       |
| Android | 支持     | 大多数文件管理器支持 SMB                         |

## 什么是smbd
```
Windows
    │
SMB Request
    │
    ▼
smbd
    │
系统调用(open/read/write)
    │
Linux Kernel
```

