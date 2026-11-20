---
type: paper
title: "NIRJALA: Draining the Sea out of Gaussian Splatting with Three-Path Optics and Reversible Physics"
authors: ["Latchan Chhetri", "Aman Kumar"]
year: 2026
published: "2026-11-17"
venue: "arXiv 预印本（cs.CV / cs.GR；主分类 cs.CV），21 页 11 图 8 表"
url: "https://arxiv.org/abs/2611.23450"
code: "论文未给（代码/项目页均未出现）"
project_page: ""
category: ["Rendering", "Participating Media", "Underwater", "Gaussian Splatting", "Inverse Rendering"]
importance: A
historical_importance: 3
game_relevance: 4
production_readiness: "Research（PyTorch + 3DGS 实现；RTX 3080 Ti 单卡训练 1 h 45 min / 3,499 MiB；渲染 44 FPS；无引擎集成、无游戏资产评测）"
user_level: Normal
status: unread
---

# NIRJALA: Draining the Sea out of Gaussian Splatting with Three-Path Optics and Reversible Physics

## TL;DR

**去水不是"把水减掉"，是"除以一个会随距离指数变大的数"——所以问题不在颜色，在条件数。**

水下 3DGS 的主流做法是继承 Sea-thru 的两条款模型 $I=J e^{-\beta_D r}+B^\infty(1-e^{-\beta_B r})$，然后把水"除掉"：$J=(I-B)/e^{-\beta_D r}$。**这一步的放大倍数就是 $e^{+\beta_D r}$**——十米处是几倍，五十米处就是几十倍。于是远处一点点 backscatter 估错，会被放大成很大的颜色误差，光度损失再把这个误差推回几何。

本篇加**第三条光路**——前向散射（场景光被水分子改一次方向后仍然到达相机）：

$$I_\lambda = \underbrace{J_\lambda T^D_\lambda(r)}_{\text{直达}} + \underbrace{B^\infty_\lambda\left(1-T^B_\lambda(r)\right)}_{\text{后向散射（幕光）}} + \underbrace{J_\lambda T^S_\lambda(r)}_{\text{前向散射}},\qquad T^S_\lambda(r)=\eta_\lambda\left(1-e^{-\beta^f_\lambda r}\right)$$

前向项**随距离饱和**（$r\to\infty$ 时 $T^S\to\eta$），于是总权重 $q=T^D+T^S$ 有一个与距离无关的下界：

$$\frac{1}{q(r)}\le\frac{1}{\kappa},\qquad \kappa=\eta\min\!\left(1,\frac{\beta^f}{\beta^D}\right)$$

**四个数字**（表 1 / 表 4 / 表 5）：

| 量 | 数字 | 出处 |
|---|---|---|
| SAJALA 30 场景 PSNR | **24.340 → 24.755**（最强基线 3D-UIR，+0.42 dB） | 表 1 |
| 深度 AbsRel / δ<1.25 | 0.109 → **0.091**（−17%）/ 0.782 → **0.812** | 表 1 |
| 真实 SeaThru-NeRF 四场景 | LPIPS 0.194 → **0.175**，PSNR **28.430**，**44 FPS** | 表 1 |
| **去水后的 PSNR（对真值）** | **10–16 dB**（同一批方法的水下渲染是 22–25 dB） | 表 3 |

> 最后一行是全文最该记住的一句：**"水下渲染做得像"和"把水去对了"差 10 dB 以上。** 前者几乎不约束后者——而 90% 的方法只报前者。

## Problem

水下拍到的每个像素是三项的混合（图 1a）：被吸收削弱后的物体光、水体自己散射出来的幕光、以及**被小角度散射后仍然到达相机的场景光**。三件事同时发生，而观测只有一张图。

作者把失败拆成两层（§1）：

