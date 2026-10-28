# Portable Agent Memory — Installation & Implementation Guide

修订日期：2026-10-28。本次修订分离部署规范、参数决策、验收方法与项目参考资料。

本指南用于给 Chatbot、AI Companion 和 Agent 接入持续记忆。程序负责存储、预算、调度、校验和提交；LLM 负责语义理解、信息提炼与候选记忆生成。

## 部署阅读规则（部署模型与部署人必读）

> **配置数值必须自主决定。** 本次部署所需的模型与路由、预算、周期、水位、次数、并发、检索限额、长度限制及其他参数，必须由**部署模型或部署人依据目标宿主、实际模型能力、负载与使用目标自主决定**，并记录单位、依据和验证结果。本指南不提供可跨宿主直接采用的数值默认值。参考区中的实际值、旧版推荐范围和候选值均不能自动成为本次部署配置，也不要求从这些范围内选值。
>
> **验收必须针对目标部署重新制定。** 部署模型或部署人必须根据本次宿主、底层实现、模型路由、启用能力和使用场景，自主确定验收项目、样本、判定方法与通过阈值。不得把 SolBot 的测试用例、指标、成功记录、错误码或实现细节直接作为其他宿主或通用模型的验收标准；参考项目通过测试不代表本次部署通过验收。
>
> **附录全部仅供参考。** 附录 A–D 中的历史配置、运行用例、候选方案、配置示例与外部资料用于理解设计背景。它们不能自动成为本次配置值或验收判定依据，不是部署步骤、配置默认值或通用验收基线。可以跳过项目附录完成部署。

| 内容区域 | 用途 | 部署模型／部署人的处理方式 |
| --- | --- | --- |
| 第 1–6 节：通用职责、数据与流程 | 说明架构语义和接口职责 | 映射到目标宿主的实际组件；协议语义与可调参数分别处理 |
| 第 7 节：安装和接入流程 | 本次部署的执行入口 | 先核对能力，再完成第 8 节的配置与验收决策，随后接入并验证 |
| 第 8 节：部署配置与验收决策 | 记录本次自行确定的参数和验收计划 | 填写实际值、依据和目标部署证据；未定项不能用附录数值补齐 |
| 第 9 节：DSH 宿主接入映射 | 接入位置的查找线索 | 核对当前客户端版本与真实代码；按实际接口实现，不照搬其他宿主实现 |
| 附录 A–D：参考区 | 历史记录、示例、用例和资料索引 | 只读参考；其中的配置与验收结论不自动对本次部署生效 |

部署模型可以在证据充分时自行完成参数选择和验收设计；缺少必要的宿主信息或用户目标时，记录待确认项并向部署人确认。模型窗口、最大输出和模态等服务能力以实际路由可验证的限制为准，不能自行虚构。正文中的协议初始计数、状态关系和预算不等式表示实现语义，不是性能或调优参数的示例值。最终数值即使与参考值相同，也必须记录本次独立选择的依据与验证结果，不能用数值一致性代替适配证明。

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

## 2. 通用架构与接口职责

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

| 能力 | 通用职责 |
| --- | --- |
| 空变更与失败终止 | 显式保留旧项，接受有效空变更，限制恢复尝试并保留修复入口 |
| 话题工作集 | 作为可选模块，依据目标宿主能力与维护成本决定是否启用 |
| 历史检索 | 条件触发、作用域过滤、短条目匹配和按需证据回读 |
| 原始证据与版本 | 保存来源引用、版本关系和可恢复提交记录 |
| 容量调度 | 根据自主确定的预算与水位协调历史释放和紧急整理 |
| 图片记忆 | 区分图片身份、作用域摘要和每次出现的语境 |
| 公开资料 | 由宿主接入可用查询路径，独立计量与限制注入 |
| 作用域与资源 | 检查访问范围，管理并发槽位、请求预算与尝试账本 |

## 3. 数据对象与证据结构

### 3.1 记忆内容

| 类型 | 保存内容 | 主要来源 |
| --- | --- | --- |
| `person_fact` | 稳定事实、偏好及明确纠正 | 当事人的原始消息 |
| `interaction_event` | 决策、纠错、重要互动和原因 | 对应参与者的消息与时间线 |
| `character_card` | 当前参与者的简短概括视图 | 已有事实和互动条目 |
| `topic_context` | 进行中的问题、决定、待办及下一步 | 话题相关原文与工作摘要 |
| `image_memory` | 看过的图片主体、关键文字及表达作用 | 实际视觉输入及图片特征 ID |

`short` 服务当前工作，`mid` 保存持续相关的事件和话题，`long` 保存稳定事实。层级、有效状态和存储期限分别管理。

### 3.2 目标逻辑字段

宿主将以下逻辑字段映射到已有数据库、事件日志或文件。每个适配器明确字段来源、写入者、检索索引和提交方式。下方是字段说明，不是可直接载入的宿主配置；可调项由部署模型或部署人填写，版本初值按宿主版本机制映射。

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
  version: host_defined_initial_revision
  supersedes: optional_prior_version_id
  updated_at: commit_time

EntryAlias:
  entry_id: stable_item_id
  alias: bounded_normalized_alias

EntryTag:
  entry_id: stable_item_id
  tag_id: bounded_tag

TopicState:
  enabled: host_selected_boolean
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
  summary_version: host_defined_initial_revision
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
  max_attempts: host_selected_positive_integer  # 本次部署自主决定的有限总上限
  commit_id: optional_durable_commit_id
  semantic_covered_ranges: []
  context_released_ranges: []
  gap_record_id: optional_persistent_gap_id
  error_field: optional_json_path
  error_code: optional_error_code
