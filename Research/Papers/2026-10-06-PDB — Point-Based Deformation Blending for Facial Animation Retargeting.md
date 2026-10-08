---
type: paper
title: "PDB: Point-Based Deformation Blending for Facial Animation Retargeting"
authors: [Sihun Cha, Hyeonseung Shin, Suah Yu, Junyong Noh]
year: 2026
published: "2026-10-06（arXiv v1, 2610.08672）"
venue: "arXiv Preprint（KAIST Visual Media Lab——Junyong Noh 组）"
url: "https://arxiv.org/abs/2610.08672"
code: ""
project_page: ""
category: [facial-animation, retargeting, character-animation, deformation]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Research（工程距离短：目标侧权重算一次 + 每帧控制点预测 + 矩阵乘法；无 cage/坐标/解码器/全局求解——183 帧序列端到端 1 秒；但无引擎集成与代码）"
user_level: Normal
status: unread
aliases: [PDB, Point-Based Deformation Blending, 点基变形混合, 面部表情重定向]
tags: [facial-animation, retargeting, character-animation, deformation]
---

# PDB: Point-Based Deformation Blending for Facial Animation Retargeting（Cha et al. 2026）

> **入库 2026-10-08（Run 30）。** **KAIST Visual Media Lab**（Junyong Noh 组）——与库内 [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|NMS 运动缝合]] 同一实验室（**"身体线"与"面部线"同址**）。
> **一句话定位**：mesh-agnostic 面部重定向的**第三种分解**——不逐顶点预测（会出表面噪声）、不逐三角 Jacobian 全局求解（会全局耦合），而是**"一组变形的控制点 × 一组按目标形状预测的稀疏混合权重"**，目标网格 = 两者的**矩阵乘积**，就这么多。
> **开场即命题（原文摘要）**：**without a predefined cage, precomputed coordinates, a learned per-element deformation decoder, or a global reconstruction solve**——四个"不需要"，每个都对应前人方法的一个结构性依赖。

## TL;DR

给定源中性脸 + 源表情脸 + 目标中性脸，输出目标表情脸。PDB 把变形拆成两个**显式几何因子**：

$$\underbrace{v}_{\text{变形控制点（K 个）}=\ \phi(\text{源中性},\ \text{源表情})}\qquad\times\qquad\underbrace{w}_{\text{混合权重（}N_t\times K\text{）}=\ \psi(\text{目标中性})}$$

$$\hat{P}_{\text{tgt}}=w\,v$$

- **$v$ 随每个源表情变**（表达"在动什么"）；**$w$ 每个目标身份只算一次、跨帧复用**（表达"目标脸上怎么分布"）；
- 权重用 **ReLU（允许精确 0）+ 列归一 + 行归一** 参数化——非负、稀疏、行和为 1；
- 只用**自重建监督**训练（无成对跨身份数据）——跨身份泛化由**分解结构**而不是数据承担；
- 实测：**183 帧序列端到端 1 秒**（对手 NFR 25 s / NFS 17 s）；Multiface 11,649 帧 **50 秒**（≈ 4.3 ms/帧摊销）；**30 人用户研究中原表情保真度与视觉质量双双第一**；能泛化到大幅风格化的目标头（图 14）。

## Problem

**Mesh-agnostic 面部重定向的两难（原文 §1 的失败模式分析）：**

| 表示 | 强项 | 失败模式 |
|---|---|---|
| **逐顶点位移**（NFS，Cha et al. 2025——本文作者的上一篇） | 捕捉细腻表情 | **稠密网格上出现局部表面噪声**（预测噪声逐点传播） |
| **逐三角 Jacobian**（NFR，Qin et al. 2023） | 全局求解 → 表面规则光滑 | **全局求解 → 远离区域的意外耦合**（眼睑、内嘴出现不该有的形变） |

需求：**既要局部表情保真，又要表面结构干净，还不许用全局求解把两者都污染。**（原文：*"motivate a deformation representation that preserves local facial motion and surface structure without requiring a reconstruction process via global solve"*）

