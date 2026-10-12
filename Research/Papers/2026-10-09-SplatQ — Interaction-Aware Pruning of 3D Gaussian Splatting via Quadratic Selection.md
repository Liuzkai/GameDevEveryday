---
type: paper
title: "SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection"
authors: [Hongbin Ye, Haixu Liu, Shenwei Kang, Wenxiu Wei, Tinglan Wang, Hongdong Zhu, Fan Tang, Qi Gao, Yin Ma, Kai Wen, Hai Wei, Nuo Qian, Wei-Shi Zheng, Shaobo Chen, Fa-Ting Hong, Jingke Meng]
year: 2026
published: "2026-10-09（arXiv v1, 2610.13503；cs.CV + cs.GR 双分类）"
venue: "arXiv Preprint（Under review at ICLR 2027；单位与代码待核——提交后 PDF 尚未放出；未公告先见：API 通道 2026-10-12 捕获，正式公告预计 Mon 10-12 US 批次）"
url: "https://arxiv.org/abs/2610.13503"
code: ""
project_page: ""
category: [gaussian-splatting, pruning, budget-allocation, combinatorial-optimization, ising-machine]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Research（ICLR 2027 在审；5 个 benchmark 的聚合实验；无代码）"
user_level: "Normal（结论层：'区域预算 + 联合选择'的两层分工）"
status: unread
aliases: [SplatQ, 交互感知剪枝, 二次选择, QUBO 剪枝]
tags: [gaussian-splatting, pruning, budget, optimization]
---

# SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection（Ye et al. 2026）

> **入库 2026-10-12（Run 34）。** 经 **API"未公告先见"通道**捕获（提交 10-09、周五黑窗之外、正式公告前的窗口）——本日 4 篇前沿全部来自该通道（cs.GR 窗口 5 条 + cs.CV 窗口 84 条扫描，净新）。
> **一句话定位**：**把 3DGS 剪枝从"全局 Top-K 排序"改写成"固定预算下的组合选择问题"**——先按空间/外观分块解决"**预算分到哪**"（区域配额 + 配额平滑），再在块内用**二次近似建模"联合删除效应"**（单个响应 + 成对交互）解决"**选哪些**"（QUBO，用模拟退火或相干伊辛机求解）。
> **库内位置**：GS"预算线"第三节点（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 定"预算=误差曲线上的点" → [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]] 定"预算=采样数" → **本篇定"预算=选哪一个子集"**）；**"分配语言"首次在 GS 域显式化**（区域配额——与库内 SABC 配比工作的"倒挂分配、防止单一区域吸走预算"直接同构）。

## TL;DR

**现有剪枝的病灶（原文第一句）**："依赖全局 Top-K + 独立重要性分数，**缺乏区域预算控制、忽略联合删除效应**，无法保证全场景均衡的重建质量。"

```text
SplatQ = 两层分工（"分预算"与"做选择"分开）

① 分块（分区）：
   按"空间邻近 + 外观相似"把高斯切成 size-limited 的小块
   ——"全局选择"分解成一批可处理的子问题

② 分预算（配额 + 平滑）：
   按重要性给每个块算保留配额 → 再做"配额平滑"
   ——防止预算向少数高温区域过度集中（budget smoothing）

③ 做选择（联合打分）：
   用"代表性像素颜色对高斯 opacity 变化的响应"构造删除效应的二次近似：
   · 单项响应（independent）  · 成对交互（pairwise）
   · 配额罚项（quota penalty）
   → 写成 QUBO（二次无约束布尔优化）目标
   → 用 SA（模拟退火）或 CIM（相干伊辛机）求解"联合选择"

结果：5 个 benchmark 上，聚合重建质量 / 空间质量均衡 / 渲染速度全面优于 SOTA；
消融：配额平滑 → 改善局部细节；交互建模 → 固定预算下选择质量更好。
```

## Problem

