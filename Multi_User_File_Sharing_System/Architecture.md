# Architecture

## Design Goals
The system is designed around five principles:

1. **No** direct SMB or SSH **exposure** to the **public Internet**.
2. Joining the Tailnet does not imply unrestricted server access.
3. Network access and file access are **authorised independently**.
4. Every user has an **individual identity**.
5. All file operations should be **auditable**.

## System Architecture
#### Two Network Paths to FSS
```
                                     cy-server-fss
                              +-------------------------+
                              |   /srv/storage/shares   |
                              |          ↑              |
                              |      POSIX ACL          |
                              |          ↑              |
                              | Samba Authentication    |
                              |          ↑              |
                              |       nftables          |
                              +------+--------+---------+
                                     ^        ^
                                     |        |
                    tailscale0       |        |       LAN interface
                    100.x.x.x        |        |       192.168.50.x
                                     |        |
        =============================+        +=============================
              TAILNET PATH                         RESTRICTED LAN ADMIN PATH
        =============================         ==============================
```

#### Access Paths by Device
```

                            Tailnet Devices                          Home LAN Devices
                                   |                                        |
                          |--------+---------|                    |---------+---------|
                          |                  |                    |                   |
                          v                  v                    v                   v
                   +-------------+    +-------------+      +-------------+    +-------------+
                   | User Device |    | Admin Device|      |  cy-server  |    | Other LAN   |
                   +------+------+    +------+------+      +------+------+    | Devices     |
                          |                  |                    |           +------+------+
                          |                  |                    |                  |
                      SMB :445          SMB :445                  |                  |
                          |               SSH :22                 |                  |
                          |                  |                  SSH :22              |
                          +--------+---------+                    |                  |
                                   |                              |                  X
                                   v                              |                 DROP
                            +-------------+                       | 
                            |  Tailscale  |                       |
                            |   Grants    |                       |
                            +------+------+                       |
                                   |                              |
                                   v                              |
                            +-------------+                       |
                            |  Tailnet    |                       |
                            +------+------+                       |
                                   |                              |
 cy-server-fss                     |            cy-server-fss     |
    port:445<----(admin & user)----|---(admin)---->port:22 <------|

```

#### Internal DNS Resolution

`fss.cy-server.com` provides a human-readable name for the FSS Tailnet address.

```text
Tailnet Device
      |
      | query: fss.cy-server.com
      v
Tailscale Split DNS
      |
      | *.cy-server.com
      v
Merlin Router
    dnsmasq
      |
      | resolves to FSS Tailscale IP
      v
  100.x.x.x
```

The router participates only in DNS resolution.

After resolution, SMB/SSH traffic travels directly between the client and FSS through the Tailnet.

## Identity & Permission Model
| Role | SMB | SSH | File Access |
|---|---:|---:|---|
| Admin | Yes | Yes | Administrative / authorised access |
| File User | Yes | No | Defined by POSIX ACL |
| cy-server | No | Yes via LAN | Server administration |
| Other LAN Device | No | No | None |

```text
Network Identity
└── Tailscale user / group
    └── Controls network reachability

File Identity
└── Samba / Unix account
    └── Controls filesystem access
```

## Network Interface Model
| Interface    | Purpose                           | Allowed Inbound Access                         |
| ------------ | --------------------------------- | ---------------------------------------------- |
| `tailscale0` | Primary FSS service access        | SMB for File Users/Admins; SSH for Admins      |
| `enp3s0`     | Physical LAN / Tailscale underlay | Tailscale transport; SSH from `cy-server` only |

## Storage Layout

### Current
```
256GB SSD
│
├── /
│
├── /etc                    ← All configuration files
├── /opt                    ← Application files / container definitions
├── /var                    ← Runtime data, databases, caches, and system logs
│
└── /srv
    ├── storage             ← Persistent service data
    │   └── shares          ← Currently shared data          
    │
    ├── logs                ← Long-term audit, security, and network logs
    │   ├── samba
    │   ├── firewall
    │   └── traffic
    │
    └── backups             ← Temporarily staged backup files

```

### Upgrade Planned
```
256GB SSD — System
├── Debian
├── Samba
├── Tailscale
├── nftables
├── Container Runtime
├── /etc
├── /var
└── /opt


2TB SSD — Persistent Data
└── /srv
    ├── storage
    │   ├── shares
    │   └── docker
    ├── logs
    │   ├── samba
    │   ├── firewall
    │   ├── traffic
    │   └── ids
    └── backups


2TB HDD — Recovery Backup
├── /srv
├── /etc
├── ACL exports
└── Recovery scripts
```


## Logging / Observability Pipeline
```
File Operations
Samba
  |
  v
vfs_full_audit
  |
  v
journald / rsyslog
  |
  v
Audit Logs


Firewall Events
nftables
  |
  v
journald
```

## Backup Architecture
```
        cy-server-fss
        +------------------+
        | shared/          |
        | backup/          |
        +--------+---------+
                 |
                 | SMB
                 | via Tailnet
                 v
        +------------------+
        | Windows          |
        | Workstation      |
        +--------+---------+
                 |
                 | Manual periodic copy
                 v
        +------------------+
        | Mechanical HDD   |
        | Backup Storage   |
        +------------------+
```

## Current vs Planned Architecture
### Current
- FSS remains physically connected to the main home LAN.
- User access is performed through the Tailnet.
- LAN-originated inbound traffic is dropped by default.
- `cy-server` is the only LAN host allowed to SSH into FSS.
- File operations are audited.

### Planned
- Dedicated VLAN for server infrastructure.
- Automate periodic backups from FSS to dedicated backup storage.
- Build a monitoring dashboard for FSS file activity and network traffic statistics.
- Develop a lightweight host/network IDS for FSS to detect suspicious activity and generate alerts.
- Redesign the FSS storage architecture into dedicated system, persistent-data, and recovery-backup tiers using a 256 GB system SSD, 2 TB data SSD, and 2 TB recovery HDD.

