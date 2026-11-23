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

| 层级 | 谁在用 | 相对延迟 | **V100 实测（周期）** | 相对带宽 |
|---|---|---|---|---|
| 寄存器 | shader 局部变量 | 1× | —（操作数直取，1–2 拍） | 极高 |
| shared / L1 | 同工作组共享、纹理缓存 | ~10–30× | **28** | 高 |
| L2 | 跨 SM 复用（如 GS 排序、后处理） | ~100× | **193** | 中 |
| VRAM | 纹理、G-buffer、render target | ~300–1000× | **375**（TLB 命中）/ **1029**（TLB miss） | **低一个数量级** |

**推论**：**移动端的第一约束几乎总是 VRAM 带宽**（LPDDR 与独立显存差一个数量级），而**高端 PC 的第一约束常是"带宽 + 固定功能吞吐"**，不是 ALU。这解释了为什么同一个特效在 Android 上"降分辨率最有效"，在 PC 上却常常"降 shader 复杂度更有效"。

### 实测刻度：台阶不是均匀的（2026-11-23 补）

上表的相对倍数是"量级"，[[Jia 2018 — Dissecting the NVIDIA Volta GPU Architecture via Microbenchmarking|Jia 2018]] 用指针追逐把它钉成了绝对数（V100 PCIe，1.38 GHz）。**真正值得注意的是相邻两级的比值，而不是绝对值**：

| 台阶 | 比值 | 读法 |
|---|---|---|
| L1 → L2（28 → 193） | **6.9×** | **这一跳才是断崖**——L1 未命中等于把这次访问丢到 crossbar 上 |
| L2 → DRAM（193 → 375） | **1.9×** | **只差不到 2 倍**：L2 命中本来就要付 crossbar 的钱，所以"L2 没命中"没有想象中贵 |
| DRAM → DRAM+TLB（375 → 1029） | **2.7×** | 真正贵的是**页表没命中**，不是"内存慢" |

**三条可直接用的推论**：

1. **优化 L2 命中率，天花板很低**（1.9×）；**优化 L1 复用，收益最大**（6.9×）。这解释了为什么 shared memory / 寄存器分块（tiling）比"提高 L2 命中"更值钱。
2. **大跨度随机访问的真正杀手是 TLB**（1029 vs 375）。一个 16 GiB 的虚拟纹理 / 稀疏体素结构，如果跨页跳跃，成本是"普通显存访问"的 **2.7 倍**——这不是带宽问题，是地址翻译问题。
3. **延迟要换算成"需要多少 warp 才藏得住"**：指令依赖延迟 4 周期（FFMA，Volta）、L1 命中 28、L2 193。**在最坏假设下**（每个 warp 同时只有 1 个在途访存、每 4 拍发一条），藏住 L2 需要约 $193/4\approx48$ 个 warp——**已经逼近 Volta 的 64 warp/SM 上限**。真实情况靠"每 warp 多个在途访存（MLP）"缓解，所以**提高 occupancy 与提高 MLP 是两条路**；这也解释了为什么"occupancy 不是越高越好、但太低一定糟"。

> **上面第 3 条是个数量级估算，不是定理**：它的价值在于给出"48 这个量级"——一旦你发现自己的 shader 只有 8 个 warp 在飞，就**不必再看 stall reason**，答案已经在那了。

> **注意单位**：以上全是**周期**。换卡要换频率（$1/1.38\,\text{GHz}\approx0.72$ ns/周期），**跨架构不要直接引用周期数**（移动端的 wave 大小与 tile 内存结构不同）。

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
- [[Shadow Mapping]] —— 一次额外的"以光源为视角的渲染"，成本单位是"一遍场景"；
- [[Profiling and Measurement]] —— **本页的下游**：这张成本地图要靠三个域的仪器才读得出来（2026-11-23 建页）；
- [[Jia 2018 — Dissecting the NVIDIA Volta GPU Architecture via Microbenchmarking|Jia 2018]] —— 本页"实测刻度"那一节的来源（微基准量出的 28 / 193 / 375 / 1029）。

## Game Applications

- 五档画质的降级顺序设计（PC 与 Android 不同）；
- 特效预算的归因（先分 CPU/GPU，再分像素/几何/提交）；
- 判断一项新技术（Nanite / Lumen / MegaLights / 神经层）动的是哪一类资源。

## Personal Knowledge

Current Level: **Normal** —— 执行模型与带宽层级你熟悉；本页的价值是**把"哪个指标落在哪个单元"写成一张可查的表**，以及在评审时先问"这三类资源里谁先见底"。

## Next Step

拿一次真实的 GPU 抓帧（RenderDoc 或平台 profiler），**把一帧的耗时按"提交 / 几何 / 像素 / 计算"四段拆开**，与上表对一遍。**如果拆不出来，说明你对这一帧的成本归因还停在"感觉贵"。**

**（2026-11-23 补）拆之前先读两页**：[[Profiling and Measurement]] 给"这个数属于哪个域、能分辨多小的差"；本页给"这个数落在哪个硬件单元"。**两页各答一半问题——只看本页会得出"L2 好慢所以别用 L2"这类错结论（实测 L2→DRAM 只差 1.9×）。**