```

`MediaReference` 负责图片身份、来源引用和可读范围，`VisualMemory` 负责作用域内的解释，`MediaOccurrence` 负责每次出现的发送者与语境。相同图片可在同一作用域内跨发送者复用摘要。

### 3.3 证据与版本

- 每个事实版本关联原始消息 ID；派生摘要另存 `derived_from_ids`。
- 整理旧 episode 时，沿证据引用读取需要判断的原文片段。
- 证据计数按唯一原始消息或独立事件计算。
- 旧事实继续生效时，使用稳定条目 ID 表达“保留”。
- 新事实和事实变更附上支持该变更的证据。
- 角色卡由事实和互动记录刷新；共享原始证据引用。
- 文件存储采用提交清单、内容哈希、版本检查与可恢复游标；数据库存储采用事务完成相同语义。

## 4. 记忆生命周期与任务终态

### 4.1 实时接收与超长输入消极静默

程序完成消息标准化、原文幂等写入和附件索引，再按宿主预设阈值检查单次交互文字量。阈值按部署配置选择，并明确计量单位和会话适用范围；合并交互采用总文字量口径。

超出阈值时，原始记录标记为 `archive_only`、`model_eligible: false`，轮次直接进入 `passive_silent_overlength`。该交互仅保存原文；主 Agent、子 Agent、参与预判、视觉调用和记忆整理均从模型输入中排除这类正文。历史查询、摘要和引用回读沿用同一准入标记。对外保持静默，轮次在入口结束。

通过准入检查的消息进入近期工作集。图片根据特征 ID 查缓存，文本根据近期出现与响应记录提供重复线索。当前意图、新配文、追问和明确纠错进入回复决策。

### 4.2 短周期整理

短周期由部署模型或部署人依据目标宿主的消息增长、整理耗时、允许积压与资源限制自主决定，并用本次负载验证。到期且有增量时固定快照边界；容量事件可提前把同一作用域的任务提为待处理。调度器合并重复请求，并保持每个作用域一个有效写入任务。

每批处理流程：

1. 固定游标范围、档案版本、证据集合与输入哈希。
2. 按独立请求预算准备原文和必要旧记忆。
3. LLM 返回 `retain`、新增／更新项、删除项和 episode 候选；可选话题模块启用时附加话题变化。
4. 程序校验引用、作用域、字段结构和版本。
5. 保存结果与提交清单，记录 `committed` 或 `no_change`。
6. 更新已处理批次和覆盖范围，保留当前输入、近期引用和现有核心事实，再选择可释放内容。

空变更保留旧有效条目，写入批次回执并推进处理游标。它与“空证据批次”的程序直达路径分别计量。

### 4.3 长周期整理

长周期由部署模型或部署人依据目标宿主的候选积累量、事实更新频率与整理成本自主决定。任务读取增量候选、已有版本和相关原始证据，将结果分为保留、更新、合并、撤回及仅归档。

纯图片外观、单次反应和重复梗图点评保留在事件或视觉层。用户明确表达的偏好、反馈和互动约定按文字证据进入人物或事件记忆。提交后刷新检索索引和角色卡。

### 4.4 活跃话题：按宿主能力选用

独立话题工作集属于可选能力。部署模型或部署人根据目标宿主是否具备该能力、持续话题需求与维护成本，明确决定启用或不启用；未启用时使用近期对话、核心事实和条件检索衔接上下文。

后续启用时，为 `active / waiting / closed` 状态保存工作摘要、未决事项与证据引用。单独计量话题维护调用、每轮注入量和回忆节省量，根据持续讨论质量与总 token 成本决定启用范围。

### 4.5 失败处理与隔离批次释放

| 结果 | 处理 |
| --- | --- |
| 有效变更 | 提交并记录语义覆盖范围 |
| 有效空变更 | 保留旧有效项，记录完成回执和语义覆盖范围 |
| 结构或证据字段错误 | 记录字段路径和输入／输出哈希，在本次部署自主确定的剩余尝试额度内执行定向修复 |
| 版本冲突 | 重新读取当前版本并重建增量 |
| 临时服务错误 | 在批次尝试额度内退避重试 |
| 尝试额度耗尽 | 转入 `quarantined`，保存原文引用、错误记录和修复入口 |

每批模型调用总上限和定向修复上限均由部署模型或部署人自主决定。总上限必须有限，定向修复和临时错误重试共同消耗总额度，重启后沿用累计次数。具体次数需要结合目标模型的失败类型、恢复收益与调用成本验证。

程序分别维护处理进度、语义覆盖范围和活跃上下文释放范围。`committed / no_change` 记录已接受的整理结果，`quarantined` 记录待修复的语义缺口。后续独立批次继续调度。

达到历史释放水位时，程序按以下顺序处理隔离批次：

1. 核对该批次原文已完整持久化到 `RawMessage`，且稳定 ID 可回读。
2. 持久化缺口记录，包括作用域、原文范围或消息 ID、时间范围、主体索引、失败原因、输入哈希与修复状态。
3. 保留仍处于有效处理状态的当前输入、必要引用和完整工具调用／结果配对；其余隔离批次旧原文允许移出活跃上下文。紧急策略已取消轮次的固定内容转为可释放历史。
4. 在释放账本中登记 `released_reason: quarantined_raw_durable`，批次语义状态继续保持 `quarantined`。
5. 缺口记录留在数据库，主请求按需要注入有界缺口提示；条件检索可按时间、主体或引用读取相关原文片段。

后台修复从持久化缺口回读原文，成功后提交记忆并更新语义覆盖范围。隔离原文的长期保存由 Raw Archive 承担，活跃上下文容量由释放账本管理。

## 5. 上下文预算与调度

> **本次预算须独立计算。** 先核实目标路由的实际可用窗口、最大输出及输入模态，再由部署模型或部署人自主分配各工作流的输入、输出预留与计量余量。本文不以任何参考项目的窗口大小作为通用部署门槛，也不保证某个固定窗口下的可用性。

### 5.1 完整请求计量

```text
I = 指令 + 工具与输出 Schema + 历史 + 当前消息 + 检索结果 + 图像输入
T = I + 输出预留 R + 计量余量 M
正常提交条件：T < 该工作流请求硬上限 C，且 C ≤ 当前模型路由可用窗口 W
紧急抢占条件：T ≥ C
```

在最终请求装配完成后计量；修复消息、工具返回和补看图片加入后重新计量。分别记录 `input_estimate`、`output_reserve`、`projected_total` 和服务返回的实际 `usage`。

**图片输入采用路由动态计量。** 宿主读取目标模型视觉路由的规则，结合实际送入的尺寸、缩放、细节等级，以及该路由采用的瓦片或 patch 计算方式估算视觉 tokens，并计入同时发送的说明文字。动图按实际提交的帧或拼图计量。

路由规则无法取得时，采用宿主可配置的静态保守估算，并记录 `image_estimate_method: static_fallback`；规则可用后切换到动态计量。使用同一路由、相近尺寸和同类输入的实际 usage 校准估算与余量。多图场景逐张计量后汇总。DSH 的 Token Meter 采用路由提供的图像计量规则，并为缺少规则的路由提供启发式估算。见 [DSH Token Meter](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/token-meter.md)。

### 5.2 分级容量处理：消极静默与紧急抢占静默

准备水位和历史释放水位由部署模型或部署人依据目标宿主的增长速率、整理时延与释放能力自主决定。两者均以包含输出预留和计量余量的预计总占用为口径，并满足“准备水位 < 释放水位 < 请求硬上限”。

1. **准备水位：** 合并同一作用域的容量通知，有可用后台槽位时执行增量整理。
2. **历史释放水位：** 优先移出 `committed / no_change` 覆盖的旧内容；对 `quarantined` 批次，完成原文持久化核对和缺口登记后，同样允许释放旧原文。
3. **请求硬上限：** 立即进入最高模型任务优先级的紧急整理状态。子 Agent 可以直接抛弃当前待回复轮次及其他低优任务，取消相关模型调用和待发内容，对外保持静默，集中完成历史释放、整理与提交。当前轮次记为 `preempted_silent_pressure` 并结束。
4. **单次交互超长：** 在请求装配和任何模型调用前执行阈值检查。超出预设文字量时，仅落盘原始记录，子 Agent 不读取，主请求装配排除该正文，直接消极静默；准入与后续读取规则见第 4.1 节。
5. **紧急阶段结束：** 根据整理终态和释放后的容量恢复后续新消息调度。已静默的轮次保持终态；其发送回执与取消记录用于去重。紧急整理沿用有界尝试、隔离和缺口修复流程。

原始消息接收和持久化在紧急阶段持续运行，前台保持静默。宿主在最终发送前核对轮次取消标记，晚到的旧轮次结果按已结束状态处理；已经确认发送的片段保留回执，其余待发片段取消。

隔离批次释放遵循第 4.5 节的独立账本。释放后的目标水位设在准备水位以下，形成回差。普通证据整理按独立请求预算分批，`archive_only` 记录始终由模型准入过滤。本次水位、释放目标和工作流预算记录到部署配置决策表，依据目标环境独立计算与验证。

### 5.3 工作流预算

| 工作流 | 配置方法 |
| --- | --- |
| 主对话 | 继承宿主已验证的请求预算与输出预留 |
| 整理 | 独立设置总预算；扣除指令、旧记忆、Schema 和输出预留后计算证据批次容量 |
| 本地检索 | 限制结果条数与注入 tokens；查询在本地程序中执行 |
| 公开搜索 | 独立设置请求预算和返回摘要预算；复用已缓存的有效资料 |
| 视觉调用 | 核对输入模态，按实际视觉路由与图像尺寸动态计量；规则缺项时采用可配置静态保守值 |

为每类独立工作流选择完整请求预算，扣除固定输入、输出预留和余量后，再计算可用证据容量。主对话与整理分别按自身负载校准；模型支持的推理等级通过同批证据比较质量、完成率、时延和实际用量。

宿主自行组装的搜索整理请求采用独立总预算。服务端原生网页路径由服务端展开页面上下文，本地控制查询、调用次数、返回摘要和后续注入额度，并记录实际 usage。

视觉工作流沿用第 5.1 节的动态计量方法。静态估算值由部署模型或部署人按实际视觉路由配置与校准；其他项目的每图常数不能替代本路由的测量。

### 5.3.1 本地程序分区

分区在请求交付模型前由本地程序执行：准入检查 → 标注内容来源 → 选择条目 → 分配额度 → 序列化完整请求 → 最终计量 → 提交。超长交互在准入阶段静默，硬上限触发紧急抢占静默。

| 层级 | 本地执行内容 |
| --- | --- |
| 单请求内容区 | 对当前输入、历史、记忆条目、图像说明和工具结果分别计量、排序、裁剪或分批 |
| 固定开销与输出 | 计入指令、输出 Schema、工具定义、输出预留和估算余量 |
| 独立工作流 | 主对话、整理、自组装搜索请求各用自己的材料与预算 |
| 并发与资源 | 调度槽位、等待队列、尝试次数、实际调用用量 |

宿主复用自己的计量器和请求装配入口，按显式内容类型划区，并在 Provider 完成 Schema 和工具装配后核验完整请求。检索额度计入父级工作区；最终按总预算再次检查。

本地保存的条目按需进入请求，分区与预算调度采用普通程序运算。LLM 执行语义整理和回答。每次请求的所选分区最终组成该次模型输入。

### 5.4 调度顺序与抢占

原始消息持久化和已开始提交的状态一致性维护持续运行。模型任务的优先级为：

1. **最高优先级紧急整理：** 硬上限触发时，当前待回复轮次与低优任务进入取消终态；子 Agent 对外静默，优先释放历史、完成整理与提交。
2. **正常主对话：** 容量正常且通过交互长度准入的消息，以及完成该回答所需的检索、搜索或补看。
3. **到期记忆维护：** 短周期、长周期及隔离缺口修复。
4. **后台补充工作：** 知识补充、派生视图刷新等维护任务。

抢占状态由宿主程序维护，并同步到任务队列、模型取消信号和发送出口。紧急阶段结束后调度新的合格轮次，已取消轮次的待回复内容保留终态。同一作用域的容量任务、周期任务与重试共享任务标识和尝试账本。

### 5.5 成本记录

按主对话、整理、检索后续调用、公开搜索、视觉首读和补看分别记录实际输入、输出、缓存命中与耗时；额外记录超长静默数、紧急抢占数、已取消任务与晚到结果数。整理同时统计唯一批次数、调用次数、空变更数、失败类别和重复输入量。订阅额度统计保存时间窗、起止值及同期其他调用范围。

## 6. 检索、纠错与媒体分流

### 6.1 条件触发的 SQL 条目检索

检索条件包括明确历史询问、已知实体或条目别名的相关询问、指向旧记录的引用、事实纠正，以及模型主动提出的 `memory.recall`。普通闲聊沿当前上下文处理。

**已知实体／别名触发由本地程序匹配，LLM 调用数为 0。** 宿主在内存缓存当前授权作用域内生效的 `canonical_key` 与 `EntryAlias` 词表，按需要构建字典树或多模式自动机。消息进入后由本地代码完成 Unicode／大小写标准化、词边界处理和扫描；命中后携带条目 ID、分类与作用域触发 SQL 候选查询。

词表随条目版本、别名、有效状态和权限范围更新；每个作用域使用独立缓存。模型主动回忆是已有主调用的可选输出，实体触发扫描本身使用上述缓存路径。本地程序路径检查可记录触发耗时、误触发率，并核对触发扫描自身没有发起模型调用；误触发率等指标的判定阈值由本次计划自主决定。这项检查不构成目标模型语义质量通过的证据。

检索链路为：

1. **限定范围：** 先按作用域、模型准入标记、人物、条目状态和时间条件筛选。
2. **分级分类：** 按大类、子类、实体、标签及别名缩小 SQL 候选集合。
3. **精确与近似匹配：** 优先稳定 ID、规范名称和别名；对有界候选的短名称、标签及摘要做本地近似排序。
4. **读取短条目：** 选取条目 ID、简述、时间和证据引用；条目数量、单条长度与总注入预算由部署模型或部署人依据本次召回目标、主请求可用空间和负载自主决定。
5. **展开证据：** 当前问题需要细节时，按已获模型准入的原始消息或既有证据片段 ID 回读相关内容，再核算额度；`archive_only` 正文由同一过滤规则排除。

条目保存阶段校验名称、别名、标签和摘要长度；查询阶段限制候选数量；展开阶段限制单片段与整轮注入量。长原文保存在原文层，通过稳定引用访问。近似结果保留各自条目 ID，合并动作沿用证据与版本流程。

分类信息复用已有整理产物，近似排序在本地程序执行。指标覆盖触发准确性、分类命中、条目召回率、长片段处理和总注入 tokens。

### 6.2 纠错

原始纠正消息先落盘。程序以当前版本为基础应用编辑，保存旧版本、变更原因和新证据，再刷新当前事实与角色卡。操作者纠正具有可识别的来源和时间，后台更新比较证据时间与条目版本。

### 6.3 图片身份与摘要

宿主明确并版本化图片身份算法，记录方向、透明背景、像素、尺寸与动画的规范化方式。摘要复用按算法定义的同一图片身份进行；可见内容变化与仅元数据变化需要按本宿主的算法分别验证。原文件内容摘要用于资产完整性核对。

缓存键采用 `(scope_id, image_feature_id)`。主请求中的重复图片以特征 ID 引用同一份摘要目录。每次出现仍保留发送者、时间、配文与引用关系。

近似图片匹配使用单独的候选关系和阈值。精确 ID 继续作为已保存摘要的主键。

### 6.4 表情包分流

1. 保存平台 `sub_type`、商城表情及动画信息，作为首次分类线索。
2. 在首次已经需要的视觉调用中同时生成 `visual_form`、`default_usage` 和摘要。
3. 纯反应图保存一句含关键配文及表达作用的描述，例如“猫咪配字‘今天休息’，表达暂停营业和放松”。
4. 信息图、故障截图、混合用途图片以及明确读字请求进入详细理解路径。
5. 新出现的语境由 `MediaOccurrence` 表达，更新本次用途和回复判断。
6. 长周期候选按证据类型分流：素材描述进入视觉层，明确的用户反馈与约定进入人物或事件层。

同图复读结合已有响应记录决定静默；新问题、新配文和纠错重新参与语义判断。

### 6.5 原图补看

摘要未覆盖当前细节时，程序根据特征 ID 和来源消息查找本轮／近期缓存，再尝试目标宿主平台提供的回取路径。持久数据保存图片 ID、视觉摘要和消息引用。原图可用性随缓存期限和平台状态更新。

本指南的视觉补看路径使用近期缓存与平台回取。部署时核对目标宿主是否实际支持这些来源，并据此声明可用范围；无法读取原图时不能声称已完成视觉补看。

## 7. 安装和接入流程

> 本节只描述本次部署的执行步骤。所有需要写入配置的数值必须由部署模型或部署人自主决定；执行前完成第 8 节的参数决策和验收计划。附录中的配置与用例不能填补未定项，也不能作为本次验收标准。

### Step 1：建立目标能力与使用需求清单

读取目标宿主代码与有效配置，登记版本、消息入口、身份映射、Session、原文库、请求装配点、计量器、定时器、提交方式和图像来源。核实实际模型路由的上下文窗口、最大输出、输入模态、工具和结构化输出能力、可用推理等级；记录本次使用场景、负载与可接受成本。按第 8 节自主制定配置值和目标部署验收计划，缺项标记待确认。

### Step 2：映射存储与实现职责

为原文、记忆、话题、媒体和任务建立逻辑字段映射。选择目标宿主已有的数据源作为原始消息主表，为证据引用建立稳定 ID；文件档案采用版本索引和提交清单，数据库采用相应事务语义。映射以目标宿主的实际接口为准；DSH 接入位置可参考第 9 节并重新核对。

### Step 3：实现整理状态机并验证目标模型产物

实现固定快照、显式旧项保留、有效空变更、定向修复、有限尝试、隔离缺口与恢复。调用次数与修复额度采用本次自主决定的配置。使用目标模型和目标宿主协议设计成功、空变更、异常和恢复样本，分别记录模型原始输出、程序校验结果与游标变化；其他项目的字段名、错误码和既往成功次数不构成本次通过条件。

### Step 4：接入时钟与覆盖记录

复用目标宿主定时器，应用本次选择的短／长周期、并发与调度额度，合并容量通知和周期任务，以提交覆盖记录控制历史释放。按实际启用情况接入话题工作集，观测任务积压、完成耗时和恢复行为，再调整本次配置。

### Step 5：接入条件检索

建立条目、分类、别名、人物和时间索引。满足触发条件后分级筛选并匹配短条目，按原文引用展开细节。结果条数、字段长度、候选数量、单片段长度与总注入预算均采用本次自主决定的值；显式回忆工具与程序检索共用访问范围和额度规则。根据目标语言、领域和数据分布设计检索验收样本。

### Step 6：接入最终请求计量与容量协调

对主对话、整理、搜索、格式修复和视觉补看分别计算完整预算，每次最终装配后重新计量。容量任务与周期任务共用提交和失败状态机。使用目标宿主的实际装配结果验证预算边界、历史释放、轮次取消和恢复；水位、余量、延迟与成本的可接受范围由本次验收计划确定。

### Step 7：按目标能力接入媒体分流

目标部署启用视觉能力时，核对图片身份算法、摘要缓存、首次分类、关键配文保留、候选记忆路由和原图补看来源；图片尺寸、缓存期限、摘要长度与计量参数由部署模型或部署人自主决定。文本模型只有接入并验证可用的视觉路径后才能验收视觉能力；本次未启用的能力须在部署范围中明确标注。

### Step 8：执行本次验收计划并形成部署记录

执行第 8 节中针对目标宿主与模型重新制定的验收计划，按本次启用能力选择程序测试、真实模型语义测试和端到端交付验证。样本、指标定义、观察周期和通过阈值均由部署模型或部署人自主决定并记录依据。固定样本回放、异常恢复、中文检索或视觉测试是否适用，由本次能力与使用场景决定。

部署记录包含宿主与模型版本、生效配置、适用范围、实际观测、证据及每项结论。分别标记“通过”“未通过”“未验证”“不适用”；不适用项说明原因，未验证项不计为通过，已执行但未达到条件的项目明确记为未通过。模拟客户端和结构检查不能替代真实模型与目标交付路径的验证。参考用例只帮助理解设计背景，不能证明本次部署成功。持续观测沿用第 5.5 节的计量维度与本次自行制定的判定方法。

## 8. 部署配置与验收决策（待填写）

### 8.1 参数决策表

> **取值责任归本次部署模型／部署人。** 所有数值须基于目标环境自主决定，记录单位、依据和验证状态。实际模型能力是需要核实的外部限制；在这些限制内选择的预算与运行参数才是本次部署配置。下表列出需要作出决策的类别，不预置通用数值，也不保证某个统一配置适合所有宿主。

| 决策类别 | 本次需要确定的内容 | 选择与核实依据 |
| --- | --- | --- |
| 模型与路由 | 主对话、整理、搜索、视觉所用路由与推理设置 | 实际可用接口、能力、任务需求及成本 |
| 服务能力边界 | 实际上下文窗口、最大输出、支持的输入模态 | 目标路由的有效元数据、配置和实测限制 |
| 各工作流预算 | 总请求预算、输出预留、计量余量、证据批次容量 | 完整请求计量、模型能力与本次负载 |
| 整理与调度 | 短／长周期、补充回顾周期、并发、队列与退避参数 | 消息增长、完成耗时、积压、资源和恢复要求 |
| 失败恢复 | 批次总尝试上限、定向修复上限、超时与重试间隔 | 目标模型的失败类型、恢复收益及允许成本 |
| 容量协调 | 准备水位、释放水位、释放目标 | 整理时延、历史增长及可释放空间；注明按何种总量计量 |
| 输入准入 | 单次与合并交互的文字量阈值及单位 | 本次交互习惯、允许输入范围、模型与宿主限制 |
| 检索与记忆条目 | 返回条数、候选数量、字段与片段长度、注入额度 | 本次语言／领域的召回需求和父工作流可用预算 |
| 媒体与缓存 | 尺寸处理、摘要长度、缓存期限、动态计量或静态保守估算 | 实际视觉路由规则、图片来源与可用性 |
| 可选模块与数据期限 | 话题工作集等能力的启用范围、适用的存储期限 | 目标宿主实现与本次使用需求 |

每项记录至少包括：参数名、本次值、单位、适用工作流／作用域、选择依据、证据来源、验证状态。实际配置字段以目标宿主为准，本文逻辑字段需要通过适配器映射。

自主取值仍需满足本架构的语义约束：完整请求预计总量小于该工作流硬上限，硬上限不超过实际可用窗口；输出预留不超过路由最大输出；准备水位低于释放水位，释放水位低于硬上限，释放目标低于准备水位；检索和图像占用计入父请求；修复与临时错误重试共同消耗有限的批次尝试额度。这些关系需在目标实现中验证，不能用“与参考数值相同”替代验证。

### 8.2 针对目标部署重新制定验收计划

> **本指南不提供跨底层实现通用的项目验收标准。** 本节提供制定计划的方法，不提供 SolBot 或其他项目的统一通过标准。每次部署须依据真实宿主、模型能力、使用目标和已启用范围独立制定验收计划。参考区的调用次数、失败率、token 节省量、图片数量、延迟、错误码及测试结果均不能自动成为本次阈值或预期输出。

验收对象是本次宿主、适配器与模型组成的实际链路。程序行为、模型语义与最终交付分别取证，不能用其中一层的成功替代其他层的结论。

先写明目标宿主与版本、模型路由与能力、启用模块、使用场景、负载、验收范围和证据采集方式。再由部署模型或部署人选择适用检查、构造目标环境样本、定义指标及通过条件，并在执行前记录判定依据。阈值或范围在测试后调整时，记录调整原因与对应的重新验证结果。

| 可选检查维度 | 在目标部署中需要设计与观察的内容 |
| --- | --- |
| 存储与提交 | 原文可回读、引用与作用域正确、提交一致性、去重与恢复；按目标存储实现设计故障场景 |
| 模型语义与协议 | 目标模型能否生成可接受的候选、保留有效旧项、处理空变更与纠错；结构合法性和语义正确性分别判定 |
| 检索质量 | 依据本次语言、领域和历史查询需求选择样本，定义触发、召回、证据回读与注入量的判定方法 |
| 预算与调度 | 使用目标计量器、请求装配和任务调度路径，观察边界行为、积压、释放与取消后的恢复 |
| 媒体能力 | 仅对本次启用的实际视觉路径设计身份、摘要复用、细节补看与可用性测试；纯文本部署说明不适用范围 |
| 实际交付 | 验证宿主收到、处理并交付结果的完整路径；记录取消、丢失、重复及未交付情况 |
| 负载与资源 | 由本次使用目标确定观察周期、样本量、允许时延、资源占用与调用成本，并在目标环境测量 |

以上是制定计划的检查维度，不是必须原样复制的测试套件。计划必须覆盖本次启用能力与使用者目标；具体检查项、数据、数量和阈值由本次部署自行确定。接口名称、字段结构、token 估算方法和输出格式随目标实现映射；可迁移的架构语义需要转化为本次可观察的行为，不能直接运行另一个项目的验收脚本并据此给出结论。

每个验收项记录：适用条件、目标能力、测试方法、样本来源、指标定义、独立制定的通过条件、实际结果、证据与结论。结论限定在已测环境：

- **通过：** 目标部署实际执行了该项，证据支持本次预定通过条件。
- **未通过：** 已执行且证据表明未达到本次通过条件；记录失败表现、影响与待修复项。
- **未验证：** 未执行或证据不足，记录原因与待处理项，不计为通过。
- **不适用：** 本次未启用或不要求该能力，记录能力边界与原因；不能用此标签掩盖已承诺能力的失败。

模拟客户端可验证相应程序路径，结构校验可验证输出格式；它们各自支持的结论须单独记录。真实模型语义表现和实际交付结果需要目标路径的相应证据。其他模型、宿主或参考用例的测试成功，不能替代这些证据。

### 8.3 待填写的部署记录结构

以下仅表示部署记录结构，**不是任何宿主可直接导入的配置文件，也不是可直接执行的验收计划**。`null` 表示尚未自主确定，不能解释成无限制、关闭、自动值或默认值；空列表表示尚未列出本次项目。正式启用前，必须完成适用参数与验收项目的填写和宿主字段映射；不适用项须明确记录原因。

```yaml
deployment_record:
  status: draft_requires_deployment_decisions
  target_host_and_version: null
  enabled_capabilities: []
  intended_use_and_workload: null
  decision_owner: null
  verified_model_capabilities:
    route_id: null
    context_window_tokens: null
    max_output_tokens: null
    input_modalities: []
    evidence_ref: null
  independently_selected_parameters:
    workflows: []  # 为每个启用工作流记录总预算、输出预留、余量及依据
    short_interval_seconds: null
    consolidation_interval_seconds: null
    batch_max_attempts: null
    targeted_repair_max_attempts: null
    concurrency_limit: null
    retry_delay_seconds: null
    prepare_ratio: null
    history_release_ratio: null
    release_target_ratio: null
    interaction_text_threshold: null
    interaction_text_unit: null
    retrieval_top_k: null
    retrieval_candidate_limit: null
    retrieval_injection_tokens: null
    media_settings: null  # 启用视觉时自主填写，否则记录不适用原因
  parameter_decisions: []  # 参数、本次值、单位、范围、依据、证据、验证状态
  acceptance_plan:
    target_environment_and_scope: null
    cases: []  # 本次自行设计的方法、样本、判定条件和证据要求
    thresholds_and_rationale: []  # 本次自行决定的指标定义、阈值与依据
    not_applicable_capabilities: []  # 能力与不适用原因
  acceptance_results: []  # 实际结果、证据、通过／未通过／未验证／不适用
  deployment_conclusion_and_limits: null
