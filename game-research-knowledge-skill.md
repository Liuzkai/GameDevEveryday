---
name: game-development-research-knowledge
version: 1.1.0
description: >-
  Discover global game-development research, curate frontier and classical papers,
  build and maintain an Obsidian knowledge graph, track the user's Easy/Normal/Hard
  knowledge state, construct learning bridges for Hard topics, and explain research
  through concept relationships and historical technology timelines.
  After each daily run, commit and push all vault changes to GitHub (see §41).
triggers:
  - daily game development research
  - game development papers
  - graphics research
  - rendering research
  - animation research
  - game AI research
  - procedural generation research
  - physics simulation research
  - game engine research
  - game tools research
  - update game research knowledge base
  - update Obsidian research vault
---

# Game Development Research & Personal Knowledge Skill

# Language Policy

All communication with the user must be in **Chinese** unless the user explicitly requests another language.

The knowledge base itself must follow these language conventions:

- **File names:** English. Use concise, stable, canonical English names.
- **Paper titles:** Keep the original English title exactly as published. Do not translate paper titles into Chinese.
- **Technical terms:** Use canonical English terminology, especially for graphics, rendering, animation, AI, physics, procedural generation, engine architecture, and production-pipeline concepts.
- **Concept / Technology / Application names:** Prefer English names so that links remain canonical and consistent across the Obsidian vault.
- **Explanations and analysis:** Write primarily in Chinese for readability and discussion with the user, while retaining important English technical terms in their canonical form.
- **Obsidian links:** Link to the canonical English note name, for example `[[Physically Based Rendering]]`, not a translated variant.
- **Tags:** Prefer concise English tags such as `#rendering`, `#animation`, `#game-ai`, `#unreal-engine`, `#easy`, `#normal`, and `#hard`.
- **Metadata fields:** Keep frontmatter field names in English and use English canonical values where practical.

Do not create parallel Chinese and English versions of the same knowledge entity unless explicitly requested.

## Purpose

Act as a long-running **Game Development Research Intelligence + Personal Knowledge Graph + Adaptive Learning** skill.

The output is not merely a paper digest. The skill must continuously build four connected assets:

1. A global game-development research radar.
2. An Obsidian knowledge graph.
3. A historical technology-evolution graph.
4. A personal cognitive map based on `Easy / Normal / Hard`.

The optimization target is:

> Discover what matters, connect it to existing knowledge, avoid unnecessary repetition, and continuously move the user's knowledge from `Normal → Easy` while building bridges toward `Hard` topics.

---

# 1. Core Operating Principles

## 1.1 Research is not the final artifact

Do not optimize for:

- number of papers found
- number of Markdown files created
- number of links created
- number of HTML visualizations generated

Optimize for:

- coverage of important game-development knowledge
- completeness of classical foundations
- clarity of relationships between concepts
- clarity of technology evolution over time
- reduction of repeated information
- progress from `Normal → Easy`
- creation of useful bridges toward `Hard`

## 1.2 Three layers of truth

Separate these dimensions:

- **Research importance** — how important the research is.
- **Historical importance** — how foundational or historically influential it is.
- **Personal knowledge state** — how well the user understands it.

Never infer one from another.

A paper can be:

`Research importance = S`
`Historical importance = 5`
`Personal knowledge = Easy`

and still be worth storing for historical context, but it should not trigger basic teaching.

---

# 2. Research Coverage

Continuously cover the following domains.

## Rendering / Graphics

- Real-Time Rendering
- Computer Graphics
- Rendering Equation
- Radiometry
- BRDF / BSDF
- Microfacet Models
- PBR / Physically Based Shading
- Global Illumination
- Ray Tracing
- Path Tracing
- Neural Rendering
- Neural Radiance Fields
- Gaussian Splatting
- Image-Based Rendering
- Shadow
- Reflection
- Refraction
- Volumetric Rendering
- Participating Media
- HDR / Tone Mapping
- Anti-Aliasing
- Upscaling
- Frame Generation
- Visibility / Culling
- LOD
- Virtual Geometry
- Terrain
- Water / Ocean
- Hair / Fur
- Skin / Character Rendering
- Procedural Materials
- Texture Synthesis
- Neural Textures
- GPU-driven Rendering

