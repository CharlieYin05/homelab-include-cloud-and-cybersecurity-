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

Zero-Trust Architecture


## Core Design
### cy-server
Role: network and service hub
### cy-server-fss
Role: dedicated file server
### Windows Workstation
Role: primary desktop / streaming host
### Android Tablet
Role: portable remote display / client
### MacBook
Role: main development and administration endpoint
### Mi Play
Role: portable network probe
