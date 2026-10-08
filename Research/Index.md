---
type: moc
title: "Game Development Research — Index"
created: 2026-09-07
---

# Game Development Research — Index

全球游戏研发技术情报系统 + 个人认知知识库。

## 入口

- **今日**：[[2026-10-08]]（昨日：[[2026-10-07]]）
- **本周综合**：[[2026-W40]]（覆盖 **9-28 ~ 10-4**，✅ 已收官；下一周 W41 待产出，覆盖 10-5 ~ 10-11）
- **首月月报**：[[2026-09-Monthly|Monthly 2026-09]]（覆盖 9-7 ~ 9-30，✅ 2026-09-30 产出）
- **本月雷达**：[[2026-09]]（✅ 2026-09-30 月结刷新）
- **认知模型**：[[Personal Knowledge Model]] ✅ 校正于 2026-10-05 / **更新于 2026-10-08（毛发线六节点闭合；面部域开线）**——推送按校正结果组织

## Papers（92）

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

## Concepts（34）

**基础锚点**
- [[Real-Time Rendering]] ★ 2026-09-30 补齐（首日起引用悬空，月结日修复——**域地图 / 索引**：管线骨架 / 着色 / 光照 / 重建 / 神经五块 + 你的活跃收口点）
- [[Global Illumination]] ★ 2026-09-30 补齐（**八代近似谱系表**：环境光 → SH → 辐射度 → PRT → 屏幕空间 → **缓存族（三源齐 ✅ 10-04）** → 光追 → 神经；[[Neural Global Illumination]] 桥的具名前置三块全部可读——**"样本定预算 × 误差定预算"双轨语言**）
- [[Rendering Equation]] ★ 2026-09-15 入库（谱系最大锚点补齐）
- [[Microfacet Theory]] ★ 2026-09-16 入库（当前学习目标节点：D·G·F）
- [[Participating Media]] ★ 2026-09-16 入库（VFX/OverDraw Easy 域 ↔ 理论的最大缺口）★ **2026-09-21 找到历史入口：[[Reeves — Particle Systems (1983)]] §5 的两个"做不到"（粒子被打亮 / 云的自阴影）正是它的起点**
- [[Shadow Mapping]] ★ 2026-09-21 入库（**填补库里零覆盖的"阴影"域**；**"动态灯光 ≤3/≤2/≤1/0"的成本依据 = 每盏灯 ≈ +1× 场景渲染**；同时是 MegaLights 复审的对照基线）
- [[Linear Transport Theory]] ★ 2026-09-23 入库（**"输运"母题的共同坐标原点**：微面多次散射 / 体积多次散射 / 毛发 / 云是同一条方程的特例；配套手册 [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]]）

**你的领域（Easy）**
- [[Particle Systems]] ★ 2026-09-21 入库（**VFX 域唯一的历史锚点**；五步帧循环 = Niagara 生命周期；**"屏占比 × 密度"是"同屏粒子数"预算的原始出处**）
- [[Real-Time VFX Performance Budgeting]]
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

**Hard 待建桥**
- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Differentiable Rendering]] ★ 第二个瓶颈
- [[Neural Global Illumination]]
- [[Neural Animation]]
- [[Motion Generation]]
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

最后更新：2026-10-08（Run #30：**毛发生产化日 —— 六节点闭合 + 面部域开线** —— 经典：[[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]]（WDAS 生产毛发模型；near-field 取消 70 点求积 / logistic / 第四叶；着色 ~20×；⚠️ 此前 "Pixar" 系误记，已更正）；前沿 3 篇入库：[[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]]（面部重定向"四个不需要"）· [[2026-10-07-DynaConTalk — Wavelet-Constrained Diffusion for Long-Form and Controllable Holistic Co-Speech 3D Motion|DynaConTalk]]（小波带扩散；同误差 −89.4%）· [[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing|Multiphysics 教程]]（Bender 组）；新概念 [[Facial Animation]]；产业：**NVIDIA RTX Spark 发布**（N1X 平台；10-16 上架）+ E-Day 售后线）
　　▸ **10-7 回顾**（Run #29）：**能量日**——d'Eon 2011（毛发能量页）+ NEF / SteadySplats / PhysLDM；DXR 2.0 将标准化 Mega Geometry
　　▸ **10-6 回顾**（Run #28）：**分工日**——Neuroll（毛发仿真轴）+ CurveCodec 2（动画压缩开线）+ DDGI 2019（缓存族封顶）；PKM 按实际阅读校正
　　▸ **10-5 回顾**（Run #27）：**漫反射侧结账日**——Hammon 2017（阻塞 16 天经新 CDN 解除；PBR 经典队列清空）+ Budgeted-GS（GS 预算线开线、Zador 定律）
　　▸ **10-4 回顾**（Run #26）：缓存族第三源日（Ward 1988）——三源齐 + W40 周报
　　▸ **10-3 回顾**（Run #25）：缓存族双源日——RSM 2005 + IR 1997；"固定样本预算"四连；River + GALA
