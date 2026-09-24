---
type: paper
title: "A Multi-Faceted Exploration (Parts 1–4)"
authors: [Stephen Hill]
year: 2018
published: "2018-05-13 ~ 2019-03-30（系列四篇）"
venue: "Self Shadow 博客技术文章系列（Part 1 / 2 / 3 / 4 + 2 个 WebGL 在线演示；Part 4 附带托管 ILM Turquin 技术报告）"
url: "https://blog.selfshadow.com/2018/05/13/multi-faceted-part-1/"
code: ""
project_page: "https://blog.selfshadow.com/（WebGL 演示：multi_fresnel.html / multi_compare_1.html）"
category: [rendering, pbr, multiple-scattering, energy-compensation, real-time]
importance: "A（经典）"
historical_importance: 4
game_relevance: 5
production_readiness: "Industry Adopted（路线 ③ 的工程化中段；④⑤ 两族的直接上游）"
user_level: Normal
status: unread
---

# A Multi-Faceted Exploration（2018–2019，Self Shadow 博客系列）

> 入库于 2026-09-24。**原文已逐篇核对**（Part 2 / 3 / 4 全文，Part 1 为问题陈述篇）。
> 它是"能量账本"谱系里**唯一一条"全过程记录"**：从发现问题（能量丢了）到最终形态（3 条 MAD + 单张 4 通道贴图）全部公开，每一步的动机、失败与取舍都写在原文里。

## TL;DR

**让 Kulla-Conty 的多次散射补偿真正"实时可用"的那座桥。** 四篇文章走完一条完整的工程化路径：

1. **Part 2**：修正 Imageworks 的 $F_{ms}$ —— 把"单散射事件"从几何级数里剔除（$F_{avg} \to F_{avg}^2$），**这是"重复计数"的账务修正，不是新物理**；
2. **Part 3**：不再用"猜"的 Fresnel 标量，改为**从 Heitz 模型预计算 $E_{Fms}(\mu_o)$** → 立刻撞上 3D LUT 问题（$F_0$ 是第三维）→ 用 **Schlick 的线性性**把 $F_0$ 拆成多项式 → **2D LUT**；
3. **Part 4**：给出预计算算法（随机游走**能量吞吐向量**）+ 重拟合 → **单张 4 通道贴图 + 3 条 MAD**，运行时可重建任意 $F_0$ 的 $E_{Fms}$；
4. **附**：托管 **Turquin（ILM）技术报告** —— 路线 ⑤（缩放 lobe）的原始文献，库内挂了 4 天的"**待核实**"条目就此落地。

> **一句话定位**：**它把"$1-E(\mu)$ 表的拟合"从"每个材质一条曲线"推进到"任何 $F_0$ 共用一张贴图"。** 关键数学借口只有一条：**Schlick Fresnel 对 $F_0$ 是线性的** —— [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] 写下那个形式时未必想到，26 年后它是降维的钥匙。

```text
3D LUT（μo × roughness × F0，RGB 三次采样）   ← 直接存 EFms 的代价
      ↓ Schlick 线性性：F = F0(1−s) + s   ⇒   EFms = Σ wi·F0^i
2D LUT（μo × roughness，存各阶 wi）           ← Part 3
      ↓ 高阶能量很少：重拟合为三次多项式
1 张 4 通道贴图 + 3 条 MAD                     ← Part 4
```

## Problem

单散射微面 BRDF 在**高粗糙度端系统性丢失能量**（furnace test 下右半的球明显变暗）。这条线上已有 Heitz 2016（精确、但求值要随机数）与 Kulla-Conty 2017（便宜、但用"漫反射形状的 lobe"去补）。

**Hill 系列要解决的是一个工程问题**：Kulla-Conty 的补法在高粗糙度端**仍与 Heitz 真值有肉眼可见的偏差**（part 2 的 furnace 对比图：Imageworks 偏亮、欠饱和）。偏差来自两处：

1. $F_{ms}$ 的推导里**混入了已经被 $f_{ss}$ 算过一次的单散射事件**（重复计数）；
2. $F_{ms}$ 是"一个标量 × 一个漫反射形状"，而 Heitz 2016 早就指出**次级 lobe 不是漫反射形状，而是主 lobe 的缩小版**。

## Historical Context

