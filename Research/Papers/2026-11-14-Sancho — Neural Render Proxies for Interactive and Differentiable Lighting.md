---
type: paper
title: "Neural Render Proxies for Interactive and Differentiable Lighting"
authors: [Sergio Sancho, Alexander Rath, Marco Manzi, Pascal Chang, Amit H. Bermano, Derek Nowrouzezahrai, Markus Gross, Marios Papas]
year: 2026
published: "2026-11-12（arXiv v1）；EGSR 2026"
venue: "Computer Graphics Forum 45(4)（Proc. EGSR 2026）；20 页 + 6 页附录"
url: "https://arxiv.org/abs/2611.17185"
code: ""
project_page: ""
category: [lighting, relighting, differentiable-rendering, neural-rendering, production-pipeline]
importance: A
historical_importance: 3
game_relevance: 3
production_readiness: Early Production
user_level: Normal
status: unread
aliases: [NRP, Neural Render Proxy, 神经渲染代理, Sancho 2026, Neural Render Proxies]
tags: [lighting, relighting, differentiable-rendering, neural-rendering, disney, production]
---

# NRP（Sancho et al. 2026）

## TL;DR

**把渲染拆成两半：一半只依赖场景（路径），一半只依赖灯（发光）。前者做一次，后者学成一个 7 MB 的函数。于是"改一盏灯等 30 分钟"变成"改一盏灯等 17 毫秒"，而且可微。**

但这个"拆"不是免费的技巧，它有一张许可证：**虚拟光源只加发光、不产生散射**。有这张许可证，路径集合与灯无关，才能把 `Render` 重排成 `SamplePaths` + `GatherLight`。

**这篇最容易被误读的一点**：加速**不是网络带来的**，是解耦带来的。`GatherLight` 本身只要 13.89 ms（512 spp，885×500），网络那 9.13 ms 并不比它快。**网络真正买到的是三样东西——19.25 GB 变成 7.4 MB、一个不可微的渲染器变成可微的函数、以及时间上的稳定。** 标题里的 "Neural" 是手段，不是原因。

## Problem

动画生产里的打光迭代发生在流程**最末端**——材质、相机、几何都定死了，才轮到灯光。于是每一次微调都要整帧重渲：

| 场景 | 512 spp / 960×402 的路径渲染 | 内存 |
|---|---|---|
| Cave | 28 m 12 s | 28 GB |
| Office | 30 m 46 s | 19 GB |
| Tunnel | 35 m 37 s | 14 GB |

（32 线程 Xeon Gold 6342 @ 2.8 GHz；表 4）

而更麻烦的是流程是**单向街道**：能从场景参数生成图像，但不能把"我想要这里亮一点"反推回 3D 灯光参数——因为生产渲染器（Hyperion、RenderMan）不可微。

两条现成的路都不通：

- **图像空间 / 生成式**（合成、diffusion relighting）：能快速出好看的图，但**不遵守物理输运**——焦散、互反射、材质外观都不对，而且给出的不是"灯光参数"，无法回填进生产管线；
- **可微渲染器**（Mitsuba 3、Dr.Jit、辐射反向传播、路径重放）：数学上对，但每一步都要**完整场景 + 重新光追**。在生产级场景上，这等于把最贵的那一步乘以迭代次数。

## Historical Context

```text
1977  Nicodemus —— 定义 BSSRDF / BRDF 的严格形式（与本节无关，但同一批形式化工作）
        ↓
1993  Schoeneman et al. —— "用画的方式反解灯光"（paint-based inverse lighting）
        ↓
2002  Pellacini / 2005 Lpics —— deep framebuffer relighting：缓存 first-hit 表面响应，
        在 GPU 上重跑简化 light shader。REYES 时代的生产级交互预览
        ↓
2005  PBMF07 —— 把"黑盒渲染器 + 非线性优化"用于几十个灯光参数
        ↓
2005  PRT（Kautz-Sloan-Lehtinen）—— 把输运投影到固定基上；但只能远程环境光
        ↓
2018  Hyperion（Disney）—— 全面转向路径追踪，为的是逃开"烘焙失效"的代价
        ↓
2021  NRC —— 在线训一个小 MLP 预测路径顶点的辐射度，加速收敛；
        但**当前光照被隐式编码进权重**，不能查询新光照
        ↓
2024  ZeroGrads —— 局部神经代理 + 非可微渲染器；但优化时**渲染器仍在环里**
        ↓
2026  ★ NRP —— 一次渲染之后，渲染器与场景都不再需要
```

