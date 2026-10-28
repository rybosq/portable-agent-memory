# Portable Agent Memory — Installation & Implementation Guide

修订日期：2026-10-28。参考部署：大肥龙 Sol / SolBot V1.03，包标识 `solbot-v103-4e36ac820f5d`。

本指南用于给 Chatbot、AI Companion 和 Agent 接入持续记忆。程序负责存储、预算、调度、校验和提交；LLM 负责语义理解、信息提炼与候选记忆生成。

第 2 节、第 10.2 节和第 11 节记录 SolBot V1.03 的配置、运行数据与现有处理路径；第 10.3 节列出 SolBot 后续候选参数。通用章节提供架构、字段、调度规则与推荐区间，宿主按自己的模型路由和负载选择具体值。

> **Warning — 模型窗口兼容性：** 本指南的推荐配置面向 **500K–1M+ 可用上下文窗口**。低于 500K 的模型（例如 200K 级窗口）不建议直接套用这些分配；应另设小窗口配置，重新计算工作区、辅助输入与输出预留，缩小增量批次和检索量，并测量整理频率与开销。

本文自包含关键实现逻辑、参数口径和参考用例。外部宿主依据自身代码库完成字段映射与接入。

## 1. 目标与职责

### 1.1 目标

- 持久保存原始消息、来源、时间、引用关系和附件索引。
- 让近期对话、正在进行的话题和相关旧记忆共同服务当前回复。
- 按时间和上下文压力处理增量内容，让活跃上下文保持有界。
- 为事实、纠错、互动事件和视觉摘要保存可回溯的证据。
- 复用同图摘要，保留新配文、新问题和新的互动意义。
- 为整理任务定义成功、空变更、重试和终止状态。

### 1.2 程序与 LLM 的分工

| 工作 | 程序执行 | LLM 执行 |
| --- | --- | --- |
| 消息进入 | 身份映射、幂等落盘、建立引用 | 按需理解当前消息 |
| 图片复用 | 计算特征 ID、查作用域缓存、选择像素或摘要 | 理解首次图片、生成摘要、提出补看请求 |
| 记忆检索 | 查询索引、过滤作用域、排序和限额 | 结合结果回答；必要时提出补充查询 |
| 周期整理 | 固定批次、准备证据、调度请求 | 提炼新增事实、事件、话题和候选更新 |
| 提交 | 校验结构和引用、检查版本、持久化、推进游标 | 返回结构化候选结果 |
| 容量管理 | 最终请求计量、分批、排队、按持久化状态和释放规则移出旧历史 | 生成有证据的工作摘要 |
| 失败处理 | 记录原因、限制尝试次数、进入明确终态 | 在定向修复请求中修正具体字段 |

### 1.3 数据流约束

1. 原始消息先持久化，再进入后续处理。
2. 语义记忆保存来源引用；纠错生成新版本。
3. 活跃上下文的释放依据原文持久化状态、批次终态与当前输入需要；语义覆盖和上下文驻留分别记录。
4. 空变更是有效整理结果，保留仍有效的记忆并记录批次完成。
5. 每次模型请求独立计量完整输入及输出预留。
6. 宿主已有的 Session、数据库、定时器和 Compaction 作为接入基础。

## 2. SolBot V1.03：实际配置与运行情况

### 2.1 模型、预算与工作流

| 项目 | 实际配置或实现 |
| --- | --- |
| 主对话 | `gpt-6.1-sol`，Medium，订阅授权路径 |
| 主工作区配置 | `context_workspace_tokens = 512000` |
| 辅助输入配置 | `context_auxiliary_tokens = 128000` |
| 输出预留 | `context_output_reserve_tokens = 128000` |
| 宿主总请求预算 | `context_total_tokens = 768000` |
| 容量预整理通知 | 预计总量达到总预算的 70%，即 537600 tokens |
| 已覆盖历史释放水位 | 预计总量达到总预算的 85%，即 652800 tokens |
| 整理模型 | 继承当前主配置，实际为 Medium |
| 整理证据批次 | 使用 `context_workspace_tokens`；旧档案另使用辅助区额度 |
| 整理独立 96K 配额 | 实现状态：待加入参数实验 |
| 本地历史检索 | 档案文本匹配、SQL `LIKE`，每个查询最多返回 12 条 |
| 公开知识查询 | 缓存、稳定知识的百科接口、Low 原生网页查询 |
| 搜索独立 72K 配额 | 宿主自组装搜索整理请求的候选值；当前原生网页内部上下文由服务端管理 |
| Reserve 共享账本 | 实现状态：未建立；现有任务由独立请求路径与本地工作线程调度 |

这里的 768000 是宿主配置的请求预算。模型服务窗口由所用路由的模型元数据和服务能力确定。主工作区、辅助输入与输出预留分别参与本地预算检查。

SolBot V1.03 的本地计量使用 `o200k_base`，为每张图片固定预留 16384 tokens，辅助区额外加入 4096 tokens 的结构余量。这些数值属于该部署的静态估算配置。服务返回的 `usage` 保存实际请求用量；通用视觉路由按第 6.1 节进行动态计量。

### 2.2 现有生命周期

1. 消息原文写入持久记录，近期记录参与主请求装配。
2. 后台记忆工作线程约每 5 秒检查任务；短周期配置为 1800 秒，常规整理为 21600 秒，周回顾为 604800 秒。
3. 短周期任务按封闭快照和游标准备原文，输出角色卡增量、最多三条 `core_facts` 和 `episode_notes`。
4. 提交成功后保存 episode 与档案、核对提交哈希，归档快照游标覆盖的活跃记录，再推进调度游标。
5. 六小时整理读取已提交 episode 中的 `episode_notes`，结合旧长期档案进行取舍。
6. 旧资料通过 `memory.recall` 按需回取；当前主请求也会装入角色卡、核心事实和操作者已纠正的记忆。

