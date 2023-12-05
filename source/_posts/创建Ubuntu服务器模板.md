---
title: 创建Ubuntu服务器模板
date: 2023-10-20 14:41:45
tags:
  - Linux
  - Server
categories:
  - 杂项
---

## 添加 sysctl 参数

### fs 参数

```shell
cat > /etc/sysctl.d/99-fs.conf <<EOF
# 最大文件句柄数
fs.file-max=1048576
# 最大文件打开数
fs.nr_open=1048576
# 同一时间异步IO请求数
fs.aio-max-nr=1048576
EOF
```

### vm 参数

```shell
cat > /etc/sysctl.d/99-vm.conf <<EOF
# 内存耗尽才使用swap分区
vm.swappiness=10
# 当内存耗尽时，内核会触发OOM killer根据oom_score杀掉最耗内存的进程
vm.panic_on_oom=0
# 允许overcommit
vm.overcommit_memory=1
# 定义了进程能拥有的最多内存区域，默认65536
vm.max_map_count=262144
EOF
```

<!-- more -->

### 修改 Limits

```shell
cat > /etc/security/limits.d/99-ubuntu.conf <<EOF
* - nproc 1048576
* - nofile 1048576
root - nproc 1048576
root - nofile 1048576
EOF
```

### 设置 Journal

```shell
sed -e 's,^#Compress=yes,Compress=yes,' \
    -e 's,^#SystemMaxUse=,SystemMaxUse=5G,' \
    -e 's,^#Seal=yes,Seal=yes,' \
    -e 's,^#RateLimitBurst=1000,RateLimitBurst=5000,' \
    -i /etc/systemd/journald.conf

```

### 更换国内源

/etc/apt/sources.lis 文件最前面添加

```c
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
```

```shell
sudo apt-get update
sudo apt-get upgrade -y
```

## 修改 History 配置

```shell
cat > /etc/profile.d/history.sh <<EOF
export HISTSIZE=10000
export HISTFILESIZE=10000
export HISTCONTROL=ignoredups
export HISTTIMEFORMAT="`whoami` %F %T "
export HISTIGNORE="ls:pwd:ll:ls -l:ls -a:ll -a"
EOF
```

### 设置时区

```shell
sudo timedatectl set-timezone Asia/Shanghai
```

### 安装 Docker

```shell
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
```

设置 docker 配置 /etc/docker/daemon.json

```json
{
  "cgroup-parent": "systemd.slice",
  "data-root": "/data/docker",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65535,
      "Soft": 65535
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 65535,
      "Soft": 65535
    }
  },
  "insecure-registries": [],
  "log-driver": "json-file",
  "log-opts": {
    "max-file": "3",
    "max-size": "100m"
  }
}

完整版：
{
  "cgroup-parent": "systemd.slice",
  "data-root": "/var/lib/docker",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65535,
      "Soft": 65535
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 65535,
      "Soft": 65535
    }
  },
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "insecure-registries": [],
  "log-driver": "json-file",
  "log-opts": {
    "max-file": "3",
    "max-size": "100m"
  },
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5,
  "registry-mirrors": [
    "https://pqbap4ya.mirror.aliyuncs.com"
  ],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

### 设置当前用可以用 docker ```

```shell
sudo usermod -aG docker ${USER}
```

### 创建密钥

```shell
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
touch authorized_keys
cat xxx_rsa.pub > authorized_keys
chmod 600 authorized_keys

#测试
ssh localhost

#这时候也可以考虑加入本地的pub到authorized_keys中

```

## 附可能用到配置

### 设置 static ip

```shell
sudo vim /etc/netplan/01-network-static-config.yaml
```

```yaml
#根据实际dhcp信息修改 ip a
network:
  version: 2
  renderer: networkd
  ethernets:
    ens34:
      dhcp4: no
      addresses: [10.8.7.213/24]
      gateway4: 10.8.7.254
      nameservers:
        addresses: [114.114.114.114]
```

```bash
sudo netplan apply
```
