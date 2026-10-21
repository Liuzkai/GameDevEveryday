---
type: paper
title: "InfiniDream: Open-Ended Photorealistic 3D Worlds Through Procedural Generation"
authors: ["Yi-Chuan Huang", "Hao-Jen Chien", "Zheng-Hui Huang", "Yu-Lun Liu", "Kaipeng Zhang", "Zhixiang Wang"]
year: 2026
published: 2026-10-19
venue: "arXiv preprint (cs.CV)"
url: "https://arxiv.org/abs/2610.26853"
code: ""
project_page: "https://yichuanh.github.io/infinidream-page/"
category: ["procedural-content-generation", "3d-scene-generation", "vlm-agent", "world-building"]
importance: A-
historical_importance: 2
game_relevance: 5
production_readiness: "Prototype（依赖 Infinigen/Blender 全栈；室外为主；动画为预定义运动集）"
user_level: Normal
status: unread
---

# InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation

## TL;DR

**"不要让 VLM 写程序化生成代码，要让它改程序化生成代码"**——三条现有路线都不行：VLM 从零写 Blender 代码只能出简化几何；预定义工厂（Infinigen 那类 4 万行手工程序化库）变体有限；把整个代码库丢给 agent 又改不准。InfiniDream 的答案是 **Symbolically Constrained Editing（SCE）**：先用**深度 2 的 AST 依赖分析**圈出"可编辑表面"，把 4 万行代码库压成一个有边界的编辑空间，再让 VLM 在这个空间里做**三类离散决策**（Tweak / Create / Promote）。

第二半是**放置**：不再让 VLM 猜坐标，而是把语义约束翻译成 **确定性视锥网格搜索**——9×5（或 17×9）网格、深度按 ×1.3 递增到 80 m、五道确定性闸门（可见性 / 碰撞 / 实例间距 / 表面策略 / 投影面积带），取"由近到远第一个全部通过"的候选。**无需额外 VLM 调用、无需迭代渲染验证**。

数字：物体级 CLIP 30.07 / DINOv3 0.554 / 代码可执行率 98%（对手 VIGA 27.20 / 0.395 / 90%）；场景级 "物体不可读率" 从 agent 直接放置的 **50.9% 压到 7.8%**。代价：只做室外、动画只能用预定义运动集、且**放置是相机视锥优先的**。

![[Research/Files/Pictures/Huang 2026 — 图3 程序化资产生成四策略对比.png]]
**读图**：同一句提示词（彩色蝴蝶 / 高山野兔）下的四种路线。第 1 列 VLM 从零写代码——几何简化、颜色平；第 2 列预定义工厂——只有库里那几种（兔子是"通用四足"）；第 3 列 VLM 直接操作整个 Infinigen——能改但改歪（蝴蝶的翅膀结构散了）；第 4 列本文——既保住程序化的结构，又长出了新特征（蝴蝶翅脉、兔毛）。**这张图就是 SCE 存在的理由：自由度不是越多越好，"有边界的自由"才可控。**（图 3，Huang et al. 2026）

## Problem

程序化生成的价值在于**可控、可编辑、可种子化**（这三样是网格资产给不了的），但它的瓶颈 20 年没变——**规则编写太贵**（[[Müller — Procedural Modeling of Buildings (2006)]] 原文自认 "rule authoring is laborious"）。2023 年后的 LLM/VLM 想接这个活，但三条路都撞墙：

1. **VLM 直接写生成代码**（VIGA / BlenderRAG / 各类 "code-as-3D"）：能跑，但产出的几何**普遍简化**——LLM 写不出 4 万行级别的真实感细节；
2. **只用预定义工厂**：质量高，但**新颖性为零**——"给我一只彩虹色鹦鹉"只会得到库里最像的那只鸟换色；
3. **VLM agent + 完整代码库访问权**：能力上限最高，但**编辑不精确**——agent 会改到不该改的地方，且每次失败的代价是一次完整渲染验证。

放置侧同样如此：让 VLM 直接输出 3D 坐标，**准确率低且无法验证**；用渲染回路反复试，成本高到不可用。

