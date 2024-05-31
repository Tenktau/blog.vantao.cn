---
title: 利用 OpenMV 实现 AprilTag 跟踪以及通过 UART 串口实时输出坐标信息
date: 2024-05-31 +0800
author: <vantao>
categories: [Development,Embedded]
tags: [站点更新日志]
---

## 项目介绍

此项目为本人在参与**Vision Board 创客营活动**第二阶段（应用作品设计）过程中所产出的内容。主要内容是实现 LED 矩阵实时响应 AprilTag 的坐标进行同步移动。

## 实现方案

总共涉及三款硬件，具体内容为：使用 **Vision Board** 作为上位机，并通过 OpenMV 程序获取 AprilTag 实时坐标、再通过 UART 串口通讯发送坐标信息给下位机；**BLE-Nano** 作为下位机接收坐标信息并将信息转化为 LED 矩阵坐标，再通过 FastLED 库实现对 **WS2812b** 灯带的 LED 操控。

## 开发现况

1. 由于手头上的 WS2812b 灯带连接触点氧化、无法焊接导线从而成为方形矩阵，故暂时放弃二维矩阵的模式，转而以初步实现一维矩阵（5 个 LED 灯珠）为目标。
2. 由于手头上持有的 BLE-Nano 是以蓝牙功能为卖点的设备，故蓝牙串口与 UART 串口之间可能存在某种冲突而无法正确接收 UART 串口协议信号。但是经过 USB 转 TTL 模块验证，Vision Board 可以正常通过串口输出，连线图以及输出效果如下图所示：

![测试 Vision Board 是否能通过串口正确发送坐标信息的连线图](https://file1.elecfans.com/web2/M00/EB/64/wKgaomZYoKOAMaBrAFZ8x5H8TnA442.jpg)

![测试 Vision Board 是否能通过串口正确发送坐标信息的输出效果](https://file1.elecfans.com/web2/M00/EA/7C/wKgZomZYnCCAMEu7AAh2KK3YZ7M274.gif)

后面有空的话再研究一下 BLE-Nano，再不行就换一块 Arduino 板子。目前来看，不妨将这个半成品作为一种 API 般的存在吧…~~（虽然这个半成品的实现门槛很低就是了）~~

### 待办 / TODO

- [ ] 调通上位机与下位机之间的 UART 通讯
- [ ] 当识别不同编号的 AprilTag 时，灯带发出不同颜色
- [ ] 实现二维 LED 矩阵形式的跟踪效果

> 我也将 OpenMV 以及 Arduino 工程文件[开源在 GitHub](https://github.com/Tenktau/LEDFollower)，欢迎对该项目感兴趣的朋友们前来指正以及补充。权当抛砖引玉，恳请各位不吝赐教！
