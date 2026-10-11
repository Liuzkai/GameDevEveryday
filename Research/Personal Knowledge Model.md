---
type: personal-knowledge-model
created: 2026-09-07
updated: 2026-10-11
status: calibrated-from-reading
reading_checkpoint: 2026-09-18
read_paper_count: 30
---

# Personal Knowledge Model

> **2026-10-05 已按实际阅读记录校正。** 用户自述已读到 9 月 18 日；本次核对论文 frontmatter，共 **27 篇标记为 `read`**。阅读记录是确定事实，概念掌握程度仍按明确自评与已有证据分层记录。
> `reading_checkpoint` 是用户的阅读位置，不是按论文发布时间划定的已读边界，也不是实际完成阅读的日期。部分较晚入库的论文同样已标 `read`，一并计入；较早但仍为 `unread` 的论文不自动改成已读。
>
> **2026-10-07 更新（阅读推进，逐文件核实）**：已读 **27 → 30 篇**——[[Karis — Real Shading in Unreal Engine 4 (2013)]] / [[2026-09-17-DSD — Diffusion Skill Discovery]] / [[2026-09-27-Plate-Local River Generation]] 三篇转 `read`。**Karis 读后 PBR 阅读侧全清**（唯一剩余动作 = furnace test）；**Marschner 深读进行中**（status 仍为 `unread`，不代改——用户自制 R/TT/TRT 光路 SVG、三处勘误、挂载原文 PDF）；用户新建学习白板 `Learning/Canvas/Notes.canvas`（Shading Model D/G/F + IBL/Split-Sum + Marschner 三线，含 EnvBRDF LUT 补图）。
> **2026-10-08 更新（仅资料侧；个人阅读状态无变化）**：毛发线第六节点补齐——[[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]]（WDAS 生产模型；自测 18 → **21 条**；六节点闭合 1989→2016）；**面部域开线**——新概念 [[Facial Animation]] + 首节点 [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]]（结论层 Normal 可读）；物理域新增经典侧地图（Multiphysics 教程，待按需查阅）。
> **2026-10-09 更新（读者侧产物记录 + 资料侧）**：**工作区新增自制五路线对比图解**（`pbr_multiple_scattering_comparison.html`，10-08 提交；含白炉测试交互计算器 + UE `ShadingEnergyConservation` 检查点）——**"先用自己的话比较五条路线"这个动作已出现实物产物**（不代改任何 `status`；引擎实测仍未记录）。资料侧：**路线⑤原始文献入库**（[[Turquin — Practical Multiple Scattering Compensation for Microfacet Models (2019)]]，五条补法全节点闭合）；面部域捕捉层首节点（[[2026-10-07-emg2face — Expressive Facial Animation with High-Density Surface EMG|emg2face]]）；NRC 镜面支线（NRC-Spec）。**已读账本维持 30 篇。**
> **2026-10-10 更新（资料侧 + 方法侧；阅读状态无变化）**：**几何建模域开域**——经典 [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]] + 新概念 [[Subdivision Surfaces]]（三条规则 + 四性质；与逆问题样本 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]] 构成"正↔逆"48 年弧线，**可选 45 分钟小闭环已列入 Daily 动作清单**）；物理域材料层 +1（[[2026-10-08-RiCo — Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning|RiCo]] 结论层；**"指标看不见的价值"判据**入工具箱）;可微桥 +1 软材料（[[2026-10-08-Sensitivity as an Arbitrary Output Variable for Differentiable Rendering|Sensitivity AOV]]，≈10 分钟、无数学）；GS 线几何质量维度（[[2026-10-08-PCAsplat — Gaussian Splatting with Local PCA Regularization|PCAsplat]]）。**已读账本维持 30 篇**；既有动作清单全部不变（furnace test / 毛发 21 条 / Neural GI 桥 / 动态灯光复审）。**方法侧新规律**：周五提交在周六的 API 中不可见（cs.GR/cs.CV 双验证）——周末运行以 listing 为主通道。
> **2026-10-11 更新（资料侧；阅读状态无变化）**：**48 年弧线中段补齐**——[[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)]]（**求值线**：递归 → 特征基直接求值，"推翻一个信念"；结论层 ≈25 分钟）+ [[DeRose — Subdivision Surfaces in Character Animation (1998)]]（**生产化线**：半锐折痕 / 布料三件套 / 标量场，结论层 ≈30 分钟——**库内对 TA 最友好的经典**）——"**规则 / 算得出来 / 用得起来**"三件套齐；[[Subdivision Surfaces]] 概念笔记的"Stam 1998 / DeRose 1998 记录位"兑现，Next Step 已加两篇（按需）。**W41 收官**（10-5~10-11：24 Papers 含 8 经典 / 3 新概念 / 8 图解 + 读者实物图解）。**已读账本维持 30 篇**；动作清单不变——**"动作月"中旬盘点已写入 [[2026-W41]]：建议从五件中至少执行一件（推荐动态灯光复审，材料 100% 封盘）**。

## 如何理解和更新这个模型

- **已读**：以论文的 `status: read`、`status: [read]` 或 YAML 列表中的 `read` 为准；不等于通过全部自测或完成实现。
- **已掌握（Easy）**：明确的 `user_level: Easy` 或已有掌握记录。单篇 Easy 只确认该篇覆盖范围，不自动扩展为整个领域 Easy。
- **学习区（Normal）**：能够沿现有知识继续学习；新增加的概念层判断会明确标为推断。
- **推导或实现缺口（Hard）**：保留具体缺口，但不再把已读的 Hard 论文描述成“尚未接触”或“不能读”。
- **资料已入库**只代表可用材料。未读、未自测、未实践分别记录，不用入库日期代替个人进度。

继续修改论文或概念笔记的 `status` / `user_level`，或直接告诉我掌握情况即可。**本次仅更新本模型；未批量改写论文或概念笔记的属性。** 下表中的分层推断不视为那些概念笔记已经完成升档。

## 工作背景与已有基础

沿用原有背景：资深游戏技术美术，工作涉及材质、Houdini 过场特效与 PCG；Houdini、Maya、Unreal Engine、Substance 为熟悉工具。PBR 材质制作是强项；渲染底层、角色/体积/水体渲染、游戏动画系统和 PCG 算法仍需按具体主题判断。

**工程经验与理论掌握分开记录**：会做预算、使用材质或工具，不足以证明已掌握全部底层渲染理论。

## 本次校正的主要变化