- **Top-K 的两个盲点**：
  1. **无区域预算控制**——重要性是全局分数，高关注区（或高梯度区）会把预算吸走，其他地方崩坏；"cannot ensure balanced reconstruction quality across a scene"（**质量均衡**是显式目标）；
  2. **忽略联合删除效应**——删除不是独立事件：A 和 B 单独删没事、一起删就出洞（pairwise interaction）；Top-K 对每个高斯独立打分，天然看不到这种"组合效应"。
- **问题重定义**："在固定预算下，选出最优子集" = 一个**组合优化问题**（不是排序问题）。

## Core Idea

三层递进，每一层都在"就近修正"上一层：

1. **分块**——把可组合爆炸的全局选择拆成"size-limited blocks"的子问题（与用户熟悉的"先定域、后算局部"同族：[[2026-09-27-Plate-Local River Generation|板块局部河流生成]] 的"确定性划分让局部独立"在 GS 剪枝上的版本）；
2. **配额平滑**——分配层的"反独占"机制：**不平均分（那会浪费），也不赢家通吃（那会崩坏）**——"smooth importance-based block retention quotas to prevent excessive budget concentration"；
3. **二次近似**——用渲染可见量（像素颜色）对 opacity 的响应来衡量每个高斯的贡献，**并把"成对交互"显式写进目标函数**（individual + pairwise + quota penalty → QUBO）。

## Technical Approach（按摘要可核部分）

- **分块依据**：spatial proximity（空间邻近）+ appearance similarity（外观相似）；
- **选择目标**：QUBO（Quadratic Unconstrained Binary Optimization）——二次无约束布尔优化，标准形式：
  ```text
  min  Σ_i c_i x_i + Σ_{i<j} q_ij x_i x_j     （x_i ∈ {0,1}：保留/删除）
  ```
  其中 c_i 来自单项响应、q_ij 来自成对交互、另有配额罚项——**"剪枝"被翻译成"自旋玻璃/伊辛问题"的标准语言**；
- **求解器**：模拟退火（SA，经典）**或相干伊辛机（CIM，物理/光学求解器）**——把"非线性光学的自然计算硬件"用作图形学组合问题的求解器（本库"用对工具"家族的新物种：不是近似、不是神经网络，是**物理退火/量子启发硬件**）；
- **评估**：5 个 benchmark；三组消融（budget smoothing / interaction modeling / 联合）。

## Key Contribution

- **"分预算"与"做选择"的解耦**（对用户最值钱的一条）：区域配额解决"macro 分配"，联合选择解决"micro 选址"——**两级之间用配额罚项衔接**。这与用户的 SABC 预算配比（总量按档位倒挂分配 + 档内按技能选点）是**同一结构的两个域**；
- **"联合删除效应"的显式建模**：剪枝文献长期用独立重要性分数，本篇证明"成对交互"在固定预算下可被量化并改善选择质量；
- **伊辛机入场**：CIM 作为 QUBO 求解器出现在图形学管线里（此前典型场景是组合优化/机器学习）。

## Why It Works

- 重建质量对"删除哪些高斯"的敏感度是**空间不均匀**的（平坦区删一个无所谓、高频区删一个出洞）→ 均匀/全局策略必然错配，**"配额 + 平滑"是对这种不均匀性的第一层修正**；
- 同一高斯对的贡献有**重叠**（两个高斯补偿同一区域）→ 独立打分系统性高估"删一个的影响"、低估"删一对的影响"，**二次项是对这种重叠的第二层修正**。

## Limitations（以现有信息可判的部分）

- QUBO + SA/CIM 的**求解开销**与高斯数量、块大小的关系摘要未给（在线剪枝 vs 离线资产化两种用法未明确）；
- 5 个 benchmark 的绝对数字、baseline 清单未展开（正式 PDF 放出后需核）；
- CIM 需要特定硬件（相干伊辛机）；SA 路径的实用速度待核；
- 与 **Budgeted-GS** 的"单参数连续扫"范式是两条路线（**组合离散选择 vs 连续预算曲线**），互补关系待对比。

