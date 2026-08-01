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
在 global 底下加入：
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









