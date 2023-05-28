---
title: "首页"
date: 2023-05-28T15:31:19+08:00
draft: false
---

# 这里介绍安装Kubernetes环境的方法。

**建议：**

| 环境        | 单机版建议                                         | 搭建原生集群建议                                             |
| ----------- | -------------------------------------------------- | ------------------------------------------------------------ |
| Windows     | 个人环境：DockerDesktop<br>商业环境：-             | 建议1：vagrant、hyperV 组合<br>建议2：vagrant、busybox 组合  |
| MacOS       | 个人环境：DockerDesktop<br>商业环境：colima        | 建议1：multipass + qemu<br>建议2：vagrant + qemu `限IntelCPU` |
| Linux云主机 | 直接上集群版，单机也能跑                           | 直接安装吧                                                   |
| Linux物理机 | 建议1： colima<br>建议2： kind<br>建议3： minikube | 直接安装吧                                                   |

