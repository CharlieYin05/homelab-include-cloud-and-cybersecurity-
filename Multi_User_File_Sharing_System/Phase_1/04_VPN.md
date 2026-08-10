# 安装 VPN

**2026-08-09**

---

## 目标
- 让管理员和用户都能远程访问 FSS 服务器
- 重塑整个已有的 Tailnet 规则，让它支持多人使用并增加安全性

--- 
## 架构
```
                         ┌─────────────────────┐
                         │   cy-server-fss     │
                         │        FSS          │
                         │     tag:fss         │
                         │  192.168.XX.XX      │
                         │  100.105.xxx.xxx    │
                         └─────────┬───────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                 TCP/445        TCP/22         TCP/22
                   SMB            SSH            SSH
                    │              │              │
                    │              │              │
          ┌─────────▼──────┐ ┌────▼───────┐ ┌────▼─────────────┐
          │ group:file-user│ │ group:admin│ │     cy-server    │
          │   普通文件用户   │ │    管理员   │ │   网络中枢服务器   │
          └─────────┬──────┘ └────┬───────┘ │ 192.168.XX.XX    │
                    │              │        │ 100.65.xxx.xxx   │
                    │              │        │ Subnet Router    │
                    │              │        └────────┬─────────┘
                    │              │                  │
                    └──────────────┼──────────────────┘
                                   │
                         ┌─────────▼─────────┐
                         │     Tailnet       │
                         │ Tailscale Grants │
                         │  Least Privilege │
                         └─────────┬─────────┘
                                   │
             ┌─────────────────────┼──────────────────────┐
             │                     │                      │
             ▼                     ▼                      ▼
       tag:fss              tag:network-hub        192.168.50.0/24
       FSS 服务               中枢服务器服务              家庭 LAN
             │                     │                      │
       ┌─────┴─────┐         ┌─────┴──────┐         ┌─────┴──────┐
       │           │         │            │         │            │
     :445        :22       SSH :22    本机运行服务   Router       其他设备
     SMB         SSH                             192.168.50.1
       │           │                                  │
file-user ✅   file-user ❌                      file-user ❌
admin     ✅   admin     ✅                      admin     ✅
```

## 权限
```
访问关系：

group:file-user
    └──→ tag:fss
          └── TCP/445 SMB                 ✅
    └──→ FSS SSH / cy-server / Home LAN   ❌


group:admin
    ├──→ tag:fss
    │     ├── TCP/445 SMB                 ✅
    │     └── TCP/22  SSH                 ✅
    │
    ├──→ tag:network-hub
    │     ├── TCP/22   SSH                ✅
    │     └── TCP/8787 ClipCascade        ✅
    │
    └──→ 192.168.50.1
          ├── TCP/80       Router Web     ✅
          └── TCP/UDP 53   DNS            ✅


cy-server
    └──→ FSS TCP/22 SSH                   ✅


其他未明确授权的 Tailnet 流量
    └──→ DROP / DENY                      ❌
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

#### 4.5 给 `cy-server` 单独创建 tag
#### 4.6 给 `cy-server` 单独创建写规则

#### 4.7 邀请朋友以 `file-user` 身份加入我的 tailnet
#### 4.8 再手动把朋友加入 `file-user` group
#### 4.9 朋友测试服务：
```
# SMB（应该成功）
nc -vz 100.105.XXX.XXX 445

# SSH（以下应该超时）
nc -vz -w 5 100.105.XXX.XXX 22

# LAN 内部服务（以下应该超时）
nc -vz -w 5 100.65.XXX.XXX 22
nc -vz -w 5 100.65.XXX.XXX 8787
nc -vz -w 5 100.65.XXX.XXX 8000
nc -vz -w 5 100.65.XXX.XXX 4430
```
---
## 遇到的问题1

### 症状：
在 Mac 开启 Tailscale 后出现：
- Tailscale 内部设备仍可访问。
- SMB 等 Tailscale 服务正常。
- 普通互联网网站大部分无法打开。
- 关闭 Tailscale 后互联网立即恢复。
- Tailscale GUI 提示：
```
DNS Unavailable

Tailscale can't reach the configured DNS servers.
Code: dns-forward-failing
```

最初表现很像“开启 Tailscale 后断网”，但后续确认实际上是 DNS 解析故障，而非 Internet 网络中断。

### 解决过程
#### 1.基础网络检查
Tailscale 开启状态下：
```
ping -c 3 192.168.50.1
```
成功：
```
3 packets transmitted, 3 packets received
```
说明 Mac → 家庭路由器正常。

然后测试公网 IP：
```
ping -c 3 1.1.1.1
```
同样成功，但是 `nslookup google.com` 失败......

初步诊断：DNS 故障。

#### 2.检查 Tailscale DNS
检查 DNS：
```
tailscale dns status
```
输出：
```
Tailscale DNS: enabled.
```
DNS 配置如下：
```
MagicDNS: enabled

Resolvers:
(no resolvers configured, system default will be used)

Split DNS Routes:
cy-server.com -> 192.168.50.1         ← 我的内网服务

System DNS:
192.168.50.1                          ← 普通公网 DNS 最终也需要通过路由器解析
```
测试 Tailscale internal resolver：
```
tailscale dns query google.com
```
输出失败：
```
failed to query DNS:
waiting for response or error from [192.168.50.1]:
context deadline exceeded
```

初步判断故障路径：
```
Application
    ↓