```

## 9. DSH / 大肥鱼类宿主的接入映射

本节保留依据 2026-10-28 查阅的 DeepSeek Harness 上游文档建立的接入位置映射。不同 DSH 客户端、分支和版本可能采用不同接口与底层实现，部署时必须核对目标代码和有效配置。该映射不指定本次参数值、测试样本或验收阈值；这些内容由部署模型或部署人按第 8 节重新制定。

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

## 参考区边界：以下内容仅供参考，不规定本次配置与验收标准

部署执行以正文第 7 节为入口，参数与验收计划在第 8 节独立制定。以下历史资料不再向正文提供默认数值或通过标准。按片段读取本文件时，也须保留每个参考小节的“仅供参考”声明。

## 附录 A：SolBot V1.03 历史配置与运行情况（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

参考部署：大肥龙 Sol / SolBot V1.03，包标识 `solbot-v103-4e36ac820f5d`。下列记录保留原审计时点与统计口径。

### A.1 模型、预算与工作流（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

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

SolBot V1.03 的本地计量使用 `o200k_base`，为每张图片固定预留 16384 tokens，辅助区额外加入 4096 tokens 的结构余量。这些数值属于该部署的静态估算配置。服务返回的 `usage` 保存实际请求用量；通用视觉路由按第 5.1 节进行动态计量。

### A.2 现有生命周期（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

1. 消息原文写入持久记录，近期记录参与主请求装配。
2. 后台记忆工作线程约每 5 秒检查任务；短周期配置为 1800 秒，常规整理为 21600 秒，周回顾为 604800 秒。
3. 短周期任务按封闭快照和游标准备原文，输出角色卡增量、最多三条 `core_facts` 和 `episode_notes`。
4. 提交成功后保存 episode 与档案、核对提交哈希，归档快照游标覆盖的活跃记录，再推进调度游标。
5. 六小时整理读取已提交 episode 中的 `episode_notes`，结合旧长期档案进行取舍。
6. 旧资料通过 `memory.recall` 按需回取；当前主请求也会装入角色卡、核心事实和操作者已纠正的记忆。

当前活跃记录释放按批次快照边界执行。独立话题工作集列为后续迭代的可选功能，默认关闭。周期配置同时受新增证据、成功完成时间和容量通知影响。

### A.3 运行审计快照（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

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

### A.4 上下文和图片数据（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

同一审计快照中的三个作用域计量值如下：

| 作用域样本 | 估算输入 tokens | 输出预留 tokens | 预计总量 tokens |
| --- | ---: | ---: | ---: |
| A | 80098 | 128000 | 208098 |
| B | 24507 | 128000 | 152507 |
| C | 19754 | 128000 | 147754 |

2026-10-28 13:41 的视觉缓存统计为 89 条摘要，共 7754 字符，平均 87.1 字符；记录到 73 次图片输入复用。六小时任务的直接输入来自 `episode_notes`，视觉缓存通过独立表保存。

V1.03 的受控真实模型测试中，同一消息和历史的一组图片缓存对照，输入用量从 12282 降至 12028 tokens，净减少 254 tokens，即 2.07%。这 254 tokens 衡量“移除像素、加入特征 ID 与缓存摘要”后的整次输入差值；单图视觉 token 量需依据路由规则或独立分项计量取得。图片输入数从 1 降至 0。该组计量记录中的原图请求输出格式校验结果为 `CONTROL_OR_UNASSIGNED_CHARACTER`，缓存请求为 `OK`。图片语义与补看行为另有 8 项受控用例记录。

### A.5 历史实施位置对照（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

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

### A.6 历史图片身份与存储选择（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

SolBot V1.03 使用缩略处理前的可见像素计算特征 ID：先纠正方向、展开透明背景，再纳入像素及尺寸；动画另纳入原文件摘要。相同可见像素的元数据变化保持相同 ID，改字或改变可见内容会形成新 ID。原文件 SHA-256 用于资产完整性核对。

参考项目当时已取消本地原图持久存储方案，视觉补看沿用短时缓存与平台回取路径。

## 附录 B：历史配置示例与候选参数（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

### B.1 旧版通用目标配置示例（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

以下保留旧版文档的字段与数值范围，供理解设计背景。其中带有 `recommended` 的字段也是历史参考标签，不构成本次部署的推荐值或取值边界。不能直接导入此段，也不要求在所列范围内选择；本次配置须独立完成正文第 8 节的决策记录。

> **历史示例的适用背景：** 旧版范围面向 500K–1M+ 可用上下文窗口，曾要求低于 500K 时另行配置。该范围不是通用部署门槛，也不是模型能力或验收标准；所有目标窗口均需基于实际路由重新分配和验证。

```yaml
agent_memory:
  profile_state: reference_only_not_an_effective_deployment_config
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

