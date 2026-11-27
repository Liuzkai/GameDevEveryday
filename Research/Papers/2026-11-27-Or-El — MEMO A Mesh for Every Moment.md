---
type: paper
title: "A Mesh for Every Moment"
authors: ["Roy Or-El", "Jeong Joon Park", "Ira Kemelmacher-Shlizerman"]
year: 2026
published: "2026-11-25"
venue: "arXiv 预印本（cs.CV / cs.GR），27 页；University of Washington + University of Michigan"
url: "https://arxiv.org/abs/2611.34939"
code: "未给"
project_page: "https://memo-4d.github.io"
category: ["Gaussian Splatting", "Dynamic Scene", "Surface Reconstruction", "Mesh Extraction", "Performance Capture"]
importance: A
historical_importance: 3
game_relevance: 4
production_readiness: "Research（多机位同步采集 + 每场景约 18 min/A100；逐帧网格无时间一致性，不能直接当动画用）"
user_level: Normal
status: unread
aliases: [MEMO, A Mesh for Every Moment, 每帧网格, 时间高斯核运动模型, Blender-Moments]
tags: [gaussian-splatting, dynamic-scene, surface-reconstruction, mesh-extraction, volumetric-capture]
---

# MEMO — 每一帧都有一张网格，因为每个高斯在每一刻都得是"表面"

## TL;DR

动态 3DGS 只优化**外观**。要从它拿到几何，标准做法是渲染一堆深度图、做 TSDF 融合、再 marching cubes——**固定网格 + 浮空块（floaters）**，结果又噪又破。已有的"动态表面重建"方法则几乎都要求**前景 mask + 单个物体**，做不了完整场景。

MEMO 的判断是：**问题不在提取器，在表示**。它给每个高斯挂一个**覆盖整条时间轴的时间高斯核运动模型**（位置 + 旋转都是核的加权和），于是：

- 模型在**每一帧**都成立（不是只在关键帧附近成立）；
- 因此**不透明度可以是常数**——高斯不需要"随时间开关"；
- 因此**每个高斯在每一刻都是一个合法的表面元**，可以携带朝向法线；
- 因此**表面在任意 $t$ 都有定义**，网格是"取出来"的，不是"事后融出来的"。

结果（Blender-Moments，9 个有逐帧 GT 网格的场景）：Chamfer **0.51**、法向 Chamfer **24.6°**、联合 F-score **0.556 / 0.746**，全部第一；训练 **18 min**（同为几何方法的 DG-Mesh 要 **142 min**）。

## Problem

三件事同时不成立，才让"动态场景的逐帧网格"一直没人做干净：

1. **表示层**：4DGS / Deformable-3DGS / SC-GS / FreeTimeGS / AST-3DGS 都只编码外观与运动，**没有表面的表示**（隐式或显式都没有）。几何只通过光度损失被"间接约束"，于是大量 floaters。
2. **提取层**：TSDF 融合 + marching cubes 是**固定分辨率网格**——细化一处要细化全部，且 floaters 会污染融合体。
3. **场景层**：少数做动态表面的方法（D-2DGS / DG-Mesh / 各类 hybrid）**依赖前景 mask 或输入网格**，只能做孤立物体；DG-Mesh 甚至在真实场景上跑不了（Table 1 里那两格是"—"）。

## Historical Context

```text
静态侧（已解决得不错）
2020  NeRF                    体密度 + 体渲染，表面从密度里"长"出来
2023  3DGS                    外观与速度，但没几何
2024  2DGS / PGSR            把高斯压扁成 surfel ⇒ 深度可融合
2024  SuGaR / GOF            Gaussians → 场 → marching tets（不再融合深度）
2025  MILo / Gaussian Wrapping 每高斯带 SDF 或朝向法线 ⇒ 精细表面
2026  (本库已有) T2-Splat     另一边：让拓扑可动

动态侧（本页的位置）
2021–23  D-NeRF / Nerfies / HyperNeRF   变形场把射线搬到 canonical 空间
2024  4DGS / Deformable-3DGS            同一套 canonical-deformation 搬到高斯
2024  Dynamic3DGS / Spacetime-GS        每高斯显式存轨迹（逐帧或特征）
2025  FreeTimeGS / AST-3DGS             让"每高斯的生命周期"可学（位置 + 不透明度随时间变）
2026  D-2DGS / DG-Mesh                  动态表面，但**要 mask、要物体**
2026  MEMO                              把静态侧的"场提取"整条搬到**完整动态场景**
```

