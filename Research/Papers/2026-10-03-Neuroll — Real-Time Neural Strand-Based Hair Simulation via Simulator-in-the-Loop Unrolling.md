---
type: paper
title: "Neuroll: Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling"
authors: [Gene Wei-Chin Lin, Jessica Jia-En Lee, Yu Ju Chen, Egor Larionov, Tuur Stuyck]
year: 2026
published: "2026-10-03（arXiv v1, 2610.04689）"
venue: "arXiv Preprint（ACM TOG 投稿；Meta（Vancouver/USA）× Meta Reality Labs × NVIDIA；cs.GR）"
url: "https://arxiv.org/abs/2610.04689"
code: ""
project_page: ""
category: [hair, simulation, physics-based-animation, neural-simulation, character, real-time]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Prototype（3000 股 0.460 ms/帧、线性扩展到 12 万股免重训；Meta 头像/游戏场景明确列为目标）"
user_level: "Normal（机制层）"
status: unread
tags: [hair, hair-simulation, neural-simulation, physics, real-time]
---

# Neuroll: Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling

> **入库 2026-10-06（Run 28）。** Meta（Vancouver/USA）× Meta Reality Labs × NVIDIA（Tuur Stuyck）联合；**ACM TOG 投稿**；10-3 提交、经 API"未公告先见"通道捕获。**本文是库内毛发线的"仿真轴"第一节点**——此前毛发线（[[Hair Rendering]]）全部节点（Kajiya-Kay 1989 / Marschner 2003 / Scheuermann 2004 / Zinke 2008 / HairCS 2026）都在**渲染侧**；这一篇回答另一半问题：**"头发怎么动"**。

## TL;DR

**用一个"镜像经典积分器输入输出形式"的神经网络替代毛发仿真的时间积分步骤**——输入 = 前一帧发丝状态（位置+速度，全部在**发丝局部坐标系**）+ 材料刚度标量 + 边界条件 + 外力 + 碰撞几何；输出 = 下一帧状态。训练不用任何预生成数据集：**模拟器在训练回路里现场监督**（simulator-in-the-loop），并用**随机展开视界**（m ~ U{1..10}）一次训出"准静态 + 动态"两种模式。结果：3000 股毛发 **0.460 ms/帧**（比经典 Cosserat 模拟器快两个数量级以上，比前作 Quaffure 快 7.7×），**连续动态帧数 2000 帧打满**（前作 Neuralocks 仅 131 帧就衰减变"硬"），碰撞穿插 0.070%（比值好一个数量级）；且免重训泛化——未见发型/动作/体型/物种。

> **一句话定位**：毛发线"**怎么动**"的新节点；[[Neural Physics Simulation]] 的第一个"**真人角色级**"（而非 VFX 粒子级）实例；"**神经替代哪一个部件、保留什么接口**"这一判据的又一份干净样本（今天的主线，与 CurveCodec 2 / DDGI / LoCoSplat 同题）。

## Problem（2026 的上下文）

- **经典路线的瓶颈**：Cosserat rod + GPU 求解器（Daviet 2023 / Hsu 2025 等）已能让数千股发丝"在高端配置上实时"——但**消费级硬件仍不可行**；工程妥协（guide strands / mesh 近似 / 子步进）一路牺牲质量；
- **前作神经方案的三处缺口**：
  - **GroomGen（2023）**：靠艺术家合成数据监督 → 迁移性差、数据工程重；
  - **Quaffure（2025）**：只做**准静态**（姿态→形变映射，单次前向，4096 股封顶）；
  - **Neuralocks（2026）**：引入惯性损失做动态，但**惯性损失的权重极其敏感**（在"过冲"与"过刚"之间没有满意中点），且**不把上一帧状态作为网络输入**——动力学只由损失函数诱导，长序列必然衰减（131 帧后低于 35% 参考运动量）；
- **布料侧的参照**：HOOD 一类自监督网络（能量势直接训练）成功，但**毛发用的 Cosserat rod 弹性势条件数差**，直接搬布料的"能量最小化"训练会让收敛显著退化；
- 目标：**免数据集、免运行时物理求解、长时稳定、跨外观/材质/动作/体型泛化**的实时发丝积分器。

## Previous Work（谱系）

```text
毛发仿真表示史（原文 Related Work）：
  悬臂梁（Anjyo 1992）→ 体素方法（Hadap 2001）→ Super-helices（2006）
  → Cosserat rod（Pai 2002 / Kugelstadt 2016）→ 离散弹性杆（Bergou 2008）
  → 弹簧-质点（Selle 2008）→ 显式网格（Yuksel 2009）
  → GPU 求解器（Daviet 2023 / Wu 2023-24 卷发 / Hsu 2025 高刚度稳定积分）
  ↓ 神经方案（Meta 线）
  GroomGen（2023，数据驱动）→ Quaffure（2025，准静态全空间）
  → Neuralocks（2026，strand 网络 + 惯性损失）
  ↓
★ Neuroll（2026）：神经时间积分器 + simulator-in-the-loop + 随机展开视界
```

