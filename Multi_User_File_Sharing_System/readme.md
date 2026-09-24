# Multi-User File Sharing System

A self-hosted multi-user file sharing system built around Samba, Tailscale, POSIX ACLs and audit logging.

The system is designed to provide remote file access while keeping administrative access, file permissions and auditability separated.

## What It Does

- Remote SMB access through Tailscale
- Separate administrator and file-user access
- Per-user Samba authentication
- POSIX ACL-based file permissions
- Samba file-operation auditing
- Host firewall enforcement with nftables
- Offline backup workflow

## Architecture at a Glance
```
 Admin Device                                File User
      |                                          |
      | SSH + SMB                                | SMB only
      |-------------------|  |-------------------|
                          |  |
                          v  v
                    +-------------+
                    |  Tailscale  |
                    |   Grants    |
                    +------+------+
                           |
                           v
                    +-------------+
                    |  nftables   |
                    +------+------+
                           |
                           v
                    +-------------+
                    |    Samba    |
                    | Authentication
                    +------+------+
                           |
                           v
                    +-------------+
                    |  POSIX ACL  |
                    +------+------+
                           |
                           v
                    /srv/shares/shared
```

## Security Model

User access passes through multiple independent controls:
```
Tailscale Identity
      ↓
Tailscale Grants
      ↓
nftables
      ↓
Samba Authentication
      ↓
POSIX ACL
      ↓
Shared Files
```

## Documentation

- [Architecture](./Architecture.md)
- [Threat Model](./Threat-Model.md)
- [Operations / Troubleshooting](...)