**这条线的主轴不是"更快的渲染"，是"渲染器还在不在环里"。** Lpics 在环（每次重跑 shader）、ZeroGrads 在环（每次重渲低 spp 更新代理）、NRC 在环（继续追路径）——**NRP 是第一次把它彻底拿出去**。

## Previous Work（三类，卡在同一个位置）

1. **交互式重打光 / deep framebuffer**（Lpics、Lightspeed）：缓存**first-hit** 表面响应，只重跑简化 light shader。三个硬限制——**只有直接光、只有 first-hit、任何相机/几何/shader 改动都让 bake 作废**。本篇是"路径追踪时代"的同一精神：把 first-hit 换成**全路径**。
2. **神经重打光**（LightFormer、RenderFormer、各类 diffusion relighting）：LightFormer 把虚拟点光编码成 embedding 能实时，但**不支持体积输运、没做反解**；RenderFormer 免逐场景训练但**限制在 4096 三角形 + 单一 BRDF**。全部不做 inverse lighting optimization。
3. **可微渲染**（Mitsuba / Dr.Jit 一族）：梯度是对的，但**每一步都要场景在环**。Lipp 2024 的视角无关伴随光路已经把"光照设计"这条支线做到交互级——但仍是"每一步都光追"。

**缺口就是这一格：把渲染器拿出环之后，还能保持物理输运与可微。**

## Core Idea

### 一、解耦的许可证是"虚拟光源"

虚拟光源（virtual light）在生产渲染里是一种约定：光线打到它，**把发光加进图像，然后射线继续走、不改变方向**。它不遮挡、不散射、不参与路径构造。

于是路径集合**与灯的位置无关**。这就允许把 `Render` 的两件事换序：

```python
# 传统：路径与发光交错
def RENDER(px, scene, virtLights):
    I = 0
    for i in range(Nspp):
        T = 1; x, w = SampleCamera(px)
        for j in range(maxDepth):
            x = Intersect(scene, x, w)          # ← 需要场景
            for L in virtLights:
                if Hit(L, prev_x, x): I += T * Le(L)   # ← 需要灯
            w, fs, p = SampleBsdf(x, -w)
            T = T * fs / p
    return I

# 本篇：拆成两遍，各自只需要一半信息
def SAMPLEPATHS(px, scene):        # 只要场景，不要灯
    P = []
    for i in range(Nspp):
        T = 1; x, w = SampleCamera(px); path = []
        for j in range(maxDepth):
            x = Intersect(scene, x, w)
            w, fs, p = SampleBsdf(x, -w); T = T * fs / p
            path.append((T_before, x))
        P.append(path)
    return P                        # → 落盘：19 GB

def GATHERLIGHT(virtLights, P):    # 只要灯，不要场景
    I = 0
    for path in P:
        for (T, x0, x1) in path:
            for L in virtLights:
                if Hit(L, x0, x1): I += T * Le(L)
    return I                        # → 13.89 ms
```

**这张对照是整个方法的经济学**：`SAMPLEPATHS` 吞掉了渲染里最贵的全部（场景加载、求交、shader 求值），做一次就够；`GATHERLIGHT` 只是一堆线段-光源求交，GPU 带宽吃得下。

代价也随之而来：**丢了 NEE 与 path guiding**——它们都需要在造路径时知道灯在哪。所以只能靠 BSDF 采样，小光源和"被遮挡的角落"方差极大。

### 二、用线性性把"灯"从网络输入里提出来

$I_{px} = \sum_v E(v)\cdot \mathrm{Gather}_{type(v)}(v, P_{px})$（式 1）

这是**线性输运的直接推论**：像素值对每盏灯的强度是线性的。于是网络不必学"亮度"，只需学"单位强度的一盏灯在这个像素上贡献多少"，再按 $E(v)$ 加权求和（式 3）。

两个后果：

- **一个网络只服务一种灯型**（sphere 4 参数 / quad 8 参数）。参数越多越难收敛——textured light 与任意形状面光被列为 future work，不是因为不想做，是因为**输入维度直接决定训练难度**；
- **成本随灯数线性涨**（式 3 是对灯求和的）。这不是实现问题，是分解本身的结构。