- **训练方法学的谱系**：teacher forcing 的分布漂移问题（Bengio 2015 课程学习）→ 神经 PDE 求解器的展开（unrolling）与 pushforward 技巧（Brandstetter 2022）→ List et al. 2024 的"有/无时间梯度"对比 → **本篇：随机视界 + 全时间梯度（R10）**。

## Core Idea（机制）

### ① "神经时间积分器"= 镜像经典积分器的 I/O

| | 经典积分器 | Neuroll |
|---|---|---|
| 输入 | 前状态、材料参数、边界条件、外力、碰撞几何 | **同左**（前状态 = 位置+速度）|
| 输出 | 下一帧状态 | **下一帧形变**（相对 rest 状态）|
| 内部 | 数值求解 PDE | MLP（2 层 × 512 宽，1.63 MB）|

- 关键设计声明：**"积分器消费显式物理状态，而不是姿态历史或学到的隐状态"**——不用 GRU、不用 GNN；输入因此紧凑，且**推理时的行为（刚度、边界条件、外力）可以直接通过操纵输入来控制**；
- 三个配套组件：
  - **Strand 自编码器**：rest 状态压成 32 维潜码（+ UV + 刚度标量）；
  - **Body field（表示无关的碰撞信号）**：在身体表面**预选 200 个时序一致的点**（只要求跨帧对应，不要求网格），编码成 32 维潜码——网格 / 点云 / 高斯泼溅 / 体积表示都可以驱动同一个网络（推理时间与几何分辨率无关）；
  - **局部坐标系**：所有输入与监督信号统一在**每根发丝的根部局部系**（strand space）。

### ② 训练：模拟器在环 + 随机视界

- **循环**：随机抽发型 / 动作帧 / 体型 / 刚度（s ∈ [0.01, 1] 由 [20, 2000] 线性映射）/ 视界 m；发丝先从 rest 状态刚体变换到身体上、**用模拟器预滚 15 帧**得到"披挂"初始态；然后**模拟器与网络并行推进 m 帧**——模拟器（自研 Cosserat rod + VBD）逐帧给出参考状态，网络自回归地预测自己的下一帧；
- **损失**：`L = Σ_t (L_sim + L_collision)`——`L_sim` 是 strand space 里的位置 L2（**替代**了前作的惯性损失与 rod 弹性势）；消融表明 rod 势"冗余且有害"（加入后穿插翻倍），碰撞项单独就够；
- **网格不在计算图里**：模拟器只提供监督目标、不参与前向传播——**求解器可微性不是设计约束**；梯度只在网络自身的 m 步链上传播；
- **随机视界双模式**：短视界（m=1）时状态贴近"刚体变换的 rest"→ 网络学到**准静态披挂**；长视界时状态携带惯性历史 → 学到**动态传播与自身误差的消化**。**同一次训练同时获得两种模式**——推理时"每帧重置状态 = 准静态 / 保留状态 = 动态"，一个开关切换；
- **梯度消融**：R10（随机视界 + 全时间梯度）为默认；RNT10（去掉时间梯度，只保留分布暴露）几乎同分——**"展开的收益主要来自暴露于推理时输入分布，而非长程梯度"**（与 pushforward 技巧的动机一致）；teacher forcing 直接发散（两帧就炸，拉伸违规 24 cm）。

### ③ 为什么"局部坐标系"是关键先验

- 世界系变体（输入相对发根但用世界轴）的消融：**运动比 0.342 vs 0.632（网络几乎输出刚体）、动态帧 30 vs 800、穿插 11.45% vs 0.001%**；
- 原因（原文）：世界系下"同一物理形变随头部朝向出现在许多不同输入里"→ 网络退化为"近零形变"解；**strand space 把发丝规范化（canonicalize）：rest 构型与朝向无关，朝向依赖被压缩成"重力方向"一个向量**——"先验进结构"家族在仿真域的又一实例。

## Key Numbers（原文逐表核对）

**Table 1（3000 股 × 2000 帧；GT = 完整物理模拟）**