当前活跃记录释放按批次快照边界执行。独立话题工作集列为后续迭代的可选功能，默认关闭。周期配置同时受新增证据、成功完成时间和容量通知影响。

### 2.3 运行审计快照

数据截点：2026-10-28 14:04:26，Asia/Hong_Kong。

| 项目 | 记录值与统计口径 |
| --- | --- |
| 模型调用记录时间范围 | 2026-10-25 20:40:29 至 2026-10-28 14:03:56，约 65 小时 23 分钟的记录跨度 |
| 模型调用记录数 | 3220 条 |
| 原始消息记录数 | 6225 条 |
| 后台调用记录时间范围 | 2026-10-27 10:11:09 至 2026-10-28 13:59:15 |
| 短周期调用 | 300 次：成功 55 次，失败 245 次，调用失败率 81.7% |
| 主要失败原因 | `SHORT_MEMORY_NEEDS_NEW_EVIDENCE` 242 次 |
| 其余失败原因 | Provider 请求失败 2 次，角色卡主体证据校验失败 1 次 |
| 重复最多的任务 | job 52：累计 167 次调用，167 次失败 |
| 失败短周期调用用量 | 输入 5795654 tokens，输出 336231 tokens |
| 成功短周期调用用量 | 输入 1734838 tokens，输出 80301 tokens |
| 六小时整理 | 7 次调用，7 次成功；输入 169742 tokens，输出 4036 tokens |
| 请求硬上限紧急整理 | 实现状态：待增加抢占静默与整理提交协调 |
| 订阅额度百分比观测 | 对应时间窗与起止额度数据：缺项 |

调用次数包含重试。批次完成率另以唯一批次为分母。该故障已登记为 **P0：`short-memory-new-evidence-retry`**。

当前失败路径记录错误后，将下一次尝试设为 300 秒后，任务继续处于待处理状态。代码中的空数组可以通过结构校验；旧核心事实在 key、正文和证据引用完全一致时可以沿用。P0 审查聚焦模型产物与这套约定的匹配情况，以及重复任务的终止机制。

### 2.4 上下文和图片数据

同一审计快照中的三个作用域计量值如下：

| 作用域样本 | 估算输入 tokens | 输出预留 tokens | 预计总量 tokens |
| --- | ---: | ---: | ---: |
| A | 80098 | 128000 | 208098 |
| B | 24507 | 128000 | 152507 |
| C | 19754 | 128000 | 147754 |

2026-10-28 13:41 的视觉缓存统计为 89 条摘要，共 7754 字符，平均 87.1 字符；记录到 73 次图片输入复用。六小时任务的直接输入来自 `episode_notes`，视觉缓存通过独立表保存。

V1.03 的受控真实模型测试中，同一消息和历史的一组图片缓存对照，输入用量从 12282 降至 12028 tokens，净减少 254 tokens，即 2.07%。这 254 tokens 衡量“移除像素、加入特征 ID 与缓存摘要”后的整次输入差值；单图视觉 token 量需依据路由规则或独立分项计量取得。图片输入数从 1 降至 0。该组计量记录中的原图请求输出格式校验结果为 `CONTROL_OR_UNASSIGNED_CHARACTER`，缓存请求为 `OK`。图片语义与补看行为另有 8 项受控用例记录。

## 3. 本次选定的通用架构

```text
接入与身份映射 → 原始事件持久化
                       ├→ 活跃消息 / 可选话题工作集
                       ├→ 图片特征 ID / 作用域视觉缓存
                       └→ 增量整理任务 → 候选记忆 → 校验与提交
                                                    ├→ 记忆版本与证据索引
                                                    └→ 已处理批次与覆盖记录

当前消息 → 检索触发条件 → SQL 条目分级匹配 → 请求分区装配 → 完整预算检查 → 主模型
                                              ↑             │
                                   已提交的工作摘要 ← 容量整理协调
```

| 比较项目 | 采用的设计 | SolBot 实施位置 |
| --- | --- | --- |
| 空变更与失败终止 | 文档的有效空轮次、有界恢复；补齐显式保留旧项协议 | P0 整理协议修复 |
| 正在进行的话题 | 独立话题工作集，按需评估额外调用与注入成本 | P2 后续可选，默认关闭 |
| 历史检索 | 条件触发，SQL 分类筛选、短条目近似匹配、按需证据回读 | P1 条目检索 |
| 原始证据与版本 | 文档的版本链；复用现有文件提交、哈希与纠错日志 | P1 证据索引 |
| 容量调度 | 分级释放、超长输入消极静默与硬上限紧急抢占静默 | P1 容量协调 |
| 图片身份 | 现有可见像素特征 ID、作用域缓存和补看接口 | 保留并扩展 |
| 表情包描述 | 极简语义加关键配文，首次必要识图时同时分类 | P1 表情包分流 |
| 公开资料 | 现有缓存、百科接口、Low 网页查询路径 | 保留，补齐独立预算 |
| 跨群人物信息 | 当前作用域隔离和已认证操作者的受控共享 | 保留 |
| 工作流资源 | 独立请求预算、并发槽位与尝试次数账本 | P0 / P1 分阶段实施 |
| 原图回读 | 现有短时缓存及平台回取 | 保留现有路径 |

## 4. 数据对象与证据结构

### 4.1 记忆内容

| 类型 | 保存内容 | 主要来源 |
| --- | --- | --- |
| `person_fact` | 稳定事实、偏好及明确纠正 | 当事人的原始消息 |
| `interaction_event` | 决策、纠错、重要互动和原因 | 对应参与者的消息与时间线 |
| `character_card` | 当前参与者的简短概括视图 | 已有事实和互动条目 |
| `topic_context` | 进行中的问题、决定、待办及下一步 | 话题相关原文与工作摘要 |
| `image_memory` | 看过的图片主体、关键文字及表达作用 | 实际视觉输入及图片特征 ID |

