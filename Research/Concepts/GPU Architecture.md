---
type: concept
title: "GPU Architecture"
user_level: Normal
tags: [gpu, performance, architecture]
---

# GPU Architecture

> 本页不是"GPU 内部结构百科"，而是一张**成本地图**：你的五个实测指标（Particles / DrawCalls / Primitives / OverDraw / CPUTime / GPUTime）分别落在哪个硬件单元上，以及为什么"算力够但帧率不够"是常态。

## Definition

GPU 是**为吞吐量设计的、以延迟隐藏换面积的并行处理器**：大量简单的执行单元 + 深流水线 + 高带宽显存，通过"同时驻留很多个 warp"来掩盖单条指令的等待。

**理解它只需要三件事**：执行模型（谁在并行）、存储层级（数据离计算多远）、固定功能单元（哪些阶段是硬连线、改不了）。

## Core Principle

> **GPU 的性能不是由"峰值算力"决定，而是由"三类资源中最先见底的那一个"决定：算力 / 带宽 / 延迟隐藏能力。**

```
              ┌─ 算力（ALU / SFU / RT core / Tensor core）
一帧的成本 ←──┼─ 带宽（VRAM ↔ L2 ↔ L1/shared ↔ 寄存器）
              └─ 延迟隐藏（occupancy × 指令级并行）
```

三者互相牵制：提高 occupancy 要吃寄存器 ⇒ 寄存器压力大 ⇒ 溢出到 local memory ⇒ **反而增加带宽**。**任何"优化"都得说出它动的是哪一类资源**。

## 执行模型

- **SIMT**：一个 warp（NVIDIA 32 线程）/ wave（AMD、移动端 32–64）共用一条指令流；**分支发散（divergence）会让两条路径串行执行**——这就是"粒子 shader 里的 `if` 很贵"的真正原因（不是跳转贵，是两边的活都得干）；
- **Occupancy**：SM 上同时驻留的 warp 数 ÷ 上限。它是**延迟隐藏的库存**，不是目的本身——带宽受限的 shader 提高 occupancy 往往没用；
- **指令级并行（ILP）**：同一 warp 内独立指令可以重叠等待，所以"少依赖链"有时比"少指令"更值钱。

> **TA 视角的一句话**：**看 shader 贵不贵，先看它是不是"带宽型"（大量纹理采样/读写）还是"算力型"（大量 ALU）**——两者的优化方向完全相反。

## 存储层级与带宽（量级感）

| 层级 | 谁在用 | 相对延迟 | 相对带宽 |
|---|---|---|---|
| 寄存器 | shader 局部变量 | 1× | 极高 |
| shared / L1 | 同工作组共享、纹理缓存 | ~10–30× | 高 |
| L2 | 跨 SM 复用（如 GS 排序、后处理） | ~100× | 中 |
| VRAM | 纹理、G-buffer、render target | ~300–1000× | **低一个数量级** |

**推论**：**移动端的第一约束几乎总是 VRAM 带宽**（LPDDR 与独立显存差一个数量级），而**高端 PC 的第一约束常是"带宽 + 固定功能吞吐"**，不是 ALU。这解释了为什么同一个特效在 Android 上"降分辨率最有效"，在 PC 上却常常"降 shader 复杂度更有效"。

## 固定功能单元（改不了的那部分）

| 单元 | 干什么 | 你的哪个指标落在这里 |
|---|---|---|
| **光栅化器** | 三角形 → 片元（quad 2×2 粒度） | **Primitives**；细碎三角形会浪费 quad |
| **ROP / OM** | 深度测试、混合、MSAA resolve、写入 | **OverDraw**（真正的读写账单） |
| **纹理单元** | 采样 + 过滤 | 采样型 shader 的瓶颈 |
| **前端 / 命令处理器** | 解析 draw call、状态切换 | **DrawCalls** |
| **计算/调度器** | compute / mesh / ray 的线程派发 | Particles（GPU 模拟） |

> **一个常被忽略的事实**：**ROP 的吞吐是"每时钟处理多少像素"，与分辨率直接绑定**。所以"渲染分辨率"是唯一一个能同时影响所有像素级成本的旋钮——这也是为什么动态分辨率总是第一个被打开的开关。

## 两种架构：IMR 与 TBDR

| | IMR（PC 独显主流） | TBDR（移动端主流） |
|---|---|---|
| 渲染顺序 | 逐 draw 直接写 VRAM | 先把几何分桶到 tile，再逐 tile 着色 |
| 不透明 overdraw | 靠 early-Z 剔除 | 靠 **HSR**（Hidden Surface Removal）**整 tile 前置剔除** |
| 混合带宽 | 每次混合读+写 VRAM | 在**片上 tile 内存**完成，只有最终结果写回 |
| 主要风险 | 带宽 + 过度绘制 | **tile 内存溢出 ⇒ flush ⇒ 断崖** |
| 半透明 | 贵在填充 | 贵在**打断 HSR + 占 tile 内存** |

