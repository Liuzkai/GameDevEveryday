---
type: personal-knowledge-model
created: 2026-09-07
updated: 2026-10-29
status: calibrated-from-reading
reading_checkpoint: 2026-09-18
read_paper_count: 40
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
> **2026-10-12 更新（资料侧；阅读状态无变化）**：**三角分支补齐（"三角对偶日"）**——[[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)]]（**A 经典**：三角分支定义者；分裂 1→4 + 两条平均 + **μ₃²=μ₂"平方关系反推权重"**；⚠️ 原文 PDF 直链全部失效——书目三源交叉 + Stam 附录 B 一手转述，核证清单入笔记）+ [[Stam — Evaluation of Loop Subdivision Surfaces (1998)]]（**A 经典**：三角版求值，"**同样的机器换坐标系，细节不同而已**"；12 点四阶箱样条 / **3/4 免费午餐 + 无限瓦片** / K=N+6 / N=3 Jordan 特例；**lpdata50.dat + lptest.c 可下载**——1998 年求值器实测材料）——**48 年弧线双轨（四边/三角）主体闭合**；[[Subdivision Surfaces]]"Loop 1987 记录位"兑现、Next Step 新增三角两篇（按需 ≈40 分钟 + 可选打印实验）。**前沿资料侧**：GS"**分配三连**"（[[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection|SplatQ]] 保留端配额+联合选择 / [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction|CompactSplat]] 生成端内容自适应+推理旋钮 / [[2026-10-09-H2-Ray — A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering|H2-Ray]] 执行端均衡复用）+ 流式动作 [[2026-10-09-Sonic-AMG — Streaming Human Motion Generation from Spatial Audio|Sonic-AMG]]（"**流式两错位**"检查单）——材料层就绪，按需取用（各 10–25 分钟）。**已读账本维持 30 篇**；动作清单不变。
> **2026-10-13 更新（读者侧 + 资料侧）**：**WorldCrafter 转 `read`（读者自己在另一设备标记）——已读账本 30 → 31 篇**（[[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory|WorldCrafter]] 入账）；读者同时落定 **README 许可章节**（MIT 范围 = 原创代码 + `Files/Html/` 自写图解；第三方按原许可）与仓库基础设施（Obsidian 插件同步策略）。**资料侧（"流体开域 + 感知优先日"）**：**[[Stam — Stable Fluids (1999)]] 入库（A 经典）**——**流体域开域**（无条件稳定求解器：拉格朗日回溯 + 隐式扩散 + 投影；"感知优先"设计哲学的 1999 原话出处；傅里叶"换空间"）；新概念 [[Fluid Simulation]]（四步分解 + 三路线 + 时间线）；**Stam 双身份闭环**（2005 细分奥斯卡三人组 × 2008 Maya Fluid Effects——与库内 10-10~10-12 细分弧线由同一人串起）。**前沿 3 篇**：[[2026-10-09-Efficient Confidence-Aware Densification for 3DGS|Confidence-Aware Densification]]（**分配四端之"生长端"**："预算跟着缺口走"）/ [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]（**"水的两条路"**：感知路线 2026 样本）/ [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]]（**"优先级 > 权重"**）——新概念 [[Motion Retargeting]] 建立（动画数据管线第三环）。**已读账本 31 篇**；动作清单不变（**furnace test / 动态灯光复审为两大首件**）。
> **2026-10-18 更新（资料侧；阅读状态无变化；W42 收官）**：**采样线理论源头补齐——三件旧工具集齐日**。本周经典四连（[[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR DI 2020]] → [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)|ReSTIR GI 2021]] → [[Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)|GRIS 2022]] → **[[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS 1995]]**）：家族学术主线结构齐 + **"组合权重的 30 年降本史"**（MIS 定理 1 → Talbot O(M²) 痛点 → Bitterli O(M) 无偏化 → GRIS 置信权重）一条线可读；**GRIS 附录 A 推导链（Hard）的完整前置链就此闭合**（MIS/RIS/ReSTIR 均 Normal 结论层）。两枚工程接口条款入库：MIS §5.4（近似权重仍无偏——ReSTIR O(M) MIS 的理论许可）、§5.2（坏权重任意坏、坏样本配比最多差 n 倍——预算论证样板）。**已读账本维持 38 篇**（10-16 读者标记 4 篇后无新 read；读者近编辑 MS microfacet 两篇——PBR 复习窗口打开）；动作清单不变——**"动作月"连续催办第 6 日，五件动作零执行**；W42 周报已产出（[[2026-W42]]）。
> **2026-10-20 更新（读者侧 + 资料侧）**：**读者标记 2 篇 read——已读账本 38 → 40 篇**（[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] 毛发线工程经典 + [[2026-09-01-ToCo-Mesh — Topology-Consistent Dynamic Mesh Reconstruction via Adaptive Tessellation and Surface-Aligned 2DGS|ToCo-Mesh]] GS 网格重建线，逐文件核实）。**读者同日自建 [[Lightmass in the Rendering Knowledge Map|Lightmass 与预计算光照专题]]**（知识关系向 explainer：渲染方程→GI 求解→烘焙/动态→引擎系统→Lightmap 数据四层区分；红墙照红地板的最小实验设计；官方文档访问被 403 阻断时如实标注"未读取"——**核验纪律的又一次自我执行**；自述"不自动修改论文阅读状态或个人知识等级"）——**学习焦点信号：预计算光照/烘焙线打开**。资料侧对接：经典线当日入库 [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]]（Radiosity 源头，恰为该专题"漫反射互反射特解"的论文节点 + GI 概念页谱系表第 3 行落位）；前沿入库 PhysSkin++（实时物理蒙皮，rig 互补目标）。frontmatter `read_paper_count` 由过期值 31 修正为 40。动作清单不变（**"动作月"第 8 日，五件零执行**）。
> **2026-10-14 更新（读者侧 + 资料侧）**：**读者在另一设备产出专题报告《现代实时光照的共同设计框架：MegaLights、Lumen、ReSTIR 与 RTXDI》**（`Learning/Modern Real-Time Lighting Framework.md`，417 行 + 3 张自制 SVG；含版本证据表与固定代码提交）——报告自述"**本次没有运行 UE benchmark，也不改变知识库中的已读／掌握状态**"：**已读账本维持 31 篇**；报告把"**采样 → 时空复用 → 重建／去噪 → 更新／失效**"确立为阅读框架，并列出 **5 项最小实践路线**（两盏灯采样 / 门开关历史 / 离屏遮挡 / Lumen 缓存 / 透明分层——均为教学建议，未执行）。**资料侧（"采样与重建双端开域日"）**：经典双纳入库——[[Talbot — Importance Resampling for Global Illumination (2005)]]（**RIS 源头**：三步重采样 + 校正因子 + 方差劈分 + 等时间准则 M=TY/TX；**"ReSTIR 的理论前置"——读者报告自己指认的库内缺口，当日结清**）+ [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]]（**实时重建开端**：时域累积 + 方差引导 à-trous + 三条边缘停止；1 spp / 10 ms / ≈10× 稳定性）；新概念 [[Resampled Importance Sampling]] + [[Reservoir Sampling]]（"一个槽位代表整条候选流"；RTXDI 固定提交可直读）。**前沿 3 篇**：[[2026-10-09-MeshRepair — Autoregressive Topology Repair for Generated Artisan Meshes|MeshRepair]]（**就地修复**：编辑序列而非新网格；quad F1 = 下游硬约束）/ [[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology|CAST]]（**拓扑感知动捕**：可观测性分析 + 可靠性门控；捕捉 ⇄ 重定向边界合并）/ [[2026-10-10-TETRIS-4DGS — Efficient Post-Training Compression of 4D Gaussian Splatting|TETRIS-4DGS]]（4DGS 秒级 40× 压缩；"翻译给旧工具"）。**已读账本维持 31 篇**；动作清单不变（**furnace test / 动态灯光复审为两大首件**；新增候选：光照框架线——两篇结论层各 ≈20 分钟 + 读者报告"两盏灯采样实验"）。