`short` 服务当前工作，`mid` 保存持续相关的事件和话题，`long` 保存稳定事实。层级、有效状态和存储期限分别管理。

### 4.2 目标逻辑字段

宿主将以下字段映射到已有数据库、事件日志或文件。每个适配器明确字段来源、写入者、检索索引和提交方式。

```yaml
RawMessage:
  id: stable_message_id
  host_id: deployment_id
  scope_id: conversation_scope
  sender_id: authenticated_or_platform_identity
  timestamp: event_time
  received_at: ingestion_time
  source_kind: direct_or_quote_or_forward_or_assistant_or_tool
  text: original_text
  attachment_refs: []
  reply_to: optional_message_id
  content_hash: original_content_digest
  admission_state: eligible_or_archive_only
  model_eligible: host_admission_boolean
  silent_reason: optional_passive_overlength_reason

MemoryItem:
  id: stable_item_id
  scope_id: conversation_scope
  owner_id: optional_person_id
  kind: person_fact_or_interaction_event_or_character_card_or_topic_context_or_image_memory
  tier: short_or_mid_or_long
  category_id: bounded_category_id
  parent_category_id: optional_parent_category
  canonical_key: normalized_short_name
  brief_summary: bounded_interpretation
  summary_tokens: measured_tokens
  detail_ref: optional_original_or_chunk_reference
  evidence_message_ids: []
  derived_from_ids: []
  source_type: self_report_or_observed_interaction_or_external_source_or_inference
  status: active_or_superseded_or_retracted
  version: 1
  supersedes: optional_prior_version_id
  updated_at: commit_time

EntryAlias:
  entry_id: stable_item_id
  alias: bounded_normalized_alias

EntryTag:
  entry_id: stable_item_id
  tag_id: bounded_tag

TopicState:
  enabled: false
  topic_id: stable_topic_id
  scope_id: conversation_scope
  state: active_or_waiting_or_closed
  working_summary: bounded_summary
  open_questions: []
  decisions: []
  next_actions: []
  evidence_message_ids: []
  last_relevant_at: timestamp

MediaReference:
  asset_id: content_identity
  identity_algorithm: versioned_pixel_digest_algorithm
  source_message_ids: []
  platform_ref: optional_platform_message_reference
  availability: recent_cache_or_platform_or_expired
  readable_scope_ids: []

VisualMemory:
  scope_id: conversation_scope
  asset_id: content_identity
  visual_form: illustration_or_photo_or_screenshot_or_info_graphic_or_mixed_or_unknown
  default_usage: reaction_or_informational_or_mixed_or_unknown
  summary: concise_semantics_with_key_caption
  source_visual_read_id: actual_visual_input_record
  summary_version: 1
  read_at: timestamp

MediaOccurrence:
  message_id: stable_message_id
  scope_id: conversation_scope
  asset_id: content_identity
  sender_id: message_sender
  caption: current_message_caption
  platform_subtype: optional_platform_hint
  occurrence_usage: reaction_or_informational_or_mixed_or_unknown

OrganizerJob:
  job_id: stable_job_id
  scope_id: conversation_scope
  cursor_start: first_boundary
  snapshot_end: sealed_last_boundary
  input_hash: prepared_input_digest
  protocol_version: version_id
  base_revision: expected_memory_revision
  state: pending_or_running_or_retry_wait_or_committed_or_no_change_or_quarantined
  attempts: 0
  max_attempts: host_selected_integer  # 推荐 2–4 次
  commit_id: optional_durable_commit_id
  semantic_covered_ranges: []
  context_released_ranges: []
  gap_record_id: optional_persistent_gap_id
  error_field: optional_json_path
  error_code: optional_error_code
```

`MediaReference` 负责图片身份、来源引用和可读范围，`VisualMemory` 负责作用域内的解释，`MediaOccurrence` 负责每次出现的发送者与语境。相同图片可在同一作用域内跨发送者复用摘要。

### 4.3 证据与版本

- 每个事实版本关联原始消息 ID；派生摘要另存 `derived_from_ids`。
- 整理旧 episode 时，沿证据引用读取需要判断的原文片段。
- 证据计数按唯一原始消息或独立事件计算。
- 旧事实继续生效时，使用稳定条目 ID 表达“保留”。
- 新事实和事实变更附上支持该变更的证据。
- 角色卡由事实和互动记录刷新；共享原始证据引用。
- 文件存储采用提交清单、内容哈希、版本检查与可恢复游标；数据库存储采用事务完成相同语义。

## 5. 记忆生命周期与任务终态

### 5.1 实时接收与超长输入消极静默

程序完成消息标准化、原文幂等写入和附件索引，再按宿主预设阈值检查单次交互文字量。阈值按部署配置选择，并明确计量单位和会话适用范围；合并交互采用总文字量口径。

超出阈值时，原始记录标记为 `archive_only`、`model_eligible: false`，轮次直接进入 `passive_silent_overlength`。该交互仅保存原文；主 Agent、子 Agent、参与预判、视觉调用和记忆整理均从模型输入中排除这类正文。历史查询、摘要和引用回读沿用同一准入标记。对外保持静默，轮次在入口结束。

通过准入检查的消息进入近期工作集。图片根据特征 ID 查缓存，文本根据近期出现与响应记录提供重复线索。当前意图、新配文、追问和明确纠错进入回复决策。

### 5.2 短周期整理

短周期推荐 **15–45 分钟**，由宿主依据消息增长和整理耗时选择具体周期。到期且有增量时固定快照边界；容量事件可提前把同一作用域的任务提为待处理。调度器合并重复请求，并保持每个作用域一个有效写入任务。

每批处理流程：

1. 固定游标范围、档案版本、证据集合与输入哈希。
2. 按独立请求预算准备原文和必要旧记忆。
3. LLM 返回 `retain`、新增／更新项、删除项和 episode 候选；可选话题模块启用时附加话题变化。
4. 程序校验引用、作用域、字段结构和版本。
5. 保存结果与提交清单，记录 `committed` 或 `no_change`。
6. 更新已处理批次和覆盖范围，保留当前输入、近期引用和现有核心事实，再选择可释放内容。