### 三、网络只需要学"从任意点到任意像素的输运"

输入：灯参数 + 像素坐标 $px$（2D，用 **2D 多分辨率 hashgrid** 编码）+ 辅助特征 $F_{px}$（albedo 3 + depth 1 + normal 3 = 7D）。
输出：该灯对该像素的贡献。8 层 × 256 神经元，**5.7–7.4 MB**。

hashgrid 编码只用在 $px$ 上——作者试过给其它输入也加编码，"little benefit"。**理由很直接：像素坐标是唯一需要"高频不连续"的那个输入**（阴影边缘在图像空间是跳变），而 albedo/normal 已经由材质与几何本身携带了高频。

## Technical Approach

| 环节 | 做法 | 关键数字 |
|---|---|---|
| 路径缓存 | fp16 位置 + **rgb9e5** 吞吐量（32 bit 共享指数，比 fp32 RGB 每段省 8 字节） | 960×402@512spp **或** 1920×804@128spp 都能进消费级显卡 |
| 重建 | 单个融合 Triton kernel 一次扫完所有线段 | 512 spp：13.89 ms / 19.25 GB |
| 训练目标 | 用 `GATHERLIGHT` **在线造样本**，不用预渲图像集 | 100k 迭代 ≈ 1 小时（RTX 5090，680×680@512spp） |
| 监督信号 | 用 OIDN 去噪后的重建值；维护 **300 张图像池**，每 5 次迭代换 2 张 | 去噪器必须"所有像素同一组灯参数"，图像池是为了补回光照多样性 |
| 损失 | Müller 式相对 MSE：$\frac{(\hat I-I)^2}{\mathrm{sg}(\hat I)^2+\epsilon}$，$\epsilon=0.01$，stop-gradient | 让暗部与亮部同等重要 |
| 反解 | Adam + 重参数化（位置/半径走 logit，颜色走 inverse softplus）；像素子采样当 mini-batch | 1% 像素 → 500 次迭代 0.9 s（1 灯） |

## Key Contribution

1. 一套**复用路径追踪器路径数据**的交互式重打光框架，且**对生产渲染器零侵入**（Hyperion 的 path guiding 记录结构可直接复用）；
2. 一个把几十 GB 路径压成几 MB **且可微**的代理，从而打开反解工作流；
3. 三类落地工具：交互重打光、**多视角同时编辑**（多机位同时挂多份 NRP）、**分层合成**（只对前景层训一个 NRP）、手绘涂鸦 / 生成图当作优化目标。

## Why It Works（数字）

### 消融表比标题诚实：位置编码单独上会掉点

| 配置 | SMAPE ↓ | PSNR ↑ | SSIM ↑ | FLIP ↓ |
|---|---|---|---|---|
| None | 0.0533 | 35.29 | 0.9488 | 0.1546 |
| Aux.（albedo+depth+normal） | 0.0290 | **39.96** | 0.9848 | 0.1017 |
| Aux. + Den.（去噪监督） | 0.0296 | 39.93 | 0.9851 | 0.1084 |
| Aux. + **Enc.**（hashgrid） | 0.0566 | **34.78** | 0.8570 | 0.1232 |
| Aux. + Enc. + Den. | **0.0263** | **41.07** | **0.9885** | **0.1014** |

> **第三行是全文最值钱的一格**：单独加位置编码，PSNR 比什么都不加还低 **0.51 dB**。论文的解释是"它让网络对高频噪声过拟合"。加上去噪监督之后，同一项从 **−0.51 变成 +1.11**。
>
> **判据：位置编码不是免费升级，它是一份"高频容量"。这份容量必须有干净的监督去填，否则它一定去填噪声。**

### 路径数据 vs 图像数据集：赢的不是算力，是复用

图 6：1024 张 1 spp 图像训练出的网络，仍比路径数据低 **2.8 dB**。把"每像素 1024 个样本"这个**固定预算**在图像数量与 spp 之间重新分配（32×32、64×16、128×8、256×4、512×2），**一条都追不上**——而路径数据只用 64 spp，等于对方 1/16 的渲染时间。

