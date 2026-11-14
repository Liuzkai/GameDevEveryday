---
type: paper
title: "Simulation of Wrinkled Surfaces"
authors: [James F. Blinn]
year: 1978
published: "1978-08（SIGGRAPH '78）"
venue: "SIGGRAPH '78, Computer Graphics 12(3):286–292"
url: "https://people.csail.mit.edu/ericchan/bib/pdf/p286-blinn.pdf"
code: ""
project_page: ""
category: [bump-mapping, microgeometry, shading, rendering-history]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Easy
status: unread
aliases: [Blinn 1978, Simulation of Wrinkled Surfaces, bump mapping 原始论文, 皱褶表面模拟, 法线扰动]
tags: [bump-mapping, microgeometry, shading, classical, rendering-history]
---

# Blinn 1978 — 那两条欠条，四十八年后还没还清

## TL;DR

**「皱褶对亮度的影响，主要来自它改变法线方向，而不是改变表面位置。」**

这一句是 1978 年写下的，也是**整个游戏渲染里用得最多的一项技术**的出生证明。它的全部内容是：**丢掉几何，只保留它对法线的那一项影响。**

这条"丢掉"的账，Blinn 在同一篇论文里**自己结清了两次**，而且都写得很直白：

1. **§4 尺度**："如果物体被放大，皱褶会被拉平。"——扰动不是尺度不变的，他当场给了修法（把扰动按 $|N|$ 归一化）。
2. **§5 过滤**："bump 贴图和最终亮度之间**不是线性关系**。平均 bump 贴图，效果是把凹凸**抹平**，而不是平均亮度。"——他给了当时的正解（超采样），也给了工程上的妥协（"过滤一下能减少最刺眼的走样"）。
3. **§6 轮廓**："必须认识到，这些皱褶**纯粹是错觉**……它们不会改变物体光滑的轮廓边缘。"

**第 2 条是今天最该被重新读到的一句。** 因为 48 年后，[[2026-11-12-Silaghi — Deformable Texture Maps Compiling Dynamic Microgeometry into Neural Textures|DTM 2026]] 在 Limitations 里原样写下：*"scale-dependent filtering remains unresolved"*。**换了表示、换了硬件、换了四代人，这条欠条没动。**

## Problem

1978 年的问题是"曲面太光滑"。

Catmull 的纹理映射（1973/1975）已经能把图案贴到曲面上，但**贴一张真实的皱褶照片不行**：

> *"The images usually looked like smooth surfaces with photographs of wrinkles glued on. The main reason for this is that the light source direction when making the texture photograph was rarely the same as that used when synthesizing the image."*

**"把皱褶照片粘在光滑表面上"**——这句话是整个 1978 年论文的动机，而且它是**光照一致性**问题，不是"分辨率不够"问题。

这个诊断值得单独看一眼：**照片里已经烘焙了当时的光**。把照片贴到别处，等于把一套光照假设搬到一个不成立的地方。

**所以修法不是"拍得更好"，是"别存光，存几何"。**

——这正是 48 年后 DTM 那句 *"we encode geometric attributes rather than radiance"* 的同一条理由。**"存几何不存辐射"这条原则，1978 年就有了，理由一模一样。**

而"建出每一道皱褶"太贵：

> *"Modelling each surface wrinkle as a separate patch would probably be prohibitively expensive."*

## Historical Context

