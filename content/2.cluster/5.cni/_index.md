---
title: "部署网络插件"
date: 2023-05-28T15:56:43+08:00
draft: false
weight: 50
---

安装好集群后，使用`kubectl get node` 查看节点状态，会显示NotReady，这是由于集群还没有初始化网络。所以需要安装一个网络插件。

## 安装网络插件

可选Flannel和Calico，以及最近非常火的cilium。

1. Flannel是一个非常轻量级的网络插件，截至 2023.05.28，只支持IPv4网络。
2. Calico支持 `IPv4 + IPv6` 双栈
3. Cilium是使用EBPF技术的网络套件，功能丰富。





### 常见网络插件对比

| 功能          | calico                     | flannel        | cilium                               | fabric                                |
| ------------- | -------------------------- | -------------- | ------------------------------------ | ------------------------------------- |
| underlay模式  | 是，依赖BPG                | 否             | 是，依赖BPG                          | 是，不依赖BGP但需要提供独立网络       |
| overlay模式   | vxlan、ipip                | vxlan          | vxlan                                | vxlan、gre、geneve、stt               |
| IPv4+IPv6双栈 | 是                         | 否             | 否                                   | 是                                    |
| 网络性能损耗  | overlay 20%<br>underlay 3% | overlay 20%    | overlay 20%<br/>underlay 3%          | overlay 20%<br/>underlay 3%           |
| DPDK          | 不支持                     | 不支持         | 不支持                               | 支持                                  |
| 网络安全策略  | NetworkPolicy              | 无             | NetworkerPolicy、基于EBPF的L4/L7策略 | 租户隔离、NetworkPolicy、Pod-Security |
| 可视化        | 商业版支持                 |                | Hubble                               | 支持                                  |
| ServcieProxy  | 依赖kube-proxy             | 依赖kube-proxy | 基于EBPF实现                         | 基于OVS流表实现                       |




## 查看规则

- vxlan: bridge fdb