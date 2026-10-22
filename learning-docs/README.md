# OpenViking Gateway 学习笔记

本组笔记基于 2026-10-22 本地调研源码；请求链路文档注明基线 `f34a54fd1`。源码功能与行号对应该调研版本，当前远端分支可能处于不同版本。

| 文档 | 内容 |
|---|---|
| [20-plugin-gateway-coexistence.md](20-plugin-gateway-coexistence.md) | memory plugin 与 Gateway 共存及自动记忆避让 |
| [21-gateway-concurrency.md](21-gateway-concurrency.md) | 并发机制、百人接入与瓶颈 |
| [23-gateway-server-request-lifecycle.md](23-gateway-server-request-lifecycle.md) | 部署依赖、请求时序、工具循环、后台保存；第 13 节包含 Docker 独立部署、Server 重启降级与 Gateway 滚动发布 |

性能和故障恢复结论的实测边界见各篇正文。