> **2026-10-21 更新（资料侧；阅读状态无变化）**：**烘焙线第二节点落位——"烘算子 vs 烘答案"**。经典 [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)]]（**A 经典·历史重要性 5**）入库：Radiosity 烘**答案**（换灯重解），PRT 烘**算子**（换灯一次点积，成本与灯数/灯尺寸无关）；三档价目 `DU` 9 系数 / `DS` 25 系数 / 光泽 25×25——**"阴影 = +2 阶、光泽 = 平方"**，与你的 SABC 档位语言同构（≈30 分钟结论层）。**GI 谱系表第 4 行落位**，第 3/4/6 行自此连成一条轴：**视角 → 光照 → 几何，每代换一个被冻结的变量**。**新概念 [[Spherical Harmonics]]** 建立——此前 SH 在库内"到处被引用、没有自己的页"，现在它把 RH 2001 的 9 系数、PRT 的 25 系数、探针/DDGI、3DGS 的视角相关颜色串成一条线。**前沿（周三空批，改做 10-19 批次补扫）**：[[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]]（A-："不要让 VLM 写程序化代码，要让它改"——AST 深度 2 的可编辑表面 + 三种编辑策略 + 确定性视锥网格五道闸门；**放置五道闸门可直接搬进 UE PCG Graph**）+ [[2026-10-19-Wan-Alpha++ — Animating a Single RGBA Asset into Controllable Transparent Video Effects|Wan-Alpha++]]（B+：alpha 一等公民的透明视频生成；**"换时间原点让旧机制可用"**；素材侧工具，不进运行时预算表）。**已读账本维持 40 篇**；动作清单不变——**"动作月"第 9 日，六件零执行**；今日新增 5 分钟零成本动作：把项目画质档位翻译成 9/16/25 个 SH 系数。

> **2026-10-22 更新（资料侧；阅读状态无变化）**：**水的第二条路落位——“不解方程，采统计”**。经典 [[Tessendorf 2001 — Simulating Ocean Water|Tessendorf 2001]]（**A 经典·历史重要性 5**）入库，与库内 [[Stam — Stable Fluids (1999)|Stam 1999]] 并列为水的两条合法路径：Stam **解 NS 方程**（逐步推进、按元素计费），Tessendorf **不解方程**——按 Phillips 谱采隨机波的统计分布，一次 IFFT 出整张海面（固定成本、与海面大小无关）。**关键不在“哪个更对”**：FFT 路线的强制条件是**可分解、可周期、不可剔除**（量化色散导致循环周期，图 8），因此不能做“一道浪打到岩石上”；它的价值在“**无边海面 + 一直循环**”这一类需求上。配套三件工具直接可用：$\omega=\sqrt{gk}$ 色散定相速、Creamer 尖锐位移定尖峰、**Jacobian $J<0$ 作泡沫遮罩**（图 16，“拿已算出来的量免费再用一次”的最干净范例）。**阅读建议**：结论层 ≈25 分钟，只看 §2 四步配方 + 图 8/12/16；推导层（谱推导、iWave 显式波动方程）留 Hard。