关键的历史断点是 2025 年那一批"**让不透明度随时间变**"的工作：它们解决了"一个高斯该在什么时候存在"，代价是**高斯可以在中途消失**——而一个会消失的东西，不能当表面元。

## Previous Work

论文对"为什么以前不行"的归因很干脆，分三族：

| 族 | 代表 | 运动怎么表示 | 为什么拿不到逐帧网格 |
|---|---|---|---|
| **变形场** | 4DGS / Deformable-3DGS / GauFRe | 学一个 $x\mapsto x'$ 的 warp | 运动被邻域共享，几何没有独立表示 |
| **控制点** | SC-GS / MoSca | 稀疏节点刚体运动 + 蒙皮权重 | 同上，且节点之外靠插值 |
| **逐高斯轨迹** | Dynamic3DGS / Spacetime-GS / FreeTimeGS / AST-3DGS | 每高斯存位置/旋转（或关键帧、样条、基函数） | **轨迹只在短窗口内成立** ⇒ 窗口外几何发散 ⇒ 必须靠**不透明度随时间变**去遮 |

第三族是最接近的，也正是 MEMO 要改的那一族。**它的诊断只有一句**：*Polynomial trajectories, in contrast, hold only within short time windows and introduce geometric noise outside them.*

## Core Idea

> **让运动模型在整条时间轴上都成立，于是"不透明度随时间变"这个补丁就不需要了。**

这句话要拆成因果链才看得出分量：

```text
参数化只覆盖短窗口
   ⇒ 窗口外的位置/旋转不可信
   ⇒ 高斯会在某些帧跑到"不该在的地方"
   ⇒ 只能让它在那些帧"不存在"（不透明度 → 0）
   ⇒ 高斯不再是一个持续存在的表面元
   ⇒ 无法携带朝向法线、无法定义场
   ⇒ 只能事后 TSDF 融合
```

MEMO 把第一环换掉：**核覆盖整条时间轴 ⇒ 每一帧都被插值（而不是外推）⇒ 不透明度可以恒定 ⇒ 高斯永远是表面元 ⇒ 表面在任意 $t$ 有定义。**

## Technical Approach

### ① 运动模型：时间高斯核（式 4–7）

每个高斯带两组核（位置、旋转），共 $K=M+2B$ 个：

$$k_{i,m}(t)=\exp\!\left(-\frac{(t-\tau_{i,m})^2}{2\ell_i^2}\right),\qquad
\Delta\mu_i(t)=\sum_m k^{\rm pos}_{i,m}(t)\,w^{\rm pos}_{i,m}$$

$$\Delta q_i(t)=\mathrm{normalize}\!\left(q_{id}+\sum_m k^{\rm rot}_{i,m}(t)\,w^{\rm rot}_{i,m}\right),\qquad q_i(t)=\Delta q_i(t)\otimes q_i$$

三个设计细节各自解决一个问题：

- **inducing time $\tau$ 用 sigmoid 约束到互不重叠的等宽格子**（宽 $1/M$）⇒ 防止所有核挤到运动剧烈处，**保证时间轴被均匀覆盖**；
- **$\ell$ 初始化成格子宽度** ⇒ 相邻核重叠，插值连续（$\ell$ 也可学）；
- **两端各加 $B=2$ 个"界外"核** ⇒ 首尾帧也是**双侧支撑**而不是外推。去掉它，Table 2 里 PSNR 掉 0.99 dB、FS 掉 0.018。

**为什么"局部刚性"够用**：假设尺度不变，只让位置与旋转随时间变。这省掉了"高斯会随时间膨胀/收缩"的自由度——而那个自由度正是几何发散的来源之一。

### ② 初始化：从点轨到闭式轨迹（§4.2）

大而快的运动（抡手臂、旋转的零件）是光度梯度最难处理的情形——梯度只能把高斯拉到**附近**的图像内容上。所以先给一个好起点：

