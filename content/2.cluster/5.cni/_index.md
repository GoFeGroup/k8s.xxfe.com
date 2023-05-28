---
title: "部署网络插件"
date: 2023-05-28T15:56:43+08:00
draft: false
weight: 50
---

安装好集群后，使用`kubectl get node` 查看节点状态，会显示NotReady，这是由于集群还没有初始化网络。所以需要安装一个网络插件。

可选Flannel和Calico，以及最近非常火的cilium。

1. Flannel是一个非常轻量级的网络插件，截至 2023.05.28，只支持IPv4网络。
2. Calico支持 `IPv4 + IPv6` 双栈
3. Cilium是使用EBPF技术的网络套件，功能丰富。