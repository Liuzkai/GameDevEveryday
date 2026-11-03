---
type: concept
user_level: Normal
aliases:
  - Skeleton Animation
  - Skeletal Animation System
  - 骨骼动画
prerequisites:
  - "[[Skinning]]"
  - "[[Motion Retargeting]]"
  - "[[Animation Compression]]"
first_introduced: "1976（Burtnyk & Wein 的交互式骨架技术）；1988（LBS 的正式表述）"
---

# Skeletal Animation

## Definition

**骨骼动画 = 一套"用少数关节驱动大量顶点"的运行时系统**，四件东西缺一不可：

| 层 | 是什么 | 谁负责 |
|---|---|---|
| **骨架** | 一棵关节树 + rest pose（每关节的局部变换） | 绑定师 |
| **姿态** | 每关节相对 rest 的旋转（+ 根位移） | 动画师 / 动捕 / 生成模型 |
| **蒙皮** | 顶点 → 关节的权重 | 绑定师（自动或手绘） |
| **求值** | 正向运动学（FK）→ 蒙皮矩阵 → 顶点 | 引擎每帧 |

**它与 [[Skinning]] 的分界要写清楚**：**Skinning 只回答"顶点怎么跟着关节动"**（LBS / DQS / 神经蒙皮，是"混合什么"的数学）；**Skeletal Animation 回答"关节怎么动、动完怎么送到 GPU"**（FK、姿态表示、重定向、压缩、LOD、root motion）。**前者是一篇论文能解决的问题，后者是一条产线。**

## Core Principle

### 一、骨架里有"结构"，也有"约定"——只有结构该被建模

今天（2026-11-02）的 [[2026-10-25-Sun — TopX Text-to-Motion for Unseen Skeletons from the Skeleton File Alone|TopX]] 把这件事讲得最清楚：

| 属于**约定**（可消、不该学） | 属于**结构**（必须学、不能消） |
|---|---|
| 关节的**排列顺序** | 树的拓扑（谁是谁的父） |
| 关节的**命名**（`L_Femur_02` vs `b_hindleg_l_1`） | 关节数 |
| 每个关节 rest frame 的**朝向** | 骨长比例 / 体型 |
| 文件的组织方式 | 关节的语义角色（前/后、左/右、末端） |

**判据**：*任何一个跨骨架的算法，先问它吃的是"约定"还是"结构"。吃约定的，换一套 rig 就崩；吃结构的，才能泛化。*

### 二、FK 的成本是"深度"，蒙皮的成本是"顶点 × 影响骨数"

$$M_j = M_{\text{parent}(j)}\cdot T_j^{\text{rest}}\cdot R_j$$

- **FK**：沿树累乘，成本 $\propto$ 关节数 × 树深（并行化后≈关节数）；
- **蒙皮**：每顶点读 4 个矩阵加权，成本 $\propto$ 顶点数 × 影响骨数；
- **GPU 侧**：骨骼矩阵通常进**常量缓冲 / 纹理**，超过一定数量（如 256 骨）就要换方案（骨骼纹理 / 分批绘制）。

**这就是"骨骼数"为什么是一个预算项**：它同时压 FK、压蒙皮带宽、压 batches。

### 三、动画数据的三个可压缩维度

| 维度 | 压缩什么 | 代表 |
|---|---|---|
| **时间** | 关键帧密度（曲线拟合、误差界） | [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model|CurveCodec 2]] |
| **空间** | 关节数（骨骼 LOD：把手指并到手掌） | 引擎内置 |
| **表示** | 从"逐关节曲线"换成"参数化/基" | 主成分、运动匹配的数据库 |

**三条互相独立**，可以叠加——**但它们降档时损失的东西不同**：时间压缩损失的是"快速动作的锐度"，空间压缩损失的是"末端细节"，表示压缩损失的是"可编辑性"。**"动画 LOD"这个词把三件事混成了一件，这是工程里最常见的口径错误。**

### 四、root motion：骨架与 gameplay 的接缝

根关节的位移有两种处理：**in-place**（动画不含位移，由控制器移动角色）与 **root motion**（动画自带位移，引擎跟随）。**这条接缝是所有"动作与碰撞不一致"问题的来源**：脚滑、穿墙、位移与输入冲突，全都出在这里。

## Historical Evolution

```text
1976  Burtnyk & Wein        交互式骨架技术：用少数关节插值驱动整幅画面（2D）
1985  Shoemake              四元数曲线：旋转的插值第一次成为良定义
1988  Magnenat-Thalmann     LBS：关节相关的局部形变（蒙皮的正式表述）
2002  Kovar  Motion Graphs  动作片段图：动画从"手 K"变成"检索 + 拼接"
2008  Kavan  DQS            把混合空间换成对线性组合封闭的表示
2020  Holden LMM            用神经网络替代数据库：运动匹配的连续化
2026  TopX                  骨架文件本身成为生成模型的接口
        ↓
      "驱动方式：手 K → 检索 → 生成；
       骨架的地位：坐标系 → 适配对象 → **接口**"
```

**（1976 / 1985 / 1988 三条为书目来源，本次未逐篇取到原文；四元数与 LBS 的年份与出处是领域共识，但引用前建议核一次。）**

