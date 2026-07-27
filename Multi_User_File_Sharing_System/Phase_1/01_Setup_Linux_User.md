# 创建与设置Linux用户

**2026-07-27**

---

## 权限模型
Role-based access control（基于角色的访问控制）
### 第一层：共享范围
```
public
restriction
private
```
### 第二层：读写权限
```
RO
RW
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

### 1.创建我的共享组(cyshare)
```
sudo groupadd cyshare
```

### 2.创建read write和read only用户
```
sudo adduser rwuser
sudo adduser rouser
```

### 3.把rw和ro用户加入cyshare共享组并检查结果
```
sudo usermod -aG cyshare rwuser
sudo usermod -aG cyshare rouser

id rwuser
id rouser
getent group cyshare
```

### 4.
