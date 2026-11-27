---
type: concept
user_level: Normal
aliases: [Metalness workflow, Metallic-Roughness, MR workflow, 金属度工作流, 金属粗糙度工作流]
prerequisites: [BRDF, Microfacet Theory, Physically Based Rendering]
first_introduced: "2012（Burley, Disney Principled BRDF）；2013 进实时引擎（Karis, UE4）；2015 起成为 glTF / USD 的默认参数化"
---

# Metallic-Roughness Workflow

## Definition

一句话：**用一张 base color 加一张 metalness，同时描述"漫反射反照率"与"镜面反照率"两个量**。渲染器按 metalness 在两者之间插值：

$$D=\mathrm{lerp}(C,\,0;\,M),\qquad S=\mathrm{lerp}(0.04,\,C;\,M)$$

$C$ = base color，$M$ = metalness，$D$ = 漫反射反照率，$S$ = 镜面反照率（垂直入射）。这两行出自 [[Burley 2012 — Physically Based Shading at Disney|Burley 2012]] 的"linear blend between two different models"，[[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi 2026]] 把它们写成式 (1)。

## Core Principle

### 一个通道，两种含义

| $M$ | $C$ 的含义 | $D$ | $S$ |
|---|---|---|---|
| 0 | 介质的**漫反射反照率**（塑料、木头、皮肤、漆） | $C$ | $0.04$（无色） |
| 1 | 金属的**镜面反照率** | 0 | $C$（有色，0.5–0.95） |
| 0.5 | **不是"半金属"**，是"这块像素一半面积是金属" | $0.5C$ | $\mathrm{lerp}(0.04,C;0.5)$ |

所以 **base color 贴图不是自描述的**：同一张中灰图，配 $M=0$ 是"灰塑料"，配 $M=1$ 是"灰金属"。**它的物理含义由另一张贴图决定**——这是整个工作流里最容易被忘掉、也最容易出错的一条。

### metalness 不是材料属性，是插值权重

Burley 的原话：*metallic: the metallic-ness (0 = dielectric, 1 = metallic). **This is a linear blend between two different models.***

这句比"金属度"三个字信息量大得多：它承认 **MR 不是一种 BRDF，是两个 BRDF 之间的线性混合**。$M=1$ 用的是"无漫反射 + 有色镜面"；$M=0$ 用的是"有色漫反射 + 4% 无色镜面"。**中间值不是物理状态，是覆盖率。**

### 三条推论

1. **贴图不能单独验收。**"这张 base color 对不对"没有答案，只能问"配这个 $M$ 对不对"。ReMeDi 把这件事命名为 **intended metalness** $\mu_C$：每张确定反照率的贴图，都是带着一个（常常是隐式的）metalness 决定被生产出来的。
2. **中间值必须有意义。**mask 会被过滤、mip 会被平均、边界会被抗锯齿插值——$M=0.37$ 会真的进渲染，所以每个中间值都得是一份合法材质。这条要求第一次被写进论文是 Burley §5.7（见下）。
3. **一致性只能被强制，不能靠"鼓励"。**参数化让作者"能"对齐，不保证"已经"对齐；自动化方法（逐通道回归的网络）尤其不会。

## 三个约定的出处

### ① 0.04 不是物理常数，是"参数区间的中点"

Burley 把 `specular`（0–1）线性映射到**入射镜面反射率** $[0,0.08]$，对应 IOR $[1.0,1.8]$；区间中点 = IOR 1.5 = 默认值。而 IOR 1.5 的 Fresnel 法向反射率刚好是

$$F_0=\left(\frac{n-1}{n+1}\right)^2=\left(\frac{0.5}{2.5}\right)^2=0.04$$

两端也对得上：$n=1.0\to0$，$n=1.8\to0.0816$。**所以 $[0,0.08]$ 是一条把二次关系的 Fresnel 曲线拉直的近似**——0.04 是这条直线的中点，同时也是 IOR 1.5 的精确值。两个理由在这里重合，这是"4%"能被当成常识的运气。

