---
type: paper
title: "A Ray Tracing Solution for Diffuse Interreflection"
authors: [Gregory J. Ward, Francis M. Rubinstein, Robert D. Clear]
year: 1988
published: "1988-08（SIGGRAPH '88, pp. 85–92；本文逐页核对 radsite.lbl.gov 官方扫描件 8 页全文）"
venue: "SIGGRAPH 1988（Computer Graphics, Vol. 22, No. 4, August 1988, pp. 85–92）"
url: "https://doi.org/10.1145/54852.378490"
code: ""
project_page: ""
category: [global-illumination, irradiance-caching, monte-carlo, ray-tracing, architectural-lighting, classical]
importance: A（经典）
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（思想层：irradiance caching / final gathering 是离线渲染器 30 余年的标准件；直接长出 Radiance 渲染器，并作为 Neural Radiance Caching 的祖先延续到 2026）"
user_level: "Normal（结论层）"
status: unread
aliases: [Irradiance Caching, IC 1988, Ward 1988, 辐照度缓存, 漫反射反射的射线追踪解, A Ray Tracing Solution for Diffuse Interreflection]
tags: [gi, irradiance-caching, monte-carlo, ray-tracing]
---

# A Ray Tracing Solution for Diffuse Interreflection（Ward 1988）

## TL;DR

**辐照度缓存（Irradiance Caching）的起源论文——缓存族的"第三种记账"：接收侧缓存。** 一句话机制：**间接光照（辐照度）是视图无关且沿表面缓慢变化的量——所以不必逐像素算，把"表面点收到的答案"存进八叉树，在误差容限内插值复用**。两个技术支点：**split sphere 误差模型**（把"答案变化多快"约成距离项 + 法线夹角项，得到可计算的误差估计）+ **以 a 为容差的缓存判据**（估计误差 < a 才可使用缓存值，否则触发新的 Monte Carlo 计算）——**质量定在一个常数上，计算量自动跟着场景走**。

> **一句话定位**：库内 [[Global Illumination]] 谱系表第 6 行（缓存族）**"接收侧"的源头**；与 [[Keller — Instant Radiosity (1997)]]（缓存"光路顶点"）和 [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]（缓存"光源视野像素"）合读 = **缓存族三种记账齐**（光侧随机 / 光侧结构化 / **接收侧答案**）。从它长出的树：Radiance 渲染器 → final gathering（离线渲染器 30 年标准件）→ DDGI（探针版缓存）→ **Neural Radiance Caching（2021，NRC）**——**NRC 要学的"缓存对象"，就是本篇定义的辐照度**。

## Problem（1988 的上下文）

1988 年光线追踪的三角处境（原文开篇）：

- **光线追踪**：漂亮地处理镜面反射 / 折射 / 直接光阴影，但**漫反射间接光只有常数 ambient 项**——"fails to produce detail in shadows and precludes the use of ray tracing where indirect lighting is important"；
- **辐射度（Radiosity）**：能算漫反射互反射，但**面片离散化**（patch）、传输矩阵完全视图无关、分辨率受面片尺寸限制——"the computation is intractable for all but the simplest scenes"；高分辨率细节要细分到不可行的程度；
- **Kajiya 适应性采样**（1986）：把整幅图当"全部光照"求解——对漫反射互反射仍然要几百个样本；
- **Wallace 两趟法**（1987）：辐射度 + 光追混合，但有两套不兼容的机制，且辐射度的"无限面片细分"问题仍在。

**本文的落点**：让光线追踪**自己**算出漫反射互反射——不用面片、不用矩阵、不为它引入第二套机制。切口是三条观察（原文 §3 逐条）：

1. 漫反射间接光的计算需要**很多样本**；
2. 得到的"间接辐照度"值是**视图无关**的（Lambertian 假设）；
3. **辐照度沿表面缓慢变化**——因为直接光和它的阴影已经在第二步（光线追踪求值）算好了。

> 三条合起来推出原文的中心句：**"the number of values would not depend on the number of pixels"**——答案的个数与像素数解耦，计算量只由"场景需要多少信息"决定。

## Core Idea（缓存的三件套）

```text
主方法（primary method）= 用分层 Monte Carlo 算出一个新的辐照度值（贵）
次方法（secondary method）= 在八叉树里找"可复用的邻居"加权平均（便宜）

每个表面点的判定（原文伪代码）：
  if 附近有一个或多个已存值（且误差达标）:  用它们的加权平均
  else:                                   算一个新值，并存进八叉树
```

