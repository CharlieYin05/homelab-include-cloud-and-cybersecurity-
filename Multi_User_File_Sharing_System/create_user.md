# 创建 Linux 与 Samba 用户

## 创建 Linux 用户
```
sudo useradd -M -s /usr/sbin/nologin <用户名>
```

## 创建该用户密码
```
sudo passwd <用户名>
```

## 给该用户分组
```
sudo usermod -aG \
<组名>,<组名>,\
<组名>,<组名> \
<用户名>
```

## 检查该用户是否分组成功
```
id <用户名>
```

## 创建该用户的 Samba 账号
```
sudo smbpasswd -a <用户名>
```

## 检查该用户是否进入 Samba DB
```
sudo pdbedit -L
```