> **结论不是"我们的数据更好"，是"同一份路径可以被任意多次光照复用"**：网络在训练中见到的光照配置数量，不受渲染预算限制。**这是"复用一份中间产物"相对"多渲几张图"的结构性优势。**

### 质量 vs 样本数：网络是一堵天花板

附录表 5（2000 组光照的中位数，Pred = 网络，Ref = 用同一批路径直接 gather 再降噪）：

| 场景 / spp | Pred PSNR | Ref PSNR | 差 |
|---|---|---|---|
| bedroom / 32 | **35.96** | 35.37 | **+0.59**（网络赢） |
| bedroom / 128 | 38.83 | 38.59 | +0.24 |
| bedroom / 256 | 40.01 | 40.06 | −0.05（交叉点） |
| bedroom / 512 | 40.65 | 41.33 | −0.68 |
| cbox / 32 | 38.62 | **39.35** | **−0.73**（网络输） |
| cbox / 128 | 41.11 | 43.60 | −2.50 |
| cbox / 512 | 42.14 | **46.75** | **−4.61** |

> **两件事同时成立**：低 spp（32–64）时网络赢，因为它把噪声平滑掉了、而逐帧降噪会抖；**128–256 spp 附近交叉**；再往上网络停住，参考值继续涨，差距越拉越大。
>
> **它的训练目标就是 512 spp 的重建，但输出只有 42.14 dB 而目标是 46.75 dB。** 差的这 4.6 dB 不是训练不足，是**容量上限**——7.4 MB 装不下 19 GB 里的全部高频。论文对"closely approximating ground-truth"这句话的诚实边界就在这里。

## Limitations（四处冷水）

**冷水一：固定相机 + 固定材质，这不是实现细节，是全部经济性的地基。**

摘要把条件写得很清楚：*"static scenes with fixed camera and materials"*。相机一动、材质一改，30 分钟的 `SAMPLEPATHS` 重来。**"30–60 Hz"的分母是"已经被摊销掉的一次 30 分钟"**，不是"每帧 17 毫秒"。

论文自己也承认序列只"thinkable, but have not been explored"——**一个镜头一个 NRP，一小时训练一个。一部片子几百个镜头。**

**冷水二：速度账要拆开看，"网络比 gather 快"是假的。**

885×500、256×8 的推理是 **9.13 ms**；同分辨率 512 spp 的直接重建是 13.89 ms **+ 2.60 ms 降噪 = 16.49 ms**，占 19.25 GB。所以网络确实更快——**但它真正的对手是 256 spp 的重建（6.90+2.60 = 9.50 ms，9.63 GB），两者时间几乎相同，内存差 1300×。**

> **网络的收益主要是存储与可微，速度只是副产品。** 把这条说清很重要：否则会误以为"用网络 = 更快"，然后在内存不成问题的场合白白引入一个 42 dB 的上限。

**冷水三：误差集中在"相机看不见、但灯可能去"的地方。**

图 14：把灯放进灯罩的腔体内部，预测的光传播明显错——因为 BSDF 采样从相机出发，**从没进过那个腔体**（无 NEE）。作者原话：*"can lead to under-sampling in occluded regions where a user might later wish to place a light source"*。

> **这是一个结构性的错位**：路径是按"相机看得见什么"采样的，而人要放灯的位置恰恰是"相机看不见什么"。**采样预算与设计意图在这里方向相反。**

**冷水四：灯型与灯数都是硬约束。**

- 只有 sphere（4 参数）与 quad（8 参数）能用；textured light / 任意形状面光留作 future work；
- 成本随灯数线性涨（表 3：500 次迭代，1 → 3 → 5 → 20 灯是 21 s → 63 s → 138 s → 551 s）；
- **light linking / 物体间非物理衰减改不了**——"物体间的传输在渲染之后就固定了"，只能靠"每层训一个 NRP"部分模拟，代价是训练多份。

## Game Development Relevance

**直接部署进游戏：不行。相机永远在动。但它的形状在游戏里早就存在，只是名字不叫 NRP。**

### 第一，"解耦"在游戏里的名字叫 deferred shading

把 `maxDepth` 设成 1，`SAMPLEPATHS` 就退化成光栅化：G-buffer 里存的正是"第一次交点的位置 + 材质 + 通量"（即路径数据），光照 pass 做的正是 `GATHERLIGHT`。**deferred shading 就是"深度 1 的 NRP"，而且是精确的、不需要网络的。**

