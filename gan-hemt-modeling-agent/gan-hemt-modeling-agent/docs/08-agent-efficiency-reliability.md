# Agent 效率、恢复、完成率与失败控制

## 1. 定位

本设计把以下四项作为 GaN HEMT 建模 Agent 的竞赛级横切能力：

1. 减少 Agent 延迟与端到端耗时。
2. 提升崩溃恢复能力。
3. 提高竞赛任务完成率。
4. 减少数据、规划、工具、拟合、QA、恢复和交付各环节失败。

四项能力不形成新的顶层架构层，而是分别落入现有 `application`、`domain`、`orchestration`、`infrastructure` 和 `governance`。所有优化都必须服从以下优先级：

> 产物合法与可提交 → 物理 QA → 双器件均有保底卡 → 任务完成率 → 精度 → 耗时与成本

任何降低耗时但损害模型卡合法性、物理 QA、双器件隔离、反作弊证据或恢复能力的优化都不得上线。

## 2. 竞赛任务的确定性定义

所有文档和实现使用同一套状态词典：

| 字段 | 含义 | 典型值 |
|---|---|---|
| `phase` | 当前业务阶段 | `DATA_AUDITED`、`FINAL_VALIDATION`、`PACKAGE_FROZEN` |
| `execution_status` | 执行器运行状态 | `RUNNING`、`RECONCILING`、`FAILED_SAFE`、`TERMINATED` |
| `stage_result` | 单个阶段的后置条件结果 | `DONE`、`NOT_APPLICABLE`、`FAILED` |
| `device_verdict` | 单器件产物结果 | `DEVICE_COMPLETE` 等四级结果 |
| `pair_verdict` | 双器件包内容结果 | `STRICT_COMPLETE`、`VALID_DEGRADED_PACKAGE`、`INVALID_PACKAGE` |
| `submission_status` | 平台上传/回执状态 | `NOT_ATTEMPTED`、`UPLOADING`、`CONFIRMED`、`FAILED`、`OUTCOME_UNKNOWN` |

### 2.1 单器件结果

一个器件只有同时满足以下条件才记为 `DEVICE_COMPLETE`：

- 已冻结一份与该器件绑定的最终模型卡。
- 模型卡通过当时可用的官方 Schema 校验，并能由指定仿真链加载。
- 完整 QA 已执行，结果、失败项和最终接受理由均已记录。
- 运行日志从会话建立到冻结连续可对账。
- 真实 MCP 调用达到官方门槛并具有语义有效的审计证据。
- 模型卡、日志、检查点和摘要中的版本及器件映射一致。

仅“程序执行到最后一个节点”、仅生成文件或仅得到较低 RMSE，都不算完成。单器件结果必须由确定性校验器给出，LLM 只能生成解释，不能自评成功。

单器件结果分为：

- `DEVICE_COMPLETE`：全部必做阶段与当前数据能力下适用的条件必做阶段均通过，满足上述完整条件。
- `DEVICE_DEGRADED_DELIVERABLE`：仍有通过 Schema、可加载性、版本化 `SUBMISSION_HARD` QA、合规证据和产物一致性校验的可提交卡，但至少一项适用的严格质量阶段因受控降级未完整达成。
- `DEVICE_FAILED_SAFE`：保留了最近可信卡、检查点和失败证据，但当前产物未通过全部可提交硬门禁。
- `DEVICE_FAILED`：不存在可验证产物，或状态/证据已无法可信恢复。

真正不适用的可选阶段标记为 `NOT_APPLICABLE`，不会降低结果等级；适用但失败后被跳过的阶段不得伪装成“不适用”。

### 2.2 双器件竞赛结果与完成率

一次正式竞赛运行只有同时满足以下条件才记为 `STRICT_COMPLETE`：

- 两个器件分别达到 `DEVICE_COMPLETE`。
- 形成且校验通过 2 份模型卡、2 份独立日志和 1 份全局总结。
- 两器件的会话、参数卡、数据引用、检查点和事件流无交叉。
- 全局总结统计可由两份日志重算，提交包摘要和文件数量正确。
- 整组运行存在符合官方范围的国产大模型真实调用与决策证据；若项目额外要求每器件调用，只能作为内部加严指标单独报告。