### B.2 SolBot V1.03 实际配置映射（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

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

### B.3 SolBot V1.05 候选参数（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

候选项不代表已实施或已验证；其中的起点、比较方法和指标也不能直接成为其他部署的验收标准。

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

### B.4 旧版正文中的数值范围（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

这些数值从部署正文移到此处保留。它们不构成跨宿主的默认值、推荐值、取值边界或验收阈值。

| 参数 | 旧版参考范围 |
| --- | --- |
| 短周期整理 | 15–45 分钟 |
| 长周期整理 | 4–8 小时 |
| 批次模型调用总上限 | 2–4 次 |
| 定向修复次数 | 1–2 次，计入批次总上限 |
| 容量准备水位 | 宿主总请求预算的 65%–75% |
| 历史释放水位 | 宿主总请求预算的 80%–90% |
| 检索返回短条目数 | 8–15 条 |
| 检索总注入预算 | 2K–6K tokens |
| 原示例面向的可用窗口 | 500K–1M+；旧文以 200K 级窗口说明另行配置的需要 |

## 附录 C：SolBot 历史运行用例（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

以下用例的证据类型、输入输出、指标和结论只属于记录中的 SolBot 版本与测试条件。可用于理解问题与设计取舍；不得直接据此验收其他客户端、底层实现或通用模型。