macOS DNS
    ↓
Tailscale DNS
    ↓
192.168.50.1
    ↓
TIMEOUT
```

### 3.验证路由器 DNS 是否正常
直接绕过 Tailscale DNS：
```
dig @192.168.50.1 google.com

dig +tcp @192.168.50.1 google.com
```
成功！说明 192.168.50.1 的 DNS 服务本身没有问题。进一步缩小问题在 Tailscale DNS

### 4.临时关闭 Tailscale DNS
关闭 DNS：
```
tailscale set --accept-dns=false
```
随后一系列外网测试都正常：
```
dscacheutil -q host -a name google.com

ping google.com
```
而且 Tailscale 服务也仍然正常（nc -vz 100.105.169.81 445 → Connection ... port 445 succeeded!）

### 5.排查 Tailscale 客户端版本
发现 Terminal 使用：
```
which tailscale
```
输出：
```
/opt/homebrew/bin/tailscale（这是 Homebrew 安装的 tailscale 1.92.3）
```
同时 Mac 还有：
```
/Applications/Tailscale.app
```

这意味着可能用 tailscale 命令时有冲突（它们 backend 来自不同构建）
```
CLI:
1.92.3-t9a08e8f1c

Backend:
1.92.3-ta17f36b9b-ga4dc88aac
```

卸载 brew 版本并且升级 app 版本的 tailscale 后依然 `tailscale dns query google.com` 失败，说明CLI/backend 版本混用确实是一个配置问题，但不是此次 DNS 故障的根因。

### 6.发现 Subnet Route 出问题
cy-server 是 Tailscale Subnet Router，并广播 `192.168.50.0/24`

而 Mac 当前就在家庭 LAN 存在重叠：
```
Local LAN
192.168.50.0/24
       ↑
       │ overlap
       ↓
Tailscale advertised route
192.168.50.0/24
```
进行 Subrouter A/B 测试。

#### 关闭 subnet route 接收后：
```
tailscale set --accept-dns=true

tailscale set --accept-routes=false
```
再次 `tailscale dns query google.com` 立刻成功。这直接锁定故障与 Tailscale 接受 192.168.50.0/24 Subnet Route 有直接关系。

### 7. 怀疑 Tailscale AC 导致的断网
回顾一下之前一直用的好好的，直到刚刚实施 Zero Trust，删除了 Allow All，改成精确 Grants。后才出现故障合理怀疑漏了一个基础设施依赖：
```
Router DNS
192.168.50.1:53
```

再次 A/B 测试

开启 subnet route 接收后：
```
tailscale set --accept-routes=true
tailscale set --accept-dns=true
```
- A 测试： `tailscale dns query google.com` 卡住
- B 测试： 给 Admin 增加一条出站规则：
  ```
  {
    "src": ["group:admin"],
    "dst": ["192.168.50.1"],
    "ip": [
        "udp:53",
        "tcp:53"
    ]
  }
  ```
  后 `tailscale dns query google.com` 立刻成功，并且网站访问正常。

### 8.根因
```
Mac
 │
 │ accept-dns=true
 │ accept-routes=true
 ▼
Tailscale DNS Forwarder
 │
 │ DNS resolver = 192.168.50.1
 ▼
Tailscale Subnet Route
192.168.50.0/24
 │
 ▼
Tailnet Access Control
 │
 │ 缺少 UDP/TCP 53 Grant
 ▼
DNS 被拒绝
```

而以前不存在问题，是因为旧策略存在：
{
    "src": ["*"],
    "dst": ["*"],
    "ip": ["*"]
}

删除 Allow All、实施最小权限后，只考虑了显式使用的：
```
Router HTTP :80
```
遗漏了：
```
Router DNS UDP :53
Router DNS TCP :53
```
因此 Tailscale DNS Forwarder 无法访问 DNS Server。

## 9.教训与经验
从 Allow All 转向 Least Privilege 时，不仅要盘点用户主动访问的应用服务，还必须盘点这些服务背后的基础设施依赖。
例如这一次：
```
用户显式使用：
├── SMB 445
├── SSH 22
├── ClipCascade 8787
└── Router HTTP 80

容易遗漏的基础设施：
├── DNS 53       ← 本次故障
├── DHCP
├── NTP
├── ICMP
└── 其他内部解析/认证服务
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

### 访问 FSS 路径

#### 管理员
- SSH：用户设备 → Tailscale magicDNS → FSS
- SSH：用户设备 → 路由器 DNS 解析 → FSS（经过 192.168.50.1:53）
- SSH：用户设备 → cy-server → FSS
- SMB：用户设备 → Tailscale magicDNS → FSS
- SMB：用户设备 → 路由器 DNS 解析 → FSS

#### file-user
- SMB：用户设备 → Tailscale magicDNS → FSS
- SMB：用户设备 → 路由器 DNS 解析 → FSS（经过 192.168.50.1:53）
---

## Tailscale 运维常用命令
查看 tailnet 里的设备状态：
```
tailscale status
```
测试延迟和查看是否中继：
```
tailscale ping <对方IP>
```

---
## 注意事项
1. 新用户加入记得手动拉入 Definitions user-group。
2. 有什么特殊需要单独写出入站规则，注意规则是单项的。