双器件结果分为：

- `STRICT_COMPLETE`：上述条件全部满足。
- `VALID_DEGRADED_PACKAGE`：包内容全部提交硬门禁通过，但至少一个器件为 `DEVICE_DEGRADED_DELIVERABLE`。
- `INVALID_PACKAGE`：缺卡、非法卡、必需证据缺失，或任一器件为 `DEVICE_FAILED_SAFE/DEVICE_FAILED`。

`pair_verdict` 只判断包内容，平台交付另用 `submission_status = NOT_ATTEMPTED / UPLOADING / CONFIRMED / FAILED / OUTCOME_UNKNOWN`。只有平台在官方最终截止前确认，才算真正交付；上传回执不确定时必须按平台查询语义对账，不能把有效包误判为已提交。

北极星指标必须分开统计：

```text
pair_strict_completion_rate = STRICT_COMPLETE 的成对运行数 / 运行前预登记的成对运行数
pair_valid_package_rate = (STRICT_COMPLETE + VALID_DEGRADED_PACKAGE) / 运行前预登记的成对运行数
competition_strict_success_rate = (STRICT_COMPLETE 且截止前 submission_status=CONFIRMED) / 运行前预登记的成对运行数
competition_valid_delivery_rate = (包有效且截止前 submission_status=CONFIRMED) / 运行前预登记的成对运行数
device_strict_completion_rate = DEVICE_COMPLETE 的器件运行数 / 运行前预登记的器件运行数
```

竞赛严格成功率是“提高竞赛任务完成率”的北极星指标；有效包率和有效交付率分别说明保底产物与上传能力，不能替代严格成功率。`DEVICE_FAILED_SAFE` 不计入成功。正式比赛本身只有一次运行，完成率来自版本固定、数据隔离的成对盲测集合，不能用单次正式结果伪造统计率。

每批盲测必须在 Agent 启动前冻结 manifest、数据/seed、代码、配置、契约版本和预期运行数。启动后的鉴权、会话、超时、崩溃和非法产物都必须留在分母中；只有启动前已确认的夹具损坏或测试基础设施无效才可排除，并须记录证据、原因和补跑编号，避免幸存者偏差。

### 2.3 端到端耗时

分别记录以下时间，避免把不同口径混在一起：

- `run_wall_time`：项目入口接受运行到不可变提交包冻结；不包含平台上传与回执，因此不能冒充正式端到端耗时。
- `official_end_to_end_time`：官方计时开始到平台确认提交完成；这是正式运行的真正端到端时间，硬上限以官方 24 小时窗口为准。
- `device_wall_time_by_device`：每个器件从会话建立到该器件最终卡冻结的映射。
- `time_to_first_valid_card_by_device`：每个器件从运行入口到首份可提交基线卡的映射。
- `time_to_dual_baseline`：运行入口到两器件均取得首份可提交基线卡。
- `time_to_best_valid_card_by_device`：运行入口到各器件最佳已验证卡形成的映射。
- `time_to_model_cards_frozen`：运行入口到两份最终模型卡冻结。
- `time_to_valid_package`：运行入口到 2 卡/2 日志/1 总结组装并通过确定性校验。
- `time_to_submission_confirmed`：运行入口到平台确认提交完成；未确认时保持空值并记录失败或右删失原因，不能用上传开始时间代替。
- `active_compute_time`：LLM、MCP、仿真、校验实际执行时间。
- `wait_time`：排队、限流退避、拟合任务轮询和网络等待时间。
- `recovery_time`：检测崩溃到恢复后第一个有效节点完成。

内部 3/6/17/20/22/23 小时门禁是减载、切换和冻结 SLO，超时必须告警和归因，但不会自动改变 `pair_verdict`；是否按时交付由官方截止与 `submission_status` 判定。

