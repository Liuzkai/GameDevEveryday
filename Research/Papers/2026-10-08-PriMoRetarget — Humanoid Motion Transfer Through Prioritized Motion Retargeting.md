---
type: paper
title: "PriMoRetarget: Humanoid Motion Transfer Through Prioritized Motion Retargeting"
authors: [Nicholas Ioannidis, Dun Yang, Max Smith, Tue Vu, Ziyu Zhang, Shayegan Omidshafiei, Xue Bin Peng]
year: 2026
published: "2026-10-08（arXiv v1, 2610.12642；周一公告组 2026-10-12）"
venue: "arXiv Preprint（Simon Fraser University × FieldAI × NVIDIA × Amii；cs.RO + cs.GR）"
url: "https://arxiv.org/abs/2610.12642"
code: ""
project_page: "https://primoretarget-page.github.io/（primoretarget-page；代码与数据承诺发表时放出）"
category: [motion-retargeting, prioritized-optimization, humanoid, character-animation]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Research（4 数据集 + 真机部署演示：Unitree G1/H1-2、DEEP Robotics DR02、Booster T1）"
user_level: "Normal（结论层 15 分钟：'优先级约束取代权重妥协' + 空空间投影直觉）"
status: unread
aliases: [PriMoRetarget, 优先级动作重定向, 优先级约束重定向]
tags: [motion-retargeting, animation, optimization, humanoid]
---

# PriMoRetarget: Humanoid Motion Transfer Through Prioritized Motion Retargeting（Ioannidis et al. 2026）

> **入库 2026-10-13（Run 35）。** 周一公告组（cs.GR cross-list）捕获；作者含 **Xue Bin Peng**（SFU——物理角色动画领域的关键作者），是"重定向（retargeting）"库内线索的第三个节点。
> **一句话定位**：**用"优先级层级"取代"加权求和"的动效重定向**——传统做法把所有目标（姿势匹配、脚部着地、接触保持、关节限位）**加权求和**，权重调不出来"顺序"：你提高某个权重，它在冲突时仍可能被牺牲；本篇把目标排成**显式优先级**，**低优先级更新被限制在"不破坏高优先级目标"的方向上**（空空间投影），关键特征（脚部位置、交互接触）就有了"硬保护"。
> **库内位置**：重定向线索第三节点（[[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement|Skinned Retargeting]] → [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]] → 本篇）；也是"**约束 vs 权重**"设计模式的又一实例——**重要的东西不该靠调参去保证，应该靠结构去保证**。

## TL;DR

**问题**：动作重定向 = 把"人（源）"的动作搬到"机器人/不同比例的角色（目标）"上；单角色动作已有成熟做法，但**多角色互动 / 角色-物体交互**（OMOMO 提箱子、双人协作）引入大量**接触与空间关系**——重定向必须在"保持接触"与"匹配姿势"之间权衡。业界惯例 = **加权多目标优化**，于是：

```text
权重只表达"相对偏好"，不表达"先后顺序"：
  w_contact 调高 10× —— 冲突时接触仍可能被牺牲（只是"更贵"了）
  每换一套动作 / 一种目标体型 → 重新调权重（原文：careful manual tuning）
```

**做法**：**显式优先级层级 + 局部空空间投影**：

```text
目标排序（高 → 低）示范：
  ① 脚部匹配（foot placement）
  ② 交互接触保持（contacts）
  ③ 姿势匹配（body pose，低优先级）
  （平滑性等"自由目标"不进层级——投影会压死它们，单列）

机制（每步优化）：
  高优先级误差 f_H(x) 线性化：f_H(x+δx) ≈ f_H(x) + J_H·δx
  低优先级更新 δx 限制在 J_H 的空空间里（J_H·δx = 0）
  ——"要改，可以；但不许碰高优先级已经把住的东西"
  关键细节：约束的是【误差向量】而不是【标量损失】——
  "误差分量之间不许互相补偿"（防止一个脚变好、另一只脚变差的"总分不变"）
```

**工程关键**：长序列 + 多角色 + 非二次目标 → **逐帧（batched per-frame）空空间投影**，把密矩阵解从"随序列长三次增长"压成"几乎常数"（RTX 5090；LAFAN1 5000+ 帧仍稳定）。

