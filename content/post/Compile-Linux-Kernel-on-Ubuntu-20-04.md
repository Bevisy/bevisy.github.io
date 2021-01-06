---
title: "Compile Linux Kernel on Ubuntu 20.04"
date: 2020-07-24T00:41:59+08:00
draft: false
tags: 
    - Kernel
categories:
    - Linux
    - Ubuntu
---

# Linux 内核编译(Ubuntu 环境)

## 下载内核代码

`https://www.kernel.org/`

## 安装依赖

```sh
sudo apt update
sudo apt install -y \
	gcc \
	make \
	libncurses5-dev \
	openssl \
	libssl-dev \
	build-essential \
	pkg-config \
	libc6-dev \
	bison \
	flex \
	libelf-dev
```

## 准备.config和自定义配置

```sh
cd {Download}/linux-5.4.32/
sudo cp /boot/config-{uname -r} .config
sudo make menuconfig
```

## 编译内核

```sh
# 编译内核
sudo make
sudo make modules_install
```

## 安装内核

```sh
sudo mv  {Download}/linux-5.4.32  /usr/src/
cd /usr/src/linux-5.4.32/
sudo make install
sudo mkinitramfs -o /boot/initrd.img-5.4.32
sudo update-initramfs -c -k 5.4.32
sudo update-grub2
```

## 验证

```sh
sudo reboot
uname -a
```