```text
1963-72  Roberts / Watkins / Phong：多边形 + 逐点光照，"太光滑"
  ↓
1974  Catmull：纹理映射 —— 把图案绑到参数域（u,v），跟着物体转
        · 但把皱褶照片贴上去 ⇒ "照片粘在光滑表面上"（光照不匹配）
  ↓
1975  Batson, Edwards & Eliason：从地形高程数据合成山体阴影
        · 法线由高程梯度算 —— Blinn 明确引用为"类似的技术"
        · ★ 这是"法线扰动"的真正源头，但它服务于地图制图，不是渲染
  ↓
1978 ★ Blinn：把"法线扰动"变成渲染原语，并给出可分离的实现
        · 只保留 N。当场写下三条限制：尺度、过滤、轮廓
  ↓
1984  Cook（Shade Trees）：真正的位移 —— 轮廓第一次会变
  ↓
1988  Max（Horizon Mapping）：从高度场算自遮挡 —— 遮挡第一次进贴图
  ↓
2001  Kaneko（Parallax Mapping）：偏移 UV —— 视差不用几何
  ↓
2006  Tatarchuk（POM）：高度场步进 —— 视差 + 自遮挡进像素着色器
  ↓
2026  DTM：α + n + h + c，且整个场以变形 d 为条件
```

**这条线的组织原则不是"越来越准"，是"每加一个通道，多买一类效果，多欠一条混合规则"。** 展开见 [[Microgeometry]]。

## Core Idea

**把"皱褶"从几何里摘出来，只保留它对法线的影响；然后把法线扰动写成一个可查表的函数。**

三步推导（原文 §2，符号照抄）：

1. 曲面 $P(u,v)$，法线 $N=P_u\times P_v$。
2. 皱褶 = 沿法线方向的位移 $F(u,v)$：$P'=P+F\,N/|N|$。
3. 对 $u,v$ 求偏导（链式法则），再**假设 $F$ 足够小**（丢掉落 $F$ 本身、只留 $F_u,F_v$）：

$$P'_u\approx P_u+F_u\frac{N}{|N|},\qquad P'_v\approx P_v+F_v\frac{N}{|N|}$$

$$N'=N+D,\qquad D=\frac{F_u\,(N\times P_v)-F_v\,(N\times P_u)}{|N|}$$

**几何解释**（这是全文最漂亮的一步）：$(N\times P_v)$ 与 $(N\times P_u)$ 都在切平面内，所以 $D$ 也在切平面内——**$N'$ 就是把 $N$ 绕切平面内的某个轴转了一个角度**，转角

$$\tan\theta=|D|/|N|$$

而那个转轴是**垂直于 $F$ 的梯度**的，于是：

> *"the perturbed normal vector will be tipped 'downhill' from the slope due to F."*

**"往山下倒"**——一句话把法线扰动的全部几何讲完了。

## Technical Approach

### 怎么查这个函数：64×64 的表 + 中心差分

$F$ 用**一张 64×64 的表**存（作者的理由很实在：解析式要 "an excessive number of coefficients"，查表更简单）。参数域 $[0,1]^2$ 缩放到 64 格。

**关键实现细节**（今天写 shader 的人都该知道为什么）：

$$F_u=\frac{\text{FVAL}(U+E,V)-\text{FVAL}(U-E,V)}{2E},\qquad E=\tfrac1{64}$$

**用差分而不是插值求导**，理由写在原文里：

> *"Since the function F appears in the calculation only in terms of its derivative we should use a higher order interpolation scheme which is continuous in derivative. Otherwise the lines between function samples may show up as creases in the surface."*

翻译：**双线性插值的值连续、导数不连续**；而这里只用到导数，所以导数不连续会直接变成**表面上的折痕**。他绕开了 B 样条（"generality is not really needed here"），改用**对双线性插值结果做差分**——导数连续、值不连续，而值反正不用。

> **这处细节解释了今天一个常见 bug 的来源**：法线贴图如果用**双线性**放大而不用 mip/各向异性，放大的边界会显出"折痕"。**1978 年就说了原因：你在用它的导数，却给了它一个导数不连续的重建。**

代价：每像素 **4 次查表**（$F_u$ 两次、$F_v$ 两次），每次双线性。

### 尺度问题与他给的修法（§4）——顺带发明了切线空间