## Historical Context

```text
2001-2006  规则时代：L-system 城市 → split grammar 立面 → CGA shape
           （[[Parish-Müller — Procedural Modeling of Cities (2001)]] / [[Wonka — Instant Architecture (2003)]] /
             [[Müller — Procedural Modeling of Buildings (2006)]]）
           —— 规则表达力到位，但"谁来写规则"没解决
  ↓
2010s      Houdini / UE PCG 把程序化做成引擎一等公民（节点图 = 规则的可视化形态）
  ↓
2023-2025  LLM 接管的第一次尝试：**写规则**
           ProxyBuild（LLM 解析属性 + 推断结构角色 + 检索组装）★ 2026-09 入库
           Co-DSL（人机协同养成概念库 + 求解器）★ 2026-10 入库
           Scene-Designer（学"设计 + 评判"的 agent）★ 2026-10 入库
  ↓
★ 2026-10  InfiniDream：**改规则**（不写新库，在既有库上做符号约束的补丁）
           + **确定性放置**（语义 → 网格搜索 → 五道闸门，不猜坐标）
```

**它在前一条线上的位置**：ProxyBuild / Co-DSL / Scene-Designer 都在解决"**规则从哪来**"，InfiniDream 换了个问法——**"已经有人写好 4 万行了，能不能只改需要改的那 200 行"**。这是把"生成"从"创造"降级为"**受约束的编辑**"，工程味重得多。

## Previous Work

- **Infinigen**（Raistrick et al.）：本文的底座，4 万行手工程序化资产工厂 + 地形管线 + asset pooling；短板是资产类别固定；
- **VIGA / BlenderRAG / Code2Worlds**：VLM 写 Blender 代码的三个代表，几何简化是共同短板（本文 Table 1/6 直接对照）；
- **VLM agent harness**：给 agent 完整代码库访问权 + 渲染反馈回路——执行率高（99%）但语义对齐差；
- **放置侧**：现有 LLM 场景合成多靠"VLM 输出坐标 + 渲染验证"，或依赖资产库的固定 snap 规则。

## Core Idea

**把"世界生成"拆成两类决策，各用各的工具：**

```text
离散、可枚举、可验证的决策  →  VLM（选哪个工厂 / 改哪个模块 / 放成什么姿态 / 几个实例）
连续、需精确、需可复现的决策 →  确定性算法（AST 依赖分析 / 网格搜索 / 约束求交）
```

这条分工在本文里出现两次，是全篇最可迁移的东西：

- **资产侧**：VLM 只做"编辑计划"（策略 + 位置 + 修改 + 依赖 + 风险五项），**落地由 Coarse Constructor 在符号约束内执行**；
- **放置侧**：VLM 只输出"语义约束"（锚定类型 / 目标区域 / 表面策略 / 密度），**坐标由网格搜索 + 五道闸门决定**。

## Technical Approach

### 1) SCE：把 4 万行代码库变成有边界的编辑空间

| 阶段 | 做什么 | 关键设计 |
|---|---|---|
| **A. 工厂映射与检索** | 按文本（可带参考图）检索最相关的 Infinigen 工厂，产出 *Factory Brief* | **Code Structure Map**（关键函数与依赖）+ **Editable Surface**（**深度 2 的 AST 依赖分析**动态圈定）+ **Linked Infrastructure**（共享底层算法以符号链接**只读**挂入） |
| **B. 结构化规划 + 粗构建** | Planner 产出 *Editing Plan*：Strategy / Locate / Edit / Rely-on / Risks | 三种策略：**Tweak**（调参）/ **Create**（新增程序化函数）/ **Promote**（必要时把编辑权扩到受保护模块）；**保留原始入口点与函数签名**（population 与 seed 机制才不会断） |
| **C. 精细细化** | Reviser 比对渲染结果与目标，在编辑边界内改代码；Selector 选最优候选 | 最多 **6 轮**，支持早停；执行失败触发 **traceback-based repair** |