| 指标 | GT | Quaffure | Neuralocks | **Neuroll** |
|---|---|---|---|---|
| 时间 / 帧 | 89.517 ms | 3.550 ms | 0.203 ms* | **0.460 ms**（全身场 0.273 + 积分器 0.187）|
| 拉伸违规 E_s（↓）| 0.230 cm | 1.527 | 0.601 | 0.850 |
| 碰撞穿插 E_col（↓）| 0.027% | 5.315% | 1.859% | **0.070%** |
| 回放误差 E_roll（↓）| 0 | 5.686 | 3.350 | 3.339 |
| 速度误差 E_vel（↓）| 0 | 0.937 | 0.956 | **0.831** |
| 运动比 E_mr（→1）| 1.0 | 0.296 | 0.495 | **0.622** |
| **连续动态帧数（↑）** | 2000 | 40 | 131 | **2000（打满）** |

\* Neuralocks 仅计积分器自身；本篇 0.460 ms 含全身场（同口径对比见正文 7.7× 对 Quaffure、>2 数量级对模拟器）。

- **扩展性**：3000 股训练 → **12 万股免重训 6.503 ms/帧**（逐发丝网络 → 线性扩展、密度无关）；
- **泛化**：4 个训练发型 + 6 段动作（13,161 帧）→ 未见 16 个发型 / 数十段动作 / 未见体型；**训练只见过重力，推理可驱动任意方向外力**（如时变风场，力向量归一化后入网）；
- **训练成本**：R10 每步 1.152 s；R30 慢 2.3× 且指标全差；TF 0.380 s/步但两帧发散。

## Why It Works（为什么成立——四条）

1. **"镜像 I/O"使知识可复用**：网络学的是"积分器这一步的函数"，而不是"某种发型的样子"——物理量（状态/刚度/碰撞）本身就是最紧凑的输入空间；
2. **规范化先验**（strand space）：把"朝向 × 形状"的大分布压成"rest 构型 + 重力方向"，是泛化跨发型/体型的表示层机制；
3. **在环监督 = 免费数据 + 在线策略**：现场生成的监督天然覆盖网络自己的误差分布（本库 [[2026-09-24-OREO — Fidelity Alignment in 3D Generation via On-The-Fly Rendering-Editing Optimization|OREO]]"监督必须跟着学生走"的仿真版本）；
4. **位置损失替代能量损失**：参考轨迹已经编码了"惯性 vs 弹性的正确平衡"，网络只需用条件良好的 L2 复现它——把"求解物理"降级为"回归轨迹"，同时靠训练分布兜住物理性。

## Limitations（原文承认）

- **长发困难**（两条原因）：训练窗口永远从"披挂态"起步 → 没见过长发大幅低频摆动；体碰撞场只覆盖头肩区域 → 长发下半段没有碰撞信号；
- **运动幅度系统性偏低**（运动比 0.622）：主要来自碰撞障碍项的方向性偏置（位置 L2 对"穿进体内"与"穿出体外"对称）——后续可扫障碍刚度/边距，或换无方向偏置的碰撞项；
- **无自碰撞**（会破坏逐发丝独立性 → 丧失线性扩展与密度无关性）；也因此**无摩擦**——对长发动力学重要；
- （工程）依赖自研模拟器；准静态/动态切换是推理期行为，训练侧不区分。

## Game Development Relevance

- **实时发丝仿真在"消费级硬件"上成立**：0.460 ms / 3000 股 / 单帧——游戏与虚拟形象场景被原文明确点名；密度无关意味着**成本与"渲染的股数"解耦**（与 [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)|Kajiya-Kay 1989]]"渲染时间与几何复杂度解耦"在仿真侧的镜像）；
- **对毛发线（[[Hair Rendering]]）**：仿真轴第一节点——"头发怎么动"此前在库内是空白（只有 UE Groom/Niagara 一行工程注记）；
- **对分档工作**：毛发档位新增两只旋钮——① **仿真模式**（off / 准静态 / 动态——一个网络三态）；② **刚度标量**（s 连续可调，免重训）——"档位 = 多个正交子系统的组合"（9-21 判断）在毛发的仿真侧又添实证；结合渲染侧的 LSS（巫 3 实测：HairWorks 开关 = 30 vs 44 fps），**"毛发重项"的头像/角色场景有了两条可分别调档的轴**；
- **与你的 VFX 域**：逐发丝独立 + 密度无关 + 线性扩展 = 与粒子系统同构的成本结构（[[Particle Systems]]：屏占比 × 密度 vs 这里的股数 × 帧）——"统一 VFX/毛发求解器"的中期想象（[[Neural Physics Simulation]] Game Applications）。

## Unreal Engine Relevance

- 非引擎绑定工作；UE 侧对应物是 **Groom 组件 + Niagara 驱动的发丝物理**——本篇可视为"Groom 物理的学习式替代"方向；但**不要强行映射**：论文用自研模拟器（Cosserat rod + VBD），与 UE Chaos 的发丝方案不是同一套实现；
- 有意义的接口判断：**"表示无关的 body field"意味着它不挑身体资产形态**（网格/点云/GS/体积均可）——若未来引擎侧引入，资产管线改动面小。

