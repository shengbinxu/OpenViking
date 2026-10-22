# 模型网关并发机制与百人接入评估（2026-10-22）

范围：`openviking_gateway/` 模型 HTTP 网关，不是 VikingBot gateway，也不是 DSH 插件 MCP 调度器。

结论：架构允许多个用户并发请求，可以作为百人接入 PoC 的起点；未压测实际部署，不能宣称已验证百人 SLA。

## 请求并发

- `openviking_gateway/app.py:63`：共享 aiohttp ClientSession，连接池总 limit=1024，limit_per_host=512。是每进程 HTTP 连接资源上限，不是用户数或吞吐保证；模型和 OpenViking HTTP 请求共享此池。
- `openviking_gateway/proxy.py:173`：每个请求异步运行；`:182` 将 JSON 解析移到线程；`:194` 异步请求上游。
- `openviking_gateway/proxy.py:472`：SSE 按块异步转发，`:493` yield chunk。等待某条流的网络 I/O 不会独占整个事件循环。
- `openviking_gateway/kernel.py:160`：大历史的链计算通过 asyncio.to_thread 卸载。线程卸载避免直接堵塞事件循环，不保证 CPU 吞吐无限增长。

## 状态隔离与竞争

- `openviking_gateway/kernel.py:197`：scope 来自 account、user_id、protocol；`:151` 和 `:201` 解析会话身份。部署需要为不同成员绑定各自 user_id，不能假设同一用户凭据代表不同人。
- `openviking_gateway/state_store.py:44`：SQLite 事务加版本条件更新，用 CAS 检测状态竞争。这不等于把整个模型请求串行化。
- `openviking_gateway/storage.py:62`：每个 Database 实例配置四个读线程、一个写线程；`:86` 启用 WAL；`:140` 通过 executor 执行数据库工作。写事务会排队；多进程仍受同一 SQLite 文件的写锁约束。

## 记忆链路

- `openviking_gateway/client.py:88`：召回 POST /api/v1/search/search，关闭 query expansion 与 rewrite；`openviking_gateway/models.py:41` 默认 recall_timeout=2 秒。
- `openviking_gateway/kernel.py:545`：召回出错或超时返回降级结果；`:565` 并发获取召回与开场画像。2 秒是召回调用超时，不能概括为所有预处理总耗时上限。
- `openviking_gateway/proxy.py:339`：完成回复的部分记账先于流结束；`:504` 使用 BackgroundTask 执行 finish。因此不能说所有持久化完全脱离响应路径。
- `openviking_gateway/app.py:110`：每进程一个 maintenance 循环，`:131` await worker.once()；`openviking_gateway/capture.py:310` 每次 claim 一项再 await deliver。这意味着每进程后台投递逐项推进。
- `openviking_gateway/capture_store.py:77`：通过事务与租约领取任务，支持多个 worker 竞争领取；`:84` 按 ready 取一项。慢投递可造成其他会话的记忆落库延迟。
- `openviking_gateway/config.py:150`：独立网关默认 workers=1，范围 1~64；`openviking_gateway/cli.py:153` 传给 uvicorn。内嵌模式移除了独立 gateway workers 配置（config.py:136），不能直接套用独立网关参数。

## 容量与验证

示例估算（不是测量）：100 人平均每 30 秒发一次模型调用，约 3.3 RPS；平均流持续 20 秒，平均在途约 67。Agent 工具循环和并行子任务会放大请求率。平均在途约等于请求率乘平均耗时。

部署验证应覆盖 100 条长 SSE、真实 Coding 历史、多用户 key、召回/写入/压缩策略，以及持续运行后的队列积压。观测端到端首 token、网关额外延迟、错误率、上游 429、内存、SQLite 等待和记忆投递延迟。

`scripts/gateway_benchmark.py:1` 使用本地合成上游，`:100` 关闭 recall/profile/capture/gateway_tools；`:298` 默认 concurrency=300。只能验证限定条件下的网关开销，不能证明完整百人记忆场景。本次 python3 导入 aiohttp 失败，未运行压测，也未改动环境。

Graphify 验证：执行 `graphify explain ProxyRequest --budget 1400` 返回无匹配节点；`graphify query openviking_gateway --budget 700` 返回 VikingBot 相关节点。现有图谱没有提供模型网关调用链证据。未重建索引，以上行为判断来自当前源码。