## Important Papers

| 论文 | 在这条线上的位置 |
|---|---|
| [[Kovar — Motion Graphs (2002)]] | 从"手 K"到"检索"的分水岭 |
| [[Learned Motion Matching (Holden 2020)]] | 检索的连续化，本库的 MM 基准 |
| [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)]] | 蒙皮数学的关键一步（详见 [[Skinning]]） |
| [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model|CurveCodec 2]] | "误差界即档位"：压缩的验收标准 |
| [[UniMate — One Unified Model to Animate Diverse Skeletons|UniMate]] | 零样本跨骨架（谱编码 + SignNet） |
| [[2026-10-25-Sun — TopX Text-to-Motion for Unseen Skeletons from the Skeleton File Alone|TopX]] | 骨架文件即接口（热核 RoPE + rest 归一化） |
| [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]] | 物理化蒙皮 |

## Related Concepts

- [[Skinning]]：**下游的数学**。Skeletal Animation 产出关节矩阵，Skinning 消费它们；
- [[Motion Retargeting]]：**同一条数据换一副骨架**——"约定"的清理工；
- [[Motion Matching]]：**从数据库里选姿态**，取代逐帧曲线；
- [[Motion Generation]] / [[Neural Animation]]：**从条件生成姿态**，取代检索；
- [[Animation Compression]]：数据的存储形态；
- [[Facial Animation]]：同一套骨架思想在面部（blendshape + 骨）上的变体；
- [[Secondary Motion]]：**骨架之上的一层算子**——骨架给"主体运动"，二次运动给"骨架停了之后的那点残余"。**2026-11-03 补**：[[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|Dynamic Skinning]] 证明它可以完全由骨架量（$\Omega_j$、$q_j$、层级）驱动，**不需要额外的自由度**——这是本页与仿真路线的分界；
- [[Physics-based Character Animation]]：把"关节怎么动"从关键帧换成物理。

## Technologies

- **姿态表示**：欧拉角（有万向锁）/ 四元数（插值良定义）/ **6D 旋转**（连续、可微，TopX 用这个）；
- **FK**：局部变换累乘，GPU 侧常预计算蒙皮矩阵；
- **蒙皮**：LBS / DQS / 校正形变（PSD、corrective blob）/ 神经蒙皮；
- **压缩**：曲线拟合 + 量化 + 熵编码；
- **重定向**：链映射 + 逆向 FK + 约束求解；
- **LOD**：骨骼裁剪、动画采样率降档、Update Rate Optimization（远处角色隔帧更新）。

## Game Applications

- **骨骼数是一个硬预算**：同屏 100 个角色 × 200 骨 = 2 万关节/帧的 FK + 蒙皮。**这与你的 SABC 预算是同一类账**——只不过计价的单位是"关节"而不是"粒子"；
- **骨骼 LOD 与动画 LOD 要分开配**：远处角色通常**同时**降"骨骼数"与"更新频率"，两者的失效方式不同（前者丢末端细节，后者丢动作锐度）；
- **VFX 挂在骨骼上**：socket / bone attach 是特效的锚点。**骨骼 LOD 一裁，挂在被裁骨骼上的特效会突然跳到父骨骼**——这是"分档改动引发表现 bug"的经典案例；
- **多人同屏的身份问题**：同一技能两个角色同时放，特效归属靠"骨骼 + 身份"两件事区分（参见 [[2026-10-29-Li — Fight World Model Customizable Combat through Visual Interaction Proxies|FWM]] 的身份绑定）。

## Personal Knowledge

`Normal`。已有基础：[[Skinning]]（LBS → DQS → 校正形变 → 神经蒙皮的四代阶梯）、[[Motion Matching]]（检索侧）、[[Motion Retargeting]]（重定向侧）。**缺的是"运行时系统"这一层**：FK 的实际成本、骨骼纹理、Update Rate Optimization、root motion 与碰撞的接缝。

**边界**：能说清"为什么 DQS 比 LBS 好"、"重定向的三步"；**不能说清**的是"骨骼数从 200 降到 100 时，CPU 与 GPU 各降多少"——这需要实测，不是读书。

## Learning Gap

- **没有一篇讲"引擎侧骨骼预算"的一手材料**：库里全是方法论文，缺"骨骼数 → FK 成本 → draw call"这条工程链；
- **动画 LOD 的三条维度没有对比数据**：时间压缩 / 空间压缩 / 表示压缩，在同一角色上的画质-成本曲线；
- **root motion 与 gameplay 的接缝**：库里完全没有这块（而它是项目里最常出 bug 的地方）。

## Next Step

1. **补一手工程材料**：UE 的 `Update Rate Optimization` 与 `Bone LOD` 文档（一小时可读，属"资料侧"动作）；
2. **补一条可测的账**：拿一个现有角色，把骨骼数从 N 砍到 N/2，测 FK 时间与 draw call 变化——**这是"骨骼预算"唯一的实证方式**；
3. **读 TopX 时带着一个问题**：它的输出是"关节旋转 + 根轨迹"，**这正好是 Control Rig 的输入**——那么"生成 → 重定向 → 进引擎"这条链里，哪一步还能省掉？
