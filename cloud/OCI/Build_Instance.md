# 如何创建服务器实例并部署网页

---

结构：
Internet → 公网 IP → OCI VCN → Public Subnet → Ubuntu Instance → Docker → Flask + Nginx + SQLite

---

##创建服务器Build Instance

  ### 1.Basic inforamtion
    Name: 服务器名自己填（本案例：cy-server-Ora(x86)）
    Create in compartment: 默认自己的账号(root)
  
    Placement/Availbility domain: 取决于创建账号时选择的城市，固定城市后选择机房
    Placement/Advanced options: 选择服务器主机部署方案（本案例：默认选On-demand capacity（放在共享宿主机上））

    Image：按需选择系统（本案例：Canonical Ubuntu 24.04 Minimal）
    Shape：按钱包选择硬件配置（本案例：VM.Standard.E2.1.Micro 「Always Free-eligible」）
    Image and shape/Advanced options
      Instance metadata service: OCI 给实例提供的“内部信息接口”
      Require an authorization header：要求访问 metadata 时必须使用 IMDSv2，并带上认证 header。开了打开安全性高，关闭兼容性高（本案例：开）
      Initialization script：开机初始化脚本，例如开机自动执行·apt update·（本案例：跳过）
      
  ### 2.Security
    Shielded instance：开启增强启动安全的选项，启用Secure Boot、Measured Boot、TPM 组合来增强 VM。关闭兼容性好（本案例：关闭）
    Advanced options/Security attributes：给资源打安全标签，然后用 ZPR 策略控制哪些资源能互相访问。（本案例：不添加）

  ### 3.Networking
    Primary VNIC/VNIC name：虚拟网卡起名字（本案例：VNIC-FLask）
    
    Primary network：复用已有VCN虚拟网络或者创建新的VCN虚拟网络（本案例：复用vcn-2026XXXX-XXXX）
    
    subnet：复用已有的LAN或者创建新的LAN。注意出入站规则对该LAN内所有Instance生效。（本案例：Create new public subnet）
      New subnet name: 局域网起名字（本案例：subnet-Flask）
      Create in campartment: 默认自己的账号(root)
      CIDR block: 虚拟局域网网段范围，不能和同一个VCN下其他subnet有重复。（本案例：10.0.1.0/24）

    IPv4 address assignment: 跳过，等Subnet随着Instance一起创建后才能设置

    Add SSH keys: 选择SSH密钥方案。（本案例：Generate a key pair for me，然后Download private key，可选Download public key）

  ### 3.Boot volume
    Boot volume：系统盘大小与硬盘吞吐量（本案例：默认46.6G）
    Use in-transit encryption：Instance ↔ 存储之间的数据流量加密（本案例：关闭）
    Encrypt this volume with a key that you manage：自管硬盘加密密钥（本案例：关闭）
    Block volumes：额外挂载硬盘（本案例：不挂载）

  ### 4. Review
    检查配置，没有问题就创建服务器实例

---

##设置网络和SSH

    ### 1.进入cy-server-Ora(x86)实例，点击Networking，查看是否有Public IPv4 Address

    ### 2.进入VNIC的IP administration，如果没有IPv4就添加一个，有就查看是Reserved还是Ephemeral并记住IP地址（本案例：有，是Ephemeral，地址为161.XX.XX.XX)

    ### 3.把下载的密钥放进~/.ssh

    ### 4.（可选）设置ssh config:
        Host server-agile
          HostName 161.XX.XX.XX
          User ubuntu
          IdentityFile ~/.ssh/ssh-key-2026-05-16.key
      
    ### 5.设置SSH密钥权限:
        ·chmod 600 ~/.ssh/ssh-key-2026-05-16.key·

    ### 6.SSH服务器：
        ·ssh server-agile`

---

##测试控制台是否工作正常

    ### 1.SSH进入服务器后设置密码：
    ·sudo passwd ubuntu·

    ### 2.进入OS Management，点击Counsole connection
        账号：ubuntu
        密码：刚刚设置的
        
---

##打开端口

    ### 1. 写subnet-Flask的入站规则
      开放80端口：
          Souce CIDR: 0.0.0.0/0（所有IP全部可以进入）
          IP Protocal：TCP（进入协议是网页跑的TCP协议）
          Souce Port Range：留空（所有端口都可以进入）
          Destination Port Range：80（进入内部subnet 80端口）
          Description：HTTP connect
    
      开放443端口：
          Souce CIDR: 0.0.0.0/0（所有IP全部可以进入）
          IP Protocal：TCP（进入协议是网页跑的TCP协议）
          Souce Port Range：留空（所有端口都可以进入）
          Destination Port Range：443（进入内部subnet 443端口）
          Description：HTTPS connect
    
      开放Ping端口：
          Souce CIDR: 0.0.0.0/0（所有IP全部可以进入）
          IP Protocal：ICMP（Ping跑的协议）
          Souce Port Range：留空
          Destination Port Range：留空
          Description：Ping
    
    ### 2.查看Oracle定制版Ubuntu默认防火墙
      查看ip table:
          ·sudo iptables-save·
          默认是：
          1 established2 icmp
          3 lo
          4 ssh
          5 reject
    （可选）查看nft table
        ·sudo nft list ruleset·

    ### 3. 开放Ubuntu ip table端口
        优先级5开放80端口
        ·sudo iptables -I INPUT 5 -p tcp --dport 80 -j ACCEPT·
        优先级6开放443端口
        ·sudo iptables -I INPUT 6 -p tcp --dport 443 -j ACCEPT·
        检查是否写入ip table
        `sudo iptables -L -n --line-numbers`
        
---

##安装docker
    https://docs.docker.com/engine/install/ubuntu/#installation-methods
    
    ### 1.安装基础包
    `sudo apt update`
    `sudo apt install -y ca-certificates curl`

    ### 2.创建 keyrings
    `sudo install -m 0755 -d /etc/apt/keyrings`

    ### 3.下载 Docker GPG key
    `sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`

    ### 4.设置权限
    `sudo chmod a+r /etc/apt/keyrings/docker.asc`

    ### 5.添加 Docker repo
    看官网

    ### 6.更新 apt
    `sudo apt update`

    ### 7.安装 Docker
    `sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

    ### 8.测试Docker
    `sudo docker run hello-world`
    

    ### 9.
    
    



    
      







    

    
      
    
  

  
