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

写入：
```
if ($programname == "smbd_audit" and $msg contains ".DS_Store") then stop
if ($programname == "smbd_audit" and $msg contains "/._") then stop

local7.notice    /srv/logs/samba/audit.log			← 路由规则
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

#### 遇到的问题1
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

---

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
---

2026/08/02 12:06pm

哪怕套用官方模版依然出错。剩下三大可能性：
1. Samba 4.22 + Debian 13 的 full_audit 回归 Bug
2. Debian 的 Samba 打包问题
3. 还有一个没验证的点：full_audit 是否依赖某些默认 VFS 模块（例如 acl_xattr）

---

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

---

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
- 不是 full_audit 模块坏了，而是写进去的 operation 名称，在这版 Samba 4.22.10 根本不存在？！

---

2026/08/02 12:38pm

TM的这个 `Samba` 安装时自带的官方文档有误 `vfs_full_audit.8.gz` 没有更新，而网上大量教程用的也是旧的 API 名称，这些旧的全部失效。最离谱的是查到官方写的 example: full_audit:success = open opendir 连开发者自己都承认这个 Example 是错的！

先给 `[private]` VFS 写一个最小配置，然后再一个一个添加并测试合法的 operation。

最小配置：
```
    vfs objects = full_audit

    full_audit:prefix = %u|%I
    full_audit:success = none
    full_audit:failure = none

    full_audit:facility = LOCAL7
    full_audit:priority = NOTICE
```
Private 测试通过。接下来就是直接看 Samba 源代码里的 full_audit operation。

---

2026/08/02 12:50pm

1.找到官方仓库，然后找这个文件：`source3/modules/vfs_full_audit.c`

2.根据日志报错，搜索 init_bitmap() 的函数的输入和输出：
```
static struct bitmap *init_bitmap(TALLOC_CTX *mem_ctx, const char **ops)
{
	struct bitmap *bm;

	if (ops == NULL) {
		DBG_ERR("init_bitmap, ops list is empty (logic error)\n");
		return NULL;
	}