1. 每帧 COLMAP 稀疏点云 → 选一个或几个 anchor 帧 → **Kalman 滤波前后各跑一遍**做关联，用 **Mahalanobis 门**拒绝不合理匹配（被拒的轨跟随邻居运动，而不是被拽走）；
2. 把每个高斯的**位置权重用 GP 回归闭式解**拟合到它的轨（此时 $\tau$ 固定在格心、$\ell$ 固定在格宽，所以是线性最小二乘）；旋转 delta 设为单位四元数；
3. 轨少于 50K 时在轨点周围的壳层里补采样，新高斯继承最近轨点的运动权重。

**为什么能闭式**：轨迹对核权重是**线性**的（式 5），所以"拟合轨迹"就是一次线性回归。这是选"核基"而不是"多项式/样条"的第二个好处——第一个好处是全时段支撑。

### ③ 训练（式 8–10）

$$\mathcal L=\mathcal L_{\rm RGB}+\lambda_D\mathcal L_D+\lambda_N\mathcal L_N$$

$\mathcal L_D$ 是 RaDe-GS 的**深度-法向一致**损失（渲染深度算法向 $N_d$ vs 由高斯形状算的几何法向 $N_g$）；$\mathcal L_N$ 是 Gaussian Wrapping 的**朝向法向对齐**损失（$\alpha$ 加权的 $N_o$ vs 同一个 $N_d$）。密度化用 Mini-Splatting2 的重要性驱动版本"抬到 4D"——比静态 GW 用的多视角损失快，且无纹理区不留洞。

**法向与运动模型的耦合是免费的**：$\Delta q$ 已经算了，法向跟着同一个旋转走，$n_i(t)=R(\Delta q_i(t))\,n_i$，**不增加任何参数**。

### ④ 提取：pivots → Delaunay → vacancy field → marching tetrahedra

推理时把运动模型在 $t$ 处求值，然后照 Gaussian Wrapping 的做法：每高斯生两个 pivot → 对所有 pivot 做 **Delaunay 四面体化** → 在 pivot 上求 vacancy field → 取 **0.5 等值面**（marching tetrahedra）。**没有深度渲染、没有 TSDF 融合。**

## Key Contribution

1. **一套"全时段支撑"的逐高斯运动参数化**，使不透明度可以恒定、高斯可以持续作为表面元。
2. **把场提取从静态搬到完整动态场景**（去掉了 mask 与物体假设）——这是本文与 D-2DGS / DG-Mesh 的位置差。
3. **Blender-Moments**：9 个动画场景 × 50 训练 + 10 测试相机 × 100–170 帧，**逐帧 GT 网格**。此前没有公开数据集把"同步多机位"与"精确时变几何"配在一起。
4. **一组诚实的评测指标**：CD 单独不够（点距几乎不罚表面噪声），所以主指标是**联合 F-score**（位置在包围盒对角线 1% 内 **且** 法向在 20° 内）。

## Why It Works

**因为"一个高斯能不能当表面"这件事，取决于它的参数化覆盖了多长时间。**

Table 2 的消融把这条因果链量了出来（三个 Blender-Moments 场景 + 四个真实场景）：

| 变体 | 合成 PSNR | 合成 FS | 合成 FR | **真实 PSNR** |
|---|---|---|---|---|
| 去掉轨迹拟合 | 26.88 | 0.295 | 0.436 | 28.98 |
| 核数减半 | 32.97 | 0.470 | 0.691 | 29.32 |
| 去掉界外核 | 33.85 | 0.497 | 0.716 | 29.34 |
| **MEMO** | **34.84** | **0.515** | **0.745** | **29.40** |

看**最后一列**：去掉轨迹拟合只掉 **0.42 dB**——因为真实场景有静态背景，PSNR 的分母大部分是背景。而作者写得明白：*the real scenes lose their fast-moving parts*。

> **0.42 dB 的代价，是"运动的部分整个消失"。**

这是本库第三次遇到同一形状（DeepMimic 的 0.730 分不会后空翻、SplatTok 的逐图元 PSNR）。**PSNR 在"目标只占画面一小块"时不是尺子。**

第二个"为什么"在提取层，而且结论反直觉：

| 同一批高斯，两种提取器 | CD ↓ | NC ↓ | FS ↑ |
|---|---|---|---|
| MEMO 原配（marching tets on vacancy field） | 0.51 | **24.6** | **0.556** |
| MEMO + TSDF 融合 | **0.50** | 29.1 | 0.468 |
| MEMO + Poisson | 1.03 | 31.1 | 0.393 |

