---
title: "准备节点"
date: 2023-05-28T15:30:02+08:00
draft: false
weight: 10
---

**一键Prepare脚本：**

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
cat >> ~/.bashrc << EOF
export PS1="[\[\033[01;32m\]\u❄\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\]]☭ "
EOF

# 设置软件源
cat > /etc/apt/sources.list << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ jammy-backports main restricted universe multiverse
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