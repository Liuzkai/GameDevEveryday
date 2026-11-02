---
type: learning-path
title: "Learning Path — 灯光系统迁移与参数拟合"
target: "[[Lighting Parameter Fitting]]"
status: Collecting
created: 2026-10-30
collection_window: "2026-11-02 ~ 2026-11-06（五个工作日，之后停止自动收集）"
tags: [lighting, inverse-rendering, relighting, calibration, tooling]
---

# Learning Path — 灯光系统迁移与参数拟合

## 为什么有这一页

**实际需求（读者自述，2026-10-30）**：正在做**旧灯光系统 → 新灯光系统**的迁移。新系统接受底层的修改，但为了把旧场景的观感接过去，需要**对灯光参数做拟合**——想做一个**自动拟合工具**。因此需要收集同类方案。

**这一页是一个"收集点"，不是一篇综述。** 2026-11-02 至 11-06 每天往里加当天的检索结果，之后停止自动收集；需要时再手工续。

---

## 把问题拆成六个槽位

做拟合工具时，真正要分别决策的是六件事。**后面收集到的每篇论文，都应当能明确落到其中一个（或几个）槽位上**——落不进去的，多半只是"看起来相关"。

| # | 槽位 | 要回答的问题 | 现状 |
|---|---|---|---|
| **① 目标** | 拿什么当"正确答案" | 旧管线的 baked lightmap？旧版引擎渲的一组图？美术的一张参考？HDR 探针？**这些目标的"可信频段"完全不同**（lightmap 只有低频漫反射，参考图带材质与色调映射） | 已有：[[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization\|Lipp 2024]]（曲面辐射场 + HSH 投影） |
| **② 参数化** | 允许调哪些量、各自的定义域 | 位置 / 朝向 / 强度 / 色温 / 色度 / 锥角 / IES / 面光尺寸。**每一项有没有"物理或工程上的硬约束"**（不能悬空、亮度有档位、型号库有限） | 已有：Lipp 2024（位置、Rodrigues 朝向、$p^2$ 保非负强度、归一化色度、IES/软边锥） |
| **③ 梯度** | 怎么拿到"往哪调"的方向 | 有限差分 / 自动微分 / 解析伴随 / 无梯度搜索。**关键坑：AD 在某些形式化下系统性失效**（Lipp 证明光照追踪就是这样） | 已有：Lipp 2024（解析伴随；FD 验证；AD 失效的机制解释） |
| **④ 优化器** | 用谁来走这一步 | GD / Adam / L-BFGS / CMA-ES / 混合整数（选灯个数）。**参数数量级与耦合结构（如"高度 ↔ 强度"）决定选型** | 已有：Lipp 2024（L-BFGS 42–45 次评估；CMA-ES 需 5–10 倍） |
| **⑤ 评估** | 怎么判断"迁移成功" | 参数空间距离？图像空间 $L_2$ / SSIM？感知指标？**以及最容易被忽略的一条：优化目标与验收指标不是同一个东西时会发生什么** | 已有：[[Giroux 2023 — Towards a Perceptual Evaluation Framework for Lighting Estimation\|Giroux 2023]]（49 人 2AFC 实验：标准 IQA 指标在高光/无参照任务上≈随机；SVR 学习组合到 0.889；accuracy 与 plausibility 要分开验收） |
| **⑥ 落地约束** | 优化结果怎么变成"能用的灯" | 贴天花板 / 吸附网格 / 量化到型号库 / 数量上限 / 与美术工作流对接 | 已有（部分）：[[Schwengerer 2022 — Inverse Method for Baked Lighting\|Schwengerer 2022]]（均匀网格初始灯阵 + "方案=初始配置×阶段序列"建模 + 方案排期实测）；记名：DIALux 自动布灯（型号库+规范约束）、Sony Spherelight、ILM LightCraft、TrueHDRI（见清单） |

**一个先验判断**：①③④ 三格已经有相当成熟的文献（可微渲染 + 逆问题），**真正难的是 ⑤ 和 ⑥**——因为这两格不是学术问题，是工程与审美问题，论文通常不写。**收集时要专门留意那些"把结果真的用起来了"的工业界/产品侧材料。**

---

## 收集清单

