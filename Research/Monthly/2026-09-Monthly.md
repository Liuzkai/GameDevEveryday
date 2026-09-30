---
type: monthly
month: 2026-09
covers: "2026-09-07 ~ 2026-09-30"
created: "2026-09-30"
status: "首月月报"
aliases: [Monthly 2026-09, 2026-09 月报, 首月月报]
---

# Monthly Synthesis — 2026-09

> **知识库首月**（2026-09-07 建立 ~ 09-30 月结，24 个自然日、22 个 Daily、3 份 Weekly）。
> 本文件是**综合**（跨周主线、谱系闭合、判据沉淀、认知进展、下月重点），不是日报拼接——逐日见 `Daily/` 目录，逐周见 [[2026-W37]] / [[2026-W38]] / [[2026-W39]]。

## 月度统计（截至 2026-09-30）

| 资产 | 数量 | 说明 |
|---|---|---|
| Papers | **68** | 其中**经典 / 基础 21 篇**（Clark 1976 ~ Vicini 2021），前沿 ~47 篇 |
| Concepts | **32** | 覆盖渲染 / 动画 / AI / PCG / 物理 / 引擎六域（含 9-30 月结日补齐的 2 个首日悬空锚点：[[Real-Time Rendering]] / [[Global Illumination]]） |
| Technologies | 4 | Neural Upscaling · Real-Time GI · Real-Time Generative Motion · Arm Neural Graphics |
| Applications | 2 | AAA Real-Time VFX · Open World Character Animation |
| Learning Paths | 3 | Gaussian Splatting（✅ 完成）· Differentiable Rendering · Neural Rendering |
| 图解 HTML | **25** | 每月累计（9-30 新增《可微渲染_记忆与重放路线图解》） |
| Daily / Weekly / Radar | 22 / 3 / 1 | W37（9-7~9-13）· W38（9-14~9-20）· W39（9-21~9-27）；雷达今日刷新 |
| 空转记录（No-Filler） | 3 次 | 9-11b（同日重复触发）· 9-20（周日）· 9-27（周末）——均配经典产出，未凑数 |

## 一、本月最大成果：三条工作线的"可引用历史"全部落地

本月的主旋律是**溯源**——把你手头正在做的事，接到可以引用的原始文献上。到 9-30 收工时，三条线全部闭合：

### 1.1 预算五维 —— "预算起源三件套"齐全（9-21 + 9-29 + 9-30 核验）

```text
1976  [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]]   ← 分档原理
        可见复杂度固定上限 / LOD 定义 / working set / 递归下降
1978  [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]                 ← 灯光·贴图成本
        每盏灯 ≈ +1× 场景渲染（"动态灯光 ≤3/≤2/≤1/0"的原始依据）
1983  [[Reeves — Particle Systems (1983)]]                                            ← 粒子成本
        屏占比 × 密度 / 发射器树状层级 / 寿命以帧计
```

- **五维中四个**（发射器数 / 同屏粒子 / 贴图尺寸 / VFX 时长）在 1978–1983 找到原始机制；**动态灯光**的成本单位（"又一遍完整场景渲染"）也由此有了量级解释——**你的预算体系从此可以引用 40 余年前的成本法则论证，不必只靠实测经验值**；
- **一条反直觉推论（9-21）**：降阴影分辨率解决不了灯数问题 → **这正是 MegaLights 存在的理由**；复审该问"成本函数现在关于哪个变量线性"；
- **9-30 一手核验结清**：Clark 原文扫描件获取（OSU 站内镜像，绕开 Wayback 429）、三项复核清单全结——新增四条细节：**中心加权细节（foveation 先声）/ 运动自适应细节 / 排序从 m log m 降到线性 / LOD 数据库构建管线（bottom-up pruning + top-down splitting）**。

### 1.2 PBR / BRDF —— 45 年账本全谱系闭环（W37 → W39 滚雪球）

