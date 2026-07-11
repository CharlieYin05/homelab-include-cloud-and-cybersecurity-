# 小米Play变废为宝计划

## 背景
  - 手机型号：小米Play
  - 系统：安卓10（miui 11）
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
  ## 问题1.1: 安卓系统不对
    ·adb shell getprop ro.build.version.release 8.1.0·
  ## 
