# 搭建 Samba 文件审计日志

**2026-08-01**

---

## 目的
让fss服务器就具备可审计性（Auditability），也为后续学习 SIEM、日志分析和事件响应打下良好基础。

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
就是用户做了什么

实现利用 Samba 的 VFS（Virtual File System）插件,流程如下：
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

### 把 Audit 日志写到 /srv/logs/samba
