---
type: paper
title: "L3G-World: Real-Time Infinite World Generation with Latent 3D Gaussian Memory"
authors:
  - Jinhyeok Choi
  - JoungBin Lee
  - Jongmin Lee
  - Minkyeong Jeon
  - Jiyoung Kim
  - Kaeun Han
  - Yoonjin Yoon
  - Seungryong Kim
year: 2026
published: 2026-10-29
venue: arXiv preprint（cs.CV），24 页，KAIST AI × Ewha Womans University
url: https://arxiv.org/abs/2610.40614
code: ""
project_page: https://cvlab-kaist.github.io/L3G-World/
category:
  - rendering
  - world-models
  - gaussian-splatting
  - streaming
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research（单卡 H100 上 480×832 / 9 FPS；静态场景，未处理动态物体）
user_level: Normal
status: unread
---

# L3G-World —— 记忆按"走了多远"长，不按"播了多久"长

> KAIST AI + Ewha Womans University + KAIST Urban AI Institute，`2610.40614`，24 页，2026-10-29。
> **一句话**：世界模型的"记忆"一直在两堵墙之间摇摆——**按时间存**会 OOM（$O(TR)$），**按空间存**又慢到不能实时；这篇同时改了两件事：**只在"没覆盖到的地方"长新高斯**（于是记忆量 $O(S)$，跟场景范围走），**每个高斯里直接存 VAE latent**（于是记忆能渲染进 DiT 的 latent 空间，不用 RGB→VAE 重编码）。

## TL;DR

- **两笔账各自归零，用的是两个不同手法**：**容量**靠"选择性生成"（只在 mask 区域 spawn，已有区域只做残差精修），**延迟**靠"存成下游要的格式"（高斯里带 VAE latent，直接渲染到 latent 空间）；
- **代价最直白处**：把 VAE latent 拿掉（改回 RGB→VAE 重编码），**VAE 空间渲染从 943.6 FPS 掉到 2.97 FPS（318×）**——这是全文最干净的"接口税"；
- **收益有 3 个数量级差**：相比最省的 3D 记忆基线，图元少 **26.7–38.9×**、重建快 **3.0–3.2×**、VAE 空间渲染快 **349–359×**；端到端 **9.02 FPS**（480×832，单卡 H100）；
- **但"实时"是相对的**：9 FPS 是相对同类（0.10–1.99 FPS）的实时，不是相对游戏（60 FPS）的实时；而且**只做静态场景**，移动物体不在表示里；
- **一处诚实的裂缝**：三个质量指标里 **PSNR / LPIPS 赢、SSIM 输**（16.00 / 0.366 vs NeoVerse 13.53 / 0.652，SSIM 0.441 vs 0.525），消融里"去掉 VAE lifting"那一版反而 PSNR 更高（15.41 vs 15.11）。**这不是噪声，是"不同指标在量不同的东西"——与 10-30 的 Texture Consolidation 是同一类现象。**

## Problem

长时程世界模型要在"记得住"和"跑得动"之间选，而两条主流路线各自把成本绑在了一个不该绑的变量上。

| 路线 | 记忆是什么 | 成本跟着谁涨 | 什么时候死 |
|---|---|---|---|
| **时序上下文**（Yume 1.5 / Matrix-Game 3.0 / WorldKV） | 历史帧 / KV cache | **生成流长度 $T$** × 分辨率 | 400 帧时 OOM（WorldKV 直接崩） |
| **显式 3D 记忆 · 前馈式**（Spatia / NeoVerse / 3DScenePrompt） | 逐像素点云 / 逐帧高斯 | **$T \times R$** | 同一块空间被反复重建，图元冗余堆积 |
| **显式 3D 记忆 · 优化式**（ArtiFixer / Lyra 2.0） | 场景级高斯（迭代优化） | **$O(S)$** ✅ | 优化太慢（0.10 FPS） |

**这张表就是全篇的动机**：能实时的都 $O(TR)$，$O(S)$ 的都慢。**"记忆跟着场景走"与"记忆能被实时读"看起来互斥**——除非你换掉两件事：长什么、以及存成什么格式。

## Historical Context

这条线在库里已经走了四站，L3G 是第五站，而且它把前四站的两条轴合成了一条：