> **前沿三篇（均为 10-21 批次）**：① [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]]（**A**）——世界模型“层次 2”的完成形态；其新意在于**“规则可执行 ⇒ 标注零模型推断”**（114,334 段字幕全由规则模板渲染，不用 VLM），与库内 [[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]] 的“不要让 VLM 写代码”共同指向同一条经验：**让模型做判断，不让它做记账**；② [[2026-10-21-Kim — RLSplat Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting|RLSplat]]（**A**）——GS 预算的“成本侧”，以“分配”代“裁剪”，显存 9.6× 少；③ [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS]]（**B+**，NeurIPS Spotlight）——连续 LoD 的排序键，用**优化前**的图像复杂度做空间先验。两篇都在同一个问题上加了一个度量：预算不只有“多少”，还有“花在哪、什么时候花”。

> **今日新判断：“预算”在库内已分成四种计价对象，而它们对“降档”的定义不同**。① **容量计价**（Budgeted-GS / Zador 律：误差 ∝ N⁻¹ᐟ²）——降档 = 减少 primitive 数；② **成本计价**（今日 RLSplat：资源线性 O(B·K)）——降档 = 降低每个元素的处理代价，数量不变；③ **精度计价**（CurveCodec 误差界 0.01–1 cm）——降档 = 放宽误差承诺；④ **时间计价**（今日 MG4.0：每帧必须重算什么）——降档 = 把某些计算从帧内挪到帧外。**实用价值**：你的 SABC 预算表目前分配的是“容量”（发射器数 / 粒子数 / 贴图尺寸）；而 C 档技能真正的痛点往往在“时间”（帧内重算什么）与“成本”（增加一个敌人同屏时每个元素的代价）——这两份账值得单独建一页。

> **已读账本维持 40 篇**（今日无读者新 `read`，`git pull --ff-only` 无新提交）；动作清单不变——**“动作月”第 10 日，六件零执行**；今日新增一个 5 分钟零成本动作：**把你手上一个 C 档技能的抽象层分成“造形”与“流动”两块，看哪一块能用固定成本的 IFFT 替换逐粒子模拟**（水面经典的钱花在哪，这就是答案）。

> **2026-10-23 更新（资料侧；阅读状态无变化）**：**角色形变支线第一次有了经典锚点——“混的对象必须同类”**。经典 [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|Kavan 2008 DQS]]（**A 经典·历史重要性 4**）入库：LBS 的病根不是权重，是**正交矩阵集合对加法不封闭**——混出来的矩阵悄悄长出缩放/剪切，于是肩塌、于是糖果纸；修法是换一个对线性组合封闭的表示（对偶四元数），**混完归一化即回 SE(3)**。成本是最有说服力的一行：顶点着色器 **36 → 43 条指令**（CPU 3.67→5.07 ms，GPU 197.4 vs 122 / 84.9 FPS）。两处诚实边界：表示不了缩放（两阶段绕）、最短路径会翻面（179°→181°）。**新概念 [[Skinning]]**（第 42 个）把四代排开：LBS 1988 → DQS 2008 → PSD/corrective → 神经蒙皮（[[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning|PhysSkin++]]）→ 肌肉层（今日 RLMuscle）——**四者不是替代，是四个成本档位**。

> **前沿两篇（10-21 提交件）**：① [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning|RLMuscle]]（**A**）——1D Hill（284 肌腱 / 50 DOF）在 RL 环内解“哪组激活能复现动作”，3D 体积肌肉（194 块 / 468K 四面体）只做一次前向；**两段共用同一条 Hill–Millard 本构律，所以形变不是拟合出来的，是“产生运动的那股力”直接推出来的**——一致性由构造保证。EMG 先验把欠定解从“最小用力”拉回“人类募集”（in-band 20%→91%）；四面体几何多重网格（重心坐标延拓 + Petrov–Galerkin 限制 $RP=I$）把全身压到 **18 ms/帧 @4090**（对照 Houdini Otis 20.8 s/帧）。**18 ms = 60 fps 预算的 108% → 一个角色就超帧，所以它是“真值工厂”不是运行时**：标定 tension line / 产 corrective 训练数据 / 单角色特写。② [[2026-10-21-Wang — LiveAct Real-Time Multi-Agent World Models with Heterogeneous Action Interfaces|LiveAct]]（**A-**）——拆掉世界模型“只有一个玩家”的隐含假设：动作 = **schema 词（语义）+ 数值（仿射调制）**，归因 = **学出来的空间门**（推理时不用 mask/box/pose/参考图）；512×448 @ **17.70 FPS**；**Act-F1 0.594 —— 四成离散事件没执行对**，“能看、能大致跟”，还不是“能玩”。

