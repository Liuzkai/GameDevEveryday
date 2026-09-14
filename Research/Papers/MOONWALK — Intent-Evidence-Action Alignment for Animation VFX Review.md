---
type: paper
title: "MOONWALK: Mediating Operations with Intent-Evidence-Action Alignment Across Junior-Supervisor Review Workflows in Animation/VFX Pre-Production"
authors: ["Shih-Yu Lai", "Wen-Fan Wang", "Sai Ling", "Shaune Jan", "Bing-Yu Chen", "Xiang Anthony Chen"]
year: 2026
published: "2026-09-09"
venue: "arXiv (cs.HC)"
url: "https://arxiv.org/abs/2609.10385"
code: "https://github.com/Akinesia112/Moonwalk/tree/english-version"
category: [production-workflow, hci, ai-tools]
importance: "B"
game_relevance: "中（VFX/动画评审流程，生产工具向）"
production_readiness: "Prototype（有 in-studio 研究）"
user_level: "Easy（流程语言）"
status: unread
---

# MOONWALK — Intent-Evidence-Action Alignment for Animation/VFX Review

## TL;DR

针对动画/VFX 前期制作中"总监意图 → 新人执行"的评审断链问题，提出 intent-evidence-action 对齐框架：意图进共享项目记录、判断锚定到证据、授权决策转成明确修改任务。**AI 只做行政协调**（标记缺失上下文、整理笔记），**创意权完整保留给人**。在真实工作室的对照研究显示：结构化工作流显著优于纯聊天机器人界面。代码已开源。

## Problem

评审标准随迭代漂移、判断失去证据基础、修改请求背后的推理在 senior-junior 交接中丢失。

## Core Idea

三个对齐：**intent**（意图落入共享记录）→ **evidence**（判断锚定参考/规格）→ **action**（授权决策转成与参考笔记直接绑定的修改任务）。

## Key Contribution

- 实证结论：结构化的 intent-evidence-action 工作流 > 无结构对话式 AI chatbot
- 权限设计原则：AI 管行政协调，**美学权威与最终优先级必须留在人手里**——人机分工的边界证据

## Game Development Relevance

- 直接对应动画/VFX 评审流程——与你的工作场景（技能 VFX 评审/预算验收）同领域
- "判断要锚定证据"与你的预算体系哲学一致：评审意见应该能指回量化指标，而不是口头漂移

## Unreal Engine Relevance

无（流程工具）。

## Limitations

HCI 研究样本量有限；前期制作（pre-production）场景，非运行时技术。

## Related Concepts

- [[Real-Time VFX Performance Budgeting]]（你的领域：预算即"锚定证据的判断标准"）

## Related Papers

- 无直接关联

## Personal Knowledge State

Current Level: Easy（流程语言无障碍）

Reason: 不涉及新理论，价值在实证与设计原则。

## Learning Path

无需学习。若想借鉴：它的 intent-evidence-action 三段式可以作为你设计"VFX 预算评审记录格式"的参考模板。

## Notes

入库理由：少数直接研究**VFX 评审工作流本身**的论文，且有真实工作室对照实验与开源代码。对"淬炼数值表 → 评审 → 返修"这类流程的数字化有参考价值。