**为什么合法**——辐照度缓存的合法性建在两道论证上：

1. **视图无关性**：E 是表面上一点的半球积分，与相机无关——存下来给任何视图、任何帧复用（甚至写文件给下一次渲染用，原文明确说了这一点）；
2. **空间缓变性 + 可估计的误差**：用 split sphere 模型给出"移动多远、转多少角度之后答案会变多少"的一阶估计——**误差估计是这套缓存能被信任的全部依据**。

## Technical Approach（机制细节）

### ① 辐照度积分（primary method 算什么）

$$E = \int_0^{2\pi}\!\!\int_0^{\pi/2} L(\theta,\phi)\cos\theta \sin\theta \, d\theta\, d\phi$$

分层 Monte Carlo（原文 Eq 2）：$E \approx \frac{\pi}{2n^2}\sum_{j=1}^{n}\sum_{k=1}^{2n} L(\theta_j,\phi_k)$，其中 $\theta_j=\sin^{-1}\!\big(\sqrt{(j-X_j)/n}\big)$、$\phi_k=\pi(k-Y_k)/n$（$X_j,Y_k$ 均匀随机数）——**2n² 个采样方向**。

### ② split sphere 误差模型（次方法的"信任半径"）

把局部环境约成一个半径 R 的球、一半亮一半暗、表面元素在球心（原文 Figure 3）。一阶 Taylor 展开给出变化上界（Eq 3a→3b）：

$$\varepsilon \;\le\; \frac{4E_0}{\pi R}\,|x-x_0| \;+\; E_0\,|\xi-\xi_0|$$

推广到任意几何（Eq 4）：

$$\varepsilon(\vec P) \;\le\; E_0\Big[\;\frac{4}{\pi}\frac{\|\vec P-\vec P_0\|}{R_0} \;+\; \sqrt{\,2-2\,\vec N(\vec P)\cdot\vec N(\vec P_0)\,}\;\Big]$$

- **位移项 ∝ 1/R**：离得越远的环境照得越匀，答案变化越慢（R₀ = 到可见表面的**调和平均距离**）；
- **朝向项只看法线夹角**：$\sqrt{2-2\vec N\cdot\vec N_0}$ 是单位圆上的弦长——**与球几何无关**；
- 原文对两项的阐释：*"The change in x becomes the distance between two points, and the change in ξ becomes the angle between two surface normals."*

### ③ 加权平均与容差常数 a（缓存怎么用值）

$$\bar E(\vec P) = \frac{\sum_{i\in S} w_i(\vec P)\,E_i}{\sum_{i\in S} w_i(\vec P)},\qquad
w_i(\vec P) = \frac{1}{\dfrac{\|\vec P-\vec P_i\|}{R_i} + \sqrt{1-\vec N(\vec P)\cdot\vec N(\vec P_i)}}$$

$$S = \{\, i : w_i(\vec P) > 1/a \,\},\qquad a = \text{user selected constant}$$

- **权重 = 估计误差的逆**（距离项 / R + 法线夹角项）——离得近、朝向像的值说话更响；
- **a 是唯一的质量旋钮**：估计误差 < a 的邻居才进集合 S；S 为空就触发新的主方法计算。**误差容限直接决定点的密度：变化慢的地方点疏、快的地方点密**（原文：*"flat surfaces in open areas will have only a few values"*）；
- **背面剔除**（Eq 6）：$d_i(\vec P)=(\vec P-\vec P_i)\cdot[\vec N(\vec P)+\vec N(\vec P_i)]/2$，若 < 0 说明该值位于 P 的"前方"（可能在被遮挡的另一侧）——**排除**；
- **递归深度分层**：不同 bounce 数的值存进**各自独立的列表**，防止"反弹后的值"冒充"直接辐照度"被替换。
- 原文对 a 的直觉注脚：对足够小的 a，S 中不会包含"距离超过平均间距"或"法线夹角超过 90°"的值——因为那样的值预期有 100% 误差。

### ④ 八叉树存储（缓存存在哪）

- 全局立方体包住整个场景 → 八叉树；**叶节点边长取"有效域" a·Rᵢ 的 2–4 倍**——保证每个值在自身层级上最多几步就能被搜到，且"有效域小"的值只在近距搜索中被看到；
- 查找递归：节点上的值满足 w>1/a 且 d≥0 则入选；只有当 P 距子立方体边界 < 半个边长时才继续向下搜；
- **复杂度**：最坏 O(N)，均匀分布 O(log N)——**"空间换时间"的第一个 GI 实例**；
- 立方体尺寸可调：更小 → 列表更短但空节点更多；更大 → 少搜树但多查值（原文明确给出这个权衡）。

