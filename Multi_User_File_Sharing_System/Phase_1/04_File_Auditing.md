# 搭建 Samba 文件审计日志

**2026-08-01**

---

## 目的
让fss服务器就具备可审计性（Auditability），也为后续学习 SIEM、日志分析和事件响应打下基础。

## 期望
例子：
```
Windows
   │
   │ 建立 test.txt
   ▼
Samba
   │
   ▼
/srv/logs/samba/audit.log

2026-08-01 15:23:11
user=cyin026
share=public
operation=create_file
path=test.txt
```

## 关于审计日志（Audit）
就是用户做了什么。

实现则利用 Samba 的 VFS（Virtual File System）插件，流程如下：
```
Windows
↓

Samba

↓

full_audit 插件
│
├──写日志
│
└──继续存文件
```

## 过程

### 1.检查是否安装了 full_audit
注意 Debian 普通用户的 `PATH` 不包含 `/usr/sbin` ，因此看不到 `/usr/sbin/smbd`。要提权。

查看是否安装：
```
sudo /usr/sbin/smbd -b | grep VFS
```

### 2.开启 full_audit

#### 2.1 先备份 Samba 配置文件
输入：
```
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.audit.bak
```

#### 2.2 编辑配置文件
在 global , public, restriction, private 底下加入：
```
vfs objects = full_audit                                           ← 开启审计日志

full_audit:prefix = %u|%I|%S                                       ← 用户名|客户端IP|共享名称
full_audit:success = mkdir rmdir rename unlink open create_file    ← 哪些成功操作要记录
full_audit:failure = none                                          ← 不要记录失败操作
full_audit:facility = LOCAL7                                       ← 发送到 syslog 的日志设施
full_audit:priority = NOTICE                                       ← 日志等级：仅提醒
```
然后保存检查语法

### 3.把 Audit 日志写到 /srv/logs/samba
流程：
```
Windows
      │
      ▼
 Samba
      │
      ▼
 full_audit
      │
      ▼
 rsyslog
      │
      ▼
 audit.log
```
Debian 里真正负责写文件的是 `rsyslog`。

#### 3.1 查看 Debian 是否自带 `rsyslog`
输入：
```
systemctl status rsyslog
```

如果没有则安装并确认：
```
sudo apt update
sudo apt install rsyslog

systemctl status rsyslog
```

#### 3.2 创建 rsyslog 规则
注意 Debian 不会自动创建 syslog 用户，默认是 Root。后续以 Root 权限操作。

创建 `rsyslog` 配置文件:
```
sudo nano /etc/rsyslog.d/30-samba-audit.conf
```

里面只写一行：
```
local7.notice    /srv/logs/samba/audit.log
```

执行：
```
sudo touch /srv/logs/samba/audit.log
sudo chown root:adm /srv/logs/samba/audit.log
sudo chmod 640 /srv/logs/samba/audit.log
```

验证：
```
ls -l /srv/logs/samba
```

重启 `rsyslog`：
```
sudo systemctl restart rsyslog
```

重启 `smbd`:
```
sudo systemctl restart smbd
```

确认没有报错：
```
sudo systemctl status rsyslog
sudo systemctl status smbd
```

#### 问题1
创建 rsyslog 规则后 SMB 访问失败。

##### 排查
本机服务器安装 Samba 客户端，如果 localhost 能连接就是客户端问题
```
sudo apt install smbclient
```

删除位于 `[global]` 的 VFS 插件，并且临时在 `[global]` 添加一条高等级日志用户排查问题：
```
log level = 3
```

然后 localhost SMB 访问：
```
smbclient -L localhost -U cyin026
```

尝试访问 public
```
smbclient //localhost/public -U cyin026
```
输出：
```
tree connect failed: NT_STATUS_UNSUCCESSFUL
```

| 测试                        | 结果                                      |
| ------------------------- | --------------------------------------- |
| 不启用 `full_audit`          | ✅ SMB 完全正常                              |
| `full_audit` 放 `[global]` | ❌ Share 无法连接 (`NT_STATUS_UNSUCCESSFUL`) |
| `full_audit` 只放 Share     | ✅ 能列出 Share，但无法进入 Share                 |

问题基本锁定在 full_audit 配置本身，而不是 Samba、权限、ACL 或 rsyslog。

审计成功记录里删除 `[public]` 的 open 和 create_file：
```
full_audit:success = mkdir rmdir rename unlink
```

2026/08/02 11:58am
只删除 open 和 create file 依然失败。怀疑是 sucess 的操作参数是老版本（不知道是 Debian 还是 Samba）。先把 Private 套用官方 full audit 模版。
```
    vfs objects = full_audit

    full_audit:prefix = %u|%I
    full_audit:success = open opendir
    full_audit:failure = all !open

    full_audit:facility = LOCAL7
    full_audit:priority = NOTICE
```

2026/08/02 12:06pm
哪怕套用官方模版依然出错。剩下三大可能性：
1. Samba 4.22 + Debian 13 的 full_audit 回归 Bug
2. Debian 的 Samba 打包问题
3. 还有一个没验证的点：full_audit 是否依赖某些默认 VFS 模块（例如 acl_xattr）

2026/08/02 12:14pm
把 Samba 日志等级提高到 11 级， 然后连接后立刻抓取记录。
客户端：
```
smbclient //localhost/private -U cyin026 -d 11
```
服务端：
```
sudo grep -R "init_bitmap" /var/log/samba
```
```
sudo grep -R "Invalid success" /var/log/samba
```

2026/08/02 12:27pm
日志中发现的异常
原来是：
```
init_bitmap: Could not find opname mkdir
```
现在改成官方 example 后：
```
full_audit:success = open opendir

smb_full_audit_connect: Invalid success operations list. Failing connect
```
日志直接说明了 full_audit 在初始化时解析 full_audit:success 列表失败，因此拒绝整个 Share 的连接。

说明：
- 不是 full_audit 模块坏了，而是写进去的 operation 名称，在你这版 Samba 4.22.10 根本不存在？！