## Historical Context / Previous Work（原文谱系）

```text
控制式变形（control-based deformation）传统：
  LBS 线性混合蒙皮（1988 起）——骨骼控制 × 逐顶点权重
  Cage-based（均值坐标 2005 / 调和坐标 2007）——cage 顶点 × 几何坐标函数
  CSFB（Kavan et al. 2024）——面部 blendshape → 线性混合蒙皮表示
        ↓
神经化：
  NeuralCage 2020（预测 cage 偏移）/ KeypointDeformer 2021（学稀疏关键点）
  DeepMetaHandles 2021（元手柄 + 预计算双调和坐标）
  VBC 2023（神经场参数化重心坐标）/ NeuralMLS 2022（稀疏控制点 + MLS 求解）
        ↓
面部专用局部表示：
  SPLOCS 2013（稀疏局部逐顶点分量 → 动画编辑）
  CUBE 2026（高维控制特征格 + B-spline 局部混合 + 残差 MLP 解码）
        ↓
★ 2026 PDB：预测"变形控制点 + 混合权重"两者本身；无 cage / 无预计算坐标 /
  无解码器 / 无全局求解
```

**关键差异（原文自述）**：
- vs **NFS**：NFS 也用"空间局部化蒙皮权重"，但**权重只是神经解码器的条件**；PDB 的权重**本身是两大显式几何因子之一**，直接做矩阵乘积——**"权重即几何"而不是"权重喂解码器"**；
- vs **CUBE（2026）**：CUBE 的局部性由 B-spline 基显式保证 + 混合后仍需残差 MLP 解码；PDB 的局部性是**学出来的**（ReLU 精确零 + 实测局部支持，图 7），混合后**无解码器**；
- vs **NeuralCage**：不预设 cage；vs **SPLOCS**：不是固定分量 + 激活系数，而是**表情相关控制点 + 身份预测权重**的双向预测。

## Core Idea

**"把'在动什么'与'在谁身上动'拆成两个张量，并让它们用各自的频率更新。"**

- **更新频率的分解**（本篇的操作核心）：
  - $w$（目标形状相关）——**每身份算一次**（推理时一次性开销；跨帧复用）；
  - $v$（源表情相关）——**每帧算**；
- **重建 = 纯矩阵乘法**——没有求解器、没有解码网络、没有几何坐标预计算。**整个方法学起来像 LBS，但控制点是被预测出来的**；
- **非负 + 精确零**的权重参数化（列归一 → ReLU → 行归一）给"稀疏支持"留出通道——与"多数 delta blendshape 空间上局部"的领域先验一致（原文引 Lewis et al. 2014）；
- **只用自重建监督**：训练时源与目标同身份，`w` 与 `v` 由同一个重建损失联合学习；泛化到跨身份靠"$v$ 与 $w$ 来自两个解耦的编码器"这一结构（**"结构代替数据"**）。

## Technical Approach

### 两个编码器（架构极简）

- **表达式点编码器 $\phi$**：输入 = 源中性 + 源表情的**逐顶点特征拼接**（位置 + 法线，6+6 维）；输出 $K$ 个变形控制点 $v\in\mathbb{R}^{K\times3}$（**训练前选定超参，本文 K=512**）；
- **身份权重编码器 $\psi$**：输入 = 目标中性网格；输出权重矩阵 $w\in\mathbb{R}^{N_t\times K}$；
- 两者都是 MLP，由"**特征变换块**"堆成——灵感来自 PointNet 的 STN（但把"无偏线性映射"换成"逐特征仿射 $y=a\odot x+b$"，且仿射参数由辅助 MLP 按输入条件生成）；
- **要求**：源内部需要顶点对应（中性↔表情），**源与目标之间不需要任何对应**——这是 mesh-agnostic 的落点。

### 权重参数化（两段式）

