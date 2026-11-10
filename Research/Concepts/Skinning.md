---
type: concept
user_level: Normal
aliases: ["Skinning", "蒙皮", "Linear Blend Skinning", "LBS", "Dual Quaternion Skinning", "DQS", "骨骼蒙皮", "Skeletal Skinning"]
prerequisites: ["Rigid Transformation", "[[Motion Retargeting]]"]
first_introduced: "Magnenat-Thalmann, Laperrière & Thalmann, Graphics Interface 1988（joint-dependent local deformations）"
---

# Skinning

## Definition

**把骨架的运动"传"到皮肤表面的过程。** 输入：一张静止姿态的网格、一副骨架、每个顶点对若干骨骼的**权重**；输出：每个顶点的形变位置。

它是**角色管线的中间层**：上游是 [[Motion Retargeting]]（把动画搬到这副骨架上），下游是渲染（顶点位置 → 光栅化）。**蒙皮不产生运动，也不产生光照——它只负责"骨动，皮跟着动，且动得像"**。

## Core Principle

所有实时蒙皮都是同一个式子的变体：

$$v'=\sum_{i=1}^{n} w_i\, T_i\, v,\qquad \sum_i w_i=1$$

差别只在**"$T_i$ 怎么混"**：

| 混法 | 混的对象 | 结果是否一定是刚体 | 代价 | 代表 |
|---|---|---|---|---|
| **LBS**（线性混合） | 矩阵的 9 个元素各自插值 | ❌ 会长出缩放/剪切 | 最低（36 条指令） | 1988 至今的默认 |
| **DQS / DLB** | 对偶四元数（8 分量） | ✅ 归一化即回 SE(3) | +7 条指令 | [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan 2008]] |
| **PSD / corrective** | 不改混法，改"每个姿态该长什么样" | —（在 LBS 之上叠修正） | 内存 ↑（每姿态一组 delta） | Lewis 2000 |
| **学习型**（神经蒙皮） | 用网络直接回归形变 | —（不保证刚体，但保证像） | 训练成本 + 推理 12–14 ms | [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]] |
| **物理/肌肉层** | 解纤维收缩与体积 | —（这是仿真，不是插值） | 18 ms/帧（单角色） | [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning|RLMuscle]] |

**四代的分界线，是"承认什么问题"**：

```text
LBS      承认"要快"，不承认"混矩阵会错"      → 糖果纸 / 塌肩（美术修）
DQS      承认"混矩阵会错"                   → 数学修，零美术成本
PSD      承认"线性权重表达不了真实形变"        → 数据修，要美术做 pose
神经蒙皮  承认"PSD 的 pose 空间也表达不了"      → 学习修，要数据与训练
肌肉层    承认"蒙皮根本不该产生形变，力才该"    → 仿真修，要算力
```

### 权重从哪来：热平衡（2026-11-05 补）

上面整页都在讲"**给定** $w_i$ 怎么混"。但 $w_i$ 从哪来？本页一直把它写成"绑定师（自动或手绘）"，等于把一个完整的工序压成了两个字。补上这一层的是 [[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran & Popović 2007（Pinocchio）]]，而那套做法**至今仍是 DCC 里"自动权重 / bone heat"那一档的原型**。

**它要满足三个性质**（这三条决定了不能用"最近距离"）：

1. 与网格分辨率无关；
2. 沿表面平滑变化；
3. **两根骨之间过渡带的宽度，大致正比于"关节到表面的距离"**。

**为什么距离法不行**：按"顶点到骨的最近距离"分配，躯干的一部分会被算给手臂——**因为距离是欧氏的，而身体是连通的**。腋下到手臂的距离比到脊柱近，但它该跟脊柱。

**改成热平衡**：把角色当绝热导体，第 $i$ 根骨温度强制为 1、其余为 0，平衡时表面顶点的温度就是这根骨的权重：

$$-\Delta w^i+Hw^i=Hp^i$$