```text
2026-10-26  UniWorld      窗口状态：深度只在接缝读一次 → 显存与时长无关（19.05 MiB）
2026-10-26  WorldPager    分层驻留：容量墙 vs 位置墙，需求流量 = 新生体积 / 编译时间
2026-10-20  WorldStream   两层流式：几何可丢弃，记录才是资产
2026-10-29  BoxWorld      薄接口：状态全符号、画面全生成
2026-10-29  ★ L3G-World   记忆按范围长 + 存成 latent —— 把"容量"与"延迟"同时降一个量级
```

[[World Models for Games]] 页里那张"接口厚度"表这次要加一行，但 L3G 加的不是厚度维度，而是**"记忆的单位"**：从"帧/KV"换成"高斯"，再换成"带 latent 的高斯"。**单位一换，成本的斜率就换了。**

## Previous Work

- **前馈 3D 重建**（pixelSplat / MVSplat / Depth Anything 3 / ZipSplat）：一次前向出高斯，快，但**逐像素密预测** ⇒ 图元爆炸；
- **token 化 3DGS**（C3G / ZipSplat 一系）：从 token 解码出紧凑高斯集——L3G-Mem 的骨架借自这里（K-means 初始化 spawn query）；
- **世界模型的 3D 记忆**（GEN3C / Spatia / SPMem / Lyra 2.0 / NeoVerse）：把生成帧重建进 3D 再渲回目标视角做条件——**这条路线的隐含成本是"每次条件都要重编码一遍"**，而没人把它算进 FPS；
- **少步蒸馏**（AnyFlow / flow-map）：把教师蒸馏成 few-step student，做实时。

## Core Idea

两个决定，各解决一堵墙：

> **决定一（容量）：记忆不是缓冲区，是"被不断修订的场景表示"。**
> 只在 **未覆盖 / 高误差** 的区域 spawn 新高斯；已经有的高斯用**残差**更新。
>
> **决定二（延迟）：存的东西要按"下游怎么消费"来存。**
> 每个高斯额外带一个 **VAE latent feature**，于是 memory 能**直接渲染进 DiT 的 latent 空间**——省掉"渲 RGB → VAE 编码"这一整段。

第二条是今天的主线：**记忆的格式不是由记忆自己决定的，是由消费者决定的。**

## Technical Approach

### 1. 记忆的更新：先渲一遍，再决定在哪长

给定上一轮记忆 $\mathcal M_{k-1}$ 和当前窗口 $\mathcal W_k$，把 $\mathcal M_{k-1}$ 从当前相机 $C_k$ 渲出来，得到 **RGB + opacity**：

- **opacity 低** ⇒ 这块没被记忆覆盖；
- **RGB 与 $I_k$ 的误差大** ⇒ 覆盖了但记得不对。

两者合起来得到 **spawn mask $S_k$**。然后只把 $S_k$ 内的几何特征拿去做 K-means，初始化 **spawn query token**——**注意这一步的顺序**：不是"先聚类再筛"，而是"先筛再聚类"。消融里把这个顺序去掉（对全部几何特征聚类），图元从 41K 涨到 75K（**1.8×**）。

同时，被当前视角看到的已有高斯，其 memory token 作为 **refinement token** 取出，与 spawn token **拼在一起**过一个 transformer（两路各自 cross-attend 几何特征 → 各自 self-attn → 再一层 **global self-attn** 让两路交换信息）。**global self-attn 这一层是必要的**：去掉它，spawn 与 refine 各干各的，PSNR 16.52 → 16.36。

### 2. latent 怎么进去：通道拼接，不是新 token

专门一条 **latent-lifting 分支**：用与 spawn query **相同的空间索引**从 $z_k$ 初始化 latent query，cross-attend 到 $z_k$，得到与 query **空间对齐**的 VAE latent token，再**按通道拼接**：

$$T^{\text{sp}}_k=\mathrm{Concat}_c(\bar Q_k;\ \bar Q^z_k)$$

**"按通道拼"而不是"按 token 拼"是关键**：一个高斯 = 一份几何参数 + 一份外观 latent，它们必须**同生同死**。按 token 拼就变成了两份可以各自漂移的东西。

渲染时直接把 latent 渲到目标相机，得到 $z^{\text{mem}}_k$ 与 alpha $o^{\text{mem}}_k$，**沿通道**与噪声 latent 拼接，喂给 DiT。

### 3. 生成端：参考帧 + 记忆条件，只有目标帧去噪

