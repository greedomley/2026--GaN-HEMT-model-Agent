# Agent 状态机与建模流程

## 1. 顶层状态机

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
  → ARTIFACTS_FROZEN
  → COMPLETED
```

任意可恢复失败进入 `RECOVERING`，恢复成功后返回最近检查点；不可恢复失败进入 `FAILED_SAFE`，保留最后一份可提交模型卡和完整证据链。

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
| Fit Group | 候选模型卡 | 参数边界、收敛、误差和调用证据 |
| Physical QA | QA 报告 | 单调性、对称性、kink、参数范围等 |
| Decide | 接受/重试/retune/回滚 | 只能从允许动作中选择 |
| Global Optimize | 联合优化候选 | 不破坏已通过的关键 QA |
| Final Validate | 最终候选 | 可加载性、Schema、QA、泛化代理 |
| Freeze Artifacts | 不可变产物集 | 模型卡、日志、总结相互一致 |

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
4. 调用 MCP 执行拟合。
5. 记录误差、QA、参数变化和调用证据。
6. 比较候选与基线。
7. 接受、有限 retune 或回滚。
8. 写入决策记录并建立检查点。

## 5. 决策约束

Agent 不得：

- 生成任意未登记参数名。
- 超出参数物理范围。
- 在没有新证据时无限重复同一动作。
- 为降低局部 RMSE 接受严重 QA 退化。
- 删除失败记录或覆盖已冻结产物。
- 用无意义 MCP 调用满足调用次数要求。

## 6. 停止条件

满足任一条件时停止当前参数组：

- 达到预设误差和 QA 目标。
- 连续改进低于最小收益阈值。
- 连续失败或回滚达到上限。
- 当前数据不足以识别该参数组。
- 剩余时间不足，必须转向最终验证。

整个器件运行至少保留三级结果：首份可提交基线、最佳已验证版本、最终冻结版本。

## 7. 状态持久化最小集合

- 运行、器件和 MCP 会话标识。
- 当前节点、参数组、轮次和执行状态。
- 已接受模型卡、候选模型卡和回滚点。
- 参数变化、误差、QA 和可视化引用。
- MCP 调用次数、错误、耗时和重试。
- LLM 模型、Token 统计、结构化决策和依据摘要。
- 已用/剩余时间、调用预算和停止原因。
- 产物路径、校验结果和冻结状态。

