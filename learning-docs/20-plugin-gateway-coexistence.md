# 插件与模型网关共存（2026-10-22）

Codex 当前配置同时启用 openviking-memory 0.11.11，并将模型 Base URL 指向本机 1935 网关。共存设计是插件负责记忆，网关转发模型请求，不把两份自动记忆功能叠加。

源码证据：

- `openviking_gateway/protocols.py:259`：`plugin_present` 检查 `x-openviking-plugin` 请求头，或客户端 user/system/developer 消息中的插件标记；`source="gateway-..."` 的网关标记被排除，避免把自身注入误认为插件。
- `openviking_gateway/kernel.py:229`：检测到插件后写入会话级 DISABLED 记录，原因 `plugin_present`。后续请求继续遵守，不因某轮标记消失而重新接管。
- `openviking_gateway/kernel.py:234`：不再作为工具提供方；`:287` 不再确认保存；`:295` 不再触发新的网关压缩；`:304` 不再进行召回和窗口提醒。
- `openviking_gateway/kernel.py:464`：请求日志标记 `plugin_present`，这是主动避让，不能直接当成故障。
- `openviking_gateway/protocols.py:280`：只有 MCP 工具不算插件。该模式下仍由网关召回和保存，只避免再次提供 OpenViking 工具。

使用建议：需要仓库标记、Agent 原生事件和本地工具记录时保留插件；需要集中管理模型凭据和路由时可同时保留网关。若要体验网关自身的记忆与上下文管理，应停用插件并开始新会话。插件存在检测不是插件健康检查，插件自身失败时不可假设网关会自动兜底。

## 首次请求能否注入上下文

能，但不能等同于 Agent 原生 SessionStart。`openviking_gateway/profile.py:211` 的 `build_profile` 按策略和预算读取用户画像，并在提供 read 工具时生成记忆目录与技能目录；`openviking_gateway/kernel.py:551` 调用它。`:585` 将构造的上下文附到当前用户消息，而非将它注册为客户端本地系统配置或技能。网关掌握的是模型 HTTP 请求，不能仅凭这层代理在客户端会话初始化时读取本地仓库配置或执行本地 hook。

本机网关 SQLite 只读检查发现 7 段会话存在 disabled 标记，其中最近记录为 2026-10-22；value 为不可直接 JSON 解码的 BLOB，未进一步解密，也未据此断言特定当前会话的详细原因。

验证边界：本次 CodeGraph 提供了 plugin_present 调用关系。执行 `graphify query --help` 和 `graphify explain plugin_present` 都报告 `graphify-out/graph.json` 不存在；已有图谱产物处于删除状态，因此未重建或恢复，也未声称已获 Graphify 验证。没有执行会话写入端到端验证。
