---
title: FPGA 设计学习笔记：时钟信号与数据信号的正确使用
description: 一次 FPGA 入门小实验遇到基础性错误的排查记录…
date: 2025-10-21 +0800
categories: [Development, FPGA]
tags: [FPGA, Vivado]
---

> 开发环境：  
> Vivado v2025.1 (64-bit)  
> SW Build: 6140274 on Thu May 22 00:12:29 MDT 2025  
> IP Build: 6138677 on Thu May 22 03:10:11 MDT 2025  
> SharedData Build: 6139179 on Tue May 20 17:58:58 MDT 2025

## 1 问题发现

在实现一个简单的按键控制 LED 的 FPGA 设计时，遇到了严重的布局布线错误：

```text
[Place 30-675] Sub-optimal placement for a global clock-capable IO pin and BUFG pair
[Place 30-99] Placer failed with error: 'IO Clock Placer failed'
```

关键错误信息显示：`key1_IBUF_inst/IBUFCTRL_INST` 被锁定在 `IOB_X0Y90`，而 `key1_IBUF_BUFG_inst` 被放置在 `BUFGCE_X0Y44`，两者不在同一时钟区域，违反了时钟布局规则。

## 2 问题定位

经过 LLM 老师的补课，发现根本原因为代码中将普通按键信号误用为时钟信号，如下所示：

```verilog
always@(key1)  // 错误的敏感列表
begin
    if(key1 == 1)
        cnt <= cnt + 1'b1;
        {led1_r,led2_r} <= cnt;
end
```

技术分析：

1. `always@(key1)` 的敏感列表导致综合工具将 `key1` 推断为时钟信号；
2. Vivado 尝试将 `key1` 分配到专用的全局时钟网络；
3. `key1` 的物理引脚不是时钟专用引脚，与 BUFG 位置不匹配；
4. 违反了 FPGA 的时钟布局规则 `rule_gclkio_bufg`。

概念混淆：

- 时钟信号：用于同步时序逻辑，需要专用布线资源；
- 数据信号：在时钟控制下采样处理，使用普通 IO 资源。

## 3 解决方案

引入差分时钟信号，同时采用同步设计：

```verilog
wire clk;
IBUFDS sys_clk_ibufds (.O(clk), .I(sys_clk_p), .IB(sys_clk_n));

// 按键同步和边沿检测
reg [2:0] key_sync;
always@(posedge clk) key_sync <= {key_sync[1:0], key1};

reg key_prev;
wire key_rise = key_sync[2] & ~key_prev;

// 在系统时钟下处理按键事件
always@(posedge clk) begin
    key_prev <= key_sync[2];
    if(key_rise) begin
        cnt <= cnt + 1'b1;
        {led1_r, led2_r} <= cnt;
    end
end
```

## 4 改进建议

1. 设计规范
   - 始终使用同步设计：避免 `always@(非时钟信号)` 的敏感列表；
   - 明确时钟域：所有寄存器应在明确的时钟控制下；
   - 异步信号处理：外部输入必须经过同步链处理；
2. 调试技巧
   - 优先检查敏感列表和时钟使用；
   - 关注 Vivado 的时钟相关警告；
   - 使用同步复位而非异步复位。

~~（又是接受 LLM 老师教育的一天啊）~~