> **今日新判断：“一致性由构造保证”是一个可命名的范式**。三篇同框：Kavan 换混合空间（平均完仍是刚体）、LiveAct 换归因方式（这路动作只作用于那个 agent）、RLMuscle 共本构律（形变与运动同源）——**它们都不写“一致性损失”，而是让错误在结构上不可能发生**。可迁移的检验只有一句：**去掉那个后处理/损失项，它还成立吗？** 成立 = 结构保证；不成立 = 事后补丁。配套的第二条判断是**“成本三级”**：换公式（DQS，+7 条指令）/ 换算法（PhysSkin++，12–14 ms）/ 换算力与离线（RLMuscle，18 ms + 4090）——**同一种质量提升，成本可以差三个数量级，选型先问自己站在哪一级**。

> **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交）；动作清单不变——**“动作月”第 11 日，六件零执行**。今日只给一件：**如果只做一件，做红墙实验**（一次动手，三处收益：Lightmass 专题闭环 + [[Global Illumination]] 谱系表第 3/4 行 + 破零）；另附 2 分钟动作：**记下你角色肩/肘塌陷的“关节、角度、镜头距离”三个数**——那就是“要不要考虑 DQS”的全部依据。

> **2026-10-24 更新（资料侧；阅读状态无变化）**：**参数化/UV 整条支线开域——“无翻转是求解器给的，无重叠要你自己查”**。经典 [[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|Lévy 2002 LSCM]]（**A 经典·历史重要性 4**）入库：三步流水线（分割–参数化–装箱）自此定型；核心手法是把共形条件写成复方程 $\partial_x U+i\partial_y U=0$ ⇒ **u、v 耦合** ⇒ **边界自由**（只钉 2 点）+ 一次稀疏求解（Hermite Gram 矩阵 $C=M^*M$，共轭梯度）；四条可证性质——唯一解 / 相似不变 / **分辨率无关** / **无翻转**。**最关键的一行边界**：**局部单射（无翻转）≠ 全局单射（无重叠）**——图 6 就是“每个三角形都没翻、但两块仍然叠在一起”，实际工程靠 stencil 检测 + 切 chart 补救。装箱率 0.37–0.48 → **0.55–0.60**（轮廓“地平线”装箱）；质量反转值得记：优化前 3.5 **劣于**调和 3.2，优化后 1.52 **优于** 1.65——**同一个目标函数，加不加面积项是两个结论**。**源头补页** [[Tutte — How to Draw a Graph (1963)]]（**A 经典·历史重要性 5**）：“调和 ⇒ 单射”的第一条定理（三连通平面图 + 凸边界 + 重心 ⇒ 无交叉 / 面为凸 / 解唯一），**凸边界既是保证也是枷锁**——LSCM 为自由边界放弃了全局单射，Campen 2016 再把它找回来，今日的 Kaushik 2026 又从“流”这个新对象上重新证了一遍。⚠️ **原文 OUP/Wiley/JHU 全部 403/404，本页为二手重建**，顶部附核证清单（✅ 标题/DOI、✅ 定理陈述、⚠️ 证明结构未核实），并顺带纠正 LSCM 参考文献 [30] 把 Tutte 误记为 1960 年 *Convex representation of graphs*。**新概念 [[Mesh Parameterization]]**（第 43 个）：三目标不可兼得表 + 8 方法 × 五列保证对照表。

> **前沿两篇（10-22 提交件）**：① [[2026-10-22-Liu — AnyGameRenderer Entity-Consistent Generative Rendering Across Games|AnyGameRenderer]]（**A**）——“层次 2”（引擎管状态、模型管画面）第一次给出**验收标准**：实体级四指标（temporal match / position match / unrendered rate / hallucination rate），并且必须先量出 **GT 天花板**（80.9 / 34.1——连真值录像自己也只能得这个分）再看模型分。两处方法可迁移：**字幕六段式契约**（把“画面里有什么”写成可校验的字段，模型只管渲染不管记账）与“**噪声是正则项**”（训练时故意加噪，逼模型别把粗模纹理当圣旨）。**VFX 四问平移**：同一套指标可以问“我这条特效的生成版，实体账对不对”。② [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik 2026 Tutte-Like 单射流]]（**A-**）——Tutte 定理的**第三个对象**：从“映射”换到“流”。球面拓扑 + 单源 $z{=}1$/单汇 $z{=}{-}1$ 解离散 Laplace（**任意正权 Laplacian**），梯度转 90° 当向量场，证明诱导的流单射（轨迹永不相交）；证明支点极漂亮——**Campen 2016 的组合通量条件 ≡ Gortler 2006 的离散 Poincaré–Hopf 指标**（图 6 三行对照）。工程钩子：对 Laplacian 边权**可微** ⇒ 能在不离开可行域的前提下优化流向。边界诚实：只球面 + 单源单汇、不覆盖全部单射场、优化框架自述“不足以做下游任务”。

> **今日新判断：“正确”有两种形态，而它们要的证据完全不同**。**可证的正确**——单射、无翻转、唯一解、混完仍是刚体（LSCM / Tutte / Kaushik / 昨日的 Kavan），证据是定理与反例，你能在写下代码前就知道它对不对；**可测的正确**——实体账、画质、FPS（今日的 AnyGameRenderer，以及你自己的 SABC 预算表），证据是仪表与基线，你必须先有 **GT 天花板** 才知道 80.9 分算好还是算差。两者之间还有第三种：**既不可证也不可测**，只能靠“看起来对不对”——这类判断在项目里占比最大，也最容易吵起来。**可操作的检验**：下次评审一个方案，先问它落在哪一格；落在第三格却按第一格的语气说话（“数学上保证”），就是危险信号。对自己的 SABC 表同理——**它是可测的，但可测的前提是先有可证的结构（发射器 → 粒子 → 像素这条账能不能对上）**，账对不上时，测出来的数只是噪声。

