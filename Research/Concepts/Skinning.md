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
```

**这四十年没有"取代"，只有"分层"**：LBS 仍是默认，DQS 是质量开关，PSD 是修形，神经蒙皮是快而像，肌肉层是慢而真。**它们各自守住一个成本档位**。

## Important Papers

- [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan et al. 2008]] — **本页主节点**：把"混矩阵会错"讲成几何论证，并给出 +7 条指令的修法
- [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]] — 神经蒙皮：前馈蒙皮场，免逐对象训练
- [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning|RLMuscle]] — 肌肉层：蒙皮之上再解一层"力"
- [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation|GradRig]] — 蒙皮权重的可微化（GS 表示）
- （库外必读）Lewis et al. 2000《Pose Space Deformation》；Magnenat-Thalmann et al. 1988（LBS 源头，PDF 已不可得，本条依二手文献核证）

## Related Concepts

- [[Motion Retargeting]]（**上游**：把动画搬到这副骨架；蒙皮是"搬完之后"的形变）
- [[Physics-based Character Animation]]（**下游**：蒙皮给的是"运动学形变"，物理给的是"动力学形变"）
- [[Neural Animation]]（学习路线的总纲）
- [[Subdivision Surfaces]]（**相邻问题**：一个是"网格怎么变细"，一个是"网格怎么跟着骨动"；两者都要在"规则"与"数据"之间选）
- [[Mesh Parameterization]]（**同构问题**：蒙皮把"运动"映射到皮肤，参数化把"曲面"映射到平面；两者都要在"混/解之后，那个好性质还在不在"——蒙皮要"仍是刚体"，参数化要"仍是单射"。**2026-10-24 交叉点**：两者都在 10 月中被"换表示让性质自动成立"这条思路各推了一步）

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
