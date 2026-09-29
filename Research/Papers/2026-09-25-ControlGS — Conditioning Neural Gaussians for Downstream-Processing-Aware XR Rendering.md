---
type: paper
title: "ControlGS: Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering"
authors: [Weikai Lin, Junjie Zhao, Carl Marshall, Sushant Kondguli, Yuhao Zhu]
year: 2026
published: "2026-09-25 (arXiv) / SIGGRAPH Asia 2026"
venue: "SIGGRAPH Asia 2026"
url: "https://arxiv.org/abs/2609.32038"
code: "https://horizon-lab.org/controlgs/"
project_page: "https://horizon-lab.org/controlgs/"
category: [gaussian-splatting, xr, neural-rendering, rendering-pipeline, real-time]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Normal
status: unread
aliases: [ControlGS, downstream-processing-aware rendering, 下游处理感知渲染]
tags: [gaussian-splatting, xr, rendering, neural-rendering, pipeline, budget]
---

# ControlGS: Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering（SIGGRAPH Asia 2026）

## TL;DR

**XR 用户看到的不是渲染器的输出。** 渲染结果要先穿过一整套**后处理管线**（镜头校正 / 抗锯齿重采样 / 显示映射）和**物理显示-光学路径**才到达眼睛——而且这套"下游"在运行时不断变化（相机位姿、**显示功耗预算**）。传统 3DGS 要么默认"下游不损伤画质"，要么无法适配下游变化。

ControlGS 的答案两条：

1. **把"渲染输出 → 人眼"之间的整条下游链建模进优化目标**（不是优化到 framebuffer，而是优化到眼睛）；
2. **让高斯按下游参数"条件化动态生成"**——下游状态（分辨率、显示功耗档）成为生成的条件输入；且**只训练一个轻量注入模块**（预训练神经高斯骨干不动）。

**结果**：在 4 档显示采样率 × 5 档功耗节省 = **20 个运行时设置**下一致提升端到端画质（跨骨干、跨数据集），开销极小；即使 1× 采样、0% 省电，仅靠镜头条件也能提升（因为它按每个高斯的屏幕投影位置适配解码）。

## Problem

原文摘要开门见山（可引用）：

> "XR users do not directly perceive the output of a rendering engine. Instead, rendered images pass through a post-processing pipeline and the physical display-optics path before reaching the eye. Critically, the exact downstream processing **can vary significantly at run time**, influenced by, for instance, camera pose and **display power budget**."

两个现实：

- **下游不是恒等变换**：镜头畸变校正、重采样、显示映射都会改变像素，物理光学还有自己的传递函数；
- **下游参数在运行时漂移**：用户转头（位姿）、设备省电（**功耗预算**）→ 同一套高斯不该用同一套解码。

现有 3DGS 的默认假设是"下游保真"，或者干脆无法适配下游变化——**优化目标截断在 framebuffer，责任甩给了后处理链**。

## Core Idea

### 1. 优化目标延伸："一直优化到眼睛"

```text
传统 3DGS：  高斯 → 渲染器 → ✂（目标截断）→ 后处理 → 显示光学 → 眼睛
ControlGS：  高斯 → 渲染器 ──────────── 目标覆盖全链 ────────────► 眼睛
                          （后处理 + 物理显示-光学全建模进损失）
```

- 下游链被显式建模：**Lens Correction（预变形）/ Anti-Aliasing Resampling / Display Mapping / Physical Display / Projective Lens**（§3.1–3.2）；
- 评价指标也跟着换：不是"渲染图 vs 真值"，而是"**post-optics 图像** vs 真值"（端到端画质）。

### 2. 运行时状态作为"生成条件"

- **Conditional Dynamic Gaussians**：神经高斯骨干输出之上，做**条件注入（Conditional Injection）**——把下游参数（降采样因子、功耗节省比例、位姿算子等）注入为条件；
- **只训练注入模块**（预训练骨干冻结）→ 训练开销最小化；
- **运行时可调注入强度**（§4.2）：不同下游模块（镜头/重采样/映射）有各自的条件通道。

### 3. 实验设置（值得记的"状态空间"设计）

| 维度 | 取值 |
|---|---|
| 显示采样率（等效降采样） | 1× / 2× / 4× / 8× |
| **显示功耗节省比例** | 0% / 20% / 40% / 60% / 80% |
| 组合 | **20 个运行时设置**，逐设置评估 PSNR / SSIM / LPIPS |

对比对象：Scaffold-GS、及其加了超采样的变体（SS-Scaffold-GS）等——在原生分辨率 0% 省电与 8× 降采样 80% 省电两端，ControlGS 都更优。

## Why It Works

1. **"谁在决定最终像素"被重新画了边界**：渲染器的输出只是中间量；把边界画到眼睛，优化才有完整的因果链（与 [[Neural Upscaling and Frame Generation]] 同题：神经层也是"渲染输出之后"的一环）；
2. **条件化代替重训**：下游参数是**连续变化的状态**，不可能每个状态训一套模型——把状态变成"条件输入"，一个模型覆盖状态空间（exactly 是 "Dynamic（自设目标帧率）"档位思想的生成模型版本）；
3. **最小改动原则的工程版**：骨干冻结、只训注入——**"把变化的部分隔离出来"**（与 ReFM 的"最小改动能量"同日入库，两个不同领域同题）。

