---
type: concept
user_level: Normal
aliases: [Bidirectional Reflectance Distribution Function, 双向反射分布函数]
prerequisites: [Radiometry, Rendering Equation]
first_introduced: "Nicodemus et al. 1977 形式化定义"
---

# BRDF

## Definition

双向反射分布函数：描述表面在某点把入射方向 $\omega_i$ 的辐照度转化为出射方向 $\omega_o$ 辐射度的能力：

$$f_r(\omega_i, \omega_o) = \frac{dL_o(\omega_o)}{dE_i(\omega_i)}$$

一个 4D 函数（两个方向各 2 个自由度），是"材质"在渲染方程中的数学形态。

## Core Principle

三个物理约束（任何合法的 BRDF 必须满足）：

1. **非负性**：反射不会产生负能量；
2. **Helmholtz 互易性**：交换入射/出射方向，函数值不变（光路可逆）；
3. **能量守恒**：对出射半球积分 ≤ 1（表面不能凭空造光）。

经验模型（Phong）经常违反 2 和 3——这正是 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 要解决的问题。

### 一个不写在三条约束里的前提：$x_o=x_i$（2026-11-14 补）

上面三条是"函数要合法"。还有一条不是约束，是**定义域**——而它比那三条更容易被违反：

$$f_r(\omega_i,\omega_o)=S\big|_{x_o=x_i}$$

**BRDF 是 BSSRDF 在"出点 = 入点"处的切片**（Nicodemus 1977 就是这么定义的）。这不是近似得准不准的问题，是**模型里根本没有"从别处出来"这条路**。

后果只有一句：**表面特征无法相互渗透**。所以用 BRDF 渲皮肤/蜡/大理石，得到的是"硬、假、像 CG"——Jensen 2001 的原话是 *"creates a hard, distinctly computer-generated appearance because it does not locally blend surface features such as color and geometry"*。具体到画面上就是三处：阴影边界不变软、颜色不跨特征渗、薄处逆光不透。

**什么时候这条前提会破？判据是"平均自由程 vs 特征尺寸"，不是材质类别**：

| 关系 | 结果 | 例子 |
|---|---|---|
| 自由程 $\ll$ 特征尺寸 | 光来不及跑出去 ⇒ **BRDF 正确** | 金属、大块塑料、清漆 |
| 自由程 $\approx$ 特征尺寸 | 光从别处出来 ⇒ **必须 BSSRDF** | 皮肤、蜡、大理石、牛奶 |
| 自由程 $\gg$ 特征尺寸 | 光直接穿过 ⇒ **既不是 BRDF 也不是 BSSRDF，是透射** | 叶片、薄纸、纱 |

> **两个常见误判**：
> ① **塑料不该做 SSS**——它 $\sigma'_s$ 高但 $\sigma_a$ 也高，光进去就没了，挂 SSS 只会让高光发闷、颜色发灰；
> ② **玻璃不该做 SSS**——它 $\sigma_a$ 低但 $\sigma'_s$ 也低，光走直线，那是折射与吸收。
>
> **只有"高散射 + 低吸收"才是次表面散射。** 详见 [[Subsurface Scattering]]。

### 第二个不写在三条约束里的前提：$x$ 被平均掉了（2026-11-18 补）

$2026$-11-14 那条补的是**定义域**（出点 = 入点）。今天补第二条，同属定义域，但更隐蔽：

$$\text{BRDF}(\omega_i,\omega_o)\;=\;\frac{1}{A}\int_A \text{BTF}(x,\omega_i,\omega_o)\,dx$$

**BRDF 是一个空间平均的结果，不是材料的本征属性。** 这个式子出自 [[Dana 1999 — Reflectance and Texture of Real-World Surfaces|Dana 1999]] 的 §1，比 BSSRDF 那条更早被写下（1977 Nicodemus 的定义本身也是"某块面积上的平均"）。

**它成立的唯一条件是"平均是安全的"**：一个像素盖住的 texel 里，每个 texel 的外观函数必须**近似相同**——否则平均出来的东西不对应任何真实表面。而这个条件什么时候成立，由 [[Texture Filtering]] 的**脚印尺寸**决定：脚印 ≫ texel ⇒ 成立；脚印 ≈ texel ⇒ 不成立。

**不成立时丢掉的是什么，有精确的清单**（三项，都是"邻域"的函数）：

| 丢掉的效果 | 依赖什么 | 为什么 $x$ 一平均就没了 |
|---|---|---|
| **视差** | 沿视线的高度场 | 眼睛一动，看到的点滑到隔壁 texel |
| **自阴影 / 遮蔽** | 沿光照方向的高度场 | 挡住这个 texel 的凸起不在这个 texel 里 |
| **互反射** | 邻域上的积分 | 反弹进来的光来自四面八方 |

