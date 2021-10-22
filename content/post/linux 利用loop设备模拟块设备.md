---
title: "linux 利用loop设备模拟块设备"
date: 2021-04-02T16:37:51+08:00
draft: false
tags:
    - skill
categories:
    - Linux
---

# linux 利用loop设备模拟块设备

## 创建虚拟设备

```sh
losetup --help
 -a 显示所有已经使用的回环设备状态
 -d 卸除回环设备
 -f 寻找第一个未使用的回环设备
 -e <加密选项> 启动加密编码
```

```sh
# 查找第一个未使用的回环设备
losetup -f
# 创建文件
dd if=/dev/zero of=./disk.img bs=4M count=1024
# 将disk.img 虚拟成一个回环设备
losetup -f disk.img
# 查询此设备
losetup -a | grep disk.img
/dev/loop25: [64768]:3296700 (/home/zbb/temp/disk/disk.img)

# 格式化设备
mkfs.ext4 /dev/loop25
# 挂载块设备
mount /dev/loop25 test # test 为自定义目录

# 卸载设备
umount test
losetup -d /dev/loop25
```