1. 列方向：$\tilde{w}_i=\mathrm{ReLU}\big(z_i/\max(\|z_i\|_2,\varepsilon)\big)$——列归一使输出尺度稳定，ReLU 给非负 + **精确零**（硬稀疏）；
2. 行方向：$w_i^j=\tilde{w}_i^j/(\sum_k\tilde{w}_k^j+\varepsilon)$——**每行（每个顶点）和为 1** → 重建 = 控制点的**仿射组合**（这也是"blending"名字的来源）；
3. 副作用（非预期但无害）：全零行会把该顶点重建到原点——原文按"参数化不额外加约束"处理。

### 训练目标（自重建 + 内外面部掩码）

$$\mathcal{L}_{\mathrm{recon}}=\tfrac{1}{N_t}\big\|M^{\mathrm{in}}_{\mathrm{tgt}}\odot(\hat{P}^{e}_{\mathrm{tgt}}-P^{e}_{\mathrm{GT}})\big\|_F^2+\tfrac{1}{N_t}\big\|M^{\mathrm{out}}_{\mathrm{tgt}}\odot(\hat{P}^{e}_{\mathrm{tgt}}-P^{n}_{\mathrm{GT}})\big\|_F^2$$

- 内脸区域学"表达式"，**外圈（头颈/发际）锚定到中性**（防止重定向把整个头带动）；
- 掩码用**以鼻尖为中心的 hat 函数**（$r_0=1.0$ 全权重、$r_1=2.25$ 过渡到 0，五次平滑 $10t^3-15t^4+6t^5$）——"连续掩码"而非硬分界。

### 实验与数字（原文 §4）

| 项目 | 结果 |
|---|---|
| 数据集 | ICT-FaceKit、Multiface；**自采集两段（用 Epic 的 Live Link Face 捕捉）** |
| 运行时（端到端，含目标特异预处理，RTX A5000） | **183 帧：1 s**（NFR 25 s / NFS 17 s）· 1,147 帧：**6 s**（79 s / 34 s）· 11,649 帧：**50 s**（17:32 / 2:29） |
| 局部表面保持（内脸 Laplacian MSE，×10⁻⁴ mm²，avg） | NFR **1.1745**（最佳）→ PDB **1.5882**（第二）→ NFS 2.7837 → NC 3052.7（崩溃） |
| 中性重建（×10⁻⁶ mm²） | NFS 8.148 最优；PDB 8.779 |
| 用户研究（30 人；表情相似度强制选择 + 质量 Likert） | **PDB 两项均第一**；NC 双双垫底；NFR/NFS "看着还行但保不住源表情" |
| 泛化 | 风格化目标头（比例/几何大幅不同）直接可用（图 14） |

> 一个有意思的对照轴：**NFR 的表面质量仍是最高的**（全局求解的正收益），但代价是"耦合错误"（眼睑/内嘴的意外形变）与 15 秒级预处理（Laplace–Beltrami 等网格相关预计算）；PDB 的立场是"**用可接受的表面质量损失，换掉全局耦合与预处理**"——**联合评估（表情精度 × 表面保持）而不是单指标**。

## Key Contribution

1. **点基双因子分解**：把面部变形表示为"变形控制点 × 混合权重"，且**两者都是被预测的**（不是预设 cage、不是预计算坐标）——前人要么预测变形本身（噪声）、要么预测 Jacobian 再解（耦合）、要么预测"权重喂解码器"（多一层网络）；
2. **四个"不需要"**：无 cage / 无预计算坐标 / 无逐元素解码器 / 无全局求解——**推理结构 = 预测 + 矩阵乘法**（工程化距离极短）；
3. **"权重即几何"**：权重从"解码器条件"升格为"显式几何因子"，重建直接可解释（图 1 的随机色权重可视化）；
4. **自重建训练即可跨身份**：分解结构本身承担泛化（无成对跨身份监督）。

## Why It Works