### 用例 A：短周期整理反复失败（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：生产运行记录。** 审计截点的 300 次短周期调用中，245 次失败。242 次错误来自新证据校验；job 52 累计调用 167 次并全部失败。

当前程序保存错误并等待 300 秒后再次调度同一待处理任务。失败调用累计产生 5795654 输入 tokens 和 336231 输出 tokens。该链路对应 P0 问题，下一版本首先处理协议匹配、有效终态与重复调用。

### 用例 B：成功短周期提交后进入六小时整理（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：生产调用统计与当前代码路径。** 截点记录了 55 次成功短周期调用和 7 次成功六小时调用。短周期提交保存 `episode_notes`、角色卡和最多三条核心事实；完整批次随后按快照游标归档活跃记录。六小时任务从 episode 文件读取候选内容，更新长期档案。

当前跨周期线索由核心事实、角色卡及显式回忆提供。独立活跃话题工作集列为后续可选项，默认关闭。

### 用例 C：同图跨发送者复用（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：真实模型受控测试，合成图片与 FakeClient。** 首次提交“今天休息”的猫咪图片时传入 1 张图片并保存摘要。另一发送者提交同图时，输入图片数为 0，模型选择静默。重启并归档后的相同图片仍复用摘要并静默。

同一图片仅改变文件元数据时保持相同特征 ID；将配字改为“今天加班”时形成新 ID，模型读取新图片并指出文字变化。