输入序列 = `[参考 latent；Concat(目标 latent, 记忆 latent, alpha)]`，**只有目标帧被去噪**，参考帧固定。时间位置编码给参考帧负号（$[-L_{ref},...,-1; 0,...,L-1]$），空间用 RayRoPE。

蒸馏走 **on-policy**（AnyFlow）：student 自己 rollout、加噪、与冻结教师做分布匹配——**用自己生成的轨迹训练，而不是只用加噪的真值**，这是 few-step 自回归的关键。

### 4. 训练

从**空记忆** $\mathcal M_0=\varnothing$ 开始，连续 $K$ 个窗口，每步对 $\mathcal M_{k-1}$ 截 stop-gradient 保证因果。损失五项：RGB $\ell_1$ / LPIPS / depth $\ell_1$ / **VAE latent MSE** / visibility（每个高斯至少落在一个输入视锥里）。

**"w/o RGB Loss"这一行必须看**：去掉 RGB 与 LPIPS 监督后，VAE MSE 0.251（vs 0.177）、cos 0.696（vs 0.782）、**PSNR 10.26（vs 16.52）**、图元 74K（vs 41K）。**这说明 latent 监督不能独自撑起几何**——latent 是压缩过的，它不告诉你"哪块该长、哪块该修"。

## Key Contribution

1. **一个 $O(S)$ 且能实时读写的 3D 记忆**：spawn/refine 双流 + mask 引导的预算分配；
2. **latent 直接可渲的记忆**：把 VAE 特征"抬"进高斯，消掉 RGB→VAE 的重编码（**318×**，见消融）；
3. **端到端实时**：9.02 FPS @480×832 / 单卡 H100，同类里唯一过 1 FPS 的；
4. **一张把"记忆缩放"写成 $O(TR)$ vs $O(S)$ 的对照表**（Table 1）——这张表本身就是贡献，因为它把"为什么慢"归因到了一个可写的式子上。

## Why It Works

**因为"记忆"这个词被拆成了两个独立的量：**

- **它覆盖多大空间**（$S$）——这是必须付的；
- **它被读过多少次**（$T$）——这是不该付的。

前馈式记忆付的是后者：每来一帧就重建一遍，同一面墙被重建 200 次。优化式记忆只付前者，但每次读都要迭代。**L3G 的做法相当于：前馈地"增量写"，但不"增量堆"**——写的时候只写缺口，读的时候直接读 latent。

而 latent 之所以能省 318×，是因为**重编码本来就是把"已经算过的东西"再算一遍**：VAE 编码器的作用是把像素压成 latent，而 latent 早就在生成端存在了，只是之前被丢掉了。

## Limitations

作者自述三条 + 我补两条：

1. **只做静态场景**：独立运动的物体不在表示里（没有动态-静态解耦）；
2. **极端大范围探索仍会涨**：$O(S)$ 不是常数，需要剪枝或分层；
3. **误差仍会沿自回归传播**：refine 能纠，但纠不完；
4. **（补）9 FPS 的口径**：单卡 H100、480×832、chunk 级自回归——**这是"研究意义上的实时"**，离"游戏意义上的 60 FPS / 1080p"还差一个数量级的算力；
5. **（补）latent 里没有分解**：存的是"长什么样的 latent"，不是 albedo/normal/光照——所以**不能改光、不能改材质**。接口省掉了翻译，也就省掉了可编辑性。

## Game Development Relevance

- **它是"资产流送"的一面镜子**：L3G 的 $O(S)$ vs $O(TR)$ 与 [[Streaming]] 页里的"驻留 / 流量"是同一组量——**驻留跟场景范围走，流量跟变化速度走**。区别在于 L3G 把"压缩"做在了**表示层**（高斯 + latent），而不是编码层（更小的贴图）；
- **"存成下游要的格式"这条可以直接搬到项目里**：现在特效资产进引擎要经过"源格式 → 导入 → 编译"，很多中间产物（比如烘焙好的曲线、预乘过的贴图）在运行时被重新算一遍。**问一句：这份数据下游第一件事是对它做什么？如果第一件事是解码，那解码后的形式才是该存的形式；**
- **9 FPS 的诚实解读**：它证明的是"这条路能实时"，不是"这个世界能实时"。**做预研评估时，这类数字要换算成"目标分辨率下的等效 FPS"，否则会高估一个数量级。**

## Unreal Engine Relevance

