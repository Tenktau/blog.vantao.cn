---
title: RevyOS 学习日志
description: 本篇文章旨在分享本人通过 Lichee Pi 4A (16+128 GB) 学习 RevyOS 过程中的心得。
date: 2025-02-18 +0800
author: <vantao>
categories: [Development,Embedded]
tags: [RevyOS,LicheePi4A]
---

## 介绍

- RevyOS：由 RuyiSDK 团队的 RevyOS 小队支持开发的一款针对 XuanTie 芯片生态的 Debian 优化定制发行版。[[了解更多]](https://github.com/revyos/revyos/blob/main/README.cn.md)
- Lichee Pi 4A：使用了 Lichee Module 4A 核心板组合出来的 Linux 开发板，以 TH1520 为主控核心，搭载 4TOPS@int8 AI 算力的 NPU，支持双屏 4K 显示输出，支持 4K 摄像头接入，双千兆 POE 网口和多个 USB 接口，音频由 C906 核心处理。[[了解更多]](https://wiki.sipeed.com/hardware/zh/lichee/th1520/lp4a.html)

## 踩坑记录（按时间顺序）

### 20250219

- **遇见的问题：**发生了[文档教程](https://docs.revyos.dev/docs/Installation/licheepi4a#%E5%86%99%E5%85%A5%E9%95%9C%E5%83%8F%E5%88%B0emmc%E4%B8%8D%E6%8E%A5%E5%85%A5%E4%B8%B2%E5%8F%A3)中提到的「数据块数量异常」现象
- **产生原因分析：**本来以为只是单纯的 USB 端口识别错误问题，但通过排除仍未解决该现象。通过[官方 Tg 群组](https://t.me/+Pi6px22-OsUxM2M1)中两位群友（@fc90d 与 @KevinMX_Neo）的帮助与分析得知，reboot 之后 `T-Head USB download gadget` 的 VID 和 PID 会变，而我使用的 VirtualBox 虚拟机工具似乎无法处理该问题。
- **解决办法：**
  1. 可以通过相关工具在 Windows 系统环境下进行镜像刷写
  2. 避免使用 VirtualBox，转而使用 VMware Workstation（后者近年宣布对个人用户免费）