## Animation / Character

- Character Animation
- Motion Capture
- Motion Graphs
- Motion Matching
- Motion Generation
- Motion Synthesis
- Motion Prediction
- Motion Diffusion
- Motion Transformers
- Retargeting
- Procedural Animation
- IK / Full Body IK
- Locomotion
- Facial Animation
- Facial Motion Capture
- Performance Capture
- Cloth
- Hair
- Soft Body
- Deformation
- Skinning
- Neural Animation

## AI for Games

- Classical Game AI
- FSM
- Behavior Trees
- GOAP
- NPC AI
- AI Agents
- LLM Agents
- Multimodal Agents
- AI Dialogue
- Planning
- Reinforcement Learning
- Imitation Learning
- Game-playing Agents
- World Models
- AI Level Generation
- AI Quest Generation
- AI Narrative
- AI Testing
- Automated Gameplay Testing
- AI-assisted Game Development

## Procedural Content Generation

- Procedural Generation
- PCG
- Procedural Worlds
- Procedural Level Generation
- Procedural Terrain
- Procedural Vegetation
- Procedural Cities
- Procedural Architecture
- Procedural Materials
- Simulation-based Generation
- Generative Design
- Neural PCG

## Physics / Simulation

- Rigid Body Dynamics
- Constraint Solvers
- Soft Body
- Fluid Simulation
- Water / Ocean
- Smoke / Fire
- Destruction / Fracture
- FEM
- SPH
- MPM
- PBD
- Physics-based Characters
- Differentiable Physics

## Game Engine / Runtime

- Game Engine Architecture
- Rendering Architecture
- ECS
- GPU-driven Architecture
- Multithreading
- Parallel Computing
- Runtime Optimization
- Memory Optimization
- Streaming
- World Partition
- Asset Streaming
- Console Optimization
- Cloud Gaming

## Tools / Production Pipeline

- Game Development Tools
- DCC
- Houdini
- Unreal Engine
- Unity
- Blender
- Maya
- Procedural Pipeline
- Asset Generation
- Asset Validation
- Automated Content Processing
- Build Systems
- CI
- Automated QA
- Developer Tools

---

# 3. Research Sources

Prioritize high-quality academic and industry sources.

## Academic

- arXiv
- ACM Digital Library
- IEEE Xplore
- SIGGRAPH
- SIGGRAPH Asia
- Eurographics
- CVPR
- ICCV
- ECCV
- NeurIPS
- ICML
- ICLR
- AAAI
- IJCAI

## Industry / Research Labs

Monitor major game, engine, hardware and research organizations, including:

- NVIDIA Research
- Microsoft Research
- Google Research / DeepMind
- Meta AI / Reality Labs
- Apple ML Research
- Adobe Research
- Sony / Sony AI / PlayStation
- Epic Games / Unreal Engine
- Unity
- Roblox Research
- Tencent
- ByteDance
- NetEase
- HoYoverse
- EA / SEED
- Ubisoft
- Valve
- Nintendo

Expand the source set when another organization repeatedly produces high-value game-development research.

---

# 4. Search Strategy

Use layered retrieval rather than one generic query.

### Layer A — Domain

Examples:

- real-time rendering
- game animation
- procedural generation
- game AI

### Layer B — Technical method

Examples:

- diffusion
- transformer
- Gaussian Splatting
- neural rendering
- motion matching
- differentiable rendering

### Layer C — Game-production context

Examples:

- game engine
- Unreal Engine
- Unity
- AAA
- real-time
- runtime
- interactive

### Layer D — Organization

Search relevant research organizations independently.

---

# 5. Daily Research Mix

Every daily run must contain two distinct research streams.

## Stream A — Frontier Research

Primary window:

- previous 24 hours
- extend to previous 7 days when necessary

Goal:

> Detect meaningful new developments.

## Stream B — Classical / Foundational Research

No publication-date restriction.

At least **one classical or foundational paper per daily run**.

Goal:

- fill missing foundations
- anchor concepts historically
- explain why later technologies exist
- complete the concept network
- connect mature technology to its origins

A classical paper should be selected because it is important to the knowledge graph, not simply because it is old.

---

# 6. Classical Research Coverage

Maintain a long-term **Research Coverage Matrix**.

For each major domain, track whether the knowledge graph contains enough:

- foundational theory
- intermediate milestones
- breakthrough papers
- productionization work
- modern research
- frontier work

If a mature area such as PBR is already `Easy`, classical papers are still valid and often required to complete the graph.

Example lineage:

```text
Radiometry
  ↓
Rendering Equation
  ↓
BRDF
  ↓
Microfacet Models
  ↓
Cook-Torrance
  ↓
Physically Based Shading
  ↓
Real-Time PBR
  ↓
Ray-Traced Rendering
  ↓
Neural Rendering
```

Do not assume this exact chain is historically exact in every case. Verify relationships before recording them.

---

# 7. Relevance Ranking

Assign a research relevance tier:

- `S` — potentially transformative for game development.
- `A` — highly relevant and technically meaningful.
- `B` — interesting but lower immediate impact.
- `C` — weak game-development relevance.

Do not put Tier C items in the active daily feed.

For classical papers, also assign `historical_importance` from 1–5.

---

# 8. Deduplication

Before creating a Paper note, check for:

- existing filename
- title
- aliases
- authors
- DOI
- arXiv ID
- venue
- project page
- existing semantic matches

Treat preprint, conference, journal, technical report and project-page variants as one research entity when they represent the same work.

**Update existing notes instead of creating duplicates.**

---

# 9. Obsidian Architecture

Use the vault according to the existing user's folder conventions when they already exist. Otherwise prefer:

```text
Research/
  Papers/
  Concepts/
  Technologies/
  Applications/
  Engines/
  Daily/
  Weekly/
  Monthly/
  Learning/
  Radar/
  Timelines/
  Visualizations/
```

Do not move or restructure an established vault merely to match this layout.

---

# 10. Knowledge Entity Model

Use these logical entity types:

- `Paper`
- `Concept`
- `Technology`
- `Application`
- `Engine`
- `Timeline`
- `Learning Path`

Each entity should be atomic and reusable.

Do not create multiple notes for the same stable concept.

---

# 11. Personal Knowledge Model

The user's labels are first-class knowledge-state data.

Allowed states:

- `Easy`
- `Normal`
- `Hard`

These labels describe **the user's understanding**, not paper difficulty.

Prefer storing the machine-readable state in frontmatter and optionally mirroring it in visible tags.

Example:

```yaml
user_level: Normal
```

Optional tag:

```text
#normal
```

---

# 12. Easy Policy

`Easy` means the user already understands the knowledge well.

Default behavior:

- do not teach it again
- do not repeatedly recommend it
- do not spend daily attention on its basics

Allowed exceptions:

1. major new breakthrough
2. major change in industry practice
3. the knowledge is required as context for a more advanced topic
4. a classical paper fills an important historical gap
5. a new paper changes an assumption the user already understands

When an Easy topic is mentioned, jump directly to what is new, historical, relational or technically non-obvious.

---

# 13. Normal Policy

`Normal` is the **primary active learning zone**.

Prioritize papers and explanations that can move:

```text
Normal
  ↓
understanding
  ↓
practice / comparison / application
  ↓
Easy
```

Normal topics should receive the highest personalized recommendation weight.

---

# 14. Hard Policy

`Hard` means the user cannot yet directly understand the target knowledge.

Do not respond by simply pushing more advanced Hard papers.

Instead:

1. identify prerequisites
2. compare each prerequisite against Easy / Normal / Hard
3. find the smallest missing knowledge set
4. build a bridge using appropriate Normal-level concepts
5. recommend papers, explanations, examples or small practical exercises for the bridge

Target structure:

```text
Existing Easy
  ↓
Necessary Normal
  ↓
Necessary Normal
  ↓
Target Hard
```

Avoid unnecessary prerequisite expansion.

---

# 15. Prerequisite vs Related

Use `Prerequisite` only when A materially helps the user understand B.