**CD 上 TSDF 反而好 0.01**（在噪声尺度内），而法向差 **4.5°**、联合 F-score 差 **0.088**。原因是 CD 只罚"点离得远"，几乎不罚"表面在抖"——一个布满 floaters 但大致包住物体的网格，CD 可以很好看。**"换提取器"买到的是法向与拓扑质量，不是位置精度**；位置精度由**表示**（就是那套运动模型）决定——用 TSDF 提取时 CD 仍是 0.50，把所有外观基线（0.81–1.87）甩开一个量级。

## Limitations

1. **逐帧网格之间没有时间一致性**：每帧独立提取，**相邻帧不共享顶点与面**。作者自陈这是 future work——**这意味着它不能直接当动画资产用**，重拓扑与顶点追踪要另做。
2. **依赖初始轨**：细、快、被遮挡的部件若被 tracker 漏掉，就会被抹平或丢失。
3. **模型是同类里最大的**：每个高斯自带 $K$ 组核权重 ⇒ 存储与显存都最高。论文没给 MB 数，但明确写了 *its model is the largest among the compared methods*。
4. **albedo 与 shading 的歧义 + 窄基线**：会留下浮雕状伪影、深度约束弱（这是所有 image-based 几何的共同病）。
5. **18 min/场景是 A100 上**；PennyGS 那一类工作已经把静态训练压到消费级卡 6 分钟，本篇没有往那个方向优化。
6. **Blender-Moments 是渲染数据**：材质简单、无运动模糊、无滚动快门——真实影视级采集的退化没有覆盖。

## Game Development Relevance

**这是"体积视频 → 引擎可用网格"这条路上的一次表示层推进，而不是一次工具链推进。**

1. **表演捕捉的终点是网格，不是高斯**：动作捕捉/体积捕捉最后要进引擎的仍是 mesh（蒙皮、碰撞、LOD、破坏）。MEMO 证明了"逐帧网格"可以从高斯表示里**直接取**，不必先融深度。
2. **自适应分辨率是免费送的**：marching tets 在**高斯所在的位置**采样场，所以网格密度跟着几何走；TSDF/Poisson 在固定网格上解，细化一处就要细化全部（还是**更大的网格**）。这一条对 LOD 与资产体积是实质差别。
3. **"不透明度恒定"这个约束本身就是资产规范**：如果高斯会中途消失，它就不是表面；这条判据可以直接搬到任何"从 splat 提网格"的管线里当检查项。
4. **不能当动画用**（见 Limitations 1）：要拿到可驱动的角色，仍需"逐帧网格 → 顶点追踪 → 重拓扑 → 蒙皮"这条后处理链，成本没有省掉。
5. **18 min/场景 + 50 机位**是离线生产的量级：适合"一次性扫描一个真实场景/表演"，不适合每帧。

## Unreal Engine Relevance

- **不直接对应 UE 模块**：输入是多机位同步视频，输出是逐帧三角网格序列。
- **可迁移的判据两条**：① 若要把高斯资产交给引擎，先问"高斯在时间上是否持续存在"——会消失的不能当碰撞体；② 场提取的自适应分辨率与 Nanite 的 cluster 思想同向（**细节密度跟着几何走，而不是跟着网格走**）。
- **反向的接口**：这个管线产出的逐帧网格可以喂给 Chaos 做破坏/布料预演，或作为 MetaHuman 类角色的几何真值。

## Technology Evolution

```text
2020  NeRF                 密度场 ⇒ 表面（Marching Cubes / 体渲染）
2023  3DGS                 外观与速度；几何是副产物
2024  2DGS / PGSR          压扁成 surfel ⇒ 深度可融合（第一族）
2024  GOF / SuGaR          从高斯定义场 ⇒ marching tets（第二族）
2025  MILo / GW            每高斯带 SDF / 朝向法向 ⇒ 精细表面
      ────────────────── 静态侧基本收敛 ──────────────────
2024  4DGS / Deformable    变形场：运动被共享，几何无独立表示
2024  Dynamic3DGS          每高斯存逐帧位置（存储爆炸）
2025  FreeTimeGS / AST     位置 + **不透明度**随时间变 ⇒ 解决"何时存在"，牺牲"持续是表面"
2026  D-2DGS / DG-Mesh     动态表面，但**要 mask、要单物体**
2026  MEMO                 全时段核 ⇒ 不透明度恒定 ⇒ 场提取搬到完整动态场景
        └ 遗留：逐帧网格无时间一致性；模型最大；依赖点轨
```