- $\Delta$：离散**曲面** Laplacian（cotangent，Meyer et al. 2003）；
- $p^i_j=1$ 表示"顶点 $j$ 的最近骨是 $i$"；
- $H_{jj}=c/d(j)^2$，**且只在"顶点到骨的最短线段整段在体内"时非零**，否则 $H_{jj}=0$。

三个工程细节值得单记：

| 细节 | 做法 | 为什么 |
|---|---|---|
| **扩散只在表面解** | 体积要四面体化，太慢；用"线段是否在体内"这个可见性测试把体积信息压成 $H$ 的一个对角元 | 用**便宜的判据**替代**昂贵的求解** |
| **$c$ 的取值** | $c\approx0.22$ 才接近体积解，但他们用 $c=1$ | 原文：**"because the results look more natural"**——**方程给形状，常数给手感** |
| **归一化** | 不需要额外步骤 | 对方程按 $i$ 求和即得 $\sum_i w^i=\mathbf 1$：**约束被编码进方程** |

**这解释了 DCC 里那句经典报错**（按方程推，原文未这么说）：若某个连通分量上所有顶点都满足"到最近骨的线段不在体内"，该块 $H=0$，方程退化成纯 Laplacian——**纯 Laplacian 在常向量上有零空间，系统奇异**。这就是 `Bone Heat Weighting: failed to find solution for one or more bones` 的数学来源：**不是权重算错了，是那一块没有热源**。修法只有三个：**补一根骨 / 修穿插让线段回到体内 / 把薄片加厚**。调参数是无效动作。

**给本页加一条判据**：看到"某块表面被远处拽走 / 腋下粘连"，那是**权重**问题（来源）；看到"弯折发生在肢体的错误位置"，那是**骨架**问题（去 [[Automatic Rigging]]），不归蒙皮管。

## Prerequisites

- **刚体变换**（旋转 + 平移）与它的几种表示：矩阵 / 四元数 / 对偶四元数——见 [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan 2008]] 的图 2（"平均点 vs 平均角"是最直观的入口）；
- **四元数与球面插值**：库内 [[2026-10-15-LinfCub — Compression of Uniformly Distributed Unit Quaternions|LinfCub]] 讲的是"怎么存得小"，本篇讲的是"怎么混得对"，**同一族数学的两个用途**；
- **线性代数中的"封闭性"**：正交矩阵集合对加法不封闭——这是全部问题的起点。

## Historical Evolution

```text
1988  Magnenat-Thalmann et al.：joint-dependent local deformation（LBS 的工程起点，为手部抓取而做）
2000  Lewis et al.：Pose Space Deformation（承认 LBS 不足，用姿态 → 修正形状补）
2003  Mohr & Gleicher：从样例中学权重（example-based skins）
2004  Hejl：直接四元数混合（有绑定约束）；Govindu：SE(3) 最优混合（迭代，太慢）
2005  Kavan & Žára：SBS（球面 + SVD + 聚类）；Cordier & Magnenat-Thalmann：log-matrix
★2008 Kavan et al.：DLB（对偶四元数线性混合 + 归一化）——"+7 条指令"的正确性
2010s 引擎把它做成蒙皮模式开关；corrective 库成为 3A 标配
2020s 学习路线：ML Deformer / 神经蒙皮（PhysSkin++ 2026：12–14 ms，与网格规模解耦）
2026  RLMuscle：把"肌肉收缩"从手写激活改成 RL 反解 + 体积仿真（18 ms/帧，194 肌肉）
2026  3DGS 时代：[[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation|GradRig]] 把蒙皮搬到高斯基元上（可微权重）
2026  [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|Dynamic Skinning]]：**分出一条新轴**——不换混法，在混完之后加一个"有时间维"的位移项（VS 2021 → AS 2022 → DS 2026 三级）
2026  [[2026-11-10-Zhu — Real-Time Self-Penetration Repair for Skinned Characters|Zhu 2026]]：**承认"蒙皮不管碰撞"**——修复放到蒙皮之外：改姿态（<1.11 ms）+ 压皮肤（1.18 ms），中间一条静态汇率
```

