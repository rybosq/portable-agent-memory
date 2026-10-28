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

请从指南第 7 节的八步安装流程开始，并按第 8 节为目标宿主制定配置与验收计划。**所有部署数值，以及验收项目、样本和通过阈值，必须由部署模型／部署人根据实际模型能力、底层实现和使用需求自主决定。** 第 9 节提供 DSH 接入位置映射，使用前须核对当前客户端。

Start with the eight-step integration process in Section 7, then define the target deployment's configuration and acceptance plan in Section 8. **The deploying model or person must independently choose all deployment values, acceptance checks, samples, and pass thresholds based on the actual model capabilities, host implementation, and intended use.** Section 9 maps DSH integration points; verify them against the target client.

> **仅供参考 / Reference only:** 历史配置、候选参数和 SolBot 用例已集中到文末附录。它们不是部署默认值或通用模型验收标准；参考项目测试通过不代表本次部署通过。Historical configurations, candidate values, and SolBot cases are grouped in the appendices. They are not deployment defaults or universal acceptance criteria; reference-project results do not establish success for another deployment.

## 许可证 / Licensing

- **文档 / Documentation:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE)。仓库内附完整许可证文本。The complete license text is included. [官方说明 / Official license page](https://creativecommons.org/licenses/by/4.0/).
- **未来源代码 / Future source code:** 计划采用 MIT License，并在源代码发布时附上对应许可证。Future source code is intended to use the MIT License, with the corresponding license included when that code is published.
