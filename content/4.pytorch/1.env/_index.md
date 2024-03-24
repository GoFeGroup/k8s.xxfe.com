---
title: "环境@Windows"
date: 2024-03-24T14:45:43+08:00
draft: false
---

# 在Win系统中安装pytorch环境

## WSL2

- 在windows环境里安装wsl2
- 设置默认用户root
- 设置工作目录

## 安装Python3包管理工具

```bash
apt install python3-pip
```


## 创建python虚拟环境

```bash
apt -y install python3-venv
python3 -m venv ~/DeepLearning/
cd ~/DeepLearning/ && source bin/active
```

## 安装torch

```bash
pip3 install torch torchvision torchaudio
```

## 验证
```bash
[root❄DESKTOP-950PPAU:~]☭ python3
Python 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import torch
>>> torch.cuda.is_available()
True
>>>
```