**这三项的价格是可以量出来的。** UBO2014 上，逐 texel 各向异性 GGX（"只有 BRDF"这一端）重建 PSNR **22.10 dB**；实测 BTF 的神经表示 **36.44 dB**。**14 dB 就是"把 $x$ 平均掉"的价格。**（[[2026-11-18-Gokbudak — One Code per Material Neural Representation and Generation of Measured BTFs|Gokbudak 2026]] 表 1）

> **一个能解释很多抱怨的判据**：**"截图看着挺好，一进游戏就假"**——不是美术没做好，是那个表示里**没有角度这个自变量**（或者更准确地说：它假设了你只有一个视角）。同类判据见 [[Bidirectional Texture Function]]。

**两条前提的关系**：$x_o=x_i$ 说的是"光不从别处**出来**"，$x$ 被平均掉说的是"表面上别处的**结构**不影响这里"。**前者管能量的路径，后者管结构的可见性。** 两条都可以被违反，而且违反的后果完全不同——前者让材质"硬"，后者让表面"平"。


## 模型谱系（按"物理成分"递增）

| 世代 | 模型 | 性质 |
|---|---|---|
| 经验 | Phong 1975 / Blinn-Phong 1977 | 形状拟合，无物理 |
| 微面理论 | Torrance-Sparrow 1967（物理光学）→ Blinn 1977 引入 CG | 有机制，Fresnel 被简化 |
| **完整物理** | **[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]** | D·G·F 齐备，能量守恒 |
| **改进分布** | [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]（GGX / Trowbridge-Reitz） | 长尾高光更贴实测；**今日实时渲染的默认 D 项** |
| **廉价的 F** | [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] | 五次幂近似 Fresnel；**今日实时渲染的默认 F 项**（对金属物理上不成立，靠 $F_0$ 参数化绕过） |
| **失效边界** | [[Hair Rendering]]（Kajiya-Kay 1989 / Marschner 2003） | 细长散射体 + 透射，**标准微面 BRDF 不适用** |
| 生产参数化 | Disney Principled 2012 → Karis UE4 2013 | 实时化，美术友好 |
| 进引擎 | [[Karis — Real Shading in Unreal Engine 4 (2013)]] | D 取 GGX 且 $\alpha=$Roughness²、G 用 Schlick 形式配 $k$、F 用球面高斯；**环境光走 [[Split-Sum Approximation]]**（2026-09-18 入库） |
| **同源漫反射** | [[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] | **漫反射与镜面使用同一套微面假设**（GGX+Smith 下的 diffuse 解："最多丢一半"→ multi=0.1159α）；**"specular 分母的 4 = (L+V=2H·V) 的平方"**；UE 的 k=α/2 与 Smith 推导同源；理想 Lambert 缺 5% ★ 2026-10-05 入库 |
| 测量数据 | MERL 100 材质库（2003） | 验证基准 |

## Prerequisites

- 辐射度量学基础：辐照度 E、辐射度 L、立体角（Radiometry）
- [[Rendering Equation]]：BRDF 是其核函数
- 微面几何直觉：半程向量 H 的角色

## Related Concepts

- [[Microfacet Theory]]（★ 微面模型是本谱系「完整物理」之后的全部内容；D·G·F 逐项拆解见此）
- [[Subsurface Scattering]]（★ **本页的"超集"：BRDF 是它在 $x_o=x_i$ 处的切片**；判据是"平均自由程 vs 特征尺寸"——2026-11-14 建页）
- [[Bidirectional Texture Function]]（★ **另一个方向的"超集"：BRDF 是它在空间维上的平均**；判据是"脚印 vs texel 尺寸"——2026-11-18 建页。**两条超集关系正好对应两个被平均掉的自变量：出点位置、表面位置**）
- [[Split-Sum Approximation]]（★ 2026-09-18 入库：BRDF 的半球积分在实时里怎么变成查表——**F 用 Schlick 形式是它能被因式分解的前提**）
- [[Participating Media]]（对偶：微面 = 随机表面，介质 = 随机体积；2026 工作已把两者打通）
- [[Physically Based Rendering]]（BRDF 是 PBR 的数学核心）
- [[Global Illumination]]（BRDF 决定 bounce 的能量分配）
- [[Inverse Rendering]]（BRDF 是要被反解的对象）

## Game Applications

- UE 材质系统的 Roughness / Metallic / Specular 参数最终都汇入 BRDF 求值；
- Clear Coat / Subsurface / Hair 等扩展 Shading Model = 在基础 BRDF 上加 lobe 或换模型。

## Personal Knowledge

Current Level: **Normal**（主动研读中，随 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 线推进）

## Learning Gap

- D/G/F 三因子的物理来源（正在补，见 Cook-Torrance 笔记的检查表）
- GGX vs Beckmann 的形状差异与视觉后果

## Next Step

完成 Cook-Torrance 笔记中的 Mastery Criteria 检查表（5 条），即可考虑升 Easy。
