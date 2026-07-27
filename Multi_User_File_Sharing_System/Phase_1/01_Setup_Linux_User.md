# 创建与设置Linux用户

**2026-07-27**

---

## 权限模型
RBAC

```
| 共享目录          | 角色组                    | 权限         |
| ---------------- | ------------------------ | ----------- |
| `public`         | `cy_public_ro`           | 只读         |
| `public`         | `cy_public_rw`           | 读写         |
| `restriction`    | `cy_restriction_ro`      | 只读         |
| `restriction`    | `cy_restriction_rw`      | 读写         |
| `private`        | `cy_private_rw`          | 读写（管理员） |
```

### 分组：
```
cy_public_ro
    ├── guest1
    ├── guest2
    └── friendA

cy_public_rw
    ├── friendB
    └── friendC

cy_restriction_ro
    ├── friendD
    └── friendE

cy_restriction_rw
    ├── friendF
    └── cyin026

cy_private
    └── cyin026
```

## 过程
### 1.创建五个用户组并检查
```
sudo groupadd cy_public_ro
sudo groupadd cy_public_rw

sudo groupadd cy_restriction_ro
sudo groupadd cy_restriction_rw

sudo groupadd cy_private_rw

getent group | grep "^cy_"
```

