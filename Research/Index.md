---
type: moc
title: "Game Development Research — Index"
created: 2026-09-07
---

# Game Development Research — Index

全球游戏研发技术情报系统 + 个人认知知识库。

## 入口

- **今日**：[[2026-10-28]]（昨日：[[2026-10-27]]）
- **本周综合**：[[2026-W43]]（覆盖 **10-19 ~ 10-25**，✅ 2026-10-25 收官——主线：**把三样东西变成"可数"**（正确的性质 / 画面的语义 / 成本的对象）；20 Papers 含 8 经典 + 5 新概念 + 64 图；上一周 [[2026-W42]]）
- **首月月报**：[[2026-09-Monthly|Monthly 2026-09]]（覆盖 9-7 ~ 9-30，✅ 2026-09-30 产出）
- **本月雷达**：[[2026-09]]（✅ 2026-09-30 月结刷新）
- **认知模型**：[[Personal Knowledge Model]] ✅ 校正于 2026-10-05 / **更新于 2026-10-28（资料侧：已读账本维持 40 篇——连续第 8 日无读者新动作；新增判断：库内"预算"语言从六种计价对象扩为**七种**（容量 / 成本 / 精度 / 时间 / 状态-记忆 / 像素覆盖 / **时间采样**），并得出"**分配方式 > 分配额度**"这条可与"换单位"并列的第二判据——RADIANT 3.3 dB vs 0.04 dB、GaussianSpline +0.91 dB vs 0.00 dB，两次比值都在两个数量级，见 PKM 2026-10-28 块 / [[Budget Units]]）**——推送按校正结果组织

## Papers（157）