| IOR $n$ | $F_0=\big(\frac{n-1}{n+1}\big)^2$ | 典型材料 |
|---|---|---|
| 1.00 | 0 | 空气 |
| 1.33 | 0.020 | 水 |
| **1.50** | **0.040** | 玻璃、塑料、多数漆、皮肤（近似） |
| 1.80 | 0.082 | 重火石玻璃、部分宝石 |
| 2.00 | 0.111 | — |

**工程后果**：0.04 对塑料、漆、皮肤、木头是对的；对**宝石、水、金属漆的清漆层**偏低。MR 工作流把介质侧的 $F_0$ 写死成 0.04（只留一个 `specular` / `Specular Level` 旋钮微调），这些材料要么抬高 specular，要么换工作流。

### ② $\alpha=\text{Roughness}^2$ 是感知线性，不是物理

Burley 自陈两条理由：不 remap 时"匹配光滑材料需要很小且不直观的值"；以及**"在粗糙与光滑之间插值，结果永远是粗糙的"**。平方让插值的中间值落在感知中点——这又一次是"中间值"问题。

### ③ 金属的 $F_0$ = base color（有色镜面）

介质 $F_0$ 无色（≈4%），金属 $F_0$ 有色且高达 0.5–0.95。这就是为什么**金属没有漫反射**：入射光要么被反射要么被吸收，没有"进去散射再出来"的那一份。所以 $M=1$ 时 $D$ 必须归零——不是近似，是定义。

## 三种参数化对照

| 工作流 | 存什么 | 什么是隐式的 | 典型失效 |
|---|---|---|---|
| **Specular–Glossiness（SG）** | 漫反射 $D$ + 镜面 $S$（各三通道）+ glossiness | 两者的 intended metalness 互不约束 | **可以同时"很介质"又"很金属"**：亮 $D$ + 有色 $S$ |
| **Metallic–Roughness（MR）** | $C$ + $M$（单通道）+ roughness | $C$ 的 intended metalness（靠作者自觉） | **$C$ 与 $M$ 不匹配**（失效模式 ①） |
| **MR + 可调 specular**（UE 早期 / OpenPBR） | $C$ + $M$ + `Specular` 标量 | 介质 $F_0$ 可偏离 0.04 | 美术把它当"加高光"的滑块，破坏能量一致性 |

MR 胜出的真实理由不是"更准"，是**更省且更难错**：单通道 $M$ 换掉三通道 $S$（省 1/3 贴图带宽），且 $D$ 与 $S$ 的非法组合被参数化本身排除。**这是"用参数化消灭自由度"的一次标准胜利。**

## 失效模式

### ① intended metalness 不匹配（2026 才有正式名字）

一张 $C$ 是"以介质身份"生产的（$\mu_C=0$），却被 $M=1$ 渲染 ⇒ **太暗**；反过来 ⇒ **太亮且平**。人工作者会自觉对齐，**逐通道回归的网络不会**：它只被教"base color 该长什么样"，没被教"这块像素是什么"。

[[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi 2026]] 给出量化：SuperMat 在同一块银灰盔甲上给出"暗 $C$ + 高 $M$"，渲染比参考暗一截；它的分网络重参数化把 base color PSNR 从 18.47 拉到 21.33，重光照 19.60 → 22.34。

### ② 高光烘进 base color

反照率贴图里带着拍摄时的镜面高光 ⇒ 换光照后高光跟着贴图走，材质"自己在发光"。扫描资产入库第一常见问题，也是 ReMeDi 分网络之后顺带改善的一项（金属分支被专门训练去"识别并去掉金属高光"）。

### ③ 中间值的语义