	bm = bitmap_talloc(mem_ctx, SMB_VFS_OP_LAST);
	if (bm == NULL) {
		DBG_ERR("Could not alloc bitmap\n");
		return NULL;
	}
   //我遇到的出错并且被打印出来↓
	for (; *ops != NULL; ops += 1) {
		int i;
		bool neg = false;
		const char *op;

		if (strequal(*ops, "all")) {
			for (i=0; i<SMB_VFS_OP_LAST; i++) {
				bitmap_set(bm, i);
			}
			continue;
		}

		if (strequal(*ops, "none")) {
			break;
		}

		op = ops[0];
		if (op[0] == '!') {
			neg = true;
			op += 1;
		}

		for (i=0; i<SMB_VFS_OP_LAST; i++) {
			if ((vfs_op_names[i].name == NULL)
			 || (vfs_op_names[i].type != i)) {
				smb_panic("vfs_full_audit.c: name table not "
					  "in sync with vfs_op_type enums\n");
			}
			if (strequal(op, vfs_op_names[i].name)) {
				if (neg) {
					bitmap_clear(bm, i);
				} else {
					bitmap_set(bm, i);
				}
				break;
			}
		}
		if (i == SMB_VFS_OP_LAST) {
			DBG_ERR("Could not find opname %s\n", *ops);
			TALLOC_FREE(bm);
			return NULL;
		}
	}
	return bm;
   //我遇到的出错并且被打印出来↑
}
```
输入：`TALLOC_CTX *mem_ctx`, `const char **ops`

输出：`return bm` 表示成功。`NULL` 表示失败。

3. 报错的地方：for 循环判断 operation 在不在 `vfs_op_names[]` 里。

4. 找 `vfs_op_names[]` 里有的 operation 到底是啥：
```
static struct {
	vfs_op_type type;
	const char *name;
} vfs_op_names[] = {
	{ SMB_VFS_OP_CONNECT,	"connect" },
	{ SMB_VFS_OP_DISCONNECT,	"disconnect" },
	{ SMB_VFS_OP_OPEN_SHARE_ROOT,	"open_share_root" },
	{ SMB_VFS_OP_DISK_FREE,	"disk_free" },
	{ SMB_VFS_OP_GET_QUOTA,	"get_quota" },
	{ SMB_VFS_OP_SET_QUOTA,	"set_quota" },
	{ SMB_VFS_OP_GET_SHADOW_COPY_DATA,	"get_shadow_copy_data" },
	{ SMB_VFS_OP_STATVFS,	"statvfs" },
	{ SMB_VFS_OP_FSTATVFS,	"fstatvfs" },
	{ SMB_VFS_OP_FS_CAPABILITIES,	"fs_capabilities" },
	{ SMB_VFS_OP_GET_DFS_REFERRALS,	"get_dfs_referrals" },
	{ SMB_VFS_OP_CREATE_DFS_PATHAT,	"create_dfs_pathat" },
	{ SMB_VFS_OP_READ_DFS_PATHAT,	"read_dfs_pathat" },
	{ SMB_VFS_OP_FDOPENDIR,	"fdopendir" },
	{ SMB_VFS_OP_READDIR,	"readdir" },
	{ SMB_VFS_OP_REWINDDIR, "rewinddir" },
	{ SMB_VFS_OP_MKDIRAT,	"mkdirat" },							       // ← 我要记录的创建目录
	{ SMB_VFS_OP_CLOSEDIR,	"closedir" },
	{ SMB_VFS_OP_OPEN,	"open" },									       // ← 我要记录的打开文件/文件夹
	{ SMB_VFS_OP_OPENAT,	"openat" },
	{ SMB_VFS_OP_CREATE_FILE, "create_file" },						       // ← 我要记录的创建文件
	{ SMB_VFS_OP_CLOSE,	"close" },
	{ SMB_VFS_OP_READ,	"read" },
	{ SMB_VFS_OP_PREAD,	"pread" },
	{ SMB_VFS_OP_PREAD_SEND,	"pread_send" },
	{ SMB_VFS_OP_PREAD_RECV,	"pread_recv" },
	{ SMB_VFS_OP_WRITE,	"write" },
	{ SMB_VFS_OP_PWRITE,	"pwrite" },
	{ SMB_VFS_OP_PWRITE_SEND,	"pwrite_send" },
	{ SMB_VFS_OP_PWRITE_RECV,	"pwrite_recv" },
	{ SMB_VFS_OP_LSEEK,	"lseek" },
	{ SMB_VFS_OP_SENDFILE,	"sendfile" },
	{ SMB_VFS_OP_RECVFILE,  "recvfile" },
	{ SMB_VFS_OP_RENAMEAT,	"renameat" },							        // ← 我要记录的重命名文件/文件夹
	{ SMB_VFS_OP_RENAME_STREAM,	"rename_stream" },
	{ SMB_VFS_OP_FSYNC_SEND,	"fsync_send" },
	{ SMB_VFS_OP_FSYNC_RECV,	"fsync_recv" },
	{ SMB_VFS_OP_STAT,	"stat" },
	{ SMB_VFS_OP_FSTAT,	"fstat" },
	{ SMB_VFS_OP_LSTAT,	"lstat" },
	{ SMB_VFS_OP_FSTATAT,	"fstatat" },
	{ SMB_VFS_OP_GET_ALLOC_SIZE,	"get_alloc_size" },
	{ SMB_VFS_OP_UNLINKAT,	"unlinkat" },							        // ← 我要记录的删除文件/文件夹
	{ SMB_VFS_OP_FCHMOD,	"fchmod" },
	{ SMB_VFS_OP_FCHOWN,	"fchown" },
	{ SMB_VFS_OP_LCHOWN,	"lchown" },
	{ SMB_VFS_OP_CHDIR,	"chdir" },
	{ SMB_VFS_OP_NTIMES,	"ntimes" },
	{ SMB_VFS_OP_FNTIMES,	"fntimes" },
	{ SMB_VFS_OP_FTRUNCATE,	"ftruncate" },
	{ SMB_VFS_OP_FALLOCATE,"fallocate" },
	{ SMB_VFS_OP_LOCK,	"lock" },
	{ SMB_VFS_OP_FILESYSTEM_SHAREMODE,	"filesystem_sharemode" },
	{ SMB_VFS_OP_FCNTL,	"fcntl" },
	{ SMB_VFS_OP_LINUX_SETLEASE, "linux_setlease" },
	{ SMB_VFS_OP_GETLOCK,	"getlock" },
	{ SMB_VFS_OP_SYMLINKAT,	"symlinkat" },
	{ SMB_VFS_OP_READLINKAT,"readlinkat" },
	{ SMB_VFS_OP_LINKAT,	"linkat" },
	{ SMB_VFS_OP_MKNODAT,	"mknodat" },
	{ SMB_VFS_OP_REALPATH,	"realpath" },
	{ SMB_VFS_OP_FCHFLAGS,	"fchflags" },
	{ SMB_VFS_OP_FILE_ID_CREATE,	"file_id_create" },
	{ SMB_VFS_OP_FS_FILE_ID,	"fs_file_id" },
	{ SMB_VFS_OP_FSTREAMINFO,	"fstreaminfo" },
	{ SMB_VFS_OP_GET_REAL_FILENAME, "get_real_filename" },
	{ SMB_VFS_OP_GET_REAL_FILENAME_AT, "get_real_filename_at" },
	{ SMB_VFS_OP_BRL_LOCK_WINDOWS,  "brl_lock_windows" },
	{ SMB_VFS_OP_BRL_UNLOCK_WINDOWS, "brl_unlock_windows" },
	{ SMB_VFS_OP_STRICT_LOCK_CHECK, "strict_lock_check" },
	{ SMB_VFS_OP_TRANSLATE_NAME,	"translate_name" },
	{ SMB_VFS_OP_PARENT_PATHNAME,	"parent_pathname" },
	{ SMB_VFS_OP_FSCTL,		"fsctl" },
	{ SMB_VFS_OP_OFFLOAD_READ_SEND,	"offload_read_send" },
	{ SMB_VFS_OP_OFFLOAD_READ_RECV,	"offload_read_recv" },
	{ SMB_VFS_OP_OFFLOAD_WRITE_SEND,	"offload_write_send" },
	{ SMB_VFS_OP_OFFLOAD_WRITE_RECV,	"offload_write_recv" },
	{ SMB_VFS_OP_FGET_COMPRESSION,	"fget_compression" },
	{ SMB_VFS_OP_SET_COMPRESSION,	"set_compression" },
	{ SMB_VFS_OP_SNAP_CHECK_PATH, "snap_check_path" },
	{ SMB_VFS_OP_SNAP_CREATE, "snap_create" },
	{ SMB_VFS_OP_SNAP_DELETE, "snap_delete" },
	{ SMB_VFS_OP_GET_DOS_ATTRIBUTES_SEND, "get_dos_attributes_send" },
	{ SMB_VFS_OP_GET_DOS_ATTRIBUTES_RECV, "get_dos_attributes_recv" },
	{ SMB_VFS_OP_FGET_DOS_ATTRIBUTES, "fget_dos_attributes" },
	{ SMB_VFS_OP_FSET_DOS_ATTRIBUTES, "fset_dos_attributes" },
	{ SMB_VFS_OP_FGET_NT_ACL,	"fget_nt_acl" },
	{ SMB_VFS_OP_FSET_NT_ACL,	"fset_nt_acl" },
	{ SMB_VFS_OP_SYS_ACL_GET_FD,	"sys_acl_get_fd" },
	{ SMB_VFS_OP_SYS_ACL_BLOB_GET_FD,	"sys_acl_blob_get_fd" },
	{ SMB_VFS_OP_SYS_ACL_SET_FD,	"sys_acl_set_fd" },
	{ SMB_VFS_OP_SYS_ACL_DELETE_DEF_FD,	"sys_acl_delete_def_fd" },
	{ SMB_VFS_OP_GETXATTRAT_SEND, "getxattrat_send" },
	{ SMB_VFS_OP_GETXATTRAT_RECV, "getxattrat_recv" },
	{ SMB_VFS_OP_FGETXATTR,	"fgetxattr" },
	{ SMB_VFS_OP_FLISTXATTR,	"flistxattr" },
	{ SMB_VFS_OP_REMOVEXATTR,	"removexattr" },
	{ SMB_VFS_OP_FREMOVEXATTR,	"fremovexattr" },
	{ SMB_VFS_OP_FSETXATTR,	"fsetxattr" },
	{ SMB_VFS_OP_AIO_FORCE, "aio_force" },
	{ SMB_VFS_OP_IS_OFFLINE, "is_offline" },
	{ SMB_VFS_OP_SET_OFFLINE, "set_offline" },
	{ SMB_VFS_OP_DURABLE_COOKIE, "durable_cookie" },
	{ SMB_VFS_OP_DURABLE_DISCONNECT, "durable_disconnect" },
	{ SMB_VFS_OP_DURABLE_RECONNECT, "durable_reconnect" },
	{ SMB_VFS_OP_FREADDIR_ATTR,      "freaddir_attr" },
	{ SMB_VFS_OP_LAST, NULL }
};
```

#### 遇到的问题2

##### 症状
打开文件后没有产生任何 full_audit 的日志。

--- 

2026-08-02 2:07pm

整理下思绪：

| 环节                              | 状态 |
| ------------------------------- | -- |
| full_audit.so 存在                | ✅  |
| `vfs objects = full_audit` 生效   | ✅  |
| `init_bitmap()` 正常              | ✅  |
| operation 名称来自 `vfs_op_names[]` | ✅  |
| `mkdir` → `mkdirat` 已验证         | ✅  |
| `open` 是合法 operation            | ✅  |
| rsyslog 正常                      | ✅  |
| LOCAL7 正常                       | ✅  |
| audit.log 可写                    | ✅  |
| Share 能正常打开                     | ✅  |
| **没有 audit 日志**                 | ❌  |


怀疑：
1. do_log() 根本没有被执行。
2. do_log() 执行了，但 log_success() 把它过滤掉了。


还要接着看源代码定位。。。好累，好饿，先吃饭去。

--- 

2026-08-02 3:36pm

目前证据链如下：
```
full_audit.so          ✅ 存在
        │
        ▼