### ⑤ 多次反弹与递归的自然刹车

- 反弹次数记录 + 用户上限，超限用常数 ambient（可为 0）；
- **"逐级采样递减"**（摘要原句：*"Successive reflections use proportionally fewer samples, which speeds the process and provides a natural limit to recursion"*）——递归树末端的采样最密，但**高层级被缓存值填满后，新光线不再传播**（原文 Figure 7：递归树 + "不再传播的主计算"）；
- **跨渲染复用**：值可写文件，下次渲染直接读——*"By reusing old values, the indirect calculation will not only proceed more quickly, it will be more accurate"*（**缓存越用越准**：预计算的区域内容差永远不会被突破）。

## Key Data（原文数字）

| 项目 | 数据 |
|---|---|
| 验证场景 | 球（反射率 70%）+ 无限平面 + 平行光源——有**解析解**（上半球闭式、下半球数值积分） |
| 验证结论 | 误差分布均匀（**尽管主计算密度差几个数量级**）；**平均误差 ≈ 估计误差的 1/4、最大误差 ≈ 2×**；误差与 a 线性相关 |
| 办公室场景 | direct only / first bounce / seven bounces：**25 / 40 / 70 小时**（VAX 11/780 各算一次） |
| 冰激凌店场景 | 圆锥间接光照明：**~30 小时**（Sun 3/60）；原文估计：逐像素独立光追 **>500 小时**，精确辐射度解 **~100,000 小时** |
| 大光源处理 | 百叶窗建模成 **6 个面光源**（预计算太阳/天空分布）；**立体角 > 1 球面度的光源从直接项挪到间接项**更高效 |

**成本账本读法**：>500h → 30h ≈ **一个数量级以上**的节省来自"答案复用"而非算得更快——**缓存改写了成本函数关于什么线性**（不再是像素数，而是"需要多少独立答案"）。

## Limitations（原文自陈）

- **split sphere 对"亮点"失效**（§5 Discussion）：聚光灯 / 镜面反射的亮斑被部分遮挡或位于地平线时，位置/朝向的微小变化带来照度剧变——"the error related to a will be much larger than the original split sphere model"；对策：**更小的 a + 更高的采样密度**（代价上升）；
- 原文承认：*"There is no known lighting calculation that can track these small 'secondary sources' efficiently."*——**亮点 / 小次级光源追踪在 1988 是公开难题**（这正是后来光子映射、重要性采样等方法的靶子）；
- 大光源 / 漫透射是"可扩展应用"而非核心验证；镜面交互仍归光追原有机制；
- 八叉树缓存值不区分"被更近物体遮挡"的情形（靠 Eq 6 部分兜底）——**精度依赖 a 的调参**。

## Historical Context & Technology Evolution

```text
1984  Goral et al.：辐射度方法（面片间传能的起点）                      [记名]
1985  Cohen & Greenberg：复杂环境的辐射度解                            [记名]
1986  Kajiya：渲染方程 + 路径追踪 ———————————————— [[Kajiya — The Rendering Equation (1986)]]
1986  Immel / Cohen / Greenberg：非漫反射环境的辐射度（矩阵的延伸）        [记名]
1986  Cook：随机采样（Monte Carlo 工具侧）                              [记名]
1987  Wallace / Cohen / Greenberg：两趟法（光追 + 辐射度混合）           [记名]
★ 1988 本文：**接收侧缓存**——把"表面收到的答案"存进八叉树、容差 a 驱动密度；
        分层 MC + split sphere 误差模型 + 加权插值 + 背面剔除 + 跨渲染复用 
        （来源：LBL 照明系统研究组——建筑照明工程而非图形学实验室）
        ↓
1992  Ward & Heckbert：Irradiance Gradients（把"梯度"存进缓存，点更稀疏）  [记名]
1990s Radiance 渲染器（Ward 等）——本篇的直系工程化身                      [记名]
2000s final gathering 进入 Mental Ray / PRMan 等离线渲染器——行业标准件     [记名]
2019  Majercik et al.：DDGI——探针网格版的"接收侧缓存"（实时化）           [记名]
2021  Müller et al.：Neural Radiance Caching（NRC）——**神经版缓存**（缓存对象不变，缓存"介质"从插值公式变成网络）  [记名，库内 [[Neural Global Illumination]] 已记录]
2026  AMD attention GI：经典缓存结构当神经网络输入（RSM 通道）——"缓存族给神经当眼睛"
```

