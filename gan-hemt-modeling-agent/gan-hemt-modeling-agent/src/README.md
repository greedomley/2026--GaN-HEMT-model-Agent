# `src` 分层说明

本目录只保留未来实现模块的位置，当前没有任何实现代码。

依赖方向以 `domain` 为核心：外部服务必须通过 `ports` 和 `adapters` 接入；`orchestration` 负责流程，不承载物理规则；`agents` 的所有输出必须由 `application` 和 `domain` 校验。

四项 Agent 横切能力沿用现有分层，不新增顶层架构：

- `application` 维护阶段契约、进度和完成判定输入。
- `domain` 定义操作结果、失败分类、deadline、降级与停止规则。
- `orchestration` 执行性能守卫、循环熔断、对账和恢复路由。
- `infrastructure` 实现性能计时、操作账本、检查点、重试和恢复。
- `governance` 执行单/双器件完成门禁、性能回归和未分类失败门禁。

详细设计见 [Agent 效率、恢复、完成率与失败控制](../docs/08-agent-efficiency-reliability.md)。这些仍是待实现职责，不代表源码模块已经存在。

详细职责见 [模块职责清单](../docs/03-module-catalog.md)。
