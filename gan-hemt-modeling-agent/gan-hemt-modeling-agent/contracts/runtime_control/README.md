# `contracts/runtime_control` 说明

本目录用于保存项目内部的 Agent 运行控制契约，不属于官方模型卡 Schema。

计划契约包括：

- `StageResult`：阶段前置/后置、`applicability`、`requirement_level`、结果、耗时、证据和下一动作。
- `OperationRecord`：副作用操作意图、稳定幂等键、不可变请求指纹、`attempt_id`、`retry_of_operation_id`、远端任务 ID、结果与对账状态。
- `CheckpointManifest`：检查点 Schema 版本、状态摘要、操作账本水位和文件摘要。
- `CompletionVerdict`：单器件四级结果、双器件三级包结果、`submission_status` 及逐项证据。
- `FailureRecord`：canonical `failure_id/error_code`、stage/category、`retry_disposition`、`side_effect_state`、安全信息、允许动作/选中 fallback、一个主因及零到多个促成因素和全部关联 ID。
- `PerformanceReport`：统一字段 `time_to_first_valid_card_by_device`、`time_to_dual_baseline`、`time_to_best_valid_card_by_device`、`time_to_model_cards_frozen`、`time_to_valid_package`、`time_to_submission_confirmed`、`official_end_to_end_time`，并含 UTC/span、关键路径、等待、检查点开销、样本 manifest 和分位数算法。

正式 Schema 必须版本化并由契约测试验证。这里的内部字段不得冒充官方契约，也不得包含密钥或正式测试原始数据。
