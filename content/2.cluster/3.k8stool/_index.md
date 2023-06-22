---
title: "安装kubernetes管理工具"
date: 2023-05-28T15:55:03+08:00
draft: false
weight: 30
---

# 安装kubernetes管理工具

需要安装的工具包括：kubeadm、kubelet、kubectl

由于国内环境，这里采用aliyun的源进行安装指定1.23.15-0版本

[阿里云Kubernetes镜像配置方法](https://developer.aliyun.com/mirror/kubernetes/?spm=a2c6h.25603864.0.0.2619274fzqz5ya)

```bash
# For Ubuntu
$ apt-get update && apt-get install -y apt-transport-https
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet=1.23.15-00 kubeadm=1.23.15-00 kubectl=1.23.15-00
$ systemctl enable kubelet && systemctl start kubelet


# 谷歌源
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

```

```bash
# For CentOS
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
$ setenforce 0
$ yum install -y kubelet-1.23.15-0 kubeadm-1.23.15-0 kubectl-1.23.15-0
$ systemctl enable kubelet && systemctl start kubelet
```