空变更保留旧有效条目，写入批次回执并推进处理游标。它与“空证据批次”的程序直达路径分别计量。

### 5.3 长周期整理

长周期推荐 **4–8 小时**，由宿主依据候选积累量和事实更新频率选择具体周期。任务读取增量候选、已有版本和相关原始证据，将结果分为保留、更新、合并、撤回及仅归档。

纯图片外观、单次反应和重复梗图点评保留在事件或视觉层。用户明确表达的偏好、反馈和互动约定按文字证据进入人物或事件记忆。提交后刷新检索索引和角色卡。

### 5.4 活跃话题：后续可选模块

独立话题工作集列为 P2 后续可选项，默认关闭。当前版本继续使用近期对话、核心事实和条件检索衔接上下文。

后续启用时，为 `active / waiting / closed` 状态保存工作摘要、未决事项与证据引用。单独计量话题维护调用、每轮注入量和回忆节省量，根据持续讨论质量与总 token 成本决定启用范围。

### 5.5 失败处理与隔离批次释放

| 结果 | 处理 |
| --- | --- |
| 有效变更 | 提交并记录语义覆盖范围 |
| 有效空变更 | 保留旧有效项，记录完成回执和语义覆盖范围 |
| 结构或证据字段错误 | 记录字段路径和输入／输出哈希，在剩余尝试额度内执行定向修复，推荐 1–2 次 |
| 版本冲突 | 重新读取当前版本并重建增量 |
| 临时服务错误 | 在批次尝试额度内退避重试 |
| 尝试额度耗尽 | 转入 `quarantined`，保存原文引用、错误记录和修复入口 |

每批模型调用总上限推荐 **2–4 次**；定向修复和临时错误重试共同消耗该额度，重启后沿用累计次数。

程序分别维护处理进度、语义覆盖范围和活跃上下文释放范围。`committed / no_change` 记录已接受的整理结果，`quarantined` 记录待修复的语义缺口。后续独立批次继续调度。

达到历史释放水位时，程序按以下顺序处理隔离批次：

1. 核对该批次原文已完整持久化到 `RawMessage`，且稳定 ID 可回读。
2. 持久化缺口记录，包括作用域、原文范围或消息 ID、时间范围、主体索引、失败原因、输入哈希与修复状态。
3. 保留仍处于有效处理状态的当前输入、必要引用和完整工具调用／结果配对；其余隔离批次旧原文允许移出活跃上下文。紧急策略已取消轮次的固定内容转为可释放历史。
4. 在释放账本中登记 `released_reason: quarantined_raw_durable`，批次语义状态继续保持 `quarantined`。
5. 缺口记录留在数据库，主请求按需要注入有界缺口提示；条件检索可按时间、主体或引用读取相关原文片段。

后台修复从持久化缺口回读原文，成功后提交记忆并更新语义覆盖范围。隔离原文的长期保存由 Raw Archive 承担，活跃上下文容量由释放账本管理。

## 6. 上下文预算与调度

> **Warning — 小窗口适配：** 本节推荐区间面向 500K–1M+ 可用窗口。低于 500K 时，需建立单独配置并校准整理批次、输出预留和检索额度；沿用大窗口配置会挤压各区空间，增加容量触发和整理请求。

### 6.1 完整请求计量

```text
I = 指令 + 工具与输出 Schema + 历史 + 当前消息 + 检索结果 + 图像输入
T = I + 输出预留 R + 计量余量 M
正常提交条件：T < 该工作流请求硬上限 C，且 C ≤ 当前模型路由可用窗口 W
紧急抢占条件：T ≥ C
```

在最终请求装配完成后计量；修复消息、工具返回和补看图片加入后重新计量。分别记录 `input_estimate`、`output_reserve`、`projected_total` 和服务返回的实际 `usage`。

**图片输入采用路由动态计量。** 宿主读取目标模型视觉路由的规则，结合实际送入的尺寸、缩放、细节等级，以及该路由采用的瓦片或 patch 计算方式估算视觉 tokens，并计入同时发送的说明文字。动图按实际提交的帧或拼图计量。

路由规则无法取得时，采用宿主可配置的静态保守估算，并记录 `image_estimate_method: static_fallback`；规则可用后切换到动态计量。使用同一路由、相近尺寸和同类输入的实际 usage 校准估算与余量。多图场景逐张计量后汇总。DSH 的 Token Meter 采用路由提供的图像计量规则，并为缺少规则的路由提供启发式估算。见 [DSH Token Meter](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/token-meter.md)。

### 6.2 分级容量处理：消极静默与紧急抢占静默

准备水位推荐为宿主总请求预算的 **65%–75%**，历史释放水位推荐为 **80%–90%**。两者均以包含输出预留和计量余量的预计总占用为口径，并满足“准备水位 < 释放水位 < 请求硬上限”。

1. **准备水位：** 合并同一作用域的容量通知，有可用后台槽位时执行增量整理。
2. **历史释放水位：** 优先移出 `committed / no_change` 覆盖的旧内容；对 `quarantined` 批次，完成原文持久化核对和缺口登记后，同样允许释放旧原文。
3. **请求硬上限：** 立即进入最高模型任务优先级的紧急整理状态。子 Agent 可以直接抛弃当前待回复轮次及其他低优任务，取消相关模型调用和待发内容，对外保持静默，集中完成历史释放、整理与提交。当前轮次记为 `preempted_silent_pressure` 并结束。
4. **单次交互超长：** 在请求装配和任何模型调用前执行阈值检查。超出预设文字量时，仅落盘原始记录，子 Agent 不读取，主请求装配排除该正文，直接消极静默；准入与后续读取规则见第 5.1 节。
5. **紧急阶段结束：** 根据整理终态和释放后的容量恢复后续新消息调度。已静默的轮次保持终态；其发送回执与取消记录用于去重。紧急整理沿用有界尝试、隔离和缺口修复流程。