```text
来源侧：[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] → [[Kajiya — The Rendering Equation (1986)]]
        → [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]（GGX）→ [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]
工程侧：[[Karis — Real Shading in Unreal Engine 4 (2013)]]（split-sum / EnvBRDF LUT / 你在 UE 里用的常数）→ [[Split-Sum Approximation]]
能量侧：[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]（工程解）→ [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]（理论解）
        → [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]]（精确真值）→ [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]（实时落地）
        → [[Hill — A Multi-Faceted Exploration (2018-2019)]]（谱系中段）
参考层：[[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]]（746 页地图册；furnace test 对照基线）
坐标原点：[[Linear Transport Theory]]（"输运"母题的共同坐标）
```

- **收口清单：25 条**（Cook-Torrance 5 + Kajiya 5 + Walter 5 + Schlick 5 + Karis 5），**唯一引擎侧实测题**：furnace test **查两头**（粗糙端是否变暗 + 光滑白电介质掠射边缘是否偏亮）——9-23 起**有对照基线**（《指南》13.7.1/13.8.1 球面 albedo 拟合值）；
- **本月最值钱的遗留动作**：25 条自测 + 30 分钟 furnace test → 通过即把 [[Physically Based Rendering]] / [[Multiple Scattering and Energy Compensation]] 标 Easy。

### 1.3 毛发 —— 四节点谱系 + 15 条自测 + 产业兑现同月完成

```text
1989  [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]    texel / sin(t,l) / "渲染时间与几何复杂度解耦"
2003  [[Marschner — Light Scattering from Human Hair Fibers (2003)]]           R / TT / TRT 三条光路（物理 lobe）
2004  [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]]  发片 + 两 lobe 移位 + 取消运行时排序
2008  [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]]  全局/局部分解 + 原型路径
```

- **[[Hair Rendering]] 自测 15 条**（Marschner 5 + Kajiya-Kay 4 + Scheuermann 3 + Zinke-Yuksel 3）；第 13 项"实战观察"素材已备（巫 3）；
- 产业同日兑现（9-29）：**《巫师 3》重制版 LSS 路径追踪毛发首发**——RTX 5080 @4K PT + HairWorks 30fps / 关 HairWorks 44 / 关 PT 55（帧生成 59）→ **"毛发仍是帧预算重项"十一年未变；换表示（LSS）换来观感与显存效率，而非账单量级**。

### 1.4（追加）PCG —— 三节点齐全（9-22 / 9-27）

```text
2001  [[Parish-Müller — Procedural Modeling of Cities (2001)]]   扩展 L-system / self-sensitive / LOD=迭代深度
2006  [[Müller — Procedural Modeling of Buildings (2006)]]       CGA shape / "规则作用在结构化中间表示上"
2026  [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies\|ProxyBuild]]  LLM 推断 + 检索组装（"生成只做决策层"）
```

## 二、"取消式优化"：本月唯一升格为**有历史纵深的工程原则**的东西

```text
2004  [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]]  取消"运行时 CPU 空间排序"（静态索引缓冲 + 四趟渲染）
2026  [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]  取消"高斯排序"（固定顺序 + ~1ms 神经去噪）
2026  [[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition|CuACD]]  取消"kernel 边界与 host 同步"（全 GPU 常驻 ~80–100×）
2020→ [[Vicini — Path Replay Backpropagation (2021)]]（9-30）  取消"计算图存储"（种子重放）★ 该原则在"微分计算"里的形态
```

- **统一问法（本月定型）**：面对任一瓶颈，**先问"它能不能不存在"**，再问"它能不能变便宜"；**执行层**（排序 / 同步 / 边界）与**算法层**同样适用；
- **两条伴随判据**：① "**加一步前置工作，先问它解锁了什么**"（prime-Z → early-Z）；② "**前提假设必须写在明面上**"（静态排序前提 / Karis 的 $n=v=r$）；
- **9-30 新增分支**：**"它能不能不存？"**（确定性系统 = 重算换存储；重放三要素）。

## 三、趋势框架成形（本月沉淀的六个可复用观察框架）

