# 系统总体架构

## 1. 架构风格

系统采用分层架构与端口-适配器模式。ASM-HEMT 建模规则位于领域层，外部 MeQLab/MCP、国产大模型和存储实现位于适配层，避免领域逻辑被特定 SDK 或服务接口绑定。

## 2. 总体组件

```text
用户 / 测试日启动器
        │
        ▼
Entry Points
        │
        ▼
Run Management ── Device Coordination ── Budget / Completion Governance
        │
        ▼
LangGraph Orchestration ── Checkpoint / Conditional / Recovery Routing
        │
        ├────────► Constrained Agent Roles ───────► Domestic LLM Port
        │
        ▼
Application Use Cases
        │
        ▼
Domain Rules: Device / Dataset / Parameter Groups / Fitting / QA / Scoring
        │
        ├────────► Modeling MCP Port ─────────────► Primarius MCP Adapter
        ├────────► Persistence Port ──────────────► State / Operation Ledger
        └────────► Telemetry Port ────────────────► Logs / Metrics / Critical Path

Cross-cutting: Compliance / Completion / Performance / Failure Control / Resilience
```

## 3. 分层职责

### 入口层

负责接收运行参数、执行启动前检查、创建正式运行上下文和返回最终状态。入口不得包含参数提取算法。

### 编排层

负责 LangGraph 状态机、节点路由、循环、deadline、重试、检查点、远端结果对账和终止。编排层描述“何时做什么”，不定义具体物理规则。循环、自愈和恢复都必须有确定性上限；恢复机制不能依赖 LLM 可用。

### Agent 角色层

Agent 仅在给定事实、允许动作和预算内做选择：

- Planner：形成分阶段计划。
- Data Analyst：解释数据质量报告。
- Extraction Strategist：选择下一参数组与拟合策略。
- QA Reviewer：解释 QA 失败并提出受约束的 retune 方向。
- Recovery Supervisor：对已知故障选择恢复方案。
- Reporter：生成决策摘要和运行总结草稿。

所有 Agent 输出必须经过结构校验和领域策略验证后才能执行。

### 应用层

负责组合领域能力完成一个完整用例，例如数据审计、单组提取、retune、恢复对账、阶段进度、最终验证、完成判定输入和提交打包。应用层不依赖具体 MCP 或 LLM SDK。

### 领域层

保存系统最重要的确定性规则：

- 器件、数据集和运行状态。
- 操作结果、失败分类、必做/可选阶段和分级交付状态。
- ASM-HEMT 参数分组及依赖。
- 拟合候选、误差比较与接受规则。
- 参数物理范围、QA、评分代理和停止策略。
- 模型卡版本、决策记录和回滚点。

领域层不得依赖 LangGraph、MCP、LLM SDK、数据库或日志框架。

### 端口层

定义外部能力的抽象契约，包括 Modeling MCP、国产大模型、状态持久化和遥测。应用和领域只面向端口，不面向具体供应商。

### 适配层

实现端口，负责协议转换、认证、限流适配、错误归一化和响应解析。首期包括 Primarius Modeling MCP 与国产大模型适配器。

### 基础设施层

负责状态存储、操作意图账本、UTC/span 日志、关键路径聚合、原子检查点、超时/重试/熔断与恢复锁等技术原语。未知结果的业务对账与 fallback 由 `application/recovery` 编排，MCP 适配器负责远端查询；基础设施层不能决定继续拟合、回滚或降级，也不能把“调用超时”直接等同为“远端未执行”。

### 治理层

负责反作弊条件、层级时间与调用预算、单/双器件完成判定、性能回归、未分类失败门禁、提交产物完整性和敏感信息检查。治理规则必须在运行前、运行中和提交前三个时点执行。

## 4. 允许的依赖方向

```text
entrypoints → orchestration/application
orchestration → application/agents/ports
agents → domain/ports
application → domain/ports
adapters → ports/domain contract objects
infrastructure → ports
governance → domain/application contracts
domain → 无外部框架依赖
```

禁止出现：

- 领域层直接调用 MCP 或 LLM。
- LLM 直接修改模型卡或跳过参数边界。
- 适配器自行决定拟合接受、回滚或停止。
- 日志系统成为建模状态的唯一来源。
- 正式测试数据复制到本地数据目录。
- LLM 自行判定模型卡完成、物理正确或恢复成功。
- 对同一器件并行执行模型卡修改、拟合提交、接受或冻结等副作用操作。
- 远端调用结果未知时不经对账直接重试。

## 5. 两器件调度

每个器件拥有完全独立的运行状态、MCP 会话、模型卡版本链和日志。`device_coordination` 只依据两侧的 QA、误差代理、剩余风险和预算分配时间，不共享器件参数。

推荐调度原则：

1. 两套器件先各自取得可提交基线。
2. 再轮流完成关键参数组。
3. 后半程优先改善当前较弱器件。
4. 提交冻结后禁止未经验证的跨器件策略迁移。
5. 同一器件保持单写者；两器件并行必须由官方并发能力开关控制。
6. 性能优化只能并行互不依赖的只读操作，缓存不得跨会话或模型卡版本。

## 6. 安全与数据边界

- 正式测试数据始终通过服务端会话引用，不在本地持久化原始内容。
- Token 和 API Key 只从运行环境注入，不进入配置样例、日志或产物。
- 日志保存调用证据和必要摘要，不泄露认证信息。
- 模型卡、日志和总结在提交前执行 Schema、完整性和敏感信息检查。

## 7. 四项 Agent 能力的架构边界

- **延迟与端到端耗时**：`observability` 记录阶段/节点/LLM/MCP/检查点耗时，`budget` 传播 deadline，`routing` 执行提前停止，`device_coordination` 只做已验证的安全并行。
- **崩溃恢复**：`resilience` 和 `checkpoints` 提供账本、锁和原子检查点，`application/recovery` 调用 MCP 查询能力编排“记录意图 → 调用 → 保存结果 → 对账/续跑”；Recovery Supervisor 只能选择已登记恢复方案。
- **任务完成率**：`run_management` 汇总阶段证据，`validation` 验证模型卡和 QA，`governance/completion` 形成确定性的单器件四级/双器件三级包结果；`submission_status` 单独记录平台交付，执行状态 `FAILED_SAFE` 不等于结果 `DEVICE_FAILED_SAFE`，二者都不冒充成功。
- **环节失败控制**：领域层定义稳定失败分类，基础设施归一化错误，编排层执行有限重试/自愈/回滚/熔断，治理层禁止未分类失败进入发布候选。

详细规则、指标和测试见 [Agent 效率、恢复、完成率与失败控制](08-agent-efficiency-reliability.md)。