细节见 [[Tile-Based Rendering]]；**对 VFX 的含义**：同一个特效在两端的最优降级顺序不同——**PC 先缩 shader，Android 先缩层数与分辨率**。

## 成本映射：你的六个指标落在哪

| 指标 | 主要硬件归属 | 典型瓶颈信号 | 第一优化动作 |
|---|---|---|---|
| **Particles** | compute 单元（GPU 模拟）/ CPU（CPU 模拟） | GPU 时间随粒子数线性上涨 | 降 simulation 频率 / 换 GPU |
| **DrawCalls** | 前端 + 驱动（CPU 侧） | CPUTime 高而 GPUTime 低 | 合批 / instancing / 减少材质切换 |
| **Primitives** | 光栅化器 + 顶点处理 | 小三角形多 ⇒ quad 利用率低 | 降细分 / LOD / impostor |
| **OverDraw** | ROP + 带宽 | GPUTime 随面积涨、随数量不涨 | **先缩尺寸**（平方律），再减层数（见 [[Overdraw]]） |
| **CPUTime** | 提交线程 + 逻辑 | 单帧 > 8 ms 且与画质无关 | 并行提交 / 减少 actor 更新 |
| **GPUTime** | 全部 | 随画质变化明显 | 按上面四行逐项归因 |

**用法**：**先看 CPUTime 与 GPUTime 的比值**——CPU 高 GPU 低是"提交问题"，GPU 高是"像素/几何问题"，两者都高才是"内容超载"。**不要在看比值之前就开始删特效。**

## 常见误区

1. **"算力还有余量"** —— 峰值 FLOPS 与游戏负载的混合比例（ALU : 纹理 : 带宽）常常差很远，余量在错误的资源上；
2. **"occupancy 越高越好"** —— 带宽受限时提高 occupancy 只是让更多 warp 一起等内存；
3. **"draw call 少就快"** —— 现代 API 下单 draw 成本已大幅下降，**状态切换与 RT 切换**往往才是真凶；
4. **"移动端就是弱一点的 PC"** —— 架构不同（TBDR），**降级顺序也不同**；
5. **"半透明和透明一样"** —— 见 [[Overdraw]] 的"单价取决于管线阶段"表：同样覆盖率，成本可差数倍。

## Historical Evolution

```text
1996–2000  3D 加速卡固定管线；fill rate 是规格表第一行
      ↓
2001–2005  可编程顶点/片元着色器（shader model 2/3）：成本从"像素吞吐"转向"每片元算法"
      ↓
2006–2010  统一着色器架构（G80 起）：顶点/像素共用 ALU 池 ⇒ occupancy 成为核心概念
      ↓
2010–2015  compute shader / tessellation 进入主流；移动 TBDR 大规模普及
      ↓
2018–2022  mesh shader、硬件光追单元、GPU-driven pipeline（见 [[GPU-Driven Rendering]]）
      ↓
2023–2026  神经渲染层（[[DLSS 5 — Generative Neural Rendering]]）把 Tensor core 纳入常规预算
```

**一条主线**：**每一代都在把"某个固定阶段"变成"可编程 + 可调度"**，代价是**成本从"硬件替你保证"变成"你自己记账"**。

## Prerequisites

- [[Real-Time Rendering]]

## Related Concepts

- [[Tile-Based Rendering]] —— 移动端架构分支；
- [[GPU-Driven Rendering]] —— 把调度也搬到 GPU；
- [[GPGPU]] —— 编程模型侧：三条锁（写位置固定 / 不可读写 / 精度打包）与"一遍 pass 一遍全屏"的成本形状；
- [[Overdraw]] —— ROP 侧的账单；
- [[Particle Systems]] / [[Niagara]] —— compute 侧的典型负载；
- [[Shadow Mapping]] —— 一次额外的"以光源为视角的渲染"，成本单位是"一遍场景"。

## Game Applications

- 五档画质的降级顺序设计（PC 与 Android 不同）；
- 特效预算的归因（先分 CPU/GPU，再分像素/几何/提交）；
- 判断一项新技术（Nanite / Lumen / MegaLights / 神经层）动的是哪一类资源。

## Personal Knowledge

Current Level: **Normal** —— 执行模型与带宽层级你熟悉；本页的价值是**把"哪个指标落在哪个单元"写成一张可查的表**，以及在评审时先问"这三类资源里谁先见底"。

## Next Step

拿一次真实的 GPU 抓帧（RenderDoc 或平台 profiler），**把一帧的耗时按"提交 / 几何 / 像素 / 计算"四段拆开**，与上表对一遍。**如果拆不出来，说明你对这一帧的成本归因还停在"感觉贵"。**