原始消息接收和持久化在紧急阶段持续运行，前台保持静默。宿主在最终发送前核对轮次取消标记，晚到的旧轮次结果按已结束状态处理；已经确认发送的片段保留回执，其余待发片段取消。

隔离批次释放遵循第 5.5 节的独立账本。释放后的目标水位设在准备水位以下，形成回差。普通证据整理按独立请求预算分批，`archive_only` 记录始终由模型准入过滤。SolBot 的精确参考值见第 10.3 节。

### 6.3 工作流预算

| 工作流 | 配置方法 |
| --- | --- |
| 主对话 | 继承宿主已验证的请求预算与输出预留 |
| 整理 | 独立设置总预算；扣除指令、旧记忆、Schema 和输出预留后计算证据批次容量 |
| 本地检索 | 限制结果条数与注入 tokens；查询在本地程序中执行 |
| 公开搜索 | 独立设置请求预算和返回摘要预算；复用已缓存的有效资料 |
| 视觉调用 | 核对输入模态，按实际视觉路由与图像尺寸动态计量；规则缺项时采用可配置静态保守值 |

为每类独立工作流选择完整请求预算，扣除固定输入、输出预留和余量后，再计算可用证据容量。主对话与整理分别按自身负载校准；模型支持的推理等级通过同批证据比较质量、完成率、时延和实际用量。

宿主自行组装的搜索整理请求采用独立总预算。服务端原生网页路径由服务端展开页面上下文，本地控制查询、调用次数、返回摘要和后续注入额度，并记录实际 usage。

视觉工作流沿用第 6.1 节的动态计量方法。静态估算值由宿主配置与校准，参考部署的每图常数仅保留在实例配置章节。

### 6.3.1 本地程序分区

分区在请求交付模型前由本地程序执行：准入检查 → 标注内容来源 → 选择条目 → 分配额度 → 序列化完整请求 → 最终计量 → 提交。超长交互在准入阶段静默，硬上限触发紧急抢占静默。

| 层级 | 本地执行内容 |
| --- | --- |
| 单请求内容区 | 对当前输入、历史、记忆条目、图像说明和工具结果分别计量、排序、裁剪或分批 |
| 固定开销与输出 | 计入指令、输出 Schema、工具定义、输出预留和估算余量 |
| 独立工作流 | 主对话、整理、自组装搜索请求各用自己的材料与预算 |
| 并发与资源 | 调度槽位、等待队列、尝试次数、实际调用用量 |

宿主复用自己的计量器和请求装配入口，按显式内容类型划区，并在 Provider 完成 Schema 和工具装配后核验完整请求。检索额度计入父级工作区；最终按总预算再次检查。

本地保存的条目按需进入请求，分区与预算调度采用普通程序运算。LLM 执行语义整理和回答。每次请求的所选分区最终组成该次模型输入。

### 6.4 调度顺序与抢占

原始消息持久化和已开始提交的状态一致性维护持续运行。模型任务的优先级为：

1. **最高优先级紧急整理：** 硬上限触发时，当前待回复轮次与低优任务进入取消终态；子 Agent 对外静默，优先释放历史、完成整理与提交。
2. **正常主对话：** 容量正常且通过交互长度准入的消息，以及完成该回答所需的检索、搜索或补看。
3. **到期记忆维护：** 短周期、长周期及隔离缺口修复。
4. **后台补充工作：** 知识补充、派生视图刷新等维护任务。

抢占状态由宿主程序维护，并同步到任务队列、模型取消信号和发送出口。紧急阶段结束后调度新的合格轮次，已取消轮次的待回复内容保留终态。同一作用域的容量任务、周期任务与重试共享任务标识和尝试账本。

### 6.5 成本记录

按主对话、整理、检索后续调用、公开搜索、视觉首读和补看分别记录实际输入、输出、缓存命中与耗时；额外记录超长静默数、紧急抢占数、已取消任务与晚到结果数。整理同时统计唯一批次数、调用次数、空变更数、失败类别和重复输入量。订阅额度统计保存时间窗、起止值及同期其他调用范围。

## 7. 检索、纠错与媒体分流

### 7.1 条件触发的 SQL 条目检索

检索条件包括明确历史询问、已知实体或条目别名的相关询问、指向旧记录的引用、事实纠正，以及模型主动提出的 `memory.recall`。普通闲聊沿当前上下文处理。

**已知实体／别名触发由本地程序匹配，LLM 调用数为 0。** 宿主在内存缓存当前授权作用域内生效的 `canonical_key` 与 `EntryAlias` 词表，按需要构建字典树或多模式自动机。消息进入后由本地代码完成 Unicode／大小写标准化、词边界处理和扫描；命中后携带条目 ID、分类与作用域触发 SQL 候选查询。

词表随条目版本、别名、有效状态和权限范围更新；每个作用域使用独立缓存。模型主动回忆是已有主调用的可选输出，实体触发扫描本身使用上述缓存路径。验收记录触发耗时、误触发率，并核对触发扫描对应的模型调用增量为 0。

检索链路为：

1. **限定范围：** 先按作用域、模型准入标记、人物、条目状态和时间条件筛选。
2. **分级分类：** 按大类、子类、实体、标签及别名缩小 SQL 候选集合。
3. **精确与近似匹配：** 优先稳定 ID、规范名称和别名；对有界候选的短名称、标签及摘要做本地近似排序。
4. **读取短条目：** 选取条目 ID、简述、时间和证据引用；建议选择 **8–15 条**短条目，总注入预算 **2K–6K tokens**，再按宿主负载选定具体上限。
5. **展开证据：** 当前问题需要细节时，按已获模型准入的原始消息或既有证据片段 ID 回读相关内容，再核算额度；`archive_only` 正文由同一过滤规则排除。

