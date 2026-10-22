# OpenViking 学习笔记

本组笔记基于 2026-10-22 本地调研源码；请求链路文档注明基线 `f34a54fd1`。源码功能与行号对应该调研版本，当前远端分支可能处于不同版本。

| 文档 | 内容 |
|---|---|
| [20-plugin-gateway-coexistence.md](20-plugin-gateway-coexistence.md) | memory plugin 与 Gateway 共存及自动记忆避让 |
| [21-gateway-concurrency.md](21-gateway-concurrency.md) | 并发机制、百人接入与瓶颈 |
| [23-gateway-server-request-lifecycle.md](23-gateway-server-request-lifecycle.md) | 部署依赖、请求时序、工具循环、后台保存；第 13 节包含 Docker 独立部署、Server 重启降级与 Gateway 滚动发布 |
| [22-gateway-plugin-tradeoffs.md](22-gateway-plugin-tradeoffs.md) | 网关与插件共存：记忆避让机制、九类成本、模型治理与观测收益、日志误读、个人与部门选型及验证边界 |
| [23-main-after-v0.5.0.md](23-main-after-v0.5.0.md) | v0.5.0 后官方 main 的 210 个提交：主要演进、升级兼容性与完整清单 |
| [24-ai-coding-observability.md](24-ai-coding-observability.md) | 部门 AI Coding 观测：采用、方法、交付与成本四层指标，项目/工序归因、个人任务证据、行业方法与最小试点 |

性能和故障恢复结论的实测边界见各篇正文。