原文的观察：把 $X,Y,Z$ 放大 2 倍，$|N|$ 放大 **4** 倍，而 $|D|$ 只放大 **2** 倍 ⇒ 转角 $\tan\theta=|D|/|N|$ **减半** ⇒ **皱褶变平**。

> *"The net effect of this is that if an object is scaled up, the wrinkles flatten out."*

修法是让扰动**按 $|N|$ 归一化**：

$$D'=a\,D\,\frac{|N|}{|D|}\ \Rightarrow\ |D'|=a|N|\ \Rightarrow\ \tan\theta'=a$$

$a$ 怎么定？他拿**单位平方面片**当参照：$X=u,\,Y=v,\,Z=0$ 时 $N=(0,0,1)$、$D=(-F_u,-F_v,0)$，于是

$$a=\sqrt{F_u^2+F_v^2}$$

**这一步的实质**是：把扰动从"参数域的斜率"换成"**相对于一个标准面的角度**"。而这正是**切线空间法线贴图**的定义——法线贴图存的从来不是"法线"，是"**相对于未扰动法线的旋转**"，且这个旋转被归一化到与物体尺度无关。

**Blinn 没有"切线空间"这个词，但 1978 年 §4 就在做这件事。** 今天的 `UnpackNormal` + 单位化 + 混合，全部继承自这一节。

### 表面怎么来：三种 $F$ 的生成方式

- 程序化（网格、浮雕字）；
- **从 Z-buffer 抄**（跑一遍渲染，把深度当高度用——1978 年的"渲染到纹理"）；
- **用手画**（把帧缓冲的一块当 $F$ 表，用绘画程序涂——"黑=低，白=高"）。

第三种最值得记：**因为只用导数，所以任何"强度恒定"的区域都看起来是平的，只有明暗变化处才有凹凸。** 这条性质今天仍成立，也是"法线贴图看起来像浮雕玻璃"的根源。

## Key Contribution

1. **一个不可能性论证**："皱褶的影响主要在法线，不在位置"——把几何问题降维成着色问题。
2. **一个可实现的公式**：$N'=N+D$，且 $D$ 在切平面内 ⇒ 等价于绕切平面内某轴旋转。
3. **一个尺度归一化**：$a=\sqrt{F_u^2+F_v^2}$ ⇒ 事实上定义了切线空间法线。
4. **三条自陈的限制**（尺度、过滤、轮廓）——**它们之后各自长成了一整条研究线。**

## Why It Works

### 因为"位置"和"法线"对图像的贡献不对等

这是本条线的第一性原理，值得背下来：

> 在**着色**里，法线是一阶量（决定 $N\cdot L$），位置是零阶量（只在**遮挡与投影**里出现）。

所以：**只要你不看轮廓、不看视差、不看自遮挡，"位置"就可以整个丢掉。** 反过来，**一旦你要轮廓/视差/遮挡，就必须把位置以某种形式请回来**——这就是 1984（位移）、1988（地平线自遮挡）、2001（视差）、2006（POM）这四步在干的事，也是 2026（DTM 的 $h$ 与 $\alpha$）在干的事。

**这条判据可以直接用来读任何一种"细节表示"**：它保留了位置的哪一部分？轮廓？视差？遮挡？还是全都没有？

### 因为"存几何不存辐射"让光照重新自由

1978 年的动机是"照片里烘焙了光"。存法线，就只存了**几何**，光可以随便换。

**48 年后这句话被重新说了一遍，而且是同一批人的另一篇论文**（DTM）。这不是巧合：**任何"从真实世界采一个东西、再放到虚拟光下"的管线，都会独立地走到这条原则**。

## Limitations

### 他自己写的三条（按严重程度）

**① 过滤（§5）——今天仍然未解，这是本篇最重的一条。**

> *"The bump texture definition function, however, does not have a linear relationship to the intensity of the final image. If the bump texture is averaged the effect will be to smooth out the bumps rather than average the intensities. The correct solution to this problem would be to compute the intensities at some high sub-pixel resolution and average them."*