**这四十年没有"取代"，只有"分层"**：LBS 仍是默认，DQS 是质量开关，PSD 是修形，神经蒙皮是快而像，肌肉层是慢而真。**它们各自守住一个成本档位**。

### 第二条轴：不改混法，只加一项（2026-11-03 补）

上面四代回答的都是同一个问题：**"$T_i$ 该怎么混"**——混得对不对（DQS）、混得够不够（PSD/神经）、还是干脆别混（仿真）。2026 年的 [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|Dynamic Skinning]] 把它拆开了：

$$p_i^{DS}=p_i^{LBS}+\sum_j\beta_{ij}\,\psi\big(\mathcal F(\Omega_j,t),q_j,p_i^{LBS}\big)$$

**混法一个字没改，只是在混完之后加了一个位移项。** 这不是"第五代蒙皮"，是**另一条轴**：

| | 混法轴（LBS→DQS→PSD→神经） | 加项轴（VS 2021 → AS 2022 → DS 2026） |
|---|---|---|
| 问的问题 | "这个顶点该被哪几根骨、按什么比例带动" | "带完之后，它还要不要多走一点" |
| 数学性质 | **插值**：要保证结果仍在 SE(3) 里（或至少别长剪切） | **信号处理**：输入是运动的导数，输出是位移 |
| 有没有时间维 | ❌ 没有（只看当前姿态） | ✅ 有（DS 的 $\mathcal F$ 看过去 $W$ 秒） |
| 能不能叠 | 只能选一种混法 | **可以叠在任何混法上**（LBS/DQS 都行） |
| 典型失效 | 糖纸、塌肩、翻面 | 穿模（没有碰撞）、脚离地（dLBS 延迟了整条变换链） |

**这条轴的存在解释了一件旧事**：本页"Game Applications"里那句"二次元/卡通角色常用缩放关节做挤压拉伸——这正是 DQS 表示不了的那一类"。**它之所以表示不了，是因为它属于加项轴，不属于混法轴。** 用混法去实现挤压拉伸，等于用插值器做滤波。

代价也要写清楚：**加项永远不产生"真正的"形变**——没有体积守恒、没有碰撞、没有动量。它是"看起来有质量"，不是"有质量"。**判据：要不要和世界交互**（要，就回 [[Cloth Simulation]] 或弹簧骨；不要，加项最便宜）。

### 第三个失效：穿模不是形变问题，是约束缺失（2026-11-10 补）

上面讲的四条混法 + 一条加项轴，全部只回答"顶点该被带到哪里"。**没有一条回答"两个部位能不能相交"。** 这不是某一代的缺陷，是分工的结果：

$$v'=\sum_i w_iT_iv\ \ \text{是姿态的纯函数}$$

纯函数的好处是它便宜、可并行、能进 GPU skin cache、能每帧跑；**坏处是它没有"世界状态"这个概念**——手掌在哪、躯干在哪，两者是否重叠，蒙皮根本不知道。

于是穿模在工程上一直是**三件手工活**：美术刷权重避开、动画师避开角度、TA 摆碰撞体。这三件事的共同前提是"动作是有限的、可枚举的"——**生成式动画（motion matching 变体、motion diffusion、LLM 驱动 NPC）把这个前提拆掉了：你没法为每一个生成出来的姿态提前摆好碰撞体。**

**判据（把三种"看起来像蒙皮问题"的东西分开）**：

| 现象 | 真正的层 | 去哪修 |
|---|---|---|
| 关节处"糖纸"、塌肩、关节处体积塌陷 | **混法**（LBS 混矩阵） | DQS / corrective / 神经蒙皮 |
| 弯折发生在错误的肢体位置 | **骨架** | [[Automatic Rigging]] |
| **两个部位互相插进去** | **约束**（蒙皮里根本没有这一层） | **蒙皮之外的后处理或求解** |
| 披风穿进大腿 | **碰撞体**（不是蒙皮） | [[Cloth Simulation]] |