**一个容易漏掉的工程细节**：要保住"同种子同结果"的多样性，不能把参数写死，而要**调整采样范围**——SCE 明确用"改分布"而不是"改常数"来实现编辑。这一条直接决定了后续"池化"能不能用。

### 2) 语义放置：三种模式 + 七种锚定

- **模式**：Hero（语义重要，单个）/ Scattering（大面积分布）/ Swarm（群体）；
- **Hero 的七种 Anchoring**：rooted / upright / tilted / conform / afloat / airborne / submerged——**"这个东西怎么站在世界上"被显式建模**，而不是让 VLM 猜坐标；
- **Surface Policy**：目标区域、允许的表面类型、坡度上限、是否要求完整支撑；
- Director 还决定实例数 $N$、原型数 $K$、World Motion（含碰撞用的**扫掠包络**）与 Local Animation。

### 3) 确定性视锥网格放置（本文最"工程"的一段）

```text
相机图像平面离散成 9×5 网格（实例≥6 时 17×9）
  → 沿每条相机射线生成候选 3D 位置（表面支撑 = 射线求交；空中/水下 = 沿深度采样）
  → 深度按 ×1.3 递增，最远 80 m（由近到远评估）
  → 五道闸门全部通过者中，取最近的一个：
       ① 可见性  ② 与运动扫掠包络的碰撞  ③ 实例间距
       ④ 表面策略（类型/坡度/四角足迹完整支撑）  ⑤ 投影面积带 a_min ≤ a(p) ≤ a_max
  → 无解时：分层约束松弛（投影面积最小 1.2% → 松弛 0.2%，绝对上限 25%；水域约束永不松弛）
```

- **投影面积带 = 隐式的尺寸感知**：大物体自然被推到远处、小物体留在近处，不需要显式写"大东西放远点"；
- **确定性哈希**保证可复现；**大面积优先**（先放大件，再填小件），避免小件占掉大件的合法位置。

### 4) 池化：把昂贵构建从 N 降到 K

$$\mathcal{M}=\{M_k=G(s_k)\}_{k=1}^{K},\qquad I_i=\mathrm{Clone}\left(M_{((i-1)\bmod K)+1}\right)$$

同一场景里 $N$ 个实例只构建 $K$ 个原型（不同 seed），其余克隆——**这是"资产预算"最朴素也最有效的一条**：贵的是"生成一个新原型"，便宜的是"再摆一个"。

## Key Contribution

1. **"改规则"而不是"写规则"**：把 LLM 的能力锚定在**既有程序化库的符号约束编辑**上，绕开"LLM 写不出真实感细节"的死结；
2. **AST 深度 2 的可编辑表面**：给出了一种**可操作的自由度边界**——不是"给不给权限"，而是"权限的拓扑范围有多大"；
3. **确定性放置取代 VLM 猜坐标**：把"语义 → 位置"从生成问题变成**搜索 + 过滤**问题，可复现、无额外推理成本；
4. **完整的"文本 → 世界"闭环**：资产（新）→ 规划 → 放置 → 池化 → 地形细化 → 动画，八项能力全打勾（Table 7 唯一全勾的方法）。

## Why It Works

- **编辑空间有界 ⇒ 失败可局部化**：VLM 只在 depth-2 的依赖闭包里动手，出错不会污染整个库；traceback repair 也因此有效；
- **离散决策可验证，连续决策可枚举**：语义约束（"靠水边""朝上"）能被翻译成布尔闸门；而坐标本身不需要"预测"，只需要"搜索"；
- **投影面积带把"构图"变成了约束**：屏幕占比是有界的，等于给"这个物体该多大、该多远"一个解析解，VLM 不必学透视；
- **池化承认了成本结构**：生成贵、克隆便宜——所有大规模世界生成系统最终都会走到这一步。

## Limitations