每个样本都保留原始值和 max。P50/P95 只在运行前冻结的同版本性能样本集上计算，并固定分位数算法；性能专项样本数不少于 20 时才使用 P95 做趋势判断。10 组成对盲测与少量近正式演练按逐次结果和 max 验收，不用 P95 掩盖单次超时。

## 3. 架构归属

| 现有层/模块 | 新增职责 |
|---|---|
| `application/run_management` | 阶段契约、进度账本、完成判定输入、运行 deadline 传播 |
| `application/device_coordination` | 双器件关键路径、预算再分配、安全并发与弱器件优先 |
| `application/recovery` | 调用 MCP/持久化端口编排远端结果对账、续跑、回滚和失败安全用例 |
| `domain/run` | 阶段状态、必做/可选任务、剩余时间、终止与降级原因 |
| `domain/policies` | 物理接受、科学停止、阶段适用性和降级资格的确定性规则 |
| `domain/decision` | 动作指纹、失败反馈、有限自愈和策略变更证据 |
| `domain/failure` | 稳定失败分类、可重试资格、副作用状态和允许处置 |
| `domain/outcome` | 单器件四级、双器件三级包结果、独立提交状态及统计口径 |
| `orchestration/routing` | deadline 路由、循环检测、恢复路由、熔断与安全降级 |
| `orchestration/checkpoints` | 原子检查点、Schema 版本迁移、恢复点选择和一致性校验 |
| `ports/modeling_mcp` | 幂等键、异步任务状态、结果查询/取消和能力声明 |
| `ports/persistence` | 状态、操作账本、恢复锁、检查点和模型卡版本的持久化端口 |
| `ports/telemetry` | 性能 span、完成、恢复和失败事件的供应商无关端口 |
| `infrastructure/resilience` | 操作账本、超时/重试/熔断、恢复锁和幂等原语；不决定业务 fallback |
| `infrastructure/observability` | UTC/span 计时、依赖图、关键路径、完成/恢复/失败聚合及性能报告的唯一指标真源 |
| `adapters/*` / `infrastructure/runtime` | 连接池和客户端复用；安全缓存只有实测收益且官方允许时才落位，不预建独立平台 |
| `governance/completion` | 单器件与双器件完成判定、保底卡与提交就绪门禁 |
| `governance/reliability` | 失败预算、未分类错误门禁、性能回归和恢复演练门禁 |

这些名称是逻辑模块名。实际 Python 物理路径必须服从项目最终选定的包布局，不能同时创建两套重复模块。

## 4. 减少 Agent 延迟与端到端耗时

### 4.1 先测量关键路径

每个节点、LLM 调用和 MCP 调用至少记录：

- 可跨进程对账的 `started_at_utc`、`finished_at_utc`、`process_epoch_id`，以及同一进程内由单调时钟计算的 `duration_ms`；崩溃前后拆成不同 span，禁止直接比较两个进程的单调时钟值。
- `queue_ms`、`network_ms`、`remote_compute_ms`（接口可提供时）。
- `device_id`、`stage`、`node`、`attempt_id`、`span_status`。
- 输入/输出大小摘要、Token、工具名和模型卡版本。
- `parent_span_id/dependency_ids`、当前 deadline 和剩余阶段预算；运行后从 span 依赖图计算真实关键路径，运行中只使用明确标注的剩余关键路径估计。

未建立至少一轮双器件基线前，不根据猜测引入缓存、并发或模型路由。

### 4.2 优化顺序

1. **减少无价值轮次**：物理计算、误差比较、边界检查和停止条件由确定性代码完成；LLM 只在计划、异常诊断和 retune 方向等决策点调用。
2. **缩小上下文与输出**：向 LLM 提供结构化事实、变化量和必要历史摘要；限制输出 Schema 和最大 Token，不传递完整原始日志。
3. **减少等待**：若 MCP 拟合是异步任务，采用 `submit → status/query → result`，使用有上限的退避轮询，不忙等、不重复提交。
4. **批量只读能力**：官方接口允许时，批量读取参数、QA 或数据摘要；批量调用仍须保持可审计语义。
5. **安全并行**：两器件仅在官方并发规则允许时并行；单器件内只并行不改变会话状态且基于同一模型卡版本的只读操作。
6. **安全缓存**：仅缓存幂等只读结果，键至少包含 endpoint、契约版本、会话、器件、模型卡版本、工具和参数摘要；模型卡变化或会话恢复后立即失效。
7. **提前停止**：达到 QA/误差目标、连续收益不足或阶段 deadline 到达时立即停止低收益搜索，转向最终验证。