vfs objects            ✅ 已加载
        │
        ▼
init_bitmap()          ✅ 正常
        │
        ▼
operation 名称         ✅ 已验证 (mkdir→mkdirat)
        │
        ▼
do_log()               ❓
        │
        ▼
syslog()               （理论上会调用）
        │
        ▼
LOCAL7                 ✅ 正常
        │
        ▼
rsyslog                ✅ 正常
        │
        ▼
audit.log              ✅ 正常
```

根据源代码已知：
```
            do_log()
               │
     ┌─────────┴─────────┐
     │                   │
syslog=true         syslog=false
     │                   │
 syslog()            DEBUG(1)
     │                   │
 rsyslog          /var/log/samba/log.*
```
因此可以故意触发 `syslog=false` 来把问题进一步缩小。

把 `[private]` 部分的 syslog ，从 true 改成false, 同时加入 sucess 改成 connect openat create_file，然后再观察。

服务端：
```
sudo tail -f /var/log/samba/log.smbd
```

客户端：
进入 private，ls，然后quit

根据输出的日志只输出了普通 Samba Debug。说明 不是 syslog() 的问题。根据源代码:
```
if (pd->do_syslog) {
    syslog(...);
} else {
    DEBUG(1, (...));
}
```
直接触发了 else 而不是 syslog()，也就是说那么多的 do_log() 函数根本没有被调用！！！（我要昏厥了）

---

2026-08-02 4:06pm

也许 do_log() 没有被执行，直接找 Samba Debian 的源码：
```
sudo apt update

