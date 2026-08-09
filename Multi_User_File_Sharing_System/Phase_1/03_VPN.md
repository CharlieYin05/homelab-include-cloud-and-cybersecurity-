# 安装VPN

**2026-08-09**

---

## 目标
- 让管理员和用户都能远程访问 FSS 服务器
- 重塑整个已有的 Tailnet 规则，让它支持多人使用并增加安全性

--- 

## 架构

```
                         FSS
                  ┌───────┴────────┐
                  │                │
              SMB :445          SSH :22
                  │                │
        ┌─────────┴──────┐         │
        │                │         │
      LAN            Tailnet     中枢服务器
192.168.50.0/24     授权设备    192.168.XX.XX
        │                │         │
      ACCEPT           ACCEPT    ACCEPT
```

---

## 过程

### 1.添加 Tailscale 软件源
输入：
```
sudo mkdir -p --mode=0755 /usr/share/keyrings
```
下载官方仓库签名密钥：
```
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg \
  | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
```
添加 Trixie stable repository：
```
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list \
  | sudo tee /etc/apt/sources.list.d/tailscale.list
```

### 2.安装 Tailscale
输入：
```
sudo apt update
sudo apt install tailscale
```
查看网络接口里 tailscale 是否识别：
```
ip -br addr
```
查看 tailscale status：
```
systemctl status tailscaled --no-pager
```

### 3.让 FSS 加入 Tailnet
登录：
```
sudo tailscale up
```

### 4.设置 Tailnet 规则（Zero Trust）
#### 4.1 创建 Admin Group
#### 4.2 给fss服务器建立机器身份
Create tag:
```
tag:fss
```
Tag owner:
```
group:admin
```
#### 4.3 给 `cy-server-fss` 赋予 `tag:fss`
- 然后查看它的拥有者是否从我改成tag：
```
tailscale status
```
- 本机设备 ping 它的 tailscale ip 看看是否通（应该通）
- 通过 tailscale IP SSH 去服务器（理论上可以因为规则还是默认 allow-all Grant）

#### 4.4 重复步骤
- 创建 `file-user` group

- 创建规则：
```
// Administrators can access SSH and SMB on FSS.
{
    "src": ["group:admin"],
    "dst": ["tag:fss"],
    "ip":  ["tcp:22", "tcp:445"]
},

// File users can only access SMB on FSS.
{
    "src": ["group:file-user"],
    "dst": ["tag:fss"],
    "ip":  ["tcp:445"]
}
```

---

## Tailnet 规则

### Tailnet 身份模型
```
用户身份
    │
    ├── 管理员（我）
    └── 普通文件共享用户

设备身份
    │
    ├── 普通个人设备
    ├── FSS
    └── 网络中枢服务器 / Subnet Router
```

### 规则表
| Source        | Destination              | Service       | Result |
| ------------- | ------------------------ | ------------- | -----: |
| 普通 Tailnet 用户 | FSS                      | SMB TCP/445   |      ✅ |
| 普通 Tailnet 用户 | FSS                      | SSH TCP/22    |      ❌ |
| 普通 Tailnet 用户 | `cy-server`              | SSH/Web/Admin |      ❌ |
| 普通 Tailnet 用户 | 家庭 LAN `192.168.50.0/24` | 任意            |      ❌ |
| 管理员设备         | FSS                      | SMB           |      ✅ |
| 管理员设备         | FSS                      | SSH           |      ✅ |
| `cy-server`   | FSS                      | SSH           |      ✅ |
| 其他未明确授权流量     | —                        | —             |      ❌ |

### 双管理路径
```
主路径：
管理员 → cy-server → SSH → FSS

备用路径：
管理员设备 → Tailscale → SSH → FSS
```

### 三层授权路径（defense in depth）
```
第一层：Tailscale
────────────────────────
这个人能不能连接 FSS:445？
这个人能不能连接 FSS:22？

          ↓ 允许

第二层：Samba
────────────────────────
这个 SMB 用户能登录吗？
能访问 public / restriction / private 哪个 share？

          ↓ 允许

第三层：Linux filesystem ACL
────────────────────────
这个 Unix 身份最终对文件
有 r / w / x 中的哪些权限？
```


---

## Tailscale 运维常用命令
查看 tailnet 里的设备状态：
```
tailscale status
```



