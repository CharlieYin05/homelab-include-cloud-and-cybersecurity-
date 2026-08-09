# 安装VPN

**2026-08-09**

---

## 目标
- 让管理员和用户都能远程访问服务器

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

---

## Tailscale 运维常用命令
查看 tailnet 里的设备状态：
```
tailscale status
```

查看