- **世界分区 / 流送**：L3G 的 spawn mask（opacity 低 + 误差高）本质上是一个**"缺口探测"**，可以类比 UE 的 World Partition 流送单元判定——但 UE 判的是"距离与体积"，它判的是"**渲出来对不对**"。**这个判据更贵但更准**：它抓的是"这块看着不对"，而不只是"这块还没加载"；
- **Niagara / 特效记忆**：如果要做"特效留下的痕迹"（脚印、弹孔、烧焦），L3G 的"只在缺口长、已有只精修"是正好的策略——**别每帧往缓冲里堆一张 decal，那是 $O(T)$**；
- **不适用处**：动态物体、需要改光的场景——**latent 记忆没有分解，改光就要重训**。

## Technology Evolution

```text
"世界模型的记忆存在哪"（成本 = 什么 × 什么）
2024  Genie / Oasis        时序上下文        O(T·R)      —— 记得住，装不下
2025  Spatia / NeoVerse    逐帧重建 3D       O(T·R)      —— 装得下，越堆越慢
2026  ArtiFixer / Lyra 2.0 场景级优化        O(S)        —— 缩放对了，读太慢
2026  ★ L3G-World          选择性写 + latent O(S) + 免重编码 —— 两头都降
        ↓
      "把格式对齐到消费者"这一步，省的是 318×；
      "把预算对齐到缺口"这一步，省的是 1.8×。
      —— 两个数量级差，说明"翻译"比"冗余"贵得多。
```

## Relationships

### Based On

- **[[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation|UniWorld]]**：同一个目标（显存与时长无关），不同的手法——UniWorld 靠**窗口边界读一次**，L3G 靠**只写缺口**。**前者是"少读"，后者是"少写"，两条都成立，可以叠加**；
- **[[2026-10-26-Liu — WorldPager Zero-Stall Tiered Memory for Long-Horizon World Models|WorldPager]]**：WorldPager 说的是"容量墙 vs 位置墙"，L3G 把容量墙整面推掉了（$O(S)$），**但位置墙它没碰**（没有预取、没有分层，只有"当前窗口看到什么就修什么"）。**两篇合起来才是完整的内存系统**。

### Extends

- **[[Gaussian Splatting]]**：把高斯从"渲染原语"扩成"**记忆单元**"——每个高斯不只带几何与颜色，还带一份**给别的模型的 latent**。这是 GS 的第二次"身份变化"（第一次是从优化结果变成前馈预测）；
- **[[Streaming]]**：$O(S)$ 的记忆 + 增量写入 = 一个**没有淘汰策略的流送器**。缺的正是 Streaming 页里那条"断崖 vs 斜坡"的讨论。

### Related

- **[[2026-10-29-Li — Fight World Model Customizable Combat through Visual Interaction Proxies|Fight World Model]]**：今天的两篇都在解决"两个子系统之间的接口"——FWM 存的是**规则的结果**（框/血条），L3G 存的是**外观的 latent**。**一个为可编辑性牺牲了画面细节，一个为速度牺牲了可编辑性**；
- **[[2026-10-30-BoxWorld Let World Engines Control and Video Generators Render|BoxWorld]]**：同一族（状态与画面分离），但 BoxWorld 的接口是**离散符号**，L3G 的接口是**连续 latent**。**符号可测、latent 只能看**——这是两条路线的根本分野。

### Followed By

- 待观察：**动态物体的记忆**（把静态/动态解耦是作者列的第一条 future work）；
- 待观察：**记忆的淘汰策略**（$O(S)$ 在超大世界里仍会涨）。

## Personal Knowledge State

`Normal`。这篇的机制不难（mask → 聚类 → transformer → 残差），**难的是"为什么这么设计"那层判断**：哪一笔该归零、归零的代价落在哪。读法建议：

1. 先读 Table 1（一页看懂"成本绑在哪个变量上"）；
2. 再读 §3.2 的 spawn mask 段落（**先筛再聚类**这个顺序）；
3. 最后读 Table 3 的 "w/o VAE Lifting" 那一行（2.97 vs 943.6）。

**跳过**：DiT 的蒸馏细节（AnyFlow 的 flow-map 推导），那是另一个领域的账。

## Learning Value

- **一条可迁移的判据**：*两个子系统之间，真正贵的是"翻译"，不是"数据量"。* 判断方法——把翻译那一步的耗时单独测出来，如果它占比 >50%，那该改的是接口，不是模型；
- **一条可迁移的工程习惯**：**冗余要按"缺口"分配，不按"帧"分配**。L3G 的 1.8× 与 318× 摆在一起说明：**对齐格式的收益，通常比省冗余大一个数量级**。