> **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交，连续第 4 日）；**动作月策略今日改动：停止计数与催促**——连续 12 日零执行说明问题不在优先级排序，而在“这些动作和日常工作流没有交点”。改为**挂在场景上**（不新增清单，只换触发时机）：① 导出 Lightmap UV 时顺手看一次重叠（30 秒，正好是今日 LSCM 图 6 的手感）；② 粒子系统调峰值时记一次同屏数（2 分钟）；③ 角色肩/肘塌陷时记下关节、角度、镜头距离三个数（1 分钟，昨日 DQS 的判据）。其余三件不再提。

> **2026-10-25 更新（资料侧；阅读状态无变化）**：**给 overdraw 立账——你五维指标里唯一没有定义页的那一项**。补两个「被引用 11 次却没有页」的概念：[[Overdraw]]（**第 44 个**）与 [[GPU Architecture]]（**第 45 个**）。
>
> - **Overdraw 的三件事**：① 它有三种口径（Coverage / Shaded / Blended），数值可差一倍以上，**混用会得出矛盾结论**；② 成本恒等式 **层数 × 屏占比 × 单价**——层数是场景属性、单价是技术属性，必须分开谈；③ 屏占比换算表：**100 个 256² sprite ≈ 3.2 倍全屏**，同样 100 个 64² 只有 0.2 倍。
> - **对你那张 SABC 表的具体修订建议**：现行「同屏粒子数 S 3000」单独存在时**没有性能含义**——按平均 sprite 尺寸换算，同一个 3000 可以是 **5.9× 全屏**（64²）也可以是 **94.8× 全屏**（256²）。建议补一行**像素预算** Σ A_i / A_screen ≤ k（S/A/B/C 各一个 k），粒子数降为**辅助约束**（它管的是 CPU 侧 spawn/更新与 draw call，不是填充）。好处是美术能自己换算：「要上 256² 就得把数量砍到 1/4」。
> - **经典 [[Everitt — Interactive Order-Independent Transparency (2001)]]（A·历史重要性 4）**：正确的半透明要付 **层数 × 一遍场景**；**加法混合买的是「无限层免排序」，卖的是「不能吸收/遮挡」**——每次「这个烟要不要更真实」都是在给层数付钱。⚠️ 它是 **14 页技术演示文稿**，**唯一定量结论「n = 4 通常够用」是目测经验值，全文无计时、无误差度量**——引用时必须标明证据强度。
> - **GPU Architecture 的用法是一张表**：六个实测指标分别落在 compute（Particles）/ 前端（DrawCalls）/ 光栅化器（Primitives）/ ROP（OverDraw）/ 提交线程（CPUTime）上；**先看 CPUTime : GPUTime 的比值，再动手删特效**——比值没看就开始优化，通常优化的是错的那一半。
> - **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交，连续第 5 日）。**今日无新前沿批次**（周日，cs.GR 最新仍为 10-22 提交件，listing 与 API 双通道核对一致）——按 no-filler 不凑数，把一天用在补欠账上。

> **2026-10-27 更新（资料侧；阅读状态无变化）**：**“换单位”日——报账的单位比报账的额度重要**。今日四篇（3 前沿 + 1 经典）各自把记账单位换了一次，指向同一命题：**换单位比调额度高两个数量级**。新概念 **[[Budget Units]]（第 47 个）** 把此前两版 PKM 的判断（10-22「四种计价对象」→ 10-25「像素预算」）收拢并扩为**六种**：容量 / 成本 / 精度 / 时间 / **状态-记忆** / 像素覆盖。
>
> - **[[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination|RADIANT]]（A）——量化铁证**：传输序列从三角形 $T$ 换成每物体 32 个 token（中位 **3,205 → 320**）值 **+3.3 dB**；而在同一单位内把 token 预算从 8 调到 64，只值 **≤0.04 dB**。**这条可以直接搬进预算评审**：一个额度砍一半只损失一点质量 ⇒ 它本来就在冗余区 ⇒ **该换的是单位，不是数值**。附带把 [[Neural Global Illumination]] 拆成两派：**拟合派**（NRC，逐场景训练，换场景重训）vs **前馈派**（RADIANT，一次离线训练，换场景零成本）——分界线的问法是“**这个网络见过我没见过的场景吗**”。
> - **[[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]]（A-，NeurIPS）——负收益证据**：State+State 变体 FGD **7.652**，比**完全不给历史**（7.065）还差。**历史给错方式是负收益，不是零收益。** 它同时给了今天最有用的一条实测论证：自相关分析显示**风格特征 >100 帧仍相关、局部运动 <20 帧衰减**——一条序列里两个时间尺度差 5 倍，**统一上下文窗口对前者不够长、对后者纯浪费**。这条直接催生了 **[[Motion Tokenization]]（第 48 个）**（此前被引用 4 处却一直缺页）。
> - **[[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation|UniWorld]]（A-）——结构性上界**：深度状态只在窗口接缝读一次，活跃显存上界 `4·L_max·S·96` 字节 = **19.05 MiB @960×544，与时长无关**。**“与时长无关”是所有长时程设计的及格线。** ⚠️ 但它的**消融几乎全跨零**（boundary-only +0.006 dB）——**架构贡献成立、机制归因不成立**，读这类论文要分开问两句。
> - **经典 [[Reyes — The Reyes Image Rendering Architecture (1987)|Reyes 1987]]（A·历史重要性 5）——你的 Overdraw 线的史前史**：微多边形 = 平面着色四边形、边长**约半个像素**（Nyquist 极限）⇒ **着色率第一次与几何复杂度解耦**（Nanite 判据的 1987 版本）；代价是**先着色后判可见 ⇒ 结构性 overdraw**（6.8 M 微多边形 / 63 万像素 ≈ 10.8 个/像素 ⇒ **约 2.7×**，⚠️ 本库反推非原文）。**1987 与 2010s 的 visibility buffer 是同一场战争的两端**：同一个“着色与可见谁先”，1987 年买的是**统一性**（一套采样点解决抗锯齿/运动模糊/景深），2010s 买的是**便宜**。⚠️ 原文六条路全断，二手重建、本页不出图。
> - **（补记 10-26 未落块）[[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)|Harris 2004 · GPU Gems 38]] + 新概念 [[GPGPU]]（第 46 个）**：**三条锁**（写位置由光栅化钉死 ⇒ 只能 gather / 一图不可同时读写 ⇒ 必须双缓冲 / 寄存器与精度受限 ⇒ 必须打包）+ **成本公式 $W H N_{pass} B$**。**对你有直接用处的一句**：说“我们跑 256² 的模拟”等于没说，**要说“256² × 65 pass”**——pass 数是被忽略得最彻底的第一乘数；Niagara 的 Simulation Stage 迭代次数就是它。
> - **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交，连续第 7 日）。**动作清单沿用 10-24 的“挂在场景上”策略**（不计数、不催促）：① 导出 Lightmap UV 时看一次重叠；② 粒子调峰值时记一次同屏数；③ 角色肩/肘塌陷时记关节、角度、镜头距离三个数。**今日可加的第 ④ 件（同样挂在场景上，30 秒）**：给一个 S 档技能的峰值帧算一次 $\sum A_i / A_{screen}$——如果 >10×，说明它现在是“数量合规、填充超支”。

