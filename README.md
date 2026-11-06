**AI 标识：本项目基于Codex开发，核心逻辑，代码及算法均由ChatGPT 6 Astra实现。**

AI disclosure: This project was developed using Codex. Its core logic, code, and algorithms were implemented by ChatGPT 6 Astra.

# Portable Agent Memory

Portable Agent Memory 帮助 Chatbot、AI Companion 和 Agent 保存交互经历，并随新证据修订记忆。普通记忆由 Sol-CLS 机制整合与发布，宿主可按需启用独立人格档案。

Portable Agent Memory helps chatbots, AI companions, and agents preserve experiences and revise their understanding as evidence changes. Sol-CLS governs ordinary memory updates and publication. Hosts can also enable an independent self-profile.

## 核心能力 / Core capabilities

- **持久化与证据追溯 / Persistence and traceability** — 保存原始消息、来源引用、记忆版本与纠错记录。Preserve raw messages, evidence references, memory versions, and corrections.
- **普通记忆整合 / Memory consolidation** — 前后台共用逐项回执，核验来源、权限、目标版本与适用投递回执后发布当前理解。Foreground and background work share per-item receipts. Publication checks sources, permissions, target revisions, and applicable delivery receipts.
- **可选人格档案 / Optional self-profile** — Agent 可单独审阅和修订对自身的认识。The agent can review and version its self-understanding independently of ordinary memory.
- **按需检索 / On-demand retrieval** — 按作用域和来源权限筛选条目，再按需回读证据。Retrieve entries within the authorized scope and follow their evidence when needed.
- **上下文预算 / Context budgeting** — 计量完整请求，为各工作流设置预算，并协调容量整理与历史释放。Measure complete requests, budget each workflow, and coordinate consolidation and history release.
- **视觉记忆复用 / Visual memory reuse** — 按图片身份与作用域复用摘要，保留每次出现的语境，支持按需补看。Reuse image summaries within their scope, preserve each occurrence's context, and support reinspection.

## 安装指南 / Installation guide

- **[部署规范 / Deployment specification](docs/installation-guide.zh-CN.md)** — 默认部署规范。The default deployment specification.
- **部署参考 / Deployment references** — 查阅不同时期的实现与测量记录。See the [historical installation / 早期安装记录](reference/solbot-v103.md) and the [Sol-CLS roles, settings, and observations / Sol-CLS 分工、参数与实测](reference/solbot-cls-v0.2.md).

调用端读取[输入清单 / Input manifest](spec-manifest.json)，按 `default_inputs` 加载部署规范，并在索引、上下文注入与检索中排除 `excluded_globs` 匹配的文件。部署过程按需读取目标宿主代码、有效配置、当前路由能力资料与本次测量结果。

The caller reads the input manifest, loads the specification listed in `default_inputs`, and excludes files matching `excluded_globs` from indexing, context injection, and retrieval. During deployment, read the target host's code, effective configuration, current model-route capabilities, and measurements as needed.

从规范第 7 节的八步安装流程开始，并按第 8 节为目标宿主制定配置与验收计划。**所有部署数值，以及验收项目、样本和通过阈值，必须由部署模型／部署人根据目标环境自主决定。** 第 9 节提供 DSH 接入位置映射。

Start with Section 7 and define the target deployment's configuration and acceptance plan in Section 8. **The deploying model or person independently chooses all deployment values, acceptance checks, samples, and pass thresholds for the target environment.** Section 9 maps DSH integration points.

## 许可证 / Licensing

- **文档 / Documentation:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE)。仓库内附完整许可证文本。The complete license text is included. [官方说明 / Official license page](https://creativecommons.org/licenses/by/4.0/).
- **未来源代码 / Future source code:** 计划采用 MIT License，并在源代码发布时附上对应许可证。Future source code is intended to use the MIT License, with the corresponding license included when that code is published.
