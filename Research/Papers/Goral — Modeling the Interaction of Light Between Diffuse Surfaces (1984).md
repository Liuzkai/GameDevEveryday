---
type: paper
title: "Modeling the Interaction of Light Between Diffuse Surfaces"
authors: ["Cindy M. Goral", "Kenneth E. Torrance", "Donald P. Greenberg", "Bennett Battaile"]
year: 1984
published: 1984
venue: "SIGGRAPH '84 (Computer Graphics 18(3), pp.213-222)"
url: "http://www0.cs.ucl.ac.uk/research/vr/Projects/VLF/vlfpapers/radiosity/Goral__Modeling_the_Interaction_of_Light_Between_Diffuse_Surfaces.pdf"
doi: "10.1145/800031.808601"
code: ""
project_page: ""
category: ["rendering", "global-illumination", "precomputed-lighting", "radiosity"]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（思想层：'视角无关 ⇒ 可预计算'一句话开出烘焙/预计算光照整条路线，Lightmass/Lightmap 的直系祖先；具体矩阵解法本身已被 progressive refinement 与路径追踪取代）"
user_level: Normal
status: unread
---

# Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)

## TL;DR

**Radiosity（辐射度方法）的起点论文**：把热工程里的辐射换热计算（form factor + 封闭腔能量平衡）第一次搬进图形学，解决了当时所有局部光照模型都算不了的问题——**漫反射表面之间的互相照亮**（红墙把地板染红 = color bleeding）。做法：把场景拆成面片，每对面片之间算一个**纯几何**的形状因子 $F_{ij}$，能量平衡变成 N 元线性方程组 $B_j = E_j + \rho_j\sum_i B_i F_{ij}$，**一次求解、处处可用**。论文里最重要的一句话藏在摘要里：**"resultant surface intensities are independent of observer position, and thus environments can be preprocessed for dynamic sequences"——视角无关，所以可以预计算**。这就是此后 40 年游戏烘焙光照（含 UE Lightmass/Lightmap）的合法性的第一份书面证明。最后用**实体纸板模型 + 实拍照片**做了定性验证——图形学早期少见的"双真值"样本。

![[Research/Files/Pictures/Goral 1984 — 图6 红蓝灰盒细分收敛与色渗RGB扫描线.png]]
**读图**：同一只红墙/蓝墙/灰墙的盒子，墙面细分从每边 1 片加到 49 片。细分越细，灰墙上的红/蓝色渗（color bleeding）越平滑，下方 RGB 扫描线强度曲线从"跳台阶"变成连续曲线——间接光的细节藏在不同面片间的能量交换里。（图 6，Goral et al. 1984）

## Problem

1984 年的反射模型全部是"局部"的——只算光源直接照到的那一份：

1. **Phong/Blinn/Cook-Torrance 都不管互反射**：漫反射项只算"光源 → 表面"，物体把光弹到别的物体上再弹回来的那份能量整体缺失。为了补，工程上塞一个**随手拍的 ambient 常数**——"死光"，毫无物理依据；
2. **Cook-Torrance 是能量平衡模型，却吃不到输入**：它需要表面收到的入射能量 $E$，而"当时没有任何图像合成方法能给出孤立物体之外的这份信息"（原文原话——对 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 而言这是悬在头上的账）；
3. **Whitted 光线追踪只覆盖镜面特解**：递归光线只沿镜面反射/折射方向追，漫反射互反射既采不到也没能量守恒保证——"point-sampled 信息不足以支撑能量平衡模型"。

## Historical Context

```text
Gouraud 1971 / Phong 1973（局部漫反射着色；ambient 常数开始混入）
  ↓
Torrance-Sparrow 1967（物理光学）→ Blinn 1977 → Cook-Torrance 1981（局部 BRDF 精化；Torrance 是本文二作）
  ↓
Whitted 1980（递归光线追踪：镜面互反射特解）
  ↓
★ Goral / Torrance / Greenberg / Battaile 1984（本篇：热工程 radiosity 进 CG，漫反射互反射特解）
  ↓
Kajiya 1986（渲染方程统一表述：radiosity 与 ray tracing 都成为它的特解——见 [[Kajiya — The Rendering Equation (1986)]]）
  ↓
Cohen & Greenberg 1985（hemi-cube：可见性/复杂环境）→ Cohen et al. 1988（progressive refinement：交互化）
  ↓
Hanrahan et al. 1991（层次 radiosity）→ Ward 1988 辐照度缓存（另一条"按需缓存"岔路）→ Keller 1997 Instant Radiosity（VPL 重解读）
  ↓
游戏侧：预计算 GI / 烘焙（Lightmass、Lightmap、SH Probe）——本文摘要那句话的工业化后代
```