## Technology Evolution（毛发线：渲染轴 × 仿真轴）

```text
【渲染轴】（库内已闭合）
1989 Kajiya-Kay → 2003 Marschner → 2004 Scheuermann → 2008 Zinke（多散射）
→ 2026 HairCS（发片⇄发丝升档）/ LSS 光追毛发（巫 3 重制版）

【仿真轴】（本篇开线）
1992 悬臂梁 → … → Cosserat rod / DER → GPU 求解器（数千股实时，高端配置）
→ 2023-2026 神经方案三代：GroomGen（数据驱动）→ Quaffure（准静态）→ Neuralocks（惯性损失）
→ ★ 2026 Neuroll：神经积分器（镜像 I/O）+ 模拟器在环 + 随机视界 → 消费级硬件实时
```

> **两条轴的会合判断**：渲染轴的关键词是"表示"（texel/cards/strands/光追基元），仿真轴的关键词是"积分"（solver → neural time-stepper）——Neuroll 把"积分"从'每帧跑 PDE'变成'每帧跑一次 1.63 MB 的前向'，**与渲染侧的"换表示"是同一哲学：把贵的东西换掉，而不是少做**。

## Relationships

### Based On
- Neuralocks（Lin et al. 2026，同组直系前作：strand 自编码器、碰撞障碍项、指标口径均沿用）
- Cosserat rod 模拟器（Hsu et al. 2025）+ Vertex Block Descent（Chen et al. 2024）——训练期的"老师"
- 神经 PDE 展开文献（List et al. 2024；pushforward 技巧 Brandstetter 2022）

### Extends / Improves
- **Extends** Neuralocks：把"末帧状态不回喂"改为"显式状态输入 + 在环展开" → 长时动态不再衰减（131 → 2000 帧）；
- **Improves** Quaffure：准静态 → 动态；4096 股封顶 → 线性扩展。

### Related
- [[Neural Physics Simulation]]（"学什么、不学什么"——本篇把"积分步骤"整个交给网络，同时保留经典 I/O 语义）
- [[Hair Rendering]]（仿真轴的对应概念；渲染=表示，仿真=积分）
- [[Particle Systems]]（逐元素独立 → 线性扩展的成本结构同构）
- [[2026-09-24-OREO — Fidelity Alignment in 3D Generation via On-The-Fly Rendering-Editing Optimization|OREO]]（on-policy 监督判据的仿真版）

### Followed By
- （观察）代码/项目页未放出；Meta 头像产品线方向；与 LSS（渲染侧光追毛发）的合流想象

## Personal Knowledge State

- `user_level: Normal（机制层）`——毛发线在你的 PKM 中为"待选支线"（Marschner / HairCS 未读；渲染侧 15 条自测挂起）；本篇为**仿真轴独立节点**，不与渲染侧自测绑定；
- **20 分钟机制层读法**：摘要 → Figure 2（训练管线：两条数据流 = strand data / motion data）→ Table 1（动态帧数 2000 vs 131 是全文最强单点）→ 消融表（strand space vs world space）。

## Learning Value

- **判据样本（今天的主线）**："神经替代哪里、保留什么"——本篇答案：**替代数值积分、保留经典积分器的 I/O 语义**（对比 CurveCodec 2"只接管熵模型"、DDGI"光追补充光栅化"、LoCoSplat"取消重型 3D 网络"）；
- **"表示即先验"第 N 例**：strand space 消融（0.342 vs 0.632）是本库该家族最干净的量化之一；
- **可迁移的工程问题**：① "训练窗口从稳态起步 → 学不到低频大摆"（长发限制）——任何"从静止态预滚"的训练策略都会遇到；② "对称损失项的方向性偏置"（碰撞穿插）——回归对称性与物理非对称性的矛盾。

## Visualization

（无独立图解；与渲染轴的对照表见 [[Hair Rendering]] 新增"仿真轴"小节与 [[2026-10-06]] 日报。）

## Notes

- **来源与核对**：arXiv **HTML 全文**（cs.GR，73,736 字符）逐节核对——Table 1/2/3/4 全部数字、消融与限制均出自 §4；作者机构从论文 header 逐行核对（Meta ×4 + NVIDIA 的 Tuur Stuyck；**Jounal：TOG** 投稿标注在论文页眉）；10-3 提交，经 API"未公告先见"通道捕获（双通道第 9 次验证）；
- 与 [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] 的对照阅读点：两篇都在"**单根发丝的线性结构**"上做文章（Zinke：一条原型路径代表全部路径；Neuroll：逐发丝独立网络 → 密度无关）——**"线性可分解"是毛发问题两次被打开的同一把钥匙**。
