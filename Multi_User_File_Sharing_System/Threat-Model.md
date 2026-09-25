# Threat Model

## Scope

This threat model covers the Multi-User File Sharing System (FSS), including:

- `cy-server-fss`
- Tailscale / Tailnet access
- Tailscale Grants
- `nftables`
- Samba authentication
- Linux users and groups
- POSIX ACLs
- `/srv/storage/shares/`
- Audit and firewall logs
- Internal DNS
- Backup storage

This model was created retrospectively near the end of Phase 1 because the system architecture evolved during implementation and troubleshooting.

---

## Security Objectives

The system is designed to:

1. Prevent direct SMB and SSH exposure to the public Internet.
2. Ensure Tailnet membership does not automatically grant FSS access.
3. Separate network authorisation from file authorisation.
4. Give each user an individual identity.
5. Restrict File Users to SMB access only.
6. Restrict administrative access to authorised administrators.
7. Prevent users from accessing files outside their assigned permissions.
8. Record security-relevant file operations.
9. Reduce lateral movement from the Home LAN.
10. Maintain recoverable copies of important data.

---

## Assets & Actors

### Assets

- Shared files:
  - `public/`
  - `restriction/`
  - `private/`
- Samba credentials
- Linux users, groups and ACLs
- Samba configuration
- Tailscale Grants
- `nftables` rules
- Audit and firewall logs
- Backup data
- Administrative access

### Actors

| Actor | Intended Access |
|---|---|
| Administrator | SMB + SSH |
| File User | SMB only |
| `cy-server` | Restricted LAN SSH administration |
| Other Home LAN Device | No direct FSS access |
| Unauthorized Internet User | No access |
| Compromised User Device | May possess legitimate user credentials |
| Compromised Admin Device | May possess privileged credentials |
| Malicious Authorised User | May misuse legitimate file access |

An authorised user is not automatically fully trusted.

---

## Trust Boundaries

```text
Internet / Home LAN
        |
        v
     nftables
        |
        v
    Tailscale
        |
        v
 Tailscale Grants
        |
        v
Samba Authentication
        |
        v
Unix UID / GID Mapping
        |
        v
Linux Groups + POSIX ACL
        |
        v
/srv/storage/shares/
├── public
├── restriction
└── private
```

Network identity and file identity are intentionally separated:

```text
Tailscale Identity
        |
        v
Network Authorisation
        |
        v
Samba Account
        |
        v
Filesystem Authorisation
```

Being allowed to reach TCP/445 does not mean a user is allowed to access every file.

---

## Security Controls

| Layer | Control | Status |
|---|---|---|
| Internet | No direct SMB/SSH exposure | Implemented |
| Tailnet | File User → TCP/445 only | Implemented |
| Tailnet | Admin → TCP/22 + TCP/445 | Implemented |
| Host Firewall | Default-deny `nftables` INPUT policy | Implemented |
| Host Firewall | LAN SSH from `cy-server` only | **Pending — Phase 1** |
| Samba | Individual user accounts | Implemented |
| Samba | `valid users` share restrictions | Implemented |
| Samba | Guest access disabled | Implemented |
| Samba | SMB3 minimum protocol | Implemented |
| Filesystem | Linux groups + SGID | Implemented |
| Filesystem | POSIX ACL + Default ACL | Implemented |
| Auditing | Samba `vfs_full_audit` | Implemented |
| Logging | `rsyslog` + `logrotate` | Implemented |
| Firewall Logging | `nftables` event logging | Implemented |
| Backup | Manual backup to Windows Workstation HDD | Implemented |
| Network Segmentation | Dedicated server VLAN | **Planned — Phase 2** |
| Observability | File/network activity dashboard | **Planned — Phase 2** |
| Detection | Host/network IDS and alerts | **Planned — Phase 2** |
| Storage | Separate system/data/recovery disks | **Planned — Phase 2** |

---

## Threats & Controls

| Threat | Potential Impact | Primary Controls | Residual Risk |
|---|---|---|---|
| Direct Internet SMB/SSH access | Unauthorized remote access | Router/NAT, `nftables` | Future misconfiguration |
| File User attempts SSH | Privilege escalation | Tailscale Grants | Grants are the identity-aware enforcement point |
| User accesses unauthorized files | Confidentiality / integrity loss | Samba `valid users`, Linux groups, POSIX ACL | Permission misconfiguration |
| Compromised File User device | Abuse of legitimate access | Tailnet identity + Samba authentication + ACL | Attacker inherits the user's permissions |
| Compromised Admin device | Full FSS compromise | Restricted admin access | Admin remains highly privileged |
| Compromised Home LAN device | Lateral movement | `nftables` default deny | Same physical LAN until Phase 2 VLAN |
| Compromised `cy-server` | Access to trusted LAN SSH path | Source-restricted firewall rule + SSH auth | Trusted infrastructure host becomes a pivot point |
| Tailscale Grants misconfiguration | Excessive network access | Explicit groups, tags and ports | Policy remains manually maintained |
| Accidental / malicious file deletion | Data loss | Audit logs + backups | Audit records but does not prevent the action |
| Disk failure | Data loss / downtime | External backup | Single production SSD and manual backups |
| Audit log tampering | Loss of accountability | `vfs_full_audit`, `rsyslog`, log rotation | Logs are stored locally |
| Suspicious activity goes unnoticed | Delayed detection | Audit + firewall logs | No IDS or automated alerting yet |
| Internal DNS failure | Hostname access unavailable | Direct Tailscale address remains available | Router DNS is still a dependency |

---

## Residual Risks

Phase 1 does not eliminate all risk.

- A legitimate user can copy and redistribute files they are authorised to read.
- A compromised File User device may act with that user's permissions.
- A compromised administrator may result in full FSS compromise.
- Tailscale Grants remain the main identity-aware control separating File Users from Administrators.
- FSS remains on the main Home LAN until VLAN segmentation is implemented.
- The planned `cy-server` LAN SSH path introduces an additional trusted-host dependency.
- Audit logs are stored locally and may be modified after a sufficiently privileged compromise.
- Backups are currently manual.
- The production server currently relies on a single SSD.
- No IDS or automated alerting is currently deployed.

---

## Phase 1 Acceptance Criteria

```text
File User -> Tailnet -> SMB 445       PASS
File User -> Tailnet -> SSH 22        FAIL

Admin -> Tailnet -> SMB 445           PASS
Admin -> Tailnet -> SSH 22            PASS

Other LAN Device -> FSS SMB 445       FAIL
Other LAN Device -> FSS SSH 22        FAIL

cy-server -> FSS LAN -> SSH 22        PASS
cy-server -> FSS LAN -> SMB 445       FAIL
```

The final `cy-server -> FSS TCP/22` firewall exception still needs to be deployed and validated before Phase 1 is considered complete.

Filesystem validation should also confirm that:

- Read-only users cannot modify files.
- Read/write users can modify authorised files.
- Users cannot access shares outside their assigned groups.
- `private/` remains restricted to authorised users.
- Default ACL inheritance works for newly created files and directories.
- Security-relevant Samba operations generate audit records.

---

## Phase 2 — Infrastructure Hardening

### Observability
- File activity visualisation
- Network traffic collection and visualisation

### Detection
- Host/network IDS
- Automated security alerts

### Network Segmentation
- Dedicated server VLAN
- Explicit inter-VLAN firewall rules

### Storage & Recovery
- 256 GB system SSD
- 2 TB persistent-data SSD
- 2 TB recovery HDD
- Automated backups
- Recovery validation













