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
  - Magisk Patch
  - termux
  - nmap

## 无头ADB控制（达成）
  1. 手机开启USB调试
  2. mac安装Android Debug Bridge
      `brew install --cask android-platform-tools`
  3. mac安装安卓投屏与控制工具
       `brew install scrcpy`
  4. 确认adb是否正常
       `adb devices`
  5. 开启投屏操控手机
       `scrcpy`

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
  2. 备份安卓DCIM文件夹（里面包括相机拍摄照片）
  `adb pull /sdcard/DCIM ~/Desktop/Mi-Play/`
  3. 备份安卓Pictures文件夹（里面包括微信下载到本地照片）
  `adb pull /sdcard/Pictures ~/Desktop/Mi-Play/`

  最终电脑文件结构：
  ```
  ~/Desktop/Mi-Play/
  ├── DCIM/
  └── Pictures/
    ├── WeiXin/
    └── ...
  ```

## Root
  1. 解锁BL（去闲鱼找万能的暴躁技术老哥半小时解决，准备Windows电脑,数据线和UU远程桌面）
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
     2.2 找到 V11.0.8.0.OFICNXM 官方 Fastboot ROM（线刷包）并下载
       `https://xiaomirom.com/download/mi-play-lotus-stable-V11.0.8.0.OFICNXM/`
     2.3 解压下载的线刷压缩包
       `tar -xzf lotus_images_V11.0.8.0.OFICNXM_20201120.0000.00_8.1_cn_800a08faa8.tgz`
     2.4 确认boot.img，大小应在30~60MB左右，然后复制一份
       ```
       charlieyin@mac ~/Documents/Mi_Play_ROM/lotus_images_V11.0.8.0.OFICNXM_20201120.0000.00_8.1_cn % find images -name boot.img
       images/boot.img

       charlieyin@mac ~/Documents/Mi_Play_ROM/lotus_images_V11.0.8.0.OFICNXM_20201120.0000.00_8.1_cn % ls -lh images/boot.img
       -rw-r--r--@ 1 charlieyin  staff    32M 20 Nov  2020 images/boot.img

       charlieyin@mac ~/Documents/Mi_Play_ROM/lotus_images_V11.0.8.0.OFICNXM_20201120.0000.00_8.1_cn % cp images/boot.img ../Backup/original_boot.img

       charlieyin@mac ~/Documents/Mi_Play_ROM/lotus_images_V11.0.8.0.OFICNXM_20201120.0000.00_8.1_cn % ls -lh ../Backup/original_boot.img
       -rw-r--r--@ 1 charlieyin  staff    32M 11 Jul 20:35 ../Backup/original_boot.img
       ```
       之后如果Magisk有问题理论上救砖：`fastboot flash boot original_boot.img`
     2.5 保存保存 boot 的 SHA256已用于后续确认boot文件是否损坏
       `shasum -a 256 ../Backup/original_boot.img | tee ../Backup/original_boot.sha256`
     2.6 手机进入Fastboot
       `adb reboot bootloader`
     2.7 保存 Fastboot 信息
       `fastboot getvar all 2>&1 | tee ~/Documents/Mi_Play_ROM/Backup/fastboot-info.txt`
     
     
       
     
       
     
     

     

    
