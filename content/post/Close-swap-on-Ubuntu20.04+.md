---
title: "Close Swap on Ubuntu20.04"
date: 2021-05-25T13:55:24+08:00
draft: false
tags:
    - skill
categories:
    - Ubuntu
    - Linux
---
# 临时关闭
```sh
sudo dphys-swapfile swapoff
```
# 永久关闭
```sh
sudo systemctl disable dphys-swapfile.service --now
# 注释fstab
#/dev/mapper/vgubuntu-swap_1 none            swap    sw              0       0
```