**这条线的刻度不是"画质"，是"一个原语在时间上是什么"**：像素级快照（Dynamic3DGS）→ 短窗口轨迹（FreeTimeGS/AST）→ 全时段轨迹（MEMO）。每换一档，可做的事就多一件。

## Relationships

### Based On

- [[Gaussian Splatting]] —— 表征与光栅化全部继承；本文改的只是"运动怎么参数化"。
- [[Isosurface Extraction]] —— 提取侧直接落在那一页的"三个决定"上：**拓扑**来自 pivot 的 Delaunay 对偶、**位置**来自四面体棱上的等值面交点、**密度**来自高斯的分布（第四种答案：密度来自表示本身）。

### Extends

- [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat 2026]] —— 同一目标（从 splat 拿可交付网格）的两条路：T2-Splat 让**拓扑在优化中动**（静态），MEMO 让**几何在时间上动**（动态）。**两者都绕开了"固定网格"这个共同敌人。**

### Related

- [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS|From Splats to Silicon]] —— 那篇给"workload 计数 → stage 时间"的账；MEMO 的账在另一头：**模型最大**（每高斯 $K$ 组核），18 min 训练对应的是"每高斯每帧都要算一次核求和"。
- [[2026-11-26-Pentapati — SplatTok Data-Driven Compression of 3D Gaussian Splats|SplatTok 2026]] —— 存储侧。MEMO 的模型是同类最大，正好是 SplatTok 那类方法的输入。
- [[Level of Detail]] —— 场提取的"密度跟着几何走"是 LOD 的天然形态（$\varepsilon$ 换一次就是一条 LOD 链）。
- [[Mesh Simplification]] —— 下游：逐帧网格要进引擎必须先简化，而"逐帧简化"与"时间一致简化"是两件不同的事（本篇没做前者之外的部分）。
- [[2026-11-24-Nge — PennyGS Full Resolution 3DGS Training in Minutes on Consumer GPUs|PennyGS 2026]] —— 反面参照：静态侧已经在压训练时间，动态侧还停在 18 min/场景。

### Contrasts

- [[2026-11-26-Peng — MESSI Multi-Expert Soccer Skill Integration|MESSI 2026]] —— 都在处理"时间上的连续性"，但对象相反：MESSI 补的是**控制策略在技能之间的缝**，MEMO 补的是**表示在帧之间的缝**。两者都发现"缝不在训练分布里"。

## Personal Knowledge State

- **Easy**：3DGS 的高斯参数与光栅化；marching cubes / tets 的基本做法；"TSDF 融合 + 固定网格"为什么会有分辨率问题。
- **Normal**：**"参数化的时间支撑 = 补丁的数量"**这条因果链（短窗口 ⇒ 不透明度开关 ⇒ 失去表面元资格）；核基为什么能闭式拟合（轨迹对权重线性）；CD 为什么不罚噪声、联合 F-score 为什么必须加"位置 **且** 法向"。
- **Hard**：GP 回归在"核中心与宽度也可学"时的可辨识性（论文直接说"relax both"然后一起学，没有讨论退化）；四元数核加和的归一化对旋转插值路径的影响。

## Learning Value

**三个可迁移动作：**

1. **看到"某个量随时间变化"时，先问它是物理需要还是参数化的补丁。** 这里不透明度随时间变不是物理（材料不会时有时无），是**短窗口轨迹**逼出来的遮羞布。**同一个问题在动画压缩、LOD 切换、缓存失效里都出现过。**
2. **消融的"某一列几乎不变"要追问那一列的分母是什么。** 28.98 vs 29.40 的 0.42 dB 背后是"运动部分消失"——**分母是静态背景**。
3. **两个指标打架时，先看它们罚的是什么。** CD 罚位置、不罚抖动；NC 罚朝向。**CD 好而 NC 差 = 网格大致包住了物体但在抖**，这正是 floaters 的签名。

