---
type: concept
user_level: Normal
aliases: [Animation Compression, 动画压缩, 骨骼动画压缩, ACL, Animation Compression Library, 动画编解码]
prerequisites: []
first_introduced: "ACL（开源生产库，2016-）；学习化路线自 2026（CurveCodec 系列）"
---

# Animation Compression

> 建立于 2026-10-06（由 [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model|CurveCodec 2]] 触发）。**本库动画线的"数据侧"第一节点**：此前动画线覆盖"动作怎么生成/怎么检索"（[[Motion Generation]] / [[Motion Matching]] / [[Neural Animation]]），这一篇补上"**动作怎么存、怎么传**"——对做预算工作的人，它是**动画内存账单**的正面问题。

## Definition

**把"每关节 × 每帧的变换流"压到最小，同时满足两条生产约束：① 骨架无关（任意 rig 都能用）；② 有声明可验证的误差界。** 动画可以压狠一点，但"破坏性误差"必须是**被界定、被声明、被验证**的——所以这个领域的核心语言是**契约（contract）**，不是单纯的压缩率。

## Core Principle

### 三本账（ACL 作者 Nicholas Frechette 的原话要点）

| 账 | 说明 |
|---|---|
| **内存足迹** | "size is king"——少碰内存往往就更快；移动端 / Switch 的 CPU cache 极小（ACL 学纹理压缩 BC7：压缩态直接在内存里用，不中间解包）|
| **解码速度 / 延迟** | 动画基本跑在 CPU，常是帧内最慢项之一；一个角色一帧可能要解 **3–20 个片段**（motion matching / blend space）|
| **导入时间** | "旧 UE5 编解码器压缩 1–2 分钟的过场动画要 45–90 分钟，ACL 几秒"——**压缩器性能是生产管线账**（现代项目 **>100k clips**）|

### 两种代码器分工（2026 年的清晰切割）

```text
运行时代码器（ACL 式）             装载时代码器（CurveCodec 2 式)
─────────────────────            ─────────────────────
无状态 / 随机访问 / 只解需要姿态     顺序 / 一次解完 / 无随机访问
为"每帧都要快"设计                 为"字节数最小 + 位精确"设计
适合热数据（locomotion 等高频混用）   适合冷数据（过场、存档、传输）
```

两者**互补而非替代**（CurveCodec 2 原文 Q&A 直答 "Could it replace ACL? No."）。

### 冗余解剖（CurveCodec 2 的三组测量）

1. **最大收益来自"从曲线自身历史预测"**（不是任何生成式组件）；
2. 其次来自**逐关节、闭环、经正向运动学验证的"哪些样本不编码"**——且"丢样本"每比特的失真代价只有"加大量化步"的 1/5–1/10；
3. **学习式"补帧"不成立**（百万样本最近邻 oracle 都不比线性插值好）——**网络真正能接管的只有"残差分布"（熵编码）**。

配套事实：**关节曲线里装着的一部分是"身体"，不是"动作"**——同一批动作重定向到另一具骨架后压缩率好 2.2–2.3×；而学到的熵模型能泛化到**完全没见过的物种**（狗，13.9×）。

### 误差契约的形式

- 两种口径：**最坏情形**（逐关节，对 ACL 自己的 worst case 设容差）或**平均**（逐片段）；
- **回退 + 计数**：不达标的片段回退到参照代码器（ACL），回退率纳入统计——质量声明是**可审计**的；
- 档位语言：**误差界即档位**（0.01 / 0.1 / 1 cm 连续可选）——与渲染侧的"预算定在样本上还是误差上"同构（参见 [[Scalability and Quality Tiers]]）。

## Historical Evolution

```text
2010s   量化 / range reduction / 变位宽（ACL 阶段账本：这些步骤合计 ≈ 17.7×；
        在其符号上再加熵编码最多再省 16–20%——"瓶颈在'编码什么'，不在后端"）
2016-   ACL（Animation Compression Library，开源）成为事实标准：误差界 + 无状态 + 随机访问
        → UE / Unity / 多家工作室采用
2026    CurveCodec 1（SIGGRAPH Asia 2026）：学习先验 + 稀疏锚点——匹配 ACL 平均误差、
        最坏情形不稳、按浮点计账
2026    ★ CurveCodec 2：决策交回经典（闭环量化 + RD 选键）+ 学习只做熵编码
        → 0.37× / 0.22× ACL 字节（两种契约口径），整型位精确
```

## Important Papers

| 节点 | 角色 |
|---|---|
| [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model]] | ★ 2026-10-06 入库：当前最优样本（两阶段 + 契约 + 0.37×/0.22×） |
| CurveCodec 1《Neural Codec for Skeletal Animation Compression》| SIGGRAPH Asia 2026 Conference Papers（DOI 10.1145/3829340.3842192）——记名；被 v2 取代 |
| ACL（工程实现）| 非论文；"误差界"的生产参照系；作者设计哲学见 CurveCodec 2 项目页邮件原文 |

## Related Concepts

- [[Open World Character Animation]]——动画管线的应用侧（本概念是其"数据侧"）
- [[Motion Matching]] / [[Motion Generation]]——"动作怎么来"；本概念管"动作怎么存"
- [[Scalability and Quality Tiers]]——"误差界即档位"的预算语言接口

## Technologies

- **ACL**（开源库；UE 以插件生态方式使用）
- 引擎内建动画 codec（传统管线，导入时间账的主要来源）
- CurveCodec 系列（研究原型；代码与浏览器 demo 已放出）

## Game Applications

- **内存预算**：动画数据是角色内容常驻大头；0.37×/0.22× 意味着同内容直接重写内存账单；
- **冷热分层**（工程推断）：冷数据（过场 / 低复用剪集）装载时解码；热数据（高频混用）保留运行时随机访问；
- **与 VFX / 用户的接口**：技能动画与特效共享时间轴——动画数据的误差档 = 动作精度预算，会传导到特效的时序锚点（见 [[Open World Character Animation]] 的耦合说明）。

## Personal Knowledge

- `user_level: Normal`（**推断值**——本域 2026-10-06 新建；按 PKM 校正原则标注）：你的"动画是内存项"工程直觉在 Easy 区；"熵编码 / 契约口径 / 闭环节奏"是新的机制层；
- 与你的工作真实接口：**误差界当成档位旋钮**、**三本账（内存/解码/导入）**、**冷热分层**。

## Learning Gap

- 熵编码侧名词（rANS / 位精确整型推理）——概念级即可，不必推导；
- 若要做引擎侧试验：需先弄清目标引擎的运行时接管路径（ACL 插件接口）。

## Next Step

1. 读 [[2026-10-03-CurveCodec 2 — Skeleton-agnostic Animation Compression with a Learned Entropy Model]] 的 **Q2（"能不能取代 ACL"）** 与三组测量——30 分钟，机制层到位；
2. 顺手查一次自己项目的动画内存占比（一个数字即可）——把本概念挂到真实账本上。
