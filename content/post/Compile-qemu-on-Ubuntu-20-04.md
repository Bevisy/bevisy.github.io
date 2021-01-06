---
title: "Compile qemu on Ubuntu 20.04"
date: 2020-07-24T22:21:51+08:00
draft: false
tags: 
    - Qemu
categories:
    - Ubuntu
---
# Compile qemu on Ubuntu 20.04

## 下载源码

```sh
git clone https://git.qemu.org/git/qemu.git
cd qemu
git submodule init
git submodule update --recursive
```

## 编译安装

```sh
./configure
make
```

## 问题

```sh
# ERROR: glib-2.48 gthread-2.0 is required to compile QEMU
$ sudo apt install -y libglib2.0-dev

# ERROR: pixman >= 0.21.8 not present.
#        Please install the pixman devel package.
$ sudo apt install -y libpixman-1-dev
```