**两个值得记住的时间差**：Radiosity 比 Kajiya 渲染方程**早两年**（先有漫反射特解，后有统一方程）；本文二作 Torrance 正是 Cook-Torrance BRDF 的那位 Torrance——局部反射模型与全局传输两条线在 Cornell 一个人身上交汇。

## Previous Work

- **热工程辐射换热**（Siegel & Howell《Thermal Radiation Heat Transfer》1981；Sparrow 1963 的 form factor 角度因子公式）：本文的直接源头——封闭腔内漫反射灰体换热的矩阵解法原样搬入，连"form factor"这个词都是热工程术语；
- **有限元的网格思想**：面片细分（elements）自认"similar to mesh generation in finite element analysis"；
- **Stokes 定理的围道积分**（Sparrow 1963）：把 $F_{ij}$ 的二重面积分降为沿边界的线积分，是本文实现可算的关键。

## Core Idea

**三步把"满房间的间接光"变成一个线性方程组：**

1. **Radiosity（辐射出射度）$B_j$**：单位时间单位面积离开表面 j 的全部能量（自发光 + 直接光 + 所有间接反弹的总和）。对理想漫反射面，它从任何方向看都一样——**这正是"视角无关"的物理根源**；
2. **形状因子 $F_{ij}$**：面片 i 发出的能量中直接落到面片 j 的比例。对漫反射它**纯几何**（只依赖形状/大小/位置/朝向），双重面积分 $F_{ij}=\frac{1}{A_i}\int_{A_i}\int_{A_j}\frac{\cos\theta_i\cos\theta_j}{\pi r^2}\,dA_j\,dA_i$；经 Stokes 定理化为围道积分（式 13）后离散求和即可，且满足三条"免费"约束——互易 $A_iF_{ij}=A_jF_{ji}$、封闭腔能量守恒 $\sum_jF_{ij}=1$、平面/凸面 $F_{ii}=0$（可用作校验和省算）；
3. **能量平衡**：$B_j = E_j + \rho_j\sum_{i=1}^{N}B_iF_{ij}$（式 6）——每个面片的出射 = 自发光 + 反照率 × 收到的份额。N 个面片 N 个未知数，Gauss 消元一次解出**全场景所有地方的间接光**。

**一句话**：把"光在场景里弹无限次"改写成"面片网络上解一次线性系统"——**用未知数从"时间维度"（反复追踪）挪到"空间维度"（N² 个 F）的方式，买断了视角自由**。

## Technical Approach

- **管线**：读入场景 → 多边形细分为"elements"（数量为输入参数，类比有限元网格）→ 逐对算 F（围道积分 + 每边分段，9 点二维求积；共线段解析处理避免 r→0 奇点）→ 组装矩阵 (I−PF)b = e → **带部分主元的 Gauss 消元** → 面片常亮度显示（可选 Gouraud 式线性插值平滑）；
- **三波段 RGB**：红绿蓝各解一遍（F 只算一次、三个波长共享——几何与光谱解耦）；
- **硬件**：VAX 11/780 + C + Grinnell 帧缓冲（512×480, 27-bit）；
- **实现边界（原文自认）**：仅凸四边形、**未处理遮挡**（hidden surfaces 不在本次实现内——可见性是次年 hemi-cube 1985 的题目）。

### 实验：一只盒子讲完全部

- **图 6**（细分收敛）：每边 1→4→9→16→25→49 片，色渗随细分平滑化；
- **图 7**（反弹次数递增，本文金图）：0 次反射 = 只有直接光（黑）→ 1/2/4/8 次 → 全解。**8 次中间反射后画面已与 Radiosity 全解"visually identical"**，RGB 扫描线同时揭示"整体变亮"（能量守恒：每次反弹都把存着的能量吐回房间）；
- **图 8**：同样的常数解 + 线性插值平滑 → 更便宜的显示侧替代方案；
- **图 9（双真值验证）**：纤维板 + 平光乳胶漆（实测近理想漫反射，θ<60°）拼一只红/蓝/白盒，实拍照片与模拟对照——早期图形学少见的"模拟 vs 实物"定性验证。

![[Research/Files/Pictures/Goral 1984 — 图7 反射次数递增至Radiosity全解的收敛对比.png]]
**读图**：上排从左到右 = 0/1/2 次内部反射，下排 = 4/8 次/Radiosity 全解。0 反射画面只剩光源面板照到的部分；随着反弹次数增加，灰墙被红/蓝墙"染色"且整体变亮；8 次反射（下中）与全解（下右）肉眼已分不出——间接光能量主要集中在前几次反弹。（图 7，Goral et al. 1984）