**为什么不能"在蒙皮里加碰撞"**：一旦混法里出现"看别的顶点在哪"的项，蒙皮就变成了**有状态的求解**——不能再按顶点并行、不能再假设"输入姿态 → 输出顶点"是无记忆的，GPU skin cache 的整个前提消失。**蒙皮之所以能每帧跑，正是因为它什么都不管。**

**所以修法只能加在旁边**，而且必须回答一个接口问题：**改姿态，还是改顶点？**

[[2026-11-10-Zhu — Real-Time Self-Penetration Repair for Skinned Characters|Zhu 2026]] 给的答案是**两个都改，按深度分工**：

- **深冲突（>20 mm）改姿态**——几何真的错了，只能用关节自由度去躲；21 个非根关节、每帧一个盒约束 QP，单核 **1.11 ms**。
- **浅重叠（<20 mm）改顶点**——几何是对的，只是被压了；无状态接触形变 + 折叠控制，**1.18 ms**，6890 顶点。
- 两层之间**不传余额，只共享一个常数** $\rho$（软厚度比例，全局 0.5 / 手查询 0.1）。

**最值得记住的一条**：这套东西能在 2.29 ms 里跑完，靠的不是算法快，是**它保住了蒙皮的无状态性**——Layer 2 明确"没有上一帧"，所以可重跑、可并行、不漂移；所有需要历史的东西（release、刹车、urgency）都被关进 Layer 1 的残差里。

**代价也在这里**：上臂-躯干这一区姿态层几乎不动（39.79% → 37.95%），全靠皮肤压缩（→ 0.11%），最终仍有 10.24% 的帧 >10 mm。**"用形状换接触"是这套分工的固定账单，不是实现不好。**


## Important Papers

- [[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran & Popović 2007]] — **权重的来源**：热平衡（表面 Laplacian + 体内可见性门控）取代距离；见本页"权重从哪来"
- [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan et al. 2008]] — **本页主节点**：把"混矩阵会错"讲成几何论证，并给出 +7 条指令的修法
- [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]] — 神经蒙皮：前馈蒙皮场，免逐对象训练
- [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning|RLMuscle]] — 肌肉层：蒙皮之上再解一层"力"
- [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation|GradRig]] — 蒙皮权重的可微化（GS 表示）
- [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|Dynamic Skinning 2026]] — **第二条轴的开端**：不改混法，在 LBS 之上加一个由关节速度驱动的 FIR 滤波位移项（0.00003 ms/面片，1.5 M 面片模型；见本页"第二条轴"）
- [[2026-11-10-Zhu — Real-Time Self-Penetration Repair for Skinned Characters|Zhu 2026]] — **蒙皮之外的那一层**：穿模不是形变错，是约束缺；两层（改姿态 1.11 ms + 压皮肤 1.18 ms）+ 一条静态汇率 $\rho$；见本页"第三个失效"
- （库外必读）Lewis et al. 2000《Pose Space Deformation》；Magnenat-Thalmann et al. 1988（LBS 源头，PDF 已不可得，本条依二手文献核证）

## Related Concepts