于是真正的问题变成：**为什么游戏不把路径深度加大？**

答案是：**深度 1 的路径可以用光栅化在 1 毫秒内重建，深度 N 的路径不行。** 相机一帧一动，路径数据就全部作废——而 NRP 的全部价值建立在"路径数据能复用很多次"上。**这不是精度差异，是"你的中间产物能不能跨帧活下来"。**

### 第二，游戏工业真正的同类不是渲染器，是"烘焙"

NRP 的形状 = **一次离线 pass 换一个运行时可查询的函数**。这正是游戏里所有 bake 的形状。区别只在**冻结了哪个轴**：

| 手段 | 冻结什么 | 放开什么 | 失效时机 |
|---|---|---|---|
| Lightmap | 几何 + 灯 + 材质 | 相机 | 灯一动就废 |
| DDGI / 探针场 | 几何 + 材质 | 相机、灯慢慢动 | 几何一动就糊 |
| Pre-integrated Skin / BRDF LUT | 积分本身 | 参数（N·L、roughness） | 出定义域就错 |
| **NRP** | **几何 + 相机 + 材质** | **灯任意动** | **相机一动就废** |

> **四者不是"先进程度"不同，是冻结的轴不同。** 这张表可以直接当选型判据：**先问"我这个场景里哪个量是不动的"，再选冻结它的那个手段。**

### 第三，它在游戏里唯一可能的落点是"相机受限"的场合

- **固定机位的过场（cutscene）**：镜头锁死的那几秒到几十秒，正是 NRP 的工作区间；
- **固定视角的玩法**（塔防、战术视角、固定机位解谜）：同上；
- **编辑器内的灯光预览**：在"锁机位调光"的工作流里，一小时训练换一下午的交互式迭代，账是划算的。

反过来说：**只要相机自由，这套东西的成本结构就崩了。** 这不是"以后算力上来就能用"的问题。

## Unreal Engine Relevance

- **落点**：Sequencer 里的灯光迭代、以及编辑器内的"锁定视口预览"。UE 的 **Lighting Scenario** + **Preview Rendering Level** 是同一诉求的弱版本（只能换预烘焙的光照场景，不能任意改灯）。
- **必须改的**：路径 dump 需要引擎暴露 path vertex + throughput。好消息是 UE 的 Path Tracer 与 path guiding 已有路径记录结构（论文明确指出这类结构"can be repurposed directly"），所以**采集端不是从零做**。
- **不该照搬的**：把 NRP 当成"实时 GI 方案"。它的成本模型里有一次 30 分钟的串行前置，与游戏"资产一变就要重来"的迭代节奏直接冲突。
- **一个可以偷的地方**：**分层**。论文的 Fig 11 是"只对前景层训一个 NRP"——这在 UE 里对应"只对一个 Actor 层做灯光代理"，比整场景训一个小两个数量级。

## Technology Evolution

```text
【线：把渲染器拿出环】
1993  画出来的反解          —— Schoeneman：人可以指定"哪里亮"
2002  deep framebuffer      —— Lpics：缓存 first-hit，重跑 light shader（直接光 + 相机锁定）
2005  PRT                   —— 把输运投影到固定基（只能远程环境光）
2018  全路径追踪            —— Hyperion：为逃开 bake 失效，代价是每次重渲
2021  NRC                   —— 在线小 MLP 加速收敛（光照隐式编码进权重）
2024  ZeroGrads             —— 局部代理 + 黑盒渲染器（渲染器仍在环里）
2026  ★ NRP                 —— 一次渲染后场景与渲染器都不再需要
                              · 19.25 GB → 7.4 MB；13.89 ms → 9.13 ms
                              · 代价：相机与材质冻结
```

```text
【线：可微化的三条路，本篇选第三条】
① 改渲染器      —— Mitsuba 3 / Dr.Jit / 辐射反向传播 / 路径重放：数学精确，每步都要场景
② 拟合局部代理  —— ZeroGrads：便宜，但每一步都要重渲一次低 spp 来更新代理
③ 重建目标函数  ★ —— NRP：一次渲染 → 学一个"灯参数 → 像素"的函数，之后再也不碰渲染器
                     代价：代理是逐场景逐视角的，且误差有天花板（42 dB vs 46.7 dB）
```