## Game Development Relevance

- **GS 资产化 / 流送预算**：GS 内容进入游戏管线的现实约束是"每场景 primitives 配额"——SplatQ 回答的正是"**配额怎么分、点在哪儿**"；
- **质量均衡是验收项**：不是平均 PSNR，而是"全场景没有局部崩坏"（空间质量均衡是显式指标）——与用户评审 VFX/画质的实际口径一致；
- **"倒挂分配"的独立样本**：配额平滑 ≈ "防止单一区域吸走预算"，是用户 SABC 配比原则在 GS 域的第三种语言（前两种：[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 的误差曲线 / [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]] 的采样数档位）。

## Unreal Engine Relevance

- 未承诺引擎集成；可想象路径：GS 插件 / 影视级 GS 资产的**离线预算化工具**（资产入库前按"场景配额"剪枝）——对应 UE 侧"资产阶段细分、运行时吃烘焙结果"的既有分工（见 [[Subdivision Surfaces]] 的引擎侧注记）。

## Technology Evolution

```text
3DGS 压缩/预算线（本库账本）：
  2023 3DGS 原版（全量 primitives）
  → 2026 压缩前沿平坦（"增益来自冗余、不是容量"——Budgeted-GS 引）
  → 2026-10 ★ Budgeted-GS：误差 ∝ N^(-1/2)（Zador）+ 因子树 LOD + 单参数 c 扫
  → 2026-10 ★ SteadySplats：训练正则 + 时空重采样（"生干净"）
  → 2026-10 ★ SplatQ（本篇）：预算 = 组合选择（区域配额 + 联合删除 + QUBO/Ising）
       ↓
  问题演化：砍多少（连续）→ 怎么压（结构）→ 选哪些（离散组合）
```

## Relationships

### Based On

- 3DGS 剪枝文献体系（未逐一列出——正式 PDF 放出后补 baseline 清单）；重要性分数（opacity / 梯度类）作为**块级配额**的输入信号。

### Extends

- **Top-K 剪枝范式**：从"独立分数的排序"扩展为"配额 + 交互的组合优化"；
- **组合优化的求解器谱系**（补充本库）：SA（经典统计物理）→ **CIM（相干伊辛机，物理硬件）**。

### Related

- [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]——同一条"预算线"的另一面（连续曲线 vs 离散选择）；
- [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction|CompactSplat]]（同日入库）——**"按内容分配"的姊妹样本**：一个在"重建时生成端分配"，一个在"剪枝时保留端分配"；
- [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]——"档位 = 采样数"与本篇"档位 = 保留集合"并列。

### Followed By

- 待观察：ICLR 2027 结果 / 代码放出 / CIM 路径的复现性。

## Personal Knowledge State

- Normal（结论层 30 分钟：分块 → 配额平滑 → QUBO 三步；QUBO 本身不需要碰，只需知道"它把'选哪些'写成标准组合优化、交给退火/伊辛机"）。

## Learning Value

- **"分配问题"的通用骨架**（对用户工作最直接）：*先分域 → 再分预算（配额 + 反独占平滑）→ 再在域内做联合选择*——可直接映射到特效预算的"技能分组 → 按 SABC 配额 → 组内选资源"三层；
- **"独立打分为什么不够"**的一句话：删一个的影响 ≠ 把删除影响相加（重叠效应）——评估任何"逐项打分再 Top-K"的工具时先问这一句。

## Visualization

![[分配三连_SplatQ、CompactSplat、H2-Ray 图解]]

## Notes

- **未公告先见记录**：本条经 API 通道于 2026-10-12 捕获（提交 10-09、listing 不可见）——"周五黑窗"解除后的第一批可见条目之一；
- 正式 PDF 放出后需补：benchmark 数字 / baseline / CIM vs SA 的实用对照。
