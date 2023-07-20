---
title: "准备节点"
date: 2023-05-28T15:30:02+08:00
draft: false
weight: 10
---

**一键Prepare脚本：for ubuntu 23.04**

```bash
#!/bin/bash
set -e

# 网络转发
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay && modprobe br_netfilter

cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.default.forwarding = 1
vm.swappiness = 0
EOF

sysctl -p /etc/sysctl.d/k8s.conf

# 关闭Swap
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab && sysctl -w vm.swappiness=0

# 设置PS1
sed -i '/^export PS1/d' ~/.bashrc
sed -i 's@ls -alF@ls -lF@' ~/.bashrc
cat >> ~/.bashrc << EOF
export PS1="[\[\033[01;32m\]\u❄\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\]]☭ "
EOF
source ~/.bashrc

# 设置软件源
cat > /etc/apt/sources.list << EOF
deb https://mirrors.ustc.edu.cn/ubuntu-ports/ lunar main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu-ports/ lunar-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu-ports/ lunar-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu-ports/ lunar-security main restricted universe multiverse
EOF

# 安装软件
apt update -y && apt install -y iptables ipvsadm iproute2 jq apt-transport-https net-tools

###################################################################################################
# 安装Kubernetes 1.23
###################################################################################################
apt install -y docker.io

# 设置docker配置
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://docker.mirrors.sjtug.sjtu.edu.cn"]
}
EOF

# 调整docker配置
sed -i 's@ExecStart=/usr/bin/dockerd -H tcp://127.0.0.1:5050@ExecStart=/usr/bin/dockerd@' /usr/lib/systemd/system/docker.service
sed -i 's@ExecStart=/usr/bin/dockerd@ExecStart=/usr/bin/dockerd -H tcp://127.0.0.1:5050@' /usr/lib/systemd/system/docker.service

# 随系统启动
systemctl daemon-reload && systemctl enable docker && systemctl restart docker

###################################################################################################
# 安装Kubernetes 1.23
###################################################################################################

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

# 设置软件源
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt update -y && apt install -y kubelet=1.23.17-00 kubeadm=1.23.17-00 kubectl=1.23.17-00
systemctl enable kubelet && systemctl start kubelet

```

**编译环境：**
```bash
#!/bin/bash
set -e
ARCH=amd64

# Golang 1.20.6
if [ "$(uname -i)" = "aarch64" ]; then ARCH=arm64; fi
wget -O- https://go.dev/dl/go1.20.6.linux-$ARCH.tar.gz | tar xz -C /tmp/
cp -af /tmp/go/* /usr/
cat >> ~/.bashrc << EOF
export GOPATH=/workspace
export GOPRIVATE=*.jd.com
export GOPROXY=goproxy.cn.direct
EOF
source ~/.bashrc
rm -fr /tmp/go

# Helm 3.12.2
wget -O- https://get.helm.sh/helm-v3.12.2-linux-$ARCH.tar.gz | tar xz -C /tmp
cp -af /tmp/linux-$ARCH/helm /usr/bin/
rm -fr /tmp/linux-$ARCH

# K9S 0.27.4
mkdir -p /tmp/k9s
wget -O- https://github.com/derailed/k9s/releases/download/v0.27.4/k9s_Linux_$ARCH.tar.gz | tar xz -C /tmp/k9s
cp -af /tmp/k9s/k9s /usr/bin/
rm -fr /tmp/k9s

# build-essential
apt update -y && apt install -y build-essential universal-ctags cscope

# EBPF & Perf
apt install -y clang llvm bpftrace linux-tools-common linux-tools-generic

# Kernel
apt install -y flex bison

# RUST
apt install -y rustc cargo
```