- **只做室外**（作者自认）：室内需要更复杂的空间关系与物体交互；
- **动画是预定义运动集**：World Motion / Local Animation 从固定集合里选，不支持定制运动（作者把"VLM 生成动画代码"列为 future work）；
- **强依赖 Infinigen + Blender 全栈**：换一套程序化库要重做工厂映射；
- **放置是"相机视锥优先"的**——这一条对游戏最致命：候选位置由当前相机射线的 9×5 网格生成，**画面外、背面、转身后没有任何保证**。它优化的是"这一张图好不好看"，不是"这个世界走进去对不对"；
- **最多 6 轮细化 + VLM 后端**：单资产的构建成本仍以"分钟级 VLM 调用"计，且质量受后端模型（Claude Opus 4.8）能力约束；
- **没有运行时预算概念**：本文是离线资产生成，产出直接交 Blender 渲染，不谈 LOD/instance/材质数——离"进引擎"还有一整段。

## Game Development Relevance

- **PCG 线的"第七个样本"**：库内已有"生成只做决策层"六例（ProxyBuild / Co-DSL / Scene-Designer 等），InfiniDream 是**第一个把决策层缩到"补丁"粒度**的——不是"生成一个新资产"，而是"在既有资产工厂上打一个受约束的补丁"；
- **对 TA 的直接启发**：**"可编辑表面"这个概念可以直接搬进 Houdini / UE PCG**——你的 HDA 或 PCG Graph 里，哪些参数是"暴露给上层的合法编辑面"、哪些是"只读基础设施"，现在通常是靠约定；SCE 把它做成了**可计算的（AST 依赖闭包）**；
- **放置的五道闸门 = 一份现成的"放置验收清单"**：可见性 / 碰撞 / 间距 / 表面策略 / 投影面积——任何做程序化摆放的人都能直接对照；
- **池化的 N→K**：开放世界植被/道具的"变体预算"就是这条公式；
- **与 VFX 的交叉**：PCG 生成的世界是特效的承载容器——**InfiniDream 这类系统把"一个场景里有多少个半透明植被/多少盏自发光物件"变成了可调参数**，这直接决定运行时特效的底噪（参见 [[Procedural Content Generation]] 的"真实交叉点"段）。

## Unreal Engine Relevance

- **UE PCG Framework 是它最接近的落地形态**：`PCG Graph` 的"规则 + 点云 + 过滤器"结构，与本文的"语义计划 + 网格候选 + 五道闸门"几乎一一对应——**本文可以当作"给 PCG Graph 写自然语言前端"的方法论参考**；
- **可编辑表面的映射**：UE 里对应"哪些 Graph 参数暴露到 Instance / 哪些是子图内部实现"——目前靠命名约定，可以借 SCE 的依赖分析思路做工具化；
- **锚定七型 ↔ UE 的 `Surface Sampler` + `Projection` + 碰撞检测**：Hero 的 rooted/conform/afloat 等，在 PCG 里是 sampler + projection 的几种组合；
- **注意**：本文产出是 Blender 资产（.blend / 网格 + 材质），**不能直接进 UE**；可迁移的是**流程与判据**，不是资产。

## Technology Evolution

```text
"内容从哪来"的四十年：
1980s-2000s  人写规则          （文法 / L-system / CGA）
2010s        人搭节点图        （Houdini / UE PCG）—— 规则的图形化
2023-2025    LLM 写规则        （ProxyBuild / Co-DSL / Scene-Designer）
2026         LLM 改规则        （InfiniDream）—— 在既有库上做受约束补丁
  ↓
下一步（本文未做，但方向清楚）：LLM 审计/维护规则库（谁保证 4 万行不出错？）
```

## Relationships

### Based On

- [[Procedural Content Generation]] —— 全部前提（规则化内容的可编辑性、种子多样性）；
- [[Müller — Procedural Modeling of Buildings (2006)]] —— "规则编写费力"这个问题定义，20 年后仍是本文的动机；
- Infinigen（Raistrick et al.，未入库）—— 4 万行程序化资产工厂，本文的底座。

### Extends

- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] —— 同样"不写规则"，但 ProxyBuild 靠**推断 + 检索**组装，本文靠**改代码**；两者是"资产化 vs 模型化"的又一对；
- [[2026-10-13-Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation|Co-DSL]] —— Co-DSL 让 LLM 与人生成"空间概念库"再交给求解器；本文让 LLM 编辑"程序化代码库"再交给确定性放置——**同一个"决策/构造分离"母题，两种资产形态**。

