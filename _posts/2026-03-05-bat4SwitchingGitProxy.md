---
title: 借助 Claude，一分钟开发一个 Git 代理切换脚本
description: 高效切换 Git 代理…LLM 太好用了！×N
date: 2026-03-05 +0800
categories: [Development, Software]
tags: [Git, Script]
---

> **批处理文件**，在 DOS 和 Windows（任意）操作系统中，`.BAT` 文件是可执行文件，由一系列命令构成，其中可以包含对其他程序的调用。这个文件的每一行都是一条 DOS 命令（大部分时候就好像我们在 DOS 提示符下执行的命令行一样），你可以使用 DOS 下的 EDIT 或者 Windows 的记事本等任何文本文件编辑工具创建和修改批处理文件。  
> 运行命令即 **DOS 命令**，主要是面向 DOS 操作系统的。DOS 命令指 DOS 操作系统的命令，因 DOS 实际上是磁盘操作系统，所以 DOS 命令是一种面向磁盘管理的操作命令。与 Windows 操作系统最大的区别在于，它以命令行的形式，靠输入命令来进行人机对话，并通过命令的形式把指令传给计算机，以实现对计算机的操作。DOS 命令主要包括内部命令、外部命令和批处理命令。

## 1 开发背景

由于网络不稳定，利用 Git 工具抓取 GitHub 仓库时、很容易超时，因此为了提升连接稳定性、需要为 Git 工具设置代理，但也随之产生了以下问题：

Git 设置了代理之后，若提供对应端口的代理软件没启动、可能会导致连接丢失，但每次都手动打代码修改 Git 代理设置是一件很麻烦的事情。

## 2 开发思路

针对我这个小众的需求，网络上没有能够轻易找到的解决方案，于是只能自己动脑想想…

本来想的是做两个 `.BAT`，一个开一个关。但转念一想，加个判断进行切换不就好了吗？但我又没有相关代码知识点储备，于是便将需求交给 Claude，从码字到获得可运行的代码、只花了一分钟！

```bat
@echo off
chcp 65001 >nul

:: 检查当前 http 代理是否已设置
git config --global --get http.https://github.com.proxy >nul 2>&1

if %errorlevel% == 0 (
    :: 已有代理，执行取消
    git config --global --unset http.https://github.com.proxy
    git config --global --unset https.https://github.com.proxy
    echo Git 代理已关闭
) else (
    :: 无代理，执行添加
    git config --global http.https://github.com.proxy socks5://127.0.0.1:1087
    git config --global https.https://github.com.proxy socks5://127.0.0.1:1087
    echo Git 代理已开启：socks5://127.0.0.1:1087
)

echo.
pause
```

> 温馨提示：对于想要使用上面这份代码的朋友，你需要注意将 `127.0.0.1:1087` 修改为你的代理软件中实际设置的代理主机 ip 和端口。
{: .prompt-warning }

## 3 事后感悟

当然，这个需求本身实现起来很简单…但如果没有 LLM，我至少还得花小半个小时了解 DOS。

不得不感慨，工具越强大、人在使用它的过程中就越能解放自己…以后是 idea 为王的时代了！
