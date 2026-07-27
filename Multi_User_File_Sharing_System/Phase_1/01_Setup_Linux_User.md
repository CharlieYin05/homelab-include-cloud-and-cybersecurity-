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

#### 确定文件夹 Onwer
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

### 4. 设置基础 `chmod`
以后所有 `public_rw` , `restriction_rw` 和 `private` 下新创建的文件和子目录，会自动自动继承目录所属 Group 的权限
```
2770
│││└── Other（其他用户）
││└── Group（组）
│└── Owner（拥有者）
└── 特殊权限（Special SGID）
```

#### 当用户访问一个文件时，Linux 大致会按这个逻辑判断：
```
1. Owner（是不是Owner？是就用Owner权限）
    │
    否
    ▼
2. ACL（有没有 ACL 针对这个用户？是就使用 ACL 权限）
    │
    否
    ▼
3. Group（是不是属于 Group？是就使用 Group 权限）
    │
    否
    ▼  
4. Other（使用 Other 权限）
```

```
sudo chmod 2770 /srv/storage/shares/public
sudo chmod 2770 /srv/storage/shares/restriction
sudo chmod 2770 /srv/storage/shares/private
```

### 5.设置ACL 
#### 5.1 给 `public` 和 `restriction` 额外授权 `cy_public_ro` 和 `cy_restriction_ro` 组可以 读 + 进入目录。
```
sudo setfacl -m g:cy_public_ro:rx /srv/storage/shares/public
sudo setfacl -m g:cy_restriction_ro:rx /srv/storage/shares/restriction
```
#### 5.2 设置给 `public`，`restriction` 目录中的新内容（文件和子目录）自动继承 Default ACL (RX)
以后在 public 下新创建的文件和子目录，会自动拥有 `cy_public_ro` 这条 ACL。
```
sudo setfacl -d -m g:cy_public_ro:rx /srv/storage/shares/public
sudo setfacl -d -m g:cy_restriction_ro:rx /srv/storage/shares/restriction
```

---
## Linux用户文件权限系统组成

### 基础概念：用户和文件都有Group
输入：
```
id cyin026
```
输出：
```
uid=1000(cyin026)
gid=1000(cyin026)
groups=1000(cyin026),1002(cy_public_rw),1004(cy_restriction_rw),1005(cy_private_rw)
```
翻译：
```
用户名：cyin026

Primary Group（gid）：
    cyin026            

Supplementary Groups（groups）：
    cy_public_rw
    cy_restriction_rw
    cy_private_rw
```



### 第一层：chmod（比喻：建筑物的基础规则）
它只认识：
```
Owner
Group
Other
```
例如 `drwxrwx---` 就是：
```
Owner  → rwx
Group  → rwx
Other  → ---
```

### 第二层：SGID —— 自动保持 Group 一致
假设文件夹是：
```
public
```
Group 是：
```
cy_public_rw
```
如果没有 SGID：
- Alice 创建 `movie.mp4` 时该电影只能是和Alice相同的group人访问
- Bob 创建的 `photo.jpg`时该照片只能是和Bob相同的group人访问
有了 SGID：
- 凡事有权限里面写入的人所创建的文件/文件夹自动继承 `public` 的 `group`，也即是 `cy_public_rw`

