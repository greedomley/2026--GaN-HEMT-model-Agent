# 模块职责清单

## 1. `src/entrypoints`

系统主入口边界。负责读取运行配置、执行 preflight、启动正式运行和返回退出状态；不包含建模规则。

## 2. `src/orchestration`

| 子模块 | 职责 |
|---|---|
| `graph` | Agent 决策图、节点和边的结构定义 |
| `state` | 图状态、节点输入输出和状态演进 |
| `routing` | 条件分支、循环、停止和降级路由 |
| `checkpoints` | 关键节点持久化、恢复和模型卡版本关联 |

## 3. `src/agents`

| 子模块 | 职责 |
|---|---|
| `planner` | 基于数据可用性、器件特征和预算形成提参计划 |
| `data_analyst` | 解释数据审计结果并标记风险，不直接修改原始数据 |
| `extraction_strategist` | 在允许策略集合内选择参数组、目标和下一轮动作 |
| `qa_reviewer` | 解释 QA 失败及误差分布，提出受约束的 retune 建议 |
| `recovery_supervisor` | 对超时、限流、会话异常和拟合失败选择恢复路径 |
| `reporter` | 生成决策摘要、运行总结草稿和人工可读解释 |

这些是职责角色，不要求一定实现成相互聊天的多个进程或多个模型。

## 4. `src/application`

| 子模块 | 职责 |
|---|---|
| `bootstrap` | 启动检查、认证检查、运行上下文初始化 |
| `run_management` | 单次正式运行生命周期、状态和最终结果管理 |
| `device_coordination` | 两器件独立运行及跨器件预算调度 |
| `data_audit` | 数据类型、偏置、温度、单位、覆盖范围和异常检查用例 |
| `extraction` | 分组参数提取用例编排 |
| `retune` | QA/误差驱动的受约束再调优用例 |
| `validation` | 模型卡可加载性、物理 QA、误差代理和稳定性验证 |
| `submission` | 竞赛产物冻结、校验和提交包准备 |

## 5. `src/domain`

| 子模块 | 职责 |
|---|---|
| `device` | 器件标识、几何/温度/偏置上下文和器件状态 |
| `dataset` | 服务端数据引用、数据能力清单和质量结论 |
| `run` | 运行阶段、预算、状态和终止原因 |
| `decision` | 决策事实、候选动作、选择、依据和结果 |
| `parameter_groups` | ASM-HEMT 参数组、依赖、冻结/解冻和顺序策略 |
| `fitting` | 拟合目标、候选结果、误差比较、接受和回滚规则 |
| `qa` | 物理 QA 项、严重等级、通过规则和修复映射 |
| `scoring` | QA 与曲线误差代理、双器件鲁棒性指标 |
| `model_card` | 模型卡版本、完整性、来源和检查点关联 |
| `policies` | 参数范围、迭代、停止、降级和安全硬规则 |

## 6. `src/ports`

| 子模块 | 职责 |
|---|---|
| `modeling_mcp` | MeQLab 能力的供应商无关端口 |
| `llm` | 国产大模型推理、结构化输出和统计端口 |
| `persistence` | 图状态、检查点、模型卡版本和运行元数据端口 |
| `telemetry` | 结构化日志、指标和调用追踪端口 |

## 7. `src/adapters`

| 子模块 | 职责 |
|---|---|
| `primarius_mcp` | 认证、会话、MCP 工具调用、错误归一化和结果映射 |
| `domestic_llm` | DeepSeek/Qwen/GLM/Kimi/MiniMax 等可替换接入边界 |

## 8. `src/infrastructure`

| 子模块 | 职责 |
|---|---|
| `persistence` | 状态和检查点的具体存储实现位置 |
| `observability` | JSONL 日志、指标、调用链和统计聚合 |
| `resilience` | 超时、重试、退避、熔断、幂等和恢复 |
| `runtime` | 环境变量、密钥注入、时钟和运行环境信息 |

## 9. `src/governance`

| 子模块 | 职责 |
|---|---|
| `compliance` | 国产大模型调用、每器件有效 MCP 调用等竞赛规则检查 |
| `budget` | 24 小时、MCP 调用、LLM Token 和重试预算 |
| `artifact_validation` | 模型卡、日志、总结的 Schema 和完整性校验 |

## 10. 仓库支撑目录

| 目录 | 职责 |
|---|---|
| `contracts` | 保存获得官方确认后的接口和产物契约 |
| `config` | 环境、模型、提参、QA 和预算策略配置位置 |
| `data` | 自建合成、回归、夹具和清单；不保存正式测试原始数据 |
| `artifacts` | 每次运行产生的模型卡、日志、总结、QA 和决策轨迹 |
| `tests` | 单元、契约、集成、端到端、恢复、回归和竞赛演练 |
| `operations` | 本地运行、竞赛运行和故障恢复手册 |