![[Research/Files/Pictures/Goral 1984 — 图9b 实体模型照片验证.png]]
**读图**：作者用漆成红/蓝/白的纤维板拼了真实盒子实拍——红墙对白墙的染色肉眼可见，与模拟定性一致。用"实拍"当第二真值，这在 1984 年是罕见的自我验证姿势。（图 9b，Goral et al. 1984）

## Key Contribution

1. **问题首次被"可计算地"提出并求解**：漫反射互反射从"画不出来"变成"解个线性方程组"；
2. **"视角无关 ⇒ 可预计算"的书面化**（摘要原话）——预计算/烘焙光照的法理起点；
3. **form factor 三约束 + 围道积分**：给了工程可算的实现路径；
4. **color bleeding 成为教学标准案例**（红蓝盒沿用至今 40 年）；
5. **学科迁移范式**：热工程 → 图形学，"换域搬运数学机器"的经典样本（与 2026 年把神经场搬进蒙皮是同一种动作）。

## Why It Works

- **漫反射的视角无关性**把渲染方程里"出射方向"这个维度整个砍掉——积分方程降成代数方程；
- **能量守恒是构造性满足的**：F 矩阵列和为 1 + 互易性，解天然不凭空造能量（图 7 的"越弹越亮"就是能量从光源摊到全屋的过程，收敛即守恒到位）；
- **F 只依赖几何**：换灯光/改反照率不用重算 F——矩阵 (I−PF) 只在 ρ 变时重组，只改灯光时连矩阵都不用动（原文特意注明）。**这就是"烘焙成本分摊"的第一张价目表**：几何变 → 全重算；材质变 → 重解矩阵；灯光变 → 只换右端项。

## Limitations

- **只有理想漫反射**：镜面/ glossy 完全缺席（作者建议小镜面区域"事后叠加"——后来产品化成 diffuse+specular 分层）；
- **每面片常亮度**：解是分片常数，靠细分密度换细节（图 6），存储 O(N²)、求解 O(N³)——真实场景直接爆炸，**1988 progressive refinement（行式更新 + 交互出图）是它的工程救星**；
- **遮挡缺失**（本次实现）：F 不含可见性时"穿墙传光"，1985 hemi-cube 补上；
- **插值是化妆**：图 8 的平滑只改善显示，不改善解本身——"多细分 vs 插值"的权衡原文写得很诚实。

## Game Development Relevance

- **烘焙光照的开山文献**：今天游戏里每张 Lightmap、每个烘焙 probe，都是"视角无关的量提前解、运行时只查表"这句话的执行——预算语言就是本文那张价目表（几何/材质/灯光三档失效成本）；
- **红墙染地板 = 读者的 Lightmass 专题实验原型**：读者在 [[Lightmass in the Rendering Knowledge Map|Lightmass 专题]] 里设计的"移动红墙/改反照率 → 旧烘焙何时失效"三步实验，与本文红蓝盒完全同构——40 年前的论文给今天的实验提供了对照组；
- **"为什么移动端基本只烘焙"的物理根源**也在本文：漫反射间接光视角无关所以能烘；一旦要 glossy/动态光源，视角相关项就得回运行时付账；
- **与实时线的对照轴**：本文"全解一次" vs [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]"按需缓存" vs [[Keller — Instant Radiosity (1997)]]"解成一组虚拟灯"——三种把无限反弹变成有限结构的答案，在 [[Global Illumination]] 概念页谱系表里各占一行。

## Unreal Engine Relevance

- **CPU Lightmass 的光谱位置**：UE 文档口径下 Lightmass 属"静态 GI 求解器"族（漫反射为主的预计算）；本笔记**不宣称** Lightmass 直接实现 radiosity 矩阵解法——工程实现细节（photon mapping/final gather 等版本差异）需另行核实，本文提供的是**问题定义与合法性**，不是实现蓝图（与读者专题的"问题/方法/系统/数据四层区分"完全一致）；
- **Lightmap 的失效规则**能从本文读出：改几何（F 重算）＞改材质（重解）＞改灯光（只换右端）——正好对应编辑器里"移动性改 Static 变 Lightmap 重烘"的日常；
- **GPU Lightmass / Lumen** 的对照阅读入口见读者专题第 4 节"采样 → 复用 → 重建 → 失效"四问表。

## Technology Evolution

