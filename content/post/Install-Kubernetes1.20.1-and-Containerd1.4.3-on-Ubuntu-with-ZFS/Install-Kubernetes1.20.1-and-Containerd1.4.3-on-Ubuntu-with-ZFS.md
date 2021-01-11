---
title: "Install Kubernetes1.20.1 and Containerd1.4.3 on Ubuntu with ZFS"
description: 基于ZFS的Ubuntu20.04安装kubernetes1.20
date: 2021-01-11T13:56:02+08:00
draft: false
tags:
    - containerd
    - kubernetes
    - kubeadm
categories:
    - Ubuntu
    - Linux
---

# 基于ZFS的Ubuntu20.04安装kubernetes1.20

## containerd 安装

```sh
containerd containerd.io 1.4.3 269548fa27e0089a8b8278fc4fc781d7f65a939b
```

### 安装

```sh
# 安装必要组件
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
# 安装gpg key
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# 验证gpg key
$ sudo apt-key fingerprint 0EBFCD88
# 添加 repo
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
# 安装containerd
$ sudo apt-get update
$ sudo apt-get install containerd.io
```

[安装参考](https://docs.docker.com/engine/install/ubuntu/)

### 配置修改适配 zfs

查看 zfs 插件

```sh
$ sudo ctr plugins ls  | grep zfs                               
io.containerd.snapshotter.v1    zfs                      linux/amd64    ok
```

准备config.toml

```sh
$ sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bk
$ sudo chmod 646 /etc/containerd/config.toml
$ sudo containerd config default > /etc/containerd/config.toml
```

修改`snapshotter`为`zfs`

```sh
[plugins."io.containerd.grpc.v1.cri".containerd]
      #snapshotter = "overlayfs"
      snapshotter = "zfs"
```

重启`containerd`，`containerd`不支持配置热加载

```sh
$ sudo systemctl restart containerd
```

### 挂载 zfs 到containerd指定目录

#### 查询 pool

```sh
$ sudo zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
rpool   460G  8.64G   451G        -         -     0%     1%  1.00x    ONLINE  -
```

#### 挂载目录

```sh
$ sudo zfs create -o mountpoint=/var/lib/containerd/io.containerd.snapshotter.v1.zfs rpool/containerd
```

#### 查看挂载的目录

```sh
$ sudo zfs list
NAME                                               USED  AVAIL     REFER  MOUNTPOINT
...
rpool/containerd                                    96K   437G       96K  /var/lib/containerd/io.containerd.snapshotter.v1.zfs
```

### 验证

#### 拉取镜像

```sh
$ sudo ctr i pull --snapshotter=zfs docker.io/library/nginx:1.17
```

#### 启动容器

```sh
$ sudo ctr run --rm --net-host -d --snapshotter=zfs docker.io/library/nginx:1.17 test
```

#### 查看nginx启动结果

```sh
$ sudo ctr t ls
TASK    PID        STATUS    
test    1494537    RUNNING
$ netstat -alptn | grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
$ curl http://127.0.0.1:80 # 正常可访问
<!DOCTYPE html>
...
<p><em>Thank you for using nginx.</em></p>
</html>
```

### 待解决的问题

#### 问题 1

非root账户，sudo权限不指定参数`snapshotter`创建container，提示`ctr: failed to mount /tmp/containerd-mount589724034: invalid argument`，无法创建容器。

切换成root用户，依旧提示，但是容器可以正常创建

```sh
(root)$ ctr run --rm --net-host -d docker.io/library/nginx:1.17 test
ctr: failed to mount /tmp/containerd-mount589724034: invalid argument
(root)$ ctr t ls
TASK    PID        STATUS    
test    1494537    RUNNING
```

## kubernetes 安装

```sh
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.1", GitCommit:"c4d752765b3bbac2237bf87cf0b1c2e307844666", GitTreeState:"clean", BuildDate:"2020-12-18T12:09:25Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.1", GitCommit:"c4d752765b3bbac2237bf87cf0b1c2e307844666", GitTreeState:"clean", BuildDate:"2020-12-18T12:00:47Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
```

## 安装

当前安装工具有很多，此处选择使用`kubeadm`

### 准备kubernetes 源(使用阿里镜像源)，并安装必要组件

```sh
sudo apt-get update
sudo apt-get install apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
sudo cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install kubelet kubeadm kubectl
```

安装完成后 kubelet 服务会处于不断重启状态，暂时不需要关注，kubelet等待连接kube-apiserver，预期现象。

## 启动kubernetes集群

#### 准备容器镜像

```sh
$ kubeadm config images list # 查看需要准备的镜像
k8s.gcr.io/coredns:1.7.0
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/kube-apiserver:v1.20.1
k8s.gcr.io/kube-controller-manager:v1.20.1
k8s.gcr.io/kube-proxy:v1.20.1
k8s.gcr.io/kube-scheduler:v1.20.1
k8s.gcr.io/pause:3.2
```

#### 启动集群

```sh
$ sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --service-cidr=10.20.0.0/16 --kubernetes-version=v1.20.1 --apiserver-advertise-address 192.168.126.246
```

- --kubernetes-version: 指定 kubernetes 版本；
- --apiserver-advertise-address：指定 kube-apiserver 监听的ip地址；
- --pod-network-cidr：指定 Pod 的网络范围；
- --service-cidr：指定 Service 的网络范围；

#### 验证安装结果

```sh
# 根据提示拷贝kubeconfig文件
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.1", GitCommit:"c4d752765b3bbac2237bf87cf0b1c2e307844666", GitTreeState:"clean", BuildDate:"2020-12-18T12:09:25Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.1", GitCommit:"c4d752765b3bbac2237bf87cf0b1c2e307844666", GitTreeState:"clean", BuildDate:"2020-12-18T12:00:47Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
```

## Flannel 安装

kubernetes v1.17+ 版本

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

需要修改配置：

```sh
net-conf.json: |
    {
      "Network": "10.10.0.0/16", # kubeadm init 创建时使用的pod-network-cidr
      "Backend": {
        "Type": "vxlan"
      }
    }
```

[参考连接](https://github.com/coreos/flannel#flannel)

## 待解决问题

containerd 一直会刷新 error 日志

`Jan 11 17:49:51 zbb-sonypc containerd[1566292]: time="2021-01-11T17:49:51.380076683+08:00" level=error msg="Failed to get usage for snapshot \"sha256:ffc9b21953f4cd7956cdf532a5db04ff0a2daa7475ad796f1bad58cfbaf77a07\"" error="zfs does not implement Usage() yet"`

[参考issue](https://github.com/containerd/zfs/issues/17)