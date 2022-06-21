---
title: "Kata Containers 安全容器GPU支持实践"
date: 2022-06-21T23:14:52+08:00
draft: false
tags:
    - kata-containers
categories:
    - Ubuntu
    - Linux
---

# kata-containers 安全容器 GPU 支持实践

本文着重讲述如何在 kata-containers 安全容器中支持 GPU 设备，对安全容器本身的介绍会比较少，如果感兴趣的话，可以参考 [kata-containers 官网](https://katacontainers.io/)。接下来，就是讲述如何一步步在安全容器中支持 GPU 设备，以`Tesla P100 PCIe 16GB`为例。

## 准备阶段
1. 下载官方 release 包
2. 下载官方代码，编译需要使用到的组件`containerd-shim-kata-v2`，`kata-agent`，虚拟机镜像 `kata-containers.img`或者`kata-containers-initrd.img`,虚拟机内核 `vmlinuz`
3. GPU 支持实践

### 准备 release 包
可以从官方的 [release 页面](https://github.com/kata-containers/kata-containers/releases)获取最新的包，此处使用当前最新版本`2.4.2`。
```sh
# 下载
wget https://github.com/kata-containers/kata-containers/releases/download/2.4.2/kata-static-2.4.2-x86_64.tar.xz
# 解压并置于 /opt 目录下
tar xf kata-static-2.4.2-x86_64.tar.xz
mv opt/kata /opt/
```

### 编译必要组件
拉取 [kata-containers 源码](https://github.com/kata-containers/kata-containers.git)
**注意**：源码需要包含此[PR](https://github.com/kata-containers/kata-containers/pull/4358)
```sh
mkdir -p $GOPATH/src/github.com/kata-containers
cd $GOPATH/src/github.com/kata-containers

git clone https://github.com/kata-containers/kata-containers.git
git clone https://github.com/kata-containers/tests.git
# 如果当前用户不是 root 用户，则需要将 tests 仓库切换到 root 用户，否则后续编译镜像需要安装 libseccomp 会失败
sudo chown -R root:root tests
```

#### 编译`containerd-shim-kata-v2`
```sh
# 编译 containerd-shim-v2
make -C $GOPATH/src/github.com/kata-containers/kata-containers/src/runtime containerd-shim-v2
```

#### 编译内核
```sh
# 准备依赖
apt-get install -y --no-install-recommends \
	    bc \
	    bison \
	    build-essential \
	    ca-certificates \
	    curl \
	    flex \
	    git \
	    iptables \
	    libelf-dev \
	    patch
# 切换目录
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/packaging/kernel/
# 下载内核源码；准备 config 文件；
./build-kernel.sh -v 5.15.23 -g nvidia -f setup
# 编译 kernel
./build-kernel.sh -v 5.15.23 -g nvidia build
# 安装 kernel
sudo -E ./build-kernel.sh -v 5.15.23 -g nvidia install
# 安装目录 /usr/share/kata-containers
vmlinuz-5.15.23-92-nvidia-gpu # 所需内核模块
## 或者
vmlinux-5.15.23-92-nvidia-gpu # 所需内核模块

# 制作 linux-header deb 或者 rpm 包，由于接下来将使用 ubuntu 作为基础镜像，所以此处使用 deb 包
cd kata-linux-5.15.23-92
make deb-pkg
## 生成 deb 包
linux-headers-5.15.23-nvidia-gpu_5.15.23-nvidia-gpu-1_amd64.deb
linux-image-5.15.23-nvidia-gpu_5.15.23-nvidia-gpu-1_amd64.deb
linux-libc-dev_5.15.23-nvidia-gpu-1_amd64.deb
```

#### 编译虚拟机镜像
虚拟机镜像分为两种 `initrd` 和 `image`，此处以 `image` 为例。
```sh
# rootfs 构建
export ROOTFS_DIR=${GOPATH}/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder/rootfs
rm -rf ${ROOTFS_DIR}
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder

# 确保 kata-containers/tests 仓库在 $GOPATH 目录下，且属主是 root 用户，否则安装 libseccomp 会失败
script -fec 'sudo -E EXTRA_PKGS="gcc make curl gnupg coreutils apt tar kmod pkg-config libc-dev libc6-dev pciutils" GOPATH=$GOPATH USE_DOCKER=true ./rootfs.sh ubuntu'
```

```sh
# 下载 GPU 驱动
## 根据官网 https://www.nvidia.com/Download/Find.aspx 选择所需 GPU 驱动；以 P100 显卡最新驱动 NVIDIA-Linux-x86_64-515.48.07.run 为例，将下载好的驱动放到目录 ${ROOTFS_DIR}/root
## 同时将编译内核生成的 deb 包防放到目录 ${ROOTFS_DIR}/root

# 编译 GPU 驱动
## 注意：确保本机有 GPU 设备，否则编译会失败；
mount -t sysfs -o ro none ${ROOTFS_DIR}/sys
mount -t proc -o ro none ${ROOTFS_DIR}/proc
mount -o bind,ro /dev ${ROOTFS_DIR}/dev
mount -t devpts none ${ROOTFS_DIR}/dev/pts
mount -t tmpfs none ${ROOTFS_DIR}/tmp

chroot ${ROOTFS_DIR}
export PATH=$PATH:/bin:/usr/local/sbin:/usr/sbin:/sbin

cd /root

# ubuntu dpkg 命令目前处于不可用状态，需要创建一些目录如下：
mkdir /var/lib/dpkg
touch /var/lib/dpkg/status
mkdir /var/lib/dpkg/updates/
mkdir -p /var/lib/dpkg/info/
touch /var/lib/dpkg/info/format-new
mkdir -p /var/cache/apt/archives/partial
mkdir -p /var/log/apt/
mkdir /var/lib/dpkg/alternatives

# 安装之前准备的内核 deb 包
dpkg -i *.deb

# 解压 NVIDIA 驱动包，并编译
chmod +x NVIDIA-Linux-x86_64-515.48.07.run
./NVIDIA-Linux-x86_64-515.48.07.run -x
cd NVIDIA-Linux-x86_64-515.48.07
./nvidia-installer -k 5.15.23-nvidia-gpu

## 编译完成后可清理 root 目录，减少镜像体积
```

```sh
# 安装 nvidia-container-toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
mkdir -p /usr/share/yrings/
mkdir -p /etc/ssl/certs
apt install ca-certificates
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt update
apt install nvidia-container-toolkit
apt clean all

## 准备 hook 文件
mkdir -p /usr/share/oci/hooks/prestart/
touch /usr/share/oci/hooks/prestart/nvidia-container-toolkit.sh
chmod +x /usr/share/oci/hooks/prestart/nvidia-container-toolkit.sh

# /usr/share/oci/hooks/prestart/nvidia-container-toolkit.sh
cat << EOF > /usr/share/oci/hooks/prestart/nvidia-container-toolkit.sh
#!/bin/bash -x

/usr/bin/nvidia-container-toolkit -debug \$@
EOF

# 退出，并清理
exit

umount ${ROOTFS_DIR}/sys
umount ${ROOTFS_DIR}/proc
umount ${ROOTFS_DIR}/dev
umount ${ROOTFS_DIR}/dev/pts
umount ${ROOTFS_DIR}/tmp
```


```sh
# 构建 image
## rootfs
export ROOTFS_DIR=$GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder/rootfs

## image
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/image-builder
script -fec 'sudo -E USE_DOCKER=true ./image_builder.sh ${ROOTFS_DIR}'
## 得到 kata-containers.img

## initrd 可按照如下方式：
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/initrd-builder
script -fec 'sudo -E AGENT_INIT=yes USE_DOCKER=true ./initrd_builder.sh ${ROOTFS_DIR}'
```

至此，全部文件准备完毕。

### 配置 kata-containers 和 containerd 以使用 GPU
```sh
# 配置 containerd
## 配置文件 /etc/containerd/config.toml，新增
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
          runtime_type = "io.containerd.kata.v2"
          # privileged_without_host_devices = true

## 将 containerd-shim-kata-v2 移至可执行目录
cp $GOPATH/src/github.com/kata-containers/kata-containers/src/runtime/containerd-shim-kata-v2 /usr/local/bin/

## 将 kata-containers.img 移至 /usr/share/kata-containers
## 将 vmlinuz-5.15.23-92-nvidia-gpu 移至 /usr/share/kata-containers

## 配置 kata-containers
mkdir -p /etc/kata-containers
cp /opt/kata/share/defaults/kata-containers/configuration.toml /etc/kata-containers/
## 修改配置文件 configuration.toml
kernel = "/usr/share/kata-containers/vmlinuz-5.10.25-85-nvidia-gpu"
image = "/usr/share/kata-containers/kata-containers.img"
enable_iommu = true
hotplug_vfio_on_root_bus = true
pcie_root_port = 1
```

验证  GPU
```sh
# 查看 GPU vfio id
$ sudo lspci -nn -D | grep -i nvidia
0000:d0:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:20b9] (rev a1)
$ BDF="0000:d0:00.0"
$ readlink -e /sys/bus/pci/devices/$BDF/iommu_group
79

# 启动容器验证
$ ctr -n k8s.io run --rm --runtime "io.containerd.kata.v2"  --device /dev/vfio/79 --rm -t "docker.io/nvidia/cuda:11.6.0-base-ubuntu20.04" cuda bash
$ nvidia-smi
Tue Jun 21 10:18:31 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.48.07    Driver Version: 515.48.07    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla P100-PCIE...  Off  | 00000000:02:00.0 Off |                  Off |
| N/A   41C    P0    30W / 250W |      0MiB / 16384MiB |      3%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

至此，我们就可以在安全容器中使用 GPU 设备。

## 参考
1. [NVIDIA-GPU-passthrough-and-Kata](https://github.com/kata-containers/kata-containers/blob/main/docs/use-cases/NVIDIA-GPU-passthrough-and-Kata.md)