## Visualization

![[Research/Files/Pictures/Choi 2026 — 图2 L3G-Mem 自适应生成与精修架构.png]]
**读图**：左边 (a) 是 L3G-Mem 的完整数据流——`Online Window`（视频 + VAE latent + 相机）进来，**先渲旧记忆**得到 RGB 与 Alpha，Alpha 低的地方进 `Spawn Mask`，只对这些区域做 `K-means` 得到 spawn query；同时被看到的高斯取出 `T^rf` 做精修。两路 token 拼在一起过 transformer（**注意中间那层 `Global Self-Attention`：它是唯一让"新长的"与"旧的"互相看见的地方**），分别解码出 `G^sp`（新高斯）与 `ΔG^rf`（残差）。左下的 `VAE Latent Lifting` 分支是今天的主角：它从 $z_k$ 取 latent，**与 spawn query 共用空间索引**，最后按通道拼进去。右边 (b) 是四行可视化：**Input Video / Memory RGB / Memory Alpha / Spawn Mask / 更新后的 Memory**——看 Spawn Mask 那一行，白色只出现在**建筑边缘与新露出的地面**，大片已覆盖区域是黑的。**这张图就是"预算按缺口分配"的字面样子。**（图 2，Choi et al. 2026）

![[Research/Files/Pictures/Choi 2026 — 图1 无限生成与显存溢出对比.png]]
**读图**：左边是"单图 + 文本 + 相机轨迹"进入 `Latent 3D Gaussian Memory`（高斯几何 + VAE latent 叠在一张彩色图上），右边是 L3G-Gen / L3G-Mem 交替的循环，产出无限延伸的场景。**最该看的是下面两张曲线**：左图 `Memory (GB)` 随生成时长——**L3G 是一条平的蓝线，WorldKV / Yume 1.5 一路爬到 OOM（红叉）**；右图 `Throughput (FPS)`——**L3G 是唯一贴着"无限生成"横线的（约 9 FPS），其余三条在 0–2 FPS 区间并陆续 OOM**。两张图合起来讲的就是 Table 1 那两列：**一个斜率，一个截距。**（图 1，Choi et al. 2026）

## Notes

**三处必须记住的数字对照**

| 对比 | 数字 | 说明 |
|---|---|---|
| 存 latent vs 存 RGB（VAE 渲染） | **943.6 vs 2.97 FPS（318×）** | 全部来自"不再重编码" |
| 选择性 spawn vs 全量聚类 | **41K vs 75K 图元（1.8×）**，PSNR 16.52 vs 16.26 | 省的是冗余，质量差一点点 |
| 精修 vs 不精修（世界级） | 自一致性 PSNR **15.11 vs 14.82**；位姿 **RotErr 5.22° vs 7.85°** | 精修不只提画质，还提**几何可控性** |

**指标打架，要记下来**

同一张 Table 2 里，L3G 的 **PSNR 16.00（最好）/ LPIPS 0.366（最好）**，但 **SSIM 0.441（输给 NeoVerse 0.525、3DScenePrompt 0.528）**。Table 3/4 的消融更直接：**去掉 VAE lifting 的那一版 PSNR 反而更高**（15.41 vs 15.11）、SSIM 也更高（0.480 vs 0.431），但 LPIPS / FID / IQ 更差。

**这不是"论文写错了"，是三个指标量的东西不同**：PSNR 量逐像素误差、SSIM 量局部结构相似、LPIPS 量感知。**当生成结果的"结构对、纹理糊"时，SSIM 会掉而 PSNR 未必**。与 [[2026-10-30-Liu — Texture Consolidation of Generated Multi-View Renderings|Texture Consolidation]] 那次 SSIM 反转（0.615 vs 0.684，赢的正是产生鬼影的做法）是同一类：**验收前先问"这个指标会不会奖励我的失效模式"。**

**一处可以马上用的对照**

`w/o Refinement` 在 Table 4 里不只掉画质，**位姿误差从 5.22° 涨到 7.85°、MEt3R 从 0.081 涨到 0.091**。也就是说：**记忆不修，不只是"看着糊"，而是"相机都不准了"**——因为下一段生成是以这块记忆为条件的，错的几何会把相机估计一起带偏。**"误差会沿自回归传播"这句话，在这里第一次有了可读的数字。**