拆成三句：
- **诊断**：$F\mapsto I$ 是**非线性**的，所以"平均 $F$ 再算 $I$" $\ne$ "算 $I$ 再平均"。
- **正解**：**超采样**（在高分辨率上算亮度再平均）。——这在 1978 年是奢侈，在 2026 年叫 **TAA / SSAA**，仍然是最贵的那个选项。
- **妥协**：*"Simply filtering the bump function can, however, reduce the more offensive artifacts of aliasing."* ——**这一句就是 mipmap 法线贴图的出生证明**：明知不对，但能压住最刺眼的走样。

**图 10 是他自己的证据**（见下）——一颗橙子，过滤前满皮疙瘩，过滤后**光滑得像塑料**。**"平均 bump 贴图 = 抹平皱褶"这句话，有图。**

**48 年后 DTM 的版本**：过滤**异质**贴图（连续量 $\alpha,h$ / 方向量 $n$ / 类别量 $c$）需要**分别处理**，仍未解决。**问题从"一个非线性"变成"三种量三种规则"，难度只增不减。**

**② 尺度（§4）——他解了，而且解法成了行业标准。**

见上：$a=\sqrt{F_u^2+F_v^2}$ 把扰动归一化成角度 ⇒ 切线空间法线。**这是三条限制里唯一被彻底解决的**，代价是"法线贴图的强度"变成了一个需要艺术家调的参数（因为归一化丢掉了绝对高度）。

**③ 轮廓（§6）——没有解，而是被判了"不可解"。**

> *"We must realize, however, that the wrinkles are purely illusory... They do not, for example, alter the smooth silhouette edges of the object."*

他没有试图修，只是**明确声明了边界**，并加了一句很诚实的观察：*"the smoothness of the silhouette edges is not overly bothersome"*（在动画序列里看起来并不太碍事）。

**"明确声明边界"这件事本身是这篇论文最值得学的地方**——三条限制，一条解了、一条给了妥协、一条判了不可解，**每条都说清楚了**。

### 我自己的三点补充

1. **§5 的"正解"在四十年后被换了名字，但没有被解决。** 超采样 → MSAA → TAA，本质仍是"在更高采样率上算再平均"，代价仍随采样数线性涨。**所以"法线贴图的 mip 是错的"这件事，今天依然是错的**——只是错得很便宜，且所有人都在错。
2. **"只用导数"这个设计，把"值的绝对高度"永久性地丢掉了。** 这直接导致：法线贴图**无法表达高度差**（不能做视差、不能做遮挡），也**无法跨贴图比较强度**（两张法线贴图混在一起，"谁更凸"没有定义）。**后来的 $h$ 通道（POM、DTM）本质上都是把"值"请回来。**
3. **代价数字**：*"bump mapping takes about 4 times as long as Phong shading and about 2 times as long as color texture mapping."* 每像素 4 次查表 + 一次叉乘 + 归一化。**1978 年 4×，2026 年基本免费**——**这项技术的成本降了四个数量级，它的两条欠条一分没还。**

## Game Development Relevance

- **这是你每天都在用、但可能没读过原文的那篇。** 读它有实际收益：**法线贴图的三条病（mip 糊、强度靠调、轮廓不漏）全部在原文里被点名了**，而且给出了当时的最优解与妥协。
- **一个立刻能用的判据**：**法线贴图不是"几何"，是"旋转"**。所以——它不能叠（两张法线贴图不能相加，只能按旋转复合）、不能缩放（强度不是高度）、不能 mip（平均旋转没有意义）。**三条禁忌，一条来自 §4，两条来自 §5。**
- **"轮廓不变"在今天的成本**：这就是为什么每个项目最后都要给"该有轮廓的地方"补几何 / 视差 / 曲面细分。**这笔钱不是引擎不好，是 1978 年那笔账的利息。**

## Unreal Engine Relevance