### 4.3 严禁的“优化”

- 不缓存或并行执行参数设置、拟合提交、模型卡导出和冻结等副作用操作。
- 不使用跨器件语义缓存复用模型参数或拟合结果。
- 缓存命中不算真实 MCP 调用，不得用缓存规避反作弊调用门槛。
- 不在未证实瓶颈前引入自托管推理引擎、分布式任务平台、多 Worker 或投机拟合。
- 不以减少 QA、日志、检查点或最终校验换取速度。
- 同一器件的副作用操作默认串行；只有官方明确支持且经过一致性测试后才能改变。

## 5. 提升崩溃恢复能力

### 5.1 恢复不变量

任何恢复路径都必须保持：

1. 已接受的最佳模型卡不丢失、不被候选覆盖。
2. 已成功的副作用操作不重复执行。
3. 未知结果先查询对账，不盲目重试。
4. 两器件状态、操作账本和产物仍完全隔离。
5. 恢复后的日志与检查点连续，能够说明崩溃点和恢复动作。

### 5.2 操作账本协议

所有可能改变 MCP 会话或产物的操作使用统一状态：

```text
PLANNED → INTENT_RECORDED → IN_FLIGHT
  ├─→ SUCCEEDED → RESULT_PERSISTED → CHECKPOINTED
  ├─→ FAILED_RETRYABLE / FAILED_PERMANENT
  └─→ OUTCOME_UNKNOWN → RECONCILING → SUCCEEDED / FAILED
```

执行协议：

1. 为一次逻辑副作用创建稳定 `operation_id` 和 `idempotency_key`。该键绑定不可变的规范化请求摘要、会话、工具/契约版本和基线模型卡版本；同一键载荷变化必须在本地拒绝。每次传输使用独立 `attempt_id`，不把 attempt 混入原键。
2. 原子写入操作意图、请求指纹、基线模型卡版本和可选 `retry_of_operation_id`。
3. 调用 MCP；接口支持时传递幂等键或任务 ID。
4. 成功后先保存结果摘要、远端任务 ID 和产物引用，再推进状态。
5. 建立包含操作账本水位的检查点。
6. `IN_FLIGHT/OUTCOME_UNKNOWN` 的查询或幂等重放必须复用原 `operation_id/idempotency_key`，禁止创建新语义操作。
7. 对已确认未产生副作用的 `FAILED_RETRYABLE`，是否复用原键服从官方幂等契约；若官方要求新尝试，则创建新的 operation/key，并用 `retry_of_operation_id` 关联原操作。

若官方接口既不支持幂等键也不提供任务查询，必须通过会话/模型卡版本和产物摘要做已演练的确定性对账；仍无法判断时保持 `OUTCOME_UNKNOWN` 并安全停止该副作用路径，不能用“可能没执行”作为重放依据。是否允许重建会话或从保底卡继续，必须以官方语义和预先演练策略为准。

### 5.3 检查点边界

以下位置必须检查点：

- 数据审计完成、模型初始化完成和计划接受后。
- 每个副作用操作意图写入后、成功结果持久化后。
- 每轮 QA 完成、候选接受/回滚后。
- 首份保底卡、最佳卡和最终卡形成后。
- 两器件预算切换、进入最终验证、模型卡冻结、全局总结/打包和提交状态变化前后。

检查点必须 Schema 版本化、原子写入并带摘要；损坏最新检查点时能够回退到最近一个完整版本。不得保存 Token、API Key 或正式测试原始数据。

### 5.4 单一恢复所有权