| 日期 | 论文 / 材料 | 落在哪个槽位 | 一句话价值 | 与本页的关系 |
|---|---|---|---|---|
| 2026-10-30 | [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization\|Lipp et al. 2024, View-Independent Adjoint Light Tracing]] | ①②③④ | 目标搬到 3D 辐射场（无相机）、解析伴随、§6.3.4 直接就是"从 lightmap 反推灯"（OpenArena，≤972 参数，42 s） | **起点**。它证明了这件事可行，也暴露了 ⑤⑥ 两格没做 |
| 2026-11-02 | [[Schwengerer 2022 — Inverse Method for Baked Lighting\|Schwengerer 2022, Inverse Method for Baked Lighting（TU Wien 学士论文）]] | ①②③④⑥ | Lipp 框架的前传+工程版：Q3A `.bsp` 解析 → lightmap→顶点辐射场投影公式 → 均匀网格初始灯阵 → 5 种优化排期实测（**先 L-BFGS 后 Adam 最优；分子集调反而最差**） | 补上了 Lipp 没写的"真做一遍"细节：初始灯阵策略与方案排期；明确"初始配置与优化器共同决定质量" |
| 2026-11-02 | [[Giroux 2023 — Towards a Perceptual Evaluation Framework for Lighting Estimation\|Giroux et al. 2023, Perceptual Evaluation Framework for Lighting Estimation]] | ⑤ | 49 人受控 2AFC 实验审计 15 个 IQA 指标：**高光材质/无参照任务上≈随机水平**；SVR 学习组合达 0.889 且可泛化；数据与代码开源 | **补上 ⑤ 号缺口**。直接回答"怎么判断像不像"：accuracy/plausibility 分开验收，指标可训练 |
| 2026-11-02 | Shlyaev 2025, Machine Learning Meets Lighting: Using Depth Estimation To Build The Light Rigs（SIGGRAPH Talks '25，Sony Imageworks）（记名未建页） | ②⑥ | 影视工业界 HDRI→灯阵管线：OpenCV 检测光源 + PatchFusion 单目深度定灯距 + Kabsch-Umeyama 对齐 Lidar；灯阵搭建"几小时 → 约一分钟" | ⑥ 的工业样本：HDRI 是影视侧的"旧数据"，面光 rig 是"新表示"；与游戏 lightmap 路线互补 |
| 2026-11-02 | Suter et al. 2016, LightCraft（ILM，SIGGRAPH 2016 Talk）（记名未建页） | ②⑥ | 双高度拍摄的 HDRI 对 + 人工标记灯角点 → Mars 摄影测量解算灯位 → 自动抽 emission map | 同上线索的更早形态（半自动）；说明"HDRI→显式灯"在影视管线已成熟十年 |
| 2026-11-02 | DIALux evo 官方功能文档：Automatic Luminaire Placement / 灯具目录 / 照度校核（记名未建页） | ⑥ | 建筑照明工业工具的落地约束全集：真实型号库（IES/LDT 配光文件）、阵列/线性布置、规范照度目标、红绿信号灯式达标显示 | ⑥ 的"产品侧答案"：优化结果必须量化到型号库 + 满足规范指标才算"能用的灯" |
| 2026-11-02 | TrueHDRI Panel（UE 插件，Bandai Namco × 东京工科大学，Fab 商店）（记名未建页） | ⑥ | UE 内一键 HDRI→IBL 环境蓝图；可把最亮区域分离成 Directional Light 供 Lumen/常规实时渲染用 | 证明"环境光→显式灯"在游戏引擎侧已有产品化先例，可直接试用参考其参数拆分策略 |
| （待填） | | | | |

**本日检索备注（2026-11-02）**：检索通道（WebSearch）正常，无失败记录。Schwengerer 论文 PDF 无 ToUnicode 文本层（提取为乱码），已改用逐页渲染+图像识读核读，方法/评估/结论章数字均已亲见；Ch.4 BSP 实现细节未逐字核。Giroux 论文 venue 是否正式收录未核（arXiv v3 为止）。另见 2026-09 的一篇 Ad-Relight（arXiv 2609.37951，广告图重打光）用到 GPT-4o 做光照一致性评审 + ILL-SIM 亮度场余弦相似度——图像到图像路线、不产出灯参数，按选文标准**记名不收**，仅其"感知评审"思路与 Giroux 呼应。

---

## 选文标准（下周每天按这个筛）

**收**：

1. **目标定义在场景空间**（曲面/体/探针）而不是图像——这是"迁移"与"渲染比对"的分水岭；
2. **参数是灯光本身**（位置/朝向/强度/色度/IES/型号选择），而不是材质或几何；
3. **显式处理"旧数据 → 新表示"的转换**（lightmap → 灯、探针 → 灯、旧 GI → 新 GI、烘焙 → 实时光追）；
4. **工业界或产品侧的做法**（DIALux / Relux / 引擎的自动布光 / 影视的 light rig 匹配），哪怕没有论文；
5. **评估方法**本身（怎么判断"像不像"、感知指标与物理指标的分歧）。

**不收**（记名即可，不建页）：

- 只做"图像到图像"的风格/光照迁移，且不产出可解释的灯参数；
- 只是又一个可微渲染框架，但优化对象是材质/几何；
- 纯实时 GI 的性能论文（与"拟合"无关）。

---

## 与已有页面的关系

- [[Differentiable Rendering]] —— 上游。本页是它在"光源参数"这个维度上的应用侧；
- [[Inverse Rendering]] —— 同族，但那一页关心"从图反推场景"，本页关心"从旧观感反推灯"；
- [[LightOpt — Lights Optimization for Real-Time Rendering]] —— **实时侧的对应物**。做工具时可能两个都要：离线用伴随法拟合，运行时用实时法维持；
- [[Global Illumination]] —— 目标数据的来源（烘焙光照属于"预计算光传输"这一支）；
- [[Physically Based Rendering]] —— IES / 面光 / 色温这些"灯到底是什么"的定义；
- [[Mesh Parameterization]] —— 若目标要存成贴图域（而不是顶点域），会用到它的 atlas 与接缝结论。

---

## 工具化时绕不开的三个决定（先写下来，等材料齐了再复核）

1. **目标用顶点域还是贴图域？** 顶点域（Lipp 的做法）与网格分辨率绑定，密网格上自由度爆炸；贴图域能复用 lightmap 的现成 UV，但会引入 atlas 接缝（见 [[Mesh Parameterization]] 的"DC 误差可消、梯度误差不能"）。
2. **灯的个数是人定还是自动定？** Lipp 明确把"选灯"留给未来工作。如果你的迁移场景灯数很多（几十上百），可能得先做一遍**离散选择**（覆盖/聚类/贪心），再对选中的做连续拟合。
3. **拟合结果要不要"可解释"？** 972 个自由参数的拟合结果，美术改不动。**要么限制参数规模，要么输出"建议 + 置信度"让人来采纳**——这条会直接决定工具的形态（一键 vs 半自动）。

---

## 备注

- 本页建立于 2026-10-30，由读者当日的实际需求触发（不是研究 Agent 的自主选题）。
- 下周（11-02 ~ 11-06）的自动收集会把结果追加到「收集清单」表，并在每个槽位下补"现状"。
- **收集窗口结束后不再自动运行**，页面保留，成为后续手工查阅的索引。
