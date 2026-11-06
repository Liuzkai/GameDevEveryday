---
type: concept
user_level: Normal
aliases:
  - Automatic Rigging
  - Auto-Rigging
  - 自动绑骨
  - 自动绑定
  - 骨骼生成
  - Skeleton Embedding
prerequisites:
  - "[[Skinning]]"
  - "[[Skeletal Animation]]"
first_introduced: "Baran & Popović, SIGGRAPH 2007（Pinocchio）"
---

# Automatic Rigging

## Definition

**把"一个静止网格"变成"一个能被骨骼驱动的资产"的那一次转换。** 产出三样东西：

| 产出 | 是什么 | 错了会怎样 |
|---|---|---|
| **骨架拓扑** | 一棵关节树：谁是谁的父、一共几根 | 动起来结构怪（脊柱连到骨盆、手指骨穿到脚底） |
| **关节位置** | 每个关节在网格内的坐标 | 弯的时候"从错的地方折"（膝盖折在小腿中段） |
| **顶点权重** | 每个顶点受哪几根骨影响、各多少 | 该跟躯干的跟了手臂（腋下粘连）、关节处糖纸 |

**它和 [[Skinning]] 的分界**：Skinning 回答"**给定**权重，怎么混才对"；Rigging 回答"**权重和骨架从哪来**"。**Rigging 是一次性的（资产级），Skinning 是每帧的（运行时）。** 这也是为什么 Rigging 的成果能烘焙进资产、而蒙皮算法是引擎里的一个开关。

**它和 [[Skeletal Animation]] 的分界**：骨骼动画页把"骨架 / 姿态 / 蒙皮 / 求值"四层列清楚了，其中"骨架"和"蒙皮权重"两栏的负责人都写着"绑定师"——**本页就是那个"绑定师"被自动化之后的样子**。

## Core Principle

### 五个子问题（不是一个）

自动绑骨常被当成一个任务，实际是五个可以独立失败的问题：

1. **骨架从哪来？** 三种立场——
   - **模板**（人给一棵树，方法负责塞进去）：[[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran 2007]]；
   - **提取**（从形状算中轴）：Teichmann & Teller 1998 / Liu 2003 / Katz & Tal 2003；
   - **生成**（从数据学一个先验）：[[Xu 2020 — RigNet Neural Rigging for Articulated Characters|RigNet 2020]] 之后的主流，[[2026-11-02-Guo — FlowRig Unified Skeleton Generation for 3D and 4D Meshes via Flow Matching|FlowRig 2026]]。
2. **关节放哪？** 本质是**子图同构**：在"体内图"上找一棵与骨架同构的子树，使某个罚项最小。
3. **权重怎么给？** 热平衡（Baran）/ 测地距离（Maya 的 geodesic voxel bind）/ 学习（SkinningTransformer 一类）。
4. **多少根骨、谁连谁？** ——**这一条直到 2026 年才变成"可指定的输入"**，在此之前它是方法的副产品。
5. **一段动画里骨架要不要变？** 3D（一次）vs 4D（每帧一套、还是整段一套）。

**五个问题里，前三个是几何，第四个是"表示"，第五个是"时序"。** 把它们混成"绑骨精度"一个指标，是这条线上最常见的误读。

### 判据：骨架是接口，不是形状的函数

Baran 2007 立下的这条，是理解整条线的钥匙：

> 提取出来的骨架"属于这个形状"，但动捕数据不属于这个形状。

后果有三条，方向相反：

- **好**：骨架一样，动作就能复用——这是"一个动作库 + 一堆角色"的前提，也是重定向的前提；
- **好**：骨架一样，工具链就能复用（IK、物理资产、动画蓝图、LOD）；
- **坏**：骨架一样，形状就得迁就它——胖角色和瘦角色共用一副骨架，形变质量必然打折。

**所以"自动绑骨"的目标从来不是"生成最自然的骨架"，是"生成最能接住动作的骨架"。** 这两个目标在有冲突时会选后者。

### 两类错误的诊断

工程上最快的判据不是看指标，是看动起来的样子：

| 现象 | 病因 | 修法 |
|---|---|---|
| 某块表面"被远处拽走"、腋下粘连 | **权重**（距离法把躯干算给了手臂） | 重算权重 / 手工刷 |
| 弯折发生在肢体的错误位置 | **关节位置**（嵌入错） | 调关节，或给一个位置提示 |
| 整段动画里骨架"闪"、骨长忽长忽短 | **时序**（逐帧独立决策） | 换方法，不是加平滑 |
| 结构怪但形变"对" | **拓扑**（父链错） | 改骨架树 |