1. **物理层**：两条款模型漏掉第三项。后果不是"不好看"，是**远处的纹理被抹平却仍然隐约可见**——这正是前向散射的特征（小角度偏折，能量没走远，只是位置糊了）。
2. **数值层（对 3D 更要命）**：去水 = 除以 $e^{-\beta_D r}$，增益随距离指数增长。**"远处的一个小误差"在优化里不是小误差**——光度损失会把它推回几何，于是深度图开始碎（图 4）。
3. **第三层，作者单独点出**：场景与水**从图像本身不可辨识**（"scene and water are not identifiable from the images alone"）。一个足够灵活的神经水模型可以把场景纹理吸进自己身上，同时完美拟合每一张观测。

## Historical Context

```text
1729/1876  Bouguer / Allard     衰减是指数律（透射率 e^{-βd}）
1924       Koschmieder          airlight：介质自己是光源，随距离增亮
1975       McCartney            大气光学手册（2002 那篇的主要二手来源）
2002       Narasimhan & Nayar   两项写成加法 E = p·D̂ + q·Â；颜色落在"二色平面"上
                               —— 明确丢掉前向散射（"立体角小，可忽略"）
2018/2019  Akkaynak & Treibitz  修订水下模型：衰减系数依赖反射率，且分 β^D / β^B
2023       SeaThru-NeRF         把两项模型放进体渲染
2024-25    WaterSplatting / SeaSplat / 3D-UIR / AquaGS / Plenodium …
                               分歧全在"水放在哪"（独立场 / 从深度预测 / 存在原语上）
2026  ★    本篇                 第三项 + 10 参数物理锚 + 30 场景 path-traced 评测集
```

**这条线真正的分水岭在 2002 年那句被引用了二十四年的话**：Narasimhan & Nayar 在推导衰减时写明——"我们忽略了每个粒子朝观察者方向前向散射的流量。幸运的是，这一项在视觉应用里很小，因为光源与观察者互相张开的立体角很小（Middleton 1949）"。

**二十四年后，这一项回来了，而它回来的理由不是"物理更真"**——作者自己在 §3.2 写得很清楚：式 (4) 是一个建模选择，不是辐射传输的推导；它是**逐像素的**（只缩放该像素自身的辐亮度，不把光从一个像素搬到另一个像素），**因此抓住了回到该像素的能量，但不建模它造成的模糊**。

> **它买的是"反演问题好解得多"（"What it buys is a much better-behaved inverse problem"）。**

## Previous Work（四条谱系，分歧点在哪）

| 族 | 代表 | 水放在哪 | 与"反演条件数"的关系 |
|---|---|---|---|
| 单图复原 | Sea-thru / 偏振 / CNN / Transformer | 不建模 3D | 一次一张图，无跨视角一致 |
| 体渲染 | SeaThru-NeRF / WaterNeRF | 独立的介质场 | 仍两项，增益仍指数 |
| splatting + 介质场 | WaterSplatting / Aquatic-GS / Plenodium | 与高斯并列渲染的场 | 渲染好，**几何最差**（见下） |
| splatting + 预测系数 | SeaSplat / 3D-UIR / DualPhys-GS | 从渲染深度与特征回归 | 两项；性能最强的一族 |
| splatting + 存系数 | WaterClear-GS / SeaFree-GS / UW-GS | 系数存在原语上 / 双颜色 | 仍是"除一个透射率" |

**本篇的差异化只有一句**：它不换"水放在哪"，它换**"水有几条光路"**，并且对"多一条光路如何改变反演"给了证明。

## Core Idea

三件事，一件比一件不显然。

### ① 多一项，换一个有界的 $q$

$$q(r)=e^{-\beta_D r}+\eta\left(1-e^{-\beta^f r}\right)$$

第一项从 1 掉到 0，第二项从 0 涨到 $\eta$。**两者之和有一个与 $r$ 无关的下界**（命题 1）：

$$\kappa=\eta\min\!\left(1,\frac{\beta^f}{\beta^D}\right),\qquad q(r)\ge\kappa>0,\qquad \frac{1}{q(r)}\xrightarrow{r\to\infty}\frac{1}{\eta}$$

证明只用了一条：$f(x)=1-e^{-x}$ 单调、凹、过原点 ⇒ $f(\beta^f r)\ge t\,f(\beta^D r)$，其中 $t=\min(1,\beta^f/\beta^D)$，于是 $q$ 是 1 与 $\eta t$ 的凸组合。