**时代注脚（三条）**：

1. **出身**：Lawrence Berkeley Laboratory **Lighting Systems Research**（建筑照明能耗研究，DOE 资助；参考文献里有 IES 照明手册与 Siegel & Howell 热辐射传热）——**光传输方法从照明工程进入图形学的历史样本**；致谢感谢了 Bill Johnston 与 Paul Heckbert（LBL 的图形学人）；
2. **与 Wallace 两趟法的对照**：后者需要"辐射度 + 光追"两套机制；本篇用**一种机制（光追）+ 一个缓存**达成同样目标——**"机制数量"也是成本**；
3. **"缓存"一词从此进入 GI 词汇表**：关键词列里赫然写着 *Caching, diffuse, illuminance, interreflection…*——**本文是"缓存族"这个家族的词源**。

## Relationships

### Based On

- [[Kajiya — The Rendering Equation (1986)]] —— 求解对象（辐照度 = 渲染方程积分核的入射侧）；
- 辐射度方法（Goral 1984 / Cohen 1985，记名）——**对照对象**：同样追求视图无关的漫反射解，但本文放弃面片离散化；
- Cook 1986 随机采样（记名）——Monte Carlo 工具侧的直接前置。

### Contrasts

- **vs 辐射度矩阵**：离散热传输矩阵（O(n²)、面片伪影） ⟷ 缓存"解本身"（无网格、精度自适应）；
- **vs Wallace 两趟法**：两套机制 ⟷ 一套机制 + 一个缓存。

### 三种记账（缓存族三源并读）

| | [[Keller — Instant Radiosity (1997)]] | [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] | **Ward 1988（本篇）** |
|---|---|---|---|
| 缓存对象 | **光路顶点**（VPL） | **光源视野像素**（pixel light） | **表面收到的答案**（辐照度） |
| 所在侧 | 光侧（随机） | 光侧（结构化） | **接收侧** |
| 复用维度 | 同一帧的多趟渲染 | 屏幕空间插值 | **表面八叉树 + 跨视图/跨帧** |
| 成本单位 | 每盏 VPL 一遍渲染 | 每像素固定 ~400 样本 | **每个"新答案"一次 MC 计算**（数量由 a 决定） |
| 预算语言 | 样本数定预算 | 样本数定预算 | **误差定预算**（a = 质量旋钮） |

### Followed By

- **Radiance 渲染器**（Ward 等，1990s）与 **final gathering**（离线渲染器 30 年标准件）——本篇的直接工程化；
- [[Neural Global Illumination]] 谱系：**NRC（2021）的缓存对象就是本篇的辐照度**——"缓存介质"从解析插值升级为神经网络，**问题定义没变**；
- DDGI（2019，记名）——探针网格版的实时化变体（"接收侧缓存"换了空间结构）。

## Why It Works（本库读法）

1. **"缓存对象的选择"决定一切**：光侧缓存（IR/RSM）缓存"光源的替身"，接收侧缓存缓存"答案本身"——**答案比光更靠近最终图像，所以复用的收益维度不同**（前者省渲染趟数，后者直接把"答案数量"与"像素数量"解耦）；判据：*这份计算的结果，有多少成分只属于这个输出点、有多少属于位置本身？*
2. **误差预算（第一个"质量旋钮"）**：固定样本数（1978/1997/2005）是"花多少力气"；固定误差 a（1988）是"要多少质量"——**计算量成为质量的函数而非相反**。与"固定样本预算"并列，构成光照成本账本的第二条语言；
3. **梯度/导数信息换稀疏性**：split sphere 的误差估计本质是"用一阶导数决定哪里可以不存点"——**先给稀疏点配上"变化率"，再让插值补稠密**（与"用一条原型路径代表全部路径"（毛发）、"一套基代表全部身份"（GALA）跨域同构：**用结构概括自由度**）；
4. **缓存越用越准**：跨渲染的文件复用让"预计算区域内容差永不突破"——**确定性收益随使用次数累积**（与 River"确定性 = 可丢弃重建"对偶：这里是"确定性 = 可累积复用"）。

## Game Development Relevance

