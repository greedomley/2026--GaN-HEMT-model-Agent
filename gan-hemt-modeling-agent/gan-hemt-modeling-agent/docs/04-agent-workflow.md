# Agent 状态机与建模流程

## 1. 顶层业务阶段与执行状态

```text
CREATED
  → PREFLIGHT
  → AUTHENTICATED
  → SESSION_READY
  → DATA_INVENTORIED
  → DATA_AUDITED
  → MODEL_INITIALIZED
  → PLAN_READY
  → EXTRACTING
  → PHYSICAL_QA
  → RETUNING / ROLLED_BACK
  → GLOBAL_OPTIMIZATION
  → FINAL_VALIDATION
  → MODEL_CARDS_FROZEN
  → PACKAGE_ASSEMBLED
  → PACKAGE_VALIDATED
  → COMPLETION_EVALUATED
  → PACKAGE_FROZEN
```

业务 `phase` 与 `execution_status` 必须分离。`phase` 只表示上图的业务位置；`execution_status` 至少包含 `RUNNING`、`WAITING_RETRY`、`RECOVERY_REQUIRED`、`RECONCILING`、`FAILED_SAFE` 和 `TERMINATED`。远端调用是否执行不确定时进入 `RECONCILING`，先查询操作账本和 MCP 远端状态；不可恢复失败进入 `FAILED_SAFE`，保留最近可信卡和完整证据链，但不得因此记为任务完成。

`TERMINATED` 只表示本次执行已终止，竞赛结果由独立确定性判定器给出 `DEVICE_COMPLETE / DEVICE_DEGRADED_DELIVERABLE / DEVICE_FAILED_SAFE / DEVICE_FAILED`，双器件再聚合为 `STRICT_COMPLETE / VALID_DEGRADED_PACKAGE / INVALID_PACKAGE`。模型卡先独立冻结，再组装并验证 2 卡/2 日志/1 总结；完成判定读取验证证据，最后才冻结不可变包。平台交付另由 `submission_status` 跟踪，不与包内容结果混用。

## 2. 主节点职责

| 节点 | 主要输出 | 必须检查 |
|---|---|---|
| Preflight | 环境与预算报告 | 网络、密钥存在性、磁盘、时钟、产物目录 |
| Authenticate | 认证上下文 | Token 有效且不写入日志 |
| Create Session | 独立器件会话 | 会话 ID、服务版本、恢复能力 |
| Inventory Data | 数据能力清单 | DC/C-V/Pulse/S 参数、温度和偏置范围 |
| Audit Data | 数据质量结论 | 单位、方向、缺测、异常点、覆盖范围 |
| Initialize Model | 初始模型卡 | ASM-HEMT、版本、参数合法性 |
| Build Plan | 分组提参计划 | 数据支持、依赖顺序、预算可行性 |
| Fit Group | 候选模型卡 | 操作意图已落账、参数边界、收敛、误差和调用证据 |
| Physical QA | QA 报告 | 单调性、对称性、kink、参数范围等 |
| Decide | 接受/重试/retune/回滚 | 只能从允许动作中选择 |
| Global Optimize | 联合优化候选 | 不破坏已通过的关键 QA |
| Final Validate | 最终候选 | 可加载性、Schema、QA、泛化代理 |
| Freeze Model Cards | 两份不可变候选卡 | 最终验证通过、器件映射和摘要固定 |
| Assemble Package | 待判定提交包 | 2 卡、2 日志、1 总结及清单已生成且相互引用 |
| Validate Package | 包校验证据 | 官方 Schema、文件数量/映射、敏感信息、摘要和可加载性通过 |
| Evaluate Completion | 逐项完成判定 | 阶段适用性、要求等级、包硬门禁和双器件隔离；平台 deadline 不混入包结果 |
| Freeze Package | 不可变提交/诊断包 | 判定结果、文件清单、摘要和失败说明已固定，不再覆盖 |

## 3. 推荐提参顺序

具体参数名称和分组必须以官方 ASM-HEMT/MeQLab 能力为准，框架只预留以下阶段：

1. 器件上下文和数据质量确认。
2. 基础 DC 与低场行为。
3. 电荷与 C-V 行为。
4. 外部寄生与 S 参数/RF 行为。
5. 温度依赖与自热。
6. Pulse I-V 与陷阱/色散。
7. 高场、kink、场板等特殊效应。
8. 分组联合优化。
9. 最终物理 QA 和隐藏网格泛化代理验证。