sudo apt install dpkg-dev

apt source samba
```

然后查看看 Debian 版本的 `vfs_full_audit.c`


缩小目标：为什么就单单连接这个行为 smb_full_audit_connect() 没有产生 do_log() 的输出？为什么这个函数没有被 Samba 注册成 VFS Hook?

根据static struct vfs_fn_pointers vfs_full_audit_fns的结构体内容 `.connect_fn = smb_full_audit_connect,` 能确定 VFS Hook 注册是正常的。

调用链：
```
smb.conf
    │
    ▼
vfs objects = full_audit
    │
    ▼
vfs_full_audit_init()
    │
    ▼
smb_register_vfs(...)
    │
    ▼
vfs_full_audit_fns
    │
    ├── connect_fn  -> smb_full_audit_connect
    ├── openat_fn   -> smb_full_audit_openat
    ├── create_file_fn -> smb_full_audit_create_file
    └── ...
```

---

2026-08-02 4:49pm

问题再缩小：为什么 Hook 被注册了，却没有产生任何 audit log？

首先是 Debian 4.22 的 vfs_full_audit.c 和 Samba github 官方仓库的源代码不一样。官方仓库的函数是 do_log(op, NULL, ...)，Debian 4.22 源码函数是 do_log(op, true, ...)。。。。。。但是不重要

---
2026-08-02 5:16pm

好烦，会不会是一开始路线就错了，例如不应该一直盯着audit.log作为唯一的日志输出对象来观察。直接暴力搜索所有带有“ok”和“connect”的log

输入：
```
sudo grep -R "ok|" /var/log
sudo grep -R "connect" /var/log
```

输出发现：
1. vfs_full_audit 工作正常，例如这一段：
   ```
   /var/log/samba/log.cy-server-fss.old:
   cyin026|::1|connect|ok|private

   cyin026|::1|openat|ok|r|/srv/storage/shares/private

   cyin026|::1|create_file|ok|0x81|dir|open|/srv/storage/shares/private
   ```
   这完全就是full_audit 的输出格式。

   而 dolog() 函数
   ```
   syslog(priority,
       "%s|%s|%s|%s\n",
       audit_pre,
       audit_opname(op),
       err_msg,
       op_msg);
   ```
   对应的输出也完全匹配：
   ```
   cyin026|::1|connect|ok|private
   ```

2. 一直找错日志了
   日志没有没有进入 `/srv/logs/samba/audit.log`，而是进入了 `/var/log/samba/log.cy-server-fss.old`

WTF? 为什么 full_audit 没有走 rsyslog，而是被 Samba 的 logging backend 接管了?

---

2026-08-02 5:29pm

SB了，之前为了做测试 `[private]` 一直是 full_audit:syslog = false。根据源代码：
```
if (pd->do_syslog) {
    syslog(...);
} else {
    DEBUG(1, ("%s|%s|%s|%s\n", ...));
}
```
如果 false 就会：
```
full_audit:syslog = false
        │
        ▼