放在库内"多次散射补法"谱系里（[[Multiple Scattering and Energy Compensation]] 的完整时间线）：

```text
2014  Heitz（JCGT 综述）提出三个问题 —— Q1 次级 lobe 什么形状？
                                       Q2 能用一个解析函数代替吗？
                                       Q3 能预计算成小表吗？
        ↓
2016  Heitz et al.：回答 Q1（随机游走真值；次级 lobe ≈ 缩小的主 lobe）
2017  Kulla & Conty：回答 Q2/Q3（Kelemen 漫反射 lobe + 32×32 表）
        ↓
2018  本系列（Hill）：发现 Kulla-Conty 的 Fms 重复计数 → 修正；
    ★ 并沿着 Q3 走到极致：不预计算"某材质的 EFms"，而是预计算"所有 F0 的系数"
        ↓
2019  Fdez-Agüera：把同一族思想搬到 IBL（零新增资源）—— 直接建立在本文之上
```

**本系列所处的精确位置**：路线 ③（查表 lobe）的**中段** —— [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Kulla-Conty]] 给了形状（Kelemen lobe），**Hill 给了形状里那个 Fresnel 因子的"正确算法 + 最终存储形式"**，[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)|Fdez-Agüera]] 再把它搬进 IBL。库内此前有 ③ 的两端而无中段 —— **今天补上的正是这一段**。

## Core Idea 一：$F_{ms}$ 的账务修正（Part 2）

Imageworks 的 $F_{ms}$ 来自 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Jakob et al. 2014 §5.6]] 的**几何级数**：每次弹射留下 $F_{avg}$、逃逸概率 $E_{avg}$、继续弹射 $1-E_{avg}$：

$$F_{ms} = F_{avg}E_{avg} + F_{avg}E_{avg}F_{avg}(1-E_{avg}) + \cdots = \frac{F_{avg}E_{avg}}{1 - F_{avg}(1-E_{avg})}$$

**问题**：第一项 $F_{avg}E_{avg}$ 就是**单散射**——它已经被 $f_{ss}$ 贡献过了，**在这里被数了第二遍**。

修正 = 级数从 $k=1$ 开始（剔除单散射项）+ 用 $\frac{1}{1-E_{avg}}$ 重归一化（保证 $F_{avg}=1$ 时 $F_{ms}=1$，因为 lobe 本身已有幅度 $1-E(\mu_o)$）：

$$\boxed{F_{ms} = \frac{F_{avg}^2\,E_{avg}}{1 - F_{avg}(1-E_{avg})}}$$

**分子从 $F_{avg}$ 变成 $F_{avg}^2$，仅此一处。** 效果：高粗糙度端从"偏亮、欠饱和"变成"略偏暗、略过饱和"——**整体更接近 Heitz 真值**。

> **原文自note**：该式曾在课件中出现错误（Hill 自述 misedit）；本修正已被并入 **Imageworks slides v2**（另附 Christopher Kulla 的 $E(\mu)$ / $E_{avg}$ 数值拟合，以及若干 speaker notes 勘误）。
> **可复用判据**：**"补能量"最常见的 bug 不是算错物理，而是把已经算过的份额重复计入** —— 拿到任何补偿项，先问"它减去单散射了吗"。

## Core Idea 二：从"猜标量"到"预计算 $E_{Fms}$"（Part 3）

$F_{ms}$ 终究是"一个标量"（甚至假设方向无关），而 Heitz 模型证明**多次散射的 Fresnel 是有方向分布的**。Part 3 的做法：

$$f_{ms} = \frac{F_{ms}\,(1-E(\mu_o))\,(1-E(\mu_i))}{\pi(1-E_{avg})} \quad\Longrightarrow\quad f_{ms} = \frac{\mathbf{E_{Fms}(\mu_o)}\,(1-E(\mu_i))}{\pi(1-E_{avg})}$$

其中 $E_{Fms}(\mu_o)$ 是**直接从 Heitz 随机游走模型预计算**的"多次散射方向 albedo（含 Fresnel）"。

**结果**：furnace test 下与 Heitz 几乎不可分辨（Part 3 Fig 2）。

**新问题**：$E_{Fms}$ 依赖 $(\mu_o, \text{roughness}, F_0)$ —— **3D LUT**，且彩色金属（铜）要 R/G/B **查三次**。对实时管线来说不可接受。