| 方向 | 实际证据 | 当前判断与推荐变化 |
|---|---|---|
| PBR / BRDF | Cook–Torrance 已读且 `user_level: Easy`；Kajiya、Walter、Schlick 已读 | Cook–Torrance 框架已确认 Easy；BRDF / PBR 整体维持 Normal，转向工程近似与能量问题，减少重复基础讲解 |
| 多次散射 | Kulla–Conty、Heitz、Fdez-Agüera、Dupuy 2026 均已读 | 已覆盖工程补偿、随机输运与闭式路线；概念比较属 Normal 学习区，推导与引擎验证尚无完成证据 |
| Gaussian Splatting | 原有 Easy 记录；Compact Neural Appearance、From Splats to Silicon、GradRig 已读 | 基础 Easy 沿用；阅读已延伸到外观压缩、系统瓶颈与可微蒙皮，后续从这些接口继续 |
| 动画检索与生成 | Learned Motion Matching、STyMo、FlexMoGen、UniMo、EMODY Flow、GestureFAR 已读 | Motion Matching 仍为 Normal，但不再作为阻止阅读所有神经动画的硬门槛；生成方法的表示、控制与延迟取舍可作为 Normal 入口（推断） |
| 神经 / 生成渲染 | DLSS 5、Magpie、PBR-Latent 已读，三篇笔记均为 Normal | 系统边界与工程取舍已有阅读基础；这部分按 Normal 入口组织（推断），训练与生成模型推导仍保留 Hard |
| 光传输与介质 | Kajiya、Gaussian Light Transport、GPIS、Grid-Free Monte Carlo 已读 | 已从表面反射延伸到光场表示、随机几何与按需求解；不把高维拟合、随机过程或 PDE 推导视为已掌握 |
| 粒子 / 阴影 | Reeves、Williams 已读且 Easy | 保留机制层 Easy；成本模型已读，项目内测量仍待验证 |

## 当前知识分级

### Easy — 已有基础，不重复基础教学

| 知识范围 | 证据与边界 |
|---|---|
| Cook–Torrance 微面反射框架 | **本次直接核实**：[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 为 `read` + `Easy`；不自动把整个 [[BRDF]] / [[Physically Based Rendering]] 升档 |
| [[Gaussian Splatting]] 基础 | 沿用 2026-09-11 的四项判据完成记录与概念笔记 Easy；不包含后来全部新变体 |
| [[Particle Systems]]、[[Shadow Mapping]] 机制 | [[Reeves — Particle Systems (1983)]]、[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] 均为 `read` + `Easy`；项目内成本测量另列 |
| PBR 材质制作与常用 DCC / 引擎工具 | 沿用背景记录，区别于 PBR 理论与引擎实现 |
| [[Real-Time VFX Performance Budgeting]]、[[Niagara]]、[[Scalability and Quality Tiers]] | 沿用既有工作背景分级；本次阅读未提供新的实践验收记录 |
| [[Real-Time Rendering]] 的工程应用层 | 沿用原有 Easy；不再用“能做预算”推断整个底层理论已掌握 |

### Normal — 当前主要学习区

| 知识 | 阅读基础与下一处缺口 |
|---|---|
| [[BRDF]]、[[Physically Based Rendering]]、[[Microfacet Theory]] | Cook–Torrance 框架已 Easy，Kajiya / Walter / Schlick 已读；[[Karis — Real Shading in Unreal Engine 4 (2013)]] 仍未读，工程链尚未完成 |
| [[Rendering Equation]]、[[Global Illumination]] | Kajiya 与 Gaussian Light Transport 已读；可继续比较采样与函数表示，不能由此确认 GI 缓存族已掌握 |
| [[Multiple Scattering and Energy Compensation]] | 四篇核心材料已读；工程补偿与精确输运的比较可以继续，Heitz / Dupuy 推导及实测结果未确认 |
| [[Split-Sum Approximation]] | 沿用 Normal；Fdez-Agüera 已读，Karis 未读，需补全其 IBL 近似与 LUT 假设 |
| [[Motion Matching]] | Learned Motion Matching 已读；可从检索、步进、解码的对应关系继续，自测未确认；原 40 分钟练习保持可选 |
| [[Neural Animation]]、[[Motion Generation]] 的方法概览与工程层 | **分层推断为 Normal 入口**：已有六篇动画方向阅读；不自动修改概念笔记的整体 Hard，也不确认扩散 / flow 训练推导已掌握 |
| [[Neural Rendering]]、[[Generative Rendering]] 的系统层 | **分层推断为 Normal 入口**：DLSS 5 / Magpie / PBR-Latent 已读；可比较规则与画面边界、表示选择、吞吐与延迟 |
| [[Participating Media]]、[[Linear Transport Theory]] 的概念层 | 沿用概念笔记 Normal，新增 GPIS / Dupuy / Grid-Free 阅读覆盖；随机几何与输运推导另列 Hard |
| [[Procedural Content Generation]] | 沿用 Normal，ESG 已读；不能据此认定城市 / 建筑形状文法经典已读 |
| [[Tile-Based Rendering]]、[[GPU-Driven Rendering]] | 沿用 Normal；From Splats to Silicon 提供性能分析阅读依据，不等于全部管线知识已掌握 |
| [[Neural Upscaling and Frame Generation]]、[[Temporal Stability and Artistic Intent]] | 沿用 Normal；DLSS 5 / Magpie 可作为已读对照，项目内效果与延迟尚未验证 |
| [[Hair Rendering]] | 沿用角色渲染基础与待补状态；HairCS / Marschner 仍未读，毛发 15 条判据不能计为通过 |

### Hard — 已接触，但推导或实现仍需桥接

| 知识 | 已有入口 | 未确认的部分 |
|---|---|---|
| [[Differentiable Rendering]]、[[Inverse Rendering]] | GradRig、PBR-Latent 已读 | 自动微分、梯度估计、可微光栅化及优化实现；不能直接升为 Normal |
| 神经 / 生成方法的训练层 | 渲染与动画方向已有多篇 Normal 材料 | 网络训练、扩散 / flow matching / 蒸馏推导与实现 |
| [[Neural Global Illumination]] | Kajiya + Gaussian Light Transport 已读 | 神经缓存训练与更新；RSM / Instant Radiosity 仍未读；Gaussian Light Transport 是非神经对照，不等于神经 GI 已掌握；**（10-09 资料侧新增：NRC 镜面支线 NRC-Spec——"旧概念的新组合"，按需 15 分钟）** |
| 微面多次散射与随机几何的推导层 | Heitz、Dupuy、GPIS 已读 | 随机输运、自由程 / 相位函数、水平穿越统计与闭式推导 |
| 瞬态扩散 / Monte Carlo 数学 | Grid-Free Monte Carlo 已读 | 热核、exit time 采样、PDE 与方差分析 |
| [[Physics-based Character Animation]] 的 RL / 生物力学层 | Athletic Sprinting 已读 | 奖励、控制策略训练与肌肉仿真实现；DSD / LYRIC 尚未读 |
| [[Neural Physics Simulation]] | 粒子机制 Easy，ESG 提供场景与物理约束入口 | attention、学习式仿真与训练；不再断言“只缺一个 attention 词汇” |

## 下一步：按实际阅读缺口衔接

1. **PBR 阅读侧已完成（2026-10-07 更新）**：[[Karis — Real Shading in Unreal Engine 4 (2013)]] 已转 `read`（含 EnvBRDF LUT 补图）——五篇经典 + Karis 的全部阅读材料**清账**。**唯一剩余动作 = furnace test 两项检查**（对照基线 = d'Eon 2022 手册 13.7.1/13.8.1）；完成后 [[Physically Based Rendering]] / [[Split-Sum Approximation]] 的阅读侧判据即全部满足。
2. **多次散射由阅读转向验证（2026-10-09 更新）**：四篇已读不必再按"首次介绍"推送；**"用自己的话比较 Kulla–Conty / Heitz / Fdez-Agüera / Dupuy"已出实物产物**——工作区自制五路线对比图解（10-08 提交，含白炉测试交互与 UE 检查点），且 **10-09 补入路线⑤独立节点**（Turquin 2019：F_ms 四级简化链、非互易边界、"gain to the closure"形态——补齐"比较"所需的最后一块一手材料）。**剩余动作 = 按所测 BSDF 的条件设计 white furnace test（对照 d'Eon 2022 手册 13.7.1/13.8.1），记录能量守恒与补偿开关差异**；普通环境光下的粗糙度对比可作观察，不能替代受控测试。
3. **动画线允许继续前进**：以 LMM 为检索 / 学习基线，对照 STyMo / FlexMoGen 的风格控制、UniMo 的表示、EMODY / GestureFAR 的音频驱动与流式延迟。Motion Matching 自测作为补缺工具，不要求完成后才读所有生成动画。
4. **Hard 按具体问题建桥**：可微方向从已读 GradRig / PBR-Latent 引出“优化什么参数、损失是什么、梯度从哪里来”，再用 [[Learning Path — Differentiable Rendering]] 补缺；神经方向沿 [[Learning Path — Neural Rendering]] 补训练侧概念。
5. **毛发线已启动（2026-10-07 更新）**：[[Marschner — Light Scattering from Human Hair Fibers (2003)]] **深读进行中**（自制光路 SVG + 三处勘误 + 原文 PDF 挂载；status 未改）；配套材料已补 [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]]（能量页，Weta 生产模型）。**（2026-10-08 更新）生产化页补入**：[[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]]（WDAS；near-field 取消求积 + logistic + 第四叶）——**自测 18 → 21 条**，六节点闭合（1989→2016）；阅读顺序不变（Marschner → d'Eon 2011 → [可选] Chiang 2016 → 21 条自测）。仍为未读：[[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]]、[[2026-09-18-LYRIC — Language-Driven Physics-Based Character Control]]；[[2026-09-17-DSD — Diffusion Skill Discovery]] ✅ 已读。
6. **面部线开线（2026-10-08，资料侧）**：[[Facial Animation]] 概念建立 + [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]] 入库——**按需阅读（结论层 ≈20 分钟），不设硬动作**；blendshape 基础列为未来按需材料。

