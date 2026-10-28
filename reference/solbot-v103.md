> 历史参考数据。用于查阅项目记录；部署参数、实现与验收依据由部署规范和目标环境证据确定。

## 附录 A：SolBot V1.03 历史配置与运行情况（仅供参考）

参考部署：大肥龙 Sol / SolBot V1.03，包标识 `solbot-v103-4e36ac820f5d`。下列记录保留原审计时点与统计口径。

### A.1 模型、预算与工作流（仅供参考）

**已生效配置**

| 项目 | 已生效配置或实现 |
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
| 本地历史检索 | 档案文本匹配、SQL `LIKE`，每个查询最多返回 12 条 |
| 公开知识查询 | 缓存、稳定知识的百科接口、Low 原生网页查询 |

**计划与候选（未生效）**

| 项目 | 原记录状态 |
| --- | --- |
| 整理独立 96K 配额 | 实现状态：待加入参数实验 |
| 搜索独立 72K 配额 | 宿主自组装搜索整理请求的候选值；当前原生网页内部上下文由服务端管理 |
| Reserve 共享账本 | 实现状态：未建立；现有任务由独立请求路径与本地工作线程调度 |

这里的 768000 是宿主配置的请求预算。模型服务窗口由所用路由的模型元数据和服务能力确定。主工作区、辅助输入与输出预留分别参与本地预算检查。

SolBot V1.03 的本地计量使用 `o200k_base`，为每张图片固定预留 16384 tokens，辅助区额外加入 4096 tokens 的结构余量。这些数值属于该部署的静态估算配置。服务返回的 `usage` 保存实际请求用量；通用视觉路由按[部署规范](../docs/installation-guide.zh-CN.md) 第 5.1 节进行动态计量。

### A.2 现有生命周期（仅供参考）

1. 消息原文写入持久记录，近期记录参与主请求装配。
2. 后台记忆工作线程约每 5 秒检查任务；短周期配置为 1800 秒，常规整理为 21600 秒，周回顾为 604800 秒。
3. 短周期任务按封闭快照和游标准备原文，输出角色卡增量、最多三条 `core_facts` 和 `episode_notes`。
4. 提交成功后保存 episode 与档案、核对提交哈希，归档快照游标覆盖的活跃记录，再推进调度游标。
5. 六小时整理读取已提交 episode 中的 `episode_notes`，结合旧长期档案进行取舍。
6. 旧资料通过 `memory.recall` 按需回取；当前主请求也会装入角色卡、核心事实和操作者已纠正的记忆。

当前活跃记录释放按批次快照边界执行。独立话题工作集列为后续迭代的可选功能，默认关闭。周期配置同时受新增证据、成功完成时间和容量通知影响。

### A.3 运行审计快照（仅供参考）

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

同一审计快照中的三个作用域计量值如下：

| 作用域样本 | 估算输入 tokens | 输出预留 tokens | 预计总量 tokens |
| --- | ---: | ---: | ---: |
| A | 80098 | 128000 | 208098 |
| B | 24507 | 128000 | 152507 |
| C | 19754 | 128000 | 147754 |

2026-10-28 13:41 的视觉缓存统计为 89 条摘要，共 7754 字符，平均 87.1 字符；记录到 73 次图片输入复用。六小时任务的直接输入来自 `episode_notes`，视觉缓存通过独立表保存。

V1.03 的受控真实模型测试中，同一消息和历史的一组图片缓存对照，输入用量从 12282 降至 12028 tokens，净减少 254 tokens，即 2.07%。这 254 tokens 衡量“移除像素、加入特征 ID 与缓存摘要”后的整次输入差值；单图视觉 token 量需依据路由规则或独立分项计量取得。图片输入数从 1 降至 0。该组计量记录中的原图请求输出格式校验结果为 `CONTROL_OR_UNASSIGNED_CHARACTER`，缓存请求为 `OK`。图片语义与补看行为另有 8 项受控用例记录。

### A.5 历史实施位置对照（仅供参考）

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

SolBot V1.03 使用缩略处理前的可见像素计算特征 ID：先纠正方向、展开透明背景，再纳入像素及尺寸；动画另纳入原文件摘要。相同可见像素的元数据变化保持相同 ID，改字或改变可见内容会形成新 ID。原文件 SHA-256 用于资产完整性核对。

参考项目当时已取消本地原图持久存储方案，视觉补看沿用短时缓存与平台回取路径。

## 附录 B：历史配置示例与候选参数（仅供参考）

### B.1 旧版通用目标配置示例（仅供参考）

以下保留旧版文档的字段与数值范围，供理解设计背景。其中带有 `recommended` 的字段也是历史参考标签，不构成本次部署的推荐值或取值边界。不能直接导入此段，也不要求在所列范围内选择；本次配置须独立完成[部署规范](../docs/installation-guide.zh-CN.md) 第 8 节的决策记录。

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