1. **更新频率分解**（个人解释，本篇最可迁移的一条）：**"只随身份变的东西绝不每帧算"**——$w$ 一次性、$v$ 每帧；两类信息各自走各自的编码器，天然解耦（跨身份时"换权重、保控制点"成立）；
2. **在两个失败模式之间选第三条路**：逐点预测的病 = 噪声逐点传播；全局求解的病 = 误差全局传播；**低维控制点 + 稀疏线性混合**把两者都绕开——控制点数量 K≪N_t 提供"低维瓶颈"（压缩即正则），稀疏权重提供局部性；
3. **非负 + 精确零**让"稀疏支持"可从数据中学出（而不是像 cage 类方法那样靠几何坐标硬保证）——**用参数化结构替代几何结构**；
4. **可解释性是副产品**：权重矩阵行和 1 + 稀疏 → 直接可视化"每个控制点管哪些区域"（图 7）。

## Limitations

1. **高频细节丢**：皱纹、突发微表情捕捉不住——控制点倾向聚在形变大的区域（眼、嘴），"wrinkle 们太细且因人而异"（原文自述）；
2. **K=512 是启发式**，非最优；增大 K 会牺牲紧凑性并放大 $N_t\times K$ 权重的存储/乘法成本；
3. **无代码/项目页**（截至入库）；**venue 未定**（预印本）；
4. **训练数据范式仍是"配准过的网格对"**（ICT/Multiface 类）；对扫描/非配准管线的适配未验证；
5. 每次推理仍需为每个**目标**做一次 $w$ 预测（一次性），且 $w$ 是稠密存储（$N_t\times K$）——大网格上的内存账未展开。

## Game Development Relevance

**4/5。面部重定向是"捕捉 → 引擎"管线的中段刚需，本篇的方法形态与游戏资产流水线的耦合点非常直接。**

- **管线位置**：演员面部捕捉（**评测里直接用 Epic 的 Live Link Face 采集**）→（**重定向 = 本篇**）→ 目标角色头（不同拓扑/分辨率/风格化）。游戏里的等价问题：**同一段表情数据要打给主角 / NPC / 不同头模**——mesh-agnostic + "权重一次烘焙"的形态正是**"每个角色头资产烤一次、之后每帧几乎白嫖"**的生产模型；
- **工程账（推断）**：4.3 ms/帧摊销（含矩阵乘法 + 控制点预测）——对过场/对话特写是**实时可行量级**；"无全局求解 + 无网格预计算"意味着**新头模上线不需要 Laplacian 预处理**（NFR/NFS 各要 ~15 s 级目标侧准备，且失败要重来）；
- **风格化头**（图 14）对游戏尤其重要：卡通比例、极端形状的头模是游戏常态，"大幅几何差异仍能重定向"是其卖点之一；
- **与 [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA]] 对照（"分解家族"两样本）**：GALA = "**一套共享基 + 身份特异的线性组合**"（基共享、系数个体化）；PDB = "**目标特异权重（一次）+ 源特异控制点（每帧）**"——同一母题"把常量与变量分开、把共享与个体分开"的面部情态两解；
- **与 [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]] 互为镜像**：ReFM = **身体**重定向的"欠定问题形式化 + 流模型精修"；PDB = **面部**重定向的"表示层分解"——**身体/面部两条重定向线在库内同日前后配对**（有意思：这也说明重定向本身是一个跨部位的独立问题域）。

## Unreal Engine Relevance

- **捕捉侧**：Epic 的 Live Link Face（ARKit blendshape 流）出现在本篇评测管线中——如果做引擎内验证，这是现成入口；
- **可映射的系统（推断，非实装断言）**：
  - **MetaHuman / 面部 rig 的"重定向层"**：把"表情控制点 × 目标权重"当成一类**离线重定向烘焙**（每头一次）来理解；
  - **Control Rig / 骨骼解算**：PDB 的矩阵乘积形态与 LBS 同构——**在引擎里就是"比骨骼更自由的线性混合器"**；
  - **工具侧**：如果要在 UE 里搭"一串表情数据批量打给 N 个头"的流水线，本篇给出的是"无求解、可批量、可并行"的算法形态参照。
- ⚠️ 边界：本篇是研究方法（无代码、无引擎集成），不要外推到"可直接替换 MetaHuman 重定向"。

