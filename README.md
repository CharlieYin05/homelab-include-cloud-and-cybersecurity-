# Homelab Infrastructure

A living homelab environment for experimenting with networking, Linux infrastructure, access control, self-hosting, and cross-device services.

**This infrastructure is continuously evolving.**
The topology, services, access policies, and device roles documented here represent the current state of the lab and will change as the environment grows.

## Architecture at a Glance

### Physical Network Topology
```
                                   Merlin Router
                                    - LAN Internal DNS
                                     - cy-server.com --------------|
                                     - portainer.cy-server.com ----| 
                                     - kvm.cy-server.com ----------|----- → cy-server's IP
                                     - clipcascade.cy-server.com --| 
                                     - npm.cy-server.com ----------|
                                     - fss.cy-server.com ---------------- → cy-server-fss's tailscale IP
                                       |
                                       |(Home LAN)
                                       |
  |—---------------—-------------------|--------------------------------------|-----------------------------|
  |                                    |                                      |                             |
  |                                    |                                      |                             |
cy-server                        cy-server-fss                  |======= Windows Work Station         Other Home Device                   
 - Tailscale Exit Node            - Tailscale Access Only       |         - Tailscale node
 - Tailscale Subrouter            - Samba                       |         - Sunshine
 - Docker                         - vfs_full_audit              |         - ClipCascade (Client)                   
  - ClipCascade (Server)                                        |             |
  - OneKVM (Controller side) ============(HDMI to USB)==========|             |                
  - Nginx Proxy Manager                                                       |
                                                                          OpenWRT Router
                                                                           - Wake on LAN
                                                                              |
                                                                              |(Sunshine LAN)
                                                                              |
                                                                         Android Tablet
                                                                          - Moonlight V+
```
### Virtual Network (Tailscale) Topology
```
                                                                  Charlie's Tailnet(Zero-Trust)
                                                                             |
                                                                             |
  |-------------------------------|----------------------------|-------------------------------|----------------------------------|---------------------------|
  |                               |                            |                               |                                  |                           |
cy-server                   cy-server-fss               Windows Work Station                Macbook                             Mi Play                Other Personal Device
 - Docker                    - Tailscale Access Only     - Sunshine                          - Main Develop Device               - Got Root Privilege   - Moonlight
  - ClipCascade Server       - Samba (SMB Server)        - ClipCascade Client                - ClipCascade Client                - nmap                 - SMB Client
  - OneKVM (Controller side) - vfs_full_audit            - SMB Client                        - SwitchyOmega---(SOCK5 tunnel)---- - ssh -D
  - Nginx Proxy Manager           |                                                          - SMB Client
 - Tailscale Exit Node            |                                                          - Moonlight
 - Tailscale Subrouter--|         |
                        |         |
                        |         |
                       Home LAN   | 
                                  | 
                              Other User
                               -SMB Client
```
### Access Control Model
The Tailnet is not configured as a flat trusted network.
Access is granted by role and service.

| Source | Destination | Access |
|--------|-------------|--------|
| Admin  | `cy-server` | SSH, HTTPS |
| Admin  | `cy-server-fss` | SSH, SMB |
| Admin  | Home Router | SSH, Web UI, DNS |
| Admin  | Internet | Via Tailscale Exit Node |
| File User | `cy-server-fss` | SMB only |
| File User | Home Router | DNS only |

> Device membership does not automatically imply access to every service.

### Key Traffic Flows
```text
Shared Folder Access
User -> Tailnet -> fss.cy-server.com -> Samba

Infrastructure Administration
Admin -> Tailnet -> cy-server / Home Router

LAN Game Streaming
Windows Workstation -> Sunshine -> OpenWRT -> Android Tablet / Moonlight V+

SOCKS5 Proxy
MacBook -> SwitchyOmega -> ssh -D -> Mi Play



| Node | Primary Role |
|---|---|
| `cy-server` | Network hub, Docker host, Exit Node, Subnet Router |
| `cy-server-fss` | Audited Samba file server |
| `Windows Workstation` | Sunshine host and main workstation |
| `MacBook` | Main development / administration client |
| `Android Tablet` | Moonlight streaming client |
| `Mi Play` | SSH SOCKS5 tunnel endpoint |