- 同一 `run_id + scope_id` 同时最多一个恢复执行者，其中 `scope_id` 至少支持 `device:<device_id>` 和 `global`。器件拟合使用器件级单写者；全局总结、打包、冻结、上传和提交结果对账使用 run 级单写者。
- 恢复入口本身必须幂等；重复点击/重复启动只返回当前恢复状态。
- 恢复优先使用 LangGraph Checkpointer 和简单进程外持久化；没有实测规模需要时不引入 Temporal 等重型平台。
- 文件产物使用临时文件、完整性校验和原子重命名，避免留下半写模型卡或半写日志。

## 6. 提高竞赛任务完成率

### 6.1 阶段契约

每个状态机阶段定义：

- 前置条件和所需数据能力。
- `applicability = APPLICABLE / NOT_APPLICABLE / UNKNOWN`，以及判定证据。
- `requirement_level = SUBMISSION_HARD / STRICT_QUALITY_REQUIRED / OPTIONAL`。
- 成功后置条件和必须产生的证据。
- 运行中不可破坏的不变量。
- 最大耗时、最大尝试次数和剩余时间门槛。
- 允许的 fallback、可跳过条件和下一安全状态。
- 失败分类以及是否允许自愈、重试、回滚或降级。

只有后置条件通过，进度账本才把阶段结果标记为 `DONE`；只有证据证明不适用时才标记 `NOT_APPLICABLE`，未知不能冒充不适用。

### 6.2 适用性与要求等级

- **`SUBMISSION_HARD`**：鉴权/合规证据、器件映射、合法且可加载模型卡、官方/项目安全硬 QA、日志连续性、2 卡/2 日志/1 总结及敏感信息检查。适用项失败即不能形成有效包。
- **`STRICT_QUALITY_REQUIRED`**：数据能力适用时的基础 DC 及官方或冻结策略要求的 C-V、S 参数、温度/自热、Pulse/陷阱等质量阶段。适用项失败不能记为 `DEVICE_COMPLETE`，但在全部 `SUBMISSION_HARD` 通过时可形成降级有效包。
- **`OPTIONAL`**：高场、场板、联合精修和其他低收益 retune；只在保底卡已存在且时间预算允许时执行，跳过不降低结果等级。

缺少某类数据或参数不可辨识时，应记录证据并把相关质量阶段标为 `NOT_APPLICABLE` 或 `UNKNOWN`，不能让整个器件无限等待；`SUBMISSION_HARD` 不允许因数据不足被悄悄降级。官方硬 QA、项目安全硬 QA 和软评分目标必须在版本化策略中分层；官方结论未到位的项目阈值标记为暂定。

### 6.3 有限自愈与降级

- 结构化输出或工具参数校验失败时，把脱敏后的具体错误、允许修复动作和剩余尝试次数回传 Agent。
- 同一失败最多进行配置化的有限自愈；每次必须改变动作、参数或策略，禁止原样重试。
- 连续失败后依次选择：缩小参数组、回滚、切备用国产模型、跳过可选分支、冻结最近有效卡。
- 人工介入仅在官方规则允许时用于恢复或提交复核，并单独记录；人工替代核心 Agent 建模不得计入 Agent 完成率。

## 7. 减少各环节失败

### 7.1 统一失败分类

| 环节 | 典型失败 | 默认处置 |
|---|---|---|
| preflight/auth | 配置缺失、网络不可达、Token 无效、时钟错误 | 赛前 dry-run 阻断；官方计时开始后仍消耗墙钟预算并触发 deadline/减载，不消耗尚未发生的拟合调用预算 |
| session | 建会话失败、会话过期、状态不一致 | 查询能力/状态，有限重建或恢复，不盲重建 |
| data audit | 数据缺失、单位/方向不明、能力不可用 | 标记不可辨识分支，阻断受影响参数组 |
| model init | 版本不符、Schema/参数名错误、初始卡非法 | 停止拟合，回到契约/初始化修复 |
| planning/LLM | 非法 JSON、未知动作、错误参数组、上下文不足 | 结构校验、事实补全、有限自愈或确定性计划 |
| MCP/tool | 参数错误、429、超时、5xx、返回格式错误 | 区分逻辑/瞬时/未知结果，校验、退避、对账或熔断 |
| fitting | 不收敛、长时间无进展、参数贴边 | 缩小参数组、回滚、改用已登记策略或停止 |
| QA/acceptance | 局部变好但全局退化、严重物理违规 | 确定性拒绝候选并回滚 |
| loop/budget | 重复动作、无收益 retune、deadline 临近 | 动作指纹检测、熔断、转最终验证 |
| persistence/recovery | 检查点损坏、重复恢复、未知副作用 | 回退完整检查点、恢复锁、操作对账 |
| artifacts/submission | 器件串卡、文件缺失、摘要不一致、敏感信息 | 冻结门禁阻断并从不可变来源重建提交包 |