## Visualization

![[Research/Files/Pictures/Or-El 2026 — 图1 每一帧都有一张网格.png]]

**读图**：三行（大象、鳄鱼、甲虫车）沿时间轴铺开，**首尾两帧是顶点色、中间帧是灰模**——这是本文的主张本身：不是"某一帧的重建"，是"每一帧都有一张完整网格"。注意三行的运动幅度：大象是缓慢转身、鳄鱼是爬行、甲虫车是**开门 + 轮子转动**（局部快速旋转，正是短窗口轨迹最容易崩的那一类）。所有网格都是从同一批高斯里"取"出来的，没有一次深度融合。（图 1，Or-El et al. 2026）

![[Research/Files/Pictures/Or-El 2026 — 图2 运动模型与提取管线.png]]

**读图**：上半是训练回路——Canonical Gaussians（带朝向法线，就是那些小箭头）→ 运动模型在 $t$ 处求值 → Gaussians at time $t$ → 三个损失（$\mathcal L_{\rm RGB}$ 对渲染图、$\mathcal L_D$ 对 $N_g$ vs $N_d$、$\mathcal L_N$ 对 $N_o$ vs $N_d$）。下半是推理回路——同一批高斯 → 四面体化 → 求 vacancy field（**红=占据、蓝=空**）→ marching tets 出网格。**上下两半共用同一套参数**，这就是"网格不是事后融的"的图示。（图 2，Or-El et al. 2026）

![[Research/Files/Pictures/Or-El 2026 — 图3 覆盖全时段的核与短窗口核.png]]

**读图**：单个高斯的一个位移分量。横轴是归一化时间，**注意它画到了 $-0.2$ 与 $1.2$**——超出 $[0,1]$ 的那两段就是**界外核**（红色三角，$B=2$），它们的唯一作用是让首尾帧也有双侧支撑。蓝圈是内部核（等宽格子、$\ell$ 初始化成格宽 ⇒ 相邻重叠）。粗蓝线是所有核的加权和，**在整条轴上都是插值**，没有一段是外推。绿叉是 $t=0.5$ 处的预测值——它同时被左右两侧若干核支撑。**"全时段支撑"这句话的全部内容就是这张图。**（图 3，Or-El et al. 2026）

![[Research/Files/Pictures/Or-El 2026 — 图5 与基线的网格对比.png]]

**读图**：前三行是有 GT 的 Blender-Moments（Lego / Elephant / Toyota），后两行是真实场景（Flame salmon / Yoga）。看三处：① **Deformable-3DGS 那一列布满灰色小点**——那就是 floaters，TSDF 融合被它们污染；② **DG-Mesh 明显过平滑**（Lego 的凸点、Toyota 的引擎细节都没了），因为 Poisson 重建偏好光滑；③ **D-2DGS 在 Yoga 那行直接失败**（大片破洞），它的深度正则化在真实场景上不成立。**MEMO 每行都最接近 GT**，包括 Lego 的凸点与 Toyota 的进气格栅。（图 5，Or-El et al. 2026）

## Notes

原文 PDF 已归档：`Research/Files/PDF/Or-El 2026 - MEMO A Mesh for Every Moment.pdf`（42.7 MB，27 页）。

- **"每帧独立提取 ⇒ 无时间一致性"是本页最重要的一条边界**：它决定了这份产出是"几何真值"而不是"动画资产"。写进任何管线评估时都要标出来。
- **Blender-Moments 的价值可能大于方法本身**：在此之前，"同步多机位 + 逐帧 GT 网格"没有公开组合，所以"动态表面重建"这个任务一直缺主指标。**数据集论文常常是判据论文。**
- 作者 **Roy Or-El**（StyleGAN 系人脸编辑）+ **Jeong Joon Park**（NeRF 系）+ **Ira Kemelmacher-Shlizerman**（UW，3D 人脸/人体）——这是"生成式人脸"转向"体积捕捉"的一次组合。
- 与 [[2026-11-26-Peng — MESSI Multi-Expert Soccer Skill Integration|MESSI]] 的一个共同点值得记：**两者都在"训练分布没覆盖的那一小段"上失败**（MESSI 是技能接缝，MEMO 是 tracker 漏掉的细快部件）。

**相关每日记录**：[[2026-11-27]]
