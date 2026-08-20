# 环节失败控制测试规划

针对 preflight/auth、session、data audit、model init、planning/LLM、MCP/tool、fitting、QA、loop/budget、persistence/recovery 和 artifacts/submission 分别建立失败用例。

每个用例必须验证：

- 产生 canonical `FailureRecord`：含 `failure_id/error_code`、stage/category、`retry_disposition`、`side_effect_state`、允许动作/选中 fallback、主因状态、一个主因及零到多个促成因素和全部关联 ID。
- 逻辑错误不盲重试，瞬时错误不超过预算，未知结果先对账。
- 有限自愈每次改变动作或参数，同一动作重复超限后熔断。
- fallback 不绕过参数边界、QA、合规和产物校验。
- 失败进入回归库并能聚合到明确类别。
- 默认策略下，确认无副作用的瞬时故障不超过 3 次退避重试，结构/参数修正不超过 2 次，同证据同动作指纹出现 2 次即熔断；所有上限均从版本化配置读取并受 deadline 约束。

发布候选不得存在未分类失败或无界循环。

测试前预登记分母，并分别报告 `stage_terminal_failure_rate`、`operation_failure_rate`、`attempt_failure_rate`、`retry_amplification`、`outcome_unknown_rate` 和 `self_heal_success_rate`。尝试失败但最终恢复与终态失败必须分开；同 fixture/seed 的配对 A/B 不得越过冻结基线的版本化非劣阈值。
