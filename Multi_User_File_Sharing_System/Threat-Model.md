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

## 7. Residual Risks

## 8. Out of Scope

## 9. Acceptance Criteria

---

验证（Authentication）与授权（Authorization）防御链
```
① Tailscale
   Authentication
   “你是谁 / 是哪个设备？”
          ↓
   Authorization
   Grants：
   group:file-user → tag:fss:445
          ↓

② nftables
   Authorization
   “这个来源/接口能不能访问这个端口？”
          ↓

③ Samba
   Authentication
   “Samba 用户名密码正确吗？”
          ↓
   Authorization
   “这个用户能访问哪个 Share？”
          ↓

④ Linux filesystem
   Authorization
   “这个 Unix 身份对文件 r/w/x 哪些权限？”
```