pd->do_syslog = false
        │
        ▼
不会调用 syslog()
        │
        ▼
调用 DEBUG(1,...)
```
所以没有经过 syslog() 日志存在 `/var/log/samba/log.cy-server-fss.old`.......

重新改回 true 再测试发现 audit.log 终于运行了。

---

## 4.设置日志轮转

### 目标：使用 `logrotate` 保留12个月的记录

#### 4.1 针对 /srv/logs/samba/audit.log 新增一个规则
编辑 logrotate 配置文件：
```
sudo nano /etc/logrotate.d/samba-audit
```

照着 Debian 自带的格式写
```
/srv/logs/samba/audit.log
{
    rotate 12									← 保留 12
    monthly										← 个月

    missingok									← 日志不存在也不报错
    notifempty									← 如果日志格式为空则不论转

    compress									← 历史日志全部压缩
    delaycompress								← 最新的一份历史日志先别压缩，下次再压

    sharedscripts								← 共享 postrotate 次数

    postrotate									← 日志轮转完成后，再执行下面这些命令。
        /usr/lib/rsyslog/rsyslog-rotate			← 用于秩序吸入 audit.log ，而不是指向旧的 audit.log.1 
    endscript
}
```

## 日志生命周期
- Samba：产生日志（Producer）
- rsyslog：收集并写入日志（Collector）
- logrotate：管理日志生命周期（Rotation）


## VFS操作文档（扒源码版本）

| VFS Operation   | 含义                          | 用户实际操作                                   |
| --------------- | --------------------------- | ---------------------------------------- |
| **connect**     | 客户端连接到 Share                | 打开 `\\server\private`、手机进入 `private` 文件夹 |
| **disconnect**  | 客户端断开 Share                 | 关闭 SMB、退出文件管理器、网络断开                      |
| **openat**      | 打开一个文件或目录（Linux `openat()`） | 浏览目录、双击图片、打开 PDF、查看文件属性                  |
| **create_file** | SMB CREATE 请求（打开或创建文件/目录）   | 创建文件、打开文件、创建目录，甚至有些客户端浏览目录都会触发           |
| **mkdirat**     | 创建目录                        | 新建文件夹                                    |
| **renameat**    | 重命名或移动                      | 改文件名、移动文件到另一个目录                          |
| **unlinkat**    | 删除目录项                       | 删除文件（以及部分删除目录操作）                         |


## 开源项目排障方法：
1. 日志定位函数
2. 阅读函数，而不是通读整个项目（找调用链）
3. 验证假设
4. 排除一个又一个可能性
5. 让证据决定下一步，而不是凭感觉改配置

---

## 一般的大型 C 语言开源项目结构如下：
```
配置文件
     │
     ▼
