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

### 1.安装 Samba
安装：
```
sudo apt update
sudo apt install samba
```
查看默认监听端口：
```
sudo ss -tulpn | grep smbd
```
查看配置文件：
```
ls -l /etc/samba/
```
查看 Samba 数据库：
```
sudo pdbedit -L
```

### 2.创建 Samba 用户
创建一名 Samba 用户之前一定要先创建他的 Linux 用户并分好组。因为 Samba 用户 = Linux 用户 + Samba 密码。Samba 用户最终在要 Samba 数据库找到对应的 Linux 用户并映射上去。
- Samba 账号负责认证（Authorization）
- Linux 账号负责授权（Authentication）

确认 Linux 用户的存在：
```
id 用户名
```

创建 Samba 用户并设置密码：
```
sudo smbpasswd -a 用户名
```

查看 Samba 用户（数据库）：
```
sudo pdbedit -L
```

### 3.共享文件
Share（共享）是网络如入口，最终映射到 Linux 路径，而不是真的 Linux 路径。映射的路径要自己写。

先备份 samba 配置文件：
```
sudo cp /etc/samba/smb.conf.bak /etc/samba/smb.conf
```

然后进入配置文件编辑：
```
sudo nano /etc/samba/smb.conf
```

末尾添加要映射的路径
```
[public]									← 客户端里显示的入口名字
    path = /srv/storage/shares/public			                        ← 文件路径
    browseable = yes							        ← 客户端关闭隐藏文件
    read only = no								← Samba 不阻止用户写入 
    valid users = @cy_public_rw @cy_public_ro	← 哪些 Samba 用户允许连接
```

检查语法：
```
testparm
```

重启 Samba 服务：
```
sudo systemctl restart smbd
```

再检查状态（是否activate)：
```
systemctl status smbd --no-pager
```

最后测试安卓 16，iPadOS 18，Windows 11，macOS 15 SMB访问

### 问题1：iPad 识别成read only, 其他平台可以rwx

---
## 问题排查思路2.0
```
① 网络
   │
   ├─ ping
   ├─ TCP 445
   └─ smbd 是否运行
        │
        ▼
② Share
   │
   ├─ testparm
   └─ smbclient -L localhost
        │
        ▼
③ 用户认证
   │
   ├─ pdbedit -L
   ├─ smbpasswd
   └─ journalctl
        │
        ▼
④ Samba 配置
   │
   ├─ valid users
   ├─ write list
   └─ browseable
        │
        ▼
⑤ Linux 权限
   │
   ├─ ls -l
   ├─ getfacl
   └─ id
        │
        ▼
⑥ SELinux/AppArmor
```



