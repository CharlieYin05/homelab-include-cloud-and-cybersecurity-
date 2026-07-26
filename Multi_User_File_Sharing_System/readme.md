# 家庭零信任文件共享系统

本项目不是为了寻找最简单的文件共享方案，而是以"共享一个文件夹"为目标，通过渐进式工程实践学习 Linux、网络、安全和运维。

设计原则：
- 分层防御（Internet → LAN隔离 → Tailscale（身份认证） → Tailscale Grants（网络访问控制） → nftables（主机防火墙）→ Samba（文件共享） → POSIX ACL（文件权限））
- 最小暴露面（ip，端口，协议，文件夹，服务）
- 最小权限*（控制网络权限：能不能到达服务器 + 控制资源权限：到达服务器以后还能干什么）
- 可观测性（网络：流量监控 + 资源：谁，什么时候，动过什么文件）
- 数据保护（定期冷备份在其他硬盘）

### *最小权限
Least Privilege
├── Network Access Control
│     ├── Router NAT
│     ├── Tailscale
│     ├── Grants
│     └── nftables
│
└── Resource Access Control
      ├── Samba Users
      ├── Shared Folder
      └── POSIX ACL (rwx)
      
## 目标1（暂定）
- 所有远程访问只能通过 Tailscale。
- 外部用户只看到文件服务器本身。
- 外部用户不能访问家庭路由器、主局域网或其他家庭设备。
- 外部用户只能访问文件服务器的 TCP 445 端口。
- 文件服务器不向外部用户开放 SSH、Web、数据库或其他服务端口。
- 每个用户使用独立的 Tailscale 身份和 Samba 账户。
- 支持 Windows、macOS、iOS 和 Android。
- 使用 POSIX ACL 控制不同用户和组的读、写、执行权限。
- 记录文件创建、修改、重命名、删除和失败操作。
- 记录 VPN 接口和无线接口的流量。

## 目标2（暂定）
- 部署 Immich 相册。
- Immich 仅允许通过 Tailscale 访问。
- 增加 Suricata IDS。
- 增加 Prometheus、Loki 和 Grafana。
- 实现日志集中化、流量可视化和安全告警。

## 工程化实施

Phase 1
=================
- VPN
- Firewall
- Samba
- ACL
- Monitoring
- Backup

↓

Phase 2
=================
- Docker
- Immich
- HTTPS
- Reverse Proxy
- Photo Permission

↓

Phase 3
=================
- Suricata
- Grafana
- Loki
- Prometheus
- Alert
- Incident Response