**第三行是 2026 年才被明确的一类**——它不能靠"权重刷得好"解决，因为病不在权重。

## Prerequisites

- **线性混合蒙皮**（[[Skinning]]）：知道 $v'=\sum_i w_iT_iv$ 里 $w_i$ 的角色，才知道要解什么；
- **离散曲面 Laplacian 与 cotangent 权重**：热平衡权重就是在解 $-\Delta w+Hw=Hp$，不懂 Laplacian 就只能把它当黑盒；
- **图的子图同构**：离散嵌入那一步的本质；
- **蒙皮不是形变**（[[Skeletal Animation]] 的"四层"）：绑骨只管"骨动皮跟着动"，不管"动得像不像"。

## Historical Evolution

```text
1988  LBS 表述：权重是输入，没人管它从哪来
  ↓
2007  ★ Baran & Popović（Pinocchio）
        · 嵌入而非提取 ⇒ 骨架成为接口
        · 热平衡权重 ⇒ 权重第一次有算法
        · max-margin 学罚项配比 ⇒ "品味"可标注
        · 81% 成功 / 31 s / 单线程 CPU
  ↓
2011  Bounded Biharmonic Weights（同组）：热扩散 → 有界调和，权重更局部、更可控
  ↓
2010s bone heat 成为 DCC 默认（Blender "With Automatic Weights" 沿用同一构造：
        曲面热平衡 + 体内可见性门控）；
        Maya 走另一条路（geodesic voxel bind：体素化 + 测地距离）
        ⇒ 工序定型为"自动打底 + 手工修关节"
  ↓
2020  ★ RigNet：GNN + 加权 mean-shift + MST + 测地蒙皮（**无模板**）
        · 关节点 IoU 36.5% → 61.6%；蒙皮 L1 0.65（Maya）→ 0.39
        · 带宽滑杆 = 骨骼 LOD 的第一个旋钮（几何参数，不是模型条件）
        · 离散结构交给算法（模态个数定关节数、MST 定拓扑），连续量交给网络
  ↓
2023-25  MagicArticulate / UniRig / RigAnything / Puppeteer
        · 大模型化、自回归、逐子部件
        · 共同病：骨骼数不可指定；逐帧套用会闪
  ↓
2026  FlowRig：整段序列当一个样本 + 骨骼数当条件（1–128）+ 拓扑池化一次
        静态 1.1 s（GPU），24 帧 23.4 s
```

**三十年没有"取代"，只有"分层"**——和 [[Skinning]] 页的结论一致：

| 层 | 回答什么 | 今天还在用吗 |
|---|---|---|
| 几何（2007） | 给定模板，塞进去 | ✅ 仍是 DCC 默认，且是**最快的**（无训练、可解释） |
| 学习（2020→） | 从数据学先验，免模板 | ✅ 处理"没有合适模板"的资产（动物、怪物、道具） |
| 时序（2026） | 一整段序列共享一套骨架 | ⚠️ 离线可用，未进产线 |

## Important Papers

- [[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran & Popović 2007（Pinocchio）]] — **本页主节点**：问题定义、"接口"立场、热平衡权重、max-margin 罚项学习，四件事一次做完
- [[2026-11-02-Guo — FlowRig Unified Skeleton Generation for 3D and 4D Meshes via Flow Matching|FlowRig 2026]] — **把"骨骼数"变成输入**，并把 3D/4D 统一成一个任务
- [[Xu 2020 — RigNet Neural Rigging for Articulated Characters|RigNet 2020]] — **学习路线的起点，本页时间线的中段**：无模板（对比 2007 的四套模板）、"离散结构交给算法"（mean-shift 取模态 + MST 保合法）、**第一个把权重质量单独列表并赢过 DCC 默认算法的工作**（L1 0.39 vs Maya GeoVoxel 0.65）；带宽滑杆 = 骨骼 LOD 的第一个旋钮
- （库外对照）MagicArticulate / UniRig / RigAnything / Puppeteer —— 精度与速度的横向快照见 FlowRig 表 1

## Related Concepts

