# `artifacts` 说明

用于保存运行产生的模型卡、日志、总结、QA 报告和决策轨迹。

目标结构按器件与全局产物分开：

```text
runs/<run_id>/
├─ devices/<device_id>/
│  ├─ cards/
│  ├─ logs/
│  ├─ qa/
│  └─ checkpoints/
├─ summary/       全局单次运行总结
├─ quality/       性能、恢复、完成和失败内部报告
├─ package/       冻结提交包与 manifest
└─ submission/    上传请求、平台回执和对账状态
```

`quality/` 中的统计必须能由结构化事件重算；盲测聚合率不得写入单次正式总结伪装成正式运行统计。

正式实现时，每次运行应使用独立 `run_id` 隔离，冻结产物不得覆盖。