### Related

- [[2026-10-16-Scene-Designer — Learning Design Intelligence for Agentic 3D Scene Generation|Scene-Designer]] —— 那边学"设计 + 评判"（模型化路线），这边编辑"工厂代码"（资产化路线）；
- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction|Mira-Scene]] —— "结构锚定"母题的邻居：那边锚在像素↔规范坐标，这边锚在**相机视锥网格**；
- [[2026-10-14-Q-MAGIC — Quad-only Mesh Autoregressive Generation|Q-MAGIC]] —— 网格生成 vs 程序化生成：前者出"一件资产"，后者出"一族资产 + 参数"，可编辑性差一个量级。

### Followed By

- 观察项：这套"符号约束编辑"是否被移植到 Houdini HDA / UE PCG Graph（那才是游戏生产真正会用的形态）；
- 观察项：放置的"视锥优先"是否会改成"导航可达性优先"（游戏关卡的真需求）。

## Personal Knowledge State

- `user_level: Normal`（**结论层 ≈20 分钟**）——取三条即可：
  1. **"改规则"比"写规则"现实**：LLM 的正确用法是**在既有程序化库上打受约束的补丁**；
  2. **自由度要可计算**：AST 深度 2 的依赖闭包 = "哪些能改"的机械定义，比"给不给权限"精确；
  3. **放置 = 搜索 + 闸门，不是预测**：五道闸门（可见/碰撞/间距/表面/投影面积）+ 由近到远，是可复现的工程解。
- **需要警惕的一条**：它的放置是**为镜头服务**的（相机视锥网格），不是为玩法服务的——把它当"关卡生成"读会读错。

## Learning Value

- **与你现有工具的连接**：如果你用 UE PCG，本文的"五道闸门"可以直接写成 PCG Graph 的五个 filter 节点序列——**这是本文最可立即复用的部分**；
- **与你预算工作的连接**："池化 N→K"是资产预算的公式化表达；"投影面积带"是**屏幕占比约束**的解析形式——与你 VFX 预算里"屏占比 × 密度"是同一族语言；
- **不建议**：复现其 Infinigen 栈（依赖重、且产出不进引擎）。

## Visualization

![[Research/Files/Pictures/Huang 2026 — 图5 InfiniDream 端到端世界生成管线.png]]
**读图**：A 场景选择（70 个预生成环境：14 biome × 5 seed）→ B Director 产出结构化 Scene Plan（add/replace/delete + 工厂 + 规格 + 实例数 + 原型数 + 世界运动；右侧是 Hero 七锚定 / Scattering 密度 / Swarm 群规模的语义槽位）→ C 资产生成（SCE）→ **D 放置：相机视锥离散成网格，绿=通过、蓝=表面类型不符、红=坡度过陡、黄=填满画面、灰=没打到东西**——五道闸门的颜色就是它的失败原因分类 → E 池化与种群化（seed 0..K → ×N 实例）→ F 地形细化与渲染。**D 那一格是全文最值得抄的工程图**：把"该放哪"变成"哪些格子被否掉、为什么"。（图 5，Huang et al. 2026）

> 视觉材料为论文原图（图 3 / 图 5），版权归原作者，仅作学习引用。

## Notes

- **为什么给 A- 而不是 A**：架构上的"决策/构造分离"是干净的，但**每一半都是已有思想的组合**（AST 约束编辑、网格 + 闸门放置、asset pooling），且放置的视锥假设把它锁在"合成数据/单帧构图"场景；对游戏的价值在方法论，不在系统；
- **与昨日 [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]] 的共同点**：两篇都在"把约束焊进表示/流程"——一个焊进网络输出层（ONI），一个焊进代码依赖图（AST 闭包）。这是 2026 年很稳定的一条方法论线；
- **一处存疑**：Table 2 场景级的 DINOv3（0.361）明显低于物体级（0.554），作者未解释；从任务性质看合理（场景包含大量非主体元素），但如果要引用这两组数，不要混着比。
