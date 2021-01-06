---
title: "Ubuntu 20.04 显示器分辨率模式调整"
date: 2020-07-27T10:24:29+08:00
draft: false
tags: 
categories:
    - Ubuntu
---
# Ubuntu 20.04 显示器分辨率模式调整

## 问题描述

Ubuntu 20.04 提示无法识别显示器，默认以分辨率1024x768(4:3)展示

## 问题解决

添加新的分辨率模式

### 查询输出接口

```sh
$ xrandr
Screen 0: minimum 320 x 200, current 2624 x 900, maximum 16384 x 16384
eDP-1 connected primary 1600x900+0+0 (normal left inverted right x axis y axis) 309mm x 174mm
   1600x900      60.01*+  59.99    59.94    59.95    59.82  
   1440x900      59.89  
...
   360x202       59.51    59.13  
   320x180       59.84    59.32  
DP-1 disconnected (normal left inverted right x axis y axis)
HDMI-1 disconnected (normal left inverted right x axis y axis)
DP-2 connected 1024x768+1600+0 (normal left inverted right x axis y axis) 0mm x 0mm # 注意此行
   1024x768      60.00* 
   800x600       60.32    56.25  
   848x480       60.00  
   640x480       59.94  
HDMI-2 disconnected (normal left inverted right x axis y axis)

```

外界显示器输出接口为 DP-2

### 生成分辨率模式并添加

```sh
$ cvt 1920 1080
# 1920x1080 59.96 Hz (CVT 2.07M9) hsync: 67.16 kHz; pclk: 173.00 MHz
Modeline "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync

# 新建分辨率模式(可以不执行)
$ sudo xrandr --newmode "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
# 添加分辨率模式
$ sudo xrandr --addmode DP-2 "1920x1080_60.00"
$ sudo xrandr --output DP-2 "1920x1080_60.00"
```

## 总结

这里主要是通过修改输出信号来适配外接显示器，实际上外接显示器的型号未识别。