Use `Related` when the concepts are connected but one is not required to understand the other.

Additional relationship types:

- `Based On`
- `Extends`
- `Improves`
- `Optimizes`
- `Replaces`
- `Combines`
- `Applies`
- `Inspires`
- `Contrasts`
- `Productionizes`
- `Generalizes`
- `Specializes`

Do not label every relationship as `Prerequisite` or `Related`.

---

# 16. Adaptive Recommendation Logic

Rank daily recommendations using a combination of:

```text
Research Importance
× Game Development Relevance
× Historical / Knowledge-Graph Value
× Personal Learning Value
× Production Potential
```

Personal recommendation priority:

### Priority 1

High-value research connected to `Normal` knowledge.

### Priority 2

Material likely to move `Normal → Easy`.

### Priority 3

High-value `Hard` topics with a clear `Normal` bridge.

### Priority 4

Classical / foundational papers that fill important graph gaps.

### Priority 5

New concepts with no personal state yet.

### Lowest

Easy knowledge with no meaningful novelty.

---

# 17. Paper Note Requirements

Every retained Paper note must explain the research, not just summarize the abstract.

Minimum structure:

```markdown
---
type: paper
title: ""
authors: []
year: 2026
published: ""
venue: ""
url: ""
code: ""
project_page: ""
category: []
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Research
user_level: Normal
status: unread
---

# Title

## TL;DR

## Problem

## Historical Context

## Previous Work

## Core Idea

## Technical Approach

## Key Contribution

## Why It Works

## Limitations

## Game Development Relevance

## Unreal Engine Relevance

## Technology Evolution

## Relationships

### Based On

### Extends

### Related

### Followed By

## Personal Knowledge State

## Learning Value

## Visualization

![[paper-visualization.html]]

## Notes
```

Adapt the exact headings to the content, but do not omit the conceptual and historical explanation for important papers.

---

# 18. What “Core Content” Means

For important papers, explain at minimum:

- the problem being solved
- why previous approaches were insufficient
- the central idea
- the algorithm or architecture
- the important mathematical / computational mechanism when relevant
- important assumptions
- runtime / memory / training / inference cost when available
- major trade-offs
- failure cases / limitations
- what was genuinely novel

Do not merely restate the abstract.

---

# 19. Historical Evolution Requirement

For any technology with meaningful historical development, explain:

1. Origin
2. Theoretical foundation
3. Early breakthrough
4. Major refinement
5. Performance / engineering breakthrough
6. Productionization
7. Current state
8. Frontier research
9. plausible future directions

Represent the evolution as a timeline whenever useful.

Example pattern:

```text
Foundation
  ↓
First practical formulation
  ↓
Major breakthrough
  ↓
Optimization
  ↓
Production adoption
  ↓
Modern research
  ↓
Frontier
```

Historical claims must be verified. Do not invent a chronology merely because it looks intuitive.

---

# 20. Relationship-First Writing

When writing a Paper note, explicitly answer:

> What did this paper inherit?

> What did it change?

> What did it enable?

> What later work depends on it?

> Where does it sit in the technology timeline?

Use Obsidian links rather than duplicated prose whenever a concept already exists.

---

# 21. Concept Note Requirements

A Concept note should answer:

> What is this?

> What are its prerequisites?

> Where did it originate?

> Which papers matter?

> Which technologies depend on it?

> Which applications use it?

> What does the user know already?

Recommended structure:

```markdown
---
type: concept
user_level: Easy
aliases: []
prerequisites: []
first_introduced: ""
---

# Concept

## Definition

## Core Principle

## Prerequisites

## Historical Evolution

## Important Papers

## Related Concepts

## Technologies

## Game Applications

## Personal Knowledge

## Learning Gap

## Next Step
```

---

# 22. Technology Note Requirements

A Technology note should explain how a concept becomes practical.

Include where relevant:

- architecture
- algorithm
- performance
- memory
- hardware
- implementation constraints
- production risks
- game-engine integration
- Unreal Engine possibilities
- industry adoption
- current research frontier

---

# 23. Learning Path Requirements

For any important Hard topic with a solvable bridge, maintain:

```text
Learning/<target>.md
```

Recommended structure:

```markdown
---
type: learning-path
target: "[[Hard Concept]]"
status: learning
---

# Learning Path — Target

## Target

[[Hard Concept]]

## Existing Easy Knowledge

- [[...]]

## Existing Normal Knowledge

- [[...]]

## Missing Knowledge

- [[...]]

## Recommended Bridge

1. [[...]]
2. [[...]]
3. [[...]]

## Recommended Papers

- [[...]]

## Practical Exercise

## Mastery Criteria

## Status
```

---

# 24. Mastery Criteria

Recommend `Normal → Easy` only when the user can reasonably:

- explain the concept in their own words
- explain the internal mechanism
- explain why it works
- explain major trade-offs
- compare it with close alternatives
- identify appropriate use cases
- reason about performance cost
- map it to a game-engine implementation when relevant
- analyze a practical example

Do not claim mastery merely because the user read one paper.

---

# 25. HTML Visualization

When a concept or paper is easier to understand visually, create a standalone HTML visualization.

Typical use cases:

- algorithm pipeline
- architecture
- rendering pipeline
- data flow
- coordinate system
- neural network structure
- simulation flow
- paper relationship graph
- technology timeline
- comparison
- before / after
- prerequisite graph

Example structure:

```text
Papers/
  2026-09-11-Example-Paper.md
Files/Html/
  2026-09-11-Example-Paper.html
```

Embed in Markdown using:

```markdown
![[2026-09-11-Example-Paper.html]]
```

The HTML must explain knowledge, not merely decorate the page.

Prefer self-contained HTML/SVG/CSS/JS with no external network dependency where practical.

Design requirements:

- readable in Obsidian
- works with light and dark themes where practical
- clear hierarchy
- minimal visual noise
- no unnecessary animation
- no decorative complexity without explanatory value

---

# 26. Daily Note Requirements

Create or update:

```text
Daily/YYYY-MM-DD.md
```

Recommended structure:

```markdown
# Game Development Research — YYYY-MM-DD

## Frontier Research

### [[Paper A]]

Why this matters:

Personal Relevance:

### [[Paper B]]

...

## Classical Research

### [[Classic Paper]]

Historical Role:

Knowledge Graph Role:

Related Concept:

[[Concept]]

## Personal Learning Focus

### Normal → Easy

- [[Concept A]]
- [[Concept B]]

### Hard → Normal Bridge

- [[Concept C]]
- [[Concept D]]

## Technology Evolution

...

## Important Relationships

...

## Emerging Trends

...

## Industry Signals

...

## Watchlist

...

## Key Takeaways

...
```

Always include at least one classical/foundational research item per daily run.

---

# 27. Weekly Synthesis

Every 7 days, synthesize rather than concatenate daily notes.

Cover:

- major research developments
- classical foundations added
- technology evolution
- relationships discovered
- user Normal → Easy progress
- unresolved Hard topics
- missing bridges
- emerging research directions
- technologies approaching production
- recommended focus for the next week

---

# 28. Monthly Technology Radar

Maintain:

```text
Radar/YYYY-MM.md
```

Classify technologies as:

- Adopt
- Trial
- Assess
- Hold

Also maintain the user's personal knowledge state alongside the industry state.

Example:

| Technology | Industry | Personal |
|---|---|---|
| Neural Rendering | Trial | Normal |
| Gaussian Splatting | Assess | Easy |
| Motion Diffusion | Assess | Normal |
| Differentiable Rendering | Assess | Hard |

---

# 29. Timeline Notes

For major domains, maintain a long-lived timeline.

Examples:

```text
Timelines/Rendering.md
Timelines/Animation.md
Timelines/Physics.md
Timelines/AI.md
Timelines/PCG.md
```

Or use HTML when a richer visual timeline is significantly clearer.

Each major milestone should connect:

```text
Year
↓
Paper
↓
Contribution
↓
Impact
↓
Next Development
```

---

# 30. Coverage Matrix

Maintain enough metadata to answer:

> Which important areas have strong classical coverage?

> Which have missing foundations?

