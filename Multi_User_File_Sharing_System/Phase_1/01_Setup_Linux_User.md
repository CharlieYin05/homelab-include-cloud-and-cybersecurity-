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

Primary Group（gid）：            ← cyin026 所创建的文件/文件夹自动属于cyin026 文件 Group
    cyin026            

Supplementary Groups（groups）：  ← cyin026 被手动分配到的 Groups
    cy_public_rw                ← cy_public_rw 即是用户group，也是文件group，相当于标签被用户和文件引用两次
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
- Charlie 创建 `movie.mp4` 该文件默认：
```
Owner = charlie
Group = charlie（charlie的Primary Group)
```
之后哪怕Bob的Supplementary Groups里有cy_public_rw他也无法读写 `movie.mp4`。因为该文件的group只继承charlie的primary group。

如果有 SGID：
- Charlie 创建 `movie2.mp4` 该文件默认：
```
Owner = charlie
Group = cy_public_rw
```
Bob能打开。

### 第三层：ACL —— 基础规则不够用了
chmod 最大限制每个文件只能有一个 Group。ACL 则允许为多个用户或多个组分别指定权限。

没有 ACL 的文件例子：
```
Owner = Alice
Group = cy_public_rw
权限：
Owner : rw-
Group : rw-
Other : ---
```
如果除了 `cy_public_rw` 有rw权限，还希望 `cy_public_rw` 的用户有r权限 chmod做不到。

有 ACL 的文件例子
```
report.docx
Owner = Alice
Group = cy_public_rw
ACL = group:cy_public_ro, group:auditor, user:jack
```
就能实现更多的用户/用户组能对该文件有独特权限



### 第四层：Default ACL
文件夹本体有了ACL，但是文件夹里面的子文件夹/文件怎么？就需要继承一个主文件的ACL

