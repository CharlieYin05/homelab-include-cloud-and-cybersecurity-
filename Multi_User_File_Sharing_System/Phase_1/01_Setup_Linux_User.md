# 创建与设置Linux用户

**2026-07-27**

---

## 权限模型
### 第一层（固定）：RBAC

| 共享目录          | 角色组                    | 权限         |
| ---------------- | ------------------------ | ----------- |
| `public`         | `cy_public_ro`           | 只读         |
| `public`         | `cy_public_rw`           | 读写         |
| `restriction`    | `cy_restriction_ro`      | 只读         |
| `restriction`    | `cy_restriction_rw`      | 读写         |
| `private`        | `cy_private_rw`          | 读写（管理员） |

#### 分组：
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

### 第二层（灵活）：ACL
用 ACL 管理共享目录里面的特殊子目录。
例如：
```
restriction/
├── Bob_and_Charles_only
├── Alice_only
├── Finance
└── Project_X
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

### 2.创建共享目录
```
sudo mkdir -p /srv/storage/shares/public
sudo mkdir -p /srv/storage/shares/restriction
sudo mkdir -p /srv/storage/shares/private
```

### 3. 设置Owner

#### 确定文件夹Onwer
| 目录          | Owner   | Group            |
| ----------- | ------- | ----------------- |
| public      | root    | cy_public_rw      |
| restriction | root    | cy_restriction_rw |
| private     | 我      | cy_private_rw     |

```
sudo chown root:cy_public_rw /srv/storage/shares/public
sudo chown root:cy_restriction_rw /srv/storage/shares/restriction
sudo chown cyin026:cy_private_rw /srv/storage/shares/private
```

### 4. 先给三个文件夹设置基础 chmod
以后所有新文件都会自动继承目录所属 Group
```
2770
│││└── Other（其他用户）
││└── Group（组）
│└── Owner（拥有者）
└── 特殊权限（Special SGID）
```

当用户访问一个文件时，Linux 大致会按这个逻辑判断：

1. Owner（是不是Owner？是用Owner权限）
    │
    否
    ▼
2. ACL（有没有 ACL 针对这个用户？是使用 ACL）
    │
    否
    ▼
3. Group（是不是属于 Group？是使用Group权限
    │
    否
    ▼  
4. Other（使用 Other 权限）

```
sudo chmod 2770 /srv/storage/shares/public
sudo chmod 2770 /srv/storage/shares/restriction
sudo chmod 2770 /srv/storage/shares/private
```
