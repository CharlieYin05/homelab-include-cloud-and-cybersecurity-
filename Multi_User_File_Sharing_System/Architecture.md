# Architecture

## Design Goals
The system is designed around five principles:

1. **No** direct SMB or SSH **exposure** to the **public Internet**.
2. Joining the Tailnet does not imply unrestricted server access.
3. Network access and file access are **authorised independently**.
4. Every user has an **individual identity**.
5. All file operations should be **auditable**.

## System Architecture
#### Two Path that can access to fss:
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

#### Two Path interm of devices:
```

                            Tailnet Devices                         Home LAN Devices
                                   |                                       |
                          |--------+---------|                   |---------+---------|
                          |                  |                   |                   |
                          v                  v                   v                   v
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
                                   |                              |
fss port:445<----(admin & user)----|---(admin)--> fss port:22 <---|
```
## Layered Architecture

## Access Flows

## Identity & Permission Model

## Network Interface Model

## Storage Layout

## Logging / Observability Pipeline

## Backup Architecture

## Current vs Planned Architecture