`stage` 与 `category` 是两个维度。稳定 `category` 至少覆盖 `CONFIGURATION`、`AUTHENTICATION`、`CONTRACT_VIOLATION`、`TRANSIENT_TRANSPORT`、`RATE_LIMIT`、`TIMEOUT`、`OUTCOME_UNKNOWN`、`DATA_QUALITY`、`PARAMETER_INVALID`、`FIT_NONCONVERGENCE`、`QA_REGRESSION`、`NO_IMPROVEMENT`、`LOOP_DETECTED`、`BUDGET_EXHAUSTED`、`CHECKPOINT_CORRUPT`、`ARTIFACT_INVALID`、`COMPLIANCE_FATAL` 和 `UNCLASSIFIED`。供应商错误先映射到稳定类别，同时保留脱敏原码；发布候选的 `UNCLASSIFIED` 终态数必须为 0。

### 7.2 结构化错误信封

每个失败至少包含：

```text
failure_id, error_code, run_id, device_id, span_id, operation_id,
checkpoint_id, model_card_version, stage, category,
retry_disposition, side_effect_state, attempt_id,
safe_message, allowed_actions, selected_fallback,
primary_cause_status, primary_cause, contributing_causes,
input_summary, remaining_budget
```

`retry_disposition` 至少区分 `NEVER / RETRY_SAME_OPERATION / RETRY_NEW_OPERATION / RECONCILE_FIRST`；`side_effect_state` 至少区分 `NONE / NOT_STARTED / COMMITTED / UNKNOWN`。根因允许“一项稳定主因 + 零到多项促成因素”，`primary_cause_status` 区分疑似与已确认。

原始堆栈和供应商响应可以进入受控诊断日志，但不得把密钥、认证头、正式原始数据或不可控大文本回传 LLM。

默认恢复梯由版本化配置管理：确认无副作用的瞬时网络/429/5xx 在内部阶段 deadline 与官方剩余墙钟共同允许时最多退避重试 3 次；结构化输出或参数格式错误最多让 Agent 修正 2 次；同一证据下同一动作指纹出现 2 次即熔断。参数越界、拟合不收敛和 QA 退化不得作为网络错误原样重试，必须换已登记策略、缩小参数组、回滚或停止。具体数值在取得真实基线后可收紧或调整，但任何配置都必须有上限。

### 7.3 失败闭环

每次演练或回归后：

1. 按阶段与分类聚合失败和耗时。
2. 选择影响完成率或关键路径最大的 Top 1～3 原因。
3. 每次只修改一个主要变量，并为失败样本新增回归用例。
4. 在相同 fixture/seed/version 上做配对 A/B，按预先配置的最小改善量、重复性容差和质量非劣阈值比较完成率、QA、最差器件、耗时和恢复结果。
5. 未达到预注册改善门槛或造成其他指标越过非劣阈值时回滚策略，不能用“可解释”接受退化。

发布候选不得存在未分类失败；“捕获异常后继续”不等于失败已处理。

## 8. 最小状态扩展

在原有运行状态基础上增加：