条目保存阶段校验名称、别名、标签和摘要长度；查询阶段限制候选数量；展开阶段限制单片段与整轮注入量。长原文保存在原文层，通过稳定引用访问。近似结果保留各自条目 ID，合并动作沿用证据与版本流程。

分类信息复用已有整理产物，近似排序在本地程序执行。指标覆盖触发准确性、分类命中、条目召回率、长片段处理和总注入 tokens。

### 7.2 纠错

原始纠正消息先落盘。程序以当前版本为基础应用编辑，保存旧版本、变更原因和新证据，再刷新当前事实与角色卡。操作者纠正具有可识别的来源和时间，后台更新比较证据时间与条目版本。

### 7.3 图片身份与摘要

SolBot V1.03 使用缩略处理前的可见像素计算特征 ID：先纠正方向、展开透明背景，再纳入像素及尺寸；动画另纳入原文件摘要。相同可见像素的元数据变化保持相同 ID，改字或改变可见内容会形成新 ID。原文件 SHA-256 用于资产完整性核对。

缓存键采用 `(scope_id, image_feature_id)`。主请求中的重复图片以特征 ID 引用同一份摘要目录。每次出现仍保留发送者、时间、配文与引用关系。

近似图片匹配使用单独的候选关系和阈值。精确 ID 继续作为已保存摘要的主键。

### 7.4 表情包分流

1. 保存平台 `sub_type`、商城表情及动画信息，作为首次分类线索。
2. 在首次已经需要的视觉调用中同时生成 `visual_form`、`default_usage` 和摘要。
3. 纯反应图保存一句含关键配文及表达作用的描述，例如“猫咪配字‘今天休息’，表达暂停营业和放松”。
4. 信息图、故障截图、混合用途图片以及明确读字请求进入详细理解路径。
5. 新出现的语境由 `MediaOccurrence` 表达，更新本次用途和回复判断。
6. 长周期候选按证据类型分流：素材描述进入视觉层，明确的用户反馈与约定进入人物或事件层。

同图复读结合已有响应记录决定静默；新问题、新配文和纠错重新参与语义判断。

### 7.5 原图补看

摘要未覆盖当前细节时，程序根据特征 ID 和来源消息查找本轮／近期缓存，再尝试 QQ 平台回取。持久数据保存图片 ID、视觉摘要和消息引用。原图可用性随缓存期限和平台状态更新。

本地原图持久存储方案已取消。视觉补看沿用现有路径。

## 8. 安装和接入流程

### Step 1：建立能力清单

读取宿主代码与有效配置，登记消息入口、身份映射、Session、原文库、请求装配点、计量器、定时器、提交方式和图像来源。模型能力清单包含上下文窗口、最大输出、支持的输入模态和可用推理等级。

### Step 2：映射存储

为原文、记忆、话题、媒体和任务建立逻辑字段映射。选择已有数据源作为原始消息主表。为证据引用建立稳定 ID；为文件档案增加版本索引和提交清单。

### Step 3：先完成整理状态机

实现固定快照、显式旧项保留、有效空变更、定向修复、尝试上限、隔离缺口与恢复。使用已发生的失败样本回放模型输出、校验结果和游标变化。

### Step 4：接入时钟与覆盖记录

复用现有定时器，合并容量通知与周期任务，以提交覆盖记录控制历史释放。独立话题工作集按后续可选模块管理。

### Step 5：接入条件检索

建立条目、分类、别名、人物和时间索引。满足触发条件后分级筛选并匹配短条目；细节按原文引用展开。显式回忆工具与程序检索共用作用域、单条限制和总注入预算。

### Step 6：完成最终请求预算检查

对主对话、整理、搜索、格式修复和补看分别配置预算。每次最终装配后重新计量。容量任务与周期任务共用提交和失败状态机。

### Step 7：接入媒体分流

复用图片特征 ID 和已有摘要缓存，加入平台类型线索、首次分类、关键配文保留和候选记忆路由。按短时缓存和平台来源的可用状态接入补看。

### Step 8：验证并形成部署记录

执行固定样本回放、异常恢复、中文检索和视觉语义测试。部署记录包含生效配置、版本、实际用量、批次完成率、主对话延迟及原文完整性。持续观测按第 6.5 节的同一口径累计。

## 9. DSH / 大肥鱼类宿主的接入映射

本节依据 2026-10-28 查阅的 DeepSeek Harness 上游文档建立适配映射。

| 通用能力 | DSH 接入点 |
| --- | --- |
| 原始历史 | Session 的持久事件日志与模型历史投影 |
| 当前记忆快照 | System Prompt / Runtime Context 的作用域贡献机制 |
| 候选步骤决策 | `agent/pre-step` |
| 最终请求计量 | 宿主装配路径与 `ctx.tokenMeter.measure()` |
| 活跃历史替换 | Compaction / Surface 的范围替换与提交机制 |
| 整理后台任务 | 宿主插件或服务中的任务队列与定时器 |

`agent/pre-step` 处理候选步骤；最终请求预算检查覆盖装配后的全部材料。Token Meter 的 `measure()` 提供请求测量，`contextPressure()` 提供最近记录的展示值。见 [Agent 接口](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/README.md) 和 [Token Meter](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/token-meter.md)。

Compaction 接口支持自动压力整理、空闲整理和指定范围整理。适配器使用原生替换与提交机制处理活跃历史。触及硬上限时进入紧急抢占静默；交互长度准入由宿主在模型步骤之前执行。见 [Compaction](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)。