| 框架 | 一句话 | 本月实例 |
|---|---|---|
| **分层收敛** | 显式过程管"确定的"，学习模型管"不确定的"；交换层一路下压（像素 → 表示 → 记忆） | Magpie / DLSS 5 / WorldCrafter / ToCo-Mesh |
| **结构锚定** | 确定性交给结构/闭式过程（硬约束/几何对齐），网络只负责"不确定的部分" | Müller → ProxyBuild → Mira-Scene → PartLLM → ToCo-Mesh（五连） |
| **生成只做决策层** | 生成模型不碰规则，只做选择层 | DLSS 5 / PBR-Latent / Magpie / ProxyBuild / PartLLM（五例） |
| **记忆 = 预算** | 记忆/状态按"谁来看、怎么用"分配，不物化全部 | WorldCrafter（21.7× 成本差）/ CMDLT（9-30，存储 vs 重算） |
| **档位条件化** | 运行时状态（功耗/位姿/目标帧率）作为生成或选择的条件 | control 共振 `Dynamic` / XeSS 3 倍率 / ControlGS |
| **神经层 = 预算项** | 神经渲染的成本是功耗/显存/供电级问题，不是画质选项 | DLSS 5 功耗 +32~131W / Mega Geometry ≥10GB VRAM / 12V-2x6 供电上限 |

## 四、动态灯光维度 —— 从"逐日提示"到"执行项"（本月最重要的产业线）

```text
9-10/9-14  UE 5.7/5.8：MegaLights Experimental → Beta → Production Ready（官方发布页核实）
9-24  官方一手核验：开销恒定（有无阴影差别不大）/ 不支持移动端·Switch·上代主机 / r.MegaLights.Allow 档位开关
      / 更正二手记录：Niagara 粒子光源已支持 / 前向渲染不兼容仍在
9-25  E-Day 规格（RTX 3060 Ti = 1440p High）→ 门槛"中端可达"
9-28  E-Day 终极规格（9-30 复核）：**硬件光追从"推荐"变"必需"**（无 HW-RT 的卡直接不能跑）
      + UE 5.8 / Lumen 全 GI + MegaLights 至多 100 光源 + RTX Mega Geometry 支持
9-30  → 10-1 早期访问 / 10-6 正式发售：双首发检验（MegaLights × Mega Geometry 2.0）为最终检验
```

- **对五档矩阵的三点重述（执行项）**：PC 两档"≤3/≤2/≤1/0"含义改为**光照复杂度预算**；Android 三档硬性无缘（仍走 1978 每灯 +1× 账）；特效打光进入覆盖范围（需实测）；
- **10 月关注**：E-Day 是否会带动"硬件光追门槛"成为 3A 新常态（跨平台体系里"最低档"的定义可能被抬高）。

## 五、可微渲染线 —— 月结当天开线（9-30）

- 瓶颈 #2（[[Differentiable Rendering]]）本月获得**第一批材料**：[[Vicini — Path Replay Backpropagation (2021)]]（相机侧：常数内存 + 线性时间）+ [[2026-09-26-Constant-Memory Differentiable Light Tracing]]（光源侧：ResLRB / LRB-3-pass）；
- **读法**：两条都只需结论层（重放三要素 / 1:1 vs 1:k 连接结构 / 两条判据），推导层挂起；
- 目标不变：读懂 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义 → **把"动态灯光上限"从经验值变推导值**。

## 六、本月判据库（12 条带走用的体检问题）