- [[Skinning]]（**下游**：本页给权重，蒙皮页用权重。**本页补上了蒙皮页一直当作"给定"的那一环**）
- [[Skeletal Animation]]（**上游契约**：骨架是运行时的数据接口；本页的产出必须满足它）
- [[Motion Retargeting]]（**逻辑上的兄弟**：重定向要求"骨架一样"，绑骨决定"骨架是什么"——**两者是同一个接口问题的两端**）
- [[Mesh Parameterization]]（**同族数学**：都在网格上解 Laplacian 系统，都把边界条件当设计变量。参数化固定边界求内部映射，热平衡固定骨温求权重）
- **反面立场**：[[Physics-based Character Animation]]（那边是"不要骨架，直接解力"——是同一个目标的两条路）

## Technologies

- **Bone heat / 热平衡权重**（DCC 默认）：曲面 Laplacian + 体内门控；**失败可预测**（见 Baran 页"为什么能 work"第 3 条：没有热源的连通分量会让系统奇异）；
- **Geodesic voxel bind**（Maya）：体素化 + 测地距离，解决同一个"欧氏距离会跨过空隙"的问题，**路径不同**（测地 vs 扩散）；
- **学习型 rigger**（RigNet → FlowRig 一系）：免模板，但要训练数据与推理成本；
- **骨骼 LOD / 骨骼精简**：引擎侧的对应物，走的是"删骨 + 重算权重"，**保拓扑不保形状**——与"重新生成一副骨架"是两条不同的路。

## Game Applications

- **扫描 / 外包 / UGC 资产入库**：只有网格、没有骨架的资产，先自动绑一次再进管线；
- **LOD 骨架**：远景 12 根、中景 20 根、特写 27 根（FlowRig 图 6 的直接演示）。**过去这是三份绑定资产，现在是同一个模型换一个条件**；
- **动物 / 怪物 / 道具**：没有人形模板可用，学习型方法是唯一现实选择；
- **4D 缓存转骨骼**：把顶点动画（Alembic、解算缓存、商店买的带动画资产）反推成骨骼动画——**转成骨骼才能重定向、才能压缩、才能进动画蓝图**；
- **大量 NPC 变体**：同一套动画要套到几十个体型上，绑骨质量决定"要不要每人手修"。

## Personal Knowledge

Current Level: **Normal**

- **Easy**：自动权重是"打底"、关节处要手修、糖纸是蒙皮问题不是绑骨问题；
- **Normal**：为什么距离法会跨空隙、"嵌入 vs 提取"的取舍、热平衡为什么自动尊重连通性、骨骼数为什么在自回归方法里不可指定；
- **Hard**：A* 下界构造、flow matching 的 OT 推导、max-margin 的非凸搜索——**除要复现，都不必碰**。

## Learning Gap

**缺的不是"知道有自动绑骨"，是三样具体的量：**

1. **中段仍然只有一半**：2020 的 [[Xu 2020 — RigNet Neural Rigging for Articulated Characters|RigNet]] 已入库（2026-11-06），但 **2020 → 2026 之间的 2023–2025 那一批（MagicArticulate / UniRig / RigAnything / Puppeteer）仍是库内零覆盖**。**结果是"2020 的中等形态"与"2026 的整体形态"之间还缺一次"大模型化"的过渡**——看不出精度是什么时候、靠什么跳上去的；
2. **"骨骼数—精度"曲线没有实测**：FlowRig 给了 $n=27/20/12$ 的定性图，但**没有给"少到几根开始崩"的定量拐点**。这个数对 LOD 骨架的决策是直接可用的，现在只能靠试；
3. **权重质量的评估缺失**：所有方法都在比"骨架对不对"（J2J/J2B/B2B），**几乎没有人比"权重刷得好不好"**——而美术每天在修的恰恰是后者。

## Next Step

**一个 20 分钟的实验，验证"热源"这条判据**：

1. 拿一个**薄片 + 穿插**的资产（披风、裙摆、或两层壳的角色），在 Blender 里做 `Armature Deform → With Automatic Weights`；
2. 看它是否报 `Bone Heat Weighting: failed to find solution for one or more bones`；
3. 如果是，**不要调参数**——把网格切开看连通分量，找出"所有顶点到最近骨的线段都不在体内"的那一块（通常就是薄片或内层壳）；
4. 修法只有三个：**给那块补一根骨 / 修穿插让线段回到体内 / 把薄片加厚到 ≥2τ**。

**做完这一步，你就把"自动权重为什么失败"从玄学变成了可诊断的问题**——这是本页最实用的一条。

**读法建议**：[[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran 2007]] 只需读到 §4（权重）结束，约 40 分钟；§3.3 的 max-margin 与 §3.4 的 A* 可以跳，读结论即可。
