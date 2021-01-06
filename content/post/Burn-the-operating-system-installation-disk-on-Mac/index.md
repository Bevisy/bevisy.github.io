---
title: "Burn the Operating System Installation Disk on Mac"
date: 2021-01-06T19:22:16+08:00
image: "macbook-wallpaper.jpg"
tags: 
    - macbook
    - iso
    - linux
categories:
---
# 在Macbook上烧录iso镜像，制作操作系统盘

## 查看烧录U盘
```shell script
diskutil list # 例如：/dev/disk3
```

## 取消挂载U盘
```shell script
diskutil unmountDisk /dev/disk3
```

## 烧录
```shell script
# 以 ubuntu-20.04.iso 为例
sudo dd if=~/Downloads/ubuntu-20.04.iso of=/dev/disk3 bs=1m && sync # sync 作用: 使缓存落盘
```

## 弹出U盘
```shell script
diskutil eject /dev/disk3
```