### 用例 D：摘要缺少细节时补看（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：真实模型受控测试。** 测试缓存摘要保留主体和配字，省略角落数字。用户追问四位数字时，模型先返回 `visual_reads`；宿主补入图片后，模型回答 `4827`。

这一流程有两次模型请求，补看请求包含 1 张图片。当前像素来源为缓存或平台消息回取。

### 用例 E：重复文本、新图片与多图比较（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：真实模型受控测试。** 首次文本获得回复，另一发送者复读时模型选择静默。相同问题配上另一张图片时模型正常回答。两张已有摘要的图片再次共同出现时，输入图片数为 0，模型分别回答“今天休息”和“今天加班”。

### 用例 F：按需查找历史记录（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：当前代码路径。** 模型提出 `memory.recall` 后，宿主从查询文本提取关键词，在长期档案和授权作用域内的原始聊天记录中匹配。SQL 原文路径每个作用域最多取 80 个候选，每个查询最终返回 12 条。主模型收到结果后继续作答。当前查询实现为文本匹配和 `LIKE`。

### 用例 G：请求装配时的容量处理（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：当前代码路径与运行计量。** 宿主在图片缓存替换后估算请求。达到 70% 时提交容量通知；达到 85% 时尝试释放已有记忆覆盖的历史。继续超出工作预算时，装配器先减少回忆结果，再减少本次请求选择的旧消息。

