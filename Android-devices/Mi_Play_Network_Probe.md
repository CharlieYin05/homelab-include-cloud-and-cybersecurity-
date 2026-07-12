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
  ### 问题1：恢复出厂设置后无法安装Magisk，执行su后没有反应
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
  
## 3. 安装termux
  ### 问题2：即使开启USB安装也无法ABD安装termux(疑似MIUI拦截）
  ```
  charlieyin@mac ~/Documents/Android/Termux % adb install termux-app_v0.118.3+github-debug_arm64-v8a.apk
Performing Streamed Install
adb: failed to install termux-app_v0.118.3+github-debug_arm64-v8a.apk: Failure [INSTALL_FAILED_USER_RESTRICTED: Install canceled by user]
```
  解决方法：把APK Push安装到安卓，然后ROOT权限安装
  ```
  charlieyin@mac ~/Documents/Android/Termux % adb push ~/Documents/Android/Termux/termux-app_v0.118.3+github-debug_arm64-v8a.apk /data/local/tmp/termux.apk
/Users/charlieyin/Documents/Android/Termux/termux-app_v0.118.3+... 1 file pushed, 0 skipped. 14.7 MB/s (35106607 bytes in 2.285s)
charlieyin@mac ~/Documents/Android/Termux % adb shell su -c "pm install -r /data/local/tmp/termux.apk"
Success
```
## 4. 安装tailscale
  ```
  adb install tailscale-android-universal-1.98.8.apk
  ```
  然后登陆账号并记住tailscale IP地址
  
## 5. 安装SSH
  进入Termux后：
  ```
  pkg update
  pkg upgrade
  pkg install openssh
  sshd -V
  ```
  记录并设置Termux用户密码
  ```
  whoami

  passwd
  ```

  开启SSH并查看SSH端口：
  ```
  sshd

  ss -ltn
  ```
  mac SSH到手机
  `ssh -p 8022 u0_aXXX@100.XXX.XX.XX`
  ```

## 5. 安装BusyBox

## 6. 安装Python