**看图 3 左边**：横轴是 $\rho=\beta^f/\beta^D$，纵轴是最坏增益 $1/\kappa$，四条曲线是四个 $\eta$。**阴影区标出了"物理分支能到达的 $\rho$ 范围"**（$\rho=0.5\hat g\le0.475$）——即：**只有 $\beta^f$ 与 $\beta^D$ 同量级时，这个界才真的收紧**。作者没有回避这一点，反而把它写成第二条实践教训：*"the guarantee is only as strong as $\kappa$：当 $\beta^f$ 相对衰减很小时，界会像 $1/\rho$ 一样松掉。"*

### ② 一个数，两个角色：条件数同时是梯度

这是全文最漂亮的一步，在附录 C.3（命题 2），正文只提了一句。

式 (3) 对 $J$ 是**线性**的，系数就是 $q$。所以

$$\frac{\partial I}{\partial J}=q(r),\qquad \frac{\partial \ell}{\partial J}=q(r)\cdot\frac{\partial \ell}{\partial I}$$

**反演时你要除的那个 $q$，正是前向时你要乘的那个 $q$。** 两条款模型下 $q=e^{-\beta_D r}\to0$：远处高斯拿到的光度梯度指数级变小，于是

- 远处颜色**学得慢**；
- 更容易被 backscatter 项"解释掉"（因为它加性地就能拟合一个偏亮的远处）。

**所以"病态"与"学不动"不是两件事，是同一个数。** 这解释了为什么加了第三项之后**深度**会变好——不是因为它更懂水，是因为远处终于有梯度了。

顺带一个细节：神经渲染分支的 $T^{D,neu}$ 被一个外层 sigmoid 夹在 $[0.5,\ \sigma(1)\approx0.73]$，作者说得很直白——**那是"有界增益"不是"透射率"**，所以图 7 的分量图不能当物理量读。

### ③ 用一个"太小"的模型把另一个模型钉住

PRATIBIMBA（"倒影"）只有一个想法：**如果神经水模型太灵活，它会替场景解释一切；那就配一个表达力不够的物理模型，双向都走一遍。**

十个全局参数 $\Theta_{\text{phys}}=\{\hat c_\lambda,\hat B^\infty_\lambda,\hat\eta_\lambda,\hat g\}$：每通道三个标量 + 一个各向异性样参数，$\hat\beta^f_\lambda=0.5\,\hat c_\lambda\max(\hat g,0)$。损失是四项（式 14）：

$$\Omega_{ir}=\underbrace{\|I'-I_{gt}\|_1}_{\text{重生成}}+\underbrace{10^{-2}\|J'-\mathrm{sg}(J_c)\|_1}_{\text{恢复落回辐射亮度}}+\underbrace{\mathcal L_{cons}}_{\text{双向一致}}+\underbrace{\|I_{phys}-I_{gt}\|_1}_{\text{用 }J_c\text{ 正向合成}}$$

两个方向**互相拉扯**：正向要求"十个标量能从 $J_c$ 解释出观测"，反向要求"用十个标量反解神经图，要落回同一个 $J_c$"。**一个每通道只有标量的模型表达不了场景结构，所以神经分支藏进水里的纹理，会在这两项里变成残差。**

作者的自我限定是本篇最诚实的一句：

> **PRATIBIMBA 不使分解唯一，它只是"缩小优化器能落进去的那组分解"（narrows the set of decompositions the optimiser can settle into）。它的参数是逐场景的锚，不是可迁移的水的测量。**

### ④ 顺带：分箱灰世界严格强于全局（命题 3）

SeaSplat 对整幅恢复图的均值施加灰世界约束。水下**色偏随距离变**，全局平均会让近处偏红与远处偏青互相抵消。作者给了一个两箱反例：$\mu_1=(0.6,0.4,0.5)$、$\mu_2=(0.4,0.6,0.5)$，各自通道均值 0.5、都不灰，但全局均值正好是 $(0.5,0.5,0.5)$。于是改成按距离分四箱（2–98 百分位、至少 256 像素）各自平衡。

## Technical Approach

