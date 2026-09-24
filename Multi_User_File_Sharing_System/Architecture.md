# Architecture

## Design Goals
The system is designed around five principles:

1. **No** direct SMB or SSH **exposure** to the **public Internet**.
2. Joining the Tailnet does not imply unrestricted server access.
3. Network access and file access are **authorised independently**.
4. Every user has an **individual identity**.
5. All file operations should be **auditable**.

## System Architecture
```
                      Internet
                          │
                          │ No Port Forwarding
                          ▼
                ┌───────────────────┐
                │ File Share Server │
                |  (cy-server-fss)  |
                │-------------------│
                │  Tailscale        │
                │      │            │
                │  nftables         │
                │      │            │
                │  Samba            │
                │      │            │
                │  POSIX ACL        │
                │      │            │
                │ /srv/shares/shared│
                └───────────────────┘


Admin Device                         User Device
    │                                  │
    └──────── Tailscale VPN ───────────┘
                     │
                     ▼
              Tailscale Grants
          管理员：SSH + SMB
          用户：仅 SMB TCP 445
```
## Layered Architecture

## Access Flows

## Identity & Permission Model

## Network Interface Model

## Storage Layout

## Logging / Observability Pipeline

## Backup Architecture

## Current vs Planned Architecture
