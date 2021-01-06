---
title: "Kata-containers Compile And Installed"
date: 2020-07-25T23:06:12+08:00
draft: false
tags: 
    - Kata
categories:
    - KataContainers
---
[TOC]

# kata-containers 编译安装

## kata-runtime 编译安装

```sh
# download source code
$ go get -d -u github.com/kata-containers/runtime
$ cd ${GOPATH}/src/github.com/kata-containers/runtime
# compile and install
$ make
$ make install

# Install Dir
/usr/libexec/kata-containers/kata-netmon
/usr/local/bin/kata-runtime
/usr/local/bin/containerd-shim-kata-v2
/usr/share/defaults/kata-containers/*
```

## kata-shim 编译安装

```sh
# download source code
$ go get -d -u github.com/kata-containers/shim
$ cd ${GOTAH}/src/github.com/kata-containers/shim
# compile and install 
$ makn
$ make install

# Install Dir
/usr/libexec/kata-containers/kata-shim
```

## kata-proxy 编译安装

```sh
# download source code
$ go get -d -u github.com/kata-containers/proxy
$ cd ${GOTAH}/src/github.com/kata-containers/proxy
# compile and install 
$ make
$ make install

# Install Dir
/usr/libexec/kata-containers/kata-proxy
```

## 编译 kata 所需的 kernel

```sh
# download source code
$ go get -d -u github.com/kata-containers/packaging
$ cd ${GOTAH}/src/github.com/kata-containers/packaging/kernel
$ git checkout stable-1.12

# On Ubuntu20.04 should install some essential packages
$ sudo apt install -y \
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
# Also you should install yq from github: https://github.com/mikefarah/yq
# 注意：如果缺少依赖，会导致内核编译所需要的 .config 文件，无法主动生成，可以将 configs/ 和 configs/fragments 目录下对应文件拼接成完整文件。
$ ./build-kernel.sh -v 5.4.60 -f -d setup # stable-1.12 支持的最新LTS内核
# compile kernel
$ ./build-kernel.sh -v 5.4.60 -f -d build

# Output File:
${GOPATH}/src/github.com/kata-containers/packaging/kernel/kata-linux-5.4.60-89/vmlinux

# Install Dir:
/usr/share/kata-containers/vmlinux
```

## 编译 agent  (可选)

```sh
$ go get -d -u github.com/kata-containers/agent
$ cd $GOPATH/src/github.com/kata-containers/agent && make
```



## 编译 rootfs 文件系统

```sh
# stable-1.10 方法
# Download source code
$ go get -d -u github.com/kata-containers/osbuilder

# generate rootfs
$ export ROOTFS_DIR=${GOPATH}/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs
$ sudo rm -rf ${ROOTFS_DIR}
$ cd $GOPATH/src/github.com/kata-containers/osbuilder/rootfs-builder
# ${distro} 需要替换成具体的系统，推荐 centos
# 此处增加额外的包，是为了后续便于进入虚拟机调试
#$ script -fec 'sudo -E GOPATH=$GOPATH USE_DOCKER=true EXTRA_PKGS="bash coreutils" ./rootfs.sh ${distro}'
$ script -fec 'sudo -E GOPATH=$GOPATH USE_DOCKER=true EXTRA_PKGS="bash coreutils vim net-tools procps curl iproute" http_proxy=http://{proxy}:{ip} https_proxy=http://{proxy}:{ip} ./rootfs.sh ${distro}'
# 由于网络原因，可以构建时候添加 http_proxy 代理;
$ script -fec 'sudo -E GOPATH=$GOPATH USE_DOCKER=true EXTRA_PKGS="bash coreutils" http_proxt=http://{IP}:{PORT} ./rootfs.sh ${distro}'

# 开启vm debug（host进入vm）
## 方案一(官方文档目前的方案，支持kata-containers 1.10.7;kata-agent 1.10.7)
## 1.确保rootfs已安装 bash coreutils
## 2./etc/kata-containers/configuration.toml 配置文件修改
$ sudo sed -i -e 's/^kernel_params = "\(.*\)"/kernel_params = "\1 agent.debug_console"/g' "${kata_configuration_file}"
## 3.确保配置文件的[proxy.kata] enable_debug 不为 true （默认不为true）
$ sudo awk '{if (/^\[proxy\.kata\]/) {got=1}; if (got == 1 && /^.*enable_debug/) {print "#enable_debug = true"; got=0; next; } else {print}}' /etc/kata-containers/configuration.toml > /tmp/configuration.toml
## 4.使用 socat 访问，回车可进入bash，虚拟机刚启动会刷新大量日志，等待日志刷新完毕即可

# 方案二（之前使用的方案，也是官方方案，通过添加额外的kata-debug.service，相比方案一qemu无需传递内核参数agent.debug_console，同样确保配置文件的[proxy.kata] enable_debug 不为 true （默认不为true））
# Create a debug systemd service
$ cat <<EOT | sudo tee ${ROOTFS_DIR}/lib/systemd/system/kata-debug.service
[Unit]
Description=Kata Containers debug console

[Service]
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
StandardInput=tty
StandardOutput=tty
# Must be disabled to allow the job to access the real console
PrivateDevices=no
Type=simple
ExecStart=/bin/bash
Restart=always
EOT

# Add a dependency to start the debug console:
$ sudo sed -i '$a Requires=kata-debug.service' ${ROOTFS_DIR}/lib/systemd/system/kata-containers.target

# Output File:
${GOPATH}/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs

# 注意事项
# 1.安全容器开启debug console后，host使用socat访问vm，访问console.sock一次只允许一个连接，如果已存在进入虚拟机的连接，则其它连接均被阻塞
```

