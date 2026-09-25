# Threat Model

## Status

This document is a retrospective threat model for the Multi-User File Sharing System.

Threat modelling was performed after most of Phase 1 had been implemented because the architecture evolved during construction and troubleshooting.

Phase 1 is approximately 90% complete. Controls that are not yet deployed are explicitly marked as **Pending** rather than treated as existing protections.

---

## Scope

This threat model covers the current FSS architecture and its primary security boundaries:

- `cy-server-fss`
- Tailscale / Tailnet access
- Tailscale Grants
- FSS host firewall (`nftables`)
- Samba authentication
- Linux users and groups
- POSIX ACLs
- Shared files under `/srv/storage/shares/`
- Samba audit logging
- Internal DNS used for `fss.cy-server.com`
- Restricted LAN administration from `cy-server`
- Current manual backup workflow

The model focuses on access to the FSS service and protection of shared data.

---

## Security Objectives

The system is designed to achieve the following security objectives:

1. Prevent direct SMB and SSH exposure to the public Internet.
2. Ensure that joining the Tailnet does not automatically grant access to FSS services.
3. Separate network-level authorisation from file-level authorisation.
4. Provide every user with an individual identity.
5. Restrict ordinary file users to SMB access only.
6. Restrict administrative SSH access to authorised administrators.
7. Prevent users from accessing files outside their assigned permissions.
8. Record security-relevant file operations for later investigation.
9. Reduce direct exposure of FSS to other Home LAN devices.
10. Maintain a recoverable copy of important data.

---

## Assets

| Asset | Security Requirement |
|---|---|
| `public/` shared files | Integrity and access control |
| `restriction/` shared files | Confidentiality, integrity and access control |
| `private/` shared files | Strong confidentiality and integrity |
| Samba credentials | Confidentiality |
| Linux user identities and groups | Integrity |
| POSIX ACLs and filesystem permissions | Integrity |
| Samba configuration | Integrity |
| Tailscale Grants | Integrity |
| nftables rules | Integrity |
| Audit logs | Integrity and availability |
| Backup data | Integrity and availability |
| FSS operating system | Integrity and availability |
| Administrative access | Strong authentication and restricted reachability |

---

## Actors

| Actor | Trust Level | Intended Access |
|---|---|---|
| Administrator | Trusted / privileged | SMB and SSH |
| File User | Partially trusted | SMB only |
| `cy-server` | Trusted infrastructure host | Restricted LAN SSH administration |
| Other Home LAN Device | Untrusted for FSS access | None |
| Unauthorized Internet User | Untrusted | None |
| Compromised File User Device | Untrusted despite valid identity | May possess valid Tailnet and SMB credentials |
| Compromised Admin Device | High-impact threat | May possess administrative credentials |
| Malicious Authorised User | Partially trusted | Valid access that may be intentionally misused |

An authorised user is not automatically considered fully trusted.

A legitimate File User may still attempt to access restricted files, misuse granted access, delete data, or redistribute files that they are authorised to read.

---

## Trust Boundaries

```
                         PUBLIC INTERNET
                               |
                               |
                        Home Router / NAT
                               |
                               v
                         Physical LAN
                               |
                          FSS enp3s0
                               |
                +--------------+--------------+
                |                             |
                | Tailscale Transport         |
                |                             |
                v                             |
             Tailscale                        |
                |                             |
                v                             |
        Tailscale Identity                    |
                |                             |
                v                             |
        Tailscale Grants                      |
                |                             |
                v                             |
          FSS tailscale0                      |
                |                             |
                +-------------+---------------+
                              |
                              v
                           nftables
                              |
                              v
                    Samba Authentication
                              |
                              v
                   Unix Identity Mapping
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

A second, narrowly restricted administration path exists:
```
cy-server
    |
    | TCP 22 only
    v
Home LAN
    |
    v
FSS enp3s0
    |
    v
nftables
    |
    v
SSH
```
All other direct SMB/SSH access from Home LAN devices is intended to be denied.

---

## Identity and Authorisation Boundaries

The system intentionally separates network identity from file identity.

```
Network Identity
Tailscale user / group
        |
        v
Tailscale Grants
        |
        | controls service reachability
        v
Service Identity
Samba account
        |
        v
Linux UID / GID
        |
        | controls resource access
        v
Filesystem Authorisation
Linux Groups + POSIX ACL
```
A user being allowed to reach TCP/445 does not imply that the user is allowed to access every Samba share or every file.

---

## Existing Security Controls

| Layer              | Control                                           | Status                       |
| ------------------ | ------------------------------------------------- | ---------------------------- |
| Internet perimeter | No direct SMB/SSH Internet exposure               | Implemented                  |
| Tailnet            | Identity-based Tailscale access                   | Implemented                  |
| Tailnet            | `group:file-user` → FSS TCP/445 only              | Implemented                  |
| Tailnet            | `group:admin` → FSS TCP/22 and TCP/445            | Implemented                  |
| Host firewall      | Default-deny nftables INPUT policy                | Mostly implemented           |
| Host firewall      | SMB/SSH access through `tailscale0`               | Implemented                  |
| Host firewall      | LAN SSH exception for `cy-server` only            | **Pending final deployment** |
| Samba              | Individual Samba accounts                         | Implemented                  |
| Samba              | `valid users` share restrictions                  | Implemented                  |
| Samba              | Guest access disabled                             | Implemented                  |
| Samba              | SMB3 minimum protocol                             | Implemented                  |
| Samba              | Unnecessary print/NetBIOS functionality disabled  | Implemented                  |
| Filesystem         | Linux users and groups                            | Implemented                  |
| Filesystem         | SGID group inheritance                            | Implemented                  |
| Filesystem         | POSIX ACL / Default ACL                           | Implemented                  |
| Auditing           | Samba `vfs_full_audit`                            | Implemented                  |
| Logging            | rsyslog audit collection                          | Implemented                  |
| Logging            | logrotate retention                               | Implemented                  |
| Firewall logging   | nftables event logging                            | Implemented                  |
| Backup             | Periodic manual backup to Windows Workstation HDD | Implemented after Phase 1    |
| IDS                | Host/network intrusion detection                  | Planned                      |
| Traffic monitoring | Network traffic collection / dashboard            | Planned                      |
| Network Segmentation | Dedicated server VLAN                           | Planned                      |

