- **`UnpackNormal` + 切线空间**：UE 的法线贴图约定（切线空间、$Z$ 向上、8 位编码）就是 §4 那个 $a=\sqrt{F_u^2+F_v^2}$ 的工程化产物。**"Normal Strength" 这个材质参数，物理意义是"$\tan\theta$ 的缩放"，不是"高度"。**
- **`MipBias` / `r.MaxAnisotropy` 对法线贴图的效果与对颜色贴图不同**：因为 §5 的非线性，法线贴图的 mip 不是"更糊"，是"更平"。**这就是远处地面看起来像塑料的直接原因。**
- **POM / 视差在 UE 里是材质节点拼出来的**（`ParallaxOcclusionMapping` 节点），它补的正是 §6 那条"轮廓与视差"的欠账——**用每像素多次采样换的**。

## Technology Evolution

```text
【一条主线：丢掉几何之后，把它的影响一通道一通道地请回来】
1975  Batson：地形高程 → 法线（制图用）
1978 ★ Blinn：N。买"明暗"；欠"轮廓、视差、遮挡、过滤"
1984  Cook：位移。买"轮廓"；代价——多边
1988  Max：地平线映射。买"自遮挡"；代价——预计算
2001  Kaneko：偏移 UV。买"视差"；代价——小高度假设
2006  Tatarchuk：高度场步进。买"视差 + 自遮挡"；代价——多次采样
2026  DTM：+α +c +h + 变形条件。买"覆盖率、材质身份、变形响应"
           欠——过滤（原样继承 1978）
```

## Relationships

### Based On

- **Batson, Edwards & Eliason 1975**（*Computer Generated Shaded Relief Images*）—— 从高程数据算法线扰动的源头，Blinn 明确引用。**这是本条线真正的第 0 站，而且它不在渲染圈里。**
- **Catmull 1973/1975**（纹理映射）—— 被反驳的对象，也是被继承的前提（参数域绑定）。

### Extends

**无——它是这条线的起点。** 唯一的前驱（Batson 1975）在制图领域，不构成"扩展"关系。

### Followed By

- **Cook 1984（Shade Trees）** —— 位移映射，补上轮廓。
- **Max 1988（Horizon Mapping）** —— 自遮挡。
- **Kaneko 2001 / Tatarchuk 2006** —— 视差与 POM。
- [[2026-11-12-Silaghi — Deformable Texture Maps Compiling Dynamic Microgeometry into Neural Textures|DTM 2026]] —— 把 $\alpha,n,h,c$ 一起编译，并让整个场以变形为条件。**它继承的第一条限制就是本文 §5。**

### Related

- [[Microgeometry]] —— 本页是这条概念线的起点（今天新建）。
- [[Texture Filtering]] —— §5 是那个页面"为什么过滤不平凡"的历史源头；今天补的 DTM 是它的当代形态。
- [[Blinn 1978 — Simulation of Wrinkled Surfaces|Williams 1983]] —— 见下"两条欠条"。

### Contrasts

- **Williams 1983（[[Williams 1983 — Pyramidal Parametrics|Pyramidal Parametrics]]）**：**同一个时代的另一张欠条。** Williams 写"mip 最严重的妥协是它是对称的"；Blinn 写"皱褶纯粹是错觉，而且不能过滤"。**两张欠条，一张 43 年后被 AF-NTC 还了一半（2026-11-11），一张 48 年后仍挂在 DTM 的 Limitations 里（2026-11-12）。**
  **这两篇放在一起，能看出 1978-1983 那批人的共同工作方式：他们不只造工具，还把工具的失效边界写进论文。** 今天读他们，读的其实是**那份边界清单**。

## Personal Knowledge State