后置参数组不得用于掩盖前置基础模型的明显错误。

## 4. 单轮迭代

每轮严格执行：

1. 从最近已接受模型卡创建候选版本。
2. 选择一个参数组和一个明确目标。
3. 应用物理边界、冻结依赖和调用预算。
4. 计算稳定动作指纹和幂等键，原子写入副作用操作意图。
5. 调用 MCP 执行拟合；异步能力可用时按 `submit → status/query → result` 受限轮询。
6. 先持久化远端任务 ID、结果摘要和操作终态，再推进图状态；调用结果不确定时转入对账，禁止盲目重调。
7. 记录误差、QA、参数变化、分段耗时和调用证据。
8. 比较候选与基线。
9. 接受、有限 retune 或回滚。
10. 写入决策记录并原子建立包含操作账本水位的检查点。

## 5. 决策约束

Agent 不得：

- 生成任意未登记参数名。
- 超出参数物理范围。
- 在没有新证据时无限重复同一动作。
- 在副作用结果未知时直接重试，或并行修改同一器件状态。
- 为降低局部 RMSE 接受严重 QA 退化。
- 删除失败记录或覆盖已冻结产物。
- 用无意义 MCP 调用满足调用次数要求。
- 让 LLM 绕过确定性完成判定、错误分类或恢复对账。

## 6. 停止条件

满足任一条件时停止当前参数组：

- 达到预设误差和 QA 目标。
- 连续改进低于最小收益阈值。
- 连续失败或回滚达到上限。
- 当前数据不足以识别该参数组。
- 剩余时间不足，必须转向最终验证。
- 阶段 deadline 已到或当前动作不再位于可接受的关键路径预算内。

整个器件运行至少保留三级结果：首份可提交基线、最佳已验证版本、最终冻结版本。

## 7. 状态持久化最小集合

- 运行、器件、MCP 会话、事件、追踪、操作和幂等标识。
- 当前业务阶段、节点、参数组、轮次、执行状态、下一动作、`process_epoch_id` 和最近进展 UTC 时间。
- `state_schema_version`、图/代码/配置/契约版本和随机种子。
- 基线、最佳已验证、候选、回滚和最终冻结模型卡的引用与摘要。
- 参数变化、误差、QA 和可视化引用。
- MCP/LLM 调用次数、canonical FailureRecord、UTC/span 分段耗时、重试 disposition 和副作用状态。
- 操作账本中的意图、稳定幂等键/请求指纹、attempt/retry_of、远端任务 ID、结果摘要和关联检查点。
- LLM 模型、Token 统计、结构化决策和依据摘要。
- 官方最终提交绝对 deadline、内部运行/器件/阶段 deadline，已用/剩余时间、调用/Token/检查点/重试预算和停止原因。
- 最近完整检查点、检查点序号/摘要、`recovery_scope_id`、恢复所有者、恢复次数和崩溃点。
- `applicability`、`requirement_level`、`stage_result`、`device_verdict`、`pair_verdict`、降级原因和 `submission_status`。
- 产物路径、校验结果、冻结状态、失败信封和 fallback 记录。

实际关键路径、P50/P95、完成率和失败分布从事件流聚合，不反复塞入每个检查点；状态只保存 span 依赖、剩余时间估计与 deadline slack。同版本样本 `n≥20` 才计算 P50/P95。正式测试原始数据、Token 和 API Key 不得进入状态或检查点。

## 8. 横切路由规则

每个阶段都必须定义前置条件、`applicability`、`requirement_level`、成功后置条件、最大耗时/尝试、允许动作、失败分类和 fallback。只有后置条件满足才记为 `DONE`；数据能力确实不适用时才记为 `NOT_APPLICABLE`。`SUBMISSION_HARD` 失败使包无效，`STRICT_QUALITY_REQUIRED` 失败最多形成降级包，`OPTIONAL` 跳过不影响等级。

失败统一按“逻辑错误、瞬时错误、远端结果未知、数值/物理失败、预算不足、合规/状态损坏”路由：逻辑错误先确定性修正，瞬时错误按剩余 deadline 有限重试，未知结果先对账，拟合/QA 失败换已登记策略或回滚，预算不足转最终验证，合规或状态损坏进入失败安全。Recovery Supervisor 只能在这些已登记路径中选择，恢复正确性不依赖 LLM。

关键路径、操作账本、完成结果和失败信封的完整契约见 [Agent 效率、恢复、完成率与失败控制](08-agent-efficiency-reliability.md)。