| 论文                                                                            | 分级  | 用户水平        | 与你相关度        |
| ----------------------------------------------------------------------------- | --- | ----------- | ------------ |
| [[Learned Motion Matching (Holden 2020)]]                                     | S（基准） | Normal→Easy | **最高（MM 桥接材料）** |
| [[LightOpt — Lights Optimization for Real-Time Rendering]]                    | S   | Normal      | **最高**       |
| [[DLSS 5 — Generative Neural Rendering]]                                      | S   | Normal      | 高            |
| [[MotionBricks — Scalable Real-Time Motions]]                                 | S   | Hard        | 中（时序锚点）      |
| [[Magpie — Real-Time World Renderer for Interactive Games]]                   | A   | Normal（接口层） | 高（战略性）       |
| [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]]              | A   | Normal      | 高（方法论+生产）    |
| [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]] | A   | Normal      | 中高（方法论）      |
| [[Lightweight Attention-based Indirect Illumination (AMD)]]                   | A   | Hard        | 中            |
| [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]         | B+  | Normal      | 中高（预算语言）     |
| [[Inverse Rendering for Modeling with Line Primitives]]                       | A   | Hard        | 低（Watchlist） |
| [[UniMate — One Unified Model to Animate Diverse Skeletons]]                  | A   | Hard        | 低（Watchlist） |
| [[LLM-Guided RL for Adaptive NPC Behavior]]                                   | B   | Normal      | 低            |
| [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]    | B   | Hard        | 低            |
| [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]]   | B   | Normal      | 低（Watchlist） |
| [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]] | A- | Hard（桥极短） | 中（概念启发） |
| [[FlexMoGen — Flexible Motion Generation from Language and Style References]]                  | A   | Normal      | **高（MM 瓶颈之桥）** |
| [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]]                      | B+  | Normal      | 高（预算方法论）     |
| [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]]                 | A-  | Hard        | 中（管线痛点）      |
| [[RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting]]       | B+  | Hard        | 中（灯光线对照）     |
| [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]    | A-  | Normal（系统侧） | 中高（系统方法论+新阵营锚点） |
| [[SceneHI — High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination]]   | B+  | Hard（概念可读） | 中（生成式资产烘焙对照）   |
| [[MOONWALK — Intent-Evidence-Action Alignment for Animation VFX Review]]                     | B   | Easy（流程语言） | 中（评审工作流模板）     |
| [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]                         | S（经典） | **Easy（10-05 确认）** | **高（当前学习焦点）** |
| [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] | S（经典） | Normal | **高（PBR 环境光半边 / 移动端 GI 根源）** |
| [[2026-09-14-Gaussian Light Transport]]                                                      | A-  | Hard（概念桥短） | 高（GI 显式路线对照）     |
| [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]]                  | A-  | Hard        | 中（零示范范式）        |
| [[Kajiya — The Rendering Equation (1986)]]                                                    | S（经典） | Normal | **高（渲染谱系最大锚点 / PBR 容器侧）** |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]]                             | A-  | Hard（取两个认知点即可） | 中（方法论对偶样本）     |
| [[2026-09-15-UniMo — Unifying Human and Animal Motion Generation]]                            | B+  | Hard（取一句话即可） | 中低（数据价值 > 方法价值） |
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]                   | S（经典） | Normal（研读中） | **最高（GGX 论文：你用的 D 与 G 出自这里）** |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]                       | A-  | Hard（D/G 部分 Normal 可读） | 中高（Smith 假设 → height-field 极限） |
| [[2026-09-16-ESG — Generating Physically Consistent Dynamic 3D Scenes from Text]]             | B+  | Normal | 中（架构模板 > 技术） |
| [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]                  | S（经典） | Normal（研读中） | **最高（你实际在用的那个 F；D·G·F 三因子至此全部有出处）** |
| [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]]                        | A-  | Normal | 高（发片→发丝自动升档；分档阶梯首次可上可下） |
| [[2026-09-17-EMODY Flow — Emotion-Aware Audio-Driven Full-Body Motion Generation]]             | B+  | Normal（只取一条诊断） | 中（"弱条件被抑制"修法可迁移） |
| [[Karis — Real Shading in Unreal Engine 4 (2013)]]                                            | S（经典） | Normal（研读中） | **最高（PBR 工程侧收口：UE 里那些常数从哪来 + split-sum）** |
| [[Marschner — Light Scattering from Human Hair Fibers (2003)]]                                 | S（经典） | Normal | **高（毛发理论源头 R/TT/TRT；微面 BRDF 的失效边界）** |
| [[2026-09-17-DSD — Diffusion Skill Discovery]]                                                 | A-  | Hard（可只取一个抽象） | 中（技能库宽度；与 [[Motion Matching]] 构成检索对偶） |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] | A   | Normal（概念）+ Hard（推导） | **最高（PBR 最后缺口的理论天花板）** |
| [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]                      | S（经典） | Normal（研读中） | **最高（PBR 最后缺口的工程解：4KB 表补回能量）** |
| [[2026-09-18-LYRIC — Language-Driven Physics-Based Character Control]]                          | A-  | Hard（只取一条架构） | 中（物理动画第四篇：能和物体接触） |
| [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]]                    | S（经典） | Hard（**四条结论可按 Normal 读**） | **最高（能量账本的"真值参照系"；"不可实时"定义了后续所有工作）** |
| [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] | A（经典） | Normal | **最高（本库"研究 → 引擎"距离最短：零新增资源 + 论文自带 GLSL）** |
| [[Reeves — Particle Systems (1983)]]                                                            | S（经典） | **Easy（你的专业域）** | **最高（★ 你五个预算维度中四个的原始出处；库内 VFX 域唯一历史锚点）** |
| [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]                                  | A（经典） | **Easy（机制）/ Normal（成本模型）** | **最高（★ "动态灯光 ≤3/≤2/≤1/0"的原始依据；补齐零覆盖的阴影域）** |
| [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]]        | A-  | Normal（只取一条配方） | 中低（**`freeze-and-distill` 配方第 3 例**；无引擎侧实现） |
| [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space]]                        | A-  | Normal（渲染方程改写层可直接读） | **高（PBR 首次被改写进 VAE 潜空间；"分层收敛"交换层压到"表示"的实例 + 成本转移账本）** ★ 2026-09-21 补录 |
| [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] | B+  | Normal（取三条抽象即可） | 中（**PCG 域 2026 样本：LLM 解析 + 拓扑角色推断 + 检索组装；"生成只做决策层"第 4 例**） ★ 2026-09-22 |
| [[Müller — Procedural Modeling of Buildings (2006)]]                                            | S（经典） | Normal | **高（PCG 域历史锚点：CGA shape / CityEngine 理论基础；"规则必须作用在结构化中间表示上"）** ★ 2026-09-22 |
| [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]]                                  | S（经典/手册） | Normal（结论层）/ Hard（推导层） | **最高（★ 库内两条线"多次散射 + 参与介质"的共同参考层：746 页地图册 + MC 交叉验证；furnace test 的第一个对照基线）** ★ 2026-09-23 |
| [[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory]]         | A-  | Normal（接口层 + 三条抽象） | 中高（**"记忆=预算"第一例；北大 × 腾讯 ARC Lab；隐式记忆 vs 显式 warp 的 21.7× 成本差**） ★ 2026-09-23 |
| [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]]         | B+  | Normal（取三条抽象即可） | 中（**"结构锚定"母题第三例；"稠密化不够、有界才是关键"的最干净量化判据**） ★ 2026-09-23 |
| [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] | A-  | Normal（GS 之上的新研究） | **高（"取消排序"第四条 GS 路线；成本函数置换样本：全管线 2.1×、去噪 +1 ms 常数；"无干净引导时 trust 必须靠学"）** ★ 2026-09-24 |
| [[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation]]             | A-  | Normal      | 中高（**分件 = 意图条件生成**；腾讯 Visvise / SIGGRAPH Asia 2026；**粒度可点单**；"生成只做决策层"第 5 例） ★ 2026-09-24 |
| [[Hill — A Multi-Faceted Exploration (2018-2019)]]                                            | A（经典） | Normal      | **最高（能量账本谱系缺失中段：Fms 修正 → Schlick 线性性 2D LUT → 单贴图 3 MAD；Turquin TR / Lagarde18 两处待核实同日结案）** ★ 2026-09-24 |
| [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]                        | S（经典） | **Normal（实时侧源头）** | **最高（★ 毛发实时侧源头：texel 三元组 / sin(t,l) 漫反射 + 圆锥高光 / "渲染时间与几何复杂度解耦"= 预算与几何解耦的最早表述 / "texel↔几何切换"= 分档原则的 1989 版本）** ★ 2026-09-25 |
| [[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition]]                  | A-  | Normal      | 高（**碰撞代理的凸分解全 GPU 常驻：~80–100×，单网格十几秒 → 0.2 秒级；"overnight bake → 交互式"；"取消式优化"第二例：取消 kernel 边界与 host 同步**） ★ 2026-09-25 |
| [[2026-09-24-OREO — Fidelity Alignment in 3D Generation via On-The-Fly Rendering-Editing Optimization]] | B+  | Normal      | 中（**3D 生成器后训练：Render-Edit-Optimize 循环；消融表四种监督方式三种失败（on-policy / latent 对比 / 显式伪目标为唯一成立组合）；界定 DMD 的适用边界**） ★ 2026-09-25 |
| [[2026-09-01-ToCo-Mesh — Topology-Consistent Dynamic Mesh Reconstruction via Adaptive Tessellation and Surface-Aligned 2DGS]] | A-  | Normal      | 中高（**动态网格重建：固定拓扑 + 误差驱动 split/merge + Surface-Aligned 2DGS；USTC / SIGGRAPH Asia 2026；"拓扑一致性 = 可动画的货币"；13K 顶点 vs 729K 高斯**） ★ 2026-09-26 |
| [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] | A（经典） | **Normal（实时工程侧）** | **最高（★ 实时毛发工程源头：发片模型 + 两 lobe 移位近似（扰动切线）+ "取消运行时排序"（静态索引缓冲 + 四趟渲染 + early-Z）——"取消式优化"的 2004 年版本；毛发线三节点齐全、12 条自测）** ★ 2026-09-26 |
| [[Parish-Müller — Procedural Modeling of Cities (2001)]] | S（经典） | **Normal** | **高（★ PCG 域第二锚点 / CityEngine 最早出处：扩展 L-system（参数外移，9 条规则）+ self-sensitive（树状→网络）+ LOD = 迭代深度；与 Müller 2006 同一作者团队的接力关系；13K 建筑 ≈ 10 分钟）** ★ 2026-09-27 |
| [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] | A（经典） | **Normal（实时侧多散射）** | **最高（★ 毛发谱系"多散射"节点落库：全局/局部分解 + "一条原型路径代表全部路径"（透射连乘 + 方差求和）+ fback 材质属性 + 三档实现；7.8h → 5.2min → 14fps 且零调参；与 PBR"补能量 vs 推输运"跨域同构）** ★ 2026-09-28 |
| [[2026-09-25-DiffusionShadow — Diffusion-based Shadow Caching for Neural Volume Rendering]] | B+  | Normal（只取一条抽象） | 中（**"预计算数据的第四种存法"：生成式记忆（扩散模型作压缩缓存）；立场声明"不泛化、要记忆"；UC Davis × Argonne / SciVis 领域**） ★ 2026-09-28 |
| [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting]] | A-  | Normal      | **中高（★ "初始化解降级"：复制动作/ HumanIK 输出只是起点、目标由四能量定义；Autodesk；欠定问题的正确形式化）** ★ 2026-09-29 |
| [[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering]] | B+  | Normal      | 中（**"优化到眼睛"：后处理+显示光学纳入目标；运行时状态（含功耗预算）条件化生成；SIGGRAPH Asia 2026；"档位条件化"第三线索**） ★ 2026-09-29 |
| [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]]                | S（经典） | **Easy（机制）/ Normal（成本模型）** | **最高（★ 分档的祖先：LOD / 可见复杂度固定上限 / working set / 递归下降；"预算起源三件套"最上游；✅ 9-30 扫描件补核结清——中心加权细节 / 运动自适应细节 / 排序 m log₂m→≈pm / LOD 构建管线）** ★ 2026-09-29 |
| [[Vicini — Path Replay Backpropagation (2021)]]                                                | S（经典） | **Hard（推导层）/ Normal（结论层 4 条）** | **最高（★ 可微渲染"记忆与时间轴"起点：种子重放 + 局部 AD = 常数内存 + 线性时间；21.6GB→0.3GB；"用确定性换存储"的 2021 形态；[[LightOpt — Lights Optimization for Real-Time Rendering\|LightOpt]] 前置）** ★ 2026-09-30 |
| [[2026-09-26-Constant-Memory Differentiable Light Tracing]]                                    | A-  | **Hard（推导层）/ Normal（结论层）** | 中高（**光源侧补全：ResLRB / LRB-3-pass；"重放边界的判据 = 路径-输出连接结构（1:1 vs 1:k）"；体检新分支"它能不能不存？"**） ★ 2026-09-30 |
| [[Kovar — Motion Graphs (2002)]]                                                               | S（经典） | **Normal（图时代框架 / 过渡三问）** | **最高（★ 动画线首个经典锚点：图时代开山——相似度矩阵局部最小值 = 天然拼接点 / 固定 ≈1/3 s 混合窗 / SCC 剪枝 + 分支定界；move trees 自动化；"观察频率决定验收严格度"的 2002 表述）** ★ 2026-10-01 |
| [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph]]             | A-  | Normal      | **中高（★ "过渡长度 = 图路径"：聚类转移图最短路定长度 + 引导式生成；**最小转移长度 0.3 s 沿用 Kovar 2002**；14.5 ms 单次缝合在 60 FPS 帧预算内；"结构交给搜索、细节交给生成"）** ★ 2026-10-01 |
| [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering]]               | A-  | Normal      | **高（★ "取消排序"线第 5 节点：随机透明双流按成本路由 + 跨场景时空重建；桌面 2.3–2.7× 3DGS；**移动端 73.2 FPS@540p 含 NPU 重建**；"档位 = 采样数（1/4/16-spp）"）** ★ 2026-10-01 |
| [[2026-09-29-Texture Space Material Diffusion]]                                                | B+  | Normal      | 中高（**NVIDIA / nvdiffrec 团队："换空间"家族第 2 例**——材质生成搬进纹理空间（已知投影当归纳偏置）；PBR 五通道直出 + 8K 材质超分；训练规模 ≠ 使用规模） ★ 2026-10-01 |
| [[Wonka — Instant Architecture (2003)]]                                                        | S（经典） | Normal | **最高（★ PCG 立面细节侧锚点：split grammar + 双重控制（attribute matching / control grammar）；"规则选择"问题出处；PCG 四节点齐全 2001→2003→2006→2026；与 2001/2006 同一批人接力）** ★ 2026-10-02 |
| [[2026-09-30-Lens Flare Removal and Reconstruction]]                                           | B+  | Normal（接口层） | 中高（**光学效应题域第一条："两个账本一套表示"——去光晕（清理户）+ 相机锚定 1D 高斯重建（艺术家户）；1D 约束消融 33.27 vs 27.77 dB；RTX 4090 92.7 FPS、合批 +1.8%**） ★ 2026-10-02 |
| [[Keller — Instant Radiosity (1997)]]                                                           | A（经典） | **Normal（结论层）** | **高（★ VPL 起源："光照场 = 一组点光源" + quasi-random walk（确定性无方差）+ jittered low-discrepancy sampling；每盏 VPL 一遍渲染（成本 ∝ 灯数）；与 RSM 合读 = 缓存族两种记账；致谢彩蛋：Stamminger 提供硬件——8 年后他成了 RSM 作者）** ★ 2026-10-03 |
| [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]                                      | A（经典） | **Normal（结论层）** | **最高（★ 缓存族 RSM 侧源头：四缓冲 shadow map + pixel light + 每像素 ~400 固定样本 + 屏幕空间插值；"把成本从'∝ 光源数'改写成'∝ 像素 × 常数'"——MegaLights"每像素光照预算"的 2005 原型；Neural GI 桥具名前置 ✅）** ★ 2026-10-03 |
| [[2026-09-27-Plate-Local River Generation]]                                                     | A-  | Normal      | **高（★ 开放世界水系："确定性 + 地形感知 + 下坡保证 + 无限域"四者首次同时成立；板块双表示 + 懒加载逐板块缓存 + 构造性保证；37.9 ms/10 板块、1.13 µs/样本、0.169 MiB 缓存；诚实反例 0.0065%；PCG 自然地形分支首个样本）** ★ 2026-10-03 |
| [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars]]                   | B+  | Normal      | 中高（**"一套基代表全部身份"：动画 ≈ 共享 blendshape 基的线性组合（含未见身份）；蒸馏为基 + 浅层 MLP、宿主零重训——154→5.5 ms / 42.9 s→16.1 ms（×2,659）/ 手机 60 fps；"神经 → 传统资产形态"翻译样本**） ★ 2026-10-03 |
| [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]                            | A（经典） | **Normal（结论层）** | **最高（★ 缓存族"接收侧"源头 / 辐照度缓存：答案视图无关 + 误差容限 a + 八叉树 + 跨渲染复用；"误差定预算"第一文献——500h→30h；**NRC 2021 直接祖先**（缓存对象未变、介质变））** ★ 2026-10-04 |
| [[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] | S（经典） | **Normal** | **最高（★ 漫反射侧结账：与 GGX+Smith 同源求解（单次散射"最多丢一半"）→ multi=0.1159α；**"specular 的 4 = (L+V=2H·V) 的平方"完整推导**；**UE 的 k=α/2 与 Smith 推导同源**；理想 Lambert 缺 5% = 1.05/π；**原文阻塞 16 天后经新 CDN 解除**）** ★ 2026-10-05 |
| [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD]] | A- | **Normal** | **高（★ GS 的 LOD/预算线开线："误差 ∝ N⁻¹ᐟ²"（Zador 律）+ 因子树（矩匹配聚合 119×）+ 部署时单参数 c 扫连续内存-质量曲线；官方城市数据 71 fps @1080p 全 SH（vs Octree-GS 11.5 fps / 6.2×）；**"已发表压缩器前沿平坦：增益来自冗余、不是容量"**；对偶证书 = **预算可认证性范式**）** ★ 2026-10-05 |
| [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling]] | A- | **Normal（机制层）** | **高（★ 毛发"仿真轴"首节点：神经时间积分器"镜像"经典积分器 I/O；模拟器在环 + 随机视界；3000 股 0.460 ms/帧、动态帧 2000 打满（前作 131）、12 万股免重训；strand-space 消融 0.632 vs 0.342 = "表示即先验"最干净量化）** ★ 2026-10-06 |
| [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model]] | A- | **Normal（推断）** | **最高（★ 动画压缩域开线：两阶段 = 经典决策 + 学习熵编码；0.37×/0.22× ACL 字节（两种契约口径）；"误差界即档位"（0.01–1cm）+ 契约/回退/计数 = 预算可认证性第三形态；ACL 作者邮件 = 引擎三本账一手资料）** ★ 2026-10-06 |
| [[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]] | A（经典） | **Normal（结论层）** | **最高（★ 缓存族第四节点/现代形态：探针网格 × 8×8+16×16 编码 × 每帧 m×n 射线 × 滞后摊销 × Chebyshev 矩可见性；6 ms/frame vs 1 min/frame；"探针数量 > 分辨率"调参次序；"光追补充光栅化"分工原句；RTXGI 产品线起点——**Neural GI 桥经典侧自此完整**）** ★ 2026-10-06 |
| [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] | S（经典） | **Normal（能量页）** | **最高（★ Weta 生产毛发模型：球面高斯卷积重建守恒 M_p（非高斯 + off-specular peak）+ 方位向取消求根改积分 + 任意阶（TRRT ≈ 白发掠射 15%）；"白环境实验"= 毛发版 furnace test；pbrt-v4 采用其 M_p 与色素 RGB；与用户进行中的 Marschner 深读直系对接）** ★ 2026-10-07 |
| [[2026-10-05-Neural Emission Fields — Real-time Rendering of Pre-integrated Neural Emitters]] | A- | Normal | **高（★ "取消运行时积分"：预集成 + 神经场 = 可携带照明资产（平移/旋转/缩放免重训）；全高清 1.2/2.5 ms 恒定、交叉点 267 顶点（Dragon 436k 顶点上解析法 ~1952 ms）；对照 oracle 代理需 ~734 spp；预计算存法谱系第 ⑤ 种；MegaLights 的"零采样"互补面）** ★ 2026-10-07 |
| [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering]] | A- | Normal | **高（★ GS 排序线第 6 节点（收口）：训练方差正则 + 时空重采样"生干净"；1 spp 31.44 dB（+13.1 dB vs 前作）、收敛极限与排序版 L1 < 10⁻⁴；3DGS 原班人马（Kerbl/Kopanas）+ SVGF 血统（Wyman）；"档位 = 采样数"每档可交付）** ★ 2026-10-07 |
| [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation]] | B+ | **Hard（取三条抽象）** | 中（**"混沌判据"：混沌动力学上确定性回归收敛到非物理平均值、扩散才建模分布；时空 VAE 78× 压缩 / 2.48 mm；可微逆问题 40 秒级（vs DiffIPC 26 分钟）**） ★ 2026-10-07 |
| [[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]] | A（经典） | **Normal（生产化页）** | **最高（★ WDAS 生产毛发模型：near-field 取消 70 点求积（把宽度积分外包给路径追踪器；毛发着色 ~20× / 整帧 >10×：869→76 min）+ logistic 方位分布 + 第四叶折无穷阶（furnace test 通过）+ 感知均匀六参数；**毛发谱系六节点闭合（1989→2016）**；⚠️ 更正此前 "Pixar" 误记——实为 Walt Disney Animation Studios）** ★ 2026-10-08 |
| [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting]] | A- | **Normal（结论层）** | **高（★ 面部线首节点："四个不需要"（无 cage / 预计算坐标 / 解码器 / 全局求解）的点基双因子——身份权重算一次 × 表情控制点每帧 = 矩阵乘积；183 帧端到端 1 秒（对手 25 s / 17 s）；30 人用户研究双第一；风格化头泛化；GALA"分解家族"、ReFM"重定向对"）** ★ 2026-10-08 |
| [[2026-10-07-DynaConTalk — Wavelet-Constrained Diffusion for Long-Form and Controllable Holistic Co-Speech 3D Motion]] | A- | **Normal（结论层）** | **高（★ "换空间"家族第 3 例（小波系数空间）：同位姿误差下时间抖动 −89.4% / 分布距离 −92.1%；FGD 1.75（对比最好 3.56）；matched-noise 三合一（延续/修复/参考）；自带 FGD 频带审计（99.51% / 9.14%）；代码已放出）** ★ 2026-10-08 |
| [[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing]] | B+ | **Normal（地图层）/ Hard（推导层）** | **中（★ Bender 组（RWTH Aachen）EG 2026 教程：能量 / 约束（PBD·XPBD）/ 粒子（SPH）/ 欧拉-混合（MPM）四族全景 + 耦合 + 框架选型 + ML 趋势——物理域"查阅式地图"，地位对照 [[Real-Time Rendering]]）** ★ 2026-10-08 |
| [[Turquin — Practical Multiple Scattering Compensation for Microfacet Models (2019)]] | A（经典） | **Normal** | **最高（★ 路线⑤原始文献——五条补法全节点闭合；F_ms 四级简化链 → F₀；"gain to the closure"（采样/PDF 全复用）；导体 7×/电介质 15×；Unity HDRP 与 Filament 采用；⚠️ 修正"只 IBL"旧记录——方法不限光源）** ★ 2026-10-09 |
| [[2026-10-08-Neural Caching of Prefiltered Radiance for Specular Lighting]] | B+ | **Normal（结论层）** | **高（★ 缓存族镜面支线开线："视图无关"前提被正面打破——反射方向参数化 + roughness 预滤波 + split-sum 重建；同一张 BRDF 积分图第三次委派）** ★ 2026-10-09 |
| [[2026-10-08-Tabula Rasa — Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation]] | B+ | **Normal（结论层）** | 中高（**"噪声 = 接口契约"**：单位方差给扩散模型 / 时空相关由光传输诱导；1024² 帧 2.44 ms；三行代码；UCL × Meta（Hery/Marshall））★ 2026-10-09 |
| [[2026-10-07-emg2face — Expressive Facial Animation with High-Density Surface EMG]] | B+ | **Normal（结论层）** | 中高（★ **面部域捕捉层首节点**：HD-sEMG 遮挡免疫（眼部 r 0.79）、0.43 mm、眼罩实验暴露光学"幻觉式失败"；非因果 1.3 s；UBC × Google × Imperial）★ 2026-10-09 |
| [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]] | S（经典） | **Normal（结论层）** | **最高（★ 几何建模域第一个经典锚点：三条细分规则统治全部 DCC / 游戏 / 汽车曲面语义；"发明者无法证明、合作者实现验证"；Coons 命名奇异点；规则先凭审美（"too pointy"故事）；论文自认三个开放问题——与今日 SubDGuide 构成"正问题↔逆问题"48 年弧线）** ★ 2026-10-10 |
| [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction]] | B+ | **Normal（结论层）** | **最高（重拓扑逆问题："planner 判断 / 工具构建 / 验证回滚"两权分离——"永不生成顶点/连接"；Chamfer-L1 0.641%→0.443%；Mercedes-Benz × TU Darmstadt；与 Catmull-Clark 1978 构成 48 年正逆对话）** ★ 2026-10-10 |
| [[2026-10-08-RiCo — Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning]] | B+ | **Hard（推导）/ Normal（结论层）** | 中高（**物理接触维度开线**："接触是局部的"写进网络结构——跨物体推理限定在稀疏局部接触邻域；接触保真 ΔPTR 11.0% / ΔMPD 2.22 mm；**"指标看不见的价值"判据**）★ 2026-10-10 |
| [[2026-10-08-PCAsplat — Gaussian Splatting with Local PCA Regularization]] | B+ | Normal（结论层） | 中高（**GS 几何质量线**：视角无关局部 PCA 正则修 floaters——"看不见的高斯也有监督"；下游 Poisson 重建 / 分割免修直用）★ 2026-10-10 |
| [[2026-10-08-Sensitivity as an Arbitrary Output Variable for Differentiable Rendering]] | B+ | **Hard（推导）/ Normal（结论层）** | 中（**可微渲染"输出/检查层"开线**："导数是一等渲染输出" + deferred shading 式"算一次看多处" + 固定/自由双相机分离；SIGGRAPH Asia 2026 TC；同作者双篇之一）★ 2026-10-10 |
| [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)]] | S（经典） | **Normal（结论层）** | **最高（★ 细分"求值线"：递归展开为矩阵幂 → 特征基直接求值——"推翻一个普遍信念"；任意 (u,v) 成本 ≈ 双三次样条；七共享函数 / n=4 时=幂基；实现 = EIGENSTRUCT 一张表 + 两个函数；2005/2018 奥斯卡）** ★ 2026-10-11 |
| [[DeRose — Subdivision Surfaces in Character Animation (1998)]] | A（经典） | **Normal（结论/生产层）** | **最高（★ 细分"生产化线"：半锐折痕 0→∞ 连续谱（混合细分"前 s 步锐→后光滑"）+ 布料能量三件套 + 粗化层级碰撞 + (x,y,z,s,t) 五维细分标量场 → 《Geri's Game》；**库内对 TA 最友好的经典**）** ★ 2026-10-11 |
| [[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection]] | B+ | Normal（结论层） | 中高（★ **保留端分配**：配额平滑 + 联合删除效应 → QUBO/相干伊辛机；"分域 → 配额（反独占）→ 域内联合选择"三步骨架 = SABC 配比工作的算法版；ICLR 2027 在审）★ 2026-10-12 |
| [[2026-10-09-Sonic-AMG — Streaming Human Motion Generation from Spatial Audio]] | A- | Normal（结论层） | 高（★ **流式空间音频→全身动作首例**：what/where 双 token 分工 + rehearsal training + spatial feedback——"流式两错位"检查单入库；Brown × MIT × Yale × HKU；实时在线生成）★ 2026-10-12 |
| [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction]] | B+ | Normal（结论层） | 中高（★ **生成端分配**：内容自适应（复杂区密集小高斯 / 平坦区放大尺度调制）+ "用尺度换数量" + 推理时稀疏度旋钮；NeurIPS 2026 / 上海交大）★ 2026-10-12 |
| [[2026-10-09-H2-Ray — A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering]] | B+ | Normal（结论层） | 中（★ **GS 光追架构首开**：负载不均 / 遍历冗余 / 异构求交三难 + LB-SBVH / RCPR / UIE 三招；3.77×~9.26× vs RaySplats；"高斯的统计 ≠ 三角形的统计"）★ 2026-10-12 |
| [[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)]] | A（经典） | **Normal（结论层）** | **最高（★ 三角分支定义者：分裂 1→4 + 两条平均 + μ₃²=μ₂"平方关系反推权重"；四阶箱样条规则区；与 Catmull-Clark 构成四边/三角对偶；⚠️ 原文直链失效——核证清单入笔记）** ★ 2026-10-12 |
| [[Stam — Evaluation of Loop Subdivision Surfaces (1998)]] | A（经典） | **Normal（结论层）** | **最高（★ 三角版求值："同样的机器换坐标系，细节不同而已"；12 点四阶箱样条 + 3/4 免费午餐 + 无限瓦片；K=N+6 / 6 个共享特征基；lpdata50.dat + lptest.c 公开；N=3 Jordan 特例）** ★ 2026-10-12 |
| [[Stam — Stable Fluids (1999)]] | A（经典） | **Normal（结论层）** | **最高（★ 流体域开域：无条件稳定求解器（拉格朗日回溯 + 隐式扩散 + 投影）——"不管时间步多大都不爆炸"；"感知优先"设计哲学原话出处；傅里叶"换空间"彩蛋；Maya Fluid Effects 理论来源；2005/2008/2019 三座奥斯卡谱系）** ★ 2026-10-13 |
| [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS]] | B+ | Normal（结论层） | 中高（★ **分配四端之"生长端"**：误差驱动加密（梯度排名 × 低置信度 + 贡献度重定位）——"预算跟着缺口走，不跟着现状走"；"先跑的不一定先到"；24 场景全预算超 MCMC）★ 2026-10-13 |
| [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes]] | B+ | Normal（结论层） | 中（★ **水线第二节点 / 场景动态化"注入派"**："三不"（无捕获/无重训/无模拟）+ 2 MB 可携带运动场 + 无缝循环；"感知优先"2026 样本）★ 2026-10-13 |
| [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting]] | B+ | Normal（结论层） | 中高（★ **重定向三节点之优化式**：优先级层级取代加权求和（空空间投影 + 误差向量保护）；"重要的东西用约束保护"；真机验证 + 下游 RL）★ 2026-10-13 |
| [[Talbot — Importance Resampling for Global Illumination (2005)]] | A（经典） | Normal（结论层） | **最高（★ 采样端开域：RIS——"ReSTIR 的理论前置"当场结清；校正因子（reservoir W 的祖先）/ 方差劈分 / 等时间准则 M=TY/TX；直接光 10–70%）** ★ 2026-10-14 |
| [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]] | A（经典） | Normal（结论层） | **最高（★ 重建端开域：1spp → 10ms 实时重建；时域累积 + 方差引导 à-trous + 三条边缘停止；"可信度旋钮"；库内降噪条目的共同祖先（Wyman 血统））** ★ 2026-10-14 |
| [[2026-10-09-MeshRepair — Autoregressive Topology Repair for Generated Artisan Meshes]] | B+ | Normal（结论层） | 中高（★ **AI 资产返修工位**："编辑序列而非新网格"（未触及面原样通过）；"quad 率 = 下游硬约束"；SubD 链前置节点）★ 2026-10-14 |
| [[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology]] | A- | Normal（结论层） | 高（★ **动捕 ⇄ 重定向边界合并**：可观测性分析 + 可靠性门控（"能测的用几何、测不到的交给学习"）；A100 51+ FPS，代码开源；Motion Retargeting 捕捉侧节点）★ 2026-10-14 |
| [[2026-10-10-TETRIS-4DGS — Efficient Post-Training Compression of 4D Gaussian Splatting]] | B+ | Normal（结论层） | 中（★ **4DGS 部署端**：后训练 + codec 复用 + 重要度码率；秒级 40×；"翻译给旧工具"接口复用）★ 2026-10-14 |
| [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)]] | A（经典） | **Normal（结论层）** | **最高（★ ReSTIR 家族定义文献：流式 RIS + 蓄水池可合并时空复用 + 多源 PDF 的 O(M) MIS 无偏化；"过滤 PDF 而非颜色"；340 万动态灯 50 ms；Talbot 2005 对读位当日结清；读者框架采样+复用+失效三阶段原文锚齐备）** ★ 2026-10-15 |
| [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)]] | A（经典） | Normal（结论层） | **最高（★ 家族中段补齐：候选从光源点改表面点；Jacobian 修正（方向采样系特有）+ 样本验证（"复用必须自带失效条件"第三例·主动重验证级）；UE4 内 8–18 ms，MSE 9.3–166×；48 B/蓄水池、25% 像素跑多跳——预算劈分两样本）** ★ 2026-10-16 |
| [[2026-10-13-Image Diffusion Models Are Good 3D Human Motion Generators]] | A- | Normal（结论层） | 高（★ **"不动数据、动表示"跨骨架第二答案**：拓扑感知 joint-time 图像 + 双潜流 SiT；HumanML3D+KIT-ML 单模型不 retarget；表示兼容=预训练继承权）★ 2026-10-15 |
| [[2026-10-14-GLoD-LVSM — Learning Geometry-Grounded Continuous-LoD Scene Representations for Large View Synthesis Models|GLoD-LVSM]] | A- | Normal（结论层） | 高（★ **引擎三件套（LoD/预算/剔除）回归神经表示**：连续场锚点数=预算输入（token −33.4% ↔ 1.23×，放宽档 −85.5% ↔ 3×）；几何接地→视锥剔除可用；涌现深度+锚点贴表面；与 Budgeted-GS 构成 LVSM/GS 双子）★ 2026-10-16 |
| [[2026-10-13-Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation]] | B+ | Normal（取三条抽象） | 中高（★ **"生成只做决策层"第 6 例**：概念库人机协同养成 + concept-graph + 求解器；PCG 规则获取 20 年演化收口：手写→学习→对话）★ 2026-10-15 |
| [[2026-10-14-Q-MAGIC — Quad-only Mesh Autoregressive Generation|Q-MAGIC]] | B+ | Normal | 中高（★ **网格资产线"生成侧出四边形"工位**：奇点优先序列化（约束最紧先走）+ 环形相对偏移 tokenization；MeshRepair（修复）→ 本文（生成）→ SubDGuide（SubD）→ CuACD（碰撞）四工位闭合）★ 2026-10-16 |
| [[Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)|GRIS — Foundations of ReSTIR]] | A（经典） | Normal（结论层） | **最高（★ 家族统一理论：RIS 推广到"相关样本+未知 PDF+跨域"——M 从样本计数改置信权重，M-cap 成理论必需（图 2：无偏但不收敛两像素反例）；canonical 占比 \|R\|/M≥γ 是 O(1/M) 收敛条件；ReSTIR PT 1080p 40–80 ms；"复用必须自带失效条件"账本第 4 例·理论判据级）** ★ 2026-10-17 |
| [[2026-10-15-DisplacementQueries — Geometric Queries on Tessellation-Free Displaced Surfaces|DisplacementQueries]] | A- | Normal（结论层） | 高（★ **位移曲面"免细分"路线补齐查询能力**：一阶纹理空间层级（仿射高度平面+保守残差）——求交之外解锁表面采样/最近点/specular 搜索；内存省 ≥55.4× GM vs FCPW 预细分（速度换 7.48×）；Nanite 细分路线的对照证据；位移发光体采样直接可搬 VFX）★ 2026-10-17 |
| [[2026-10-15-LinfCub — Compression of Uniformly Distributed Unit Quaternions|LinfCub]] | B+ | Normal | 高（★ **四元数压缩第一次有解析误差座次**：ℓ∞cub（八面体+单参数立方 warp）最大误差最小；48 bit/动画关键帧、32 bit/切线帧、30 bit/GS 旋转（24 bit≈Exp 30 bit）——动画资产位宽预算维度的理论依据）★ 2026-10-17 |
| [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS — Optimally Combining Sampling Techniques]] | A（经典·历史重要性 5） | Normal（结论层） | **最高（★ 采样线三件旧工具就此集齐：multi-sample 模型 + balance/power 启发式 + 定理 1/2——不用知道 f 就 provably good；β=2 是 30 年事实标准；§5.4"近似权重仍无偏"= ReSTIR DI O(M) MIS 的理论许可；§5.2"坏权重任意坏、坏样本配比最多差 n 倍"= 预算论证样板；4 图原图提取）** ★ 2026-10-18 |
| [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]] | A（经典） | Normal（结论层 ≈25 分钟） | **最高（★ Radiosity 源头 / 烘焙法理起点："视角无关⇒可预计算"；form factor 三约束 + 能量矩阵 B=E+ρH；红蓝盒 color bleeding + 实体模型实拍验证——读者 Lightmass 专题红墙实验的 40 年前原型；GI 概念页谱系表第 3 行落位，全表行行有节点；3 图原图）** ★ 2026-10-20 |
| [[2026-10-19-PhysSkin++ — Real-Time and Generalizable Physics-Based Animation via Self-Supervised Neural Skinning\|PhysSkin++]] | A- | Normal（系统侧 ≈20 分钟） | 中高（★ CVPR 2026 Highlight 期刊扩展：前馈蒙皮场自编码器免逐对象训练 + PISSL 稳定化 + rig 互补弹性响应；每步 12–14 ms 与网格规模解耦；mesh/点云/3DGS 通用绑定；"预算决定表示精度"第 5 员——handle 数 m 旋钮；代码开源）★ 2026-10-20 |
| [[Kelemen — Coupled Specular-Matte BRDF Model with Importance Sampling (2001)]] | A（经典） | Normal（研读中·对接 CT 焦点） | **最高（★ PBR×采样双线交汇：G 项简化"分母消失"（1/(1+LV)）+ D·F 合并"密度×一维表" + matte 耦合 = Kulla-Conty 补偿公式代数雏形——能量补偿路线③源头；Ward 密度闭式逆采样 + 近最优权重；附录含 CT 完整光子概率推导（半程向量 Jacobian 1D 版）；"Kelemen 给公式 → Kulla-Conty 给工程 → Heitz 给真值"；8 图原图）** ★ 2026-10-19 |
| [[2026-10-16-Structural Affinity — An Algorithm for Coherent Cage-Based Shape Design\|Structural Affinity]] | A- | Normal（结论层） | 高（★ 变形域"正则进参数化"：双笼 min-cap 亲和矩阵，O(N+L) 传播 + 可逆性=表达力不减；"参数化承担 93–98% 正则"；"预算决定表示精度"第 4 员——α/d 是变形预算旋钮）★ 2026-10-19 |
| [[2026-10-16-Scene-Designer — Learning Design Intelligence for Agentic 3D Scene Generation\|Scene-Designer]] | B+ | Normal（取三条抽象） | 中高（★ "设计智能 = 会设计 + 会评判"：受控降级造偏好排序 + 轨迹压缩 110→30 步 + Scene-GRPO 双局部奖励；与 Co-DSL 构成 LLM-PCG 架构分岔（模型化 vs 资产化）；全开源）★ 2026-10-19 |
| [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)]] | A（经典·历史重要性 5） | Normal（结论层 ≈30 分钟） | **最高（★ 烘焙线第二节点 / "烘算子 vs 烘答案"的开篇：换灯只需一次点积，成本与灯数灯尺寸无关；三档价目 DU 9 / DS 25 / 光泽 25×25——"阴影 +2 阶、光泽 平方"；三条锁：几何静态/运动刚性/光照低频；GI 谱系表第 4 行落位；4 图原图）** ★ 2026-10-21 |
| [[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation\|InfiniDream]] | A- | Normal（结论层 ≈20 分钟） | 中高（★ **"不要让 VLM 写程序化代码，要让它改"**：AST 深度 2 依赖闭包 = 可编辑表面 + Tweak/Create/Promote 三策略；确定性视锥网格放置 + 五道闸门（不可读率 50.9%→7.8%）；池化 N→K；**放置是镜头优先、不是玩法优先**）★ 2026-10-21 |
| [[2026-10-19-Wan-Alpha++ — Animating a Single RGBA Asset into Controllable Transparent Video Effects\|Wan-Alpha++]] | B+ | Normal（结论层 ≈15 分钟） | 中（**素材侧**：alpha 一等公民的透明视频生成——TABD 符号掩码 + 透明引导均值采样 + **时间相位重参数化**（换时间原点让首尾帧条件重新可用）；不降 overdraw、不进预算表，只改试错速度）★ 2026-10-21 |
| [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds\|Matrix-Game 4.0]] | A | **Normal（系统侧 ≈20 分钟）** | **最高（★ 世界模型"层次 2"最强样本：可执行 harness 三角色（agent 写规则 / 引擎管状态 / 视频模型管画面）——**规则可执行因此标注可零模型推断**（114,334 段 / 1,712 小时字幕全由模板渲染，不用 VLM）；三阶段因果蒸馏 + 九项推理优化；**4.5→83.4 FPS / 1110→112 ms @5090** ——"实时是系统账，不是模型账"；5090/5080/5070Ti 三档硬件）** ★ 2026-10-22 |
| [[2026-10-21-Kim — RLSplat Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting\|RLSplat]] | A | **Normal（结论层 ≈15 分钟）** | **高（★ GS 预算的"成本侧"：用"分配"替代"裁剪"——GSPO 策略学高斯配额 k~Mult(B,p)（Σk=B）+ 资源线性解码（局部 cross-attn + 层次邻居搜索 O(BK)）；2K 预算 +0.9 dB / 显存 9.6× 少 / 速度 4.1×；**关键诚实点：策略 24.83 vs Sobel 启发式 24.76 dB（仅 +0.07）——学到的是"分配"这个动作**；Voronoi/offset/projection 三正则消融分别 −2.51 / −0.53 / 训崩）** ★ 2026-10-22 |
| [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting\|KoalaGS]] | B+ | **Normal（结论层 ≈15 分钟）** | 中高（★ 连续 LoD 的排序键：用**优化前**的图像复杂度（delentropy 最强，2D ρ≈−0.60 / 3D −0.64）做空间先验 —— "先验在训练前就可得"；p=αe^C̃ 迁移偏置 + complexity dropout δ=0.2；排序键 S=α+λs/e^C̃ + 残差方差正则；100%→0.1% 连续全谱评测；**如实边界：对 MGS 只 +0.12/+1.23 dB，主要增益来自与不排序 MCMC 的对比**；NeurIPS 2026 Spotlight） ★ 2026-10-22 |
| [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning\|RLMuscle]] | A | **Normal（结论层 ≈25 分钟）** | **最高（★ 角色形变的"肌肉层"首个全身样本：1D Hill 解控制（284 肌腱/50 DOF，PPO 共训）+ 3D 体积仿真（194 肌肉/468K 四面体），**共用同一条本构律 → 形变不是拟合，是"产生运动的那股力"**；EMG 先验把冗余解拉回生理（in-band 20%→91%，相关 −0.04→0.97）；四面体 GMG（重心坐标延拓 + Petrov–Galerkin 限制 $RP=I$）把全身压到 **18 ms/帧 @4090**，对照 Houdini Otis 20.8 s/帧；**诚实边界：3D 不在 RL 环内、细长肌肉退出多重网格、游泳用椭球阻力代理**；18 ms = 60 fps 预算的 108% → **它是"真值工厂"不是运行时**；5 图原图）** ★ 2026-10-23 |
| [[2026-10-21-Wang — LiveAct Real-Time Multi-Agent World Models with Heterogeneous Action Interfaces\|LiveAct]] | A- | **Normal（结论层 ≈20 分钟）** | 高（★ **世界模型的"第二个玩家"**：动作 = schema 词（语义）+ 数值（仿射调制）→ 一套接口吃二进制按钮与连续机械臂指令；归因 = **学出来的空间门**，推理时不需要 mask/box/pose/参考图（WanToFight 0.886 但要用 identity crop）；三阶段因果 + DMD：**512×448 @ 17.70 FPS**（Stage 2 仅 1.66 FPS，10.7×）；自建 17,951 轨迹 / 119.54 小时；**如实边界：Act-F1 0.594（四成离散事件没执行对）**）★ 2026-10-23 |
| [[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)\|Kavan 2008 — DQS]] | A（经典·历史重要性 4） | **Normal（结论层 ≈25 分钟）** | **最高（★ 角色形变支线的第一个经典锚点 / 糖果纸的标准答案：LBS 病根 = 正交矩阵对加法不封闭，混出缩放剪切；DLB = 对偶四元数线性混合 + 归一化 → 必然刚体；成本 **36 → 43 条顶点着色器指令**（CPU 3.67→5.07 ms，GPU 197.4 vs 122 / 84.9 FPS）；两处边界：表示不了缩放（两阶段绕）、最短路径会翻面（179°→181°）；新建 [[Skinning]] 四代谱系；4 图原图）** ★ 2026-10-23 |
| [[2026-10-22-Liu — AnyGameRenderer Entity-Consistent Generative Rendering Across Games\|AnyGameRenderer]] | A | **Normal（结论层 ≈20 分钟）** | **最高（★ 生成式渲染第一次给自己立**实体级验收**：纯 gameplay 视频输入（灰盒/图元/语义图/线框均可，**不要 G-buffer/类别标签/low-poly 代理**）+ 六段式**字幕契约**（哪个色块 = 哪个实体；位置/尺寸/时序/**人数**严格保留；**故意保留"列出但未出现"的角色**以逼模型从视频判断存在）；三源配对 10,330 段 / 14.8 h（GEAR 38 款 agent 游戏多流 / UE 双流按失败模式脚本化 / 真实录像 3D 框）；temporal **50.9→62.9**、漏人数变化 34.7→**21.3**、幻觉 52.1→46.0；**公布 GT 天花板：真值视频的幻觉率亦 34.1%，故 62.9 = 可测区间 50.9→80.9 的 38%**；**离线工具**（33B / 20 步 / 16×H200×48h，1344×768）——与 MG4.0 构成"通用 ↔ 实时"轴的两端；3 图原图）** ★ 2026-10-24 |
| [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes\|Kaushik 2026 — Tutte-Like 单射流]] | A- | **Hard（结论层 Normal ≈20 分钟）** | 中（★ **Tutte 定理的第三个对象：从"映射"到"流"**——球面拓扑 + 单源 $z{=}1$/单汇 $z{=}{-}1$ 解离散 Laplace（**任意正权 Laplacian**），梯度转 90° 作向量场，**证明诱导的流单射**（轨迹永不相交）；几何图像 = 轨迹锁死在等值线上；证明支点 = **Campen 组合通量条件 ≡ Gortler 离散 Poincaré–Hopf 指标**（图 6 三行对照）；工程钩子 = 对 Laplacian 边权**可微** ⇒ 可在不离开可行域的前提下优化流向（逼近非单射场 / 满足 10 个稀疏方向约束）；边界：只球面+单源单汇（图 3 反例）、不覆盖全部单射场、优化框架自述"不足以做下游任务"；3 图原图）** ★ 2026-10-24 |
| [[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)\|Lévy 2002 — LSCM]] | A（经典·历史重要性 4） | **Normal（结论层 ≈30 分钟）** | **最高（★ 参数化/UV 整条支线的**工业起点**：三步流水线定型（分割–参数化–装箱）；共形写成复方程 ⇒ u、v 耦合 ⇒ **边界自由**（只需钉 2 点）+ 一次稀疏求解；四条可证性质（唯一解 / 相似不变 / **分辨率无关** / **无翻转**）；**关键边界：局部单射 ≠ 全局单射**——重叠靠 stencil 检测 + 切 chart 补救；装箱率 0.37–0.48 → **0.55–0.60**（轮廓地平线装箱）；质量反转：优化前 3.5 劣于调和 3.2，优化后 1.52 优于 1.65；新建 [[Mesh Parameterization]]；4 图原图）** ★ 2026-10-24 |
| [[Tutte — How to Draw a Graph (1963)\|Tutte 1963 — How to Draw a Graph]] | A（经典·历史重要性 5） | **Hard（原文不可得，二手重建）** | 高（★ **"调和 ⇒ 单射"的第一条定理 / 全书源节点**：三连通平面图 + 凸边界 + 重心 ⇒ 直线无交叉、面为凸、解唯一；图形学版本 = 边界钉圆 + 解 Δu=Δv=0；**凸边界是保证也是枷锁**——LSCM 为自由边界放弃全局单射，Campen 2016 再把它找回来；⚠️ **原文 OUP/Wiley 均 403，本页由 Gortler 2006 + LSCM 引言 + zbMATH 三源重建，附核证清单与"未核实"标记**；同时纠正 LSCM 参考文献 [30] 的年份/标题错误）** ★ 2026-10-24 |
| [[Everitt — Interactive Order-Independent Transparency (2001)\|Everitt 2001 — Depth Peeling]] | A（经典·历史重要性 4，**技术演示文稿·无基准数据**） | **Normal（结论层 ≈20 分钟）** | **中高（★ OIT 的基准解 / 你的 Overdraw 线的历史锚点**：正确半透明的传统解是「让应用排序」，depth peeling 改成**每像素用层号排序**——双深度缓冲 ping-pong，深度测试反向（GREATER）让「比上一层更远的最近片元」成为这一层的可见面；成本 **层数 × 一遍场景**（片元逐层下降、几何不降）；借 shadow map 当只读深度 + `DEPTH_REPLACE` 解精度不变性；**唯一定量结论是「n = 4 通常够用」（无计时、无误差度量）**；四代演进 = 顺序外包给谁（应用 / 像素 / 随机 / 权重）；4 图原图）** ★ 2026-10-25
| [[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)\|Harris 2004 — GPU Gems 38]] | A（经典·历史重要性 4，**书章·非同行评审**） | **Normal（结论层 ≈20 分钟）** | **最高（★ GPU 化的翻译手册 / 你的 Niagara GPU 模拟的历史锚点**：四个翻译（数组=纹理 / 下标=纹理坐标 / for 循环=片元程序 / 迭代反馈=双缓冲）＋ 一次时间步 = 一串**全屏 pass**（5 + 40~80 次 Jacobi，**压力求解吃掉 90% pass**）；**关键判据：半拉格朗日对流天然是 gather 不是 scatter**——"我这个格子取谁的值"是读问题，正好落在 2004 年硬件的能力里；RTT（双纹理、交换近乎免费）vs CTT（单纹理、拷贝 ∝ 尺寸）；边界 = 内部 quad + 四条线段 = 每步两个 draw call；**6× 快于 CPU / 128² 云模拟 >80 迭代/秒 @FX 5950**（均无基线细节，当量级读）；成本式 $\text{成本}\approx W H (5+N_{jac}) B$ → 256²×65×48 B ≈ 204 MB/帧；5 图原图）** ★ 2026-10-26
| [[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination\|RADIANT]] | A | **Normal（结论层 ≈25 分钟）** | **最高（★ 前馈 GI 的完成形态 / "换单位"的第一号证据**：每物体 2048 点 → PointNet++ → **32 个 surface-patch token**，传输序列从三角形 $T$ 压到 $32N$（中位 **3,205 → 320**，$O(T^2)\to O(N^2)$）；**不变形式因子注意力**把 4 维几何不变量 $G_{ij}=(-\|Δ\|^2, m_i^\top Δ, -m_j^\top Δ, m_i^\top m_j)$ 因子化成 query-key 内积（不物化 $O(N^2)$ bias）；point-splat 7×7 免网格 G-buffer + 物体 id routing；**32.22 dB per-image / 27.65 pooled，比 RenderFormer-V2 高 3.3 dB**；500 物体 512² **0.83 s**（8.5× FlashAttention 版 / 22.2× SDPA）；**消融铁证：换单位值 3.3 dB，调 token 预算（8→64）只值 ≤0.04 dB**；诚实边界 = 0.83 s 非实时、只评估到 50 实体、全合成数据；3 图取 2）** ★ 2026-10-27
| [[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation\|UniWorld]] | A- | **Normal（系统侧 ≈20 分钟）** | **高（★ 世界模型的"薄接口"路线：**异构世界源（引擎 / 真实视频 / 合成）→ **图像对齐深度** → 共享 DiT，于是"引擎深度"与"单目估计深度"在接口上等价；**noise-aligned overlapping continuation**（$P_{i,t}(z)=M_i\odot[tH_i+(1-t)\varepsilon_i]+(1-M_i)\odot z$——幂等、$t=1$ 精确还原、不需额外模型评估）+ **rolling depth latent state**（深度只在窗口接缝读一次，窗口内 8 次评估复用）；**深度活跃上界 `4·L_max·S·96` B = 19.05 MiB @960×544，与时长无关**；14.472 FPS @960×544 / **8×H100**；PSNR 13.40；VBench 80.54；**必须说清：消融几乎全跨零（boundary-only +0.006 dB），架构成立但机制归因不成立**）** ★ 2026-10-27
| [[2026-10-26-Liu — PASPA Past as State Present as Attention\|PASPA]] | A- | **Normal（结论层 ≈25 分钟）** | **高（★ 长时程动作生成的当前最优解 / "用错单位"的第一个可量化反例**：NeurIPS 2026，同济 × 东大 × 阿里；BGDN（块级 Gated DeltaNet，块首读保因果 + 可吃 Tensor Core）+ BMHSA（块对角 mask，$O((NK)^2d)\to O(NK^2d)$）；**自相关实测论证**：风格特征 >100 帧仍相关、局部运动 <20 帧衰减——**一条序列两个时间尺度差 5 倍**；BEAT2 FGD **3.71** / SHOW **0.00111**（−43.9%）；**消融铁证：State+State 7.652 比完全不给历史 7.065 还差**——历史给错方式是负收益；600 s 输入 56.0 s 推理、峰值显存 0.36–0.48 GB（**成本回到线性**）；3 图原图）** ★ 2026-10-27
| [[Reyes — The Reyes Image Rendering Architecture (1987)\|Reyes 1987 — 渲染架构]] | A（经典·历史重要性 5，**⚠️ 二手重建**） | **Normal（结论层 ≈30 分钟）** | **最高（★ 架构层第一篇 / 你的 Overdraw 线的史前史**：Cook / Carpenter / Catmull，SIGGRAPH 1987；**八条设计原则**（自然坐标 / 向量化 / 统一表示 / 局部性 / **线性** / 大模型无上限 / 后门 / 纹理）+ **八阶段管线**（bound→cull→split→dice→shade→break→sample-visibility→filter）；**微多边形 = 平面着色四边形、边长约半像素（Nyquist 极限）** ⇒ **着色率第一次与几何复杂度解耦**（Nanite 判据的 1987 版本）；16 子像素 jitter 一套采样点同时解决抗锯齿 / 运动模糊 / 景深；ε 平面免 hither 裁剪与逆透视；**代价：先着色后判可见 ⇒ 结构性 overdraw**（6.8 M 微多边形 / 63 万像素 ≈ 10.8 个/像素 ⇒ **约 2.7×**，⚠️ 本库反推非原文）；定量：1986 圣诞卡 8 小时 CPU、Luxo Jr 724×434 1.5 h/帧、目标 3 分钟/帧；**⚠️ 原文六条路全断，由 ResearchGate 作者 OCR 稿重建，核证清单见笔记，本页不出图**）** ★ 2026-10-27
| [[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting\|GaussianSpline]] | A- | **Normal（结论层 ≈20 分钟）** | **中高（★ "分配方式 > 分配额度"的第二号证据 / 4DGS 显式轨迹路线的当前样本**：Ben-Gurion，8 页 4 图；每高斯轨迹 = $K+1$ 节点 + $2K$ 内部点 + **$K$ 个可学段宽**（softmax logits ⇒ $t_k=\sum_{j<k}\mathrm{softmax}(w)_j$ 天然单调）+ **反射式 $G^1$** $C_{k,0}=J_k+\lambda_k(J_k-C_{k-1,1})$（不打折但允许速度变化）；**消融铁证（二维表横纵两读）**：纵读挪段 +0.37/+0.91/+0.00 dB，横读加段 31.61 ≤ 32.08 **收益为 0**；checker sphere 41.18 / flames 27.42（+3.46 超 STG）/ **soft contact 26.84 输给 Gaussian-Flow 31.10（−4.26）**——全局共享一组段宽 logits 抓不住瞬时碰撞；**诚实边界：纯离线、无运行时与显存数字、四元数线性插值大角速度必失真**；4 图取 3）** ★ 2026-10-28
| [[2026-10-24-Truong — Explicit Frictional Contact Modeling in Neural Garment Simulation\|Truong 2026 — 显式摩擦接触]] | A- | **Normal（结论层 ≈25 分钟）** | **高（★ "学习丢掉的是旋钮不是精度"的量化证据 / 神经布料的后处理算子范式**：WACV 2027 已接收，NSYSU；**四段算子** = 估法向载荷 $\hat\lambda_n=m[n^\top(\hat a+ge_{up})]_+$ → 闭式库仑修正（**arrest length $b=\mu h^2[n^\top(\hat a+ge_{up})]_+$，质量抵消，$b$ 只是一个长度** ⇒ 块软阈值，$\varepsilon_v=2\times10^{-3}$ m/s）→ 体表等距折线（投影—裁剪—展开，不走空间弦）→ **修正作线性化膜波传播**（$M\ddot w=-K_0w$，限每边应变 $\|w_i-w_j\|\le\gamma_{max}\ell_{ij}$）；**μ 响应表 = 分离证据**：CC+ours 0.994/1.047/1.139 vs **按 μ 重训的 native 0.927/0.927/0.924（平的）**  vs PPF 1.004/1.038/1.314；穿透 ×10⁻³%：PPF **0.2** / CC+ours 6.1 / native 10.4（**仍差 30 倍，因为它是"一次修正"不是"一个解"**）；消融：传播去掉应变限制 = **1149.0% / 1268.6%** 偏差；**8.1× 端到端（27.2 vs 220.4 s）/ 5.6× 逐帧（86 vs 481 ms）**，GNN 58% + 算子 42%（传播 13%）；**补了 Baraff-Witkin 1998 缺的静摩擦**；3 图取 2）** ★ 2026-10-28
| [[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State\|WorldStream]] | A | **Normal（结论层 ≈30 分钟）** | **最高（★ 世界模型的**最厚接口** / 运行时 PCG 的第一个完整样本**：中国科大 × 京东，19 页 18 图；**决策与实现彻底分离，中间只留一份记录**——Macro/Micro 规划器每字段输出"首选值 + 分布"，**有界联合求解** $z^*=\arg\max_{z\in F}[\sum_q\log p_q(z_q)+R(z,N_i)]$；实现是**确定性代码，模型不碰顶点**；**双前沿流式**（语义前沿 $K_p/H_p$ 跑在前面，实现前沿 $K_c\le K_p$ 只编译马上要用的，$C\subseteq Q$）；**持久记录** $R_i=(G_i,M_i,B_i,O_i,U_i,\rho_i,V_i)$、$X_i=\mathrm{Compile}(R_i)$——**几何是可再生资源，记录才是资产**；四组铁证：联合求解 **37%→100%** 可执行（独立 argmax 让 **63%** 违反硬约束）/ 无 Micro 邻接 **2,686 栋 vs 250 栋（10.7× 过度建造）** 边界跳变 0.361 vs 0.089 / 4 km 路线 100 次回放总等待 **3.52 s** / 密林编译 10.78→**3.32 s（3.24×）**；交互 76/80=95.0%；**软肋：局部编译均值 2.134 s、最大 3.318 s ⇒ 转向即 2 秒黑屏，3.52 s 是均值、均值掩盖分布**；三套异步一致性机制（边界记录 / 计划键 / 修订号）= 数据库的**乐观并发控制**；3 图原图）** ★ 2026-10-28
| [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)\|Baraff-Witkin 1998 — Large Steps]] | A（经典·历史重要性 5） | **Normal（结论层 ≈35 分钟）** | **最高（★ 此后 28 年所有布料解算器的骨架 / "分布 > 总量"的最早纯样本**：SIGGRAPH 1998，CMU（论文标注 Pixar）；**刚性 ODE 的步长被最硬模态钉死** ⇒ 后向 Euler 让步长与刚度解耦：**弯曲刚度扫 10,000×，运行时间变 <5%**；每步解 **18,000×18,000** 稀疏系统（modified-PCG 50–100 迭代）；**三件比"隐式积分"更重要的技术**：① **约束靠改质量矩阵** $W=\frac1m S_i$，$S_i\in\{I,\ I-pp^\top,\ I-pp^\top-qq^\top,\ 0\}$ ⇒ **约束精确满足、与迭代次数无关**；② **modified-PCG** 每步投影回可行方向（`filter(a)` 不变式），约束力从残差 $A\Delta v-b$ 读出；③ **位置修正项 $y_i$** 塞进顶行 ⇒ **一步去穿透且零额外计算**；Table 1（SGI Octane 195 MHz）：2,602 粒子 2.23 s / 6,450 衬衫 7.32 s / 8,757 裤子 38.5 s / 4,530 长裙 10.2 s，**C/C（碰撞约束）峰值 57.7%**；规模律 500→7,359 粒子 0.23→10.3 s（**超线性，$O(n^2)$ 碰撞在长大**）；步长 0.625–33 ms 自动浮动（**~50×**），30 Hz 下平均 2–3 步/帧；**未解决：静摩擦**（斜面上缓慢下滑）——28 年后由 Truong 2026 补上；**成本搬家说法的源头**；6 图取 4）** ★ 2026-10-28

## Concepts（49）

**基础锚点**
- [[GPU Architecture]] ★ 2026-10-25 补齐（**成本地图**：执行模型（SIMT/发散/occupancy）× 存储层级（4 级带宽差一个数量级）× 固定功能单元（ROP 绑分辨率）三类资源；**六个实测指标各落哪个单元**的表 + 五条常见误区（"算力有余量" / "occupancy 越高越好" / "draw call 少就快"）；此前被 Tile-Based / GPU-Driven / Niagara / 粒子 / 分档 / 预算六页引用而缺页）
- [[GPGPU]] ★ 2026-10-26 入库（**编程模型侧，GPU Architecture 的另一半**：**三条锁**（写位置由光栅化钉死 ⇒ 只能 gather；一图不可同时读写 ⇒ 必须双缓冲；寄存器/精度受限 ⇒ 必须打包）+ **成本公式 $W H N_{pass} B$** + 两代写法对照（2004 借片元管线 vs 2009 compute：**真正补上的只有 scatter 与组内通信**）+ 谱系 2001→2020s；**判据：问一条 GPU 模拟贵不贵，只答分辨率是没答，要答"分辨率 × pass 数"**）
- [[Real-Time Rendering]] ★ 2026-09-30 补齐（首日起引用悬空，月结日修复——**域地图 / 索引**：管线骨架 / 着色 / 光照 / 重建 / 神经五块 + 你的活跃收口点）
- [[Global Illumination]] ★ 2026-09-30 补齐（**八代近似谱系表**：环境光 → SH → 辐射度 → PRT → 屏幕空间 → **缓存族（三源齐 ✅ 10-04）** → 光追 → 神经；[[Neural Global Illumination]] 桥的具名前置三块全部可读——**"样本定预算 × 误差定预算"双轨语言**）
- [[Rendering Equation]] ★ 2026-09-15 入库（谱系最大锚点补齐）
- [[Microfacet Theory]] ★ 2026-09-16 入库（当前学习目标节点：D·G·F）
- [[Participating Media]] ★ 2026-09-16 入库（VFX/OverDraw Easy 域 ↔ 理论的最大缺口）★ **2026-09-21 找到历史入口：[[Reeves — Particle Systems (1983)]] §5 的两个"做不到"（粒子被打亮 / 云的自阴影）正是它的起点**
- [[Shadow Mapping]] ★ 2026-09-21 入库（**填补库里零覆盖的"阴影"域**；**"动态灯光 ≤3/≤2/≤1/0"的成本依据 = 每盏灯 ≈ +1× 场景渲染**；同时是 MegaLights 复审的对照基线）
- [[Linear Transport Theory]] ★ 2026-09-23 入库（**"输运"母题的共同坐标原点**：微面多次散射 / 体积多次散射 / 毛发 / 云是同一条方程的特例；配套手册 [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]]）
- [[Spherical Harmonics]] ★ 2026-10-21 入库（**"方向上的函数"的标准低频基**：四条性质（正交归一 / 旋转不变 / 卷积 / 乘积投影）各对应一个工程收益；**9 / 25 / 36 系数 = "质量档位有数学刻度"的少数样本**；串起 [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)|RH 2001 的 9 系数]] → [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT 的 25 系数]] → 探针/DDGI → 3DGS 的视角相关颜色）

**你的领域（Easy）**
- [[Particle Systems]] ★ 2026-09-21 入库（**VFX 域唯一的历史锚点**；五步帧循环 = Niagara 生命周期；**"屏占比 × 密度"是"同屏粒子数"预算的原始出处**）
- [[Real-Time VFX Performance Budgeting]]
- [[Overdraw]] ★ 2026-10-25 补齐（**五维指标里唯一没有自己页的那一项**：三种口径（Coverage / Shaded / Blended，可差一倍以上）+ 成本恒等式（**层数 × 屏占比 × 单价**）+ 屏占比换算表（64²/128²/256²/512² 各占全屏多少、几个铺满 1×）+ 带宽换算（RGBA8 混合 8 B/片元）+ **"单价取决于管线阶段"五档表**（不透明 early-Z 近乎免费 / alpha test 半价 / 半透明全价 / 加法混合免排序仍全价 / +屏幕采样翻倍）；**对 SABC 表的修订建议：补一行"像素预算" $\sum A_i/A_{screen} \le k$**——"同屏粒子数 ≤3000"在 64² 与 256² 之间差 16 倍填充）
- [[Niagara]]
- [[Scalability and Quality Tiers]]
- [[Gaussian Splatting]] ✅ 2026-09-11 标 Easy

**Normal 学习区**
- [[Procedural Content Generation]] ★ 2026-09-22 入库（**PCG 域第一个概念锚点**；源头 [[Müller — Procedural Modeling of Buildings (2006)]] + 前沿 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]]；**与 VFX 的真实交叉点：PCG 生成的场景 = 特效承载容器**）
- [[Multiple Scattering and Energy Compensation]] ★ 2026-09-19 入库、**2026-09-20 由三条路线扩为五条**（**PBR 最后一块账本**：单次散射丢了什么、五条补法各缺哪一角、**成本与覆盖面严格反向**；含唯一的引擎侧实测题 —— furnace test **要查两头**）
- [[Split-Sum Approximation]] ★ 2026-09-18 入库（PBR 工程侧最后一块：环境光镜面怎么变成两次查表）
- [[Hair Rendering]] ★ 2026-09-17 入库（资产表示 + 着色 + 分档三层；**也是 [[Microfacet Theory]] 失效的边界**）★ 2026-09-18 补齐理论源头 Marschner 2003
- [[Motion Matching]] ★ 关键瓶颈（9-17 起转 Watchlist 静默项）
- [[BRDF]] ★ 主动研读中（2026-09-14 信号）★ 2026-09-18 谱系补至"进引擎"
- [[Physically Based Rendering]] ★ 主动研读中（**来源侧 9-17 闭合、工程侧 9-18 闭合**）
- [[Tile-Based Rendering]]
- [[GPU-Driven Rendering]]
- [[Neural Upscaling and Frame Generation]]（实为 Technology）
- [[Temporal Stability and Artistic Intent]]
- [[Animation Compression]] ★ 2026-10-06 入库（**动画线"数据侧"第一节点**：三本账（内存/解码/导入）+ 误差界即档位 + "运行时代码器 vs 装载时代码器"分工；源头 CurveCodec 2 / ACL）
- [[Facial Animation]] ★ 2026-10-08 入库（**面部域开线（库内最后的大品类空白）**：重定向 = "身份 × 表情"分解 + 三种失败模式（表面噪声/全局耦合/身份泄漏）；首节点 [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]]（点基双因子，"四个不需要"）；挂载位：GALA（化身表示）/ emg2face（非光学捕捉，记名））
- [[Subdivision Surfaces]] ★ 2026-10-10 入库、**10-12 三角分支补齐**（**几何建模域第一个概念锚点**：规则 + 四结构性质 + 双轨 48 年弧线——四边形轨（1974 双源头 → 1978 正问题 → 1998 求值/生产 → 2005 奥斯卡 → 2026 逆问题）+ **三角轨（1987 Loop 规则 → 1998 Stam 三角版求值 → 共享逆问题时代）**；源头 [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]] / [[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)]] + 逆问题 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]）
- [[Mesh Parameterization]] ★ 2026-10-24 入库（**几何处理域开线（UV 展开）**：三步流水线（分割–参数化–装箱）+ **三目标不可兼得表**（保角 / 保面积 / 单射）+ **"局部单射 ≠ 全局单射"** 这张最容易踩的坑；谱系 1963 Tutte → 2001 stretch → 2002 LSCM → 2004 ABF → 2016 Campen 全局单射 → 2026 Tutte-Like 单射流；**Game 侧最实用的一句：无翻转是求解器给的，无重叠要你自己查**）
- [[Fluid Simulation]] ★ 2026-10-13 入库（**物理/仿真域开域（流体）**：四步分解（外力→对流→扩散→投影）+ 三路线（网格/粒子/混合 FLIP）+ 时间线 1965→2026；"水的两条路"（模拟 vs 感知）与"预算轴 = 网格 × 步数"；首节点 [[Stam — Stable Fluids (1999)]]）
- [[Motion Retargeting]] ★ 2026-10-13 入库（**动画数据管线第三环**（检索/压缩/转换）："要保清单"五级 + 三路线（IK 优化 / 数据驱动 / 优先级优化）；节点：[[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]] / [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement|Skinned Retargeting]] / [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]；★ 10-14 新增捕捉侧节点 [[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology|CAST]]）
- [[Resampled Importance Sampling]] ★ 2026-10-14 入库（**采样端理论**：三步重采样 + 校正因子 + 方差劈分 + 等时间准则 M=TY/TX；ReSTIR 谱系的底座；与 [[Reservoir Sampling]] 互链）
- [[Skinning]] ★ 2026-10-23 入库（**角色形变域的骨架层**：四代谱系（LBS 1988 → DQS 2008 → PSD/corrective → 神经蒙皮 → 肌肉层）+ 一张"混法对照表"（混什么 / 是否保刚体 / 代价 / 代表）+ 三条失败模式（糖果纸 / 塌肩 / 翻面）；**"混的对象必须同类"** 是它的中心判据；与 [[Motion Retargeting]]（上游）、[[Physics-based Character Animation]]（下游）接链）
- [[Cloth Simulation]] ★ 2026-10-28 入库（**第 49 个 / 同样是被引用 3 处却一直缺页**：三层（本构 / 积分 / 接触）+ **成本恒等式**$\text{每帧成本}=\text{步数（由最硬约束决定）}\times\text{每步代价（由规模与迭代数决定）}$ + **四条路线对照表**（显式 $O(n^2)$ / 隐式 $O(n^{1.5})$ / 投影 PBD $O(n)$ / 学习一次前向）+ **"成本会搬家，不会消失"** + **四种失效模式**（穿模 / 糖纸 / 自碰撞漏检 / **静摩擦缺失**）；谱系 1987 Terzopoulos → **1998 Baraff-Witkin（步长与刚度解耦）** → 2002 Bridson（自碰撞）→ 2007 PBD → 2016 XPBD → 2019 HOOD → **2026 Truong（把 μ 从权重里挖回来）**；✅ 与 [[Skinning]] 的判据：**穿模八成是碰撞体问题，糖纸八成是蒙皮问题**）
- [[Reservoir Sampling]] ★ 2026-10-14 入库（**流式加权抽样容器**："一个槽位代表整条候选流"；RTXDI 固定提交可直读；"一个表示代表全部"家族成员）
- [[ReSTIR]] ★ 2026-10-15 入库（**采样线合流点**：流式 RIS + 时空复用 + MIS 无偏化；DI/GI/GRIS/RTXDI/MegaLights 方法族底座；与读者框架"采样+复用+失效"逐项咬合）
- [[Budget Units]] ★ 2026-10-27 入库、**10-28 扩为七种**（**第 47 个 / 你整张 SABC 五维表的元层审计**：**七种计价对象**（容量 / 成本 / 精度 / 时间 / **状态-记忆** / 像素覆盖 / **时间采样**）+ **四组"分配方式 vs 分配额度"量化对照**（RADIANT 换单位 +3.3 dB vs 调 token 预算 ≤0.04 dB；**GaussianSpline 挪段 +0.91 dB vs 加段 0.00 dB**；PASPA 用错记忆单位 7.652 **比不给历史 7.065 还差**；UniWorld 把单位从"帧"换成"窗口"后显存上界与时长解耦）+ **三判据**（"内容翻十倍会翻十倍吗" / "额度锁在哪个时间尺度上" / "会不会随时长增长"）；**第七种的关键差别：前六种都能摊销，第七种不能**——一段 3 s 的信号，信息全在 0.3 s 的峰里；**落到你的表有两个动作：给"同屏粒子数"补一行像素预算；把"VFX 时长 ≤3s/2s/1s/0.5s"从"长度约束"改成"形状约束"**）

**Hard 待建桥**
- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Differentiable Rendering]] ★ 第二个瓶颈
- [[Neural Global Illumination]]
- [[Neural Animation]]
- [[Motion Generation]]
- [[Motion Tokenization]] ★ 2026-10-27 入库（**第 48 个 / 此前被引用 4 处却一直缺页**：四种切法（逐帧连续 / 逐帧离散 VQ / 片段级 clip token / **块级 block**）+ 三条设计轴；**核心判据：token 化的单位必须匹配你要记住的那个时间尺度**——由 PASPA 的自相关实测（风格 >100 帧仍相关、局部运动 <20 帧衰减）给出实证）
- [[Inverse Rendering]]
- [[Neural Physics Simulation]]（★ 全库桥最短的 Hard）
- [[Physics-based Character Animation]]（桥：排在 Motion Matching 之后）
- [[World Models for Games]]（Normal 读法：只学接口）

## Technologies（4）

- [[Neural Upscaling and Frame Generation]]（Adopt / Normal）
- [[Real-Time Global Illumination]]（Adopt / Normal）
- [[Real-Time Generative Motion]]（Assess / Hard）
- [[Arm Neural Graphics]]（Assess / Normal）

## Applications（2）

- [[AAA Real-Time VFX]] — 你的领域，含外部研究映射表与 4 个 Open Question
- [[Open World Character Animation]]

## Learning Paths（3）

- [[Learning Path — Gaussian Splatting]] — ✅ 完成（2026-09-11 标 Easy），基础推送已停
    - 产出笔记：[[GS 图解 1 — 协方差与椭球：高斯的形状说明书]] · [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]
- [[Learning Path — Differentiable Rendering]] — 目标是读懂 LightOpt
- [[Learning Path — Neural Rendering]] — 从你已懂的 TAA 出发

### 专题报告

- [[Modern Real-Time Lighting Framework|现代实时光照的共同设计框架]] — 比较 MegaLights、Lumen、ReSTIR 与 RTXDI 的采样、复用、重建和失效；含版本证据表、公开代码定位与流程图。[GitHub 阅读](Learning/Modern%20Real-Time%20Lighting%20Framework.md)
- [[Lightmass in the Rendering Knowledge Map|Lightmass 在渲染知识体系中的位置]] — 连接渲染方程、BRDF、GI 缓存与预计算；含层级关系图、红墙到 Lightmap 的几何／索引图、论文对照和来源边界。[GitHub 阅读](Learning/Lightmass%20in%20the%20Rendering%20Knowledge%20Map.md)

## 目录结构

```
Research/
    Papers/       论文笔记（一论文一文件，同一研究的不同版本合并）
    Concepts/     概念（"这个知识是什么"）
    Technologies/ 技术（"如何真正可用"）
    Applications/ 应用（游戏工业落点）
    Engines/      引擎与生产
    Daily/        每日研究
    Weekly/       每周综合
    Monthly/      每月总结
    Learning/     学习路径
    Radar/        技术雷达
```

## 使用约定

1. **原子化**：一个文件 = 一个稳定知识实体
2. **链接优先于复制**：发现新知识先查是否已有对应笔记
3. **frontmatter 的 `user_level` 是机器判断的唯一来源**，普通 tag 只是给人看的
4. **Easy 不重复推送**，除非该领域出现重大突破
5. **Hard 不强行推送**，先找 Normal 桥

---

最后更新：2026-10-27（Run #49：**「换单位」日**。读者侧：**远程 0 新提交**，已读账本维持 **40 篇**（连续 7 日无新 `read`）；前沿：周二班**如 10-26 的时钟分析所预测——确实吞下周末积压**，但体积不大（cs.GR 新条目仅 2 条：RADIANT / PASPA，均为周五提交、周一 20:00 ET 才公告），cs.CV 补扫抓到 UniWorld；主线：**四篇论文（3 前沿 + 1 经典）各自把"记账单位"换了一次**，指向同一命题——**换单位比调额度高两个数量级**（RADIANT：换单位 +3.3 dB vs 调 token 预算 ≤0.04 dB）。产出：[[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination|RADIANT]]（A，前馈 GI：物体 token 把传输序列 3,205→320）、[[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation|UniWorld]]（A-，薄接口世界模型：显存上界与时长解耦，**但消融几乎全跨零**）、[[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]]（A-，NeurIPS：**State+State 7.652 比不给历史 7.065 还差**——历史给错方式是负收益）、[[Reyes — The Reyes Image Rendering Architecture (1987)|Reyes 1987]]（A 经典·历史重要性 5：微多边形半像素 = 着色率与几何解耦的 1987 版本，**先着色后可见 ⇒ 结构性 overdraw ≈2.7×**；⚠️ 二手重建、本页不出图）；新概念 **[[Budget Units]]（第 47）** 与 **[[Motion Tokenization]]（第 48）**；概念回填 8 处（GI 谱系第 8 行拆出"拟合派 vs 前馈派"、World Models 新增**第五维度「接口的厚度」**、Overdraw 时间线补 1987 节点等）；6 张论文原图逐张核验；Papers 149→**153**，Concepts 46→**48**）
