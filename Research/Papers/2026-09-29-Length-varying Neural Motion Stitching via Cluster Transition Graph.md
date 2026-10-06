---
type: paper
title: "Length-varying Neural Motion Stitching via Cluster Transition Graph"
authors: [Haemin Kim, Junghyun Nam, Seokhyeon Hong, Vanessa Tan, Junyong Noh]
year: 2026
published: "2026-09-29 (arXiv v1); SIGGRAPH Asia 2026 Journal Track"
venue: "SIGGRAPH Asia 2026 (Journal Track)"
url: "https://arxiv.org/abs/2609.37167"
code: "Coming soon（项目页标注）"
project_page: "https://haem-k.github.io/nms/"
category: [animation, motion-synthesis, motion-stitching, neural, graph-search]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research（代码即将放出；推理 14.5 ms 已接近实时）
user_level: Normal
status: unread
tags: [animation, motion-stitching, motion-graph, transition]
---

# Length-varying Neural Motion Stitching via Cluster Transition Graph（KAIST, SIGGRAPH Asia 2026）

## TL;DR

**运动缝合的"长度问题"被图解决了**：给定两段动作，先把它拆成**离散动作块**（CVQ-VAE 聚类），这些块构成一张**聚类转移图**（节点 = 簇，边 = 可转换关系）；在图上找**最短路径**，路径长度**隐式决定过渡长度**，路径本身当**引导序列**喂给 Transformer 生成最终过渡姿势，并预测"第二段动作该放哪"（根位移）。

与库内既有方法的差别一句话：**别人把"过渡长度"当超参（固定值 / 手动指定 / 直接估秒数），本文把它当"路径搜索的副产品"**——动作差得越远，路径越长，角色获得越多时间调整姿态（示例：同样两段输入，因脚相位不同自然分出 64 帧 vs 80 帧两个版本，后者自动补了一步）。

> **一句话定位**：[[Kovar — Motion Graphs (2002)]] 的"**相似度 → 图 → 搜索**"结构，在 2026 年以"聚类转移图 + 神经生成"的形态重开——**固定 1/3 秒混合窗升级为自适应变长转移**（本文最小转移长度 0.3 秒即采用 Kovar 2002 的值）。KAIST VML（Junyong Noh 组）。

## Problem

**运动缝合（motion stitching）**：把两段已有动作无缝接成一段新动画。已有方法的限制：

1. **需要手动选过渡区间**，或**假设固定过渡长度**——限制了"能连的动作对"范围；
2. 输入动作差异大时（爬行 → 投篮 → 慢速移动），**固定长度不够用**——角色没有足够时间调整姿态，过渡会"跳"或"僵"；
3. "显式算秒数"路线（直接回归时长）**无法解释为什么是这个长度**，泛化差。

关键观察：**过渡长度应该由"两段动作在运动语义空间里隔多远"决定**——而这个"隔多远"，可以用**已有的运动库结构**来度量，不需要新增网络。

## Previous Work

- **对齐 + 混合路线**：把第二段对齐到第一段再插值（IB 等）——快（1.5 ms），但质量上限低；
- **in-betweening 路线**：给起止帧插中间帧（CondMDI / NeMF / SILK 等）——能生成，但延迟高（NeMF 22 s / CondMDI 15 s）或约束不适配缝合（首尾必须分别为两段输入的边界）；
- **变长过渡的探索**：有工作用启发式或额外网络估计时长——本文对比组之一（BC = 从动作数据里预测时长）；
- **图路线的前史**：[[Kovar — Motion Graphs (2002)]] 用全库帧对的 O(F²) 相似度找过渡点——本文自述"Many existing methods computed pairwise distances between all possible pose combinations…（Kovar et al.）"；其做法**不搜帧对，搜簇**（用聚类把 O(F²) 的图变成可维护的转移图）。

## Core Idea

**三段式：聚类 → 图寻路 → 神经生成。**

```text
输入 A、B
   ↓ ① 运动聚类（CVQ-VAE 编码动作块 → 码本簇）
A、B 各块映射到簇 ID
   ↓ ② 聚类转移图寻路（A* 最短路）
路径 = 【过渡长度 + 引导序列】
   ↓ ③ Transformer 生成器（引导 + A、B 原动作）
过渡姿势 + B 的根放位
   ↓
无缝连接的新动作
```

**②的边权**（原文）：转移代价 = 走这条边的通行代价 + **边界处根差异**（"在接缝处转换的成本"）——两个权重项用一个系数 λ 配比。**路径长 = 过渡帧数**（"We estimate the transition length indirectly through graph traversal"）。

**③的巧思**：网络除生成过渡姿势外，**预测"第二段动作的根目的地"**——即第二段该平移到哪才能无缝续上（原文：destination of the root to align the second input motion）——把"对齐"从人类手工/后处理变成网络输出。

## Technical Approach