```sh
# stable-1.12
# Download source code
$ go get -d -u github.com/kata-containers/osbuilder
$ cd $GOPATH/src/github.com/kata-containers/osbuilder
# 构建rootfs
mkdir -p ${PWD}/myrootfs
export USE_DOCKER=true
export EXTRA_PKGS="vim bash coreutils vim net-tools procps curl iproute"
export AGENT_VERSION="1.12.0"
export AGENT_SOURCE_BIN="$GOPATH/src/github.com/kata-containers/agent" # kata-agent 二进制文件目录
export http_proxy=http://{}
export https_proxy=http://{}
./rootfs-builder/rootfs.sh -r ${PWD}/myrootfs centos # 可根据需求修改 distro
## 如果出现镜像构建在下载github文件卡住，可将对应{distro}目录下的Dockerfile.in模板，直接添加上ENV proxy
# 构建img
# build image based rootfs created above
sudo USE_DOCKER=true ./image-builder/image_builder.sh  "${PWD}/myrootfs"
```

## 编译 rootfs.img

```sh
# stable-1.10 方法
# make sure rootfs is not MODIFIED!!! if you want to add new Agent
# install agent (optional)
$ sudo install -o root -g root -m 0550 -t ${ROOTFS_DIR}/bin ../../agent/kata-agent
$ sudo install -o root -g root -m 0440 ../../agent/kata-agent.service ${ROOTFS_DIR}/usr/lib/systemd/system/
$ sudo install -o root -g root -m 0440 ../../agent/kata-containers.target ${ROOTFS_DIR}/usr/lib/systemd/system/

# Compile
$ cd $GOPATH/src/github.com/kata-containers/osbuilder/image-builder
$ script -fec 'sudo -E USE_DOCKER=true ./image_builder.sh ${ROOTFS_DIR}'

# install
$ commit=$(git log --format=%h -1 HEAD)
$ date=$(date +%Y-%m-%d-%T.%N%z)
$ image="kata-containers-${date}-${commit}"
$ sudo install -o root -g root -m 0640 -D kata-containers.img "/usr/share/kata-containers/${image}"
$ (cd /usr/share/kata-containers && sudo ln -sf "$image" kata-containers.img)

# Output File:
$GOPATH/src/github.com/kata-containers/osbuilder/image-builder/kata-containers.img
```



## 编译 initrd.img

```sh
# make sure rootfs is not MODIFIED!!! if you want to add new Agent
# install agent(optional)
$ sudo install -o root -g root -m 0550 -T ../../agent/kata-agent ${ROOTFS_DIR}/sbin/init

# Compile
$ cd $GOPATH/src/github.com/kata-containers/osbuilder/initrd-builder
$ script -fec 'sudo -E AGENT_INIT=yes USE_DOCKER=true ./initrd_builder.sh ${ROOTFS_DIR}'

# install
$ commit=$(git log --format=%h -1 HEAD)
$ date=$(date +%Y-%m-%d-%T.%N%z)
$ image="kata-containers-initrd-${date}-${commit}"
$ sudo install -o root -g root -m 0640 -D kata-containers-initrd.img "/usr/share/kata-containers/${image}"
$ (cd /usr/share/kata-containers && sudo ln -sf "$image" kata-containers-initrd.img)

# Output File:
$GOPATH/src/github.com/kata-containers/osbuilder/initrd-builder/kata-containers-initrd.img

```

# 编译 qemu on aarch64