**次要代价（原文明说）**：这一版**破坏互易性**（$\mu_o$ 与 $\mu_i$ 互换结果不同）——对实时无碍，对双向路径追踪是问题。

## Core Idea 三：Schlick 线性性 → $F_0$ 的多项式拆分（Part 3）

**关键观察**：$E_{Fms}$ 对 $F_0$ 的依赖是**多项式结构**的。起点是同一个"split"思路在单散射上的成功（[[Split-Sum Approximation|Karis 2013]] / Lazarov 2013 / Drobot 2013 的 IBL 预积分）：

$$\text{Schlick: } F = F_0 + (1-F_0)(1-\omega_h\!\cdot\!\omega_o)^5 = F_0(1-s) + s,\quad s=(1-\omega_h\!\cdot\!\omega_o)^5$$

**对 $F_0$ 线性** → 积分可以拆成"被 $F_0$ 染色"与"不染色"两部分：

$$E_{Fss}(\mu_o) = w_0 + w_1 F_0 \quad\Longleftrightarrow\quad \text{（预积分表从 3D 降为 2D）}$$

推到多次散射：光在微面上弹 1…N 次，每弹一次就多乘一次 $\approx F_0$，于是

$$\boxed{E_{Fms}(\mu_o) = \sum_{i=0}^{N-1} w_i(\mu_o, \text{roughness})\; F_0^{\,i}}$$

**$w_i$ 与 $F_0$ 无关** → 可以预计算进 **2D LUT**（坐标只有 $\mu_o$ 与 roughness），运行时对**任意 $F_0$** 重建 $E_{Fms}$。**"每个材质一条曲线"变成"全部材质共用一族系数"。**

> **可复用判据（本系列最值钱的一条）**：**看一个表格能不能降维，先看它依赖的"材质参数"在 BRDF 里是不是线性的。**（这也是 [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] 那个形式在 26 年后仍在被"继续兑付"的原因。）

## Technical Approach：预计算算法与重拟合（Part 4）

**① 预计算 $w_i$：随机游走 + 能量吞吐向量**

每次随机游走携带一个**向量** $e_0 \ldots e_{N-1}$（初始 $e_0=1$，其余 0），跟踪"被 $F_0^i$ 缩放的能量份额"。每遇一次散射事件（Schlick 因子 $s$），更新：

```glsl
for (int i = N - 1; i > 0; i--)
    e[i] = mix(e[i - 1], e[i], s);   // e[i] = s*e[i] + (1-s)*e[i-1]
e[0] *= s;
```

多次游走后取平均（**只统计多弹射的游走**，单散射计零）→ 即 $w_i$ 系数。

**② 重拟合为"单张 4 通道贴图 + 3 条 MAD"**

$w_6$ 等高阶项仍有残余能量 → 原文承认"**完全精确需要 2–3 张贴图**，还要不少 shader 数学"。但能量集中在前几阶 → 用 **MiniMaxApproximation**（最小化**相对**误差，对低端曲线尤其重要）把源多项式重拟合为三次曲线：

```glsl
vec3 EFms(vec3 F0, float mu, float rough)
{
    vec4 w = EFmsFactors(mu, rough);              // 一次纹理采样，4 通道
    return w.x + F0*(w.y + F0*(w.z + F0*w.w));    // Horner：3 条 MAD
}
```

> **"更准 vs 能用"在方案内部再演一轮**：3D LUT（准、贵）→ 2D LUT（准、中）→ **单张贴图 + 3 MAD（够准、几乎免费）**。每一步都靠"被舍弃的那部分误差足够小"续命：第一次舍弃的是"$F_0$ 的独立性"（靠 Schlick 线性性），第二次舍弃的是"高阶弹射的精确形状"（靠能量集中假设）。

**演示**：两个 WebGL demo（`multi_fresnel.html`：复现 $E_{Fms}$ 与 $w_i$ 分解；`multi_compare_1.html`：左半 Imageworks+改良 vs 右半 Heitz 真值）。

## 附：Turquin TR —— 路线 ⑤ 的原始文献（Part 4 托管，今日一并核对）

Part 4 以"Bonus exercise"形式托管了 **Manu（Emmanuel Turquin，ILM）** 的技术报告：