> **2026-10-28 更新（资料侧；阅读状态无变化）**：**“分布 > 总量”日——预算的总额常常不是你能动的那个数**。今日四篇（3 前沿 + 1 经典）各自在一条**不同的轴**上做同一件事：把预算从“均摊”改成“按需”。新概念 **[[Cloth Simulation]]（第 49 个）** 把 1998 经典与 2026 前沿放进同一页；**[[Budget Units]] 由六种计价对象扩为七种**，新增 **「时间采样」**。
>
> - **[[Baraff-Witkin — Large Steps in Cloth Simulation (1998)|Baraff-Witkin 1998]]（A·历史重要性 5）——分布命题的最早纯样本**：标题里的 “Large Steps” 就是“少步、大步”——后向 Euler 让**步长与刚度解耦**（弯曲刚度扫 **10,000×**，运行时间变 **<5%**），代价是每步解 **18,000×18,000** 稀疏系统。**三件比“隐式积分”更重要的技术**：① 约束靠**改质量矩阵** $W=\frac1m S_i$ ⇒ **精确满足、与迭代次数无关**；② **modified-PCG** 每步投影回可行方向（`filter(a)` 不变式），约束力从残差 $A\Delta v-b$ 读出；③ **位置修正项 $y_i$** 塞进顶行 ⇒ **一步去穿透且零额外计算**。**它最诚实的是 Table 1**：2,602 粒子 2.23 s / 6,450 衬衫 7.32 s / 8,757 裤子 38.5 s，而 **C/C（碰撞约束）峰值占 57.7%**——**换了分配方式，瓶颈立刻换人**。**未解决：静摩擦**（斜面上缓慢下滑），28 年后由 Truong 2026 补上。
> - **[[2026-10-24-Truong — Explicit Frictional Contact Modeling in Neural Garment Simulation|Truong 2026]]（A-，WACV 2027）——学习丢掉的是旋钮，不是精度**：神经解算器把摩擦写进训练损失 ⇒ μ 被烧进权重、推理时调不动。本文在预测**之后**接四段算子，核心是把库仑摩擦从“力不超过 $\mu\lambda_n$”换成“**这一步最多能走多远**”——**arrest length $b=\mu h^2[n^\top(\hat a+ge_{up})]_+$，质量被约掉，$b$ 只是一个长度** ⇒ 块软阈值、闭式、无迭代。**分离证据**：CC+ours 的 μ 响应 0.994/1.047/1.139，而**按 μ 重训的 native 是 0.927/0.927/0.924（平的）**。代价：穿透率仍差 30 倍（6.1 vs PPF 0.2），因为它是“一次修正”不是“一个解”；8.1× / 5.6× 加速，GNN 58% + 算子 42%。
> - **[[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting|GaussianSpline]]（A-）——“分配方式 > 分配额度”的第二号证据**：每高斯轨迹 = 节点 + 可学**段宽**（softmax ⇒ 天然单调）+ **反射式 $G^1$**。**二维消融表要横纵两读**：纵读“挪段”值 **+0.37 / +0.91 / +0.00 dB**，横读“加段”**31.61 ≤ 32.08（收益为 0）**。⚠️ 它的 **soft contact 只有 26.84，输给 Gaussian-Flow 31.10（−4.26 dB）**——**全局共享一组段宽 logits 抓不住瞬时碰撞**，这是一个明确、可验证的后续。**新判据：前六种计价对象都能摊销，第七种「时间采样」不能**——一段 3 s 的信号，信息全在 0.3 s 的峰里。
> - **[[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]]（A）——世界模型最厚的一端 + 运行时 PCG 的第一个完整样本**：决策与实现彻底分离，中间只留一份**记录**（$R_i=(G_i,M_i,B_i,O_i,U_i,\rho_i,V_i)$，$X_i=\mathrm{Compile}(R_i)$）——**几何是可再生资源，记录才是资产**；**双前沿流式**（语义前沿跑在前面，实现前沿只编译马上要用的）。**三个反直觉的数**：① 独立 argmax 让 **63%** 的计划违反硬约束（联合求解把 37% 拉到 100%）；② 去掉 Micro 邻接上下文，建筑从 250 涨到 **2,686（10.7×）**——**局部最优的失败方式不是稀疏，是拥堵**；③ 编译优化 3.24× 才是它的真瓶颈，而局部编译均值 2.134 s / 最大 3.318 s 意味着**转向即 2 秒黑屏**（3.52 s 是均值，均值掩盖分布）。
> - **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交，连续第 8 日）。**动作清单沿用“挂在场景上”策略**，不计数、不催促：① 导出 Lightmap UV 时看一次重叠；② 粒子调峰值时记一次同屏数；③ 角色肩/肘塌陷时记关节、角度、镜头距离三个数；④ 给一个 S 档技能的峰值帧算一次 $\sum A_i / A_{screen}$。**今日新增第 ⑤ 件（30 秒，直接对着你的五维表）**：把“VFX 时长 ≤3s/2s/1s/0.5s”翻译一次——**这个数约束的是长度，还是形状？** 如果是长度，那它约束的是“最长能播多久”，而真正决定性能的是“峰值落在第几帧、持续多少帧”。**这两个问法会给出不同的预算。**

