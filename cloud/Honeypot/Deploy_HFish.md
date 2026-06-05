# HFish蜜罐部署

___
## 项目简介
  - 基于 OCI 的云蜜罐实验环境
  - 使用 HFish 收集互联网攻击流量
  - 分析自动化扫描、爆破、恶意连接行为
  - 学习云安全与主机加固
___
## 项目目的
 - 学习搭建蜜罐
 - 观察真实互联网攻击
 - 研究攻击者行为
 - 为未来的IDS做流量收集
 - 用家里机器怕被打穿+运行商封号
___
## 为何选择HFish
  ### 常见蜜罐方案对比
    | 方案         | 原因                    |
    | ---------- | ------------------       |
    | HFish      | 轻量、中文生态好、支持多协议  |
    | Cowrie     | SSH/Telnet 强，但功能单一   |
    | T-Pot      | 太重，OCI 免费机型可能跑不动  |
    | Dionaea    | 偏恶意软件收集              |
    | OpenCanary | 更适合轻量告警              |
  ### 选择HFish原因：
    - 我关注的B站UP·主网络小白_Uncle城·用它
    - 支持协议全面
    - 有中文教程，适合入门
    - 未来会考虑增加或更换蜜罐
___
## 结构
```text
  Internet
     ↓
  OCI Public IP
     ↓
  VCN
     ↓
  Public Subnet
     ↓
  Ubuntu
     ├── Docker
     │     └── HFish
     └── UFW/iptables
```
___
## 技术栈
  - Oracle Cloud Infrastructure (OCI)
  - Ubuntu Server 24
  - Docker / Docker Compose
  - HFish
  - UFW / iptables
