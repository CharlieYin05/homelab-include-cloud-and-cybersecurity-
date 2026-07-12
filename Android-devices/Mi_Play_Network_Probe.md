# 探针计划

## 目标功能
  - 远程控制
  - nmap局域网端口扫描
  - ping服务器测验吃

## 1. 恢复出厂设置
  - 开机后开启USB调试

## 2. 重新安装Magisk并安装TermuX
  ```
  adb devices

  cd /Users/charlieyin/Documents/Android/Magisk_v30.7

  adb install Magisk-v30.7.apk
  ```
  ### 问题1:恢复出厂设置后无法安装Magisk，执行su后没有反应
    ```
    charlieyin@mac ~ % adb shell
    lotus:/ $ which su
    /sbin/su
    lotus:/ $ su -v
    30.7:MAGISKSU
    lotus:/ $ su
    whomai
    whoami
    id
    ```
    解决方法：卸载开机自带的Magisk，重新安装
  
## 3. 安装tailscale

## 4. 安装nmap

## 5. 安装BusyBox

## 6. 安装Python
