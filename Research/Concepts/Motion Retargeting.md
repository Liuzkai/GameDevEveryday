---
type: concept
user_level: Normal
aliases: [Motion Retargeting, 动作重定向, Retargeting, 运动重定向]
prerequisites: [骨架/关节链概念, IK（逆运动学）直觉, 动捕数据格式基础]
first_introduced: "1990s（IK 式重定向 / 相对空间几何表示）；2024-2026（优化式与学习式两条线并行）"
---
# Motion Retargeting

## Definition

**把"源形态"的动作数据搬到"目标形态"上，同时保住动作的"定义性特征"**——人→人形机器人、动捕演员→不同比例的玩家角色、大骨架→小骨架。**目标是"像原动作"，不是"逐关节字面复制"**：脚要踩在地上、手要拿住箱子、两人不能穿模——**这些"关系"比关节角度更重要**。

## Core Principle

**重定向 = 一类约束优化问题**，要同时保住的东西（优先级从高到低是设计核心）：

```text
典型的"要保清单"：
① 交互关系：接触（手-物、脚-地）、空间关系（双人相对位置）
② 任务特征：脚部位置/滑动、平衡
③ 姿势匹配：关节角度/末端位置（"像不像"）
④ 形态约束：关节限位、自穿透
⑤ 平滑性：时间连续性（抖动）
```

**三种路线**：

```text
IK 式优化（工业主流）：把上面清单写成目标/约束，逐链或全身求解
  —— 核心难题：目标冲突时"谁让路"？（权重 = 软汇率，调参负担）
数据驱动：学习"解耦运动与形态"的表示（受训练分布限制）
优先级优化：显式层级 + 空空间投影（重要的升级为约束）★ 2026 新节点
```

**两条核心难点**：① **目标冲突**（接触 vs 姿势，改善一个会伤害另一个）；② **分布跨度**（比例/拓扑差异越大，"像"的定义越模糊）。

## Prerequisites

- 骨架层级与关节链、正向/逆向运动学（IK 直觉即可）；
- 动捕数据的表示（关节旋转/位置轨迹）；
- 基础优化直觉（目标函数、约束、权重 vs 层级）。

## Historical Evolution

```text
1990s-2000s  IK 式重定向：相对空间几何表示交互 [Gleicher 等]；
             进入 DCC 工具链（MotionBuilder HumanIK 等）
2010s        数据驱动线兴起：从运动数据学"形态无关"表示；
             接触感知重定向（几何 + 接触信息）；
             游戏引擎化：Unity Mecanim 人形 Avatar、UE HumanIK
2020s        学习式与优化式并行；物理可行性增强（轨迹优化/仿真在环）
2024        库内节点：[[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement|伪影驱动运动学先验精修]]（学习式）
2026-09      库内节点：[[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]（生成式精修流；"参考解当初始化还是目标"）
2026-10      ★ 库内节点：[[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]]
             （**优先级层级 + 空空间投影**：把"谁让路"从权重升级为约束；
              逐帧投影使长序列/多角色可用；下游 RL 验证）
2026-10      ★ 库内节点：[[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology|CAST]]（**捕捉侧**）
             （目标骨架拓扑 ≠ 视频主体时的单目动捕：可观测性分析把旋转分量分成
              "骨骼方向可约束 / 不可约束"，可靠性门控做几何修正、其余交给学习——
              **"捕捉 + 重定向"的边界开始合并**；A100 51+ FPS，代码开源）
```

## Important Papers

- [[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology|CAST（2026）]] —— **捕捉侧**：任意目标拓扑的单目动捕（可观测性分析 + 可靠性门控；"捕捉即重定向"雏形）★ 2026-10-14 入库
- [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget（2026）]] —— **优先级层级重定向**（约束取代权重；向量误差保护；逐帧工程化）★ 2026-10-13 入库
- [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]] —— 学习式（伪影驱动先验）
- [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM（2026）]] —— 生成式精修

## Related Concepts

- [[Motion Matching]] / [[Motion Generation]] —— 数据"来源侧"；重定向是数据"入库侧"（动捕 → 可用于检索/训练的形态）
- [[Physics-based Character Animation]] —— 下游：重定向动作作为 RL 参考轨迹（PriMoRetarget 的验证方式）
- [[Animation Compression]] —— 同属"动画数据管线"（压缩 vs 转换），预算语言互补

## Technologies

- **UE IK Rig / IK Retargeter**（引擎侧工业形态：链式 IK + 逐链求解的"简化层级"）
- **Unity Mecanim Humanoid Avatar**（内置人形重定向）
- **Autodesk MotionBuilder HumanIK**（影视/动捕侧标准工具）
- 机器人侧：Unitree G1/H1-2、Booster T1 等（RL 训练参考轨迹）

## Game Applications

- 动捕数据 → 玩家角色/多体型 NPC（自定义体型、怪物、双人交互动作）；
- 复杂交互动画（搬箱、开门、双人协作）的批量重定向——"接触别丢、脚别滑"是验收点；
- 跨品类复用：一套动作数据 → 多种骨架资产。

## Personal Knowledge

- 新增概念（2026-10-13 建立）：`user_level: Normal`（**推断**：动画管线背景 + 库内三篇材料；优化推导层未验证）；
- 库内连接：与 [[Motion Matching]]（检索侧）、[[Physics-based Character Animation]]（下游）构成"动作数据全管线"三环。

## Learning Gap

- IK 求解器实现细节（阻尼最小二乘、信赖域）× 按需；
- 数据驱动线的"形态解耦表示"具体构造（按需）。

## Next Step

1. （可选，≈15 分钟）读 [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]] 结论层——"层级 ≥ 权重 + 空空间投影"两句即可；
2. （观察项）UE 侧是否引入"显式优先级"式多目标重定向（当前为链式 IK 惯例）。
