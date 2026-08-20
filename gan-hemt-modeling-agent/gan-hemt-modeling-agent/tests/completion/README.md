# 任务完成率测试规划

任务完成率使用确定性完成判据，不以“状态机走到最后”代替成功。

必须覆盖：

- 单器件 `DEVICE_COMPLETE`、`DEVICE_DEGRADED_DELIVERABLE`、`DEVICE_FAILED_SAFE` 和 `DEVICE_FAILED` 四类结果。
- 双器件包内容 `STRICT_COMPLETE`、`VALID_DEGRADED_PACKAGE` 和 `INVALID_PACKAGE` 三类结果，以及独立的 `submission_status`。
- `applicability` 与 `requirement_level` 独立：确实不适用才标记 `NOT_APPLICABLE`；`SUBMISSION_HARD` 失败使包无效，`STRICT_QUALITY_REQUIRED` 失败最多形成降级包，`OPTIONAL` 跳过不降级。
- 两器件均严格成功并形成 2 卡、2 日志、1 总结。
- 至少一侧受控降级但提交硬门禁全部通过，只计入有效包率；上传确认另行断言。
- 缺少必做产物、合规证据或最终校验时拒绝标记完成。
- 一侧器件完成、一侧失败时成对任务不完成。
- 时间不足时冻结最近有效卡并生成可诊断的完成判定。

每批盲测在 Agent 启动前冻结 manifest、运行数、fixture/seed 和版本；启动后的鉴权、MCP/LLM、崩溃与非法产物失败都保留在分母。只有启动前确认的测试基础设施/夹具无效可留证排除并补跑。

项目初始发布门禁要求最近 10 组成对盲测的有效包率为 10/10、严格包率至少为 9/10，并分别报告；`DEVICE_FAILED_SAFE` 不计成功。10 组只是最低门禁，报告还必须包含 `n`、场景分层和预选置信区间，不能宣称总体可靠率已达到 90%/100%。提交演练另要求有效包在官方/模拟截止前 `submission_status=CONFIRMED`。