## Limitations

- **Sim-to-Real Gap（作者自承，§6）**：物理显示-光学路径是**仿真**的——真机的光学差异未闭环；
- 泛化问题：**非对称光学**（asymmetric optics）的泛化性被作者列为待研究方向；更复杂的下游模块（additional downstream modules）同理；
- 面向 XR 头显场景；非 XR 平台（PC / 主机 / 移动直屏）的"下游链"组成不同，结论不能直接平移。

## Game Development Relevance

**3/5 —— 领域（XR）离 NGR 较远，但两条抽象直接可迁移。**

1. **"端到端"从口号变成方法**：库内 9-11 的"组件加速 ≠ 端到端加速"是**负例**（优化了组件、端到端没快）；ControlGS 是**正例**——"既然端到端才是真目标，就把端到端纳入优化"。**迁移句：评价一次优化时，问"它优化的是中间产物还是最终产物"**；
2. **"运行时状态 = 生成条件"**：**显示功耗预算**直接成为条件维度——**预算不再只是被约束的对象，也是被条件化的信号**。与《控制：共振》的 `Dynamic`（自设目标帧率、闭环）同族：**档位正在从"预设"走向"条件化 + 闭环"**；
3. **对分档体系的接口**：如果未来移动端也跑"神经层 + 生成式渲染"，"按功耗/温度状态条件化"很可能是档位系统的下一代形态——本库当前五档是**静态定义**，值得记一个演进方向。

## Unreal Engine Relevance

- XR 的高斯渲染目前不在 UE 主路径（UE 的 XR 走传统光栅 + 延迟着色；3DGS 在 UE 侧多为实验插件）——本文的"引擎内落地"路径不清晰；
- 但**"后处理链损伤画质"在 UE 里同样存在**（升频/去噪/锐化/色彩管线都会改变像素）——ControlGS 的"把后处理纳入目标"与 UE 的 TSR/DLSS 调参文化是同一个问题的不同解。

## Technology Evolution

```text
3DGS 研究的关注点迁移：
2023-24  "把 GS 做出来"（质量 / 训练）
2025-26  "把 GS 做便宜"（排序取消 / 密度 / 存储：[[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]]、[[Compact Neural Appearance Models for Efficient Gaussian Splatting]]）
2026 ★   "把 GS 放到真实管线里"（下游感知：目标不再截断在渲染器输出）
          ↑ 与"生成条件化"合流：条件输入 = 运行时状态（位姿 / 功耗）
```

## Relationships

### Based On

- **[[Gaussian Splatting]]**（Easy 域之上的新研究：本文质疑的正是 GS 类方法"下游保真"这个默认假设——属 [[Index#使用约定]] Easy 例外第 5 条："新论文改变了你已理解的假设"）。

### Related

- **⟷ [[Neural Upscaling and Frame Generation]]**：两者都在"渲染器输出之后"做事——DLSS 是神经层的**成本转移**，ControlGS 是神经层的**目标延伸**；合起来看：**"渲染输出"这个边界正在双向淡化**；
- **⟷ [[Scalability and Quality Tiers]]**：20 个运行时设置 = 一个"下游状态空间"；条件化生成 = "动态档位"——与《控制：共振》`Dynamic` 闭环、XeSS 3 帧生成倍率构成"**档位条件化**"的第三条线索（2026-09-22 记录）；
- **⟷ [[Temporal Stability and Artistic Intent]]**：下游链的每一环都会改变最终观感——"优化到眼睛"意味着"艺术意图"也要在 post-optics 层面定义（弱关联，记为观察）。

## Personal Knowledge State

- **user_level: Normal**：GS 基础你已经 Easy（9-11），本文是"GS 之上的新研究"且**不需要新数学**——两个抽象（端到端目标 / 状态条件化）都能直接搬去压测你的分档思考；
- 与你五档工作的距离：**"静态五档" vs "条件化 + 闭环"** 是这份笔记最值钱的一问。

## Learning Value

1. **判据（今日入库第二条）**：**优化目标画到哪里，决定你优化的是什么**——问"它优化的是中间产物还是最终产物"；再问"中间还有几环被别人负责，它们真的负责了吗"；
2. **技法**："**只训练注入模块**"——把"条件化"与"骨干"解耦，是"小改动适配大状态空间"的标准工程手法；
3. **状态空间设计**：2 个维度各 4-5 档 = 20 个设置的全矩阵评估——**给"运行时状态"建一张组合表**，正是你分档矩阵的思路在别人领域的镜像。

## Notes

- arXiv 2609.32038v1（cs.CV / cs.GR 交叉，2026-09-25 提交；**cs.GR 非主分类 → 由 API 窗口捕获**）；SIGGRAPH Asia 2026 正式论文（36 页，含附录）；
- 署名：University of Rochester（Weikai Lin / Junjie Zhao / Yuhao Zhu，horizon-lab）+ 工业合作者（Carl Marshall / Sushant Kondguli）；
- 全文 HTML 已抓取核对：20 设置矩阵、条件注入/骨干冻结、post-optics 指标、sim-to-real 与不对称光学限制、"even at 1× / 0% lens condition improves"均出自原文；
- 代码与项目页：https://horizon-lab.org/controlgs/ 。
