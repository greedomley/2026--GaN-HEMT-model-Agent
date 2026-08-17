# `src` 分层说明

本目录只保留未来实现模块的位置，当前没有任何实现代码。

依赖方向以 `domain` 为核心：外部服务必须通过 `ports` 和 `adapters` 接入；`orchestration` 负责流程，不承载物理规则；`agents` 的所有输出必须由 `application` 和 `domain` 校验。

详细职责见 [模块职责清单](../docs/03-module-catalog.md)。