- **文件**：《Practical multiple scattering compensation for microfacet models》5 页，×[下载](https://blog.selfshadow.com/publications/turquin/ms_comp_final.pdf)（今日已下载核对）
- **方法**：**不换 lobe 形状，直接缩放已有主 lobe**：
  $$\rho = \rho_{ss} + F_{ms}k_{ms}(\omega_o)\,\rho_{ss},\qquad k_{ms}(\omega_o) = \frac{1-E_{ss}(\omega_o)}{E_{ss}(\omega_o)}$$
  由构造保证 $E = 1$（能量守恒）；$F_{ms}$ 直接复用 Kulla-Conty 的式（7）。
- **动机（原文引 Heitz 2016 Fig 15）**：**"次级 lobe 不像漫反射，而像主 lobe 的缩小版"** —— 所以补法应当是"把主 lobe 放大"，而不是"贴一个漫反射 lobe 上去"。**这句话解释了路线 ⑤ 为什么"最便宜且形状对"**。
- **原文自评**：比较"基本是定性的，更彻底的定量分析留作未来工作"；并有未来方向清单（给所有光滑 LUT 找解析拟合、考虑不拆 Fresnel 直接上 3D/4D 表……）。
- **实装去向**：Unity HDRP（Lagarde & Golubev 2018 的 *Multiple Scattering GGX* 一节）与 Google Filament —— **库内"待核实"的 `[Lagarde18]` 原始条目即"ACM SIGGRAPH 2018 Advances in Real-Time Rendering 课程页面"**（Part 4 脚注给出）；配合本 TR，路线 ⑤ 的**两条原始文献至此都可直接访问**。

## Why It Works

- $F_{ms}$ 修正解决的是**账务错误**（重复计数），不是物理近似 → 修正后"必然"更接近真值；
- Schlick 线性性是**精确的代数性质**（不是近似），拆分本身零误差；误差只来自"忽略 $F_0$ 高次项"与"重拟合" —— 都由"能量集中在前几阶"兜底；
- 最终形式（单贴图 + 3 MAD）保留了**互易性以外的全部工程优点**，且成本进入"约等于 0"区间。

## Limitations

- **互易性被破坏**（Part 3 明说；实时可接受，BDPT 等场景不可用）；
- 完全精确需要 2–3 张贴图（重拟合是有损的）；
- 系列只处理**导体/解析光**（Part 2 脚注明说"忽略折射与透射"；IBL 留给后续工作 → Fdez-Agüera 2019）；介质/电介质版本由 Kulla-Conty slides 用另一组表处理；
- 系列**未完结**（Part 4 结尾预告"下一文做更详细的定量比较"——后续未见发布；此处以现有四篇为准）。

## Game Development Relevance

- **直接对应你引擎里可能的"能量补偿"开关**：UE 侧的 SunSky/IBL 场景若采用 ③④ 一族实现，**其系数表（DFG 或专用 LUT）的来历就是本系列 + Karis 2013**；
- **furnace test 的"进化史"在本系列可逐帧看到**（Part 2 Fig 1→Fig 5→Part 3 Fig 2）：这正是你那条"查两头"实测题的**对照图册** —— 校正不足（偏亮）与校正过头（偏暗）的差异都有人给你画好了；
- **分档含义**：3 条 MAD + 1 张 4 通道贴图 ≈ 零边际成本 → **属于"应默认开启"的一类**（与 [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)|Fdez-Agüera]] 的结论一致）；
- **方法论含义**：本系列展示了"**同一条近似可以在其内部连续降维三次**"的完整过程 —— 对任何"贵但准"的预计算方案都适用。

## Unreal Engine Relevance

- UE 的 `EnvBRDF`（R16G16）承载的是**单散射**的 split-sum；若管线采用多次散射补偿，其多散射项在 ④ 路线下正是"复用同一张表"（`Ess = f_ab.x + f_ab.y`）；
- 本系列的 **4 通道贴图 + Horner 3 MAD** 是"自研管线实现能量补偿"的最小可行形态 —— 适合作为 [[Real-Time VFX Performance Budgeting|预算]] 讨论里"新增固定开销 ≈ 0"的参照样本；
- 注意：**UE 默认不无条件开多重散射补偿**；是否启用属于材质/管线决策（见 [[Multiple Scattering and Energy Compensation]] "Technologies" 节）。

