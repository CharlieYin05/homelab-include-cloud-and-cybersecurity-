# Create Linux and Samba User

## Create Linux User
```
sudo useradd -M -s /usr/sbin/nologin <USER_NAME>
```

## Create User Password
```
sudo passwd <USER_PWD>
```

## Assign User to Group
```
sudo usermod -aG \
<GROUP_NAME>,<GROUP_NAME>,\
<GROUP_NAME>,<GROUP_NAME> \
<USER_NAME>
```

## Check User's Current Group
```
id <USER_NAME>
```

## Create Samba account for the User.
```
sudo smbpasswd -a <USER_NAME>
```

## Check whether the user has been added to the Samba database.
```
sudo pdbedit -L
```