Parser（lp_parm_*）
     │
     ▼
Module Init（注册）
     │
     ▼
Function Pointer Table（Hook）
     │
     ▼
真正的函数（smb_full_audit_openat）
     │
     ▼
日志
```

---

## Samba 配置文件操作手册

### 只查看配置文件：
```
cat /etc/samba/smb.conf
```

### 修改配置文件：
```
sudo nano /etc/samba/smb.conf
```

### 修改后检查重启
```
sudo testparm
sudo systemctl restart smbd
```

---

## Samba `audit.log` 操作手册

### 查看全部记录
```
sudo cat /srv/logs/samba/audit.log
```

### 查看最近20行
```
sudo tail -20 /srv/logs/samba/audit.log
```

### 查询某个用户
```
sudo grep "用户名" /srv/logs/samba/audit.log
```

### 查询某个IP
```
sudo grep "IP地址" /srv/logs/samba/audit.log
```

### 查询失败操作
```
sudo grep "fail" /srv/logs/samba/audit.log
```

### 只查看删除操作
```
sudo grep "unlinkat" /srv/logs/samba/audit.log
```

### 实时监控
```
sudo tail -f /srv/logs/samba/audit.log
```

---

## Samba `rsyslog.d` 操作手册
通常用于添加过滤条件

编辑：
```
sudo nano /etc/rsyslog.d/30-samba-audit.conf
```
重启：
```
sudo systemctl restart rsyslog
```

---

## logrotate 配置文件操作手册
查看已有的 logrotate 规则
```
ls /etc/logrotate.d/
```

查看已生成的 logrotate samba.log 有些哪：
```
ls -lh /srv/logs/samba
```

查看 rsyslog：
```
cat /etc/logrotate.d/rsyslog
```

编辑具体 Samba log：
```
sudo nano /etc/logrotate.d/samba-audit
```

查看已经生成的 audit.log ：
```
sudo cat /etc/logrotate.d/samba-audit
```