- 性能：`started_at_utc`、`finished_at_utc`、`process_epoch_id`、`absolute_submission_deadline_utc`、内部 `stage_deadline_utc`、`parent_span_id/dependency_ids`、`remaining_critical_path_estimate_ms`、`wait_ms`、`checkpoint_overhead_ms`。
- 操作：`operation_id`、`idempotency_key`、`request_fingerprint`、`attempt_id`、`retry_of_operation_id`、`remote_task_id`、`operation_status`、`side_effect_state`、`result_digest`。
- 恢复：`last_complete_checkpoint`、`recovery_scope_id`、`recovery_owner`、`recovery_attempt`、`crash_point`、`recovery_started_at_utc`。
- 阶段/完成：`applicability`、`requirement_level`、`stage_result`、`device_verdict`、`pair_verdict`、`submission_status`。
- 失败：规范化 `FailureRecord` 引用、`self_heal_count`、`loop_fingerprint` 和 `selected_fallback`。

字段的正式 Schema 位于内部运行控制契约，不混入官方模型卡 Schema。

## 9. 指标与发布门禁

| 指标 | 定义 | 项目初始发布目标（非官方阈值） |
|---|---|---|
| 双器件严格包率 | `STRICT_COMPLETE / preregistered_pair_runs` | 最近 10 组成对盲测至少 9/10，与降级包分开报告 |
| 双器件有效包率 | `(STRICT_COMPLETE + VALID_DEGRADED_PACKAGE) / preregistered_pair_runs` | 最近 10 组成对盲测为 10/10 |
| 截止前有效交付率 | 有效包且官方/演练平台截止前 `submission_status=CONFIRMED` | 所有预登记提交演练为 100% |
| 单器件合法产物率 | 形成合法卡、完整日志与单器件判定的比例 | 100% |
| `time_to_dual_baseline` | 入口到两器件均有保底卡 | 每次完整窗口演练 ≤6 小时，按逐次值和 max 验收 |
| `time_to_model_cards_frozen` | 入口到两份最终模型卡冻结 | 每次近正式演练 ≤22 小时 |
| `time_to_valid_package` | 入口到包内容校验通过 | 每次近正式演练 ≤23 小时 |
| `time_to_submission_confirmed` | 入口到平台确认完成 | 每次近正式演练不晚于官方 24 小时截止 |
| `official_end_to_end_time` | 官方计时开始到平台确认完成 | 每次正式/近正式演练 ≤官方 24 小时窗口 |
| 恢复安全不变量通过率 | 无重复副作用、最佳卡丢失和器件/全局作用域串扰 | 所有必测与随机注入均为 100% |
| 续跑成功率 | 在 deadline 内恢复到与无故障基线等价的结果等级和模型卡容差 | 必测可恢复点 100%；预登记且不少于 20 次随机注入时 ≥95% |
| 安全遏制率 | 无法可靠对账时确定性进入 `FAILED_SAFE` 并保全证据 | 单独报告，不计入续跑成功率 |
| 恢复耗时 | 崩溃检测到首个恢复后有效节点完成 | 本地/Mock 性能集 `n≥20` 时 P95 ≤5 分钟并报告 max；官方环境记录基线 |
| 重复副作用 / 最佳卡丢失 | 重试/恢复造成重复状态修改或有效卡遗失 | 0 / 0 |
| 无界循环 / 未分类终态失败 | 超限仍执行，或没有 canonical FailureRecord | 0 / 0 |
| `stage_terminal_failure_rate` | 适用阶段最终 `FAILED` 数 / 预登记适用阶段实例数 | 同 fixture/seed 的配对 A/B 不高于冻结基线非劣阈值 |
| `operation_failure_rate` | 终态失败逻辑操作数 / 全部逻辑操作数 | 同故障模型下不高于冻结基线非劣阈值 |
| `attempt_failure_rate` | 失败传输/执行 attempt 数 / 全部 attempt 数 | 与终态失败分开报告，不得用恢复成功掩盖高频失败 |
| `retry_amplification` | 全部 attempt 数 / 唯一逻辑 operation 数 | 同场景不高于冻结基线非劣阈值 |
| `outcome_unknown_rate` | 曾进入 `OUTCOME_UNKNOWN` 的副作用操作 / 全部副作用操作 | 同场景不回归，冻结时未对账数量为 0 |
| `self_heal_success_rate` | 有限自愈后阶段成功的案例 / 触发自愈案例 | 同场景不低于冻结基线非劣阈值 |
| 检查点开销 | 每次运行 `sum(checkpoint_ms) / run_wall_ms`，另报单次写入耗时 | 每次运行 ≤5%；性能集 `n≥20` 才计算跨运行/单写 P95，并始终报告 max |