- **mask 边缘被过滤** ⇒ $M=0.37$ 会真的进渲染。Burley §5.7 的原话：*the mask will be filtered and at the blurred edge of the mask the material response must remain plausible*——这是"中间值必须合法"第一次成为设计要求。
- **mip 链平均 $M$** ⇒ 远处金属边界"糊成半金属"，且金属与介质的 $C$ 平均后两头都不对。
- **8 bit + sRGB 误读** ⇒ $M$ 的 0.5 不是线性 0.5。

**判据**：把 $M$ 当**覆盖率**读，才能回答"这个中间值该长什么样"。ReMeDi 的 $C=\mathrm{lerp}(C_d,C_m;M)$ 就是这条判据的直接实现——中间值不是猜的，是两端按覆盖率混合出来的。

### ④ 0.04 被当成通用常数

见上表。宝石、水、清漆层明显高于 0.04。

### ⑤ 通道打包与色彩空间

glTF 2.0 规定：base color 贴图 **sRGB**，metallic-roughness 贴图 **线性**（G = roughness、B = metallic）。把打包图当 sRGB 读（或反过来）会让 $M$ 与 $R$ 整体偏移——跨引擎搬运时"材质突然变亮/变糙"的最常见原因。

### ⑥ Roughness 与 Smoothness 的符号

UE 用 roughness，Unity 用 smoothness（$=1-R$），glTF 用 roughness。**反了不会报错**，只会让整批材质反向。

## Historical Evolution

```text
1981  Cook-Torrance        微面 BRDF 的物理框架（D·G·F），参数是物理量（F0、m、粗糙度的统计量）
        ↓
2012  Burley / Disney     把物理量重写成 1 个颜色 + 10 个归一化标量；metallic = 两模型线性混合
        ↓                 《无敌破坏王》全片生产验证；"principled 而非 physical"是自觉的选择
2013  Karis / UE4         实时化：GGX + Smith + Schlick + split-sum；MR 成为引擎默认
        ↓
2015+ glTF / USD          MR 成为交换格式的默认参数化（单通道 M 省 1/3 贴图带宽）
        ↓
2017  Hammon / Kulla-Conty 能量侧修补：漫反射与多次散射两册账
        ↓
2026  ReMeDi              把"$C$ 必须匹配 $M$"从"鼓励"变成"构造"：C = lerp(C_d, C_m; M)
```

**这条线的刻度不是"更物理"，是"更少自由度、更难出错"。** Burley 自己写得很清楚：*we need our shading model to be art directable and not necessarily physically correct*——工程目标从一开始就是"让所有参数组合都合法"，而不是"让每个参数都可测"。

## Important Papers

