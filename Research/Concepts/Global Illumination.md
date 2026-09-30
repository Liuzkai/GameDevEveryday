---
type: concept
user_level: Normal
aliases: [Global Illumination, GI, 全局光照, 全局照明, 间接光]
prerequisites: ["[[Rendering Equation]]"]
first_introduced: "1986（Kajiya 渲染方程给出完整表述）；工程近似路线自 1980s-90s 起（辐射度 / 路径追踪）"
---

# Global Illumination

## Definition

**把"间接光"完整计入的图像合成问题**——物体不仅被光源直接照亮，还被**其他物体反弹的光**照亮；而反弹光自身又参与下一次反弹，直到能量耗散。

```text
直接光（Direct）：光源 → 表面 → 眼睛                    只要打得到光源就行
全局光（GI）：    光源 → A → B → C → … → 眼睛            每个 bounce 都是一次全场景积分
```

- **GI 的完整解** = [[Kajiya — The Rendering Equation (1986)]] 的完整解；"直接光"只是它忽略多次反弹项的近似；
- **GI 的实时化史 = 一部"如何假装自己算了全部反弹"的近似史**——本笔记的核心价值就是这张近似谱系表。

## Core Principle

**为什么 GI 天然贵：账单写在"反弹次数"里。**

- 一次间接反弹 = 对**全场景**再积分一次（射线上要问"我打到了什么、它怎么反光"）；
- 反弹两次、三次……成本随路径长度增长，而画面质量的边际收益递减（能量逐次衰减）；
- **所以所有实时 GI 方案的共同结构是**：把"无限次、空间无界的反弹积分"改写成**有限采样 + 缓存/预计算/降维**的某个组合。**成本函数关于哪个变量线性**——评估任何 GI 方案的第一问。

## 传统 GI 近似谱系（**读完这张表即可接手 [[Neural Global Illumination]] 的桥**）

| 代 | 方案 | 思路 | 成本/限制 |
|---|---|---|---|
| 0 | 环境光常数 / 半球光 | 全局一个数 | 无方向性（"死光"） |
| 1 | **IBL / 环境贴图** | 把"远处环境"作为光源；镜面用反射捕获 | 无局部遮挡（漏光） |
| 2 | **SH 表示（9 阶）** | 环境光的解析低频表示 | [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]；移动端 GI 的根源 |
| 3 | **辐射度（Radiosity）** | 把间接光变成"面片之间传能量"的矩阵问题 | 静态、漫反射专用（1990s 离线/烘焙） |
| 4 | **PRT / 预计算转移** | 预烘"表面如何接收环境"的转移算子 | 静态几何；材质自由度受限 |
| 5 | **屏幕空间族（SSAO → SSGI / SSR）** | 只在已有像素里找反弹 | 便宜；**屏幕外一律丢失**（"屏幕空间 vs 引擎感知"） |
| 6 | **缓存/Direct 祖先族：RSM / VPL / Irradiance Caching / DDGI** | 把间接光"记在小本子上"复用 | ⚠️ **这一族是 [[Neural Global Illumination]] 的具名前缺口（桥材料）**：RSM=第一个 bounce 存进阴影贴图；VPL=把发光面变成"很多小灯"；缓存=空间上插值复用 |
| 7 | **光追 GI（硬件）** | 真反弹，降噪器补噪声 | DDGI / Lumen（UE 默认）/ MegaLights（直接光侧）；贵，PC 高端 |
| 8 | **神经 GI** | 让网络学"间接光长什么样" | 离实时有数量级差距；见 [[Neural Global Illumination]]（Hard） |

## Historical Evolution

```text
1986  Kajiya：渲染方程（"全部反弹"第一次有完整表述）+ 路径追踪解
1984-90s  辐射度方法（面片间传能）——工程界第一次尝到"间接光"的甜头（离线）
1990s-2000s  PRT / SH 光照（[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]）
2000s  实时阴影/环境光工程化；SSAO（2007 前后）
2010s  SSGI / SSR / 体素 GI / 光追 GI 原型（DDGI 2019）
2020s  Lumen 把"软件/硬件光追 GI"做成引擎默认；烘焙与实时合流
2021  神经 GI 成为独立研究方向（本库 [[Neural Global Illumination]]，Hard）
2026  MegaLights（动态光源侧 Production）+ E-Day 把"全 GI"与硬件光追门槛绑定（强约束样本）
```

