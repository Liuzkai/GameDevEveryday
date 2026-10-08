---
type: concept
user_level: Normal
aliases: [Facial Animation, 面部动画, Facial Retargeting, 面部重定向, 表情重定向, 面部捕捉]
prerequisites: []
first_introduced: "blendshape / FACS（1970s 语境）；mesh-agnostic 神经重定向（2020s）"
---

# Facial Animation

> 建立于 **2026-10-08（Run 30）**。由 [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]] 入库触发——**面部线是库内最后一个"大品类"空白**（身体动画有 [[Motion Matching]] / [[Motion Generation]] / [[Neural Animation]] 一串节点；面部此前零覆盖）。
> 定位：**这是"角色动画"的第二个独立问题域**，不是身体动画的附属——它的表示、数据形态与失败模式都不同（见下）。

## Definition

**让"脸"可信地动起来：表示（脸怎么被驱动）+ 数据（动从哪来）+ 重定向（动怎么换到另一张脸）。** 三层各不相同，但互相绑定——

| 层 | 关心什么 | 典型形态 |
|---|---|---|
| **表示** | 脸由什么参数驱动 | 骨骼（jaw/brow/eye 等有限自由度）· **blendshape（混合形状，几十到几百维）** · 神经（mesh-agnostic 的隐式表示） |
| **数据** | 表情从哪来 | 光学捕捉（头戴摄像机 / 头盔）· 视频驱动 · 音频驱动 · **非光学新通道（EMG 等）** |
| **重定向** | 同一段表情怎么打到不同的头 | 源身份 → 目标身份的表情迁移（**PDB 的域**） |

**与身体动画的三处根本差异：**
1. **形变是高频局部现象**：皱纹、眼睑、内嘴——"表面细节的忠实度"在面部里的权重远高于身体（一只小指头抖动没人看，一条下眼睑曲线错了全毁了）；
2. **自由度形态不同**：身体是骨架链 + 时序检索/生成；面部是**高密度非刚体形变场**（或等价的低维参数流）；
3. **数据形态不同**：身体捕捉是稀疏关节轨迹；面部捕捉是**逐帧高维参数流 / 高密度网格**——这也是"重定向"在面部比在身体更棘手的原因（见下）。

## Core Principle

### ① 面部重定向 = "身份 × 表情"的分解问题

任意两张脸之间做表情迁移，本质上是把一块**纠缠的形变**拆成两半：**哪些成分属于"这张脸"（身份/结构），哪些成分属于"这个表情"（源运动）**。历史方法（按表示分族）：

```text
几何控制族：LBS 蒙皮（1988）· cage-based（2005–2007 均值/调和坐标）· CSFB（2024，blendshape→蒙皮）
        ↓ 神经化
神经控制族：NeuralCage 2020 · KeypointDeformer 2021 · DeepMetaHandles 2021 · VBC 2023
        ↓ 两强并立
逐顶点位移族（捕捉细腻、有表面噪声） ←→ 逐 Jacobian + 全局求解族（表面光滑、有全局耦合）
        ↓
★ 点基双因子族：PDB 2026（变形控制点 × 目标权重 = 矩阵乘积——无 cage/坐标/解码器/全局求解）
```

**三种典型失败模式**（评估任何重定向方案先查这三个）：① **表面噪声**（逐点预测的随机误差直接显形）；② **全局耦合**（全局求解把远端的错误拖过来）；③ **身份泄漏**（源的脸型特征混进目标）。

### ② "数据流"的三问

- **捕捉帧率够不够**：面部微表情在 10–30 Hz 量级，而瞳孔/嘴唇有更快分量——多数管线做时间滤波（代价是"生动度"）；
- **重定向的单位是什么**：每帧重算全量？还是"目标侧算一次 + 每帧只算变化量"？（**PDB 的答案：权重一次、控制点每帧**——与 [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA]] 的"共享基 + 个体系数"同母题）；
- **修得动吗**（生产刚需）：生成结果必需可局部修复（关键帧钉住 + 其余补全）——**"可控性"在面部管线里与质量同级**。

## Prerequisites

- （无强制前置；本域自成一个问题空间）
- 有用的背景：你对角色资产管线（头模拓扑、blendshape/骨骼 rig）的工程经验

## Historical Evolution