后续推荐避免重讲已读论文；只有用于对照、回答具体疑问或支撑实践时才回引。优先延续 PBR / 微面理论主线，同时保留动画与神经渲染的工程入口。此处优先级来自本次阅读分布的推断，不代表用户已经承诺执行这些练习。

## 掌握判据与实践状态

| 范围 | 当前记录 | 如何更新 |
|---|---|---|
| Gaussian Splatting 基础四项 | 沿用 9 月 11 日完成记录 | 不要求重复验收 |
| Cook–Torrance | 明确 Easy | 不再以“用户未校正”处理 |
| PBR / BRDF 五篇、共 25 条自测清单 | **五篇全部已读（Karis 2026-10-07 转 read）**；无全部通过记录 | 完成相应自测后再判断整体是否 Easy |
| 多次散射 | 四篇已读 + 路线⑤独立节点补入（**10-09**）；**"用自己的话比较"已出实物产物（自制五路线图解，10-08）**；推导与实践未确认 | 概念比较、数学理解、引擎实测分开记录；一次观感测试不代表掌握整个领域 |
| Motion Matching 六项 | LMM 已读；判据完成情况未知 | 可按实际疑问检查，不自动增加必做任务 |
| Hair Rendering 二十一项 | Marschner 深读中；d'Eon 2011 能量页 + Chiang 2016 生产化页入材料（2026-10-08） | 从进行中的 Marschner 继续，后续按需补充 |
| 可微渲染四项 / 神经渲染五项 | 已有应用侧阅读入口；判据通过情况未知 | 沿对应 Learning Path 逐项确认 |
| 阴影占比 / 屏占比粒子密度 / 动态灯光分档复审 | 旧模型提出过，未见执行结果 | 保留为可选实践；不得从资料入库推断已执行 |

## 已读论文账本（30 篇）

以下清单逐项来自 frontmatter 核对（27 篇 = 2026-10-05 校正；**+3 篇 = 2026-10-07 更新**）。**不推定阅读完成日期**；保留各篇自己的难度分层。以后有新标记时，以论文文件当前属性为准并刷新本表。

| 已读论文 | 当前笔记的 user_level |
|---|---|
| [[2026-09-14-Gaussian Light Transport]] | Hard |
| [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]] | Hard |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] | Hard |
| [[2026-09-15-UniMo — Unifying Human and Animal Motion Generation]] | Normal |
| [[2026-09-16-ESG — Generating Physically Consistent Dynamic 3D Scenes from Text]] | Normal |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] | Hard |
| [[2026-09-17-EMODY Flow — Emotion-Aware Audio-Driven Full-Body Motion Generation]] | Normal |
| [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space]] | Normal |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] | Normal（概念层）+ Hard（推导层） |
| [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]] | Normal |
| [[Compact Neural Appearance Models for Efficient Gaussian Splatting]] | Normal |
| [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] | Easy |
| [[DLSS 5 — Generative Neural Rendering]] | Normal |
| [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] | Normal |
| [[FlexMoGen — Flexible Motion Generation from Language and Style References]] | Normal（Motion Matching 侧）/ Hard（生成侧） |
| [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]] | Normal |
| [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]] | Normal |
| [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] | Hard |
| [[Kajiya — The Rendering Equation (1986)]] | Normal |
| [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] | Normal（研读中） |
| [[Learned Motion Matching (Holden 2020)]] | Normal → Easy 桥接材料 |
| [[Magpie — Real-Time World Renderer for Interactive Games]] | Normal |
| [[Reeves — Particle Systems (1983)]] | Easy |
| [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]] | Normal |
| [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] | Normal |
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] | Normal |
| [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] | Easy |
| [[Karis — Real Shading in Unreal Engine 4 (2013)]] ★ 10-07 更新 | Normal（含 EnvBRDF LUT 补图） |
| [[2026-09-17-DSD — Diffusion Skill Discovery]] ★ 10-07 更新 | Hard（可只取一个抽象） |
| [[2026-09-27-Plate-Local River Generation]] ★ 10-07 更新 | Normal |