- [[Burley 2012 — Physically Based Shading at Disney|Burley 2012]] —— ★ **本页的出生证明**：metallic 的定义（"linear blend between two different models"）、specular 的 $[0,0.08]$ 映射、$\alpha=R^2$、以及"mask 被过滤后中间值必须合法"这条要求
- [[Karis — Real Shading in Unreal Engine 4 (2013)]] —— 进实时引擎的那一步（MR + GGX + split-sum；非金属 $F_0$=0.04）
- [[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] —— 漫反射侧与 0.04 / $k=\alpha/2$ 的独立核算
- [[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi 2026]] —— 把"$C$ 与 $M$ 的一致"写进参数化（2026）

## Related Concepts

- [[Physically Based Rendering]] —— 本页是它定义里"三件套"的第二件（微面 BRDF / 金属度工作流 / IBL）
- [[BRDF]] —— $D$ 与 $S$ 最终要变成什么
- [[Microfacet Theory]] —— $D\cdot G\cdot F$ 的拆解；$F_0$ 就是这里的 $S$
- [[Split-Sum Approximation]] —— $F_0$ 在这个近似里被预积分进 EnvBRDF LUT（所以换 $F_0$ 的代价是查表，不是重算）
- [[Inverse Rendering]] —— MR 参数是反解目标；**"intended metalness"是反解特有的失败模式**（正向渲染没有这个问题，因为 $M$ 是输入）
- [[Bidirectional Texture Function]] —— 同一条"中间值"判据的另一侧：空间维可滤波、角向维不可
- [[Texture Filtering]] —— mip 会平均 $M$，而平均后的 $M$ 仍然要能被渲染器解释
- [[Subset Selection]] —— 都在做"用更少的自由度表示更多的东西"，只是这里的自由度是**参数维度**

## Technologies

- **glTF 2.0 `pbrMetallicRoughness`** —— baseColor（sRGB）+ metallicRoughness（线性，G/R、B/M）
- **USD `UsdPreviewSurface`** —— metallic / roughness / ior 三个独立输入（比 MR 多一个 ior）
- **OpenPBR Surface** —— MR 与 specular 两套并存，base_metalness 仍是主旋钮
- **Substance / UE / Unity 的导入设置** —— 真正决定"反不反"的地方

## Game Applications

| 场景 | 这条知识用来干什么 |
|---|---|
| **扫描资产入库** | 先查"高光有没有烘进 base color"，再查 $M$ 与 $C$ 是否同源 |
| **AI 生成材质** | 逐通道回归必然产生 intended metalness 不匹配 ⇒ 验收要看**重光照**，不能只看贴图 |
| **跨引擎搬运** | 三处符号/空间：roughness↔smoothness、baseColor sRGB、打包图线性 |
| **移动端打包** | $M$ 与 $R$ 共占一张 RGBA（G/B），$M$ 的位深决定金属边界的干净程度 |
| **风格化 / 非写实** | MR 的"物理"是参数化层面的，不是外观层面的——卡通材质照样可以用 MR + 拉高 $F_0$ |
| **材质库规范** | 给 $M$ 定"只许 0/1 还是允许中间值"，直接决定 mip 与抗锯齿的行为 |

## Personal Knowledge

**作者层面 Easy**：金属度怎么画、怎么打包、UE 里 Metallic 滑块往哪边推。

**页面主体 Normal**：① 0.04 的来源（$[0,0.08]$ 的中点 = IOR 1.5 的精确值）；② metalness 是**覆盖率**而不是材料状态，因此中间值有确定语义；③ 三处交叉失效（sRGB / smoothness 符号 / 高光烘进 albedo）。

**Hard**：$F_0$ 在 split-sum LUT 里的重参数化（换 $F_0$ 只换查表索引）；多层材质的层间能量分配（clearcoat 不扣基底能量这个近似）。

## Learning Gap

- **没有亲手做过"intended metalness 错配"的最小实验**：同一张 $C$ + 两个 $M$（0 与 1）渲染对比，30 秒就能看见"太暗/太亮且平"。**这是本页唯一需要动手的一条。**
- 库内**没有 SG 工作流的任何一手材料**（Substance 的 PBR Guide 是二手转述）——"SG 为什么被淘汰"目前只有 ReMeDi 的转述。
- **$F_0$ 与 IOR 的完整映射表**（含复折射率金属）没有；金属的 $F_0$ 目前只当"有色"用，没有具体数值来源。
- **清漆层不扣基底能量**这个近似（Burley §5.4 末尾）没有量化——按他的说法 clearcoat 能量很小，但"多小"没给数。

## Next Step

1. **30 分钟实验**：拿一张扫描来的金属贴图，固定 $C$ 分别用 $M=0$ 与 $M=1$ 渲一张，再固定 $M$ 把 $C$ 的饱和度改 ±20% 渲一张。**四张图并排看**——这就是 ReMeDi 全文的动机，肉眼可见。
2. **查一次自己的材质库**：随机抽 10 个材质，看 `Specular` / `Specular Level` 有几个被改离默认值。**偏离默认的那些就是 0.04 不适用清单**。
3. 读 [[Burley 2012 — Physically Based Shading at Disney|Burley 2012]] 的 §5.1–§5.7（约 6 页）+ 图 16/19，把 10 个旋钮与"中间值合法"这条对上。
