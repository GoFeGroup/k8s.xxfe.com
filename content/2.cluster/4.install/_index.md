---
title: "初始化集群"
date: 2023-05-28T15:56:15+08:00
bookCollapseSection: true
draft: false
weight: 40
---

# 初始化集群

## 安装Master-IPv4

```bash
# 使用docker作为容器管理工具，需指定1.23版本，同时指定aliyun镜像仓库
$ kubeadm init --apiserver-advertise-address=192.168.64.6 \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version=v1.23.15 \
    --service-cidr=10.96.0.0/12 \
    --pod-network-cidr=10.244.0.0/16 \
    --v=5
# --apiserver-advertise-address: kube-apiserver对外侦听的地址；
#                                由于kubeadm默认使用eth0的地址作为侦听地址，在某些情况下不适用
# --image-repository: 由于一些不可描述的原因，国内无法下载kubernetes的容器镜像，这里使用阿里云的镜像仓库
# --kubernetes-version=v1.23.15: 指定安装kubernetes的版本，1.23版本是支持docker作为容器支持的
```

## 安装Master-IPv6

如果需要支持IPv6，可在cidr中增加IPv6的地址段:

```bash
kubeadm init --apiserver-advertise-address=192.168.64.6 \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version=v1.23.15 \
    --service-cidr=10.96.0.0/12,2001:db8:41:1::/112 \
    --pod-network-cidr=10.244.0.0/16,2001:db8:42:0::/56 \
    --v=5
```

## 多Master部署

如果需要支持多Master部署，那么需要首先修改集群的kubeadm-config，增加：controlPlaneEndpoint: 192.168.64.6:6443

```yaml
    kind: ClusterConfiguration
    kubernetesVersion: v1.23.15
    controlPlaneEndpoint: 192.168.64.6:6443
```

然后在第二台Master上部署证书

```bash
$ scp /etc/kubernetes/pki/sa.*           \
    /etc/kubernetes/pki/ca.*             \
    /etc/kubernetes/pki/front-proxy-ca.* \
    master02:/etc/kubernetes/pki/
$ scp /etc/kubernetes/pki/etcd/ca.* master02:/etc/kubernetes/pki/etcd/
```

在Master节点获取join token：
```bash
$ kubeadm token create --print-join-command
```

在第二台master上执行Join命令:

```bash
$ kubeadm join 192.168.64.6:6443 --token xxx \
    --discovery-token-ca-cert-hash sha256:xxxx \
    --control-plane 
```

## 跳过kubeproxy

```bash
$ kubeadm init \
  --skip-phases=addon/kube-proxy
```