- **骨架**：髋关节为根；对抗训练时另用头关节；60 FPS 数据；
- **动作块**：最小转移长度 **0.3 秒**（"following Kovar et al. (2002)"）——块是拼装长过渡的原子；变长数据由多块拼接而成（训练用滑窗 + 不同 stride 生成不同长度样本）；
- **聚类**：CVQ-VAE 变体（无监督），码本映射用**余弦相似度**最近码；
- **寻路**：A* 最短路（网络 = 簇转移图）；可加**用户控制方向**（同一对输入按指定方向产生不同过渡）；
- **生成器**：Transformer 编码器骨干；**LSGAN** 损失的对抗训练；
- **确定性与可变性**：固定代价设置时输出**确定性**（不产生随机变化；换代价可换结果）——**取舍在"可复现"侧**（原文：无 generative variation，但换来无需额外训练即可复用）。

## Key Contribution

1. **变长过渡的"结构化"估计**：过渡长度 ≡ 聚类图最短路长度——**不用额外网络、不用手动指定、不显式算秒数**，且天然可解释（走了多少个语义簇）；
2. **引导式生成**：路径当"每姿势初始化引导"，把生成的不确定性压到姿势细节层，缓解 in-betweening 的结构歧义；
3. **根目的地预测**：缝合最后一块手工活（对齐落位）交给网络；
4. **工程侧面**：**14.5 ms 单次缝合**（推理，RTX 3090）——在 60 FPS 帧预算（16.7 ms）之内，是库内缝合类方法中最接近可用的（见下表）；LAFAN1 训练、三个数据集（LAFAN1 / 100STYLE / Mixamo）**免重训**测试。

## Results

**推理耗时对比（ms，原文 Table）**：

| 方法 | 长度估计 | 运动生成 | 总计 |
|---|---|---|---|
| IB（对齐+混合） | – | 1.5 | **1.5** |
| RMR | – | 2.6 | 2.6 |
| SILK + BC | 0.7 | 6.5 | 7.2 |
| TS + BC | 0.7 | 6.5 | 7.2 |
| RMIB + BC | 0.7 | 50.3 | 51.0 |
| NeMF / CondMDI 类 | – | ~15–22 s | **不可实时** |
| **NMS（本文）** | **6.3** | **8.2** | **14.5** |

**质量指标**：FID / detail distance / jitter / FS（foot skating）/ GP（ground penetration）+ L2P / L2Q（Harvey et al. 协议）；变长组整体优于固定长度组与"直接算时长"组；消融显示去掉引导（guide）会出现**接缝处 popping**——证明"路径当引导"不是装饰而是结构约束。

**行为示例**：脚相位不同 → 64 vs 80 帧（自动加一步）；不同动作对（爬行/投篮/慢速移动）都能连；未见动作（跨数据集）零重训可用。

## Why It Works

- **把"长度"变成"结构问题"**：固定长度假设的是"所有动作对都在相似距离上"；图路径承认"隔多远走多远"——**长度估计与生成共用同一个语义空间**，因此不需要单独学一个时长回归器；
- **离散化换稳定性**：簇是离散的、图是确定的——搜索不会漂移（对照"直接回归秒数"的连续值回归的不稳定）；
- **生成被约束在两条已知动作的边界内**：首尾锚定 + 路径引导 ⇒ 网络只需在"已知结构"里补细节，比开放生成容易。

## Limitations

- **A* 是延迟大头**（6.3 ms / 14.5 ms），原文自述"实时应用里当用 A* 时推理时间可能成为限制"——实时场景目前只有"离线缝合 / 预计算"或将来换更快寻路；
- **确定性换可复现**：不产生同一输入的多变体（做内容工具时是优点，做变化度需求时是缺点）；
- **无环境感知**：缝合结果不与场景互动（原文列为未来方向，点名"与周围环境的交互"）；
- **依赖聚类质量**：码本覆盖不到的运动域（原文提到某数据集 root movement 变化小、码本较紧）会限制可连范围；
- **未开源**（代码 coming soon，截至本日未放出）。

## Game Development Relevance

- **对 [[Motion Matching]] 瓶颈线的新材料**（你是首要目标读者）：把"过渡问题"的第三个子问题（**切多久**）从"配置项"变成**可估计量**——UE/工业界现状是 Blend Time 手工配置或按动作对约定，这条路线展示"图结构估长度"的可行性；
- **运行时可行性已入视野**：14.5 ms 单次缝合 + 确定性输出——对**预计算缝合**（技能动画拼接、过场、视频/演示）是马上可用的形态；对运行时（Motion Matching 之上的 transition 层）是"下次再做快一点"的候选；
- **内容生产形态**："任意两段素材自动缝合 + 方向可控"——动作素材库的**复用杠杆**（对照 [[Kovar — Motion Graphs (2002)]] 的镜像数据实验：少量素材生成新动作）；
- **对分档体系的类比**：变长过渡 ≈ "**按动作差档位给不同预算**"——差异大就是"高档位过渡"（长、需生成），差异小就是"低档位"（短、可混合）——**过渡成本跟着语义距离走**。