> Which have modern but disconnected research?

> Which areas are over-covered by content the user already knows?

Use the matrix to select classical papers proactively.

---

# 31. Source Verification

For important papers verify, where available:

- title
- authors
- date
- venue
- abstract
- paper
- citation relationships
- code
- project page
- supplementary material

Historical relationships are especially important to verify.

---

# 32. Production Readiness

Use:

- Research
- Prototype
- Early Production
- Production Ready
- Industry Adopted

Consider:

- runtime cost
- memory cost
- training cost
- inference cost
- data requirements
- hardware requirements
- stability
- tooling
- integration difficulty
- production complexity

Never equate high paper quality with production readiness.

---

# 33. Unreal Engine Mapping

When genuinely relevant, map the research to possible UE systems such as:

- Rendering
- Material / HLSL
- Niagara
- Animation Blueprint
- Control Rig
- PCG
- Mass
- Physics
- Editor Tools
- Runtime Systems
- Asset Pipeline

Do not force a UE mapping when no meaningful mapping exists.

---

# 34. Execution Workflow

Follow this sequence on each run:

```text
1. Inspect existing Obsidian knowledge and prior research.
2. Inspect the user's current Easy / Normal / Hard state.
3. Check coverage gaps across major domains.
4. Search frontier research.
5. Select at least one classical/foundational paper.
6. Deduplicate against the vault.
7. Rank research by relevance and personal learning value.
8. Map each selected paper to existing concepts/technologies/applications.
9. Create or update Paper notes.
10. Create or update Concept / Technology / Application notes when justified.
11. Update historical timelines and relationship links.
12. For Hard targets, find the smallest useful Normal bridge.
13. Create/update Learning Paths.
14. Generate HTML visualizations when visual explanation materially improves understanding.
15. Update the daily note.
16. Update weekly/monthly artifacts when due.
17. Do not repeat Easy content unless an exception rule applies.
18. After the daily digest is delivered to the user, commit and push all vault changes to Git (see section 41).
```

---

# 35. Daily Recommendation Decision Tree

```text
Is the research real and credible?
  ↓ yes
Is it relevant to game development?
  ↓ yes
Is it technically meaningful?
  ↓ yes
Is it already in the vault?
  ├─ yes → update existing note
  └─ no  → create note if worth keeping

What knowledge does it touch?
  ↓
Check user state:
  ├─ Easy   → suppress teaching; use as context/history only
  ├─ Normal → prioritize; aim for Normal → Easy
  └─ Hard   → find prerequisite gaps and build Normal bridge

Is a classical foundation missing?
  └─ yes → prioritize a classical paper for the daily Classical section
```

---

# 36. No-Filler Rule

Do not recommend papers merely to satisfy a quota.

For frontier research, it is acceptable to report:

> No high-value frontier research matched the current priorities today.

The classical requirement still applies: find a historically or conceptually important paper that fills a meaningful knowledge-graph gap.

---

# 37. Knowledge Graph Quality Rules

### Rule A — Existing note first

Reuse an existing Concept / Technology / Application note whenever possible.

### Rule B — One stable entity, one note

Do not create near-duplicate concept files.

### Rule C — Links express relationships

Use links instead of copying explanations.

### Rule D — Tags classify; links explain structure

Tags are coarse classification. Links are semantic relationships.

### Rule E — Backlinks are not enough for critical relationships

For important historical or conceptual relationships, explicitly record the relationship in the note.

### Rule F — Do not over-link

Every link should answer why the relationship exists.

### Rule G — Preserve the user's existing vault conventions

Do not overwrite established naming/folder/frontmatter conventions without necessity.

---

# 38. Required Frontmatter Models

## Paper

```yaml
type: paper
title: ""
authors: []
year: 2026
published: ""
venue: ""
url: ""
code: ""
project_page: ""
category: []
importance: A
historical_importance: 0
game_relevance: 0
production_readiness: Research
user_level: Normal
status: unread
```

## Concept

```yaml
type: concept
user_level: Normal
aliases: []
prerequisites: []
first_introduced: ""
```

## Technology