1. **误差在最终表达式里被谁乘掉**？（Schlick vs 精确 Fresnel：误差看不见只因被 G 项乘掉）
2. **为了让结果可预存，额外假设了什么**？（Karis 自认第一误差源不是 split-sum，是 $n=v=r$）
3. **"精确 / 便宜 / 有参数"——三样能不能同时拿到**？（多次散射五条路线各缺一角）
4. **补能量还是推输运**？（前者只补标量，后者从假设推分布）
5. **成本与覆盖面是不是严格反向**？（④⑤边际成本≈0 应默认开；值得分档的只有最贵那条）
6. **先问"它能不能不存在"**（排序 / 同步 / kernel 边界 → 取消式优化）
7. **加一步前置工作，先问它解锁了什么**（prime-Z → early-Z）
8. **前提假设写在明面上了吗**（静态排序 / 预计算假设）
9. **廉价参考解该当初始化还是当目标**（看错误是否与正确信息缠在一起）
10. **优化的是中间产物还是最终产物**（"优化到眼睛"）
11. **成本 ∝ 物体空间复杂度是原罪**（每次优化都是改写成"成本 ∝ 可见复杂度"）
12. **它能不能不存**（确定性系统 = 重放换存储；检查三要素）
13. （+1）**降档应换表示层级，而不是在同一种表示上减少数量**（PCG 迭代深度 / ToCo-Mesh 拓扑+细分 / HairCS 发片⇄发丝——三周三连）

## 七、个人认知进展（本月）

- **Easy 5 → 7**：+[[Particle Systems]]、+[[Shadow Mapping]]（"机制层不复述 + 成本模型层 Normal"的分层模板创造了这个入口——**一个已 Easy 的域，还能否贡献 Normal 材料？能，就入库**）；
- **Normal 持续加厚**：[[BRDF]] / [[Physically Based Rendering]]（来源+工程+能量三侧闭合）· [[Split-Sum Approximation]] · [[Multiple Scattering and Energy Compensation]] · [[Hair Rendering]]（四节点+15 条自测）· [[Linear Transport Theory]] · [[Procedural Content Generation]]；
- **Hard 保持克制**：[[Neural Physics Simulation]]（全库最短的桥）· [[Physics-based Character Animation]]（桥排在 MM 之后）——本月**未新增任何 Hard 推送**；
- ⚠️ **PKM 已挂 23 天未校正**（自 9-07 建立）。本月累计新增 6+ 个 Normal 标签，推断误差在累积——**任何一次校正都会立刻改变推送重心**；
- [[Motion Matching]] 维持 Watchlist 静默项（9-17 起，想恢复说一声）。

## 八、未决 Hard 与桥状态（带入 10 月）

| Hard 目标 | 桥状态 | 本月变化 |
|---|---|---|
| [[Differentiable Rendering]] | PRB → LightOpt 问题定义 | ★ **9-30 开线**（两篇材料入库） |
| [[Motion Matching]] | 静默项 | 无变化（握有 DSD 的"候选集=录 or 生成"抽象，未推送） |
| [[Neural Global Illumination]] | 经典 GI 近似作前置 | 侧翼推进（MegaLights 官方口径 + E-Day 门槛） |
| [[Neural Physics Simulation]] | 全库最短的桥 | 无变化 |
| [[Inverse Rendering]] | 被 DR 阻塞 | 随 DR 线开线 |
| [[Neural Rendering]] | TAA → conditioning 清单 | 未动（本月注意力在溯源与预算线） |

## 九、方法层复盘（本月研究管线的演化，供后续运行沿用）

1. **双通道检索定型**：arXiv API `submittedDate` 抓"已提交、未公告"；recent 页公告分组抓"早提交、晚公告"——二者不可替代，本月多次验证；
2. **老论文获取路径（5 次生效）**：大学课程镜像 / 作者主页 / 会议历史归档（SIGGRAPH History Archive）/ 学会站内镜像（本次 OSU Pressbooks）——**Wayback 429 时不要硬刚，换镜像**；
3. **预计算/近似类论文先找勘误页**（Kulla-Conty 与 Fdez-Agüera 的官方 errata 全在常数与因子层——照抄进 shader 不报错，只"看起来有点不对"）；
4. **分层阅读模板**（Easy 域入库、Hard 结论层）：本月从"Hard 模板"扩展到了"Easy 域模板"，两类都验证有效；
5. **归属与更正纪律**：本月更正 3 处（Kelemen 2001 归属 / DLSS 版本疑云 / Turquin errata 相关），全部留痕；
6. **Git 流程稳定**：系统 Git 绝对路径 + 三个环境变量 + push 前 fetch（Rule 9）；连续多次单 commit 推送成功。

