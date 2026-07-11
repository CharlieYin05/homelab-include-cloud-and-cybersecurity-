# 小米Play变废为宝计划

## 背景
  - 手机型号：小米Play
  - 系统：安卓8.1（MIUI v11）
  - 手机现状：手机屏幕漏液损坏，目前右边30%无法显示，按键一切正常，电池寿命未知

## 目标
  1. 获取ABD
  2. 导出相片
  3. 获取Root权限
  4. 打造成nmap网络探针

## 技术栈
  - ADB
  - scrcpy
  - termux
  - nmap

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
