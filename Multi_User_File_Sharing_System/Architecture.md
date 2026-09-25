# Architecture

## Design Goals
The system is designed around five principles:

1. **No** direct SMB or SSH **exposure** to the **public Internet**.
2. Joining the Tailnet does not imply unrestricted server access.
3. Network access and file access are **authorised independently**.
4. Every user has an **individual identity**.
5. All file operations should be **auditable**.

## System Architecture
#### Two Network Paths to FSS:
```
                                     cy-server-fss
                              +-------------------------+
                              |   /srv/shares/shared    |
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
              TAILNET PATH                           HOME-LAN PATH
        =============================         ==============================
```

#### Access Paths by Device:
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

## Identity & Permission Model
| Role | SMB | SSH | File Access |
|---|---:|---:|---|
| Admin | Yes | Yes | Administrative / authorised access |
| File User | Yes | No | Defined by POSIX ACL |
| cy-server | No | Yes via LAN | Server administration |
| Other LAN Device | No | No | None |

Network Identity
└── Tailscale user / group

File Identity
└── Samba / Unix account

## Network Interface Model
| Interface | Purpose | Allowed Inbound Access |
|---|---|---|
| `tailscale0` | Tailnet-native client access | SMB for users/admins; SSH for admins |
| LAN interface | Restricted local administration | SSH from `cy-server` only |

## Storage Layout
```
/srv/
├── shares/
│   └── shared/
├── logs/
│   ├── samba/
│   └── firewall/
└── backups/
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


Traffic Statistics
Network Interfaces
  |
  v
vnStat
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
- Backups are currently performed manually on a periodic basis.

### Planned
- Dedicated VLAN for server infrastructure.
- Stronger network segmentation between servers, clients and IoT devices.
- Build a monitoring dashboard for FSS file activity and network traffic statistics.
- Develop a lightweight host/network IDS for FSS to detect suspicious activity and generate alerts.
- Scheduled automated backups from FSS to the Windows Workstation backup storage.