- **Easy**：法线贴图是什么、切线空间、强度参数、POM——日常在用，不必复述。
- **今天补的四条（都属"知道但没想过"）**：
  1. **法线贴图存的是"旋转"，不是"法线"，也不是"高度"。** 三条禁忌（不能叠、不能缩放、不能 mip）全部由此推出。
  2. **§5 的非线性是 mip 法线贴图错误的根源**，而且 1978 年就写了正解（超采样）与妥协（过滤）。**今天的 TAA 仍是那个正解，只是更便宜。**
  3. **"只用导数"这个设计永久丢掉了绝对高度**——所以后来必须有 $h$ 通道（POM、DTM）把值请回来。
  4. **$a=\sqrt{F_u^2+F_v^2}$ 就是切线空间法线的定义式。** 所谓"Normal Strength"调的是 $\tan\theta$ 的缩放。

## Learning Value

**最高（作为历史锚点）。** 它是本库 [[Microgeometry]] 一页的起点，也是"每加一个通道买一类效果"这条判据的第一个样本。更实际的收益：**它把法线贴图的三条病一次讲完**，而且给了当时的正解——**这比任何现代教程都干净，因为现代教程通常只讲怎么用，不讲哪里会坏。**

## Visualization

**图 10｜过滤 bump 贴图的前后对照（他自己的证据）。**

![[Research/Files/Pictures/Blinn 1978 — 图10 平均 bump 贴图等于抹平皱褶.png]]

**读图**：左边橙子皮上密布凹凸，右边**同一颗橙子被"过滤"过**——不是变糊，是**变光滑**：疙瘩没了，只剩颜色还在。**这正是 §5 那句话的图像版："平均 bump 贴图的效果是把凹凸抹平，而不是平均亮度。"**
注意这个区别：**颜色贴图 mip 之后是"糊"，法线贴图 mip 之后是"平"。** 糊还能看，平就变成塑料。**这就是为什么"远处的地面看起来像塑料"是一个 1978 年就预言了的 bug。**（出处：Blinn 1978，图 10）

**图 11｜旋转球体上的皱褶错觉。**

![[Research/Files/Pictures/Blinn 1978 — 图11 旋转球体上的皱褶错觉.png]]

**读图**：上排是**那个 $F$ 函数本身**（把高度表当曲面画出来，能看到"辫子"状的花纹起伏）；下四张是球体在旋转中的四帧。
看**球体边缘**——**轮廓是一个完美的圆**，皱褶到边缘就消失了。**这就是 §6 那句"purely illusory"**：它没有改变任何位置，所以轮廓不受影响。
这也是当年他敢说"not overly bothersome"的底气：**在 1978 年的分辨率与动画里，圆轮廓几乎看不出来。** 到了 2026 年，这条边界变成了"必须补几何"的硬成本。（出处：Blinn 1978，图 11）

## Notes

- 7 页，11 图，零表格、零定量评估（除了 §6 那句"4× Phong / 2× 纹理映射"和"每张图 3–7 分钟"）。**说服力全在图上**——和 [[Williams 1983 — Pyramidal Parametrics|Williams 1983]] 一样。
- 出处：ACM DL 需订阅；`people.csail.mit.edu/ericchan/bib/pdf/p286-blinn.pdf` 一次 200（17.7 MB / 7 页扫描版，**带 OCR 文本层**）。**"经典老论文先找教学 bib 镜像"这条固定动作，这是第 10 次生效。**
- **写作上值得学的一处**：他在 §2 开头先否定"直接建几何"，再给出替代方案，并且在**给出方案之前**就把它的适用条件说了（"$F$ 相对表面足够小"）。**先划适用域，再讲方法**——这是 1978 年的论文习惯，现在很少有人这么做。
- 与昨天 [[Williams 1983 — Pyramidal Parametrics|Williams 1983]] 的并置见上（Contrasts）。**两篇相隔 5 年，各自写下一张四十年后才被（部分）偿还的欠条。**
- 待核：§5 的"超采样"与今天 TAA 在法线贴图上的等价性（TAA 用的是历史帧，不是同一帧的更高采样率，**理论上不等价**——这条值得单独查）。
