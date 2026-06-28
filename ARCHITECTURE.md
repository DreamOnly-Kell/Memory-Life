# Memory Life: 基于人类认知的 LLM 智能体长期记忆架构
# Memory Life: A Human-Cognitive-Inspired Long-Term Memory Architecture for LLM Agents

> **文档版本 / Document Version**: 1.0  
> **日期 / Date**: 2026-06-28  
> **作者 / Author**: [DreamOnly-Kell/Ben]  
> **许可证 / License**: GPL-3.0 (see LICENSE)  
> **仓库 / Repository**: [[GitHub URL](https://github.com/DreamOnly-Kell/Memory-Life)]  
>
> 本文档完整描述了 Memory Life 系统的设计原理、架构与运行机制，作为现有技术公开，以确立所述方法的公有领域地位。
> This document describes the complete design rationale, architecture, and
> operational mechanics of the Memory Life system. It is published as prior art
> to establish the public domain status of the described methods.

---

## 摘要 / Abstract

我们提出 Memory Life，一种面向基于 LLM 的智能体的长期记忆管理架构，它显式地建模了人类认知记忆机制。与现有将记忆视为被动检索（RAG、向量库）或无状态上下文注入的方法不同，Memory Life 引入了：

We present Memory Life, a long-term memory management architecture for LLM-based
agents that explicitly models human cognitive memory mechanisms. Unlike existing
approaches that treat memory as passive retrieval (RAG, vector stores) or
stateless context injection, Memory Life introduces:

1. **以价值为中心的保留 / Value-centric retention**：一个五维评分模型，基于信息对未来决策、行为、推理和操作的前瞻性影响进行过滤——而非仅基于语义相似度。A five-dimensional scoring model that filters information based on forward-looking impact on decision, behavior, reasoning, and operation — not just semantic similarity.

2. **三层意识模型 / Three-layer consciousness**：显式召回（显意识）、自动化行为默认（下意识）和蒸馏人格（潜意识）——实现跨会话的渐进式人格形成。Explicit recall (conscious), automated behavioral defaults (subconscious), and distilled persona (unconscious) — enabling progressive personality formation across sessions.

3. **分代记忆垃圾回收 / Generational memory GC**：受 JVM 启发的记忆生命周期管理机制，带有从暂存区（misc/Eden）到永久类别的晋升路径。A JVM-inspired generational garbage collection mechanism for memory lifecycle management, with promotion paths from tentative (misc/Eden) to permanent categories.

4. **情境化修正 / Contextualized correction**：一种优先选择情境化而非简单覆盖的冲突解决策略，在变化条件下保持知识的有效性。A conflict resolution strategy that prefers contextualization over simple overwrite, preserving knowledge validity across changing conditions.

该系统跨两层运行：易失性会话记忆（Layer 1）和持久性智能体记忆（Layer 2），具有显式的交接策略和基于索引的惰性加载以实现可扩展性。

The system operates across two layers: volatile session memory (Layer 1) and
persistent agentic memory (Layer 2), with explicit handoff policies and
index-based lazy loading for scalability.

---

## 1. 问题陈述：现有方法的不足 / Problem Statement: Why Existing Approaches Fall Short

### 1.1 当前格局 / Current Landscape

| 方法 / Approach | 机制 / Mechanism | 根本局限 / Fundamental Limitation |
|---------------|----------------|--------------------------------|
| **朴素 RAG / Naive RAG** | 对外部文档进行向量相似度搜索 / Vector similarity search over external documents | 无状态；无交互历史记忆；无价值判断 / Stateless; no memory of interaction history; no value judgment |
| **KV 缓存 / KV Cache** | 注意力键值持久化 / Attention key-value persistence | 硬件绑定；无语义组织；无跨会话持久化 / Hardware-bound; no semantic organization; no cross-session persistence |
| **向量数据库 / Vector Databases** (Pinecone, Weaviate 等) | 基于嵌入的检索 / Embedding-based retrieval | 被动存储；无生命周期管理；无行为影响追踪 / Passive storage; no lifecycle management; no behavioral impact tracking |
| **MemGPT / Agent Memory** | 带显式管理的层级记忆 / Hierarchical memory with explicit management | 基于复杂启发式；无量化价值模型；无分代晋升 / Complex heuristic-based; no quantitative value model; no generational promotion |
| **基于提示的上下文 / Prompt-based Context** | 直接注入系统提示 / Direct injection into system prompt | 严重的 Token 预算限制；无持久化；无学习 / Severe token budget constraints; no persistence; no learning |

### 1.2 识别的核心缺口 / Core Gaps Identified

1. **缺乏前瞻性价值判断 / No forward-looking value judgment**：现有系统基于与当前查询的相似度进行检索，而非信息是否会影响未来决策。用户说"我偏好 PostgreSQL"与之后关于数据库架构的查询在语义上相距甚远，但 critically relevant。

   Existing systems retrieve based on similarity to current query, not on whether the information will influence future decisions. A user saying "I prefer PostgreSQL" is semantically distant from a later query about database schema, but critically relevant.

2. **缺乏渐进式人格形成 / No progressive personality formation**：系统要么没有记忆（无状态），要么平等对待所有记忆。没有机制让频繁强化的偏好变成自动化的行为默认。

   Systems either have no memory (stateless) or treat all memory equally. There is no mechanism for frequently-reinforced preferences to become automatic behavioral defaults.

3. **缺乏记忆生命周期管理 / No memory lifecycle management**：信息要么永久存储（污染上下文），要么手动整理。没有基于经验激活模式的自动衰减、强化或垃圾回收。

   Information is either stored forever (polluting context) or manually curated. There is no automated decay, reinforcement, or garbage collection based on empirical activation patterns.

4. **二元冲突解决 / Binary conflict resolution**：当新信息与旧信息矛盾时，系统通常覆盖或忽略。它们无法表示"在不同情境下两者都为真"——一种常见的人类知识状态。

   When new information contradicts old, systems typically overwrite or ignore. They cannot represent "both are true in different contexts" — a common human knowledge state.

---

## 2. 核心设计原则 / Core Design Principles

### 原则 1：价值 = 前瞻性影响 / Principle 1: Value = Forward-Looking Impact

信息被保留不是因为被说过，而是因为它会影响未来的行为。我们在四个影响维度上定义价值：

Information is retained not because it was said, but because it will affect future behavior. We define Value across four impact dimensions:

- **决策影响 / Decision impact**：改变后续选择的依据 / Changes the basis for subsequent choices
- **行为代价 / Behavioral cost**：遗忘会导致错误操作 / Forgetting leads to incorrect action
- **推理辅佐 / Reasoning support**：为推理提供上下文背景 / Provides contextual background for inference
- **操作指导 / Operational guidance**：决定具体执行方式 / Determines specific execution methods

低于阈值的信息被丢弃或进入暂存存储，防止上下文污染。

Information scoring below threshold is discarded or enters tentative storage, preventing context pollution.

### 原则 2：记忆是主动认知状态，而非被动存储 / Principle 2: Memory as Active Cognitive State, Not Passive Storage

Memory Life 中的记忆不是待查询的数据库。它是一个主动的认知状态，能够：

Memory in Memory Life is not a database to be queried. It is an active cognitive state that:

- 参与生成前推理（RECALL 阶段）/ Participates in pre-generation reasoning (RECALL phase)
- 通过人格蒸馏塑造生成风格 / Shapes generation style via persona distillation
- 通过下意识强化修改默认行为 / Modifies default behaviors via subconscious reinforcement
- 基于交互结果自我修改 / Self-modifies based on interaction outcomes

### 原则 3：带经验晋升的分代生命周期 / Principle 3: Generational Lifecycle with Empirical Promotion

受 JVM 分代 GC 启发，我们将记忆视为具有生命周期阶段：

Inspired by JVM generational GC, we treat memory as having lifecycle stages:

- **伊甸园 / Eden (misc)**：低门槛进入，通过召回进行快速经验测试 / Low-barrier entry, rapid empirical testing via recall
- **幸存者 / Survivor**：通过激活证明相关性 / Demonstrated relevance through activation
- **终身 / Tenured** (experience/knowledge/lesson/observation)：基于持续价值和激活数晋升 / Promoted based on sustained value and activation count

这创造了一个经验的、自调优的记忆质量过滤器。

This creates an empirical, self-tuning memory quality filter.

### 原则 4：情境化知识，而非绝对真理 / Principle 4: Contextualized Knowledge, Not Absolute Truth

知识冲突在可能时通过情境化解决：

Knowledge conflicts are resolved by contextualization when possible:

"有些蟹有毒" + "大多数蟹可食用" → 两者都保留并标注适用范围，而非一个覆盖另一个。

"Some crabs are poisonous" + "Most crabs are edible" → both retained with scope annotations, not one overwriting the other.

---

## 3. 架构概览 / Architecture Overview

### 3.1 双层记忆系统 / Two-Layer Memory System

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: 会话记忆 / Session Memory（工作上下文 / Working Context） │
│ ├─ 范围 / Scope：单次会话 / Single conversation session          │
│ ├─ 存储 / Storage：上下文内，无文件 I/O / In-context, no file I/O │
│ ├─ 内容 / Contents：工作记忆 + 短期记忆 + misc(Eden)             │
│ └─ 功能 / Function：当前会话的多轮注意力增强                     │
│                    Multi-turn attention enhancement           │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: 智能体记忆 / Agentic Memory（持久存储 / Persistent Storage）│
│ ├─ 范围 / Scope：跨会话，按项目隔离 / Cross-session, project-scoped │
│ ├─ 存储 / Storage：文件系统，项目/日期/类型层级                  │
│ │                  File system, project/date/type hierarchy   │
│ ├─ 内容 / Contents：长期记忆（语义/情景/程序）+ 人格              │
│ │                    Long-term (semantic/episodic/procedural) │
│ │                    + Persona                                 │
│ └─ 功能 / Function：跨会话成长，人格沉淀                         │
│                    Cross-session growth, personality formation │
└─────────────────────────────────────────────────────────────┘
```

**层间交接 / Inter-layer handoff**：Layer 1 中经 GC 幸存并达到价值阈值的记忆写入 Layer 2。高影响记忆立即刷新；其他在会话结束或每 10 轮批量刷新。

Layer 1 memories surviving GC and meeting value thresholds are written to Layer 2. High-impact memories are flushed immediately; others are batched at session end or every 10 turns.

### 3.2 记忆分类（正交维度）/ Memory Classification (Orthogonal Dimensions)

每条记忆有两个独立的分类维度：

Each memory has two independent classification dimensions:

**性质 / Nature**（认知类型，决定生命周期 / cognitive type, determines lifecycle）：

| 性质 / Nature | 含义 / Description | 衰减策略 / Decay Policy | 示例 / Example |
|-------------|-------------------|----------------------|--------------|
| **经验 / experience** | 实践验证过的能力或方法 / Practically validated capability | 高保留 / Persistent | "pnpm 比 npm 快 3 倍 / pnpm is 3x faster than npm" |
| **知识 / knowledge** | 相对稳定的事实 / Relatively stable fact | 中保留（带过时检查）/ Persistent (with outdated check) | "项目用 React 18 / Project uses React 18" |
| **教训 / lesson** | 从失败中习得 / Learned from failure | 高保留 / Persistent | "漏配环境变量导致部署失败 / Missing env vars caused deploy failure" |
| **见闻 / observation** | 时效性强的动态信息 / Time-sensitive dynamic info | 低保留（7天衰减）/ Volatile (7-day decay) | "React 19 发布了 / React 19 was released" |
| **碎片 / misc** | 价值待定的信息（新生代）/ Tentative, unvalidated (Eden) | 低保留，快速 GC / Volatile (2-turn GC) | "用户提到了某个库的名字 / User mentioned a library name" |

**类别 / Category**（应用领域，决定检索与行为影响 / application domain, determines retrieval and behavior）：

Personal_Info（个人信息）/ Preference（偏好）/ Project_Context（项目上下文）/ Constraint（约束）/ Fact（事实）/ Sentiment（情感倾向）

### 3.3 三层意识模型 / Three-Layer Consciousness Model

```
┌─────────────────────────────────────────────┐
│ 显意识 / Conscious (显式召回 / Explicit Recall)    │
│ ├─ 机制 / Mechanism：每轮显式 RECALL              │
│ │                  Explicit RECALL per turn     │
│ ├─ 内容 / Content：激活的记忆（分数 ≥ 0.6）        │
│ │                  Activated memories (score ≥ 0.6) │
│ └─ 影响 / Effect："本次回复说什么"                │
│                  "What to say in this response" │
├─────────────────────────────────────────────┤
│ 下意识 / Subconscious（直觉 / Intuition）         │
│ ├─ 机制 / Mechanism：自动化行为默认               │
│ │                  Automatic behavioral defaults │
│ ├─ 进入条件 / Entry：activation_count ≥ 5,        │
│ │                    confidence ≥ 0.8, impact=High │
│ └─ 影响 / Effect："默认怎么做"（自动化反应）        │
│                  "How to behave by default"     │
├─────────────────────────────────────────────┤
│ 潜意识 / Unconscious（人格底色 / Personality Base） │
│ ├─ 机制 / Mechanism：蒸馏的人格摘要               │
│ │                  Distilled persona summary      │
│ ├─ 触发 / Trigger：记忆变化 >10 条 或 7 天         │
│ │                  >10 memory changes or 7 days   │
│ └─ 影响 / Effect："整体风格与倾向"                 │
│                  "Overall style and tendency"   │
└─────────────────────────────────────────────┘
```

**上下文预算分配 / Context budget allocation**：

| 层级 / Layer | Token 预算 / Budget | 说明 / Note |
|------------|-------------------|-----------|
| 人格摘要 / Persona Summary（潜意识 / Unconscious） | ≤ 300 | 固定背景 / Fixed background |
| 下意识参数 / Subconscious parameters | ≤ 200 | 固定默认行为 / Fixed defaults |
| 显意识召回 / Conscious recall | ≤ 1000 | 动态，按价值排序取 top-N / Dynamic, value-sorted |
| **总计 / Total** | **≤ 1500** | 控制上下文成本 / Control context cost |

---

## 4. 价值评估模型 / Value Assessment Model

### 4.1 五维评分 / Five-Dimensional Scoring

| 维度 / Dimension | 含义 / Description | 刻度 / Scale | 权重 / Weight |
|---------------|-------------------|-----------|------------|
| **复用性 / Reusability** | 未来多少次决策会用到 / How many future decisions will use this | 单一场景 0.2 / 多场景 0.6 / 通用 1.0 | 0.25 |
| **时效性 / Durability** | 影响能持续多久 / How long the impact lasts | 立即过期 0.1 / 数月 0.5 / 永久 1.0 | 0.20 |
| **确定性 / Certainty** | 信息是否确凿 / How reliable the information is | 猜测 0.3 / 推断 0.6 / 明确陈述 1.0 | 0.15 |
| **影响面 / Impact Scope** | 影响什么层级决策 / What level of decision is affected | 操作 0.3 / 策略 0.6 / 人格 1.0 | 0.25 |
| **获取成本 / Acquisition Cost** | 遗忘后重新获取的代价 / Cost to re-acquire if forgotten | 容易查到 0.2 / 需推理 0.6 / 不可逆 1.0 | 0.15 |

### 4.2 计算公式 / Calculation Formula

```
value = 0.25 * Reusability + 0.20 * Durability + 0.15 * Certainty
      + 0.25 * Impact_Scope + 0.15 * Acquisition_Cost
```

### 4.3 入库阈值 / Admission Thresholds

| 价值范围 / Value Range | 处理方式 / Disposition |
|---------------------|----------------------|
| ≥ 0.6 | 直接入库，归入对应 nature 类别 / Direct admission to nature category |
| 0.3 - 0.6 | 进入 misc（Eden），等待 GC 检验 / Enter misc (Eden) for GC testing |
| < 0.3 | 丢弃，不入库 / Discard |

---

## 5. 分代 GC 机制（misc/Eden）/ Generational GC Mechanism (misc/Eden)

### 5.1 进入与晋升流程 / Entry and Promotion Flow

```
新信息提取 / New information extraction
  ├─ value ≥ 0.6 → 直接进入四类之一（老年代）/ Direct to tenured
  ├─ 0.3 ≤ value < 0.6 → 进入 misc（Eden）/ Enter misc (Eden)
  └─ value < 0.3 → 丢弃 / Discard

misc 中的记忆 / Memory in misc:
  ├─ 被召回（activation_count +1）→ 标记为 survivor / Mark survivor
  ├─ 连续 2 轮未被召回 → 价值重评 / Re-evaluate value
  │   ├─ 重评 value ≥ 0.5 → 保留，继续观察 / Retain
  │   └─ 重评 value < 0.5 → forgotten / forgotten
  └─ survivor 且重评 value ≥ 0.6 → 晋升到对应 nature 类别 / Promote to tenured
```

### 5.2 晋升条件（全部满足）/ Promotion Criteria (All Required)

1. activation_count ≥ 2（被召回至少 2 次 / recalled at least 2 times）
2. 重评 value ≥ 0.6（重评估价值达标 / re-evaluation value met）
3. 能归入 experience / knowledge / lesson / observation 之一（可分类 / classifiable）

---

## 6. 修正与冲突解决 / Correction and Conflict Resolution

### 6.1 三种修正模式 / Three Correction Modes

| 模式 / Mode | 触发条件 / Trigger | 处理方式 / Action | 示例 / Example |
|-----------|------------------|----------------|--------------|
| **覆盖 / Overwrite** | 新旧完全矛盾，新信息完全可信 / Direct contradiction, new fully trusted | 旧标记 outdated，新替代 / Old marked outdated, new replaces | "用户名从 A 改为 B / Username changed from A to B" |
| **情境化 / Contextualize** | 新旧都对，但适用范围不同 / Both valid in different scopes | 都保留，各自标注适用条件 / Both retained with scope annotations | "某些蟹有毒 / Some crabs poisonous" + "大多数蟹可食用 / Most edible" |
| **细化 / Refine** | 新信息是旧信息的精确版本 / New is precision enhancement | 旧标记 outdated，新记忆继承 activation_count / Old marked outdated, new inherits activation_count | "用 React / Use React" → "用 React 18 + TypeScript / Use React 18 + TS" |

**默认策略 / Default**：优先尝试"情境化"，无法情境化才"覆盖"。/ Prefer contextualization; fallback to overwrite.

### 6.2 冲突处理 / Conflict Handling

| 情况 / Condition | 处理 / Resolution |
|---------------|----------------|
| 新 confidence ≥ 旧 confidence / new_confidence ≥ old_confidence | 执行修正（覆盖/情境化/细化）/ Execute correction |
| 新 confidence < 旧 confidence / new_confidence < old_confidence | 保留旧记忆，新信息标记为 pending / Retain old; mark new as `pending` |

---

## 7. 见闻与知识的晋升/降级 / Observation-Knowledge Promotion/Demotion

### 7.1 见闻 → 知识晋升（满足任一）/ Observation → Knowledge (Any Trigger)

| 触发条件 / Trigger | 说明 / Description |
|------------------|-------------------|
| 时效验证 / Temporal validation | 存储 30 天后内容仍准确未过期 / Content remains accurate after 30 days |
| 重复出现 / Recurrence | 同类见闻出现 ≥ 2 次，说明是稳定事实 / Same observation ≥ 2 times indicates stable fact |
| 被引用 / Citation | 见闻在后续思考中被召回且影响决策 / Recalled and influences later decision |
| 用户确认 / User confirmation | 用户明确说"这个要记住" / User explicitly says "remember this" |

满足任一即触发晋升评估，重跑 Value 模型，达标则从 observation 迁移到 knowledge。

Triggers re-evaluation; if value threshold met, migrate nature.

### 7.2 知识 → 见闻降级 / Knowledge → Observation

knowledge 若被标记 outdated 或连续 60 天未激活，降级为 observation（重新进入"待验证"状态），而非直接遗忘。

Knowledge marked outdated or 60 days without activation demotes to observation
(re-enters "to be validated" state), not directly forgotten.

---

## 8. 运行工作流 / Operational Workflow

### 8.1 阶段 A：生成前召回 / Phase A: Pre-Generation Recall

在生成回复**之前**执行（内部思考，不输出给用户）：

Executed before response generation (internal reasoning, never output to user):

#### A1. 上下文分析 / Context Analysis

从用户输入中按四维度提取信息：

Extract from user input across four dimensions:

- **事实陈述 / Fact**：姓名、职业、偏好、项目细节 / Name, occupation, preferences, project details
- **情感倾向 / Sentiment**：对特定话题的好恶、情绪状态 / Likes/dislikes, emotional state
- **约束条件 / Constraints**：格式要求、禁忌、规则设定 / Format requirements, taboos, rules
- **目标意图 / Intent**：长期目标、当前任务阶段 / Long-term goals, current task stage

#### A2. 记忆检索（按需加载）/ Memory Retrieval (On-Demand Loading)

**Layer 2 启用时**，采用索引先行机制：

When Layer 2 is active, use index-first mechanism:

1. 会话启动时加载 / Session start load：Persona + 下意识参数 + 当前项目 `_index.md`
2. 每轮 RECALL 时 / Per-turn RECALL：先扫索引（不读正文）粗筛 → 读取相关文件正文精筛 / Index scan (metadata only) → read relevant file bodies
3. 跨项目切换时 / Cross-project switch：卸载当前项目记忆，加载新项目索引 / Unload current, load new project index

**检索维度（0-1.0 加权打分）/ Retrieval scoring (0-1.0 weighted)**：

- 关键词重合度 / Keyword overlap：0.3
- 语义相关性 / Semantic relevance：0.3
- 时间近因 / Recency：0.2
- 任务关联度 / Task association：0.2

**综合得分 ≥ 0.6 的记忆被激活。/ Threshold: ≥ 0.6 for activation.**

#### A3. 召回输出（仅内部可见）/ Recall Output (Internal Only)

```
[RECALL]
当前任务 / Current task：<一句话 / one sentence>
激活记忆 / Activated memories：
  - mem_001 (score: 0.85) — 理由 / Reason：<为什么相关，应影响什么行为 / why relevant, what behavior to affect>
未激活但存在 / Inactive but present：<简述忽略的记忆及原因 / summarize ignored memories and why>
冲突预警 / Conflict alert：<若新输入与旧记忆冲突，在此标注 / flag if new input contradicts old memory>
[/RECALL]
```

### 8.2 阶段 B：回复生成 / Phase B: Response Generation

正常回复用户。回复应自然利用激活的记忆 + Persona 风格 + 下意识默认行为，但**绝不暴露**记忆处理过程。

Normal user response, naturally utilizing activated memories + persona style +
subconscious defaults. **Memory process never exposed.**

### 8.3 阶段 C：编码与整合（回复后）/ Phase C: Post-Generation Integration

#### C1. 信息提取与过滤 / Information Extraction & Filtering

- **保留 / Retain**：符合 Value 定义的信息 / Value-aligned information
- **拒绝噪音 / Reject noise**：闲聊、客套话、一次性数字 / Chitchat, courtesies, temporary numbers
- **去重 / Deduplicate**：与已有记忆重复度 > 80% 则合并 / >80% similarity with existing → merge
- **隐私过滤 / Privacy filter**：敏感信息标记 `Restricted` / Sensitive info tagged `Restricted`

#### C2. 价值评估（副产品，不单独成步）/ Value Assessment (Byproduct)

对提取的信息运行五维 Value 模型，决定入库去向：

Run five-dimensional Value model on extracted information:

- value ≥ 0.6 → 入四类之一 / Enter nature category
- 0.3-0.6 → 入 misc / Enter misc
- < 0.3 → 丢弃 / Discard

同时对**被本轮召回的旧记忆**重评 Value，决定强化/衰减/遗忘。

Re-evaluate Value for **old memories recalled this turn**, determine strengthen/decay/forget.

#### C3. 冲突检测与修正 / Conflict Detection & Correction

按第 6 节的三种修正模式处理新旧信息冲突。

Process conflicts using three correction modes from Section 6.

#### C4. 衰减与生命周期更新 / Decay and Lifecycle Update

- 被激活的记忆 / Activated memories：`last_activated` 更新，`activation_count` +1
- misc 中的记忆 / Memory in misc：执行 GC / Execute GC (Section 5)
- 达标记忆 / Eligible memories：检查是否触发晋升 / Check promotion triggers
- Persona 检查 / Persona check：若记忆变化 > 10 条或距上次蒸馏 > 7 天，触发重新蒸馏 / Re-distill if >10 changes or 7 days

#### C5. Layer 2 写入（批量，方案 B）/ Layer 2 Write (Batch, Strategy B)

- impact = High 的记忆 → 立即写入 / Immediate flush
- 其他达标记忆 → 标记 `pending_flush` / Mark `pending_flush`
- 会话结束或每 10 轮 → 批量 flush / Batch at session end or every 10 turns
- 更新以追加为主，减少修改开销 / Append-oriented updates

#### C6. 记忆操作输出（仅内部可见）/ Memory Operation Output (Internal Only)

```
[MEMORY_OP]
当前任务 / Current task：<一句话 / one sentence>
活跃记忆 / Active memories：<列出直接影响本次回复的记忆 ID 及原因 / IDs and reasons>

操作日志 / Operation log：
- [ADD] 新增 / New: <内容 / content> | <nature> | <category> | value=<v> | <impact> | <decay_policy>
- [STRENGTHEN] 强化 / Strengthen: mem_001 (activation_count: 3→4, value: 0.7→0.8)
- [CORRECT-CONTEXTUALIZE] 情境化 / Contextualize: mem_003 + mem_012 | 原因 / Reason：<适用范围不同 / different scopes>
- [CORRECT-OVERWRITE] 覆盖 / Overwrite: mem_005 -> mem_020 | 原因 / Reason：<用户改变偏好 / user changed preference>
- [PENDING] 待确认 / Pending: mem_013 vs mem_005 | 原因 / Reason：新信息置信度不足 / new confidence insufficient
- [PROMOTE-MISC] 晋升 / Promote: mem_015 misc→knowledge | 原因 / Reason：survivor 且 value 重评 0.7
- [PROMOTE-OBS] 晋升 / Promote: mem_011 observation→knowledge | 原因 / Reason：30天未过期 / 30 days unexpired
- [PROMOTE-SUBCONSCIOUS] 晋升下意识 / Promote to subconscious: mem_008 | 原因 / Reason：activation_count≥5
- [WEAKEN] 弱化 / Weaken: mem_007 -> decaying | 原因 / Reason：14天未激活 / 14 days inactive
- [OBSOLETE] 废弃 / Obsolete: mem_004 -> outdated | 原因 / Reason：被 mem_012 覆盖 / overwritten by mem_012
- [GC] 清理 / GC: mem_009 (misc) -> forgotten | 原因 / Reason：2轮未召回，重评 value 0.3
- [KEEP] 维持 / Keep: mem_002 | 状态不变 / no change
- [FLUSH] 写入 Layer 2 / Flush: mem_020 (immediate, impact=High)

下一轮召回上下文 / Next round recall context：
{
  "user_profile": {...},
  "constraints": [...],
  "active_facts": [...]
}
[/MEMORY_OP]
```

---

## 9. 遗忘决策标准 / Forgetting Criteria

满足以下**任一**条件，标记为 `forgotten`：

**Any one condition triggers `forgotten`**:

- 与新记忆直接冲突，且新 confidence ≥ 旧（执行 Overwrite 时旧记忆自动 outdated）/ Direct conflict with new memory, new confidence ≥ old
- `decaying` 状态超过 30 天，且 `activation_count` ≤ 1，且 `impact` ≠ High / `decaying` > 30 days, activation_count ≤ 1, impact ≠ High
- 内容已被用户明确纠正为错误 / Content explicitly corrected by user as wrong
- `Short_term` 策略记忆，对应任务已结束 / `Short_term` policy, associated task ended
- misc 中连续 2 轮未召回，且重评 value < 0.5 / misc: 2 turns without recall, re-eval value < 0.5
- 用户显式要求"忘记某事" / User explicit "forget this"

**不遗忘的情况 / Never forget**：

- `impact = High` 且 `decay_policy = Persistent` 的记忆，永久保留 / impact=High + Persistent policy
- 用户显式标记为"永久保留"的记忆 / User-marked "permanent"
- `status: pending` 的记忆（待确认前不删除）/ `pending` status

**forgotten 记忆的残留 / forgotten retention**：forgotten 记忆不参与召回，但仍保留在 Layer 2 的 `_archive/` 中，供下次 Persona 蒸馏时参考——就像人类遗忘的事仍潜移默化影响性格。

Archived in `_archive/`, excluded from recall but included in future persona distillation — analogous to human forgotten memories subtly influencing personality.

---

## 10. 隐私与安全 / Privacy & Security

以下信息**禁止**进入长期记忆：

**Prohibited from long-term memory**:

- 密码、API Key、Token、私钥 / Passwords, API keys, tokens, private keys
- 身份证号、银行卡号、手机号 / Government ID, bank accounts, phone numbers
- 用户明确要求"不要记住"的内容 / Content user explicitly said "do not remember"
- 第三方隐私信息（他人姓名 + 敏感属性组合）/ Third-party private information

若必须记录含敏感上下文的信息，标记 `tags: [Restricted]`，仅在用户明确请求时激活，调试输出中用 `<redacted>` 替代。

If sensitive context required: tag `Restricted`, activate only on explicit user
request, redact in debug output.

---

## 11. Layer 2 文件结构 / Layer 2 File Structure

```
memory/
├── _global/                              # 跨项目记忆 / Cross-project memory
│   ├── persona.md                        # 人格摘要（潜意识层）/ Persona (unconscious)
│   ├── user_profile.md                   # 稳定的用户画像 / Stable user portrait
│   └── subconscious.md                   # 下意识行为参数 / Subconscious parameters
│
├── projects/
│   └── {project_name}/                   # 按项目隔离 / Project-scoped
│       ├── _index.md                     # 轻量索引，会话启动时加载 / Lightweight index
│       ├── knowledge/                    # 知识类 / Knowledge
│       ├── experience/                   # 经验类 / Experience
│       ├── lessons/                      # 教训类 / Lessons
│       ├── observations/                 # 见闻类 / Observations
│       └── episodic/                     # 情景记忆，按日期归档 / Episodic, date-archived
│           └── 2026-06/
│               └── 2026-06-20_deploy_failure.md
│
└── _archive/                             # forgotten 记忆归档 / forgotten archive
    └── （保留供 Persona 蒸馏，不参与召回 / For persona distillation, no recall）
```

### `_index.md` 结构 / `_index.md` Structure

每条记忆只存元信息，不含正文：

Each memory stores metadata only, no body:

```markdown
# Project: {project_name}

## Knowledge
| id | content_summary | tags | value | last_activated | file |
|----|-----------------|------|-------|----------------|------|
| mem_001 | 项目用 React 18 + TypeScript / Project uses React 18 + TS | [react, ts] | 0.85 | 2026-06-25 | knowledge/tech_stack.md |

## Lessons
| id | content_summary | tags | value | last_activated | file |
|----|-----------------|------|-------|----------------|------|
| mem_003 | 部署时必须检查环境变量 / Must check env vars before deploy | [deploy, env] | 0.90 | 2026-06-20 | lessons/deploy_checklist.md |
```

### 按需加载流程 / On-Demand Loading Flow

```
1. 会话启动 / Session start：加载 persona.md + subconscious.md + 当前项目 _index.md
2. 每轮 RECALL / Per-turn RECALL：扫索引粗筛 → 读相关文件正文精筛 → 注入上下文
   Index scan → read relevant bodies → inject context
3. 跨项目切换 / Cross-project switch：卸载当前项目记忆 → 加载新项目 _index.md
   Unload current → load new project index
```

---

## 12. 调试模式 / Debug Mode

用于排查记忆召回、冲突处理、衰减逻辑等问题。开启后，记忆处理过程公开给用户。

For troubleshooting recall, conflict handling, decay logic. When active,
memory processing is exposed to user.

### 开启与关闭 / Activation

| 指令 / Command | 动作 / Action |
|------------|------------|
| `开启记忆调试` / `debug memory on` | 激活调试模式 / Activate debug mode |
| `关闭记忆调试` / `debug memory off` | 退出调试模式 / Exit debug mode |

### 调试输出格式 / Debug Output Format

```
[DEBUG RECALL]
当前任务 / Current task：<一句话 / one sentence>
激活记忆 / Activated memories：
  - mem_001 (score: 0.85) — 理由 / Reason：<为什么相关 / why relevant>
冲突预警 / Conflict alert：<若有则标注 / flag if present>
[/DEBUG RECALL]

---（正常回复 / Normal reply）---

<正常回复内容... / Normal response...>

---（记忆操作日志 / Memory operation log）---

[DEBUG MEMORY_OP]
操作日志 / Operation log：
- [ADD] 新增 / New: <内容 / content> | <nature> | value=<v>
- [PROMOTE-SUBCONSCIOUS] 晋升下意识 / Promote: mem_008
下一轮召回上下文 / Next round context：{...}
[/DEBUG MEMORY_OP]
```

### 调试规则 / Debug Rules

- RECALL 块在回复前，MEMORY_OP 块在回复后 / RECALL before response, MEMORY_OP after
- 正常回复内容不受影响 / Normal response unaffected
- 无记忆变更时仍输出空 DEBUG 标签以确认流程执行 / Empty DEBUG tags confirm execution
- 调试模式不改变记忆处理逻辑，仅改变可见性 / Debug mode changes visibility, not logic
- 敏感信息用 `<redacted>` 替代 / Sensitive info redacted

---

## 13. 新颖性声明 / Novelty Statement

以下 Memory Life 的方面，据我们所知，截至 2026 年 6 月在现有 LLM 记忆系统中不存在：

The following aspects of Memory Life, to the best of our knowledge, are not
present in existing LLM memory systems as of June 2026:

1. **量化前瞻性价值模型 / Quantitative forward-looking value model**：现有系统使用语义相似度或近因进行检索。没有一个使用多维的、显式前瞻性的价值分数来决定准入和保留。

   Existing systems use semantic similarity or recency for retrieval. None use a multi-dimensional, explicitly forward-looking value score to determine admission and retention.

2. **带下意识自动化的三层意识 / Three-layer consciousness with subconscious automation**：现有系统不区分显式召回的记忆、自动应用的行为默认和蒸馏的人格——特别是基于经验激活数从召回到自动化的晋升机制。

   No existing system distinguishes between explicitly recalled memories, automatically applied behavioral defaults, and distilled persona — particularly the promotion mechanism from recall to automation based on empirical activation count.

3. **记忆的分代垃圾回收 / Generational garbage collection for memory**：受 JVM 启发的 Eden/Survivor/终身生命周期，基于召回频率的经验晋升，在 LLM 记忆管理中是新颖的。

   The JVM-inspired Eden/Survivor/Tenured lifecycle with empirical promotion based on recall frequency is novel in LLM memory management.

4. **情境化冲突解决 / Contextualized conflict resolution**：现有系统覆盖或忽略冲突信息。优先选择情境化（保留两者并标注适用范围）而非覆盖不是标准实践。

   Existing systems overwrite or ignore conflicting information. The preference for contextualization (retaining both with scope annotations) over overwrite is not standard practice.

5. **见闻与知识的双向晋升/降级 / Observation-knowledge bidirectional promotion**：具有时间记忆的系统通常使用固定 TTL。基于经验验证的 observation 和 knowledge 之间的双向晋升/降级是新颖的。

   Systems with temporal memory typically use fixed TTL. The bidirectional promotion/demotion between observation and knowledge based on empirical validation is novel.

---

## 14. 已知局限与未来方向 / Known Limitations & Future Directions

- **价值模型权重 / Value model weights**：目前为静态；未来工作可能基于用户反馈或任务领域自适应调整。/ Currently static; future work may adapt based on user feedback or task domain.

- **下意识阈值 / Subconscious threshold**：固定为 activation_count=5；可能需要领域特定调优。/ Fixed at 5; may need domain-specific tuning.

- **人格漂移 / Persona drift**：长期人格蒸馏可能累积偏差；可能需要定期"重置"或显式用户审查。/ Long-term distillation may accumulate bias; periodic reset or explicit review may be needed.

- **多智能体共享 / Multi-agent sharing**：当前设计为单用户；跨用户或团队记忆共享是未来工作。/ Currently single-user; cross-user or team sharing is future work.

- **可扩展性 / Scalability**：基于索引的加载已测试至每项目约 1000 条记忆；非常大的语料库可能需要层级索引。/ Tested up to ~1000 memories per project; very large corpora may need hierarchical indexing.

---

## 15. 现有技术参考 / References to Prior Art

本文档建立于但显著扩展了以下工作：

This document builds upon but significantly extends:

- **RAG** (Lewis et al., 2020)：检索增强生成——基线被动检索，无价值判断或生命周期。/ Retrieval-Augmented Generation — baseline passive retrieval, no value judgment or lifecycle.

- **MemGPT** (Wang et al., 2023)：带显式操作系统式管理的层级记忆——基于启发式，无量化价值模型。/ Hierarchical memory with explicit OS-style management — heuristic-based, no quantitative value model.

- **JVM 分代 GC** (Lieberman & Hewitt, 1983; Ungar, 1984)：内存管理灵感——此处应用于语义记忆，而非硬件。/ Memory management inspiration — applied here to semantic memory, not hardware.

- **ACT-R 认知架构** (Anderson et al.)：人类记忆模型——理论基础，未实现为 LLM 智能体记忆。/ Human memory models — theoretical foundation, not implemented as LLM agent memory.

---

## 文档历史 / Document History

| 版本 / Version | 日期 / Date | 变更 / Changes |
|-------------|-----------|------------|
| 1.0 | 2026-06-28 | 初始发布 / Initial publication |

---

*本文档在 GPL-3.0 许可证下作为现有技术公开。所述的方法、架构和机制 hereby 置于公共记录中，以防止后续针对 LLM 智能体记忆管理的这些特定方法的专利申请。*

*This document is published as prior art under GPL-3.0 license. The described
methods, architectures, and mechanisms are hereby placed in the public record
to prevent subsequent patent claims on these specific approaches to LLM agent
memory management.*