- **距离**：$r=(\rho+10^{-5})^{-1}$，$\rho$ 是 alpha 合成出来的逆深度（式 1）——**距离是渲染出来的，不是查出来的**，这是所有 GS/NeRF 类水下方法共同的前提。
- **外观嵌入**：16 维，来自编码后的相机位姿（沿用 NeRF-W）。所有光学模块都是 1×1 卷积，额外成本小。
- **前向项网络**：$\eta=\sigma(a_\eta)$、$\beta^f=\mathrm{softplus}(a_f)$，与 $r\oplus E_c$ 卷积得到。
- **物理分支的输入**是**detach 过、归一化过的** $\bar r$（95 百分位归一、clip 到 $[0,2]$）——**它不能反传梯度到几何**，只做锚。
- **正则**（式 15）：$\frac13\sum\hat B^{\infty 2}+\frac13\sum(T^{S,neu})^2+(1-\hat g^2)$。前两项"阻止幕光与前向项解释一切"；第三项把 $|\hat g|$ 往上推——**按命题 1，$\hat g$ 越大 $\kappa$ 越大、界越紧**。消融显示 $\Omega_{reg}$ 比 $\Omega_{ir}$ 更重要（27.94 vs 28.23 dB）。
- **深度**：Depth Anything V2-Large 单目先验，按 COLMAP 稀疏逆深度做逐图 scale/shift 对齐，不可靠处 mask，第 500 步后开启。

## Key Contribution

1. **TRIVENI**：三光路模型 + 有界反演增益的证明（命题 1）+ 误差传播的显式界（推论 1）+ 噪声方差界（推论 2）+ **梯度下界（命题 2）**。
2. **PRATIBIMBA**：可逆物理锚，双向耦合 10 参数模型与神经水模型。
3. **SAMARASA**：按距离分箱的颜色先验（命题 3 证明它严格强于全局）。
4. **SAJALA**：30 个 Blender Cycles path-traced 场景（Coral Reef 7 / Cave 6 / Ruin 8 / Sunken Ship 9），每场景 120 视角，**同时给出无水图、深度、法线**；Ruin V 有蓝/绿两种水 × 四档浓度 + 一个非均匀浊度变体。

## Why It Works

**推论 1 是"为什么几何也变好"的答案**，值得单独看：

| 误差源 | 三项模型 | 两项模型 |
|---|---|---|
| backscatter 误差 $\delta$ | $\lvert\hat J-J\rvert\le\lvert\delta\rvert/\kappa$，**与 $r$ 无关** | $\lvert\delta\rvert e^{\beta_D r}$ |
| 衰减误差 $\hat\beta_D\neq\beta_D$ | $\left\lvert\hat J/J-1\right\rvert\le\dfrac{\lvert\hat\beta_D-\beta_D\rvert}{\max(\beta_D,\hat\beta_D)\,\hat\kappa}$，**在中间距离最大、远处趋于 0** | $e^{(\hat\beta_D-\beta_D)r}-1$，$\hat\beta_D>\beta_D$ 时无界 |

**注意第二行那个"远处趋于 0"**——它不是笔误。远处 $q\to\eta$（常数），$J$ 由 $\eta$ 而不是由指数决定，所以衰减估错的影响在远处反而消失，峰值落在中距离。图 3 右边就是这条曲线：橙色（两项）一路指数上扬，蓝色（三项）先升后降，并且始终在虚线界之内。

**这就是"深度也变好"的机制**：光度损失不再把远处的颜色误差放大后推回几何，图 4 里基线的深度图碎成块，本篇保持连贯。

## Limitations

作者写了一整节 "Threats to Validity"（§6），罕见地完整：