模型路由同时检查 `context_window`、`max_output_tokens`、`input_modalities` 和推理等级。文本路由配合独立视觉路由处理图片；图像路由直接处理已选定的视觉输入。见 [DeepSeek 模型元数据](https://api-docs.deepseek.com/api/list-models/)。

接入顺序：复用 QQ 消息与身份主表 → 映射证据和记忆版本 → 合并整理队列 → 注入当前快照 → 接入计量与 Compaction。Skills 提供模型行为说明，插件或服务代码实现持久任务与提交控制。

## 10. 配置示例

### 10.1 通用目标配置

以下模板列出推荐范围。宿主在接入时从范围内选择具体值，写入自己的生效配置，并记录选择依据与测量结果。

> **Warning — 配置适用窗口：** 推荐配置面向 500K–1M+ 可用上下文窗口。低于 500K 时使用独立的小窗口配置，重新分配输入、输出和整理预算。

```yaml
agent_memory:
  profile_state: host_selects_values_from_recommended_ranges
  compatibility:
    preferred_context_window: "500K–1M+"
    smaller_window_profile: independent_sizing_and_load_validation
  model_capabilities: read_from_effective_route
  storage:
    raw_archive: reuse_host_source
    evidence_index: enabled
    revision_history: enabled
    media_scope_key: [scope_id, asset_id]
  admission:
    interaction_text_threshold: host_selected_with_explicit_unit_and_scope
    merged_interaction_measure: total_text
    over_threshold_action: persist_raw_and_passive_silent
    over_threshold_model_eligible: false
    eligibility_applies_to: [main, child_agents, engagement, vision, organizer, retrieval]
  lifecycle:
    short_interval_minutes_recommended: [15, 45]
    consolidation_interval_hours_recommended: [4, 8]
    capacity_event: coalesced_with_scheduled_job
    active_topics_enabled: false
    active_topics_schedule: optional_later_iteration
  organizer:
    result_states: [committed, no_change, quarantined]
    unchanged_items: retain_by_id
    max_model_calls_per_batch_recommended: [2, 4]
    targeted_validation_repairs_recommended: [1, 2]
    repair_budget: included_in_batch_attempt_limit
    attempt_counter: durable
    progress: processed_batches_semantic_coverage_and_context_release
    quarantine_release:
      trigger: history_release_watermark
      requirements: [raw_durable, gap_record_durable, required_context_preserved]
      semantic_state_after_release: quarantined
      repair_source: raw_message_references
  retrieval:
    trigger: historical_or_entity_or_reference_or_correction_or_explicit_recall
    entity_trigger:
      execution: local_program
      lexicon: scoped_active_canonical_keys_and_aliases
      matcher: trie_or_multi_pattern_automaton
      refresh: entry_revision_and_scope_access_changes
    candidate_source: sql_classified_entries
    ranking: exact_then_local_short_entry_similarity
    detail_fetch: by_evidence_reference
    explicit_recall: enabled
    top_k_recommended: [8, 15]
    injection_tokens_recommended: [2000, 6000]
    limits: [field_length, candidate_count, evidence_chunk, total_request]
  context:
    partition_owner: local_host_program
    part_identity: explicit_content_type
    accounting: final_request_input_plus_output_and_margin
    budgets: host_selected_per_workflow
    prepare_ratio_recommended: [0.65, 0.75]
    history_release_ratio_recommended: [0.80, 0.90]
    ratio_basis: host_total_request_budget
    release_target: below_prepare_watermark
    releasable_batch_states: [committed, no_change, quarantined]
    pressure_policy: passive_silence_and_emergency_preemptive_silence
    hard_limit_trigger: projected_total_gte_limit
    hard_limit_action: cancel_current_reply_and_lower_priority_model_tasks
    emergency_priority: highest_model_task_priority
    emergency_outbound: silent
    cancelled_turn_state: terminal
    next_turn_dispatch: after_emergency_capacity_recovery
    ingest_persistence: continuous
  media:
    identity: versioned_visible_pixel_digest
    classify_on_first_needed_visual_call: true
    reaction_summary: concise_semantics_with_key_caption
    usage: per_asset_hint_and_per_occurrence_context
    reinspection: recent_cache_then_platform
    persistent_original_storage: false
    token_meter:
      primary: target_route_dynamic_image_rules
      factors: [submitted_dimensions, resize, detail_level, route_tile_or_patch_rules]
      fallback: host_configured_conservative_static_estimate
      calibration: comparable_route_and_input_usage
  telemetry:
    dimensions: [workflow, scope, job, batch, attempt, outcome]
    usage: provider_reported_input_output_and_cache
```

### 10.2 SolBot V1.03 实际配置映射

```yaml
solbot_v103:
  snapshot_date: "2026-10-28"
  package_id: solbot-v103-4e36ac820f5d
  model: gpt-6.1-sol
  main_reasoning: medium
  organizer_reasoning: inherited_medium
  public_web_reasoning: low
  context_workspace_tokens: 512000
  context_auxiliary_tokens: 128000
  context_output_reserve_tokens: 128000
  context_total_tokens: 768000
  context_prepare_percent: 70
  context_rollover_percent: 85
  quick_memory_interval_seconds: 1800
  memory_interval_seconds: 21600
  weekly_review_seconds: 604800
  quick_memory_enabled: true
  compact_memory_enabled: true
  content_memory_enabled: true
  short_commit:
    core_facts_max_items: 3
    history_release: sealed_snapshot_cursor
    unchanged_core_fact_check: exact_key_text_and_refs
    empty_arrays: accepted_by_schema
  failure_retry:
    delay_seconds: 300
    attempt_limit: null
  historical_recall:
    search: profile_text_and_sql_like
    top_k_per_query: 12
  visual_cache:
    key: [scope, image_feature_id]
    persistent_data: visual_summary
    original_access: recent_memory_and_platform_message_lookup
  planned_features:
    active_topic_table: optional_later_iteration_disabled
    conditional_sql_entry_retrieval: pending
    meme_classification: pending
    independent_organizer_budget: pending
    persistent_original_asset_store: cancelled
```

### 10.3 SolBot V1.05 候选参数

| 参数 | 起点 | 选择依据 |
| --- | --- | --- |
| 整理推理等级 | Medium 基线，Low 候选 | 相同证据的准确率、完成率、耗时与实际用量 |
| 整理总请求预算 | 96000 候选 | 完整请求占用、跨批覆盖和摘要完整性 |
| 搜索整理请求预算 | 自组装请求采用 72000 候选；原生网页采用查询、调用次数和摘要限额 | 查询路径、返回规模和回答完整性 |
| 容量水位 | 现行 70%／85% | 主对话延迟、积压量及紧急抢占静默次数 |
| 隔离批次释放 | 候选规则：达到 85% 释放水位后，已持久化原文且已登记缺口的隔离批次可移出旧活跃上下文 | 原文可回读、缺口可修复、当前必要输入保留 |
| 短／长周期 | 现行 1800／21600 秒 | 批次完成率、积压量及整理用量 |
| 尝试与定向修复 | 候选总上限 3 次，其中定向修复最多 1 次 | 重启计数连续、终态清晰 |
| 交互文字阈值 | 由部署配置选择并记录单位和适用范围 | 超长交互落盘、模型调用增量为 0、对外静默 |
| 硬上限策略 | 紧急整理抢占当前待回复与低优任务，对外静默 | 取消终态、发送出口检查和后续新消息恢复 |
| 条目检索注入 | 条件触发，Top 12、4096 tokens 候选 | 触发准确性、证据召回率与实际输入增量 |

## 11. SolBot 现有运行用例

### 用例 A：短周期整理反复失败

**证据类型：生产运行记录。** 审计截点的 300 次短周期调用中，245 次失败。242 次错误来自新证据校验；job 52 累计调用 167 次并全部失败。

当前程序保存错误并等待 300 秒后再次调度同一待处理任务。失败调用累计产生 5795654 输入 tokens 和 336231 输出 tokens。该链路对应 P0 问题，下一版本首先处理协议匹配、有效终态与重复调用。

### 用例 B：成功短周期提交后进入六小时整理

**证据类型：生产调用统计与当前代码路径。** 截点记录了 55 次成功短周期调用和 7 次成功六小时调用。短周期提交保存 `episode_notes`、角色卡和最多三条核心事实；完整批次随后按快照游标归档活跃记录。六小时任务从 episode 文件读取候选内容，更新长期档案。

当前跨周期线索由核心事实、角色卡及显式回忆提供。独立活跃话题工作集列为后续可选项，默认关闭。

### 用例 C：同图跨发送者复用

**证据类型：真实模型受控测试，合成图片与 FakeClient。** 首次提交“今天休息”的猫咪图片时传入 1 张图片并保存摘要。另一发送者提交同图时，输入图片数为 0，模型选择静默。重启并归档后的相同图片仍复用摘要并静默。

同一图片仅改变文件元数据时保持相同特征 ID；将配字改为“今天加班”时形成新 ID，模型读取新图片并指出文字变化。

### 用例 D：摘要缺少细节时补看

**证据类型：真实模型受控测试。** 测试缓存摘要保留主体和配字，省略角落数字。用户追问四位数字时，模型先返回 `visual_reads`；宿主补入图片后，模型回答 `4827`。

这一流程有两次模型请求，补看请求包含 1 张图片。当前像素来源为缓存或平台消息回取。

### 用例 E：重复文本、新图片与多图比较

**证据类型：真实模型受控测试。** 首次文本获得回复，另一发送者复读时模型选择静默。相同问题配上另一张图片时模型正常回答。两张已有摘要的图片再次共同出现时，输入图片数为 0，模型分别回答“今天休息”和“今天加班”。

### 用例 F：按需查找历史记录

**证据类型：当前代码路径。** 模型提出 `memory.recall` 后，宿主从查询文本提取关键词，在长期档案和授权作用域内的原始聊天记录中匹配。SQL 原文路径每个作用域最多取 80 个候选，每个查询最终返回 12 条。主模型收到结果后继续作答。当前查询实现为文本匹配和 `LIKE`。

### 用例 G：请求装配时的容量处理

**证据类型：当前代码路径与运行计量。** 宿主在图片缓存替换后估算请求。达到 70% 时提交容量通知；达到 85% 时尝试释放已有记忆覆盖的历史。继续超出工作预算时，装配器先减少回忆结果，再减少本次请求选择的旧消息。

当前超大私聊输入路径会记录 `current_private_message_exceeds_working_budget` 并交给服务处理。后续容量方案采用超长交互消极静默和硬上限紧急抢占静默。快照样本 A 的估算输入为 80098，加入 128000 输出预留后，预计总量为 208098。

### 用例 H：公开知识查询

**证据类型：当前代码路径。** 稳定公开名词先查询缓存和百科接口；实时信息或需要进一步检索的查询进入 Low 原生网页路径。结果保存来源和时间，再提供给主对话。私人历史问题沿本地 `memory.recall` 路径处理。

### 用例 I：操作者纠正记忆

**证据类型：当前代码路径。** 编辑请求携带文档、条目、版本和本次纠正消息中的证据。程序检查版本，写入带有编辑前内容的事务日志，再更新目标文件并将日志标记为 `committed`。长期观察项记录 `operator_correction`、证据引用和纠正时间。后台更新比较纠正时间与候选证据时间。

### 用例 J：空证据批次与旧核心事实保留

**证据类型：当前代码路径。** 准备结果的证据数组为空时，后台线程直接调用批次完成逻辑。证据存在时，模型输出中的空数组可通过结构校验。保留旧核心事实的当前条件为 key、正文、证据引用均与旧项一致；其他核心事实和 episode 项需要包含本批次的新证据引用。

## 12. 公开参考资料

- [DSH Session](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)
- [DSH Compaction](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)
- [DSH Runtime Context](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/README.md)
- [DSH Agent](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/README.md)
- [DSH Token Meter](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/token-meter.md)
- [DeepSeek 模型元数据](https://api-docs.deepseek.com/api/list-models/)