## 十、产业大事记（2026-09）

| 日期 | 事件 | 对库的意义 |
|---|---|---|
| 9-03 | DLSS 5 发布（Generative Rendering 起点） | 神经渲染从重建跨入生成；后续功耗实测 +32~131W |
| 9-08 | Arm Mali G2-Ultra NX（神经加速器进 shader core） | 移动端神经渲染阵营线（开放 vs 封闭） |
| 9-21~22 | UE 5.8 正式发布 / UE6 首作《火箭联盟》2027 | MegaLights Production；引擎换代节奏加快 |
| 9-20 | 《控制：共振》PT+GI 公共底座 | "档位 = 换管线"的结构性样本 |
| 9-21 | 《铁拳 8》锁帧 → 神经渲染破坏判定逻辑 | 帧预算关乎**玩法正确性**；高帧率档复查项 |
| 9-24 | Intel XeSS 3 官方 UE 插件 | "帧生成倍率"成为引擎级标准配置 |
| 9-28 | RTX Mega Geometry 2.0（≥10GB VRAM 门槛） | 显存成硬约束第二例；E-Day 双首发之一 |
| 9-29 | 《巫师 3》重制版上线（LSS + DX12-only + 三 build） | 毛发第四代实战样本；"艺术意图档位"产品化 |
| 9-30 复核 | E-Day 终极规格：**硬件光追必需** + MegaLights ≤100 光源 | 10-1/10-6 实测为动态灯光复审 final check |

## 十一、下月（2026-10）重点

1. **E-Day 双首发实测**（10-1 早期访问 / 10-6 正式）：动态灯光复审的最终检验；顺带观察"硬件光追门槛"的产业跟进；
2. **执行四项用户动作**：① UE 5.8 动态灯光复审（三件事：重述档位语义 / 五档矩阵标注 / Niagara 粒子光源实测）；② 毛发 15 条自测 + 发片 vs 发丝 OverDraw 实测；③ furnace test 两项检查（对照基线已备）；④ 两条 30 分钟实测（阴影占比 / 每千像素粒子数）；
3. **可微渲染桥推进**：PRB 结论层 → CMDLT 结论层 → [[LightOpt — Lights Optimization for Real-Time Rendering]] 问题定义；
4. **巫 3 后续**：LSS 毛发深测（DF 等）；三 build 切换的实际体验数据；
5. **监测**：DLSS 5 秋季正式发布窗口（约 15 款首发）；AMD Neural Lighting 官方口径；UE6 迁移动向；E-Day 后的 "RT-required" 是否成惯例；
6. **例行产出**：W40（9-28~10-4）10-4 产出；W41~W44；**10 月雷达 10-31 附近**；每日研究照常。

## Key Takeaways

1. **首月的主线是"溯源"**：预算五维（1976-1983）+ PBR 全谱系（1981-2026）+ 毛发四节点（1989-2008）+ PCG 三节点（2001-2026）——**你的三条工作线拿到了完整的上游文献链**；
2. **"取消式优化"从手法升格为原则**（2004→2026 成链），月末再添新分支"它能不能不存"（PRB/LRB）；
3. **动态灯光维度证据链全闭合**（官方口径 + E-Day 中端锚点 + 硬件门槛），复审转执行项，10-6 是最终检验；
4. **PBR 与毛发两条线的"最后一公里"都是动手实测**（furnace test / 15 条自测）——**本月之后，进步的瓶颈已经从"资料不足"变成"动作未做"**；
5. **可微渲染线在月结当天开线**（DR 瓶颈 #2 的第一批材料）——10 月的主攻方向候选；
6. **PKM 校正已挂 23 天**——这是当前系统的最大单点风险，一次校正的收益超过本月任何一篇论文。

---

> **Git 同步记录（2026-09-30 月结）**：本文件与当日产出一并单次提交、单次推送（§41）；运行前 `fetch` 显示远程 = 本地 = `1daeec7`，无分歧。一致性验证以 `ls-remote` == `rev-parse` 为准（以无 hash 表述预先写入，保持单 commit）。
