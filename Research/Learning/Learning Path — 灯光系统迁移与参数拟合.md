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
| **② 参数化** | 允许调哪些量、各自的定义域 | 位置 / 朝向 / 强度 / 色温 / 色度 / 锥角 / IES / 面光尺寸。**每一项有没有"物理或工程上的硬约束"**（不能悬空、亮度有档位、型号库有限） | 已有：Lipp 2024（位置、Rodrigues 朝向、$p^2$ 保非负强度、归一化色度、IES/软边锥）；[[Sloan 2008 — Stupid Spherical Harmonics Tricks\|Sloan 2008]]（SH→方向光+环境光的闭式参数化，误差定义在 irradiance 空间）；[[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy\|Gkaravelis 2016]]（全向点光：位置 + RGB flux，方向分布固定）；[[Debevec 2006 — A Median Cut Algorithm for Light Probe Sampling\|Debevec 2006]]（HDRI→点光：方向+颜色，无距离无尺寸，零优化确定性输出）；[[Cieri 2019 — Streamlining IBL Workflows with Computer Vision and USD\|Cieri 2019]]（面光：位置/尺寸/roll 角/纹理，USD LightCache 承载）；[[Lin 2013 — Interactive Lighting Design with Hierarchical Light Representation\|Lin 2013]]（点光：位置+RGB 强度；聚光：位置/方向/强度/锥角，**灯数由层级 cut 定**）；[[Gardner 2019 — Deep Parametric Indoor Lighting Estimation\|Gardner 2019]]（学习侧：固定 N 盏灯的方向/距离/角尺寸/颜色 + 环境项，多余灯强度归零） |
| **③ 梯度** | 怎么拿到"往哪调"的方向 | 有限差分 / 自动微分 / 解析伴随 / 无梯度搜索。**关键坑：AD 在某些形式化下系统性失效**（Lipp 证明光照追踪就是这样） | 已有：Lipp 2024（解析伴随；FD 验证；AD 失效的机制解释） |
| **④ 优化器** | 用谁来走这一步 | GD / Adam / L-BFGS / CMA-ES / 混合整数（选灯个数）。**参数数量级与耦合结构（如"高度 ↔ 强度"）决定选型** | 已有：Lipp 2024（L-BFGS 42–45 次评估；CMA-ES 需 5–10 倍）；[[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy\|Gkaravelis 2016]]（**离散选数的组合路线**：NNLS 初始化 + 互补性层级聚类 + 树上 optimize cut；灯数是 cut 出来的）；[[Lin 2013 — Interactive Lighting Design with Hierarchical Light Representation\|Lin 2013]]（同一思想的 2013 原型：NNLS 初始化 + 误差最小合并建树 + $Impro$ 拐点定灯数 + Nelder-Mead simplex 精调——无梯度时代的标准答案）；记名：MDPI IPSO（建筑侧混合整数 PSO，UDPR 取整）、Kawai 1993（高层目标的非线性无约束优化，谱系源头） |
| **⑤ 评估** | 怎么判断"迁移成功" | 参数空间距离？图像空间 $L_2$ / SSIM？感知指标？**以及最容易被忽略的一条：优化目标与验收指标不是同一个东西时会发生什么** | 已有：[[Giroux 2023 — Towards a Perceptual Evaluation Framework for Lighting Estimation\|Giroux 2023]]（49 人 2AFC 实验：标准 IQA 指标在高光/无参照任务上≈随机；SVR 学习组合到 0.889；accuracy 与 plausibility 要分开验收） |
| **⑥ 落地约束** | 优化结果怎么变成"能用的灯" | 贴天花板 / 吸附网格 / 量化到型号库 / 数量上限 / 与美术工作流对接 | 已有：[[Schwengerer 2022 — Inverse Method for Baked Lighting\|Schwengerer 2022]]（均匀网格初始灯阵 + "方案=初始配置×阶段序列"建模 + 方案排期实测）；[[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy\|Gkaravelis 2016]]（**灯贴表面约束的实现样本**：体素化+膨胀候选域，图 11）；[[Cieri 2019 — Streamlining IBL Workflows with Computer Vision and USD\|Cieri 2019]]（**灯阵资产化与工具形态**：USD LightCache 跨 DCC 交换 + 半自动检测+美术末端的交互形态，DCC 灯型差异清单）；记名：DIALux 自动布灯（型号库+规范约束）、Sony Spherelight、ILM LightCraft、TrueHDRI、HDRI Light Export for Nuke（见清单） |

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
| 2026-11-03 | [[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy\|Gkaravelis & Papaioannou 2016, Inverse Lighting Design using a Coverage Optimization Strategy（CGI 2016）]] | ①②④⑥ | **灯的数量、位置、flux 三者一起解**：细网格候选 + NNLS 初始化 + 贡献互补性层级聚类建树 + 树上找最优 cut 定灯数；8 场景全部以更少灯拿更低误差（Office 直接光 9 盏 1.36% vs 对手 11 盏 38.3%）；自带"灯贴表面"约束（体素化+膨胀） | **补上 Lipp 明言留给未来的"离散选灯"缺口**（收集页"三个决定"之第 2 条目前最具体的算法答案）；与 Lipp 拼成"先离散后连续"两段式管线 |
| 2026-11-03 | [[Sloan 2008 — Stupid Spherical Harmonics Tricks\|Sloan 2008, Stupid Spherical Harmonics Tricks（GDC 2008 伴生文档，节选抽灯节）]] | ①② | **探针/SH → 显式灯的解析路线**：单方向光+环境光有闭式最优解（在 irradiance 空间最小二乘），多灯用球面覆盖点集 + BFGS 找峰（1–6 阶起点数 {1,3,6,10,15,22}）；抽灯本身治 ringing | "隐式表示→显式灯"的工业原始文献；与 Gkaravelis 各占一头（方向域解析 vs 空间域组合）；旧探针数据迁移的零成本初值方案 |
| 2026-11-03 | Liu et al. 2022, Optimization of Indoor Luminaire Layout Using Improved PSO（Energies 15(4):1482, MDPI）（记名未建页） | ④⑥ | 建筑侧混合整数样本：4 个混合变量（间距×2 + 行列灯数）同时定灯数与布局，UDPR 概率取整解整数部分；与 DIALux 仿真对比平均照度差 0.4% | ④ 的"混合整数"建筑样本：用参数化把"灯数"变成连续可解的整数变量，思路与 Gkaravelis 的聚类剪枝互补；目标仍是规范照度不是观感迁移 |
| 2026-11-04 | [[Debevec 2006 — A Median Cut Algorithm for Light Probe Sampling\|Debevec 2006, A Median Cut Algorithm for Light Probe Sampling（SIGGRAPH '06 Courses，一页）]] | ①② | **HDRI→显式灯的最经典零优化方案**：lat-long 图递归切成 $2^n$ 等能量区域，质心放灯、颜色取区域像素和；64 盏≈Monte Carlo 参考、256 盏几乎无法区分；lat-long 畸变补偿与 SAT 加速两坑点到 | "隐式→显式"三路线的图像域起点（Sloan 走频域、Gkaravelis 走空间域）；输出可当 Lipp 式连续拟合的初值；影视抽灯谱系的源头文献 |
| 2026-11-04 | [[Cieri 2019 — Streamlining IBL Workflows with Computer Vision and USD\|Cieri & Schwank 2019, Streamlining IBL Workflows with Computer Vision and USD（DNEG iblManager，SIGGRAPH '19 Talks）]] | ②⑥ | **"抽出的灯怎么在管线里流动"的工业答案**：Nuke 三件套（阈值+OpenCV 轮廓+dilate 聚拢，美术高层参数监督）+ **USD LightCache** 跨 DCC 灯阵资产；坦承 DCC 灯型不兼容处仍需手工微调 | ⑥ 的关键补充：回答工具形态（半自动+美术末端）与数据格式（USD schema 扩展）两个学术文献不写的问题；与 LightCraft/Spherelight 构成影视谱系 |
| 2026-11-04 | Viriyothai & Debevec 2009, Variance Minimization Light Probe Sampling（SIGGRAPH 2009 Sketches）（记名未建页） | ①② | median-cut 的直系改进：把"等能量贪心"换成方差最小化目标 | Debevec 2006 的后续；如需更高质量的 HDRI→点灯静态采样可跟进，正文未读 |
| 2026-11-04 | HDRI Light Export for Nuke（Daniele Tosti，生产用 Nuke 工具集，免费发布，GitHub 有 Gaffer/Blender 导入器）（记名未建页） | ②⑥ | iblManager 同族且**个人可直接获取**：lat-long HDRI 拆成逐灯 EXR + 元数据（EV、lux、3D 位置、距离、scale、sun/HMI 标记）；可选 Depth Anything V2 单目深度定灯距；导出 Arnold/USD 灯格式 | ⑥ 的可动手样本：与 Shlyaev 2025（Spherelight）思路同族但无需大厂内部工具；可直接拆包研究其参数拆分与元数据约定 |
| 2026-11-05 | [[Lin 2013 — Interactive Lighting Design with Hierarchical Light Representation\|Lin et al. 2013, Interactive Lighting Design with Hierarchical Light Representation（CGF/EGSR 2013）]] | ①②④⑥ | Gkaravelis 2016 直接对标的 [9]：粗格撒灯 + NNLS 解强度 + 删弱灯（5%）+ 八叉细分两轮 → **灯层级 + $Impro$ 边际收益拐点自动定灯数**（阈值 1.6）→ simplex 精调；目标画在 3D 物体表面；10–40 s/场景、$D$<1.5%；表 1 实证"照明像 ≠ 灯位对" | "先离散后连续"两段式管线的完整原型，比 Gkaravelis 早三年；与 Gardner 2019 一起把"②参数化"推进到"可变灯数"的两种解法（cut 选数 vs 固定 N + 强度归零） |
| 2026-11-05 | [[Gardner 2019 — Deep Parametric Indoor Lighting Estimation\|Gardner et al. 2019, Deep Parametric Indoor Lighting Estimation（ICCV 2019）]] | ①② | 学习侧"单张 LDR 图 → N 盏 3D 参数灯（方向/距离/角尺寸/颜色）+ 环境项"：核心是**可微"灯→环境图"投影层**绕开灯的一一对应（直接对应训练误差差数倍，定量负结果）；N=3 最优，3→5 退化；CPU 51 ms/图 | 补上"图像证据 → 显式灯参数"的学习路线；其"两步训练"（先全局环境图损失、后逐灯精调）与 Schwengerer 实测的最优排期（先 L-BFGS 后 Adam）跨范式同构，加强"先全局后精调"这一收敛信念 |
| 2026-11-05 | Kawai, Painter & Cohen 1993, Radioptimization: Goal Based Rendering（SIGGRAPH '93，pp.147–154；UUCS-93-002）（记名未建页） | ①④⑤ | 整个谱系的源头：radiosity 场景上按**高层目标**（最小能耗、"pleasantness"/"privacy"等主观印象，取自 Flynn 照明工程 + 多维标度用户实验）优化灯发射率/反射率/聚光方向；**已内嵌 Tumblin-Rushmeier 物体空间感知模型**做主观亮度比较 | ⑤ 的历史根：1993 年就有"感知模型进优化目标"的尝试（物体空间而非图像空间）；⚠️ Utah 镜像 PDF 在本环境 TLS 连接失败（多次重试无效），未建页，仅依 ACM/SIGGRAPH 历史档案摘要与多方转引记名 |
| 2026-11-05 | Weber, Garon & Lalonde 2022, Editable Indoor Lighting Estimation（CVPR 2022）（记名未建页） | ②⑥ | Gardner 2019 直接后续：混合表示 = **1 盏可编辑参数主灯 + 非参数 HDR 残差纹理 + 粗房间盒体**；"1 主灯承担 >80% 能量"假设；把可编辑性（预测后美术可改）列为与真实感同级的目标 | ② 的"可解释性优先"设计样本：直接回应收集页"三个决定"第 3 条（拟合结果要不要可解释）；正文未读 |
| 2026-11-05 | Lopez-Moreno et al. 2013, Multiple Light Source Estimation in a Single Image（CGF 32(8):170–182）（记名未建页） | ①② | 单图多光源估计的非学习路线：方位角曲线拟合 + **灯数递归细化**、天顶角用密切弧近似法线；附带 GT 数据集 | Gardner 2019 引用的前学习时代对照；灯数递归细化与 Lin 2013 的层级 cut 思想呼应；正文未读 |

**本日检索备注（2026-11-02）**：检索通道（WebSearch）正常，无失败记录。Schwengerer 论文 PDF 无 ToUnicode 文本层（提取为乱码），已改用逐页渲染+图像识读核读，方法/评估/结论章数字均已亲见；Ch.4 BSP 实现细节未逐字核。Giroux 论文 venue 是否正式收录未核（arXiv v3 为止）。另见 2026-09 的一篇 Ad-Relight（arXiv 2609.37951，广告图重打光）用到 GPT-4o 做光照一致性评审 + ILL-SIM 亮度场余弦相似度——图像到图像路线、不产出灯参数，按选文标准**记名不收**，仅其"感知评审"思路与 Giroux 呼应。

**本日检索备注（2026-11-04）**：检索通道（WebSearch）正常。两组四查询：① IES/可微灯参拟合（命中多为通用可微逆渲染材料，Debevec 2006 由历史线索锁定，建页）② 工业 light rig 匹配（中 DNEG iblManager 全文快照，建页；附带命中 HDRI Light Export for Nuke，记名）③ lightmap→灯反推（命中均为已收录的 Schwengerer/Lipp，无新）④ median-cut 正文核验（USC 官方 PDF 下载成功，全文亲读、两张原图提取并目视核验）。**失败记录**：DNEG 官网 PDF 直连返回 403 Forbidden（换 UA 无效），Cieri 2019 内容改由两处独立检索快照交叉核验（摘要至结论、参考文献齐全），图 1–3 未获取原图；如需原图日后换网络重试。Viriyothai & Debevec 2009 仅记名未读正文。**⑤ 评估槽今日仍无新增**（连续第二天）——"感知指标作场景空间光照拟合目标"的具体样本仍未命中。

**本日检索备注（2026-11-03）**：检索通道（WebSearch）正常，无失败记录。两轮四组查询：① mixed-integer 选灯（中 MDPI IPSO，记名）② SH/探针→显式灯（中 Sloan 2008 抽灯节，建页；附带命中 Ambient Dice 2017——二十面体基替换 SH 的**表示层**工作，不涉及"抽成显式灯"，按标准**不收**，仅备注其存在）③ 感知损失作可微目标（命中均为通用 LPIPS 材料综述，无"场景空间光照拟合用感知目标"的具体样本，**今日 ⑤ 槽无新增**，此方向留待后续）④ Gkaravelis 2016 由查询①的引用链发现（AUEB 官网 PDF，文本层完整）。Gkaravelis 期刊版卷期未核；Sloan 文档 SH35/SH36 版本间环境光归一化因子表述有差异，笔记内已标"未核"。

**本日检索备注（2026-11-05）**：检索通道（WebSearch）正常。三组六查询：① 逆灯光设计+lightmap 反推（命中均为已收录的 Lipp/Schwengerer，无新；附带重新发现 Schoeneman 1993 Painting with Light 摘要）② Lin et al. 层级灯表示（跟进 Gkaravelis [9] 待办，作者主页全文 PDF 下载成功、10 页全部亲读，建页）③ Kawai 1993 Radioptimization（谱系源头；**失败记录**：Utah content.lib.utah.edu PDF 多次 curl TLS 连接失败、core.ac.uk 403、ACM 需登录，正文未获取，按摘要+SIGGRAPH 历史档案记名未建页）④ 感知损失/可微光照目标（连续第三天 ⑤ 槽无论文级新增；命中 2025 SIGGRAPH 扩散先验室内光照估计把 LPIPS 当蒸馏损失、StyleLight 用 LPIPS 拟合潜码——均为图像空间目标，不构成本页 ⑤ 的场景空间样本，仅备注）⑤ 单图显式灯估计（中 Gardner 2019，arXiv 全文亲读建页；附带 Weber 2022、Lopez-Moreno 2013 记名）⑥ 插渲匹配/外观对齐方向未单独开查（明日可补）。原图 5 张提取并逐张目视核验（Lin 图 2/图 3；Gardner 图 2/图 4/图 6）。**跨范式发现**：Gardner 的"两步训练"（先全局环境图损失后逐灯精调）与 Schwengerer 实测最优排期（先 L-BFGS 后 Adam）、Lin 的"先 NNLS 初始化后 simplex"结构同构——"先全局/粗后逐灯/精"在优化与学习两侧反复出现。

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
