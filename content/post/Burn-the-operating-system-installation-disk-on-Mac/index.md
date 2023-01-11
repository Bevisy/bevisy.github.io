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

# 在 Macbook 上烧录 iso 镜像，制作操作系统盘

## 查看烧录 U 盘

```shell script
diskutil list # 例如：/dev/disk3
```

## 取消挂载 U 盘

```shell script
diskutil unmountDisk /dev/disk3
```

## 烧录

```shell script
# 以 ubuntu-20.04.iso 为例
sudo dd if=~/Downloads/ubuntu-20.04.iso of=/dev/disk3 bs=1m && sync # sync 作用: 使缓存落盘
```

## 查看 dd 烧录进度

```shell script
# 查看 dd 进程
ps -ef | grep -i dd
0   188   185   0  2:33PM ttys000    0:12.33 dd if=/Users/xx/Downloads/ubuntu-20.04.5-desktop-amd64.iso of=/dev/disk2 bs=1m
# 查看烧录进度
sudo kill -SIGINFO 188
```

## 弹出 U 盘

```shell script
diskutil eject /dev/disk3
```