```sh
# 下载代码
$ go get -d github.com/kata-containers/tests
# 准备依赖
$ sudo apt install -y libcap-ng-dev libglib2.0-dev libpixman-1-dev librbd-dev libattr1-dev libcap-dev
# 编译构建
$ script -fec 'sudo -E ${GOPATH}/src/github.com/kata-containers/tests/.ci/install_qemu.sh'

# 注意：如果安装失败，清直接删除文件夹，然后重新跑升级脚本
$ sudo rm -rf ${GOPATH}/src/github.com/kata-containers/packaging # 可不操作
$ sudo rm -rf ${GOPATH}/src/github.com/qemu
# 如果需要编译指定代码分支的qemu版本，需要切换tests分支，以及packaging分支为同一分支
# 由于脚本.ci/install_qemu.sh go get 获取packaging代码后未切换分支，所以需要构建失败后，手动切换分支
# 目前已知最新的版本的qemu patch是针对qemu 5.1的，并不能直接使用master代码，会导致patch合入失败
```

# Docker 对接 kata-runtime

修改 Docker 配置文件`/etc/docker/daemon.json`

```sh
{
  "debug": true,
  "default-runtime": "runc", # 可替换成 kata-runtime
  "runtimes": {
    "kata": {
      "path": "/usr/local/bin/kata-runtime" # 不支持直接配置成 containerd-shim-kata-v2
    }
  }
}
```

重启 docker 服务（必须）

验证修改生效

`sudo docker run --rm --name test busybox:latest uname -a` 与宿主机内核对比，验证是否生效

## 调试 kata-runtime

```sh
# docker 开启 debug: /etc/docker/daemon.json 添加参数 (需重启服务)
{ "debug": true }
# kata配置文件/etc/kata-containers/configuration.toml，开启 enable_debug

# 查看日志
$ journalctl -ft kata-runtime
```

# Containerd 对接 containerd-shim-kata-v2

修改 containerd 的配置`/etc/containerd/config.toml`

`containerd config default` 生成当前版本默认配置

```sh
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
plugin_dir = ""
disabled_plugins = []
required_plugins = []
oom_score = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  tcp_address = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[ttrpc]
  address = ""
  uid = 0
  gid = 0

[debug]
  address = ""
  uid = 0
  gid = 0
  level = "debug" # 开启debug

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
  [plugins."io.containerd.grpc.v1.cri"]
    disable_tcp_service = true
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    stream_idle_timeout = "4h0m0s"
    enable_selinux = false
    sandbox_image = "k8s.gcr.io/pause:3.1"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    disable_cgroup = false
    disable_apparmor = false
    restrict_oom_score_adj = false
    max_concurrent_downloads = 3
    disable_proc_mount = false
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"
      default_runtime_name = "runc"
      no_pivot = false
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v1"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata] # 新增
          runtime_type = "io.containerd.kata.v2"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      max_conf_num = 1
      conf_template = ""
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"
  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"
  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.snapshotter.v1.devmapper"]
    root_path = ""
    pool_name = ""
    base_image_size = ""
```

注意： 确保`containerd-shim-kata-v2` 文件在 $PATH  目录下

验证：

```sh
# 创建容器
sudo ctr -n testns run --runtime io.containerd.kata.v2  -d --rm docker.io/library/busybox:latest busybox
# 查看容器 id
sudo ctr -n testns t ls
# 进入容器
sudo ctr -n testns t exec -t --exec-id {ID} busybox sh
# 查看内核版本
$ uname -a # 对比宿主机内核
```

## 调试 containerd-shim-runtime-v2

```sh
# containerd 配置开启 debug(需重启服务)
# kata配置文件/etc/kata-containers/configuration.toml，开启 enable_debug (需重新创建安全容器)

# 查看日志
$ journalctl -ft kata
```

# 附录

## kata-containers 2.0 安装