**结果（4 数据集）**：LAFAN1（单角色）/ OMOMO（角色-物体）/ Inter-X（角色-角色）/ 自建（多角色+物体）——接触保持全场景最优、脚部滑动普遍降低、抖动低于逐帧优化基线；**重定向后的动作作为 RL 参考轨迹**：下游策略成功率最高、身体跟踪误差最低；真机部署（Unitree G1/H1-2、DEEP Robotics DR02、Booster T1）演示双机协作搬运、坡道搬运、**物体放大 1.5× 后手部接触自动适配**（"高优先级目标管住接触，而不是死跟源动作"）。

## Problem

- **重定向是"动作数据进游戏/机器人前的必经预处理"**——把源骨架动作搬到目标体型上；错的参考动作会把误差放大进下游（RL 或引擎里）；
- **加权求和的三个病**：
  1. **顺序不可表达**：权重只给"汇率"，不给"优先权"——高优先目标仍可被"总损失下降"牺牲；
  2. **调参负担**：换动作/换体型 → 重新调权重（论文原文反复强调 manual tuning）；
  3. **多目标冲突随交互升级**：多角色/物体多接触点，目标冲突指数上升；
- **已有替代的局限**：gradient-surgery 类（PCGrad 等）减少梯度干扰但**不建立顺序**；数据驱动重定向受训练分布限制。

## Core Idea

**"优先级 = 约束，不是大权重"**——把"等级"从**软偏好**（权重）变成**硬保护**（可行性集合）：

```text
权重版：min Σ w_k L_k       → 解空间里"任何方向都可以走，按汇率付钱"
优先版：x_k* ∈ argmin_{x ∈ S^{k-1}} L_k(x)   （S^{k} 逐步收紧）
        ——第 k 层只能在"前 k-1 层的最优解集里"再找自己的最优
        局部实现：低优先级梯度投影到高优先级雅可比的空空间
```

库内视角：这是"**约束三档处置**"（拒绝/修正/有条件接受，来自 [[Procedural Content Generation]] 的 CGA 抽象）在动画优化里的同构样本——**"重要的东西用硬约束保，次要的东西用软偏好让"**；也与 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]] 的"用户提示只在关键处补"同族：**把'必须对'与'尽量好'分开表达**。

## Technical Approach（要点）

- **层级定义**：L¹…L^K 从高到低；示例子集：脚部匹配 → 接触/物体交互 → 身体姿势；
- **空空间投影**：f_H = 高优先误差向量；J_H·δx = 0 的解空间即"不破坏高优先目标的一阶方向"；低优先梯度 g_L 投影进去；
- **保留误差向量（非标量）**：原文强调"Preserving the error vector, rather than only its scalar loss, prevents compensation across error components"——**不许"作弊式补偿"**（左脚本好了右脚本坏了、总误差持平）；
- **加权预热（warmup）**：初始化离解太远时，纯优先优化会困在起点附近——先用加权求和把初始解"挪进合适区域"，再切优先优化（消融：去掉 warmup 在 LAFAN1/自建集上明显退步）；
- **自由目标单列**：时间平滑等"自由目标"放进层级会被投影"压死"（抖动上升）——单列在层级外（消融：Free-last 变体抖动增大）；
- **逐帧批处理投影**：长序列的工程解法（Fig 5：密解三次增长 vs 逐帧近常数）。
- **评估**：4 数据集 + 5 指标（穿透/滑动/浮空/抖动/接触保持）+ 下游 RL 跟踪成功率；对照基线含 PHC 等成熟重定向器。

## Key Contribution

1. **优先级层级重定向框架**：加权求和 → 显式层级（可表达"谁先谁后"）；与常用重定向目标（IK 类）兼容；
2. **向量误差的空空间保持**：防"分量间补偿"——**高优先目标的每个分量都被保护**；
3. **工程可行性**：逐帧投影使优先优化可用于**长序列、多角色、非二次目标**（首次把优先优化的适用面从在线控制/MPC 扩到离线重定向）；
4. **下游验证闭环**：不只是"动作看着好"，而且"作为 RL 参考更好用"（成功率/跟踪误差）——**参考质量用下游任务度量**。

## Why It Works

- **"顺序"是冲突解决中最强的信号**：层级把"冲突时谁让路"从数值巧合（权重比）变成结构性保证（可行集合收缩）；
- **局部空空间投影**：把全局递归定义（Eq. 2/3）局部化到每步更新——**"每一步都不越界" ⇒ 整条轨迹不越界**（一阶）；
- **warmup 与自由目标的分离**：承认"层级不是万能的"——**优化器需要好起点、自由目标需要出口**（两个工程补丁方向正确）。

