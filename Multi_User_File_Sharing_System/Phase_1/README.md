# Phase 1 — Core File Server Infrastructure

Phase 1 establishes the core infrastructure of the Multi-User File Sharing System.

The goal of this phase is to build a file server that provides:

- Individual user identities
- Role-based file permissions
- Remote SMB access through Tailscale
- Administrative SSH access
- Auditable file operations
- Host-level firewall enforcement
- No direct SMB exposure to the public Internet

---

## Build Flow

```text
✓ Debian Server
     |
     v
✓ /srv Directory Structure
     |
     v
✓ Linux Users & Groups
     |
     v
✓ chmod / SGID / POSIX ACL
     |
     v
✓ Samba File Sharing
     |
     v
✓ Samba File Auditing
     |
     v
✓ Tailscale Access Control
     |
     v
nftables Host Firewall
```
---

## 00 — System Setup

Initial server installation and filesystem planning.

- Debian 13
- Samba
- ACL tools
- nftables
- Base server utilities
- /srv directory structure
- Static LAN addressing

See: ./00_Setup_System.md

---

## 01 — Directory Structure & Permissions

Builds the filesystem authorisation model.

The shared data is stored under:
```
/srv/storage/shares/
├── public
├── restriction
└── private
```

Permissions are implemented using:
```
Linux Users
    |
    v
Linux Groups
    |
    +--> chmod
    |
    +--> SGID
    |
    +--> POSIX ACL
    |
    +--> Default ACL
```
Groups provide the fixed role-based permission model, while ACLs allow more specific permissions where required.

See: 01_Directory_Structure_and_Permission.md

---

## 02 — Samba File Sharing

Provides SMB file access across Windows, macOS, Linux, Android, and iOS clients.

The access model is:
```
SMB Client
    |
    v
Samba Authentication
    |
    v
Unix User / Group Mapping
    |
    v
Samba Share Rules
    |
    v
Linux Groups + POSIX ACL
    |
    v
/srv/storage/shares/
```

Samba authenticates the user, while the Linux filesystem determines what that user is authorised to access.

See: 02_Setup_Samba.md

---

## 03 — File Auditing

Adds accountability for security-relevant file operations.

```
File Operation
     |
     v
Samba
     |
     v
vfs_full_audit
     |
     v
rsyslog
     |
     v
/srv/logs/samba/audit.log
     |
     v
logrotate
```

The audit system **records** operations such as **connection**, **file creation**, **directory creation**, **rename**, and **deletion**.

This phase also includes troubleshooting of Samba 4.22 vfs_full_audit behaviour through debug logs and source-level tracing.

See: 03_File_Auditing.md

---

## 04 — Tailnet Access

Introduces remote access and network-level identity.

Two Tailnet roles are used:

```
group:file-user
└── FSS SMB :445

group:admin
├── FSS SMB :445
└── FSS SSH :22
```

Tailscale Grants apply **least-privilege** network access instead of allowing every Tailnet member to reach every service.

Internal DNS also allows the FSS Tailnet address to be accessed through:
```
fss.cy-server.com
```

This file is kept as a construction record, so some access paths documented during this stage were later revised during the firewall phase.

---

## 05 — Host Firewall

Adds the FSS host-level network enforcement layer using nftables.

The target INPUT policy is:
```
INPUT
 |
 ├─ Loopback
 |    └─ ACCEPT
 |
 ├─ Established / Related
 |    └─ ACCEPT
 |
 ├─ Invalid
 |    └─ DROP
 |
 ├─ Required ICMP / ICMPv6
 |    └─ ACCEPT
 |
 ├─ Physical LAN Interface
 |    ├─ Tailscale transport
 |    │    └─ ACCEPT
 |    |
 |    └─ cy-server -> TCP 22
 |         └─ ACCEPT
 |
 ├─ tailscale0
 |    ├─ TCP 22
 |    │    └─ ACCEPT
 |    |
 |    └─ TCP 445
 |         └─ ACCEPT
 |
 └─ Everything Else
      └─ DROP
```

Direct SMB access from the Home LAN is not permitted.

The only planned direct LAN administrative exception is SSH from cy-server.

Status: In progress — the final cy-server -> FSS TCP/22 LAN rule still needs to be added and validated.

See: 05_nftables.md

---

## Phase 1 Access Model

When Phase 1 is complete, normal file access follows the Tailnet path:
```
File User
    |
    | SMB :445
    v
Tailscale Grants
    |
    v
tailscale0
    |
    v
nftables
    |
    v
Samba Authentication
    |
    v
Unix Identity / Group Mapping
    |
    v
Linux Groups + POSIX ACL
    |
    v
/srv/storage/shares/
```
Administrators additionally receive SSH access through the Tailnet.

A restricted LAN administration path is retained for `cy-server`:
```
Admin Device
    |
    | TCP :22 only
    v
cy-server
    |
    | TCP :22 only
    v
FSS LAN Interface
    |
    v
nftables
    |
    v
   SSH
```

---
## Next Phase

Phase 2 will focus on observability and detection rather than core file-serving functionality.

Planned areas include:

- FSS file-activity visualisation
- Network traffic collection and visualisation
- Monitoring dashboard
- Host/network intrusion detection and alerting

Backup automation and storage architecture upgrades are tracked separately as infrastructure improvements.