当前超大私聊输入路径会记录 `current_private_message_exceeds_working_budget` 并交给服务处理。后续容量方案采用超长交互消极静默和硬上限紧急抢占静默。快照样本 A 的估算输入为 80098，加入 128000 输出预留后，预计总量为 208098。

### 用例 H：公开知识查询（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：当前代码路径。** 稳定公开名词先查询缓存和百科接口；实时信息或需要进一步检索的查询进入 Low 原生网页路径。结果保存来源和时间，再提供给主对话。私人历史问题沿本地 `memory.recall` 路径处理。

### 用例 I：操作者纠正记忆（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：当前代码路径。** 编辑请求携带文档、条目、版本和本次纠正消息中的证据。程序检查版本，写入带有编辑前内容的事务日志，再更新目标文件并将日志标记为 `committed`。长期观察项记录 `operator_correction`、证据引用和纠正时间。后台更新比较纠正时间与候选证据时间。

### 用例 J：空证据批次与旧核心事实保留（仅供参考）

> **仅供参考：** 本节记录参考项目的历史配置、实现或观测，不是部署步骤、参数默认值或通用模型验收标准；本次配置和验收判定必须由部署模型／部署人依据目标环境独立制定。

**证据类型：当前代码路径。** 准备结果的证据数组为空时，后台线程直接调用批次完成逻辑。证据存在时，模型输出中的空数组可通过结构校验。保留旧核心事实的当前条件为 key、正文、证据引用均与旧项一致；其他核心事实和 episode 项需要包含本批次的新证据引用。

## 附录 D：公开参考资料（仅供参考）

以下链接提供背景与接口查找线索。实际接入以目标宿主版本和有效路由为准；外部资料与其他项目的结论不能代替本次参数决策和验收。

- [DSH Session](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)
- [DSH Compaction](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)
- [DSH Runtime Context](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/README.md)
- [DSH Agent](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/README.md)
- [DSH Token Meter](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/token-meter.md)
- [DeepSeek 模型元数据](https://api-docs.deepseek.com/api/list-models/)