- [[Motion Retargeting]]（**上游**：把动画搬到这副骨架；蒙皮是"搬完之后"的形变）
- [[Physics-based Character Animation]]（**下游**：蒙皮给的是"运动学形变"，物理给的是"动力学形变"）
- [[Neural Animation]]（学习路线的总纲）
- [[Subdivision Surfaces]]（**相邻问题**：一个是"网格怎么变细"，一个是"网格怎么跟着骨动"；两者都要在"规则"与"数据"之间选）
- [[Mesh Parameterization]]（**同构问题**：蒙皮把"运动"映射到皮肤，参数化把"曲面"映射到平面；两者都要在"混/解之后，那个好性质还在不在"——蒙皮要"仍是刚体"，参数化要"仍是单射"。**2026-10-24 交叉点**：两者都在 10 月中被"换表示让性质自动成立"这条思路各推了一步）
- [[Cloth Simulation]]（**必须分清的一对**：蒙皮是"网格按骨骼权重变形"——零自由度、纯几何、每帧算完就结束；布料是"每个顶点独立自由度 + 碰撞约束"——要解一个系统、要迭代、帧间有状态。**工程上最实用的一条判据：衣服穿模看起来像蒙皮问题，八成是碰撞体问题**（披风穿进大腿是胶囊体和布料解算的接触失效，不是权重没收好）；反过来，关节处的"糖纸"和插值塌陷才是真蒙皮问题。**2026-10-28 开域**：两页在同一天入库，正好把"哪些形变不需要解算"这件事划清）
- [[Secondary Motion]]（**相邻但不同层**：蒙皮是"骨动 → 皮动"的映射（零时间维），二次运动是"骨的导数 → 皮的附加位移"（有时间维）。**DS 式实现里两者是相加的两项，不是替代关系**；"骨停了皮还在动"这件事归二次运动管，不归蒙皮管）
- [[Automatic Rigging]]（**上游**：本页把权重当输入，那一页讲权重和骨架怎么来。**时间尺度不同——蒙皮每帧跑，绑骨只在资产入库时跑一次**，这就是它们永远不会合成一个话题的原因）

## Technologies

- **GPU skin cache**（每帧把骨骼变换写进常量/结构化缓冲，顶点着色器取 4 个权重）——LBS 与 DQS 的工程载体；
- **Corrective shape / PSD**（姿态空间修正库）——与 DQS 可叠加，但**corrective 通常在 LBS 空间标定**，换算子要重烘；
- **ML Deformer**（UE）：把"姿态 → 形变"学成网络，是 PhysSkin++ 一类的工程容器；
- **Chaos Flesh / 四面体软组织**：肌肉层的引擎侧形态。

## Game Applications

- **主角/特写**：DQS 或神经蒙皮（质量优先）；
- **群众/远景**：LBS（成本优先，关节大角度扭转时才可见差异）；
- **受击/施法的肌肉抽动**：corrective 库（离线）或单角色特写（RLMuscle 那一档）；
- **二次元/卡通角色**：常用"缩放关节"做挤压拉伸——**这正是 DQS 表示不了的那一类**，要走两阶段或干脆留在 LBS。

## Personal Knowledge

Current Level: **Normal**（推断）

- **Easy**：LBS 的糖果纸与塌肩——你在角色/材质工作上早已见过，且知道美术用 corrective 糊；
- **Normal**：为什么"混矩阵"必然产生缩放、DQS 的归一化为什么够用、两阶段缩放、翻面伪影、四代各自的成本档位；
- **Hard**：对偶四元数代数与 DLB/ScLERP 误差界——**除非要自己实现，不必碰**。

## Learning Gap

**缺的不是"知道有 DQS"，是"什么时候值得换"的判据。** 库内目前只有 Kavan 一篇提供成本数字（+7 条指令 / 1.4× CPU）；缺的是：

1. **引擎实测**：UE 里 LBS vs 自定义 DQS 的真实开销（含 skin cache、morph target、GPU 蒙皮路径）；
2. **与 correctives 的叠加代价**：换 DQS 后原有 corrective 库要不要重烘、重烘多少；
3. **"什么时候看不出差别"**：关节转角阈值、网格密度、镜头距离——**这才是决定要不要开的实务判据**。

## Next Step

**按需，不急。** 若要动手，最小闭环是：

1. 拿一个**肩/肘大角度扭转**的角色动作（扭转 ≥120°），截图记录 LBS 的塌陷位置；
2. 在引擎里开 DQS（或等价的自定义蒙皮），同镜头对比；
3. 只回答一个问题：**"这个差异在我的镜头距离下可见吗？"**——可见就值得开，不可见就留在 LBS；
4. 若已开 DQS，再检查 **corrective 库是否还匹配**（这是最容易踩的坑）。

**读法建议**：[[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan 2008]] 结论层 ≈25 分钟（TL;DR + 图 2/4 + 成本表），不必读附录推导。
