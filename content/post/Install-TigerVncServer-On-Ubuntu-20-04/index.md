---
title: "Install TigerVncServer On Ubuntu 20.04"
date: 2020-07-31T11:47:14+08:00
draft: false
tags: 
    - vnc
categories:
    - Ubuntu
---

# Ubuntu 20.04 安装 Tigervncserver

``` sh
# 安装
$ sudo apt update
$ sudo apt install xfce4 xfce4-goodies tigervnc-standalone-server # 推荐使用 xfce4 桌面

# 启动
$ vncpasswd # 配置 vnc 客户端连接密码
$ vncserver
$ vncserver -kill :1 # 关闭创建的 vnc server

# 修改配置，解决黑屏和灰屏问题
$ touch $HOME/.vnc/xstartup
$ chmod u+x $HOME/.vnc/xstartup
$ cat > $HOME/.vnc/xstartup << EOF
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
xterm -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
startxfce4 &
EOF
# 再次启动
$ vncserver :1 --geomotry 1366x768 # --geometry 参数随意

# 使用 vnc viewer 连接，端口号为 5091
```