```text
1984 Goral：封闭腔矩阵解（本篇；无遮挡、O(N³)）
  → 1985 hemi-cube（可见性进 form factor）
  → 1988 progressive refinement（行式迭代 + 交互；同年 Ward 辐照度缓存走"按需"岔路）
  → 1991 层次 radiosity（O(N log N)）
  → 1997 Keller Instant Radiosity（"解出来的 B 场 ≈ 一组 VPL"——实时重解读）
  → 2000s 游戏烘焙管线工业化（Lightmap/SH probe/PRT 各取一角）
  → 2026 Lightmass/Lumen 并存：烘焙与实时在引擎里长期共处（读者专题四问表）
```

**演进主轴**：N² 矩阵的成本压不下去，于是"一次全解"逐步让位给"迭代/按需/摊销"——但"视角无关可预计算"这条根从未被推翻。

## Relationships

- **[[Kajiya — The Rendering Equation (1986)]]**：两年后的统一表述；Radiosity = 渲染方程在"全漫反射 + 面片常亮度"下的特解（[[Rendering Equation]] 概念页历史图已挂本篇）；
- **[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]**：本文明言局部模型"吃不到入射能量"——Radiosity 是给 CT 喂 $E$ 的第一台机器；二作 Torrance 为两条线共同作者；
- **[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]**：同年代的分岔路——"全场景一次解（贵而全）" vs "视线需要哪算哪（省而局部）"；缓存思想的后代是 NRC/DDGI；
- **[[Keller — Instant Radiosity (1997)]]**：把 Radiosity 的解**重解读**为"一组虚拟点光源"——同一数学的两副面孔，VPL 进实时；
- **[[Lightmass in the Rendering Knowledge Map|Lightmass 与预计算光照专题]]**：读者 2026-10-20 建的知识关系专题——本文补上该专题"漫反射互反射特解"的论文节点；
- **[[Global Illumination]]**：概念页谱系表第 3 行（Radiosity 行）的论文节点落位。

## Personal Knowledge State

- **user_level: Normal（结论层可直读）**：GI 概念页已明确"辐射度矩阵求解细节不需要"——本篇取结论层即可：**① B=E+ρH 的能量账；② F 纯几何 + 三约束；③ 视角无关⇒可预计算；④ 失效价目表（几何/材质/灯光三档）**。四条能复述即到位（≈25 分钟）；
- 与读者 Lightmass 专题的红墙实验对读，是"读完就能在自己项目里复现"的少数经典之一；
- 排队来源：[[Rendering Equation]] / [[Global Illumination]] / [[Lightmass in the Rendering Knowledge Map|Lightmass 专题]] 三处历史段共同点名——**三线交汇的高确定性节点**。

## Learning Value

- **理解"预计算"的边界**：哪些量（视角无关的 B）能提前算、哪些（视角相关项）不能——这条判断标准今天仍是烘焙 vs 实时的分界线；
- **掌握"改 X 要重算什么"的三档失效表**——直接迁移到 SABC 预算里"动态灯光维度"的论证：灯动 = 右端项变 = 预计算失效，所以才要买运行时账；
- **看一次完整的"物理验证"姿势**：参数标定 + 实拍对照——评价任何新渲染论文时问一句"它有没有图 9b"。

## Visualization

- 图 6（细分收敛 + RGB 扫描线）／图 7（反弹递增至全解，**金图**）／图 9b（实体模型照片）已提取入库 `Research/Files/Pictures/`；
- 未提取：图 1-5（几何定义示意图——MD 公式已覆盖）、图 8（线性插值——结论一句话已写）、图 9a/9c（参数表与布置图——文字已转录）。

## Notes

- **为什么现在入库**：读者 2026-10-20 自建 [[Lightmass in the Rendering Knowledge Map|Lightmass 专题]]，其"接库内论文"表点名 Ward/Karis/R-H 2001 但"漫反射互反射特解"的 Goral 1984 只在 [[Rendering Equation]] 历史图里挂名无笔记——本次经典线正好补位，且 GI 概念页谱系表第 3 行是全表最后一个无节点的行；
- **来源与版本**：UCL VR 项目镜像 PDF（扫描版 + OCR 文本层；引用原文时注意 OCR 伪影，如 "radloslty"=radiosity）；DOI 10.1145/800031.808601；
- **实现细节勿过度外推**：本篇实现不含遮挡/仅四边形；"radiosity 方法"这一命名也在本篇首次出现（原文："referred to as the 'radiosity method'"）；
- **后续可补（按需）**：Cohen & Greenberg 1985 hemi-cube、Cohen et al. 1988 progressive refinement（若读者烘焙线加深再排）。
