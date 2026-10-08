---
type: paper
title: "A Practical and Controllable Hair and Fur Model for Production Path Tracing"
authors: [Matt Jen-Yuan Chiang, Benedikt Bitterli, Chuck Tappan, Brent Burley]
year: 2016
published: "2016-05（Eurographics 2016；Computer Graphics Forum 35(2): 275–283）"
venue: "Eurographics 2016 / CGF 35(2): 275–283"
url: "https://doi.org/10.1111/cgf.12830"
code: ""
project_page: "https://benedikt-bitterli.me/pchfm/（作者官方页；本次已下载 PDF 逐页核对）"
category: [hair, fur, scattering, path-tracing, production, energy-conservation]
importance: A（经典）
historical_importance: 4
game_relevance: 4
production_readiness: "Industry Adopted（Walt Disney Animation Studios 生产采用，Hyperion 路径追踪器渲染；致谢 Zootopia 团队——论文示例即其生产角色）"
user_level: Normal
status: unread
aliases: [Chiang 2016, PCHFM, Disney Hair, 生产路径追踪毛发模型, 迪士尼毛发模型, 生产化毛发]
tags: [hair, scattering, path-tracing, production, energy-conservation]
---

# A Practical and Controllable Hair and Fur Model for Production Path Tracing（Chiang et al. 2016）