## Relationships

### Based On

- [[Rendering Equation]] 的**线性性**——式 1 与式 3 的因子分解（$E(v)$ 提出来）完全依赖"像素值对灯强度线性"。这条性质在库内被反复用到（[[Split-Sum Approximation]] 是同一性质的另一次提因子）；
- **虚拟光源约定**——本篇的解耦不是数学恒等式，是"灯不散射"这个工程约定的推论。**这一点必须记住，否则会以为任何光源都能这么拆**；
- [[Resampled Importance Sampling]] / [[ReSTIR]] 的对照面：那些方法在**采样侧**省钱，本篇在**重建侧**换掉整个估计过程。

### Extends

- **Lpics / Lightspeed 的 deep framebuffer**：把 first-hit 缓存推广到全路径缓存。**这是一次"把老想法放到新硬件/新表示上重做"的典型**——当年的限制（只有直接光、只有 first-hit、任何改动都作废）里，前两条被解决了，第三条原样留着。

### Contrasts

- **NRC（Neural Radiance Caching, 2021）**：最值得对照的一篇。NRC 把**当前光照隐式编码进权重**，所以能加速"这一次渲染"，但不能查询新光照、也不能给灯参数梯度。**NRP 把灯参数当显式输入**，于是能查询任意光照、能反解。
  > **判据：缓存的是"当前光照下的答案"，还是"光照的函数"？** 前者换速度，后者换可微与可编辑性。
- [[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path|HaloRT]]：同一个"删掉中间产物"的动作，两个对象——HaloRT 删掉**估计量**，NRP 删掉**渲染器**。两者都把代价换成一个更难验证的假设（HaloRT 是特征充分性，NRP 是"相机与材质不变"）。

### Related

- [[Neural Global Illumination]]——本篇属于"拟合派"（逐场景训练），但多了一个轴：缓存对象是"输运"而不是"辐射度"；
- [[LightOpt — Lights Optimization for Real-Time Rendering]] 与 [[Learning Path — 灯光系统迁移与参数拟合]]——同为"灯光参数 → 图像"的可微映射，但 LightOpt 在渲染器里做，NRP 在渲染器外面做；
- [[Split-Sum Approximation]]——同样是"提因子 + 查表"，但提的是 BRDF 积分，冻结的是光照的低频性；
- [[Adjoint Methods]]——反解的另一条路：不学代理，直接算转置。**两者的取舍是"一次离线训练"vs"每次都要场景"**。

### Followed By

- **待观察**：把 NRP 扩展到视频序列（每帧一个会退化成一小时一帧）、把"相机"维度也参数化（等价于一个 5D/6D 光场代理）、以及把"欠采样区域"用主动采样（按"人可能在哪放灯"布点）修掉。

## Personal Knowledge State

**Normal。** 入口比看起来浅：

- **可读层（≈25 分钟）**：§3.1 的两段伪代码（图 3）+ 表 2 + 表 5。**带走三句话**：解耦靠虚拟光源、网络买的是存储不是速度、网络有 42 dB 的天花板。
- **需要的前置**：路径追踪的基本结构（[[Rendering Equation]] + [[Participating Media]] 的自由程采样，均为 Easy 域）；"虚拟光源"这个概念（deferred 的直觉够用）。
- **暂不读**：hashgrid 编码细节、Triton kernel、Adam 重参数化——都不是本篇的新东西。

## Learning Value

**高（★ 它给的是"缓存该缓什么"这张判据表）**：

1. 我的管线里，**哪个量在一次会话内是不变的**？（相机 / 几何 / 材质 / 灯）
2. 我缓存的是**那个量固定时的答案**，还是**那个量的函数**？
3. 我的误差**集中在哪个区域**？那个区域是不是恰好是用户最想去的地方？

第 3 条是今天最该带走的一条——**NRP 的误差集中在"相机看不见但灯可能去"的地方**，这不是巧合，是"按可见性采样 + 按设计意图查询"这个结构错位的必然结果。

## Visualization

![[Research/Files/Pictures/Sancho 2026 — 图2 训练与推理管线.png]]