1. **最好的那次运行**：每个场景训练多次，报的是**按本文指标挑出来的最好一次**；基线是官方设置单次运行。作者自己说：*"这有利于我们，NIRJALA 的数字应读作典型性能的乐观估计，小差距应当读作平手。"* 具体点了两处：Cave 无水图 0.09 dB、Panama 0.03 dB。
2. **前向项不是物理**：逐像素、不搬光、不建模模糊；命题 1/推论 1 说的是"反演条件有多好"，**不是"真实的水服从式 (3)"**。
3. **$T^{D,neu}$ 是有界增益不是透射率**（0.5–0.73），图 7 只能读作"模型的分配"。
4. **训练集是合成的**：Cycles 物理上模拟了吸收与散射，但场景里**没有悬浮颗粒、没有焦散、没有随相机移动的人造光**。
5. **参数不跨站点**：引 [16] 的实测——介质网络换一个站点掉约 **9 dB**。作者据此声明 PRATIBIMBA 的十个参数是"逐场景锚"不是"水的标定"。
6. **极浑水在图像形成模型之前就失败**：COLMAP 在清水中注册 99.5% 的帧，在 12 NTU 下**一帧都注册不上**。
7. **非均匀浊度是硬伤**：δ<1.25 从 0.987（稀疏视角）掉到 **0.586**——因为物理分支假设均匀介质。
8. **无浮渣（floater）度量**：作者承认没测，也无法排除同样的效应对真实场景成立。

## Game Development Relevance

**对游戏研发，本篇的价值不在"水下渲染怎么做"，在两件别的事上。**

1. **它是"介质反演"的最干净样本**。游戏里几乎从不反演水（我们只做正向：按距离的逐通道指数衰减 + 一个幕光色）。**正向里漏掉前向项看不出来；反演里它是发散的来源。** 这条分界线值得记住：**同一套公式，正向可以偷懒，反向不能。**
2. **它给"扫描真实场景"这条 2026 年的主流资产路线补了一个失败模式**。3DGS 扫描已经进影视与 3A 的外景管线；一旦是水下（珊瑚、沉船、泳池），就正好落在本篇的问题里——而现成的 GS 工具链不会告警，它只会给你一个"颜色随距离漂移"的模型。
3. **"先渲染再判分"的陷阱是可迁移的**：本篇表 4 显示，水越浓，**水下渲染 PSNR 越高**（25.13→27.04）而**去水 PSNR 越低**（15.18→11.18）。**难度最高的条件，在错误的指标下得分最高。** 这与 [[2026-11-19-Zoomers — ATS Anti-Aliased Triangle Splatting with Exact Per-Primitive Integrals|ATS 2026]] 的"用 PSNR 验收抗锯齿会偏向更多采样"是同一种病。

## Unreal Engine Relevance

不直接。可映射的只有两处：

- **水下后处理**：`Exponential Height Fog` + 后处理材质只能做"两项"（吸收 + 幕光色），**这正是本篇说不够的那两项**——但对**正向**渲染足够了，因为不需要反演。真正缺的是"前向散射造成的对比度衰减"：一个廉价近似是**按距离降低局部对比度 / 加一层按距离模糊**，这正是本篇**明确没做**的部分（逐像素，不搬光）。
- **`SkyAtmosphere` 的对照**：[[Atmospheric Scattering]] 里那套"球壳 + 双粒子"与本篇"水柱 + 单粒子 + 逐通道系数"共用同一个 RTE，**区别只在边界形状与系数谱**。做水下时把天空的 LUT 换一张系数表即可，但**不能换那个"地平线斜程"的几何**（水下没有地平线，只有水面）。

## Technology Evolution

```text
1975  McCartney        介质 = 衰减 + airlight（两项的源头）
2002  Narasimhan       两项写成加法；丢掉前向散射（立体角小）
2018  Akkaynak         β^D ≠ β^B，且依赖反射率
2019  Sea-thru         把这套做成可用的复原方法
2023  SeaThru-NeRF     两项进体渲染
2025  SeaSplat / 3D-UIR 两项进 splatting（最强基线一族）
2026  ★ NIRJALA       三项 + 有界增益证明 + 10 参数锚
```

**一条反向的观察**：这条线上"更准的物理"与"更好的结果"并不同步。2018 年那次修正是**更准**（分离 $\beta^D/\beta^B$）；本篇的第三项作者自己说**不是更准**（pointwise、不建模 blur），却拿到了更好的深度。**当一个领域开始用"条件数"而不是"物理保真度"当选型标准，说明它已经从建模阶段进入求解阶段。**

## Relationships

