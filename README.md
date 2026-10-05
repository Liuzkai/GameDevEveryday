# GameDevEveryday

一个持续积累的**游戏研发技术研究与个人学习知识库**。通过论文阅读、概念梳理、技术追踪和学习记录，把经典理论、前沿研究与游戏制作中的实际问题联系起来。

内容以中文 Markdown 笔记为主，配合原论文链接、图示、交互式 HTML 和教学伪代码；可在 GitHub 浏览，也可作为 Obsidian 知识库使用。

## 这个库用来做什么

- **跟进研究**：整理游戏研发相关论文、技术动态和后续观察事项，形成每日记录、周报和月报。
- **理解原理**：从经典论文出发，解释核心概念、公式、图示及实现思路。
- **连接工程应用**：记录技术的适用场景、限制、性能成本与潜在游戏开发用途。
- **建立知识关系**：用双向链接串联论文、概念、技术、应用与学习路径。
- **记录学习进度**：维护阅读状态、理解程度和自测清单，帮助决定下一步学什么。

主要主题包括实时渲染、PBR 与全局光照、Gaussian Splatting、神经与可微渲染、角色动画、物理仿真、程序化内容生成，以及 VFX 性能与质量分档。

## 从哪里开始

1. 打开 [研究总索引](Research/Index.md)，查看主题和论文入口。
2. 在 [每日研究](Research/Daily/) 中按日期阅读前沿研究与经典研究。
3. 进入 [论文笔记](Research/Papers/)，结合原文、图示和检查表深入理解。
4. 遇到陌生概念时查阅 [概念库](Research/Concepts/)，或沿 [学习路径](Research/Learning/) 补齐前置知识。
5. 通过 [周报](Research/Weekly/)、[月报](Research/Monthly/) 和 [技术雷达](Research/Radar/) 回顾主题之间的联系。

## 目录结构

| 目录或文件 | 用途 |
|---|---|
| [Research/Index.md](Research/Index.md) | 知识库总入口 |
| [Research/Papers/](Research/Papers/) | 论文阅读、方法分析、局限与学习检查表 |
| [Research/Concepts/](Research/Concepts/) | 基础概念及其关联 |
| [Research/Technologies/](Research/Technologies/) | 技术路线、工程实现与应用状态 |
| [Research/Applications/](Research/Applications/) | 游戏研发应用场景 |
| [Research/Daily/](Research/Daily/) | 按日期组织的研究记录 |
| [Research/Weekly/](Research/Weekly/) / [Research/Monthly/](Research/Monthly/) | 阶段回顾与综合整理 |
| [Research/Learning/](Research/Learning/) | 学习路径与专题讲解 |
| [Research/Radar/](Research/Radar/) | 技术跟踪与观察事项 |
| [Research/Files/](Research/Files/) | 图片及 HTML 图解等配套资源 |
| [Research/Personal Knowledge Model.md](Research/Personal%20Knowledge%20Model.md) | 个人知识状态与学习规划参考 |
| [Rules.md](Rules.md) | 研究组织与维护规则 |
| [game-research-knowledge-skill.md](game-research-knowledge-skill.md) | 辅助研究与知识整理的工作流程说明 |
| `.obsidian/` | Obsidian 配置与插件 |

## 阅读与维护方式

**在 Obsidian 中**：将仓库根目录作为已有知识库打开，可以使用笔记中的 `[[双向链接]]`、属性、关系图谱和阅读检查表。

**在 GitHub 中**：从本 README 的标准链接进入各目录。部分笔记使用 Obsidian 链接语法，GitHub 不会将其自动解析为可点击链接；HTML 图解可下载后在浏览器中打开。

笔记的 YAML 属性用于记录作者、来源、研究分类、阅读状态和理解程度。`user_level` 中的 `Easy / Normal / Hard` 表示个人当前理解程度，与论文重要性分级分开使用。个人认知模型中标为推断的内容，需要结合实际学习情况校正。

新增内容时优先补充已有主题，保留来源，并把论文结论、个人解释、工程推断和教学示例区分清楚。图片与配套资源放在 `Research/Files/`，使用有效的相对路径引用；完成阅读后更新状态或自测清单，再通过 Git 提交同步。

## 内容说明

这是持续更新的研究与学习记录，部分内容由 AI 辅助整理。涉及发布时间、性能数字或实现细节时，应结合笔记中的原始来源核对；教学伪代码不等于作者原始实现，也不一定能直接用于生产环境。

引用论文与原图的权利归相应作者或出版方；本仓库中的引用不改变原材料的许可条件。
