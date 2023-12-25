---
title: "准备节点"
date: 2023-05-28T15:30:02+08:00
draft: false
weight: 10
bookCollapseSection: true
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
# ipv4
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0

# ipv6
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1

# swap
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
OS=$(grep ^ID= /etc/os-release | awk -F= '{print $2}' | sed 's/"//g' ) 

if [ "$OS" == "ubuntu" ]; then
    OsCode=$(grep VERSION_CODENAME /etc/os-release  | awk -F= '{print $2}')
    if [ "$(uname -i)" == "aarch64" ]; then OsArch="-ports"; fi
    cat > /etc/apt/sources.list << EOF
deb https://mirrors.ustc.edu.cn/ubuntu${OsArch}/ ${OsCode} main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu${OsArch}/ ${OsCode}-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu${OsArch}/ ${OsCode}-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu${OsArch}/ ${OsCode}-security main restricted universe multiverse
EOF

    # 安装软件
    apt update -y && apt install -y iptables ipvsadm iproute2 jq apt-transport-https net-tools ipset

    ###################################################################################################
    # 安装Kubernetes 1.23
    ###################################################################################################
    apt install -y docker.io containerd iproute-tc jq ipvsadm iptables
else 
    # 安装docker-ce
    yum install -y yum-utils
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
fi

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

# containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
systemctl daemon-reload && systemctl enable containerd && systemctl restart containerd

###################################################################################################
# 安装Kubernetes 1.23
###################################################################################################

if [ "$OS" == "ubuntu" ]; then
    curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

    # 设置软件源
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

    apt update -y && apt install -y kubelet kubeadm kubectl
    systemctl enable kubelet && systemctl start kubelet

else
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
    systemctl stop firewalld && systemctl disable firewalld
    yum install -y kubelet kubeadm kubectl
    systemctl enable kubelet && systemctl start kubelet
fi

# crictl
sed -i '/^alias crictl=/d' ~/.bashrc
cat >> ~/.bashrc << EOF
alias crictl='crictl --runtime-endpoint unix:///run/containerd/containerd.sock'
EOF
source ~/.bashrc

# 启用rc.local
if [ "$OS" == "ubuntu" ]; then
    cat > /etc/systemd/system/rc-local.service <<EOF
[Unit]
Description=/etc/rc.local
ConditionPathExists=/etc/rc.local
 
[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99
 
[Install]
WantedBy=multi-user.target
EOF

cat <<EOF >/etc/rc.local
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

exit 0
EOF

    chmod +x /etc/rc.local
    systemctl enable rc-local
    systemctl start rc-local.service
fi
```

**编译环境：**
```bash
#!/bin/bash
set -e

ARCH=amd64
if [ "$(uname -i)" = "aarch64" ]; then ARCH=arm64; fi

# Golang 1.21.5
GOLANG_VERSION=1.21.5
wget -O- https://go.dev/dl/go${GOLANG_VERSION}.linux-$ARCH.tar.gz | tar xz -C /tmp/
cp -af /tmp/go/* /usr/
sed -i '/^export GO/d' ~/.bashrc
cat >> ~/.bashrc << EOF
export GOPATH=/workspace
export GOPRIVATE=*.jd.com
export GOPROXY=goproxy.cn,direct
EOF
source ~/.bashrc
rm -fr /tmp/go

# Helm 3.13.3
HELM_VERSION=v3.13.3
wget -O- https://get.helm.sh/helm-${HELM_VERSION}-linux-$ARCH.tar.gz | tar xz -C /tmp
cp -af /tmp/linux-$ARCH/helm /usr/bin/
rm -fr /tmp/linux-$ARCH

# K9S 0.29.1
K9S_VERSION=v0.29.1
mkdir -p /tmp/k9s
wget -O- https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_$ARCH.tar.gz | tar xz -C /tmp/k9s
cp -af /tmp/k9s/k9s /usr/bin/
rm -fr /tmp/k9s

# build-essential
apt update -y && apt install -y build-essential universal-ctags cscope libssl-dev pkg-config

# EBPF & Perf
apt install -y clang llvm bpftrace linux-tools-common linux-tools-generic libelf-dev libcap-dev libbfd-dev
if [ ! -d /usr/include/asm ]; then ln -s /usr/include/$(uname -i)-linux-gnu/asm /usr/include/asm; fi
if [ ! -d /usr/include/bits ]; then ln -s /usr/include/$(uname -i)-linux-gnu/bits /usr/include/bits; fi
if [ ! -d /usr/include/gnu ]; then ln -s /usr/include/$(uname -i)-linux-gnu/gnu /usr/include/gnu; fi
if [ -d /usr/include/sys ]; then 
  cp -af /usr/include/$(uname -i)-linux-gnu/sys/* /usr/include/sys/;
else
  ln -s /usr/include/$(uname -i)-linux-gnu/sys /usr/include/sys;
fi

# docker buildx
BUILDX_VERSION=v0.12.0
mkdir -p /root/.docker/cli-plugins/
wget https://github.com/docker/buildx/releases/download/${BUILDX_VERSION}/buildx-${BUILDX_VERSION}.linux-${ARCH} -O /root/.docker/cli-plugins/docker-buildx
chmod +x /root/.docker/cli-plugins/docker-buildx

# kubecm 0.25.0
KUBECM_VERSION=v0.25.0
mkdir -p /tmp/kubecm
wget -O- https://github.com/sunny0826/kubecm/releases/download/${KUBECM_VERSION}/kubecm_${KUBECM_VERSION}_Linux_${ARCH}.tar.gz | tar xz -C /tmp/kubecm
cp -af /tmp/kubecm/kubecm /usr/bin/kc
rm -fr /tmp/kubecm

# Kernel
apt install -y flex bison bc pahole

# RUST
apt install -y rustc cargo rustfmt rust-src
```


**VSCode Server:**
```bash
# 从关于中获取CommitID
COMMITID=74f6148eb9ea00507ec113ec51c489d6ffb4b771

ARCH=x64
if [ "$(uname -i)" = "aarch64" ]; then ARCH=arm64; fi

rm -fr /tmp/vscode-server-linux-${ARCH}
wget -O-  https://update.code.visualstudio.com/commit:${COMMITID}/server-linux-${ARCH}/stable | tar xz -C /tmp/

mkdir -p ~/.vscode-server/bin
rm -fr ~/.vscode-server/bin/*
mv /tmp/vscode-server-linux-${ARCH}  ~/.vscode-server/bin/${COMMITID}
```