## Technology Evolution

```text
2014  Heitz 综述（Q1/Q2/Q3 三问）
   ↓
2016  Heitz et al.（真值；次级 lobe ≈ 缩小的主 lobe）
   ↓
2017  Kulla & Conty（Kelemen lobe + 32×32 表）—— Imageworks 课件 v1/v2
   ↓
2018  ★ 本系列：
      Part 2  Fms 修正（剔除重复计数：Favg → Favg²）
      Part 3  EFms 预计算 + Schlick 拆分 → Σ wi·F0^i（2D LUT）
      Part 3′ Turquin TR（缩放主 lobe 法；ILM，同期）
   ↓
2019  Part 4（预计算细节 + 单贴图 3 MAD + WebGL 演示）
   ↓
2019  Fdez-Agüera（IBL 版：零新增资源；承接本系列系数思想）
   ↓
2022  d'Eon《Hitchhiker's Guide》（把这些都收进"模型问题"的坐标系）
```

## Relationships

### Based On

- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] —— 修正对象与 $F_{ms}$ 母式；本系列是其"**v2.5**"
- [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] —— $E_{Fms}$ 的预计算来源（真值模型）
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] —— **降维的数学钥匙**（对 $F_0$ 线性）
- [[Split-Sum Approximation]] —— "同一手法"的另一实例：先拆 $F_0$、再查表（单散射版 vs 多次散射版）

### Enabled

- [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] —— 直接下游（IBL 化）
- Turquin TR → Unity HDRP / Filament 的实装（**路线 ⑤**）

### Contrasts

- 与路线 ⑤（缩放主 lobe）**同源不同分支**：⑤ 认为"形状=缩小主 lobe、Fresnel 用一个标量"；本系列认为"Fresnel 要按方向预计算"。**两条分支的取舍点：$E_{Fms}$ 的方向依赖值不值得多一张表。**

## Personal Knowledge State

- **user_level: Normal**。前置（[[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)|Schlick]] / [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Kulla-Conty]] / [[Split-Sum Approximation]]）都在库且已研读；
- **两条一句话检验**：
  1. 能说出"**$F_{ms}$ 修正修正的是重复计数，不是物理**"；
  2. 能说出"**Schlick 对 $F_0$ 线性 → 积分能拆 → 3D 表降 2D**"。
- 不计入 PBR 收口 25 条清单（同 [[Multiple Scattering and Energy Compensation]] 惯例：**它是该概念的主文献群之一，标"读了"**）。

## Learning Value

- **当前最高价值**：补上了 ③ 路线的**中段**，使"Kulla-Conty 的 4KB 表"到"Fdez-Agüera 的一行加法"之间的**技术跳跃不再有断点**；
- 顺带解决库内两处**待核实**：Turquin TR（⑤ 原始文献）与 `[Lagarde18]` 原始书目条目（SIGGRAPH 2018 Advances 课程页）；
- 为 furnace test 提供**对照图册**（校正不足/过头的视觉差异）。

## Visualization

![[多次散射_Hill系列_从3D查表到单张贴图图解]]

## Notes

- **与 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Kulla-Conty]] 勘误的关系**：Kulla-Conty 的 $F_{avg}$ 一式有官方勘误（Turquin 指出，v2 修正）；本系列 Part 2 是**同一问题的完整推导版**（且含 Hill 自述的 slides misedit 勘误）。两件事合起来说明：**这一族的公式在"课件 → 引擎"的传播链上反复出错，落 shader 前务必核对勘误**（与本库 9-20 立下的规矩一致）；
- **原系列 `[Hill 2018a / 2018b]` 引用的对应**（澄清库内旧记载）：**2018a ≈ Part 2/3**（Fresnel 的 $F_0$ 幂级数拆分）；**2018b ≈ Part 4 的完整 $E_{Fms}$ 方案**（"更准但要 2–3 张贴图、且不管 IBL" —— 前者指重拟合前的完整系数存储，后者指系列只做解析光）。**今日以原文核对为准**；
- 阅读顺序建议：**Part 2 → Part 3 → Part 4**（Part 1 可跳）；每篇末 link 到下一篇，URL 均可直接访问。