**读图**：左边是**训练**——生产渲染器先出一份"与光无关"的路径数据（`path data`），`Gather Light` 在随机光照下把路径重组成图像，`Denoiser` 清一遍，成为监督目标；中间是 NRP 本体——`(i,j)` 像素坐标进 hashgrid，辅助特征（albedo/depth/normal 三张小图）与灯参数一起进 MLP，累积成输出；右侧上下两条是**推理**的两种用法——`Forward Relighting`（换灯出图）与 `Inverse Optimization`（图上的 LOSS 反向流回灯参数）。**注意管线里没有"场景"这个框出现在推理侧——这是全文论点的图面证据。**（图 2，Sancho et al. 2026）

![[Research/Files/Pictures/Sancho 2026 — 图4 三个生产场景的重打光结果.png]]

**读图**：三行 = 三个 Disney 生产场景，每行三种灯光配置。上行是**毛发**（注意轮廓光在绒毛上的透射），中行是**半透明材质**（布料状帐篷的透光），下行是**体积光**（隧道里的雾与光束）。**这三行是作者选来证明"我们没丢输运"的三类最难的东西——毛、透、雾**，而它们恰好都是"灯不散射"这个前提覆盖不到的边缘（毛发与透射是 BSDF 内部的事，体积靠 free-flight 采样，都不需要灯参与路径构造）。（图 4，Sancho et al. 2026）

![[Research/Files/Pictures/Sancho 2026 — 图6 路径数据训练与图像数据集训练对比.png]]

**读图**：左图纵轴是 PSNR，横轴是"图像数 × spp"。**红色单柱是 Ours（64 spp 的路径数据，37.2 dB）**；蓝色是逐张 1 spp 图像（64 → 1024 张，从 27.9 爬到 34.5）；橙色是"固定 1024 总样本预算"在不同切分下（32×32 → 512×2，从 28.7 爬到 34.2）。**所有图像派的上限都比 Ours 低 2.8 dB**。右图把横轴换成"总渲染成本"，说明**同样的算力，换个用法就差一截**——差距不在渲染了多少，在于那份渲染能不能被反复用来造**新的光照配置**。（图 6，Sancho et al. 2026）

![[Research/Files/Pictures/Sancho 2026 — 图14 灯罩内部的欠采样失效.png]]

**读图**：同一场景同一盏灯，灯被放进灯罩的**腔体内部**。左（a）是 NRP 预测，右（b）是路径重建的真值。**看灯罩下缘：真值是一道窄而硬的光锥，预测是一大片糊开的漏光。** 原因是路径从相机出发、靠 BSDF 采样，**这个腔体从相机完全不可见 ⇒ 里面一条路径都没有 ⇒ 网络只能凭训练分布外推**。**这张图是"按可见性采样 / 按设计意图查询"错位的唯一证据，也是全文最该被记住的失效模式。**（图 14，Sancho et al. 2026）

## Notes

- **一个容易被忽略的设计**：训练用的是"用路径重建出来的值"而不是"预渲的图像"，这让**监督信号与推理输入的分布完全一致**。如果改用预渲图像训练，网络会去拟合"渲染器的噪声模式"，而不是"输运本身"。**这条对任何"从中间产物学一个函数"的做法都成立：监督必须来自同一条中间产物。**
- **附录表 5 里有一格值得单看**：cbox 在 32 spp 时，网络就输给参考值 0.73 dB——**"低 spp 时网络赢"这条在 cbox 上不成立**。cbox 是纯几何 + 面光、几乎无体积、无次表面，也就是"降噪器擅长"的那类内容。**换句话说：网络赢的场合恰好是"参考值本身因降噪而抖动"的场合，不是"网络更准"的场合。**
- **与昨天的接力**：昨天 [[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path|HaloRT]] 删掉的是**估计量**，今天 NRP 删掉的是**渲染器**。两者都靠"把某件事做一次然后复用"——HaloRT 复用一条主射线的遍历，NRP 复用一整次渲染的路径。**区别在于 NRP 复用的东西必须跨会话存活（30 分钟 → 一下午），所以它要求"世界不动"；HaloRT 复用的东西每帧重建，所以它什么也不要求。**
  这条差别解释了为什么一个是论文、一个是产品：**能跨帧存活的东西可以很贵，不能跨帧存活的东西必须很便宜。**