> **入库 2026-10-08（Run 30）。** **Walt Disney Animation Studios**（Chiang / Bitterli / Tappan / **Brent Burley**——迪士尼 BRDF 作者；Bitterli 同时挂 Disney Research Zürich）。
> **⚠️ 更正留痕**：此前库内（[[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] 笔记与 [[2026-10-07]] 日报）将本篇记名作 "Pixar 生产化"——**系误记**。本篇单位是 WDAS，2016 年时四位作者均在迪士尼动画工作室（Matt Jen-Yuan Chiang 后来才去 Pixar，与本篇无关）。
> **与库内线的关系**：① [[Hair Rendering]] 谱系的**第 6 节点（生产化）**——1989 / 2003 / 2004 / 2008 / 2011 **→ 2016**；② 它是 [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] 的**直系生产实现**（沿用守恒 M_p 与 dMH13 的完美采样，把"方位向积分"的算法整个换掉）；③ 库内"取消式优化"家族的新样本——**"每次着色 70 点高斯求积"被取消**（把积分外包给路径追踪器本身）。

## TL;DR

**把"实验级的物理模型"改造成"电影生产里跑得动的模型"。** 四个改动都是结构性的，不是调参：
① **积分搬家（near-field）**——不再在每次着色里沿纤维宽度做积分（d'Eon 用 70 点高斯求积），而是**用光线与纤维的真实交点 $h$ 直接求值**，把"宽度积分"交给路径追踪器本来就有的采样机制（每个样本本来就是一条穿过纤维的光路）；
② **换分布族**——wrapped Gaussian → **logistic 分布**（可解析归一化 + 可解析求逆采样，1 个就够）；
③ **无穷折一项**——TRRT 以上的全部高阶 lobe 用**几何级数闭式**折成"第四叶"（守恒到白光掠射，通过 furnace test）；
④ **感知均匀参数化**——把物理参数（方差 ν、吸收系数 σa）重写为艺术家直觉参数（βM / βN / 多散射 albedo C），统一为**六参数 shader**。

结果：毛发着色 **~20×** 加速、该生产角色整体渲染 **>10×**（869 min → 76 min；着色 800 → 44 min），渲染从"着色占 >80% 总时长"降到"与几何遍历相当"——**原文原句："makes brute force path tracing of fur and hair possible in production rendering for the first time."**

## Problem

**旧模型是在"每像素着色一次"的时代设计的，而路径追踪器每像素要着色成百上千次。**

三条具体的"跑不动"（原文 §1–§3.2 逐条）：
1. **路径数量账**：路径追踪为解 GI 每像素产生数千样本；毛皮角色有数百万根发丝，**每条路径可能命中毛发着色器数百次**——单次求值的成本被直接放大 3–4 个数量级（原句：*"individual shader evaluation need to be computationally cheap in order for such an approach to be practical"*）；
2. **方位向积分账**：far-field 模型把"沿纤维宽度的散射响应"挤成一个聚合响应 $N_p(\varphi)=\frac{1}{2}\int_{-1}^{1}A(p,h)D(\varphi-F(p,h))dh$——d'Eon 推荐 70 点高斯求积。在路径追踪环境里，**每个着色点 70 次求值成为瓶颈**；且低粗糙度 + 长内反射路径时求积仍会振荡；d'Eon 建议的"预计算 2D 表"在**参数由贴图/动态表达式空间变化**的生产里不成立；
3. **误差放大账**：毛皮角色密度高、反射率高，**单散射的微小能量损失会被几百次散射事件放大**——所以"守恒"在这里不是学术洁癖，是画面正确性。

外加一条非技术目标：**艺术家可控制性**。物理参数（ν、σa）对美术不直观；而多散射让"单纤维参数 → 发束外观"是非线性的——需要重新参数化。

## Historical Context

```text
1989 Kajiya-Kay（经验：texel + 圆锥高光）
        ↓
2003 Marschner（物理：R/TT/TRT + 方位/纵向分解；为"每像素一次"设计）
        ↓
2007 Zinke-Weber（BCSDF 位置相关形式的先声——本篇积分搬家的灵感来源）
        ↓
2008 Zinke-Yuksel（双散射：多散射近似）
        ↓
2011 d'Eon（Weta：守恒 M_p + 取消求根 + 任意阶；为离线生产设计）
        ↓
★ 2016 本篇（WDAS）：near-field 取消求积 + logistic + 第四叶 + 参数化
        → 路径追踪时代的生产标准形态（Hyperion 实装）
        ↓
2016+ 影视毛发着色的事实基线之一；思想（廉价单次求值 + 感知参数化）
        后来贯穿实时/离线两条线
```

## Previous Work

- **Marschner 2003**：解析地做宽度积分——但**只覆盖前三个 lobe**、无法纳入方位粗糙度（本文把它当作"历史起点"）；
- **d'Eon 2011**：用高斯求积数值积分——允许方位粗糙度与任意阶，**但求积贵且低粗糙度振荡**。本文沿用其守恒 $M_p$ 与 d'Eon–Marschner–Hanika 2013 的完美采样方案（SIGGRAPH Asia 2013 Technical Briefs；库外文献），**只动方位向这一半**；
- **并发工作 Yan et al. 2015（double cylinder / medulla）**：为动物毛发加内芯柱。本文明确指出其 **tabulated 实现使空间变化参数不可行**、且缺少直观参数化（对比材料）；
- **Zinke-Yuksel 2008 双散射**：本文与它是**正交**关系——双散射是"多散射的近似"；本文的目标是**让暴力路径追踪的多散射本身变得可行**，从而**不需要**那些有偏近似（图 1：path-traced vs 双散射的观感对比——"coarse and stiff look"）。

## Core Idea

**一句话：先把"每次求值的贵"和"参数的不直观"分开解决，再让两者互不干扰。**

- **贵 → 换积分归属**：内层积分不再由着色器算，而是**由渲染器的采样来"顺带完成"**（near-field）。原文自述动机：*"A Monte Carlo renderer is by nature well equipped to compute statistically unbiased solutions to difficult integrals"*；
- **不直观 → 换坐标**：把"物理参数"翻译成"观感参数"（感知均匀映射）；把"多散射后的颜色"直接当输入（对艺术家最有用的量），反解单纤维吸收系数；
- **守恒 → 用闭式结构保证**：logistic（归一化解析）+ 第四叶（级数闭式）——**两个"把无穷/积分变成闭式"的数学动作**。

## Technical Approach

### ① Near-field：把宽度积分交给路径追踪器（Eq 4）

$$\underbrace{N_p(\varphi)=\tfrac{1}{2}\int_{-1}^{1}A(p,h)\,D(\varphi-F(p,h))\,dh}_{\text{d'Eon：每次着色求积 70 次}}\;\Longrightarrow\;\underbrace{N_p(\varphi,h)=A_p(h)\,D_p(\varphi-F(p,h))}_{\text{本文：每次着色 1 次，}h\text{ 来自光线与纤维的真实交点}}$$

- 合法性：MC 渲染器给的本来就是"带 $h$ 的一条条样本"；期望意义上收敛到同一个积分。**这不是近似——是同一积分的另一种结算方式**（无偏）；
- 附带收益（原文自述）：**近景更准**——far-field 的聚合响应丢掉了"沿纤维宽度的空间变化"，near-field 天然恢复（图 10 近景对照；RMSE 0.0140999 vs d'Eon 0.0146729）；
- 灵感来自 Zinke-Weber 2007 的 position-dependent BSDF 近似，**但动机换成了性能**（原文自注："with the motivation of improving performance instead of its original goal of obtaining close-up accuracy"）。

### ② Logistic 方位分布（Appendix A）

- 问题：wrapped Gaussian 需要**多个高斯求和**（近各向同性时 ~5 个才能把能量损失压到 <1%），且**不可解析积分、不可解析反演**（只能 Box-Muller 或查表——参数逐样本变化时都不合用）；
- 替换：logistic 分布经重参数化（归一化到 $[-\pi,\pi]$ + 匹配高斯峰：缩放因子 $\sqrt{\pi/8}$）后形状接近；
- 三条关键性质：**解析归一化**（于是"完美守恒"）、**闭式 CDF**（可解析反向采样）、**单峰单族**（一个分布覆盖从低到各向同性的全粗糙度域）。

### ③ 第四叶：无穷高阶折成一项（Eq 6）

- 守恒需要无穷阶：R/TT/TRT 三阶在**低吸收（白发）+ 掠射**下明显丢能量；d'Eon 可以加阶，但**白纤维 85° 倾角要到约 10 个 lobe 才能把损失压到 <1%**（图 4）——每样本 10 次求值在生产里不现实；
- 观察：TRRT 以上所有高阶的能量是**几何级数**——$A_{\text{fourth}}=\sum_{p=\text{TRRT}}^{\infty}A_p=(1-f)^2f^2T^3/(1-fT)$（$f$ = Fresnel，$T$ = 吸收）；
- 做法：**用一个"第四叶"代表全部高阶**——纵向仍沿高光锥（保持方向性），**方位向近似各向同性**（残余能量小、高阶方位分布高度不规则，粗化的代价可控）；
- 验证：图 5 的 **furnace test**——首位三叶（a）/四叶（b）与 20 叶 ground truth（c）对照，本篇四叶版本（d）"conserves energy as it passes the Furnace test"（e），且与 ground truth 低差异（f）。

### ④ 感知均匀参数化（Eq 7–9，六参数 shader）

| 艺术家参数 | 映射到物理量 | 映射来源 |
|---|---|---|
| 纵向粗糙度 $\beta_M\in[0,1]$ | 方差 $\nu=(0.726\beta_M+0.812\beta_M^2+3.7\beta_M^{20})^2$ | **美术实验**：让艺术家挑"感知均匀间隔"的参考图 |
| 方位向粗糙度 $\beta_N$ | logistic scale $s=0.265\beta_N+1.194\beta_N^2+5.372\beta_N^{22}$ | 数值散射仿真建立方位/纵向分布的等效关系后，复用 $\beta_M$ 的感知映射 |
| 颜色（多散射 albedo $C$） | 吸收系数 $\sigma_a=\big(\ln C/(5.969-0.215\beta_N+2.532\beta_N^2-10.73\beta_N^3+5.574\beta_N^4+0.245\beta_N^5)\big)^2$ | **渲染密发立方体 + 白穹顶**测出 $(\sigma_a,\beta_N)\to C$ 数据后最小二乘反解 |
| 主反射粗糙度 / IOR / cuticle | 涂层感、鳞片层反光增强 | 动物毛皮的启发式控制 |

- 颜色映射的思想（原文）：**"从半无限参与介质的表面 albedo 反解吸收系数"**——并论证它对**密度不变**（散射系数只缩放光路，不改变 albedo；与 Zinke 2008 的 "Backscattering Attenuation" 密度不变性互证）；
- 参数语义分工（图 12）：**纵向粗糙度控制高光宽度（shininess），方位向粗糙度控制整体柔度（softness）**——两者对多散射外观的影响都是"感知线性"的；
- **undercoat 用着色模拟**（图 14）：动物底绒几何太密无法显式建模 → 用"沿毛长抬高方位向粗糙度（类似 phase function 前/后向散射）+ 纵向粗糙度（蜷曲）"制造**高密度错觉**，同时保持外guard fur 轮廓处的通透——**"几何上做不起的，交给着色参数"**。

### 性能（图 10，Hyperion；512×512 / 256 spp；八面光 + 一个 IBL）

| 指标 | d'Eon 2011 求积版 | 本篇 near-field | 变化 |
|---|---|---|---|
| 总渲染时间 | 869 min | **76 min** | **>10×** |
| 其中毛发着色 | 800 min（>80% 总时长） | **44 min** | **~20×（着色器级）** |
| RMSE（vs 高样本参考） | 0.0146729 | **0.0140999** | 略优 |
| 采样方差 | −10.455 dB | −10.3591 dB | 相当（噪声水平未变差） |

> 原文补一句值得记的："**there is a diminishing return for further shading optimization**"——着色被压到与几何遍历可比之后，毛发渲染的下一战场回到几何侧。

## Key Contribution

1. **一个"路径追踪友好"的守恒纤维散射模型**：单次求值 O(1)、无偏、近景更准——把离线毛发从"着色瓶颈"变成"可负担"；
2. **logistic 方位分布**：给"方位粗糙度"一个同时具备解析归一 + 解析可采样 + 单分布覆盖全粗糙度的闭式（此前的"守恒 + 可采样 + 便宜"三选一被打破）；
3. **第四叶闭式**：把"无穷阶守恒"从"加 10 个 lobe"变成"加 1 个 lobe"——高阶能量的几何级数观察是核心；
4. **六参数的感知均匀体系**：生产采用层面的决定性改动——"物理正确"模型如果参数不可控，剧组不会用；本篇把 d'Eon 2011 的模型**真正送进了片厂日常**。

## Why It Works

1. **"让最擅长做积分的系统去做积分"**：MC 渲染器每个样本本来就是一条光路，宽度方向的分布被采样天然覆盖；把内层积分从"每次求值里"挪到"样本维度上"，复杂度从 70× 降为 1×——**代价是放弃了 far-field 的聚合解析形式**（换来的是无偏 + 更准）；
2. **换分布族的标准**：要当"采样分布"必须三条齐备——归一化闭式 / CDF 闭式 / 反 CDF 闭式。wrapped Gaussian 缺后两条；logistic 三条全有——**"能不能采样"是选型的硬指标**；
3. **无穷项折闭式的前提**：几何级数收敛（$fT<1$，物理上总成立）+ **残余项的能量足够小、形状足够乱**（乱到"各向同性"是合理粗化）——两条都成立时，"形状细节的损失"买来"守恒"；
4. **参数化的本质是"反解一个观测量"**：艺术家给的是"看到的颜色"，工程侧要的是"吸收系数"——中间隔着一个多散射问题；本篇用"离线的测量表 + 拟合"把这座桥搭好，**密度不变性**让这张表可以跨资产复用。

## Limitations

1. **第四叶的方位向"各向同性"是近似**：残余能量小但非零；高阶方位分布的真实形状被丢弃（原文承认，属"可接受粗化"）；
2. **主反射涂层 / undercoat 是启发式控制**，不是物理建模（"smooth cuticles + scattering medulla" 类物种）；
3. **未与实测数据对抗**：authors 自述 future work 是"用 Yan et al. 的真实毛皮反射剖面验证本模型"——即本文的验证来自自身仿真与 furnace test，不是测量；
4. **非可分离性（方位×纵向耦合）不做处理**：d'Eon 2014 的 non-separable 模型是后续（库外）；
5. **论文语境是离线路径追踪**：512×512/256spp 的片厂渲染；"实时化"另需讨论（其思想在实时侧的影响更多通过参数化与近似结构体现）。

## Game Development Relevance

**4/5。两条使用路径：离线（CGI 预告/过场）+ 光追毛发（离线管线的实时化）。**

- **与《巫师 3》重制版 LSS 的对接点（推断）**：光追毛发的核心问题正是"每像素大量样本 × 每路径多次毛发求值"——本篇"单次求值廉价化"是那条路线的**教科书第一步**。你的"光追档"毛发相关问题（OverDraw → 采样成本）可以直接在这条线上思考；
- **"参数化 = 采用开关"（对 NGR 材质/毛发规范的可迁移结论）**：任何要交给美术长期使用的 shading 体系，**观感参数化与物理正确同等重要**——本篇的 βM/βN/颜色三件套是影视侧的标准答案形态。对照你熟悉的东西：UE 的"轴向/方位向粗糙度分离"参数家族与此同构；
- **分档新样本（推断）**：本篇的降档可以发生在**"粗略化高阶项的分布形状"**上（第四叶方位向各向同性），而**不是砍掉高阶项**——守恒被保住、换的是形状细节。与"档位 = 换表示还是缩参数"（[[Scalability and Quality Tiers]]）合读：**这里换的是"近似层级"，成本几乎为零、能量账不破**；
- **furnace test 又一文档**：同期你在 PBR 侧要做的 furnace test，在毛发侧的标准形态就是本篇图 5（+ d'Eon 的白环境实验）——**"零吸收 + 均匀白环境 → 应不可见"** 是跨材质域通用的体检；
- **颜色的"密度不变性"**（对毛发资产量产有用）：多散射色主要由单纤维吸收决定、几乎不随发量密度变——**同一套颜色参数可跨发量复用**（低配少发丝时不必重调色）。

## Unreal Engine Relevance

- UE HairStrands（Groom）的着色体系是"Marschner 家族的多 lobe 结构 + 多散射近似/路径追踪选项"；**本文的模型不是 UE 的默认实现**（pbrt-v4 用的是 d'Eon 2011，见其笔记）；
- 可迁移到引擎侧的三条（推断，非实装断言）：
  1. **参数分层**：把内部物理参数（ν、σa）与美术参数（βM、βN、颜色）显式分开——自查项："我的 hair shader 参数是物理量还是观感量？美术调一个参数时知不知道自己在动什么"；
  2. **高阶项策略**：如果要补能量，优先检查"能不能用一项闭式代表高阶"（而不是逐阶加 lobe）；
  3. **在 Path Tracer / MRQ 路径下**（UE Path Tracer 渲染过场）：毛发单次求值成本 × 样本数 × 命中次数，就是本篇的问题场景原样复现——**可在引擎内复现"70× vs 1×"的量级差实验**（用 d'Eon 求积 vs near-field 的对照思路做成本拆解）。

## Technology Evolution

```text
毛发谱系（渲染轴）——六节点闭合：

1989 Kajiya-Kay（经验侧：texel + 圆锥高光）        ★ 库内
        ↓
2003 Marschner（物理侧：R/TT/TRT + 方位/纵向分解）  ★ 库内
        ↓
2004 Scheuermann（实时工程侧：发片 + 取消排序）      ★ 库内
        ↓
2008 Zinke-Yuksel（多散射近似：双散射）             ★ 库内
        ↓
2011 d'Eon（能量侧：守恒 M_p + 取消求根）           ★ 库内
        ↓
★ 2016 本篇（生产侧：near-field 取消求积 + logistic + 第四叶 + 参数化）
        → "为路径追踪时代重写"完成；片厂日常可用
        ↓
2026 LSS 光追曲线基元（离线管线实时化方向）/ Neuroll 神经仿真（仿真轴）
```

## Relationships

### Based On

- [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] —— **沿用**守恒 $M_p$、方位几何 $\Phi(p,h)$、完美采样（dMH13）；本篇 = 其"方位向 + 工程化"的再设计
- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— 分解框架（R/TT/TRT、方位×纵向）；本篇在 Discussion 中解释**为什么纵向仍选 d'Eon 而非 logistic**（d'Eon 函数据有"对立体角各向同性"的正确极限 + 完美采样；Marschner 高斯在近各向同性时会向切线方向漏能 + 1/cosθd 未采样）

### Improves / Productionizes

- [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] —— 把"离线可用"改进为"路径追踪可用 + 艺术可控"

### Related

- [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] —— **正交关系**：本篇让暴力多散射可行（从而规避近似），但也明确承认双散射类方法在**需要便宜近似时**依然有用（图 1 的对比服务于此论证）
- [[Hair Rendering]] —— 本篇补上其"生产化"横切面（第 6 节点）
- [[2026-10-05-Neural Emission Fields — Real-time Rendering of Pre-integrated Neural Emitters|NEF]] —— **"内层积分的两种去向"对照**：本篇把积分**外包**给采样器（每次求值 1 次）；NEF 把积分**预集成**成神经场（每次求值 0 积分）——同一家族的两个相反方向
- [[Multiple Scattering and Energy Compensation]] —— "守恒"主题在纤维域的又一实现（闭式折无穷阶）

### Followed By

- d'Eon et al. 2014（non-separable 模型，库外记名）；2016+ 影视毛发着色的生产基线之一
- 2026 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]（仿真轴）——与着色轴并行

## Personal Knowledge State

- **user_level: Normal（推断）**。你是做角色 VFX 与分档的：本篇的**结论层**（为什么贵、怎么便宜、参数为什么这样设计）直接可读；数学细节（logistic 重参数化、级数推导）可跳过。
- **阅读位置建议**：你的毛发线当前进行到 **Marschner（深读中）→ d'Eon 2011（能量页）**；本篇是 **"生产化页"**——在 d'Eon 之后读，或先只读 §3.2/§3.4/§4（积分搬家 / 第四叶 / 参数化）三节 + 图 5/10（furnace test 与性能对照）。**非必经，但读完后"毛发着色"从 1989 到 2016 的完整链就闭合了。**

## Learning Value

- **新增 3 条自测（并入 [[Hair Rendering]]，总数 18 → 21 条）**：
  1. 为什么"把沿纤维宽度的积分挪进路径采样"是**无偏**的？它换掉了什么代价（70 点求积 → 1 次求值），又放弃了什么（far-field 聚合解析形式）？
  2. "第四叶能代表全部高阶"的两个前提是什么？（级数可和 + 残余能量小且方位形状可粗化为各向同性）为什么它让白发掠射也守恒？
  3. 本篇说参数化是"生产采用的决定项"——**对照 d'Eon 的物理参数**，感知均匀映射解决了什么具体问题？（多散射的非线性 + 美术直觉；"调一个参数时知道自己在动高光宽度还是柔度"）
- **一条可迁移判据**：**"内层积分有三种归宿——算（求积/解析）、外包（交给采样器）、预集成（学成函数）；选哪种先问'谁天生会做这件事'"**（本篇选"外包给 MC 渲染器"；NEF 选"预集成"；d'Eon 选"算但换更便宜的形式"）。

## Visualization

![[毛发生产化一跳_Chiang 2016 图解.html]]

## Notes

- **原文核对（本次已下载作者官方 PDF、pypdf 全文抽取、逐节核对）**：near-field 公式（Eq 4）、求积的三大问题（§3.2）、logistic 三条性质与重参数化（§3.3 / Appendix A）、第四叶几何级数（Eq 6）与图 4/5、参数化映射（Eq 7/8/9）、密度不变性论证（§4.2）、undercoat 着色近似（§4.3）、性能表与 RMSE/方差（图 10 与 §5）、六参数清单（§5）、"first time possible in production" 与 "diminishing return" 原句（§5）、"successfully adopted by production at WDAS"（§7）——**均为原文逐条确认**；
- **元数据**：CGF 35(2): 275–283，DOI **10.1111/cgf.12830**（Crossref 核实；另有 SIGGRAPH 2015 Talks 1 页版 DOI 10.1145/2775280.2792559，同一工作的会议宣讲形态，按"一个研究实体"处理不另建文件）；
- **获取路径留痕**：Pixar 旧库直链已失效（站点改版为 Squarespace）→ **Ke-Sen Huang 的 EG2016 论文页 → 作者个人站 `benedikt-bitterli.me/pchfm/`**（作者归档路径第 8 次生效）；Wayback API 429 未硬刚（沿用旧教训）；
- **更正动作**：本次同步修正 [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] 与 [[2026-10-07]] 中的 "Pixar" 误记（见两处勘误行）。
