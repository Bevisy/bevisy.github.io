---
title: "Install Wechat Client On Ubuntu 20.04"
date: 2020-07-29T11:16:31+08:00
draft: false
tags: 
    - Wechat
categories:
    - Ubuntu
---

# Ubuntu 20.04 安装微信客户端

## clone repo 

[deepin-wine-ubuntu ](https://github.com/bevisy/deepin-wine-ubuntu )

## 执行脚本 

`sudo chmod a+x install_2.8.22.sh && ./install_2.8.22.sh`

## 下载微信deb包，并安装 

[下载地址](https://packages.deepin.com/deepin/pool/non-free/d/deepin.com.wechat/deepin.com.wechat_2.6.8.65deepin0_i386.deb )

### 安装微信： 

`sudo dpkg –i deepin.com.wechat_2.6.8.65deepin0_i386.deb`

### 安装字体：(参考issue：https://github.com/wszqkzqk/deepin-wine-ubuntu/issues/253) 

`sudo apt install fonts-wqy-microhei fonts-wqy-zenhei`