## Technology Evolution

```text
面部重定向的表示演化（按本文 related work 整理，不夸大）：

几何控制时代：LBS / cage-based（2005–2007 均值/调和坐标）
      ↓
神经辅助控制：NeuralCage 2020 → KeypointDeformer 2021 → DeepMetaHandles 2021
      ↓
两强并立（2022–2025）：
  逐三角 Jacobian + 全局求解（NFR 2023） ←→ 逐顶点位移 + 神经解码（NFS 2025）
      ↓
★ 2026 PDB：点基双因子分解——"预测控制点 × 预测权重 = 矩阵乘积"
      （局部性靠学、泛化靠结构、重建零求解）
```

## Relationships

### Based On

- 控制式变形传统（LBS / cage-based 的思想血脉）；PointNet 的逐样本变换思想（特征变换块的直接启发）

### Contrasts

- **NFS（Cha et al. 2025）**——同作者上一代：权重喂解码器 vs 权重直接当几何因子（"少一层网络、多一层可解释"）
- **NFR（Qin et al. 2023）**——全局求解的耦合失败样本；本文保留其"表面规则性"优点的一半（不求解但权重行和为 1 提供组凸性）

### Related

- [[Facial Animation]] —— 本篇是其"重定向"子主题的库内第一节点（同批入库）
- [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA]] —— "分解：共享 × 个体"的面部对照样本
- [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]] —— 身体侧重定向对照（表示层 vs 优化层）
- [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|NMS]] —— 同实验室（KAIST VML）身体线

### Followed By

- （待观察）无代码放出；watch 作者主页与后续 venue

## Personal Knowledge State

- **user_level: Normal（推断）**。面部动画是库内**新开的分支**（本批同建 [[Facial Animation]] 概念）；本篇的表示层论证（两种失败模式、双因子分解）**无需面部动画背景即可读懂**，数学极少（只有矩阵乘法和两个归一化）。
- **阅读建议**：§1（失败模式对照）+ §3.1/3.2（双因子与权重参数化）+ Table 5（运行时）+ 图 14（风格化泛化）——**约 20 分钟**；想深入再看 §4.1 消融（激活函数 / POU 约束 / 掩码）。

## Learning Value

- **两条可迁移判据**：
  1. **"把'只随 A 变'与'只随 B 变'拆成两个张量，各自按自己的频率更新"**——本篇面部版（身份权重一次 / 表情控制点每帧）；与 [[2026-10-07-DynaConTalk — Wavelet-Constrained Diffusion for Long-Form and Controllable Holistic Co-Speech 3D Motion|DynaConTalk]]（小波频带分解）**同日入库、同一母题**；
  2. **"在两种失败模式之间，先找是否存在'低维瓶颈 + 稀疏混合'的第三条路"**——逐点（噪声传播）vs 全局（耦合传播）之外，控制点方案把误差引到"低维、可解释、可控"的通道里。
- **一条"训练策略"观察**：跨身份泛化不靠成对数据，靠**分解结构 + 自监督**——"能力放在结构里，不放在数据里"的又一例。

## Notes

- **原文核对（arXiv HTML v1 全文抽取核对）**：双因子定义与重建式（Eq 1–4）、权重两段参数化（Eq 5–6）、hat 掩码（Eq 10–13）、自重建训练声明（§3.4）、运行时表（Table 5）、Laplacian 表（Table 4）、用户研究（§4.3，30 人）、中性重建（Table 3）、K 消融（Table 6）、限制（§5）——**均为原文逐条确认**；
- **作者组上下文**：Junyong Noh 组（KAIST VML）在库内已有 [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|NMS]]（Noh 为共同作者）；本篇延续其"动画表示工程"风格（简单结构 + 严谨对照）；
- **与用户笔记的接口（推断）**：你在材质/分档里熟悉的"**参数分层**"（v vs w 的分工）与"**共享与个体分离**"（GALA/PDB）是同一类设计直觉——可当作"资产一次烘焙、运行期复用"的又一面部样本。