- **对"预算语言"的直接补充**：你的五维预算全部是"样本数 / 数量"语言（发射器数、粒子数、灯数……）；本篇是**误差语言**的祖宗——"质量定死、开销浮动"的档位设计（与 ControlGS"功耗预算条件化"、DLSS"目标帧时间"同一思想族，跨 38 年）；
- **实时 GI 的谱系节点**：Lumen / DDGI 的辐照度探针、UE 的 ILM（Irradiance Light Map）/ VLM 概念，都与"接收侧缓存"同源——**读懂 a 容差 → 八叉树 → 插值这条链，是理解"探针密度为什么要自适应"的钥匙**；
- **开放世界接口**："答案与像素解耦、与视角解耦"的思想在今日的 World Partition / 流式光照系统里以"光照块（lighting scenario）"形式复活——**"光照的存储密度跟着需要走"**。

## Unreal Engine Relevance

- 无直接 UE 实现（概念史）；接口在词汇层：
  - **Lumen Final Gather / Radiance Cache**——名字里就带着本篇的血统（"final gathering"正是 irradiance caching 的工程名）；
  - **ILC（Indirect Lighting Cache）**（UE4 早期的 movable GI 方案）——把辐照度存进点云/探针网格、插值复用：**就是"接收侧缓存"的引擎版**；
  - 读懂本篇再看 UE 的"光照缓存精度 / 采样密度"档位，会看到同一个容差常数 a 的影子。

## Personal Knowledge State

- `user_level: Normal（结论层）`——读法：**"缓存三件套 + 一句台词"**（答案视图无关且缓变 → 存起来插值复用；误差 a 决定点密度；"答案的数量与像素数解耦"）；
- 前置链条：[[Global Illumination]]（谱系表第 6 行）→ 本篇 → [[Keller — Instant Radiosity (1997)]] / [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]（光侧两种记账）；
- **桥的连接点**：[[Neural Global Illumination]]（Hard）——**NRC 缓存的就是本篇定义的量**，缓存族"接收侧"材料至此齐备；Hard → Normal 桥的三块经典材料（光侧随机 / 光侧结构化 / 接收侧）**已全部可读**。

## Learning Value

**四条可迁移抽象**：

1. **"缓存答案本身"**——当一份计算的**结果**比产生它的**过程**更持久（视图无关 / 帧间稳定）时，缓存结果而非过程（对照：PRB 缓存"随机数种子"、River 缓存"河网"、GALA 缓存"基"——**缓存层级的四个样本同框**）；
2. **"给缓存配一个误差模型，而不是配一个阈值直觉"**——split sphere 的保守上界让"什么时候可以偷懒"变成可计算的问题；**误差模型的存在使近似可被信任**（与 Mipmap 的屏幕空间误差、Nanite 的 LOD 误差同族）；
3. **"密度跟着变化率走"**——平坦处少存、陡峭处多存：**自适应采样的祖命题**（Kajiya 1986 的"适应性"在本篇落成数据结构）；
4. **"复用的边界条件要显式管理"**——背面剔除、递归深度分层、a 的调参——**每一个"复用"都需要一整套"排除非法复用"的规矩**（工程化近似的模板）。

## Visualization

![[接收侧缓存_Ward 1988 辐照度缓存图解.html]]

## Notes

- **来源核对**：radsite.lbl.gov（Ward 官方站点）**8 页原刊扫描件逐页核读**（page1–8.gif，2026-10-04；路径来自 Utah 大学 CS6965 课程页索引，"大学课程镜像"路径的又一实例）；页码 85–92 与正文页脚逐页一致；DOI 经 Crossref/OpenAlex 双查（本库采用 SIGGRAPH '88 正刊 DOI: 10.1145/54852.378490；OpenAlex 另返回 10.1145/378456.378490 一条同年注册记录，未采用）。
- **公式全部核对原文**：Eq 2（分层 MC）、Eq 3b / 4（split sphere 误差界）、Eq 5（加权平均 + w_i + S 判据 + a 定义）、Eq 6（背面剔除）——`d_i<0` 即"值在测试点前方"被排除。
- **"instant" 的 1988 语境**：本文无实时主张（30 小时/幅），"即时"来自 1997 Keller（几秒）——**两篇相隔 9 年，成本从"小时级"走到"秒级"，2026 走到"帧内"**。
- **历史细节**：本文关键词列表是 "Caching, diffuse, illuminance, interreflection, luminance, Monte Carlo technique, radiosity, ray tracing, rendering, specular"——**"Caching" 一词由此进入 GI 词汇表**；论文来自建筑照明研究，工程化身是 Radiance——**"从照明工程到图形学、再回到建筑/影视/游戏"的完整往返**。
- 入库日：2026-10-04（Run #26）；[[Global Illumination]] 谱系表第 6 行"接收侧"自此从"按需"改为 ✅（缓存族三源齐：1988 / 1997 / 2005）。