10 组成对盲测只是最低发布门禁，不是总体可靠率达到 90%/100% 的统计证明；报告必须同时给出 `n`、器件/困难场景分层覆盖和预先选定的置信区间方法。

绝对阈值在获得官方性能基线后可通过版本化配置调整，但不能放宽官方最终截止、产物合法性、重复副作用为 0 和最佳卡不丢失等硬约束。

任何延迟优化必须在相同 manifest 的配对 A/B 中达到预注册最小改善量，同时严格包率、有效交付率、物理 QA、最差器件和恢复指标不越过预注册非劣阈值。

## 10. 测试要求

### 10.1 性能

- 节点、LLM、MCP、持久化和等待时间可完整重算。
- 使用可控延迟验证 critical path 与阶段 deadline。
- 验证只读并行不会跨模型卡版本，副作用调用保持串行。
- 验证缓存键、失效、TTL、会话恢复和缓存命中不计 MCP 调用。
- 固定 manifest、版本和分位数算法；`n<20` 报告全部样本和 max，`n≥20` 才报告 P50/P95，并同时检查严格包率、有效交付率、QA 和最差器件。

### 10.2 恢复

- 在操作意图前、远端调用中、结果返回后、QA 后和冻结中分别终止进程。
- 覆盖远端操作成功但本地未记账的 `OUTCOME_UNKNOWN`。
- 覆盖最新检查点损坏、Schema 版本迁移和重复恢复入口。
- 分别断言恢复安全不变量、续跑成功和安全遏制；`FAILED_SAFE` 不计续跑成功。

### 10.3 完成率

- 覆盖完整数据、缺少条件数据、参数不可辨识、时间不足和一侧器件失败。
- 逐项断言 `DEVICE_COMPLETE`、`STRICT_COMPLETE` 与 `VALID_DEGRADED_PACKAGE`，并独立断言 `submission_status`，禁止仅按终态节点判成功。
- 验证降级包仍通过所有提交硬门禁，但不计入严格包率；`DEVICE_FAILED_SAFE` 和缺少 `SUBMISSION_HARD` 产物均不得伪装完成。

### 10.4 失败控制

- 每类失败均能产生结构化错误信封和确定性处置。
- 逻辑错误不盲重试，瞬时错误遵守预算，未知结果先对账。
- 同一动作重复达到阈值后熔断；有限自愈每次必须改变策略。
- 每个新发现的正式演练失败都进入回归库。
- 预登记各失败率分母，分别核对尝试失败、逻辑操作终态失败、阶段终态失败、重试放大、未知结果和自愈成功，不能只验证“已分类”。

## 11. 目录与产物

- 设计说明：`docs/08-agent-efficiency-reliability.md`。
- 内部运行控制契约：`contracts/runtime_control/`。
- 版本化策略：`config/policies/`。
- 性能、恢复、完成率和失败控制测试：`tests/performance/`、`tests/recovery/`、`tests/completion/`、`tests/failure_control/`。
- 竞赛运行处置：`operations/agent-control-runbook.md`。
- 每次运行目录：

  ```text
  artifacts/runs/<run_id>/
  ├─ devices/<device_id>/{cards,logs,qa,checkpoints}/
  ├─ summary/
  ├─ quality/
  ├─ package/
  └─ submission/
  ```

  `summary/` 只保存本次运行的全局总结，盲测聚合指标进入 `quality/`；实际运行目录默认不提交 Git。

当前项目仍处于设计骨架阶段；上述目录中的 README 和测试规范表示待实现契约，不代表功能代码已经完成。