1. 下载release包

   [Release 地址](https://github.com/kata-containers/kata-containers/releases)

   下载 [Kata Containers 2.0.0-alpha3](https://github.com/kata-containers/kata-containers/releases/tag/2.0.0-alpha3)

   解压后，拷贝至文件夹 `/opt`

2. 准备配置文件和`containerd-shim-kata-v2`

   ```sh
   # 准备配置文件
   $ cp /opt/kata/share/defaults/kata-containers/configuration-qemu.toml /etc/kata-containers/configuration.toml
   # 准备 containerd-shim-kata-v2
   $ cp /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/
   ```

   配置文件修改如下：

   ```sh
   [hypervisor.qemu]
   path = "/opt/kata/bin/qemu-system-x86_64"
   kernel = "/opt/kata/share/kata-containers/vmlinuz.container"
   image = "/opt/kata/share/kata-containers/kata-containers.img"
   machine_type = "pc"
   kernel_params = ""
   firmware = ""
   machine_accelerators=""
   cpu_features="pmu=off"
   default_vcpus = 1
   default_maxvcpus = 2
   default_bridges = 1
   default_memory = 2048
   disable_block_device_use = false
   shared_fs = "virtio-9p"
   virtio_fs_daemon = "/opt/kata/bin/virtiofsd"
   virtio_fs_cache_size = 1024
   virtio_fs_extra_args = []
   virtio_fs_cache = "auto"
   block_device_driver = "virtio-scsi"
   enable_iothreads = false
   enable_vhost_user_store = false
   vhost_user_store_path = "/var/run/kata-containers/vhost-user"
   enable_debug = true
   [factory]
   [agent.kata]
   enable_debug = true
   kernel_modules=[]
   [netmon]
   path = "/opt/kata/libexec/kata-containers/kata-netmon"
   enable_debug = true
   [runtime]
   enable_debug = true
   internetworking_model="tcfilter"
   disable_guest_seccomp=true
   sandbox_cgroup_only=false
   experimental=[]
   EnablePprof = true
   ```

## 对接docker和containerd需要注意

注意：此版本无法与 docker配合使用，与containerd 使用正常。

# 问题探讨

## 1.如果关闭用来开启debug console的kata-debug.service会发生什么？

进入vm，关闭kata-debug.service，等待kata-debug.service最终停止后，重新使用socat无法再进入bash终端

```sh
bash-4.2# systemctl stop kata-debug
[  249.392708] systemd[1]: Failed to determine peer security context: Protocol not available
[  249.394714] systemd[1]: Accepted new private connection.
[  249.395681] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  249.397145] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  249.397455] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  249.397759] systemd[1]: Got message type=method_call sender=n/a destination=org.freedesktop.systemd1 object=/org/freedesktop/systemd1 interface=org.freedesktop.systemd1.Manager member=StopUnit cookie=1 reply_cookie=0 error=n/a
[  249.398545] systemd[1]: Trying to enqueue job kata-debug.service/stop/replace
[  249.398983] systemd[1]: Installed new job kata-containers.target/stop as 51
[  249.399290] systemd[1]: Installed new job kata-debug.service/stop as 50
[  249.399604] systemd[1]: Enqueued job kata-debug.service/stop as 50
Terminated
bash-4.2# systemctl status kata-debug
[  258.097267] systemd[1]: Failed to determine peer security context: Protocol not available
[  258.098268] systemd[1]: Accepted new private connection.
[  258.098453] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  258.098762] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  258.098928] systemd[1]: Got unexpected auxiliary data with level=1 and type=2
[  258.099088] systemd[1]: Got message type=method_call sender=n/a destination=org.freedesktop.systemd1 object=/org/freedesktop/systemd1/unit/kata_2ddebug_2eservice interface=org.freedesktop.DBus.Properties member=GetAll cookie=1 reply_cookie=0 error=n/a
[  258.099783] systemd[1]: Sent message type=method_return sender=n/a destination=n/a object=n/a interface=n/a member=n/a cookie=1 reply_cookie=1 error=n/a
● kata-debug.service - Kata Containers debug console
   Loaded: loaded (/usr/lib/systemd/system/kata-debug.service; static; vendor preset: disabled)
   Active: deactivating (stop-sigterm) since Wed 2020-10-14 08:11:57 UTC; 8s ago
 Main PID: 57 (bash)
   CGroup: /system.slice/kata-debug.service
           ├─ 57 /bin/bash
           └─231 systemctl status kata-debug
[  258.101662] systemd[1]: Got disconnect on private connection.
bash-4.2# system[  339.593226] systemd[1]: kata-debug.service stop-sigterm timed out. Killing.
[  339.593989] systemd[1]: Watching 57 through watch_pids1.
[  339.594710] systemd[1]: kata-debug.service changed stop-sigterm -> stop-sigkill
[  339.595681] systemd[1]: Received SIGCHLD from PID 57 (bash).
[  339.596281] systemd[1]: Child 57 (bash) died (code=killed, status=9/KILL)
[  339.596722] systemd[1]: Child 57 belongs to kata-debug.service
[  339.597060] systemd[1]: Unwatching 57.
[  339.597276] systemd[1]: kata-debug.service: main process exited, code=killed, status=9/KILL
[  339.597722] systemd[1]: kata-debug.service changed stop-sigkill -> failed
[  339.598098] systemd[1]: Job kata-debug.service/stop finished, result=done
# 此处kata-debug.service完全退出，无法再进入终端

# 尝试重新进入终端，会发现回车后无法打开bash终端
root@zbb-pc:/run/vc/vm/d658a67be082b2952c360e7539b5bfa7ac744ae8b84d844d57547d36915e184f# socat "stdin,raw,echo=0,escape=0x11" "unix-connect:./console.sock" 

# 重启容器后，可重新进入终端
# 此处发现重启容器过程，其实是新建一个同名的容器，因为会发现vm原有目录文件均被删除，生成同名文件夹。如果查看前后vm目录，应该文件inode已产生变化。
```

