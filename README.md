# Memory Life

> 基于人类认知的 LLM 智能体长期记忆架构
> A Human-Cognitive-Inspired Long-Term Memory Architecture for LLM Agents

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

## 这是什么 / What is this?

Memory Life 是一种面向基于 LLM 的智能体的长期记忆系统，它建模人类认知记忆机制——不是作为被动存储，而是作为主动认知状态，塑造行为、形成人格并自我管理其生命周期。

Memory Life is a long-term memory system for LLM-based agents that models
human cognitive memory mechanisms — not as passive storage, but as active
cognitive state that shapes behavior, forms personality, and self-manages
its own lifecycle.

**核心能力 / Key capabilities**：
- **以价值为中心的保留 / Value-centric retention**：只有具有前瞻性影响的信息被保留；闲聊和噪音被自动过滤。/ Only information with forward-looking impact is retained; chitchat and noise are automatically filtered.
- **三层意识模型 / Three-layer consciousness**：显式召回 → 自动化行为默认 → 蒸馏人格，实现跨会话的渐进式人格形成。/ Explicit recall → automated behavioral defaults → distilled persona, enabling progressive personality formation across sessions.
- **分代记忆垃圾回收 / Generational memory GC**：受 JVM 启发的记忆生命周期管理，带有从暂存区到永久类别的经验晋升。/ JVM-inspired memory lifecycle management with empirical promotion from tentative to permanent categories.
- **情境化知识 / Contextualized knowledge**：冲突通过范围标注解决，而非破坏性覆盖。/ Conflicts resolved by scope annotation, not destructive overwrite.

## 架构 / Architecture

详见 [ARCHITECTURE.md](ARCHITECTURE.md) 了解完整的技术设计，包括新颖的价值评估模型、意识层、GC 机制和运行工作流。

See [ARCHITECTURE.md](ARCHITECTURE.md) for the complete technical design,
including the novel Value assessment model, consciousness layers, GC
mechanism, and operational workflow.

## 实现 / Implementation

本仓库包含 Memory Life 架构的第一版实现（v0.1），作为参考和起点提供。

This repository contains the first implementation (v0.1) of the Memory Life
architecture, provided as reference and starting point.

## 许可证 / License

GPL-3.0。详见 [LICENSE](LICENSE)。

GPL-3.0. See [LICENSE](LICENSE).

## 贡献 / Contributing

详见 [CONTRIBUTING.md](CONTRIBUTING.md)。

See [CONTRIBUTING.md](CONTRIBUTING.md).