## Unreal Engine Relevance

- UE 动画系统的**过渡层**（State Machine / Blend Space 的 transition 规则、Motion Matching 的 continuing pose 匹配）目前都以固定/半手工时长为默认；本路线可作"**自适应过渡长度**"的概念原型（工程化仍需：寻路加速、与 PoseSearch 数据库共用聚类结构）；
- 本方法"**离线图 + 在线神经网络**"的混合形态与 UE 的**运行时预算结构**（预计算资产 / 运行时轻量推理）兼容，但推理需 GPU（RTX 3090 测量）——移动端适配未验证。

## Technology Evolution

```text
2002  Kovar：图 + 固定 1/3 秒混合窗（过渡点自动、长度固定）★ 库内经典
        ↓
2003-2010  过渡质量优化（注册曲线）/ 参数化 / Motion Fields
        ↓
2015-2020  检索时代（Motion Matching / Learned）——过渡变成"每帧混合"
        ↓
2022-2025  in-betweening / 生成式过渡（固定长度或显式时长估计）
        ↓
2026  本文：过渡长度 = 聚类图路径（结构化估计）+ 引导式神经生成 ★ 入库
      ——"图负责规划、生成负责细节"，变长、可控、几乎实时（14.5 ms）
```

## Relationships

### Based On

- [[Kovar — Motion Graphs (2002)]] —— 图搜索求连接结构的直系祖先；**0.3 秒最小转移长度直接采用 Kovar 值**（原文 "following Kovar et al. (2002)"）；
- CVQ-VAE 离散聚类（运动 tokenization 线；库内对照 [[MotionBricks — Scalable Real-Time Motions]] 的 token 化路线）。

### Extends

- 变长过渡的估计路线：从"启发式 / 额外时长网络"扩展到"图路径副产品"；
- "对齐"自动化：根目的地从手工调整变为网络预测。

### Related

- [[Motion Matching]] / [[Motion Generation]] —— **缝合 = "检索结构 + 生成细节"的混合形态**：介于两者之间的新物种（检索负责"走哪条路"，生成负责"每帧长什么样"）；
- [[Learned Motion Matching (Holden 2020)]] —— 同样以"压缩表示 + 学习化"改造数据驱动动画；区别：LMM 学习**压缩**，本文学习**过渡**；
- [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting]] —— 同为"动画后期修正"类：ReFM 把廉价解当**初始化**，本文把路径当**引导**——**"廉价参考解的角色"两连样本**（都只当起点，不当目标）。

### Followed By

- 待观察：代码放出后的工程化（寻路加速 / 与引擎动画系统集成）、环境感知扩展（原文点名）。

## Personal Knowledge State

Current Level: **Normal**。你已懂"动画管线 + 过渡机制"（Easy 层），本文要拿走的是三层 Normal 抽象：

1. **过渡长度 = 语义距离的图度量**（不是超参，不是回归值）；
2. **"离散结构 + 神经细节"的分工**（图管规划、网络管姿势）——与 [[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]]"优化到眼睛"、[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]"生成只做决策层"是同一哲学的新样本；**"结构交给搜索、细节交给生成"**；
3. **成本账**：14.5 ms 分解为"寻路 6.3 + 生成 8.2"——把延迟账**拆到阶段**是评估任何缝合/生成方案的第一动作。

## Learning Value

- **与今日经典构成教科书级对子**：[[Kovar — Motion Graphs (2002)]] 的固定窗 → 本文变长路径，**24 年跨度的"同一问题、两代答案"**——读一读能直观看到"默认值 → 自适应"的进步模式；
- **对 Motion Matching 瓶颈**：这是 9-10 以来**第一份直接改造"过渡长度"**的材料（FlexMoGen 改造风格、ReFM 改造修正目标、本文改造过渡长度）——瓶颈线的材料又厚了一层；
- **对"生成只做决策层"家族**：第 6 例（Magpie / PartLLM / ProxyBuild / Mira-Scene / OGRE 之后）——本次的决策层是"**过渡的宏结构**"。

## Visualization

![[运动缝合_Kovar 2002 与 2026 变长转移对照图解]]

## Notes

- **窗口状态**：9-29 提交、9-30 公告（Wed 9-30 listing，本日窗口）；项目页含大量可视化（脚相位差异 / 方向控制 / 与 baseline 对比）；
- **机构**：KAIST Visual Media Lab（Junyong Noh 组——VML 是 SIGGRAPH Asia 常客；Noh 组此前作品偏面部/表情，本篇为其动作线新作）；
- **原文自述定位**："motion stitching 是 motion in-betweening 的推广"——**两头都被外部给定的生成问题**；
- 数据集：LAFAN1（训练）/ 100STYLE / Mixamo（测试，免重训）；指标协议沿用 Harvey et al.。
