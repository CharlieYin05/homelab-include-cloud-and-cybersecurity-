# 小米Play变废为宝计划

## 背景
  - 手机型号：小米Play
  - 系统：安卓8.1（MIUI v11）
  - 手机现状：手机屏幕漏液损坏，目前右边30%无法显示，按键一切正常，电池寿命未知

## 目标
  1. 无头ADB控制
  2. 导出相片
  3. 获取Root权限
  4. 打造成nmap网络探针

## 技术栈
  - ADB
  - scrcpy
  - termux
  - nmap

## 无头ADB控制（达成）
  1. 手机开启USB调试
  2. mac安装Android Debug Bridge
    `brew install --cask android-platform-tools`
  3. mac安装安卓投屏与控制工具
     `brew install scrcpy`

### 遇到的问题1：scrpy只有画面没有输入指令
```
charlieyin@mac ~ % adb shell
lotus:/ $ whoami
shell
lotus:/ $ id
uid=2000(shell) gid=2000(shell) groups=2000(shell),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid) context=u:r:shell:s0
lotus:/ $ which input
/system/bin/input
lotus:/ $ input keyevent 3
Killed 
137|lotus:/ $ 
```
  - 怀疑1：/system/bin/input已损坏（已排除）
  - 怀疑2：系统在杀ABD进程（MIUI安全策略）
    ```
    W InputDispatcher: Permission denied: injecting event
    W InputManager: Input event injection from pid 31814 permission denied.

    java.lang.SecurityException:
    Injecting to another application requires INJECT_EVENTS permission
    ```
    解决方法：登陆小米账号，开启 USB 调试（安全设置）

## 导出手机照片
  1. mac创建目标文件夹
  `mkdir -p "/Users/charlieyin/Desktop/Mi-Play"`
  2. 备份安卓DCIM文件（里面包括相机拍摄照片）
  `adb pull /sdcard/DCIM ~/Desktop/Mi-Play/`
  3. 备份安卓Pictures（里面包括微信下载到本地照片）
  `adb pull /sdcard/Pictures ~/Desktop/Mi-Play/`

  最终文件结构：
  ```
  ~/Desktop/Mi-Play/
  ├── DCIM/
  └── Pictures/
    ├── WeiXin/
    └── ...
  ```

## Root
  1. 解锁BL（找闲鱼万能的闲鱼老哥）
  2. 救砖准备工作
     2.1 确认当前ROM
     ```
     charlieyin@mac ~ % adb shell getprop ro.build.fingerprint
     xiaomi/lotus/lotus:8.1.0/O11019/V11.0.8.0.OFICNXM:user/release-keys
     charlieyin@mac ~ % adb shell getprop ro.build.version.incremental
     V11.0.8.0.OFICNXM
     charlieyin@mac ~ % adb shell getprop ro.product.device
     lotus
     ```

     

    