## Limitations

- 高优先目标"锁死"后，低优先目标空间变小——低优先质量可能不如加权版（论文 fig 4 明示）；"锁什么"是设计决定，不是自动的；
- 高优先级目标本身的建模（哪些算接触、容差多少）仍需人为设定；
- 空空间投影是**局部一阶**近似——大步长/强非线性下需信赖域控制（论文用常规做法，未细述）；
- 面向人形（humanoid）设定；其它拓扑（四足等）泛化未验证。

## Game Development Relevance

- **重定向是游戏动画管线的日常活**：动捕 → 不同体型角色（玩家自定义体型、NPC 比例差异）——"权重调半天、脚还在滑"是常见痛点；
- **可迁移判据**：*"我的多目标系统里，'顺序'是隐含在权重里，还是显式写出来的？"*——凡是"这个更重要"的需求，先问它能不能升级为**约束/层级**（重定向、IK、相机、AI 行为权重皆同构）；
- **"向量误差保护"对游戏侧的意义**：防止"总体达标、局部崩坏"（如双接触点里丢一个）；
- **下游度量**：RL 参考质量/跟踪成功率——"动作数据好不好，让下游说话"（对动画数据管线验收有直接参考价值）。

## Unreal Engine Relevance

- 概念层直接映射 **UE IK Rig / IK Retargeter**：目前引擎侧重定向以链式 IK + 逐链求解为主（本质也是"分层"思路的简化版）——本篇给出"**多目标 + 显式层级 + 空空间保持**"的一般化形态；对复杂交互动画（双人搬运、开门搬箱）的批量重定向流程有方法参照。

## Technology Evolution

```text
动作重定向谱系（库内三节点）：
  早期/工业标准：IK 式逐链求解 + 加权多目标（权重手工调）
  数据驱动线：[[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement|Skinned Retargeting]]（伪影驱动运动学先验精修）
  生成式线：[[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]（语义感知精修流；"参考解当初始化还是目标"）
  ★ 本篇：优化式线——优先级层级 + 空空间投影（顺序显式化）
  下游：RL 策略训练（人形机器人）/ 游戏动画管线（动捕→游戏骨架）
```

## Relationships

### Based On

- **优先优化 / task-priority 方法**（机器人冗余控制谱系，[31,6,7] 等）——"高层目标决定解集、低层在空空间里优化"的经典原理；
- **IK 式重定向**（约束优化形式）与常用重定向目标集合。

### Extends

- 优先优化：从在线控制/MPC → **离线长序列重定向**（逐帧投影的工程化）；
- 加权多目标重定向：保持目标集合不变，置换"冲突解决机制"（权重 → 层级）。

### Related

- [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement|Skinned Motion Retargeting]] · [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]——库内重定向对照（学习式 vs 优化式；"廉价参考解定位"问题）；
- [[Physics-based Character Animation]]——下游 RL 参考轨迹应用侧；
- [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]——"约束/提示只在关键处用力"的同日呼应。

### Followed By

- 待观察：代码/数据放出（承诺发表后）；UE/游戏侧链路是否会引入"显式层级"式重定向。

## Personal Knowledge State

- Normal（结论层 15 分钟："层级 ≥ 权重" + 空空间投影直觉 + "向量不标量"）。

## Learning Value

- **三句现场可用**：① "重要的东西升级为约束，次要的留在权重里"；② "约束误差**向量**，不是损失**标量**"（防补偿）；③ "优化器要热身，自由目标要出口"（warmup + free objectives）。

## Visualization

（本篇未单独作图解；机制示意见 Daily 关系段与 [[Motion Retargeting]] 概念笔记。）

## Notes

- 周一公告组捕获；提交 2026-10-08 19:03 UTC（"公告日 ≠ 提交日"）；
- 作者单位：SFU（Ioannidis/Yang/Zhang/Peng）、FieldAI（Smith/Vu/Omidshafiei）、NVIDIA + Amii（Peng 兼职）；项目页 primoretarget-page；
- 真机：Unitree G1 / H1-2、DEEP Robotics DR02、Booster T1；代码与重定向数据承诺"upon publication"放出。
