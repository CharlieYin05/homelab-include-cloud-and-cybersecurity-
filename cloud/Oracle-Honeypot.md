# First Hands-on with Oracle Cloud (OCI): SSH Lockout & Network Misconfiguration Case

## Objective

My first hands-on experience with a cloud provider — **Oracle Cloud Infrastructure (OCI)**.
The goal was to deploy a server for cybersecurity experimentation, specifically:

* Deploy a **honeypot (HFish)** exposed to the internet
* Open common attack surface ports:

  * `21` (FTP)
  * `23` (Telnet)
  * `80` (HTTP)
  * `4433` (web)
* Harden access by:

  * Changing SSH from default port `22` → custom port
  * Restricting access via firewall rules

---
## 云环境的“三道锁”逻辑

  - 最外层：虚拟网络锁 (VCN Security List)
    这是OCI在我的服务器外面套的一个大笼子。我在系统里改任何东西，如果不先在控制台给笼子开个洞（放行 441），流量根本摸不到我的服务器。

  - 中间层：系统内核锁 (iptables/ufw)
    这是OCI镜像最TM坑爹的地方。哪怕用了 ufw，底层其实还有一套OCI预设的 iptables 规则（这货代码在ufw之上）。当运行 iptables -F 时，会删除了所有的“通行证”，还会持久化！由于默认策略 (Policy) 是 DROP，它就像一个关了门但把钥匙丢了的保安，谁也不让进。

  - 最内层：应用锁 (SSH Config)
    这是我修改SSH端口的地方。
    
---
## 经验教训
  1. 留后手
     * 确保Console connection能连上，这是最后的救命稻草
     * Ubuntu系统要设置个密码，不能只用Private Key登陆
     * SSH没有稳定前不要动端口22

  2. 对付“三道锁”
     - Instance:
       * OCI Instance检查Gateway是否开启0.0.0.0/0（
       * OCI Instance检查Security Group是否有规则凌驾于
       * Gateway对0.0.0.0/0（任何端口）开放
   - Virtual cloud networks:
     * 找到Instance所在的subnet
     * 出站规则

   - 
       
---
## What I Did

### 1. Infrastructure Setup

* Created a Compute instance (Ubuntu 24.04 ARM)
* Configured:

  * VCN (Virtual Cloud Network)
  * Public Subnet
  * Internet Gateway
* Assigned a reserved public IPv4 address

---

### 2. SSH Access Configuration

* Generated SSH key pair locally (Mac)
* Connected successfully via:

```bash
ssh ubuntu@<public-ip> -i ~/.ssh/<key>
```

* Moved key into `~/.ssh/` and set permissions:

```bash
chmod 600 ~/.ssh/<key>
```

* Configured SSH alias via `~/.ssh/config`

---

### 3. Firewall Setup (Multiple Layers)

#### A. UFW (on instance)

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 21
sudo ufw allow 23
sudo ufw allow 4433
sudo ufw enable
```

#### B. OCI Security List (cloud firewall)

Added ingress rules:

* Allow TCP from `0.0.0.0/0` to:

  * 22
  * 21
  * 23
  * 80
  * 4433

---

### 4. SSH Port Hardening

Edited:

```bash
/etc/ssh/sshd_config
```

Changed:

```text
Port 441
```

Restarted service:

```bash
sudo systemctl restart ssh
```

Also updated OCI Security List to allow port `441`.

---

## 💣 Critical Mistake (Root Cause)

After modifying SSH + firewall, I ran:

```bash
sudo iptables -F
```

### ❗ What I *thought*:

* “Flush rules → open everything”

### ❗ What actually happened:

* Cleared connection tracking / firewall state
* Broke existing SSH session
* Disrupted network stack behavior in combination with:

  * UFW
  * OCI Security List

---

## Symptoms

* SSH (port 441): `Operation timed out`
* SSH (port 22): `Connection refused`
* Netcat test: timeout
* Could not connect from:

  * Local machine
  * Mobile network
  * OCI Cloud Shell (private IP also failed)

---

## 🔒 Lockout State

Final situation:

| Access Method      | Status                |
| ------------------ | --------------------- |
| Public SSH         | ❌                     |
| Private SSH        | ❌                     |
| Cloud Shell        | ❌                     |
| Console Connection | ❌ (no IAM permission) |

➡️ Instance became **completely inaccessible**

---

## Key Learnings

### 1. Cloud Networking is Layered

There are **multiple independent firewalls**:

* OCI Security List (cloud-level)
* UFW (OS-level)
* iptables (kernel-level)

Misconfiguration across layers can cause total lockout.

---

### 2. Never Change SSH Access Blindly

Correct order should be:

```text
1. Open new port in cloud firewall
2. Modify sshd_config
3. Restart SSH
4. TEST new port (keep old one open)
5. Only then disable old port
```

---

### 3. Avoid Mixing Firewall Systems

Using all three simultaneously:

* UFW
* iptables
* OCI Security List

Leads to unpredictable behavior

**Best practice:**

* Use OCI Security List as primary control
* Minimize OS-level firewall complexity

---

### 4. iptables is NOT a “reset button”

```bash
iptables -F
```

does NOT mean:

> “allow everything safely”

It can:

* break active connections
* disrupt NAT/state tracking
* interact badly with higher-level tools (UFW)

---

### 5. Cloud Console Access is Critical

* OCI Console Connection requires IAM permissions
* Without it → no recovery path

👉 Always ensure:

* Console access is available **before risky changes**

---

### 6. nano is suck for editing code

* Becareful with the 4 space and Tab, even looks the same
* Beaware with the indentation
* Use ctrl + w for searching
* Make monitor vertical if possible

---

## Outcome

* Instance became unrecoverable
* Had to terminate it
* Lost initial deployment

However:

> This became a full real-world lesson in **cloud networking failure modes**

---

## What I Would Do Next Time

* Use **only one firewall layer initially**
* Keep SSH on default port until everything is stable
* Enable **console access before changes**
* Test incrementally (never multiple changes at once)
* Separate:
  * management access (SSH)
  * attack surface (honeypot ports)

---

## Final Reflection

This was not just a setup exercise —
it was my first encounter with:

* real cloud networking
* multi-layer security models
* failure recovery limitations

> “Breaking things in the cloud is easy.
> Recovering them requires planning.”

---

## Tags

`OCI` `Cloud` `Networking` `Security` `SSH` `Honeypot` `Linux` `DevOps` `Beginner-Lessons`