## 本次替代的旧判断

- “用户未做任何校正 / 第 28 天”已失效：27 篇 `read` 与 Cook–Torrance 的 `Easy` 都是实际反馈。
- “PBR 工程侧 9-18 闭合 / PBR 读材料阶段全部结束”曾混淆资料整理与用户阅读：Karis、d'Eon、Hill、Hammon 当前仍未读。
- “Motion Matching 未 Easy，因此神经动画不可读”过于严格：实际已有多篇动画阅读，可按层次继续学习。
- “Hard = 无法阅读”不符合当前记录：应区分已接触结论、掌握概念、理解推导、能做实现。
- 不再把每日产业动态、文献补录或推荐动作自动计入个人知识。它们的原始背景保留在下方历史记录及对应 Daily / Weekly 中。

## 历史推荐与资料准备记录（非当前个人状态）


展开 9 月至 10 月 5 日的旧记录

> 下列文字保留用于追溯旧推荐，**不是当前知识判断或待办清单**。其中“闭环”“读材料结束”“用户未校正”“唯一剩余动作”等措辞已由上文校正；产业信息和性能数据本次未重新核验，不能当作本次确认的事实或已完成的实践。


1. **本文件仍为推断值**（自 2026-09-07 建立，用户未做任何校正）——**第 28 天**。当前不影响工作（推送已按推断值自动调权）。**本次（10-5）新增 1 篇前沿（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]）+ 1 篇经典（[[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]]——**PBR 经典队列自此清空**）**。任何一次校正都会立刻改变推送重心，**仍建议做一次**；
2. **PBR / BRDF 的升档开关**：25 条自测通过即可标 Easy（我会在你标了之后停止推基础内容，转向其上的新研究）。**其中 24 条是纸面自测，唯一一条实测题是引擎侧 furnace test（30 分钟）—— 且 9-20 起升级为"查两头"：粗糙端是否变暗 + 光滑白色电介质球的掠射边缘是否有一圈偏亮**；
3. [[Motion Matching]] 的 40 分钟 Action 已于 9-17 降级为静默项——**想恢复随时说一声**；
4. **动态灯光维度复审**（UE 5.8 MegaLights 转 Production 的影响）—— **🔴 2026-09-24：材料全部就绪，转为执行项**（官方一手核验完成，见第 14 条；[[2026-W38]] 第一优先级可直接执行）；
5. **🔴 9-20 新增（对你的分档工作直接相关）**：**《控制：共振》把路径追踪 + 全局光照做成了所有光追预设的公共底座** → **"画质档位 = 在同一管线上调参数"这个隐含前提需要重新审视**。建议在 S/A/B/C × 五档矩阵里显式区分"**换参数**"与"**换管线**"两类档位差异。详见 [[2026-09-20]] 产业信号 2。
6. **🔴 9-21 更新（比 9-20 更尖锐，两条）**：
   - **档位实为"多个正交子系统的组合"**：[[2026-09-21]] 产业信号 1 查到《控制：共振》的菜单是 `RT Preset` × `Direct Lighting` × `Indirect Lighting` × `Transparency` × 各自 `Denoising` × **`帧生成倍率 2x–6x / Dynamic`** 的笛卡尔积。**你的五档可能需要定义"这一档下哪些子系统被替换"，而不是"资源上限乘多少"。** 另：**RT Ultra 档需 DLSS 4.5 Ray Reconstruction + RTX Mega Geometry → 厂商绑定**，跨平台体系里最高档可能在移动端/AMD 端根本不存在；
   - **神经渲染的开销会破坏锁帧依赖的玩法逻辑**：《铁拳 8》锁 60 → 神经渲染后 45 → **判定延迟、连招变慢动作**。**帧时间预算不只关乎流畅度，还关乎玩法正确性**；尤其**高帧率档（120 fps）下帧预算减半而特效开销不随帧率等比下降 → 建议单独复查高帧率档的特效预算**。
7. **🔴 9-21 新增：两项 30 分钟实测（都可直接做，直接把经验值换成依据）**：
   - **① 阴影占比实验**：材质极简场景抓 GPU 剖析，看 **Shadow Depths / Shadow Projection vs Base Pass** 占比 → 验证 **"材质越简，阴影越接近 50%"**（1978 年即写明的规律）。**含义：低档画质砍灯的收益比高档更大。**
   - **② 每千像素粒子数**：把现有绝对上限（3000/1200/400/100）与 **"屏占比 × 密度"** 对照一次 —— 绝对上限是**统计的**，屏占比密度是**几何的**。若两者在最重镜头下差很多，说明上限没抓住真正的成本变量。
   - 材料：[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] · [[Reeves — Particle Systems (1983)]] · [[预算五维_1978-1983_源头图解]]
8. **⚠️ 9-21 硬约束（二手，待官方核实）**：二手来源称 **UE 5.8 MegaLights 不支持半透明物体 / 流体 / 云 / 发丝，也不支持前向渲染** → **特效打光不在 MegaLights 覆盖范围内，仍走 1978 的成本法则（每盏灯 +1× 场景）**。**"动态灯光变便宜"不能直接推到 VFX 侧。动手前请核实官方 release notes。**
9. **🟡 9-21 观察：库里首次出现"来自另一个客户端"的改动**（不是校正，但是engagement 信号）。运行开始时 `git push` 被拒（`fetch first`）—— 远程比本地多两个提交（`7ea6834 Sync`、`02ae272 Sync`，来自另一设备的 Obsidian Git / 同步客户端）。**唯一的实质内容改动是 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 的 frontmatter 被重排**（内联数组 → 块列表、去引号，`status: studying` → `status: [reading]`）。
   - **⚠️ 关键：这不是 `user_level` 变更** —— `user_level: Normal` **原样未动**。该重排是 **Obsidian 属性编辑器的格式副作用**，**不能据此认为用户校正了 PKM**。
   - **但有一条正向信号**：**Cook-Torrance 笔记被打开并保存过**，而该笔记正是 **PBR 收口清单里 25 条中的 5 条所在**（且含 9-19 之前你主动研读的那条线）。**与"用户正在走 PBR 自测清单"这个判断一致**，但仍属弱推断。
   - **已把此情形写入规则文件 §41 Rule 9**（远程分歧的判定与处理流程 + "frontmatter 重排 ≠ user_level 变更"的判别规则），下次运行改为**推送前先 fetch**。