```text
1970s    FACS 面部动作编码体系（Ekman 语境）与最早的计算机面部动画（Parke 1972）——
          "脸可以被参数化"的思想起点【库外；细分史料待深挖时核实】
        ↓
1990s+    blendshape / morph target 成为影视与游戏的工业表示
        ↓
2008     LBS 式面部蒙皮与动画系统普及（"骨骼 + 混合形状"混合管线）
        ↓
2017     ARKit 将 52 维 blendshape 流带入消费级（iPhone 面部追踪）——
          "捕捉 → 引擎"的平民化通道打开
        ↓
2020s    神经化：mesh-agnostic 重定向（NeuralCage 2020 → NFS/NFR 2022–2025）
        + 实时化身：blendshape 蒸馏（GALA 2026）
        ↓
★ 2026   点基双因子重定向（PDB）——"四个不需要"的工程极简形态
        + 非光学捕捉探索（emg2face：HD-sEMG，HMD 遮挡场景）
```

> ⚠️ 历史条目以公认事实为主；深入细分（各家首作时间线）在后续深挖时逐条核实。

## Important Papers

| 论文 | 角色 |
|---|---|
| [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting]] | ★ **2026-10-08 入库：本域库内第一节点（重定向子题）**。点基双因子：变形控制点（每帧）× 目标权重（每身份一次）；无 cage/坐标/解码器/全局求解；183 帧端到端 1 秒；风格化头泛化 |
| [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars]] | 相邻子题（**化身表示**）：神经 → blendshape 基的蒸馏，"一套基代表全部身份"；手机 60 fps |
| （记名）emg2face（arXiv 2610.09304） | **数据侧新通道**：高密度表面 EMG（64 通道；HMD 遮挡场景的非光学捕捉）；亚毫秒音爆同步；这一支若成熟会改变"头盔内怎么捕捉脸" |

## Related Concepts

- [[Neural Animation]] / [[Motion Generation]] —— 身体侧的对应域（对照：检索/生成 vs 重定向/形变）
- [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA 所属]] 的"化身/表示"线
- [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]] —— **身体重定向**（对照：重定向问题在身体侧的形式化样本）
- [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|NMS]] —— 同实验室（KAIST VML）身体线（PDB 作者组）

## Technologies

- **ARKit 面部追踪**：52 维 blendshape 流的消费级标准输入
- **Epic Live Link Face**：iOS 捕捉 → UE 的现成通道（PDB 评测管线即用此采集）
- **MetaHuman**：高保真数字人脸资产 + rig 管线（引擎侧细节待核实）
- **音频驱动面部**（Audio2Face 类）：无捕捉数据时的近似通道【细节待核实】

## Game Applications

- **过场/对话特写**：面部是"演员演技"的载体——表情数据要给主角、NPC、不同头模分发（重定向的产量需求）；
- **NPC 大规模对话**：低配头骨架 + 简化重定向——**"权重一次烘焙"形态天然适合量产**（推断）；
- **玩家虚拟形象**：捕捉自己的脸 → 化身（GALA/emg2face 方向）；
- **与身体线合流**：[[2026-10-07-DynaConTalk — Wavelet-Constrained Diffusion for Long-Form and Controllable Holistic Co-Speech 3D Motion|DynaConTalk]] 已在生成"全身含面部"的说话动作——**面部正在从"单独修"变成"多路生成的一部分"**（推断）。

## Personal Knowledge

- **user_level: Normal（推断）**。你做角色 VFX/分档，面部诸环节（头模、材质、特写表现）是你的工程邻域；但**重定向的表示层与神经方法**是新的（本批随 PDB 入库开始积累）。**不推断任何"已掌握"**。
- 本域与你的直接接口：**过场特写质量**、**角色资产量产**、以及"发片/发丝"同类的"表示 ↔ 预算"问题（面部网格密度 × 形变器开销）。

## Learning Gap

1. ~~重定向表示层零覆盖~~ ✅ 2026-10-08 由 PDB 补（表示层结论完全可读）；
2. **blendshape 基础**（FACS 到 ARKit 参数空间的操作语义）——库内尚无可独立成篇的材料，**按需补**；
3. **神经方法的训练层**（扩散/flow 在面部的应用）——暂列 Hard（可沿既有 [[Neural Animation]] 的桥走）；
4. **引擎实装现状**（MetaHuman / UE 面部管线细节）——核实后再写，不预制结论。

## Next Step

1. **读 PDB 的结论层**（§1 失败模式 + §3 双因子 + 运行表）≈20 分钟——按你的"结论优先"读法；
2. 观察 emg2face 类非光学捕捉的后续（HMD 场景的潜在刚需）；
3. 若需要：把"blendshape 基础"列为未来经典候选（FACS/ArkIt 参数体系的操作性材料）。
