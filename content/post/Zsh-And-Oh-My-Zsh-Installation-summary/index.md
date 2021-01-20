---
title: "Zsh-And-Oh My Zsh-Installation summary"
date: 2020-07-18T12:14:33+08:00
draft: false
tags: 
    - Zsh
categories:
    - Linux
---
# zsh 安装总结

## 安装zsh

```sh
# Ubuntu 18.04
sudo apt update
sudo apt install zsh -y
```

## 安装 Oh My Zsh

```sh
# https://ohmyz.sh/#install
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## 插件安装

```sh
# 使用 Oh My Zsh 安装插件；默认安装目录为 $ZSH_CUSTOM/plugins (by default ~/.oh-my-zsh/custom/plugins)
git clone https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# 配置 .zshrc，添加插件列表
plugins=(zsh-autosuggestions
        zsh-syntax-highlighting)

# 启动新终端，查看效果示例
zsh
```