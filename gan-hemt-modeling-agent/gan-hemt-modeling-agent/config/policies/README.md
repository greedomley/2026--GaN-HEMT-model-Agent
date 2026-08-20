# `config/policies` 说明

本目录用于保存无密钥、可版本化的运行策略配置：

- `performance`：阶段 deadline、轮询间隔、只读并行度、缓存 TTL 和提前停止阈值。
- `recovery`：检查点边界、重试预算、恢复锁、未知结果对账和检查点回退规则。
- `completion`：阶段 `applicability`、`SUBMISSION_HARD/STRICT_QUALITY_REQUIRED/OPTIONAL`、官方/项目安全硬 QA/软目标分层，以及单器件、双器件包结果和提交状态条件。
- `failure_control`：失败分类、有限自愈次数、熔断和 fallback 路由。

本目录只保存配置实例值及其引用的 `schema_version`；验证 Schema 的唯一真源位于 `contracts/runtime_control/`。具体文件格式在工程基线确定后统一。官方契约未确认前，所有可能受官方规则影响的数值必须标记为暂定；Token、API Key、正式数据引用和个人绝对路径不得写入。