10. **🔴 9-22 新增（借 CGA shape 的一条分档体检，30 分钟可做）**：CGA shape（[[Müller — Procedural Modeling of Buildings (2006)]]）用 `r` 后缀显式区分"**会缩放的相对值**"与"**不缩放的绝对值**"——原文明确：**建筑部件并非都等比缩放**。
    - **同构到你的五档**：**特效参数也并非都随档位等比缩放**。建议给预算矩阵里每个参数标一个"**绝对 / 相对**"属性（如贴图尺寸、动态灯数偏绝对；时长、粒子密度可相对）。**凡是"整体乘个比例"的缩放方案，先检查这一项。**
    - 另借一条成本原则（同日）：**"离线生成量与运行时负载是两本账"** —— 与"成本不会消失只会转移"同构（PCG 生成端多省/多花，运行时另算）。
11. **（观察项）UE6 时间线**：2026-05 公布，**首作《火箭联盟》UE6 版已进入职业测试、2027 上线**（9-21 官宣）。含义：NGR 一类的 UE5 项目大概率全周期在 UE5 线上，但**引擎换代节奏在加快**（UE5 公布→首作约 3 年；UE6 约 1.5 年）。生产管线规划值得记一笔。
12. **🔴 9-23 新增（实测题升级为"有对照物"）**：**furnace test 的对照基线已就位** —— [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]] 的 13.7.1（Beckmann）/ 13.8.1（GGX）给出**球面 albedo 的拟合闭式**（对 η、α）与大量 benchmark 值。**此前这项实测只能"凭感觉看变暗/亮"，现在可以"实测值 vs 手册值"对照**。做法见 [[多次散射_五条补法路线与实时落地图解]] 第 5 节；先查手册、再跑引擎、后比数值。
13. **9-23 新增两条跨域借条**（供分档体检用，均为"廉价可做"类）：
    - **"记忆/状态按查询压缩"**（[[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory|WorldCrafter]]）：不存全部、不物化——**预算按"谁来看/怎么用"分配**。与你的"最重镜头资源分配"思考同构；
    - **"有界 vs 无界"体检（比 9-22 的"绝对/相对"更一般）**：[[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction|Mira-Scene]] 消融证明"稠密化不够、有界才是关键"（0.379→0.727）。**凡是让模型/参数自由回归的量，先检查能不能改成"有限区间内的对应"。**
14. **🔴 9-24 一手核验：UE 5.8 MegaLights 官方口径落定（复审材料全部就绪，可直接执行）**：
    - **官方事实**（UE 5.8 MegaLights 文档 + release notes）：Production Ready；**开销恒定**（"无阴影与有阴影光源差别不大"）；**与前向渲染器不兼容**（通用限制）；**不支持移动端 / Switch / 上代主机（PS4/XB1）**；`r.MegaLights.Allow 0` **可按 Scalability Level / Device Profile 禁用**（官方档位开关）；
    - **对五档预算的三点重述**：① **PC 两档**：动态灯光维度与"灯数"脱钩 → "≤3/≤2/≤1/0"的含义从"每盏灯成本"改为"**光照复杂度预算**（同像素重要光源数 + 降噪质量）"；② **Android 三档硬性无缘**（MegaLights 不支持移动端 → 仍走 1978 每灯 +1× 账；前向渲染不兼容再叠一层）；③ **Niagara 粒子光源已被支持**（逐发射器 "Allow Mega Lights" + "Cast Shadows"）→ **特效打光进入覆盖范围（更正 9-21 二手记录）**，但需实测（稀疏性/投影数建议为软性约束）；
    - 详见 [[2026-09-24]] 产业信号 1 与 [[Scalability and Quality Tiers]]；