### Based On

- [[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar 2002]]：两项加法模型 $E=p\hat D+q\hat A$ 的原始出处；本篇的式 (2) 就是它的水下版。
- 3DGS（Kerbl 2023）：渲染主干、alpha 合成出的逆深度当作距离。

### Extends

- **Sea-thru 族（Akkaynak & Treibitz 2018/2019）**：从两项到三项。
- [[Participating Media]]：给"介质反演"补了第六条路之后的一条——**不是更省的算，是让逆问题不发散**（见该页 2026-11-20 补节）。

### Related

- [[Atmospheric Scattering]]：同一套 RTE，边界从球壳换成水柱；且共享那句"前向散射可以忽略"的历史。
- [[Gaussian Splatting]]：距离来自 alpha 合成的逆深度——**所有这类方法的地基**。
- [[Inverse Rendering]]：介质是几何/材质/光照之外的**第四类未知量**，而它比前三者更容易"吃掉"别的未知量。
- [[2026-11-19-Yilmaz — DOOM-GS Dynamic Out-of-core Optimization for Massive Gaussian Splatting|DOOM-GS 2026]]：两篇都在改"3DGS 的某个默认假设"（那边是"块内成员不变"，这边是"水只有两条光路"），且都用**消融里几乎不动画质、却动别的量**的方式证明主项（那边是时间，这边是深度）。

### Contrasts

- **WaterSplatting**：渲染指标更好看，**深度 RMSE 最差（19.33 vs 8.16）**。作者的解释值得抄下来：*"可能是它的水场吸收了本属于场景的结构。"* **这正是"水把场景藏起来"的实证。**
- **PGSR**：完全不管水，但显式正则化几何，反而是深度最强的基线之一（RMSE 8.32、δ 0.782）。**"建模介质"不自动等于"几何更好"**——这句话应该在每个"我们加了物理模型"的摘要旁边写一遍。

## Personal Knowledge State

- **Easy**：水下照片偏青、远处发灰、红光先消失——做材质/场景的人天天见。
- **Normal（今天建立）**：**三项光路 + 那个有界的 $q$**。能说清"为什么多一项不是更准而是更好解"，以及"条件数就是梯度"这一步。
- **Hard**：推论 1 的衰减误差界（$\lvert g(r)\rvert=e^{-ar}-e^{-br}$ 在 $ae^{-ar}=be^{-br}$ 处取极值的推导）；以及 PRATIBIMBA 的 detach + 归一化范围为什么必须这样做（否则锚会把几何拖走）。

## Learning Value

**最值钱的是"命题 1 + 命题 2 是同一个 $q$"这一对。** 它把"条件数"从一个抽象的数值分析词汇，变成一个可以在代码里指认的量：**你除以的那个数，就是你乘的那个数。**

第二个价值是**测量态度**：本篇造了 30 个 path-traced 场景、专门给出无水真值，然后诚实地报告"去水只有 10–16 dB"。**很多论文不会给你这个机会去发现它的方法其实很勉强。**

## Visualization

![[Research/Files/Pictures/Chhetri 2026 — 图1 水的三条光路与反演增益.png]]

**读图**：左（a）三条光路——直达（`J·T^D`）、幕光（`B∞(1−T^B)`）、以及 TRIVENI 新增的前向散射（`J·T^S`）；中（b）**去水时要除的那个 $q$**：两项模型（橙）随距离指数上扬，三项模型（蓝）趋于 $1/\eta$ 的平台；右（c）三个输出（水下视图 / 无水辐亮度 / 深度）与 44 FPS。**（图 1，Chhetri & Kumar 2026）**

![[Research/Files/Pictures/Chhetri 2026 — 图2 NIRJALA 总体管线.png]]

**读图**：上半（a）渲染路径——COLMAP → 高斯 → 光栅化出辐亮度 $J$ 与逆深度 $\rho$ → TRIVENI 三支（direct / backscatter / forward）合成 $I^{neu}$；下半左（b）PRATIBIMBA 只训练时用：用十个标量反解 $J'$、再用 $J'$ 与 $J_c$ 各正向合成一次，两个方向互相拉；下半右（c）SAMARASA 的损失清单（分箱灰世界 4 箱、TV、光度 0.8·ℓ1 + 0.2·(1−SSIM)、边缘、尺度惩罚、深度）。**（图 2，同上）**