以下用例的证据类型、输入输出、指标和结论只属于记录中的 SolBot 版本与测试条件。可用于理解问题与设计取舍；不得直接据此验收其他客户端、底层实现或通用模型。

### 用例 A：短周期整理反复失败（仅供参考）

**证据类型：生产运行记录。** 审计截点的 300 次短周期调用中，245 次失败。242 次错误来自新证据校验；job 52 累计调用 167 次并全部失败。

当前程序保存错误并等待 300 秒后再次调度同一待处理任务。失败调用累计产生 5795654 输入 tokens 和 336231 输出 tokens。该链路对应 P0 问题，下一版本首先处理协议匹配、有效终态与重复调用。

### 用例 B：成功短周期提交后进入六小时整理（仅供参考）

**证据类型：生产调用统计与当前代码路径。** 截点记录了 55 次成功短周期调用和 7 次成功六小时调用。短周期提交保存 `episode_notes`、角色卡和最多三条核心事实；完整批次随后按快照游标归档活跃记录。六小时任务从 episode 文件读取候选内容，更新长期档案。

当前跨周期线索由核心事实、角色卡及显式回忆提供。独立活跃话题工作集列为后续可选项，默认关闭。

### 用例 C：同图跨发送者复用（仅供参考）

**证据类型：真实模型受控测试，合成图片与 FakeClient。** 首次提交“今天休息”的猫咪图片时传入 1 张图片并保存摘要。另一发送者提交同图时，输入图片数为 0，模型选择静默。重启并归档后的相同图片仍复用摘要并静默。

同一图片仅改变文件元数据时保持相同特征 ID；将配字改为“今天加班”时形成新 ID，模型读取新图片并指出文字变化。

### 用例 D：摘要缺少细节时补看（仅供参考）

**证据类型：真实模型受控测试。** 测试缓存摘要保留主体和配字，省略角落数字。用户追问四位数字时，模型先返回 `visual_reads`；宿主补入图片后，模型回答 `4827`。

这一流程有两次模型请求，补看请求包含 1 张图片。当前像素来源为缓存或平台消息回取。

### 用例 E：重复文本、新图片与多图比较（仅供参考）

**证据类型：真实模型受控测试。** 首次文本获得回复，另一发送者复读时模型选择静默。相同问题配上另一张图片时模型正常回答。两张已有摘要的图片再次共同出现时，输入图片数为 0，模型分别回答“今天休息”和“今天加班”。

### 用例 F：按需查找历史记录（仅供参考）

**证据类型：当前代码路径。** 模型提出 `memory.recall` 后，宿主从查询文本提取关键词，在长期档案和授权作用域内的原始聊天记录中匹配。SQL 原文路径每个作用域最多取 80 个候选，每个查询最终返回 12 条。主模型收到结果后继续作答。当前查询实现为文本匹配和 `LIKE`。

### 用例 G：请求装配时的容量处理（仅供参考）

**证据类型：当前代码路径与运行计量。** 宿主在图片缓存替换后估算请求。达到 70% 时提交容量通知；达到 85% 时尝试释放已有记忆覆盖的历史。继续超出工作预算时，装配器先减少回忆结果，再减少本次请求选择的旧消息。

当前超大私聊输入路径会记录 `current_private_message_exceeds_working_budget` 并交给服务处理。后续容量方案采用超长交互消极静默和硬上限紧急抢占静默。快照样本 A 的估算输入为 80098，加入 128000 输出预留后，预计总量为 208098。

### 用例 H：公开知识查询（仅供参考）

**证据类型：当前代码路径。** 稳定公开名词先查询缓存和百科接口；实时信息或需要进一步检索的查询进入 Low 原生网页路径。结果保存来源和时间，再提供给主对话。私人历史问题沿本地 `memory.recall` 路径处理。

### 用例 I：操作者纠正记忆（仅供参考）

**证据类型：当前代码路径。** 编辑请求携带文档、条目、版本和本次纠正消息中的证据。程序检查版本，写入带有编辑前内容的事务日志，再更新目标文件并将日志标记为 `committed`。长期观察项记录 `operator_correction`、证据引用和纠正时间。后台更新比较纠正时间与候选证据时间。

### 用例 J：空证据批次与旧核心事实保留（仅供参考）

**证据类型：当前代码路径。** 准备结果的证据数组为空时，后台线程直接调用批次完成逻辑。证据存在时，模型输出中的空数组可通过结构校验。保留旧核心事实的当前条件为 key、正文、证据引用均与旧项一致；其他核心事实和 episode 项需要包含本批次的新证据引用。