15. **9-24 方法论库存 +1（"取消式优化"）**：[[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 证明**"排序可以被取消，而不是优化"**（成本函数置换：∝ 场景复杂度 → 每像素 + 固定项）。**体检问法**：面对任一瓶颈，先问"**它能不能不存在**"，再问"它能不能变便宜"；
16. **9-24 补录（PartLLM，供资产流程参考）**：[[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation|PartLLM]]（腾讯 Visvise / SIGGRAPH Asia 2026）把分件做成"**意图条件的粒度接口**"——同一资产可按不同下游需求生成不同粒度分解。观察项：Visvise 是否把该能力产品化（部件级批量材质 / 碰撞 / LOD 预处理）。
17. **🔴 9-25 新增（毛发收口 + 两条新库存方法论）**：
    - **毛发着色收口窗口已打开**：[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] 入库 → 经验侧 + 物理侧两头齐备，**9 条自测（Marschner 5 + Kajiya-Kay 4）通过即可标 Easy**。**同时注意：它是"预算与几何解耦"的最早正式表述**（"渲染时间与其代表的几何复杂度无关"）——你的五维预算体系可以直接引用它作为原则溯源；
    - **"取消式优化"第 2 例（GPU 工程版）**：[[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition|CuACD]] 把凸分解的 kernel 边界与 host 同步**消掉而非优化**（~80–100×）。**体检问法扩展**：不只在算法层问"它能不能不存在"，在**执行层**（kernel 边界 / host 回环 / 数据搬运）同样适用；
    - **"监督信号三原则"（OREO 消融表，评估任何 AI 工具时直接可用）**：① 监督必须跟着学生走（on-policy）；② 显式伪目标 > 隐式 score 梯度（**DMD"作加速器行、作老师不行"**）；③ 质量提升与结构漂移必须解耦（Mask IoU 分离器）。**且三个降级变体全部差于"不训练"基线**——看见 AI 工具宣传先找"降级消融"；
    - **E-Day 规格给 MegaLights 门槛一个"中端"锚点**（RTX 3060 Ti = 1440p High）：动态灯光维度复审的最后一块拼图（**10-6 正式版实测为最终检验**）。
18. **🔴 9-26 新增（W39 收官：毛发线三节点闭合 + 三条新判据）**：
    - **[[Hair Rendering]] 的"实时化工程"缺口闭合**（[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]]）：**texel（1989）→ 物理 lobe（2003）→ 实时工程（2004）三节点齐**，Mastery 自测扩至 **12 条**（+3：两 lobe 移位机制 / 排序前提假设 / early-Z 权衡）——**通过即可把"毛发着色（含实时工程）"标 Easy**；
    - **"取消式优化"追出 2004 年版本**：Scheuermann 用"**预处理静态索引缓冲 + 四趟渲染**"替代"运行时 CPU 空间排序"（原文 *"Instead of executing a spatial sorting step on the CPU at run-time…"*）——**与 2026 的两个样本（GS 排序取消 / host 同步取消）成链，跨 22 年**；
    - **两条新判据**：① **"加一步前置工作，先问它解锁了什么"**（prime-Z 第①趟是为了避免 alpha-test 禁用 early-Z——"多一趟被随后三趟的 early-Z 收益盖过"）；② **"前提假设必须写在明面上"**（静态排序成立的前提是发片间相对运动足够小；不成立则回退 CPU 排序）——与 Karis 的"可预存需要什么假设"同一条体检；
    - **方法侧**：双通道分工补全——"**早提交、晚公告**"条目（提交日在窗口外、公告日在窗口内）API 查不到，**只能靠 recent 页公告分组捕获**。
19. **🔴 9-27 新增（PCG 谱系日：三节点齐全 + 三条可迁移抽象）**：
    - **[[Procedural Content Generation]] 三节点齐全**：[[Parish-Müller — Procedural Modeling of Cities (2001)]]（城市）→ [[Müller — Procedural Modeling of Buildings (2006)]]（建筑）→ [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]（推断）——源头/中段/前沿首次全部落库；**2001 ↔ 2006 的"同作者接力"关系**（2006 兑现 2001 的 Future Work，并以 occlusion 查询回应其"体块互不感知"问题）；
    - **三条可迁移抽象（不需要读全文，可直接用于分档体检）**：① **规则只描述结构、参数由外部策略决定**（ideal successor）——判据："任何规则系统，先问哪些部分该从规则里移出去"；② **约束三档处置**：拒绝 / 修正 / **有条件接受 + 标记 + 后期替换**（高速跨水 → 桥或隧道）；③ **细节层次 = 迭代深度**（decreasing apices）——**与 ToCo-Mesh 的"固定拓扑 + 自适应细分"同源：降档应换表示的深度/密度，而不是硬缩参数**（对分档工作的又一次确认）；
    - **成本账本（2001 原始数据）**：13,000 栋建筑 ≈ 路网 10 秒 + 建筑约 10 分钟——"database amplification"（少量规则 → 大量数据）的另一面是**生成时间成本**；与 [[Müller — Procedural Modeling of Buildings (2006)]] 的"离线生成量 ≠ 运行时负载"合流；
    - **产业侧新样本（艺术意图维度）**：《巫师 3》重制版（9-29）发售前"**氛围 vs 准确**"画质争议（CDPR 确认原版不强制替换）→ 记为 [[Temporal Stability and Artistic Intent]] 的一手样本：**"更准确的光"与"记忆里的味道"不总是重合；旧观感保留 = 事实上的"艺术意图档位"**。
20. **🔴 9-28 新增（毛发多散射日 + 跨域同构升级 + 产业双信号）**：
    - **毛发自测扩至 15 条（窗口强化）**：[[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] 入库（**全局/局部分解 + "一条原型路径代表全部路径" + 三档实现**；7.8h → 5.2min → 14fps 且零调参）——**毛发着色谱系四节点（1989/2003/2004/2008）全闭合**；新增 3 条自测：单散射为何对浅色发不够 / 原型路径的成立条件 / 全局局部为何像"阴影"与"材质"。**9-29 巫 3 重制版实测可作实战观察（第 13 项）**；
    - **跨域同构升级（今天最值钱的一条）**：毛发"**全局输运 vs 局部响应**" ⟷ PBR"**推输运 vs 补能量**"——**同一分解哲学在两个尺度成立**（微面拆能量与分布、发束拆输运与响应）。**入库为可迁移判据：复杂问题先问"能不能拆成'怎么到达'与'到达后怎么响应'两半，各自选最便宜的技术"**；
    - **两条新判据**（9-28 入库，均已写入论文笔记）：① "**一条原型路径代表全部路径**"的合法性检查——先问"结果的哪些成分依赖路径几何、哪些只依赖事件质量"，只依赖后者的可整体丢给统计量；② "**方差可加**"——选对表示（高斯族）让"多次"不升复杂度（与 Karis"为可预存额外假设什么"同族，都是"选择即代价/收益"的样本）；
    - **产业双信号**：① **RTX Mega Geometry 2.0**（流式光追几何 + 优雅降级；**E-Day 10-6 升级为"动态灯光 × 几何流式"双首发检验**；另：≥10GB VRAM 门槛——**显存成为分档硬约束的第二例**）；② **《巫师 3》重制版评测今日解禁、9-29 上线**——**Steam/GOG 保留三个 build（Classic / Next-Gen / Remastered）**：**"艺术意图档位"从概念变为产品形态**（玩家可回退旧观感）；
    - **预计算存法谱系补第四种**（[[2026-09-25-DiffusionShadow — Diffusion-based Shadow Caching for Neural Volume Rendering|DiffusionShadow]]）：全存 → 解析表（Karis/Kulla-Conty）→ 复用（Fdez-Agüera）→ **生成式记忆**。判据：*面对"存不下"的预计算数据，先问"它是对所有输入都查、还是对密集采样集查"*。
21. **🔴 9-29 新增（溯源日：分档祖先 + 两条新判据 + 毛发进入实战数据期）**：
    - **"分档的祖先"落库**：[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]] 入库——**LOD / "可见复杂度固定上限" / working set / 递归下降**；本库五维预算自此有 **1976（分档原理）+ 1978（灯光/贴图成本）+ 1983（粒子成本）** 的完整上游链。**一条自查（可直接做）**：*我的档位在限制"资源总量"，还是在限制"单帧可见量"？* — 后者才是 Clark 的原始表述；**一条判据**：*成本 ∝ 物体空间复杂度是原罪，每次优化都是在把它改写成"成本 ∝ 可见复杂度"*（⚠️ 原文扫描件被 Wayback 429 拦截，三层核验 + 复核清单见笔记）；
    - **两条新判据（今日两篇前沿）**：① **"廉价参考解该当初始化还是当目标？"**（[[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]：看它的错误是否与正确信息缠在一起——复制动作/HumanIK 输出都只能当起点，目标由四能量定义；**"降级式优化"**：先问它该扮演什么角色）；② **"优化的是中间产物还是最终产物？"**（[[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]]："优化到眼睛"；**运行时状态（含功耗预算）作为生成条件**——"档位条件化"第三线索：`Dynamic` 闭环 + XeSS 3 倍率 + ControlGS）；
    - **毛发进入实战数据期**：**《巫师 3》重制版今日 18:00 上线**——LSS 路径追踪毛发首发（HairWorks 演进形态）；实测：RTX 5080 @4K PT+HairWorks+DLSS Perf = **30 fps 裸数**（帧生成 59）、**关 HairWorks 44**、关 PT 55 → **"毛发仍是帧预算重项"十一年未变；"换表示（LSS）降不了成本量级，但换来了观感"**。[[Hair Rendering]] 第 13 项"实战观察"现在可做；
    - **"帧生成被官方规格扶正"**：CDPR/NVIDIA 把 4K PT 的 60fps 目标**建立在 DLSS+帧生成之上**（"30fps 基数 + FG"的伪影问题被点名——**基数不能太低是 FG 使用硬前提**）。
22. **🔴 9-30 新增（月结日：可微渲染开线 + Clark 结清 + 月报/雷达 + 体检三问成型）**：
    - **可微渲染瓶颈线开线**（[[Differentiable Rendering]]）：[[Vicini — Path Replay Backpropagation (2021)]] 入库（**"常数内存 + 线性时间"起点**；**结论层 4 条可直接拿走**：重放三要素 / 内存与时间两笔账分开结算 / "有偏但方向对"被证伪（符号都可能错）/ "它能不能不存？"）+ [[2026-09-26-Constant-Memory Differentiable Light Tracing]] 入库（**光源侧补全**：1:1 vs 1:k 连接结构判据；压缩 vs 累积两策略）。**读法：只读结论层，推导层挂起**；桥的目标不变——读懂 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义；
    - **Clark 1976 补核结清**（9-29 挂账）：OSU 站内镜像一次成功（"大学站内镜像"路径第 5 次生效）；三项复核全结 + **四条新细节**（**中心加权细节 = foveation 先声 / 运动自适应细节（细节量 ∝ 1/速度）/ 排序从 m log₂m 降到 ≈pm / LOD 数据库构建管线：bottom-up pruning + top-down splitting + time/space 权衡**）；
    - **首月月报 [[2026-09-Monthly|Monthly 2026-09]] + [[2026-09]] 雷达刷新**：雷达新增 4 行（Hair Rendering / Arm Neural Graphics / Physics-based Character Animation / World Models for Games）；Neural Upscaling → **Adopt+**（预算副作用）；**10 月目标 = 把"资料就绪"转为"动作完成"**（furnace test / 毛发 15 条 / 动态灯光复审 / 两条 30 分钟实测）；
    - **体检三问成型**：**"不存？→ 不存在？→ 更便宜？"**（9-30 补上第一问：确定性系统 = 重放换存储）；
    - **库内质量修复**：链接完整性检查发现 **[[Real-Time Rendering]] / [[Global Illumination]] 两个基础锚点自首日起引用悬空、从未建文件**（Index / PKM / Learning Path 多处引用受影响）→ **今日补齐**；[[Global Illumination]] 内含**八代近似谱系表**（环境光 → SH → 辐射度 → PRT → 屏幕空间 → **缓存族 RSM/VPL** → 光追 → 神经），**[[Neural Global Illumination]] 桥的具名前置缺口由此有了载体**（该概念笔记的 Prerequisites 已同步）；另修复 3 处小悬空。全库其余 37 处悬空（多为历史日报名称不匹配）记入维护台账。
23. **🔴 10-1 新增（运动缝合日：动画线首个经典锚点 + 过渡三问两代答案 + GS 排序线第 5 节点）**：
    - **动画线第一个经典锚点落库**：[[Kovar — Motion Graphs (2002)]] 入库——图时代开山（相似度矩阵局部最小值 = 天然拼接点 / 固定约 1/3 s 混合窗 / SCC 剪枝 / 分支定界搜索）；**两条可迁移记忆点**：① **"观察频率决定验收严格度"的 2002 年表述**（走路阈值最严 vs 芭蕾可松——与你的分频逻辑同构）；② **离线重活 + 在线轻活**（O(F²) 建图可接受，因为它在离线侧）；**动画线从此有"第 0 章"**（此前最老一篇是 2020 的 LMM）；
    - **过渡三问两代答案表**（沉淀进 [[Motion Matching]]）：**在哪切**（相似度局部最小值 → 簇图最短路）/ **切多久**（固定 1/3 s → 图路径决定，0.3 s 仅下界）/ **怎么接**（混合 → 生成）——**"过渡长度从超参变成结构化估计量"**（[[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph]]，14.5 ms 单次缝合，60 FPS 帧预算内；脚相位不同 → 64 vs 80 帧自动补步）；
    - **"取消排序"线到第 5 节点**（沉淀进 [[Gaussian Splatting]]）：① 优化 → ② 顺序无关 → ③ 重设计基元 → ④ 随机光栅化全删 → **⑤ 随机透明混合路由**（[[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering]]：fragment/primitive 双流按成本分拣；**移动端 73.2 FPS@540p 含 NPU 重建**——Android 档位稀缺实测）；**"档位 = 采样数（1/4/16-spp）"**；
    - **"换空间"判据第 2 例**（[[2026-09-29-Texture Space Material Diffusion]]，NVIDIA / nvdiffrec 团队）：材质生成搬进纹理空间——"**已知投影当归纳偏置**（确定性变换进结构、不进损失）+ 2D 化换先验复用 + 训练规模 ≠ 使用规模"；输出 PBR 五通道直出 + 8K 超分；
    - **方法侧**：9-30 listing 于运行前放出（周三组 12 条，3 条与 Run #22 去重）；**API 未公告通道捕获 2 条**（双通道分工第 7 次验证）；"大学课程 / 作者组归档"PDF 路径第 6 次生效（Kovar）；
    - **产业侧**：**《战争机器：E-Day》早期访问今日开跑**（10-6 正式）——NVIDIA 官方博客：MegaLights ≤100 光源 + Lumen RT GI + Nanite 100× 细节 + **RTX Mega Geometry 首次光追 Nanite 几何** + DLSS SR/Dynamic MFG；**动态灯光复审的最终检验窗口开启**（首日实战数据待收）。
24. **🔴 10-2 新增（立面细节日：PCG 四节点齐全 + E-Day 实测到账）**：
    - **PCG 域四节点齐全**：[[Wonka — Instant Architecture (2003)]] 入库（**split grammar + 双重控制**——attribute matching 两级制 / control grammar；"**规则选择**"问题出处）；**2001→2003→2006 是同一批人的接力**（Müller/Wonka 交叉合著——本库最强谱系血缘）；"**拆出去**三步曲"沉淀（参数 → 设计想法+选择策略 → 表面）；**"选择策略独立化"判据**已入 [[Procedural Content Generation]]（四条可迁移抽象）；
    - **光学效应题域开线**：[[2026-09-30-Lens Flare Removal and Reconstruction]] 入库（Meta RL）——光晕"**两个账本一套表示**"（去光晕清理户 ↔ 相机空间 1D 高斯重建艺术家户）；**1D 几何约束消融 33.27 vs 27.77 dB**（"先验进结构不进损失"家族目前最干净的量化）；RTX 4090 92.7 FPS / 合批 +1.8%；
    - **🔴 E-Day 实测到账（动态灯光复审的最后一块数据）**：MegaLights 机制（**每像素固定光照采样预算**；"**数千**（DF）vs **≤100**（NVIDIA 博客）"两口径并列）；**Mega Geometry 成本账：12GB 硬门槛 / 1–2GB 显存 / ~7–10% 成本 / DXR 2.0 的 CBLAS**；**"最高档太阳阴影换回 VSM" = "换管线"实证 +1**；**主动放弃 Ray Reconstruction**（神经层适用边界样本）；**Advanced Shader Delivery 首发**（着色器编译卡顿消除）；5090 @4K Ludicrous 原生 28/34 → +DLSS 4.5 Q 49/56 → +MFG X4 ~200。**复审材料全部就位——建议直接执行**（详见 [[2026-10-02]] 产业信号 1 与 [[Scalability and Quality Tiers]] 10-2 续记）；
    - **动作清单（延续）**：PKM 校正（25 天）· 毛发 15 条自测 · furnace test 两项检查 · 两条 30 分钟实测（阴影占比 / 每千像素粒子数）。
25. **🔴 10-3 新增（缓存族双源日：Neural GI 桥材料层打通 + 两条前沿）**：
    - **缓存族双源入库**（[[Global Illumination]] 谱系表第 6 行自此可读）：[[Keller — Instant Radiosity (1997)]]（**VPL 起源**："光照场 = 一组点光源" + quasi-random walk；每盏 VPL 一遍渲染——成本 ∝ 灯数）+ [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]（**RSM**：四缓冲 shadow map + 每像素 ~400 固定样本 + 屏幕空间插值——**成本改写成 ∝ 像素 × 常数**）。**[[Neural Global Illumination]]（Hard）的"最小前置"材料层自此齐备**——读法：两张笔记的结论层 + [[间接光缓存族_RSM 2005 与 Instant Radiosity 1997 双源图解|双源图解]]（30 分钟级）；
    - **"固定样本预算"四连**（1978 每灯 +1× → 1997 每 VPL 一遍 → 2005 每像素 400 样本 → 2026 MegaLights 每像素采样预算）——**判据固化：任何光照方案先问"成本关于哪个变量线性"**；你的五档矩阵在 PC 档可同时持有两套语言（灯数上限 / 每像素预算）；
    - **"一个表示代表全部"家族三例同框**：IR 1997（一组点光源 = 光场）· Zinke 2008（一条原型路径 = 全部路径）· **GALA 2026（一套共享基 = 全部身份；神经→线性蒸馏，手机 60 fps）**——跨 29 年、"从光到人"；
    - **PCG 第 5 条抽象**（[[2026-09-27-Plate-Local River Generation]]）："**先定域、后算局部**"——确定性划分让"需要全局视野"的问题逐块独立（**懒加载的前提是确定性**；确定性 = 可丢弃重建权 = 多人一致性）；开放世界水系"四要求同时成立"的首个构造性答案；
    - **动作清单（延续）**：PKM 校正（26 天）· 毛发 15 条自测 · furnace test 两项检查 · 两条 30 分钟实测（阴影占比 / 每千像素粒子数）· 动态灯光复审（材料全齐，E-Day 10-6 为最终检验）。
26. **🔴 10-05 新增（漫反射侧结账日：Hammon 入库 + 预算语言第三轨）**：
    - **PBR 线的"读材料"阶段全部结束**：[[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] 入库（**原文阻塞 16 天**，经 GDC Vault 新 CDN `media.gdcvault.com` 解除）——**能量账本两册账并立**（镜面侧 Kulla-Conty 查表 / 漫反射侧 Hammon 线性项）；**PBR 经典队列自上而下全部清空**（Heitz→Fdez-Agüera→d'Eon→Hill→**Hammon** ✅）。剩余全部是**你的动作**（25 条自测 + furnace test 两项 + 30 分钟实测）；
    - **三条最有用的"一句话"**（Hammon 产出，均可直接背）：① **"specular 分母的 4 = (L+V=2H·V) 的平方"**（测度变换，不是面积归一化）；② **"UE 的 k=α/2 是 Smith G1 的 lerp 放松形态"**（你一直在用的东西的出处）；③ **"理想漫反射比纯 Lambert 大 5%（1.05/π）"**（Lambert 本身就不是正确的平滑极限）；
    - **漫反射侧的一句话检验**（[[Multiple Scattering and Energy Compensation]] §4c 新增）：*"进了表面的光还会带着 Fresnel 与弹射几何回来"*——与镜面侧的"被挡住 ≠ 被吸收"配对；**"补能量 vs 推输运"判据自此双侧成立**；
    - **预算语言第三轨落库**（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]，A-）：**误差 ∝ N⁻¹ᐟ²（Zador 定律）**——与你的"样本定预算 / 误差定预算"并立，**"预算定在曲线上哪个点"自此有了理论形态**；**对偶证书** = "预算的可认证性"范式（你给 SABC 找依据时可直接参考的形式）；
    - **两条对分档工作的体检（来自 Budgeted-GS，可直接做）**：① **"我的预算是'最陡区'还是'平坦区'？"**——砍一半资源若只损失一点质量，说明预算本来就在冗余区；② **"我的档位差异是'换表示'还是'缩参数'？"**——前者每 dB 成本通常更低（LOD vs 随机子采样同内存对照：-2.12 vs -7.08 dB）；
    - **读一切压缩/优化论文的新防御**："已发表压缩器前沿平坦（斜率均值 -0.04）：**增益来自冗余、不是容量**"——先问"报告的是冗余收益还是容量收益"（与 OREO 的"先找降级消融"并列）；
    - **动作清单（更新）**：PKM 校正（28 天）· 毛发 15 条自测 · **furnace test 两项检查（PBR 队列清空后，这是 PBR 线唯一剩下的动作）** · 两条 30 分钟实测 · **动态灯光复审（E-Day 正式版 10-6/10-7 发售——最终检验窗口就在眼前）**。
27. **🔴 10-06 新增（分工日：毛发仿真轴开线 + 动画压缩域开线 + GI 缓存族封顶）**——**以下为资料侧记录；个人进度以本文件上文（校正后）与各笔记当前属性为准**：
    - **毛发线分双轴**：[[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 入库（**仿真轴首节点**，独立于渲染侧 15 条自测）——神经时间积分器"镜像"经典积分器 I/O + 模拟器在环；3000 股 0.460 ms/帧；"表示即先验"消融（strand space 0.632 vs world space 0.342）；
    - **新域 [[Animation Compression]]**：[[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model|CurveCodec 2]]（0.37×/0.22× ACL；**"误差界即档位"** + 契约/回退/计数；页面含 ACL 作者邮件 = "三本账"一手资料）——`Normal` 为推断值；
    - **[[Neural Global Illumination]] 桥经典侧完整**：[[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)|DDGI 2019]] 入库——缓存族四节点齐（Ward / IR / RSM / **DDGI**）+ 三图解；**30 分钟读法已列**（= 该桥的实际阅读动作，仍待执行）；
    - **当日最值钱判据**："新旧工具各自接管哪一段"四样本（DDGI 光追补充光栅化 / Neuroll 镜像 I/O / CurveCodec 2 只学熵编码 / LoCoSplat 取消重型网络）；"误差契约"成为预算可认证性第三形态。


---

相关：[[Index]] · [[2026-09-18]] · [[Learning Path — Differentiable Rendering]] · [[Learning Path — Neural Rendering]]