> **2026-10-29 更新（资料侧；阅读状态无变化）**：**“一笔账拆成两笔”日——两项性质不同，汇率就不存在**。今日四篇（3 前沿 + 1 经典）跨三个领域，做的却是同一件事：把一个笼统的"成本/内存/世界"拆成**两个不能互相兑换的量**。新概念 **[[Streaming]]（第 50 个）**（§2 明写却零覆盖的域，而库内已有四个样本）；**[[Budget Units]] 的“容量”一种被拆成驻留与流量两半**。
>
> - **[[2026-10-26-Liu — WorldPager Zero-Stall Tiered Memory for Long-Horizon World Models|WorldPager]]（A）——内存 = 驻留 + 流量**：**两堵墙分开算**——容量墙 $N_{fit}=1+\lfloor P/B_c\rfloor$（装不下）与**位置墙**（换出的是哪块、什么时候要回来，决定能不能提前搬）。核心是那条**可算不可调**的零停顿条件 $r_{new}/BW \le T_c$：**需求流量 = 新生体积 ÷ 编译时间，与总容量无关**——所以"加内存"治不了它。三种省法各有上限：**压缩只拿常数倍、分层拿无上限、重算把容量换成算力**。两处工程细节值得记：`O_DIRECT` 绕开页缓存（**在按内存卡死的预算里，"用缓存加速 IO"是自相矛盾**）；**预取的 P99 反转**（b1_ws：预取 P50 75.9 ms / P99 2621 ms，按需 P50 1130 ms / P99 2424 ms——**P50 赢 15 倍，P99 输 8%**）。**所以流式验收必须看 P99，不能看均值。**
> - **[[2026-10-26-Wu — BoxWorld Let World Engines Control and Video Generators Render|BoxWorld]]（A-）——世界 = 规则 + 外观**：接口压到三个字段 $S_t=\{(b_{t,i},c_i,d_{t,i})\}$（盒子 / 类别 / 存在与否），**状态全符号、画面全交给生成器**——于是规则**可执行、可测试、可回滚**；结构进的是 attention mask（$\Phi(M_t)$ 加在 logits 上），不是像素。⚠️ **最该记住的是它的对照：oracle 状态给进去，生成器也只有 16.2% 完全对**——**这条线当前的瓶颈不在规则层，而在渲染器**。它是 [[World Models for Games]] 接口厚度光谱上"薄而可控"的一极，也把该页的维度推到**第六维「记忆的容量」**。
> - **[[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]]（A-）——成本 = 表达 + 覆盖**：六边形支撑 + **一阶颜色场**（一个图元内颜色可线性变化 ⇒ 同样覆盖要更少图元）+ **蜂窝定理**（等面积周长最小）。**blends/pixel 10.47 → 2.48**，同内存全面胜 2DGS/3DGS。两个"读表要小心"的地方：① **2×2 消融的两项互相替代**——31.33 + 3.80 + 4.29 = 39.42 **≠** 37.07，**形状与容量在补同一个缺口**；② **名次随基准翻转**——"同内存"下 2DGS−3DGS 是 −2.64 dB，"同原语数"下是 −1.73 dB，**绝对名次变了，相对名次没变**。这条已写进 [[Scalability and Quality Tiers]] 作**分档的第五种语言「定基准」**。
> - **[[Zwicker — Surface Splatting (2001)|Zwicker 2001]]（A·历史重要性 5）——核 = 重建 + 带限**：屏幕空间 EWA 闭式核 $\rho_k(\mathbf x)=\frac{1}{|J|}G_{JV_kJ^\top+I}(\mathbf x-m(\mathbf u_k))$，**那个 $+I$ 就是像素的有限面积**——重建（高斯的形状）与带限（像素的尺寸）在同一个式子里相加，**混叠不是靠"多采样"解决的，是靠把像素尺寸先加进核里**。**与 3DGS 只差三处**：核的指数形式、$+0.3I$ 而非 $+I$、A-buffer 换成排序——$\Sigma'=JW\Sigma W^\top J^\top$ **是同一个式子**。2001 年的实测 **0.7 fps / 0.1 fps**（当量级读）；**每像素 3 层 / 114 B 是当年硬件的隐式上限**——与今天"每像素多少层"的预算语言同构。
> - **今日的一句话下半句**：**收益不在"减小"，而在"归零"；而归零的代价一定落在分布的另一端**。预取把均值的停顿归零 → 代价落到 P99；Hexels 把覆盖次数归零 → 代价落到"同原语数下优势变薄"；BoxWorld 把规则不确定性归零 → 代价落到渲染器（16.2%）。**所以任何"我们把它消掉了"的说法，都该追问一句：那它跑到哪儿去了？**
> - **对你那张 SABC 表最直接的一个动作**：**"同屏粒子数 ≤3000"应该拆成两个数**——**驻留**（同时在场的发射器/粒子数，绑内存与 draw call）与**流量**（每秒 spawn/更新多少实例、搬多少字节，绑带宽与提交线程）。今天的 WorldPager 说明这两个数**可以差一个数量级，而且治法完全不同**（一个加内存、一个加带宽）。这与 10-25 提的"补一行像素预算"是同一件事的两个方向：**像素预算管"画多少"，驻留/流量管"装多少、搬多快"。**
> - **已读账本维持 40 篇**（今日无读者新 `read`，远程 0 新提交，连续第 9 日；工作区 5 个文件显示 modified 但 `git diff` 为空，辨认为 **LF→CRLF 换行符差异**，非读者动作）。**动作清单沿用“挂在场景上”策略**，不计数、不催促：① 导出 Lightmap UV 时看一次重叠；② 粒子调峰值时记一次同屏数；③ 角色肩/肘塌陷时记关节、角度、镜头距离三个数；④ 给一个 S 档技能峰值帧算 $\sum A_i/A_{screen}$；⑤ 把"VFX 时长"从长度约束改问形状约束。**今日新增第 ⑥ 件（5 分钟，只需查配置）**：拿一个 S 档技能，从技能开始到第一帧特效出现的**前摇**就是它的流式窗口 $T_c$，特效资源总字节是 $r_{new}$，**两者相除就是"要多少带宽才不卡"**——这个数在你项目里应该从来没算过。

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
| [[Resampled Importance Sampling]]、[[Reservoir Sampling]]（★ 2026-10-14 新增 / **10-15 更新**） | 读者专题报告已提供概念层入口（"先从易抽取的分布产生候选，再向目标分布重采样"）；[[Talbot — Importance Resampling for Global Illumination (2005)]] 原文已入库——**分层推断为 Normal 入口**；**10-15：[[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR DI]] 原文入库 + 6 张论文原图**——"校正因子 → 蓄水池 W"对读材料齐备；方差公式推导与 \|Z(y)\|/M 定理另列 Hard |
| [[ReSTIR]]（★ 2026-10-15 新增） | 三组件分工表 + 1982→2026 谱系 + 读者框架四阶段逐项咬合表已入库——**结论层 Normal**（读 ReSTIR DI 结论层 ≈30 分钟即覆盖）；推导层（\|Z\|/M 偏差定理、GRIS 一般化）列 Hard 后置 |
| [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS]]（★ 2026-10-18 新增） | 采样线理论源头：balance/power 启发式 + 定理 1/2——**结论层 Normal**；§5.4 近似权重许可（ReSTIR O(M) MIS 的理论前提）与 §5.2 权重/样本数不对称（预算论证样板）为工程接口；三件旧工具集齐（WRS 1982/RIS 2005/MIS 1995）；GRIS 推导层（Hard）前置链闭合 |

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
7. **光照框架线（2026-10-14 新增 / 10-15 更新）**：读者专题报告《现代实时光照的共同设计框架》把 MegaLights / Lumen / ReSTIR / RTXDI 组织成四阶段阅读框架；库内三段原文现已齐备（[[Talbot — Importance Resampling for Global Illumination (2005)|RIS 2005]] / [[Schied — Spatiotemporal Variance-Guided Filtering (2017)|SVGF 2017]] / **[[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR DI 2020]]（10-15 入库，含 6 张原图）**）。推荐闭合动作：**ReSTIR DI 结论层 ≈30 分钟**（四问自测见 Daily 10-15）+ 报告自带的"两盏灯采样实验"（≈1 小时）；可选 RTXDI `Reservoir.hlsli` 15 分钟只读。**读者已按 STE 重写框架文档（10-14）——对读时以新版表述为准**。

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

## 已读论文账本（31 篇）

以下清单逐项来自 frontmatter 核对（27 篇 = 2026-10-05 校正；**+3 篇 = 2026-10-07 更新**；**+1 篇 = 2026-10-13 更新（WorldCrafter）**）。**不推定阅读完成日期**；保留各篇自己的难度分层。以后有新标记时，以论文文件当前属性为准并刷新本表。

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
| [[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory]] ★ 10-13 更新（读者标记） | Normal（世界模型线） |

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
