# 创建与设置Linux用户

**2026-07-27**

---

## 用户模型
| 用户        | SSH | sudo | Samba | 用途    |
| --------- | --- | ---- | ----- | ----- |
| `我` | ✅   | ✅    | 可选    | 系统管理员 |
| `rwuser`  | ❌   | ❌    | ✅     | 读写共享  |
| `rouser`  | ❌   | ❌    | ✅     | 只读共享  |

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

### 把rw和ro用户加入cyshare共享组并检查结果
```
sudo usermod -aG cyshare rwuser
sudo usermod -aG cyshare rouser

id rwuser
id rouser
getent group cyshare
```