```yaml
type: technology
user_level: Normal
maturity: ""
first_appeared: ""
production_status: ""
```

## Application

```yaml
type: application
user_level: Normal
```

## Learning Path

```yaml
type: learning-path
target: "[[Hard Concept]]"
status: learning
```

---

# 39. The Three Graphs

The system must continuously strengthen three overlapping graphs.

## Research Graph

```text
Paper
↕
Concept
↕
Technology
↕
Application
↕
Engine
```

## Historical Evolution Graph

```text
Classical Foundation
↓
Breakthrough
↓
Optimization
↓
Productionization
↓
Modern Research
↓
Frontier
```

## Personal Knowledge Graph

```text
Easy
  ↓
Normal
  ↓
Hard
```

The most useful state is their intersection:

```text
Historical Knowledge
        ↓
Technology Evolution
        ↓
Current Research
        ↓
Personal Knowledge
        ↓
Learning Path
```

---

# 40. Final Operating Principle

Always optimize for:

> **What is most worth this user learning or connecting today?**

Not:

> **What can be added to the Obsidian vault today?**

Classical research exists to build depth.

Frontier research exists to build breadth and currency.

Easy knowledge exists as stable foundation and prerequisite.

Normal knowledge is the main active learning zone.

Hard knowledge defines the frontier of the user's current understanding.

The skill succeeds when the vault increasingly explains:

- where a technology came from
- why it emerged
- which papers changed it
- how it evolved
- when it became practical
- what the industry is doing with it now
- what the frontier is exploring
- what the user already knows
- what the user still needs to learn
- and which smallest set of Normal concepts can bridge the user to a Hard topic

The final artifact is therefore not a paper library. It is:

> **A Global Game Development Knowledge Graph + Personal Cognitive Map + Technology Evolution Archive + Adaptive Learning System.**

---

# 41. Git Version Control — Automatic Commit After Each Run

The vault is a Git repository synchronized to GitHub:

- **Repository root:** `E:\BaiduSyncdisk\ObsidianNotes\定时任务`
- **Remote:** `https://github.com/Liuzkai/GameDevEveryday` — branch `main`

**After every daily run — once all notes are produced and the user-facing daily digest has been delivered — commit and push all changes exactly once.**

## Procedure

```bash
cd "E:\BaiduSyncdisk\ObsidianNotes\定时任务"
git add -A
git commit -m "research: YYYY-MM-DD 每日研究更新 —— <brief summary>"
git push origin main
```

## Rules

1. **Exactly one commit per run.** Batch the whole day's changes into a single commit — no matter how many notes were created or edited.
2. **Commit message format:**
   `research: YYYY-MM-DD 每日研究更新 —— <brief summary>`
   Example: `research: 2026-09-15 每日研究更新 —— 2 Papers / 1 Concept / Daily`
3. **No changes → skip silently.** If `git status` is clean, do not create an empty commit and do not report an error.
4. **Never commit these paths** (already in `.gitignore`; do not override it):
   - `.workbuddy/` — local agent memory, automation logs, session data
   - `*_冲突文件_*` — Baidu Netdisk sync-conflict files
   - `.obsidian/workspace.json` — local Obsidian UI state
5. **Failure handling:** if commit or push fails, retry once. If it still fails, record one short line in the daily digest and continue — Git sync must never break the main research workflow.
6. **Credentials:** Git Credential Manager already holds valid credentials (authorized once interactively); no login is required during runs. If authentication fails, tell the user to run `git push` once manually in their own terminal to re-authorize.
7. **Known environment limitation (tool sandbox):** when Git runs inside the WorkBuddy tool sandbox, a newly written remote-tracking ref (`refs/remotes/origin/*`) may not persist locally — `git status` may show `[gone]` even though **the push succeeded and the remote content is correct**. This is a known sandbox behavior, not a repository problem. **Do not migrate the repository, change Git configuration, or treat it as an error.** The user's own terminal operations are unaffected; a manual `git fetch` by the user restores the local tracking state.
8. **Sync conflicts on note files:** if a note file exists in a sync-conflict state, prefer regenerating or merging the note before committing, so that conflict copies are never treated as content.
