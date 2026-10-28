**AI 标识：本项目基于Codex开发，核心逻辑，代码及算法均由ChatGPT 6 Astra实现。**

AI disclosure: This project was developed using Codex. Its core logic, code, and algorithms were implemented by ChatGPT 6 Astra.

# Portable Agent Memory

面向 Chatbot、AI Companion 和 AI Agent 的可移植持续记忆架构与安装实施指南。当前仓库发布完整技术文档，涵盖宿主接入、数据结构、运行规则和参考部署记录。

A portable architecture and implementation guide for persistent memory in AI agents. This documentation repository covers host integration, data structures, operating rules, and reference deployment records for chatbots, AI companions, and agents.

## 核心能力 / Core capabilities

- **持久化与证据追溯 / Persistence and traceability** — 保存原始消息、来源引用、记忆版本与纠错记录。Preserve raw messages, evidence references, memory versions, and corrections.
- **增量整理与恢复 / Incremental consolidation and recovery** — 定义周期整理、有效空变更、有界重试与隔离批次恢复。Define scheduled consolidation, valid no-change results, bounded retries, and recovery of quarantined batches.
- **按需检索 / On-demand retrieval** — 通过作用域、分类、实体和别名筛选 SQL 条目，并按需回读原始证据。Retrieve scoped SQL entries by category, entity, and alias, with evidence lookup when needed.
- **上下文预算 / Context budgeting** — 计量完整请求，为各工作流设置预算，并协调容量整理与历史释放。Measure complete requests, budget each workflow, and coordinate consolidation and history release.
- **视觉记忆复用 / Visual memory reuse** — 按图片身份与作用域复用摘要，保留每次出现的语境，支持按需补看。Reuse image summaries within their scope, preserve each occurrence's context, and support reinspection.

## 安装指南 / Installation guide

**[阅读完整中文安装与实施指南 / Read the complete Chinese installation and implementation guide](docs/installation-guide.zh-CN.md)**

指南包含架构、字段映射、八步接入流程、配置示例、SolBot 运行记录及 DSH 宿主接入说明。请从第 8 节的安装和接入流程开始，并结合宿主实际能力选择参数。参考部署的已实现能力与后续候选方案在指南中分别标注。

The guide includes architecture, field mappings, an eight-step integration process, configuration examples, SolBot operating records, and DSH integration notes. Start with Section 8 and select parameters for your host's actual capabilities. The guide distinguishes implemented behavior from proposed changes.

> 推荐配置面向 500K–1M+ 可用上下文窗口；小于 500K 的模型需要单独配置并验证。The recommended settings target 500K–1M+ usable context windows; smaller models require a separately sized and validated configuration.

## 许可证 / Licensing

- **文档 / Documentation:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE)。仓库内附完整许可证文本。The complete license text is included. [官方说明 / Official license page](https://creativecommons.org/licenses/by/4.0/).
- **未来源代码 / Future source code:** 计划采用 MIT License，并在源代码发布时附上对应许可证。Future source code is intended to use the MIT License, with the corresponding license included when that code is published.
