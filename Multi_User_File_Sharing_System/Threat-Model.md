# 威胁建模

## 1. Purpose
本文档用于记载「文件共享项目」设计过程中所考虑的威胁

解释：
- 什么需要保护
- 潜在攻击者是谁
- 有哪些缓解措施
- 可以接受哪些风险

## 2. Assets
   
## 3. Actors
   
## 4. Trust Boundaries
   
## 5. Threats

## 6. Security Controls

管理员 SSH fss服务器
```
                    Internet
                        │
                 Tailscale Tailnet
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
     管理员设备                  网络中枢服务器
 (MacBook/Windows)             (Jump Host)
          │                           │
          │ SSH（ACL允许）             │ SSH
          │                           │
          └─────────────┬─────────────┘
                        ▼
                  fss 文件服务器
```

## 7. Residual Risks

## 8. Out of Scope

## 9. Acceptance Criteria