## Important Papers

| 节点 | 角色 | 状态 |
|---|---|---|
| [[Kajiya — The Rendering Equation (1986)]] | "全部反弹"的完整表述（本概念的容器） | ✅ 入库 |
| [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] | 环境光的 9 阶 SH（移动端 GI 根源） | ✅ 入库 |
| [[LightOpt — Lights Optimization for Real-Time Rendering]] | 用可微优化反推"光照布局怎么配" | ✅ 入库（DR 桥目标） |
| RSM / VPL 原始文献 | **具名缺口**（Neural GI 的前置） | 记名待入库 |
| 神经 GI 侧：| 见 [[Neural Global Illumination]] 的桥 | 观察 |

## Related Concepts

- [[Rendering Equation]] —— GI = 它的完整解（**本概念的直接前置**）
- [[Real-Time Global Illumination]]（Technology）—— 实时化工程侧（Lumen / DDGI / MegaLights）
- [[Neural Global Illumination]]（Hard）—— 学习侧目标；**本条目第 6 行（RSM/VPL/缓存）就是它的桥材料**
- [[Shadow Mapping]] —— 直接光侧的影子账（"每灯 +1×"）；GI 侧的成本法则在反弹次数
- [[Participating Media]] / [[Linear Transport Theory]] —— 体积 GI（雾中光柱、云的自阴影）
- [[Scalability and Quality Tiers]] —— GI 的档位化（烘焙 vs 实时、SH vs 光追）

## Technologies

- [[Real-Time Global Illumination]]（引擎侧总条目）· Lumen · DDGI · MegaLights（直接光现代化）
- 烘焙管线（Lightmass / 预计算 GI）

## Game Applications

- **开放世界的时间预算**：昼 / 夜 / 天气切换下"光照要不要重算"是 GI 选型的核心（烘焙静态 vs 实时动态）；
- **艺术意图**：GI 的"准不准"与"好不好看"在本库已有样本（[[Temporal Stability and Artistic Intent]]：巫 3 的"氛围 vs 准确"）；
- **与你预算工作的接口**：动态灯光维度（≤3/≤2/≤1/0）的成本本质 = "每盏灯要不要再买一次 GI/阴影账"——**MegaLights 之后，PC 档的账从"灯数"改写成"光照复杂度"**。

## Personal Knowledge

- `user_level: Normal`（**桥材料层**）——分层读法：
  - **机制层（Easy，不复述）**：反弹 / 能量衰减 / 直接光 vs 间接光的直觉——你的日常工作；
  - **成本模型层（Normal，本笔记主体）**：**八个近似的取舍结构**（空间边界 / 时间复用 / 维度约简 / 采样预算）；具名缺口 = RSM / VPL / 缓存族的机制（**30 分钟级，查表可得**）。
- **一句话检验**：能说出"**每一个实时 GI 方案都是在回答'把无限反弹积分换成什么有限结构'**"即到位。

## Learning Gap

- **需要**（30 分钟级）：RSM / VPL / Irradiance Caching / DDGI 四者的"缓存结构"是什么——**这是 [[Neural Global Illumination]] 桥的具名前置**；
- **不需要**：辐射度矩阵求解、PRT 数学细节（工具已废弃，只需知道它存在过）。

## Next Step

1. 查清 RSM / VPL 的代表文献（记名 → 必要时入库），补齐 [[Neural Global Illumination]] 桥的前置缺口；
2. 做"动态灯光维度复审"时，把本表第 7/8 行（Lumen / MegaLights 的能力边界）与你的五档矩阵对照一次；
3. 保持与 [[Real-Time Global Illumination]]（Technology）的同步：引擎侧有新证据（如 E-Day 实测）时双向更新。

## Notes

- **为什么现在才建**：首日建库时本概念被列为"基础锚点"但文件一直缺失（[[Neural Global Illumination]] 的桥长期缺"经典 GI 近似"这一环）——2026-09-30 链接完整性检查时发现并补齐。同期补齐的还有 [[Real-Time Rendering]]。
- 本概念**不是** [[Neural Global Illumination]] 的重复：前者是"问题与近似谱系"，后者是"神经方法目标"——关系为 Prerequisite（本概念 → 神经 GI）。