![[Research/Files/Pictures/Chhetri 2026 — 图3 反演增益与误差传播的解析曲线.png]]

**读图**：**两幅全解析、不含任何学到的量**。左：最坏增益 $1/\kappa$ 对 $\rho=\beta^f/\beta^D$，四条线是四个 $\eta$，**阴影区是物理分支实际能到达的 $\rho$**（$\le0.475$）——界只有在 $\beta^f$ 与 $\beta^D$ 同量级时才收紧。右：衰减高估 10%（实线）/ 30%（虚线）时的相对辐亮度误差随距离——两项（橙）指数发散，三项（蓝）**先升后降**并压在界内。**（图 3，同上）**

![[Research/Files/Pictures/Chhetri 2026 — 图4 SAJALA 水下视图与深度.png]]

**读图**：SAJALA 四个场景（Coral Reef / Cave / Ruin / Sunken Ship）× 十个方法的水下渲染与深度。看两件事：**细结构（海草、船桅索具）在基线里糊成一片**；**基线的深度图在 Ruin 与沉船上碎成块**，本篇保持连贯——这正是推论 1 那两行数字的可视版本。**（图 4，同上）**

![[Research/Files/Pictures/Chhetri 2026 — 图6 去水后的颜色复原对照.png]]

**读图**：**这张图是全篇最该看的**。每场景一列"水下 | 无水"，四行是 GT / 本篇 / 3D-UIR / SeaThru-NeRF。注意 GT 的无水图本身就是"去掉水之后仍然不好看"的——因为水的对比度已经被真实光学拿走了，**复原不是变漂亮，是把对比度还回去**。3D-UIR 在珊瑚处出现饱和色块。**（图 6，同上）**

![[Research/Files/Pictures/Chhetri 2026 — 图7 三条光路各自分到多少.png]]

**读图**：从左到右——恢复的无水辐亮度、神经直达权重 $T^{D,neu}$、加性幕光 $B^{neu}$、前向权重 $T^{S,neu}$、合成的水下图。**按 Remark 1 读**：$T^D$ 与 $T^S$ 都乘 $J$，图像只约束它们的和，**所以这四张图是"模型的分配"，不是"光的测量"**。**（图 7，同上）**

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Chhetri 2026 - NIRJALA Draining the Sea out of Gaussian Splatting.pdf`（21 页，6.5 MB）。
- 四个梵语命名：NIRJALA（无水）、TRIVENI（三河交汇）、PRATIBIMBA（倒影）、SAMARASA（和谐）、SAJALA（有水）。**命名与结构一一对应**，这在 CV 论文里少见。
- 论文末尾有 LLM 使用声明：用于 LaTeX 排版与"架构图的初稿"，作者强调模型未参与实验、数据、结果分析。**架构图是 LLM 参与画的**——这大概是 2026 年的新常态。
- 训练成本：**105 min / 3,499 MiB**，是同类高斯方法里最长的（WaterSplatting 23 min、3D-UIR 67 min）。作者的评价是"对离线测绘重建可接受，但是真实成本"。
- 一个容易漏的细节：**$T^B$ 与 $T^D$ 用的是不同系数**（$\beta^B$ 与 $\beta^D$），这是 2018 年那次修正留下的；本页的 $q$ 只含 $T^D$ 与 $T^S$，**幕光是加性的、不参与乘性权重**——这也是为什么 backscatter 误差的表现是"加性偏移被放大"，而衰减误差是"乘性比例失真"。
- 与 [[2026-11-19-Zoomers — ATS Anti-Aliased Triangle Splatting with Exact Per-Primitive Integrals|ATS]] 的一个隐性同构：两篇都在**给误差划边界**（ATS 的保守剔除上界、本篇的 $1/\kappa$），并且都用"边界可算"换掉了"经验阈值"。**2026 年这批渲染论文的共同动作，是把启发式换成不等式。**
